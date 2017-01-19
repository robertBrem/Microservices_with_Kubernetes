# Add a system test
Create a system test similar to [this one](https://github.com/robertBrem/BattleApp-ST).  

Therefore we need the following dependencies:
```
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.12</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>com.airhacks.rulz</groupId>
    <artifactId>jaxrsclient</artifactId>
    <version>0.0.1</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.glassfish.jersey.core</groupId>
    <artifactId>jersey-client</artifactId>
    <version>2.12</version>
    <scope>test</scope>
</dependency>
```

And the following test class:

```
public class BattleAppIT {

    @Rule
    public JAXRSClientProvider provider = buildWithURI("http://" + System.getenv("HOST") + ":" + System.getenv("PORT") + "/battleapp/resources/users");

    @Test
    public void shouldReturn200() throws IOException {
        Response response = provider
                .target()
                .request()
                .get();
        assertThat(response.getStatus(), is(200));
    }
    
}
```

The test can locally be executed with the following command:

```
HOST=localhost PORT=8080 mvn clean install failsafe:integration-test failsafe:verify
```

Include the test in the Jenkins pipeline:
```
  stage "system test"
  node {
    git url: "https://github.com/robertBrem/BattleApp-ST"
    def mvnHome = tool 'M3'
    sh "PORT=31080 ${mvnHome}/bin/mvn clean install failsafe:integration-test failsafe:verify"
    step([$class: 'JUnitResultArchiver', testResults: '**/target/failsafe-reports/TEST-*.xml'])
  }
```

Then `Build Now`.