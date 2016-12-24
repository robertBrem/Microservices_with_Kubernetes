# Add a system test
Create a system test similar to [this one](https://github.com/robertBrem/BattleApp-ST).  

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