# Setup Jenkins in Kubernetes
To deploy often we must have CI/CD. In this case we are using Jenkins.

## Start Jenkins without mounting `jenkins_home`
Jenkins has persistent data therefore we have to mount this data somewhere.
The easiest way to do that is over a node selector. A more advanced solution would be
to use GlusterFS, Flocker, NFS or something similar.

If we use node selectors for our persistence then we have to label a node.
In this case this is `5.189.153.209`.
```
kc label nodes vmi71989.contabo.host name=vmi71989
```
Now we create the Jenkins deployment without mounting the `jenkins_home`. I'm
using [this file](https://gist.github.com/robertBrem/77e7a97b57565921792631f70088c706)
as a reference.

Later we want to be able the push Docker images in the Docker registry therefore Jenkins
need to know a username and password. For this we can create another secret:
```
kc create secret generic registrykeygeneric --from-literal=username=rob --from-literal=password=1234
```

For our first deployment I'm going to comment the `jenkins_home` volume mount.
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jenkins
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: jenkins
    spec:
      containers:
      - resources:
        name: jenkins
        image: robertbrem/jenkins:1.0.5
        ports:
        - name: ui
          containerPort: 8080
        - name: hooks
          containerPort: 50000
        volumeMounts:
        - mountPath: /var/run/docker.sock
          name: docker-socket
#        - mountPath: /var/jenkins_home
#          name: jenkins-home
        env:
        - name: REGISTRY_USERNAME
          valueFrom:
            secretKeyRef:
              name: registrykeygeneric
              key: username
        - name: REGISTRY_PASSWORD
          valueFrom:
            secretKeyRef:
              name: registrykeygeneric
              key: password
      volumes:
      - name: docker-socket
        hostPath:
          path: /var/run/docker.sock
#      - name: jenkins-home
#        hostPath:
#          path: /root/jenkins_home/
      nodeSelector:
        name: vmi71989
```
Start the deployment:
```
kc create -f deployment.yml
```

## Create the persistence for `jenkins_home`
While jenkins is running connect on the server where the `nodeSelector` is pointing to.
```
ssh root@5.189.153.209
```
Now we have to find the container where Jenkins is running:
```
sudo docker ps  | grep jenkins
```
```
a80da2e388e1        robertbrem/jenkins:1.0.2                           "/bin/bash -c ./run.s"   2 minutes ago       Up 2 minutes                            k8s_jenkins.f24a85d5_jenkins-1074834219-nqlfc_default_ec99c5d7-c941-11e6-a836-0050563cad2a_5e2790be
157591884d20        gcr.io/google_containers/pause-amd64:3.0           "/pause"                 5 minutes ago       Up 5 minutes                            k8s_POD.d8dbe16c_jenkins-1074834219-nqlfc_default_ec99c5d7-c941-11e6-a836-0050563cad2a_569a356a
```
In our case this is `a80da2e388e1`. Now we copy the content of `jenkins_home` on the server.
```
sudo docker cp a80da2e388e1:/var/jenkins_home .
```
That the container can use this mount we have to change the rights of the folder.
```
sudo chown -R 1000:1000 jenkins_home/
```

## Start Jenkins with the `jenkins_home` mount
Now we can stop the current Jenkins deployment:
```
kc delete deployment jenkins
```
Then we uncomment the previously commented lines of the deployment and redeploy it.
```
kc create -f deployment.yml
```

## Make Jenkins visible
That Jenkins is visible outside the cluster we have to create a service:
```
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  labels:
    name: jenkins
spec:
  ports:
  - port: 8082
    targetPort: 8080
    nodePort: 30180
  selector:
    name: jenkins
  type: NodePort
```
To start using Jenkins we need the administrator password that is logged in the container.
We can access the log of a container over `kubectl`.
```
kc get po -l name=jenkins
```
```
NAME                       READY     STATUS    RESTARTS   AGE
jenkins-1074834219-7jws0   1/1       Running   0          3m
```
Now we can display the log:
```
kc logs jenkins-1074834219-7jws0
```
Here we can find the administrator password that we need when we start Jenkins on:
```
http://disruptor.ninja:30180
```
