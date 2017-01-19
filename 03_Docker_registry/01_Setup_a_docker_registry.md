# Setup a docker registry

## Create folders for the persistence
The Docker registry has persistent data therefore we've to mount this data somewhere.
The easiest way to do that is over a node selector. A more advanced solution would be
to use GlusterFS, Flocker, NFS or something similar.

If we use node selectors for our persistence then we've to log in to the server
that ip is used in the DNS registry. In my case that's the master.

```
ssh root@5.189.173.45
```
On the server we've to create the folder structure that gets mounted to the host.
In our case we have three folders one for the Docker images, one for our ssl certificates
and the last for the authorization information.  
```
mkdir -p registry/{images,certs,auth}
sudo docker run --entrypoint htpasswd registry:2 -Bbn rob 1234 > registry/auth/htpasswd
```
The last command creates a user with a password for the Docker registry.

## Create SSL certificates
We are going to create SSL certificates with LetsEncrypt. Therefore we have to install
LetsEncrypt on the server:
```
sudo yum update -y
sudo yum install epel-release -y
sudo yum install letsencrypt -y
```
Now we can create the certificates:
```
sudo letsencrypt certonly -d disruptor.ninja
```
The certificates are created under the following folder:
```
/etc/letsencrypt/live/disruptor.ninja
```
The certificates have to be visible for the Docker registry therefore we copy them to
the `certs` folder we have created:
```
sudo cp /etc/letsencrypt/live/disruptor.ninja/fullchain.pem registry/certs/
sudo cp /etc/letsencrypt/live/disruptor.ninja/privkey.pem registry/certs/
```

## Kubernetes deployment
To tell Kubernetes to schedule the Docker registry pod on the master node we've to 
label the node:
```
kc label nodes vmi74448.contabo.host name=vmi74448
```
Next we create a `deployment.yml` file [like this one](https://gist.github.com/robertBrem/3df0c7d672a9942bbbddb45d0b6f297a).
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: registry
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: registry
    spec:
      containers:
      - resources:
        name: registry
        image: registry:2
        ports:
        - name: registry-port
          containerPort: 5000
        volumeMounts:
        - mountPath: /var/lib/registry
          name: images
        - mountPath: /certs
          name: certs
        - mountPath: /auth
          name: auth
        env:
        - name: REGISTRY_AUTH
          value: "htpasswd"
        - name: REGISTRY_AUTH_HTPASSWD_REALM
          value: "Registry Realm"
        - name: REGISTRY_AUTH_HTPASSWD_PATH
          value: /auth/htpasswd
        - name: REGISTRY_HTTP_TLS_CERTIFICATE
          value: /certs/fullchain.pem
        - name: REGISTRY_HTTP_TLS_KEY
          value: /certs/privkey.pem
      volumes:
      - name: images
        hostPath:
          path: /root/registry/images
      - name: certs
        hostPath:
          path: /root/registry/certs
      - name: auth
        hostPath:
          path: /root/registry/auth
      nodeSelector:
        name: vmi74448
```
This file has to be deployed:
```
kc create -f deployment.yml
```
To test if the deployment is working you can display all pods:
```
kc get po
```
```
NAME                      READY     STATUS    RESTARTS   AGE
registry-95525520-9rdvc   1/1       Running   0          1m
```

## Kubernetes service
To make the registry visible outside the cluster we have to create a Kubernetes service.
The `service.yml` file can be created similar to [this file](https://gist.github.com/robertBrem/68706f161388b7307bb0).
```
apiVersion: v1
kind: Service
metadata:
  name: registry
  labels:
    name: registry
spec:
  ports:
  - port: 5001
    targetPort: 5000
    nodePort: 30500
  selector:
    name: registry
  type: NodePort
```
To test the registry we can call the following url:
```
https://disruptor.ninja:30500/v2/_catalog
```