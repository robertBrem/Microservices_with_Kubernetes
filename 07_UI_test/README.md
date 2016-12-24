# UI Test
Create a UI test similar to [this one](https://github.com/robertBrem/BattleApp-UIT).

Include the test in the Jenkins pipeline:
```
  stage "ui test"
  node {
    git url: "https://github.com/robertBrem/BattleApp-UIT"
    def mvnHome = tool 'M3'
    sh "${mvnHome}/bin/mvn clean install failsafe:integration-test failsafe:verify"
    step([$class: 'JUnitResultArchiver', testResults: '**/target/failsafe-reports/TEST-*.xml'])
  }
```

Then `Build Now`.