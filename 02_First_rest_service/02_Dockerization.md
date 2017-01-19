# Dockerization of the service

Kubernetes uses containers for the orchestration. Therefore we have to pack our service in 
a container. I'm using Docker as container technology.

At first we have to create a `Dockerfile` in the root folder of the project with the following
content:
```
FROM airhacks/wildfly

MAINTAINER Robert Brem <brem_robert@hotmail.com>

ADD target/battleapp.war ${DEPLOYMENT_DIR}
```

To test it just build the application:
```
mvn clean install
```
Then build the Docker image:
```
docker build -t battleapp .
```
After the build you can start a container for this image:
```
docker run -p 8081:8080 --name battleapp -d battleapp
```
The service is now running on the port 8081:  
```
http://localhost:8081/battleapp/resources/users
```  
The output should be something like that:
```bash
[{"name":"dan"},{"name":"robert"},{"name":"kevin"}]
```