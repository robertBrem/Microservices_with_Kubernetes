# Create the fileuploader service
To upload files we've to create a fileupload and download service.  

## Fileuploader service implementation
The creation of the health endpoint, `keycloak.json`, `web.xml`, `Dockerfile`,
`build.js` and the `pom.xml` is straight forward like in the other projects.  
The servlet itself looks like this:

```
import javax.json.Json;
import javax.json.JsonArrayBuilder;
import javax.json.JsonObject;
import javax.servlet.ServletException;
import javax.servlet.annotation.MultipartConfig;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.Part;
import java.io.File;
import java.io.IOException;
import java.io.PrintWriter;
import java.util.HashSet;
import java.util.Set;
import java.util.UUID;

@WebServlet("/fileupload")
@MultipartConfig(
        fileSizeThreshold = 1024 * 1024 * 10,  // 10 MB
        maxFileSize = 1024 * 1024 * 50,        // 50 MB
        maxRequestSize = 1024 * 1024 * 100)    // 100 MB
public class FileUploadServlet extends HttpServlet {
    private static final long serialVersionUID = -1L;

    public static final String UPLOAD_DIR = System.getenv("UPLOAD_DIR");

    protected void doPost(HttpServletRequest request,
                          HttpServletResponse response) throws ServletException, IOException {

        Set<String> paths = new HashSet<>();
        for (Part part : request.getParts()) {
            String relativePath = part.getName();
            String fileName = UUID.randomUUID().toString();
            String oldFileName = getFileName(part);
            String extension = oldFileName.split("\\.")[1];
            String relativeFilePath = relativePath + File.separator + fileName + "." + extension;
            String path = UPLOAD_DIR + relativeFilePath;
            File fileSaveDir = new File(path);
            if (!fileSaveDir.getParentFile().exists()) {
                fileSaveDir.getParentFile().mkdirs();
            }
            part.write(path);

            fileSaveDir.setExecutable(false, false);
            fileSaveDir.setWritable(false, false);
            fileSaveDir.setReadable(true, false);

            paths.add(relativeFilePath);
        }

        JsonArrayBuilder pictureBuilder = Json.createArrayBuilder();
        for (String path : paths) {
            JsonObject pathObject = Json.createObjectBuilder()
                    .add("path", path)
                    .build();
            pictureBuilder = pictureBuilder
                    .add(pathObject);
        }

        response.setHeader("Location", paths.toString());
        response.setContentType("application/json");
        PrintWriter out = response.getWriter();
        out.print(pictureBuilder.build().toString());
        out.flush();
        out.close();
    }

    private String getFileName(Part part) {
        String contentDisp = part.getHeader("content-disposition");
        String[] tokens = contentDisp.split(";");
        for (String token : tokens) {
            if (token.trim().startsWith("filename")) {
                return token.substring(token.indexOf("=") + 2, token.length() - 1);
            }
        }
        return "";
    }

}
```

## Jenkins pipeline
The Jenkins pipeline looks like this:

```
withEnv([   "VERSION=1.0.${currentBuild.number}",
            "REGISTRY_EMAIL=brem_robert@hotmail.com",
            "KUBECTL=kubectl",
            "HOST=disruptor.ninja",
            "REALM_NAME=battleapp",
            "KEYCLOAK_URL=https://disruptor.ninja:31182/auth",
            "PORT=31082"]) {

  stage "checkout, build, test and publish"
  node {
    git url: "http://disruptor.ninja:30130/rob/battleapp-fileuploader"
    def mvnHome = tool 'M3'
    sh "${mvnHome}/bin/mvn clean install"
    sh "./build.js"
  }

  stage "start test environment"
  node {
    git url: "http://disruptor.ninja:30130/rob/battleapp-fileuploader-starttestenv"
    sh "./start.js"
  }

  stage "manual testing"
  input "everything ok?"

  stage "start canary"
  input "deploy the canary?"
  node {
    git url: "http://disruptor.ninja:30130/rob/battleapp-fileuploader-canary"
    sh "./start.js"
  }

  stage "go full production"
  input "undeploy other versions?"
  node {
    git url: "http://disruptor.ninja:30130/rob/battleapp-fileuploader-prod"
    sh "./start.js"
  }
}
```

## Start test environment script
The `start.js` script for the test environment looks like this:

> We create two deployments and two services one for the fileupload a
> Java EE application and for the filedownload a simple nginx.

```
#!/usr/bin/jjs -fv

var FileWriter = Java.type("java.io.FileWriter");

var version = $ENV.VERSION;
var kubectl = $ENV.KUBECTL;

var name = "battleapp-fileuploader";
var image = "disruptor.ninja:30500/robertbrem/battleapp-fileuploader:" + version;
var replicas = 1;
var port = 8080;
var clusterPort = 8181;
var nodePort = 31082;
var deploymentFileName = "deployment.yml";
var serviceFileName = "service.yml";
var registrysecret = "registrykey";
var url = "http://disruptor.ninja:" + nodePort + "/battleapp/resources/health";
var timeout = 2;
var realmName = "battleapp";
var authServerUrl = "https://disruptor.ninja:31182/auth";
var namespace = "test";
var kafkaAddress = "kafka:9092";
var uploadDir = "/opt/jboss/uploads";
var nodeSelector = "vmi74389";
var hostPath = "/root/uploads";

var downloadName = "battleapp-filedownloader";
var downloadImage = "nginx";
var downloadReplicas = 1;
var downloadPort = 80;
var downloadClusterPort = 81;
var downloadNodePort = 31031;
var downloadDeploymentFileName = "downloadDeployment.yml";
var downloadServiceFileName = "downloadService.yml";
var downloadDir = "/usr/share/nginx/html";

var deleteDeployment = kubectl + " --namespace " + namespace + " delete deployment " + name;
execute(deleteDeployment);

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
dfw.write("        - name: REALM_NAME\n");
dfw.write("          value: \"" + realmName + "\"\n");
dfw.write("        - name: AUTH_SERVER_URL\n");
dfw.write("          value: \"" + authServerUrl + "\"\n");
dfw.write("        - name: KAFKA_ADDRESS\n");
dfw.write("          value: \"" + kafkaAddress + "\"\n");
dfw.write("        - name: UPLOAD_DIR\n");
dfw.write("          value: \"" + uploadDir + "\"\n");
dfw.write("        volumeMounts:\n");
dfw.write("        - mountPath: " + uploadDir + "\n");
dfw.write("          name: files\n");
dfw.write("      volumes:\n");
dfw.write("      - name: files\n");
dfw.write("        hostPath:\n");
dfw.write("          path: " + hostPath + "\n");
dfw.write("      nodeSelector:\n");
dfw.write("        name: " + nodeSelector + "\n");
dfw.write("      imagePullSecrets:\n");
dfw.write("      - name: " + registrysecret + "\n");
dfw.close();

var deploy = kubectl + " create -f " + deploymentFileName;
execute(deploy);

var deleteDownloadDeployment = kubectl + " --namespace " + namespace + " delete deployment " + downloadName;
execute(deleteDownloadDeployment);

var downfw = new FileWriter(downloadDeploymentFileName);
downfw.write("apiVersion: extensions/v1beta1\n");
downfw.write("kind: Deployment\n");
downfw.write("metadata:\n");
downfw.write("  name: " + downloadName + "\n");
downfw.write("  namespace: " + namespace + "\n");
downfw.write("spec:\n");
downfw.write("  replicas: " + downloadReplicas + "\n");
downfw.write("  template:\n");
downfw.write("    metadata:\n");
downfw.write("      labels:\n");
downfw.write("        name: " + downloadName + "\n");
downfw.write("    spec:\n");
downfw.write("      containers:\n");
downfw.write("      - resources:\n");
downfw.write("        name: " + downloadName + "\n");
downfw.write("        image: " + downloadImage + "\n");
downfw.write("        ports:\n");
downfw.write("        - name: port\n");
downfw.write("          containerPort: " + downloadPort + "\n");
downfw.write("        volumeMounts:\n");
downfw.write("        - mountPath: " + downloadDir + "\n");
downfw.write("          name: files\n");
downfw.write("      volumes:\n");
downfw.write("      - name: files\n");
downfw.write("        hostPath:\n");
downfw.write("          path: " + hostPath + "\n");
downfw.write("      nodeSelector:\n");
downfw.write("        name: " + nodeSelector + "\n");
downfw.write("      imagePullSecrets:\n");
downfw.write("      - name: " + registrysecret + "\n");
downfw.close();

var downloadDeploy = kubectl + " create -f " + downloadDeploymentFileName;
execute(downloadDeploy);

var sfw = new FileWriter(serviceFileName);
sfw.write("apiVersion: v1\n");
sfw.write("kind: Service\n");
sfw.write("metadata:\n");
sfw.write("  name: " + name + "\n");
sfw.write("  labels:\n");
sfw.write("    name: " + name + "\n");
sfw.write("  namespace: " + namespace + "\n");
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

var downsfw = new FileWriter(downloadServiceFileName);
downsfw.write("apiVersion: v1\n");
downsfw.write("kind: Service\n");
downsfw.write("metadata:\n");
downsfw.write("  name: " + downloadName + "\n");
downsfw.write("  labels:\n");
downsfw.write("    name: " + downloadName + "\n");
downsfw.write("  namespace: " + namespace + "\n");
downsfw.write("spec:\n");
downsfw.write("  ports:\n");
downsfw.write("  - port: " + downloadClusterPort + "\n");
downsfw.write("    targetPort: " + downloadPort + "\n");
downsfw.write("    nodePort: " + downloadNodePort + "\n");
downsfw.write("  selector:\n");
downsfw.write("    name: " + downloadName + "\n");
downsfw.write("  type: NodePort\n");
downsfw.close();

var downloadDeployService = kubectl + " create -f " + downloadServiceFileName;
execute(downloadDeployService);

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

## Start canary script
Like in the test environment we create two deployments and two services one for
the download and one for the upload:

```
#!/usr/bin/jjs -fv

var FileWriter = Java.type("java.io.FileWriter");

var version = $ENV.VERSION;
var kubectl = $ENV.KUBECTL;

var name = "battleapp-fileuploader";
var versionedName = name + "-" + version;
var image = "disruptor.ninja:30500/robertbrem/battleapp-fileuploader:" + version;
var replicas = 1;
var port = 8080;
var clusterPort = 8182;
var nodePort = 30082;
var deploymentFileName = "deployment.yml";
var serviceFileName = "service.yml";
var registrysecret = "registrykey";
var url = "http://disruptor.ninja:" + nodePort + "/battleapp/resources/health";
var timeout = 2;
var realmName = "battleapp";
var authServerUrl = "https://disruptor.ninja:30182/auth";
var kafkaAddress = "kafka:9092";
var uploadDir = "/opt/jboss/uploads";
var nodeSelector = "vmi100202";
var hostPath = "/root/uploads";

var downloadName = "battleapp-filedownloader";
var versionedDownloadName = downloadName + "-" + version;
var downloadImage = "nginx";
var downloadReplicas = 1;
var downloadPort = 80;
var downloadClusterPort = 82;
var downloadNodePort = 30031;
var downloadDeploymentFileName = "downloadDeployment.yml";
var downloadServiceFileName = "downloadService.yml";
var downloadDir = "/usr/share/nginx/html";

var dfw = new FileWriter(deploymentFileName);
dfw.write("apiVersion: extensions/v1beta1\n");
dfw.write("kind: Deployment\n");
dfw.write("metadata:\n");
dfw.write("  name: " + versionedName + "\n");
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
dfw.write("        env:\n");
dfw.write("        - name: REALM_NAME\n");
dfw.write("          value: \"" + realmName + "\"\n");
dfw.write("        - name: AUTH_SERVER_URL\n");
dfw.write("          value: \"" + authServerUrl + "\"\n");
dfw.write("        - name: KAFKA_ADDRESS\n");
dfw.write("          value: \"" + kafkaAddress + "\"\n");
dfw.write("        - name: UPLOAD_DIR\n");
dfw.write("          value: \"" + uploadDir + "\"\n");
dfw.write("        volumeMounts:\n");
dfw.write("        - mountPath: " + uploadDir + "\n");
dfw.write("          name: files\n");
dfw.write("      volumes:\n");
dfw.write("      - name: files\n");
dfw.write("        hostPath:\n");
dfw.write("          path: " + hostPath + "\n");
dfw.write("      nodeSelector:\n");
dfw.write("        name: " + nodeSelector + "\n");
dfw.write("      imagePullSecrets:\n");
dfw.write("      - name: " + registrysecret + "\n");
dfw.close();

var deploy = kubectl + " create -f " + deploymentFileName;
execute(deploy);

var downfw = new FileWriter(downloadDeploymentFileName);
downfw.write("apiVersion: extensions/v1beta1\n");
downfw.write("kind: Deployment\n");
downfw.write("metadata:\n");
downfw.write("  name: " + versionedDownloadName + "\n");
downfw.write("spec:\n");
downfw.write("  replicas: " + downloadReplicas + "\n");
downfw.write("  template:\n");
downfw.write("    metadata:\n");
downfw.write("      labels:\n");
downfw.write("        name: " + downloadName + "\n");
downfw.write("        version: " + version + "\n");
downfw.write("    spec:\n");
downfw.write("      containers:\n");
downfw.write("      - resources:\n");
downfw.write("        name: " + downloadName + "\n");
downfw.write("        image: " + downloadImage + "\n");
downfw.write("        ports:\n");
downfw.write("        - name: port\n");
downfw.write("          containerPort: " + downloadPort + "\n");
downfw.write("        volumeMounts:\n");
downfw.write("        - mountPath: " + downloadDir + "\n");
downfw.write("          name: files\n");
downfw.write("      volumes:\n");
downfw.write("      - name: files\n");
downfw.write("        hostPath:\n");
downfw.write("          path: " + hostPath + "\n");
downfw.write("      nodeSelector:\n");
downfw.write("        name: " + nodeSelector + "\n");
downfw.write("      imagePullSecrets:\n");
downfw.write("      - name: " + registrysecret + "\n");
downfw.close();

var downloadDeploy = kubectl + " create -f " + downloadDeploymentFileName;
execute(downloadDeploy);

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

var downsfw = new FileWriter(downloadServiceFileName);
downsfw.write("apiVersion: v1\n");
downsfw.write("kind: Service\n");
downsfw.write("metadata:\n");
downsfw.write("  name: " + downloadName + "\n");
downsfw.write("  labels:\n");
downsfw.write("    name: " + downloadName + "\n");
downsfw.write("spec:\n");
downsfw.write("  ports:\n");
downsfw.write("  - port: " + downloadClusterPort + "\n");
downsfw.write("    targetPort: " + downloadPort + "\n");
downsfw.write("    nodePort: " + downloadNodePort + "\n");
downsfw.write("  selector:\n");
downsfw.write("    name: " + downloadName + "\n");
downsfw.write("  type: NodePort\n");
downsfw.close();

var downloadDeployService = kubectl + " create -f " + downloadServiceFileName;
execute(downloadDeployService);

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

## Go full production script
The go full production with the current version script looks like this:

```
#!/usr/bin/jjs -fv

var version = $ENV.VERSION;
var kubectl = $ENV.KUBECTL;

var name = "battleapp-fileuploader";
var url = "http://disruptor.ninja:30082/battleapp/resources/health";
var timeout = 2;

var downloadName = "battleapp-filedownloader";

var deleteDeployment = kubectl + " delete deployment -l name=" + name + ",version!=" + version;
execute(deleteDeployment);

var deleteDownloadDeployment = kubectl + " delete deployment -l name=" + downloadName + ",version!=" + version;
execute(deleteDownloadDeployment);

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

