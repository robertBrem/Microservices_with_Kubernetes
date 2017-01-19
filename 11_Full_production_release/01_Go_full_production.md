# Go full production with the new version
After the tests on production we're ready to go full production with the new version.

## Create JavaScript
Like the canary release we start the full production step with JavaScript.
```
#!/usr/bin/jjs -fv

var version = $ENV.VERSION;
var kubectl = $ENV.KUBECTL;

var name = "battleapp";
var url = "http://disruptor.ninja:30080/battleapp/resources/users";
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

That the script can be executed it has to be executable:
```
chmod 750 start.js
```

## Add the full production release as CI step
This script has to be added to the Jenkins pipeline:
```
  stage "go full production"
  input "undeploy other versions?"
  node {
    git url: "https://github.com/robertBrem/BattleApp-Prod"
    sh "./start.js"
  }
```

Then `Build Now`.