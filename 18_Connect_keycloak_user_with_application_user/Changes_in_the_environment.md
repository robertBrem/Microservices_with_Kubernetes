# Changes in the environment scripts

## Changes to the start test environment script
The new deployment in the `start.js` script have to look like
the following snippet:

```
...
var version = $ENV.VERSION;
var kubectl = $ENV.KUBECTL;

var name = "battleapp";
var image = "disruptor.ninja:30500/robertbrem/battleapp:" + version;
var replicas = 1;
var port = 8080;
var clusterPort = 8088;
var nodePort = 31080;
var deploymentFileName = "deployment.yml";
var serviceFileName = "service.yml";
var registrysecret = "registrykey";
var url = "http://disruptor.ninja:31080/battleapp/resources/health";
var timeout = 2;
var realmName = "battleapp";
var authServerUrl = "https://disruptor.ninja:31182/auth";
var cassandraHost = "cassandra";
var namespace = "test";
var keycloakMasterClientId = "master";
...
var dfw = new FileWriter(deploymentFileName);
dfw.write("apiVersion: extensions/v1beta1\n");
dfw.write("kind: Deployment\n");
dfw.write("metadata:\n");
dfw.write("  name: " + name + "\n");
dfw.write("  namespace: " + namespace + "\n");
dfw.write("spec:\n");
dfw.write("  replicas: " + replicas + "\n");
dfw.write("  template:\n");
dfw.write("    metadata:\n");
dfw.write("      labels:\n");
dfw.write("        name: " + name + "\n");
dfw.write("    spec:\n");
dfw.write("      containers:\n");
dfw.write("      - resources:\n");
dfw.write("        name: " + name + "\n");
dfw.write("        image: " + image + "\n");
dfw.write("        ports:\n");
dfw.write("        - name: port\n");
dfw.write("          containerPort: " + port + "\n");
dfw.write("        env:\n");
dfw.write("        - name: KEYCLOAK_MASTER_CLIENT_ID\n");
dfw.write("          value: \"" + keycloakMasterClientId + "\"\n");
dfw.write("        - name: KEYCLOAK_MASTER_USER_NAME\n");
dfw.write("          valueFrom:\n");
dfw.write("            secretKeyRef:\n");
dfw.write("              name: keycloak-master\n");
dfw.write("              key: username\n");
dfw.write("        - name: KEYCLOAK_MASTER_PASSWORD\n");
dfw.write("          valueFrom:\n");
dfw.write("            secretKeyRef:\n");
dfw.write("              name: keycloak-master\n");
dfw.write("              key: password\n");
dfw.write("        - name: REALM_NAME\n");
dfw.write("          value: \"" + realmName + "\"\n");
dfw.write("        - name: AUTH_SERVER_URL\n");
dfw.write("          value: \"" + authServerUrl + "\"\n");
dfw.write("        - name: CASSANDRA_ADDRESS\n");
dfw.write("          value: \"" + cassandraHost + "\"\n");
dfw.write("      imagePullSecrets:\n");
dfw.write("      - name: " + registrysecret + "\n");
dfw.close();
...
```

We're using a new secret that we've to create:

```
kc create secret generic keycloak-master --from-literal=username=admin --from-literal=admin
```

## Changes to the start canary script
Similar to the test environment script we've to extend the
deployment in the canary with the following parts:

```
...
var keycloakMasterClientId = "master";
...
dfw.write("        - name: KEYCLOAK_MASTER_CLIENT_ID\n");
dfw.write("          value: \"" + keycloakMasterClientId + "\"\n");
dfw.write("        - name: KEYCLOAK_MASTER_USER_NAME\n");
dfw.write("          valueFrom:\n");
dfw.write("            secretKeyRef:\n");
dfw.write("              name: keycloak-master\n");
dfw.write("              key: username\n");
dfw.write("        - name: KEYCLOAK_MASTER_PASSWORD\n");
dfw.write("          valueFrom:\n");
dfw.write("            secretKeyRef:\n");
dfw.write("              name: keycloak-master\n");
dfw.write("              key: password\n");
...
```

## Changes to the pipeline
The `Jenkisnfile` has to be changed to the following:

```
withEnv([   "VERSION=1.0.${currentBuild.number}",
            "REGISTRY_EMAIL=brem_robert@hotmail.com",
            "KUBECTL=kubectl",
            "HOST=disruptor.ninja",
            "REALM_NAME=battleapp",
            "KEYCLOAK_URL=https://disruptor.ninja:31182/auth",
             "PORT=31080"]) {

  stage "checkout, build, test and publish"
  node {
    git url: "http://disruptor.ninja:30130/rob/battleapp"
    def mvnHome = tool 'M3'
    sh "${mvnHome}/bin/mvn clean install"
    sh "./build.js"
  }

  stage "start test environment"
  node {
    git url: "http://disruptor.ninja:30130/rob/battleapp-starttestenv"
    sh "./start.js"
  }

  stage "system test"
  node {
    git url: "http://disruptor.ninja:30130/rob/battleapp-st"
    def mvnHome = tool 'M3'
    sh "${mvnHome}/bin/mvn clean install failsafe:integration-test failsafe:verify"
    step([$class: 'JUnitResultArchiver', testResults: '**/target/failsafe-reports/TEST-*.xml'])
  }

  stage "ui test"
  node {
    git url: "http://disruptor.ninja:30130/rob/battleapp-uit"
    def mvnHome = tool 'M3'
    sh "${mvnHome}/bin/mvn clean install failsafe:integration-test failsafe:verify"
    step([$class: 'JUnitResultArchiver', testResults: '**/target/failsafe-reports/TEST-*.xml'])
  }

  stage "last test"
  node {
    git url: "http://disruptor.ninja:30130/rob/battleapp-lt"
    def mvnHome = tool 'M3'
    sh "${mvnHome}/bin/mvn clean verify -Dperformancetest.webservice.host=disruptor.ninja -Dperformancetest.webservice.port=31080 -Dperformancetest.webservice.threads=5 -Dperformancetest.webservice.iterations=500 -Dperformancetest.webservice.url=/battleapp/resources/health"
    archiveArtifacts artifacts: 'target/reports/*.*', fingerprint: true
  }

  stage "consumer driven contract test"
  node {
    git url: "http://disruptor.ninja:30130/rob/battleapp-cdct"
    def mvnHome = tool 'M3'
    sh "${mvnHome}/bin/mvn clean install failsafe:integration-test failsafe:verify"
    step([$class: 'JUnitResultArchiver', testResults: '**/target/failsafe-reports/TEST-*.xml'])
  }

  stage "manual testing"
  input "everything ok?"

  stage "start canary"
  input "deploy the canary?"
  node {
    git url: "http://disruptor.ninja:30130/rob/battleapp-canary"
    sh "./start.js"
  }

  stage "go full production"
  input "undeploy other versions?"
  node {
    git url: "http://disruptor.ninja:30130/rob/battleapp-prod"
    sh "./start.js"
  }
}
```

