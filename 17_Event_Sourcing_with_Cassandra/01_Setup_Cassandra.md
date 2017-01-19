# Setup Cassandra

## Setup Cassandra on the local machine
To start Cassandra on our local machine we've to execute the following
script:
```
#!/usr/bin/env bash

docker stop cassandra
docker rm cassandra
docker run --name cassandra -d -e CASSANDRA_START_RPC=true -p 9160:9160 -p 9042:9042 -p 7199:7199 -p 7001:7001 -p 7000:7000 cassandra
echo "wait for cassandra to start"
while ! docker logs cassandra | grep "Listening for thrift clients..."
do
 echo "$(date) - still trying"
 sleep 1
done
echo "$(date) - connected successfully"

echo "copy init script in container"
docker cp initial_db.sql cassandra:/

echo "create database"
docker exec -d cassandra cqlsh localhost -f /initial_db.sql
```

`initial_db.sql` contains the creation of the table:
```
CREATE KEYSPACE battleapp WITH REPLICATION = { 'class' : 'SimpleStrategy','replication_factor' : 3 };

USE battleapp;

CREATE TABLE EVENTS (
 ID text,
 NAME text,
 VERSION bigint,
 DATE timestamp,
 DATA text,
 PRIMARY KEY(ID, NAME, VERSION)
);
```

## Setup Cassandra in the cluster
To have two separate Cassandra cluster; one for the test environment and one
for the production environment we've to separate the test environment from 
the production environment. That can be achieved with Kubernetes namespaces.
We let our production environment run on the default namespace and the
test environment in a new namespace called `test`. Therefore we've to create
this new namespace in a file:
```
apiVersion: v1
kind: Namespace
metadata:
  name: test
```

Now we've to tell Kubernetes to create this namespace:
```
kc create -f namespace.yml
```

We can test if the namespace was created with the following command:
```
kc get namespace | grep test
```
```
test          Active        6h
```

For Cassandra we're going to use Kubernetes deamonsets. A deamonset
starts on each node on the cluster a pod that's defined in the
deamonset. We don't want to start Cassandra on every node therefore 
we create more node labels. In this case we set the label `group=cassandra`
on three nodes:

```
kc label nodes vmi71989.contabo.host group=cassandra
kc label nodes vmi71992.contabo.host group=cassandra
kc label nodes vmi71992.contabo.host group=cassandra
```

We can test this setting with the following command:
```
kc get no -l group=cassandra
```
```
NAME                    STATUS    AGE
vmi71989.contabo.host   Ready     6d
vmi71992.contabo.host   Ready     10d
vmi74389.contabo.host   Ready     10d
```

This is are going to be the nodes for our production Cassandra cluster.
For our test environment we specify just one node with the label
`group=cassandra-test`.

```
kc label nodes vmi74388.contabo.host group=cassandra
```
```
kc get no -l group=cassandra-test
```
```
NAME                    STATUS    AGE
vmi74388.contabo.host   Ready     10d
```

Now we've to create a service for each environment:

```
apiVersion: v1
kind: Service
metadata:
  labels:
    name: cassandra
  name: cassandra
spec:
  clusterIP: None
  ports:
    - port: 9042
  selector:
    name: cassandra
```

And here the service for the test environment:

```
apiVersion: v1
kind: Service
metadata:
  labels:
    name: cassandra
  name: cassandra
  namespace: test
spec:
  clusterIP: None
  ports:
    - port: 9042
  selector:
    name: cassandra
```

Then we've to start the services:

```
kc create -f cassandra.yml
kc create -f cassandra-test.yml
```

Now we can create the daemonset for each environment:

```
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  labels:
    name: cassandra
  name: cassandra
spec:
  template:
    metadata:
      labels:
        name: cassandra
    spec:
      # Filter to specific nodes:
      nodeSelector:
        group: cassandra
      containers:
        - command:
            - /run.sh
          env:
            - name: MAX_HEAP_SIZE
              value: 512M
            - name: HEAP_NEWSIZE
              value: 100M
            - name: CASSANDRA_SEED_PROVIDER
              value: "io.k8s.cassandra.KubernetesSeedProvider"
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          image: gcr.io/google-samples/cassandra:v11
          name: cassandra
          ports:
            - containerPort: 7000
              name: intra-node
            - containerPort: 7001
              name: tls-intra-node
            - containerPort: 7199
              name: jmx
            - containerPort: 9042
              name: cql
          volumeMounts:
            - mountPath: /cassandra_data
              name: data
      volumes:
        - name: data
          emptyDir: {}
```

And the daemonset for the test environment:

```
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  labels:
    name: cassandra
  name: cassandra
  namespace: test
spec:
  template:
    metadata:
      labels:
        name: cassandra
    spec:
      # Filter to specific nodes:
      nodeSelector:
        group: cassandra-test
      containers:
        - command:
            - /run.sh
          env:
            - name: MAX_HEAP_SIZE
              value: 512M
            - name: HEAP_NEWSIZE
              value: 100M
            - name: CASSANDRA_SEED_PROVIDER
              value: "io.k8s.cassandra.KubernetesSeedProvider"
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          image: gcr.io/google-samples/cassandra:v11
          name: cassandra
          ports:
            - containerPort: 7000
              name: intra-node
            - containerPort: 7001
              name: tls-intra-node
            - containerPort: 7199
              name: jmx
            - containerPort: 9042
              name: cql
          volumeMounts:
            - mountPath: /cassandra_data
              name: data
      volumes:
        - name: data
          emptyDir: {}
```

Then we've to start the daemonsets:

```
kc create -f cassandra.yml
kc create -f cassandra-test.yml
```

We can test the setup with the following commands:
```
kc get po -l name=cassandra
```
```
NAME              READY     STATUS    RESTARTS   AGE
cassandra-19qlh   1/1       Running   0          20h
cassandra-rtgqx   1/1       Running   0          20h
cassandra-tbhz1   1/1       Running   0          20h
```
```
kc exec -it cassandra-19qlh -- nodetool status
```
```
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address    Load       Tokens       Owns (effective)  Host ID                               Rack
UN  10.36.0.3  132.31 KiB  32           71.4%             c6e9ae7e-3806-4e7b-9d7e-f1846e71f33f  rack1
UN  10.42.0.3  132.33 KiB  32           76.2%             3026572a-f232-43fb-a63c-a32b5efb4b32  rack1
UN  10.40.0.5  148.29 KiB  32           67.9%             7f7b469e-8a6b-401d-92de-6a5b363d92e9  rack1
```

The same for the test environment:

```
kc --namespace test get po -l name=cassandra
```
```
NAME              READY     STATUS    RESTARTS   AGE
cassandra-cgj4q   1/1       Running   0          6h
```
```
kc --namespace test exec -it cassandra-cgj4q -- nodetool status
```
```
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address    Load       Tokens       Owns (effective)  Host ID                               Rack
UN  10.44.0.3  124.75 KiB  32           100.0%            5939fe71-23dd-49f5-ae87-4ce174656fb9  rack1
```

Finally we've to create the Cassandra namespace and table in both environments
similar to the local setting:

```
kc exec -it cassandra-19qlh cqlsh cassandra
```
```
Connected to Test Cluster at cassandra:9042.
[cqlsh 5.0.1 | Cassandra 3.9 | CQL spec 3.4.2 | Native protocol v4]
Use HELP for help.
cqlsh>
```

And copy and past the content of the `initial_db.sql`:

```
CREATE KEYSPACE battleapp WITH REPLICATION = { 'class' : 'SimpleStrategy','replication_factor' : 3 };

USE battleapp;

CREATE TABLE EVENTS (
 ID text,
 NAME text,
 VERSION bigint,
 DATE timestamp,
 DATA text,
 PRIMARY KEY(ID, NAME, VERSION)
);
```

The same for the test environment:

```
kc --namespace test exec -it cassandra-cgj4q cqlsh cassandra
```