# Create a test and production stage

We create a test environment similar to the test environment of the REST service.

```
#!/usr/bin/jjs -fv

var FileWriter = Java.type("java.io.FileWriter");

var version = $ENV.VERSION;
var kubectl = $ENV.KUBECTL;

var name = "battleapp-frontend-test";
var baseUrl = "disruptor.ninja";
var image = baseUrl + ":30500/robertbrem/battleapp-frontend:" + version;
var replicas = 1;
var port = 80;
var clusterPort = 3001;
var nodePort = 31030;
var deploymentFileName = "deployment.yml";
var serviceFileName = "service.yml";
var registrysecret = "registrykey";
var url = "http://" + baseUrl + ":" + nodePort;
var timeout = 2;

var deleteDeployment = kubectl + " delete deployment " + name;
execute(deleteDeployment);

var dfw = new FileWriter(deploymentFileName);
dfw.write("apiVersion: extensions/v1beta1\n");
dfw.write("kind: Deployment\n");
dfw.write("metadata:\n");
dfw.write("  name: " + name + "\n");
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
dfw.write("        volumeMounts:\n");
dfw.write("        - name: environment\n");
dfw.write("          mountPath: /usr/share/nginx/html/environment\n");
dfw.write("      volumes:\n");
dfw.write("      - name: environment\n");
dfw.write("        configMap:\n");
dfw.write("          name: battleapp-frontend-test\n");
dfw.write("          items:\n");
dfw.write("          - key: environment.json\n");
dfw.write("            path: environment.json\n");
dfw.write("      imagePullSecrets:\n");
dfw.write("      - name: " + registrysecret + "\n");
dfw.close();

var deploy = kubectl + " create -f " + deploymentFileName;
execute(deploy);

var deleteService = kubectl + " delete service " + name;
execute(deleteService);

var sfw = new FileWriter(serviceFileName);
sfw.write("apiVersion: v1\n");
sfw.write("kind: Service\n");
sfw.write("metadata:\n");
sfw.write("  name: " + name + "\n");
sfw.write("  labels:\n");
sfw.write("    name: " + name + "\n");
sfw.write("spec:\n");
sfw.write("  ports:\n");
sfw.write("  - port: " + clusterPort + "\n");
sfw.write("    targetPort: " + port + "\n");
sfw.write("    nodePort: " + nodePort + "\n");
sfw.write("  selector:\n");
sfw.write("    name: " + name + "\n");
sfw.write("  type: NodePort\n");
sfw.close();

var deployService = kubectl + " create -f " + serviceFileName;
execute(deployService);

var testUrl = "curl --write-out %{http_code} --silent --output /dev/null " + url + " --max-time " + timeout;
execute(testUrl);
while ($OUT != "200") {
    $EXEC("sleep 1");
    execute(testUrl);
}

function execute(command) {
    $EXEC(command);
    print($OUT);
    print($ERR);
}
```

It is important to make this script file executable:
```
chmod 750 start.js
```

And the corresponding pipeline entry:
```
stage "start test environment"
node {
  git url: "http://disruptor.ninja:30130/rob/battleapp-frontend-starttestenv"
  sh "./start.js"
}
```

After the test environment we add a manual pipeline step:
```
stage "manual testing"
input "everything ok?"
```

The next step is the canary release:
```
#!/usr/bin/jjs -fv

var FileWriter = Java.type("java.io.FileWriter");

var version = $ENV.VERSION;
var kubectl = $ENV.KUBECTL;

var name = "battleapp-frontend";
var nameWithVersion = name + "-" + version;
var image = "disruptor.ninja:30500/robertbrem/" + name + ":" + version;
var replicas = 1;
var port = 80;
var clusterPort = 3002;
var nodePort = 30030;
var deploymentFileName = "deployment.yml";
var serviceFileName = "service.yml";
var registrysecret = "registrykey";
var relativeUrl = "/";
var url = "http://disruptor.ninja:" + nodePort;
var timeout = 2;
var initialDelay = 15;
var readinessProbeTimeout = 10;

var dfw = new FileWriter(deploymentFileName);
dfw.write("apiVersion: extensions/v1beta1\n");
dfw.write("kind: Deployment\n");
dfw.write("metadata:\n");
dfw.write("  name: " + nameWithVersion + "\n");
dfw.write("spec:\n");
dfw.write("  replicas: " + replicas + "\n");
dfw.write("  template:\n");
dfw.write("    metadata:\n");
dfw.write("      labels:\n");
dfw.write("        name: " + name + "\n");
dfw.write("        version: " + version + "\n");
dfw.write("    spec:\n");
dfw.write("      containers:\n");
dfw.write("      - resources:\n");
dfw.write("        name: " + name + "\n");
dfw.write("        image: " + image + "\n");
dfw.write("        ports:\n");
dfw.write("        - name: port\n");
dfw.write("          containerPort: " + port + "\n");
dfw.write("        readinessProbe:\n");
dfw.write("          httpGet:\n");
dfw.write("            path: " + relativeUrl + "\n");
dfw.write("            port: " + port + "\n");
dfw.write("          initialDelaySeconds: " + initialDelay + "\n");
dfw.write("          timeoutSeconds: " + readinessProbeTimeout + "\n");
dfw.write("        volumeMounts:\n");
dfw.write("        - name: environment\n");
dfw.write("          mountPath: /usr/share/nginx/html/environment\n");
dfw.write("      volumes:\n");
dfw.write("      - name: environment\n");
dfw.write("        configMap:\n");
dfw.write("          name: battleapp-frontend\n");
dfw.write("          items:\n");
dfw.write("          - key: environment.json\n");
dfw.write("            path: environment.json\n");
dfw.write("      imagePullSecrets:\n");
dfw.write("      - name: " + registrysecret + "\n");
dfw.close();

var deploy = kubectl + " create -f " + deploymentFileName;
execute(deploy);

var sfw = new FileWriter(serviceFileName);
sfw.write("apiVersion: v1\n");
sfw.write("kind: Service\n");
sfw.write("metadata:\n");
sfw.write("  name: " + name + "\n");
sfw.write("  labels:\n");
sfw.write("    name: " + name + "\n");
sfw.write("spec:\n");
sfw.write("  ports:\n");
sfw.write("  - port: " + clusterPort + "\n");
sfw.write("    targetPort: " + port + "\n");
sfw.write("    nodePort: " + nodePort + "\n");
sfw.write("  selector:\n");
sfw.write("    name: " + name + "\n");
sfw.write("  type: NodePort\n");
sfw.close();

var deployService = kubectl + " create -f " + serviceFileName;
execute(deployService);

var testUrl = "curl --write-out %{http_code} --silent --output /dev/null " + url + " --max-time " + timeout;
execute(testUrl);
while ($OUT != "200") {
    $EXEC("sleep 1");
    execute(testUrl);
}

function execute(command) {
    $EXEC(command);
    print($OUT);
    print($ERR);
}
```

Make the script file executable:
```
chmod 750 start.js
```

The corresponding pipeline entry:
```
stage "start canary"
input "deploy the canary?"
node {
  git url: "http://disruptor.ninja:30130/rob/battleapp-frontend-canary"
  sh "./start.js"
}
```

The last step is the production step with the following script:
```
#!/usr/bin/jjs -fv

var version = $ENV.VERSION;
var kubectl = $ENV.KUBECTL;

var name = "battleapp-frontend";
var url = "http://disruptor.ninja:30030";
var timeout = 2;

var deleteDeployment = kubectl + " delete deployment -l name=" + name + ",version!=" + version;
execute(deleteDeployment);

var testUrl = "curl --write-out %{http_code} --silent --output /dev/null " + url + " --max-time " + timeout;
execute(testUrl);
while ($OUT != "200") {
    $EXEC("sleep 1");
    execute(testUrl);
}

function execute(command) {
    $EXEC(command);
    print($OUT);
    print($ERR);
}
```

Make the script file executable:
```
chmod 750 start.js
```

The corresponding pipeline entry:
```
stage "go full production"
input "undeploy other versions?"
node {
  git url: "http://disruptor.ninja:30130/rob/battleapp-frontend-prod"
  sh "./start.js"
}
```

## Creation of Kubernetes ConfigMap
In both the test and the production stage we referenced a Kubernetes
ConfigMap that we've to create first. Therefore we have to create a folder
on the local machine with the name `battleapp-frontend` and a file 
`environment.json` with the following content:

```
{
  "host": "disruptor.ninja",
  "port": 30080
}
```

We need also a `environment.json` file for the test stage. Therefore we 
create a folder `battleapp-frontend-test` and a file `environment.json`
with the following content:

```
{
  "host": "disruptor.ninja",
  "port": 31080
}
```

Now we create the two ConfigMaps:
```
kc create configmap battleapp-frontend-test --from-file=battleapp-frontend-test
kc create configmap battleapp-frontend --from-file=battleapp-frontend
```

We can test display the ConfigMaps with the following command:
```
kc get configmap battleapp-frontend-test -o yaml
```
```
apiVersion: v1
data:
  environment.json: |
    {
      "host": "disruptor.ninja",
      "port": 31080
    }
kind: ConfigMap
metadata:
  creationTimestamp: 2016-12-31T15:54:44Z
  name: battleapp-frontend-test
  namespace: default
  resourceVersion: "1049486"
  selfLink: /api/v1/namespaces/default/configmaps/battleapp-frontend-test
  uid: 72a5399b-cf71-11e6-a836-0050563cad2a
```
