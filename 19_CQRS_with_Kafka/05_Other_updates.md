# Other updates

## Updates to the integration and  consumer driven contract tests
The system tests in the command pipeline should only test the create 
methods. Therefore we've to delete all the test of the command side.
And create integration tests for the command side in the command
pipeline.

## Updates to the start scripts
We've to add the `KAFKA_ADDRESS` to the test environment start and
the canary script:

```
...
var kafkaAddress = "kafka:9092";
...
dfw.write("        - name: KAFKA_ADDRESS\n");
dfw.write("          value: \"" + kafkaAddress + "\"\n");
...
```

