# Deploy the Docker image of the service in Kubernetes
To deploy the service in Kubernetes we need a `deployment` and a `service` 
as well as infrastructure with a Docker registry. How to setup a Docker registry is
described [here](../03_Docker_registry).

## Build and push the service to the registry
If we have a Docker registry we can push our image to the registry. In the root folder
of our service we build the Docker image with the registry information.
```
docker build -t disruptor.ninja:30500/robertbrem/battleapp:1.0.0 .
```
Now we can push the image to the repository:
```
docker login disruptor.ninja:30500 --username=rob --password=1234
docker push disruptor.ninja:30500/robertbrem/battleapp:1.0.0
```

## Deploy the service in Kubernetes
To use the registry we need to create a Kubernetes secret:
```
kc create secret docker-registry registrykey --docker-username=rob --docker-password=1234 --docker-email=brem_robert@hotmail.com --docker-server=disruptor.ninja:30500
```

The `deployment` looks like this:
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: battleapp
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: battleapp
    spec:
      containers:
      - resources:
        name: battleapp
        image: disruptor.ninja:30500/robertbrem/battleapp:1.0.0
        ports:
        - name: port
          containerPort: 8080
      imagePullSecrets:
      - name: registrykey
```
Then we can create this deployment:
```
kc create -f deployment.yml
```
To see the REST service we have to expose it to the internet with a Kubernetes service:
```
apiVersion: v1
kind: Service
metadata:
  name: battleapp
  labels:
    name: battleapp
spec:
  ports:
  - port: 8081
    targetPort: 8080
    nodePort: 30080
  selector:
    name: battleapp
  type: NodePort
```
The service is now running on the following url:  
```
http://disruptor.ninja:30080/battleapp/resources/users
```  
The output should be something like that:
```
[{"name":"dan"},{"name":"robert"},{"name":"kevin"}]
```