# Create manual test
Before a version is going into production the version is normally tested manually.
We can create a manual testing step in our Jenkins pipeline as well:
```
  stage "manual testing"
  input "everything ok?"
```

Then `Build Now`.