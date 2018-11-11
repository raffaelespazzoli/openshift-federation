# Install Federation v2


# Test Federation v2

In this example we assume we have federated two clusters: raffa1 and raffa2.
We also assume that teh control plane is deployed in raffa1 and that raffa1's context is available in the variable $CLUSTER1

create a test namespace in raffa1

```
oc --context $CLUSER1 new-project federation-test
```

create the federated resources with their placement specification
```
oc --context $CLUSER1 apply -f example/federated-resources.yaml -n federation-test
```