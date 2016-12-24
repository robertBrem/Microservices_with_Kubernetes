# Create a canary release
To test the new version in production for example with A/B testing we need the possibility
to make a canary release.

## Create JavaScript
Like the start of the test environment we start the canary with JavaScript.
```
#!/usr/bin/jjs -fv

var FileWriter = Java.type("java.io.FileWriter");

var version = $ENV.VERSION;
var kubectl = $ENV.KUBECTL;

var name = "battleapp";
var nameWithVersion = name + "-" + version;
var image = "disruptor.ninja:30500/robertbrem/battleapp:" + version;
var replicas = 1;
var port = 8080;
var clusterPort = 8880;
var nodePort = 30080;
var deploymentFileName = "deployment.yml";
var serviceFileName = "service.yml";
var registrysecret = "registrykey";
var url = "http://disruptor.ninja:" + nodePort + "/battleapp/resources/users";
var timeout = 2;

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

That the script can be executed it has to be executable:
```
chmod 750 start.js
```

## Add the canary release as CI step
This script has to be added to the Jenkins pipeline:
```
  stage "start canary"
  input "deploy the canary?"
  node {
    git url: "https://github.com/robertBrem/BattleApp-Canary"
    sh "./start.js"
  }
```

Then `Build Now`.

## Test the canary release
After the first canary release we can check our cluster. It should look something like
this:
```
kc get deployment
```
```
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
battleapp-1.0.17   1         1         1            1           10m
battleapp-test     1         1         1            1           12m
jenkins            1         1         1            1           15h
registry           1         1         1            1           20h
```

```
kc get pod
```
```
NAME                                READY     STATUS    RESTARTS   AGE
battleapp-1.0.17-3230326404-2xvxb   1/1       Running   0          9m
battleapp-test-1930419673-78jpf     1/1       Running   0          13m
jenkins-3074977187-rcg1v            1/1       Running   0          15h
registry-95525520-9rdvc             1/1       Running   0          20h
```

An impressive way to test a canary release is to change the REST service and push the
change.

Then you can open a terminal and execute this script to see the change:
```
while true; do curl http://disruptor.ninja:30080/battleapp/resources/users; echo ""; sleep 1; done
```

After the canary release of the changed service the output look something like this:
```
[{"name":"Robert"},{"name":"Kevin"},{"name":"Dan"}]
[{"name":"dan"},{"name":"robert"},{"name":"kevin"}]
[{"name":"Robert"},{"name":"Kevin"},{"name":"Dan"}]
[{"name":"dan"},{"name":"robert"},{"name":"kevin"}]
[{"name":"dan"},{"name":"robert"},{"name":"kevin"}]
[{"name":"dan"},{"name":"robert"},{"name":"kevin"}]
[{"name":"Robert"},{"name":"Kevin"},{"name":"Dan"}]
[{"name":"Robert"},{"name":"Kevin"},{"name":"Dan"}]
[{"name":"Robert"},{"name":"Kevin"},{"name":"Dan"}]
```

The cluster is now looking something like this:
```
kc get pod
```
```
NAME                                READY     STATUS    RESTARTS   AGE
battleapp-1.0.17-3230326404-2xvxb   1/1       Running   0          19m
battleapp-1.0.18-3418480262-5n1d7   1/1       Running   0          2m
battleapp-test-2013584858-kq2bg     1/1       Running   0          4m
jenkins-3074977187-rcg1v            1/1       Running   0          15h
registry-95525520-9rdvc             1/1       Running   0          20h
```

## Create a readiness probe
It can happen that the Kubernetes service is routing requests to the new service
even if this service is not ready yet and we get `404` for a short period. To
suppress this behavior we can create a `readiness probe` for our canary deployment:

```
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
dfw.write("      imagePullSecrets:\n");
dfw.write("      - name: " + registrysecret + "\n");
dfw.close();
```

Now there shouldn't be any `404` errors.