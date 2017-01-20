# Setup Kafka

## Create a local Docker network
That the local development container can communicate with each other we've to extend
our local Docker container with the network. Therefore we first have to create a
Docker network:

```
docker network create battleapp
```

The updated start scripts of the existing Docker images looks like the following.  
`--net battleapp` is the new part of the scripts.

```
docker stop keycloak && docker rm keycloak
docker run -d \
--net battleapp \
-p 8080:8080 \
-v /home/battleapp/Desktop/dockervolumes/keycloakdata/:/opt/jboss/keycloak/standalone/data \
--name keycloak \
jboss/keycloak:2.4.0.Final
```

```
#!/usr/bin/env bash

docker stop cassandra
docker rm cassandra
docker run --name cassandra -d \
--net battleapp \
-e CASSANDRA_START_RPC=true \
-p 9160:9160 -p \
9042:9042 -p \
7199:7199 -p \
7001:7001 -p \
7000:7000 \
cassandra
echo "wait for cassandra to start"
while ! docker logs cassandra | grep "Listening for thrift clients..."
do
 echo "$(date) - still trying"
 sleep 1
done
echo "$(date) - connected successfully"

echo "copy init script in container"
docker cp /home/battleapp/Desktop/initial_db.sql cassandra:/

echo "create database"
docker exec -d cassandra cqlsh localhost -f /initial_db.sql
```

## Setup Kafka on the local machine
To start Kafka on our local machine we've to execute the following script:
```
echo "stop and rm kafka"
docker stop kafka && docker rm kafka

echo "stop and rm zookeeper"
docker stop zookeeper && docker rm zookeeper

echo "start zookeeper"
docker run --name zookeeper -d \
--net battleapp \
-p 2181:2181 \
-e ZOOKEEPER_ID="1" \
-e ZOOKEEPER_SERVER_1=kafka-zoo-svc \
digitalwonderland/zookeeper

echo "start kafka"
docker run --name kafka -d -p 9092:9092 \
--net battleapp \
--hostname "kafka" \
-e ENABLE_AUTO_EXTEND="true" \
-e KAFKA_RESERVED_BROKER_MAX_ID="999999999" \
-e KAFKA_AUTO_CREATE_TOPICS_ENABLE="true" \
-e KAFKA_PORT="9092" \
-e KAFKA_ADVERTISED_PORT="9092" \
-e KAFKA_ADVERTISED_HOST_NAME="kafka" \
-e KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 \
wurstmeister/kafka
```

## Setup Kafka in the cluster
Kafka needs Zookeeper for service discovery therefore we've to create a Zookeeper service and
a Kafka service.

```
apiVersion: v1
kind: Service
metadata:
  name: zookeeper
  labels:
    name: zookeeper
spec:
  ports:
  - name: client
    port: 2181
    protocol: TCP
  - name: follower
    port: 2888
    protocol: TCP
  - name: leader
    port: 3888
    protocol: TCP
  selector:
    name: zookeeper
```

```
apiVersion: v1
kind: Service
metadata:
  name: kafka
  labels:
    name: kafka
spec:
  ports:
  - name: kafka-port
    port: 9092
    protocol: TCP
  selector:
    name: kafka
```

And here the services for the test environment:

```
apiVersion: v1
kind: Service
metadata:
  name: zookeeper
  namespace: test
  labels:
    name: zookeeper
spec:
  ports:
  - name: client
    port: 2181
    protocol: TCP
  - name: follower
    port: 2888
    protocol: TCP
  - name: leader
    port: 3888
    protocol: TCP
  selector:
    name: zookeeper
```

```
apiVersion: v1
kind: Service
metadata:
  name: kafka
  namespace: test
  labels:
    name: kafka
spec:
  ports:
  - name: kafka-port
    port: 9092
    protocol: TCP
  selector:
    name: kafka
```

Then we've to start the services:

```
kc create -f zookeeper.yml
kc create -f zookeeper-test.yml
kc create -f kafka.yml
kc create -f kafka-test.yml
```

Now we can create the deployments for each environment:

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: zookeeper
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: zookeeper
    spec:
      containers:
      - env:
        - name: ZOOKEEPER_ID
          value: "1"
        - name: ZOOKEEPER_SERVER_1
          value: zookeeper
        name: zookeeper
        image: digitalwonderland/zookeeper
        ports:
        - containerPort: 2181
```

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kafka
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: kafka
    spec:
      containers:
      - env:
        - name: ENABLE_AUTO_EXTEND
          value: "true"
        - name: KAFKA_RESERVED_BROCKER_MAX_ID
          value: "999999999"
        - name: KAFKA_AUTO_CREATE_TOPICS_ENABLE
          value: "true"
        - name: KAFKA_PORT
          value: "9092"
        - name: KAFKA_ADVERTISED_PORT
          value: "9092"
        - name: KAFKA_ADVERTISED_HOST_NAME
          value: "kafka"
        - name: KAFKA_ZOOKEEPER_CONNECT
          value: zookeeper:2181
        name: kafka
        image: wurstmeister/kafka
        ports:
        - containerPort: 9092
```

And the deployments for the test environment:

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: zookeeper
  namespace: test
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: zookeeper
    spec:
      containers:
      - env:
        - name: ZOOKEEPER_ID
          value: "1"
        - name: ZOOKEEPER_SERVER_1
          value: zookeeper
        name: zookeeper
        image: digitalwonderland/zookeeper
        ports:
        - containerPort: 2181
```

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kafka
  namespace: test
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: kafka
    spec:
      containers:
      - env:
        - name: ENABLE_AUTO_EXTEND
          value: "true"
        - name: KAFKA_RESERVED_BROCKER_MAX_ID
          value: "999999999"
        - name: KAFKA_AUTO_CREATE_TOPICS_ENABLE
          value: "true"
        - name: KAFKA_PORT
          value: "9092"
        - name: KAFKA_ADVERTISED_PORT
          value: "9092"
        - name: KAFKA_ADVERTISED_HOST_NAME
          value: "kafka"
        - name: KAFKA_ZOOKEEPER_CONNECT
          value: zookeeper:2181
        name: kafka
        image: wurstmeister/kafka
        ports:
        - containerPort: 9092
```

Then we've to start the deployments:

```
kc create -f zookeeper.yml
kc create -f zookeeper-test.yml
kc create -f kafka.yml
kc create -f kafka-test.yml
```

We can test the setup with the following commands:

```
kc get po -l 'name in (zookeeper,kafka)'
```
```
NAME                         READY     STATUS    RESTARTS   AGE
kafka-1770102436-zn8qr       1/1       Running   0          5m
zookeeper-1239941466-pb4bx   1/1       Running   0          8m
```