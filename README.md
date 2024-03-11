# Zookeeper to Kraft with CFK

## Requirements

Check and review the requirements [here](https://docs.confluent.io/operator/current/co-migrate-kraft.html#requirements-and-considerations).


## Configure the ZK-based "existing" deployment


Create the 'confluent' namespace
```
kubectl create namespace confluent
``` 


Set the 'confluent' namespace as the context
```
kubectl config set-context --current --namespace confluent
```


Add Helm repo
```
helm repo add confluentinc https://packages.confluent.io/helm
```


Install the operator with kRaftEnabled flag. This could be done in a second phase, since it overwrites
```
helm upgrade --install confluent-operator \
  confluentinc/confluent-for-kubernetes \
  --namespace confluent
```


Deploy the platform with Zookeeper to simulate an existing state
```
kubectl apply -f 01-confluent-platform-zk.yaml
```


## Run the migration

We must force a repo refresh!
```
helm repo update
```

Update the operator with kRaftEnabled flag
```
helm upgrade --install confluent-operator \
  confluentinc/confluent-for-kubernetes \
  --set kRaftEnabled=true \
  --set debug=true \
  --namespace confluent
```

Enable metadata migration tracing in Kafka brokers, with snippet like the following snippet
```
apiVersion: platform.confluent.io/v1beta1
kind: Kafka
spec:
  ...
  configOverrides:
    log4j:
      - log4j.logger.org.apache.kafka.metadata.migration=TRACE
```

```
kubectl apply -f 02-kafka-with-migration-logger.yaml
```


Create the Kraft controllers CRs
```
kubectl apply -f 03-kraft-controllers.yaml
```


Launch the migration job
```
kubectl apply -f 04-kraft-migration-job
```

Monitor the migration process
```
kubectl get kraftmigrationjob kraftmigrationjob -n confluent -oyaml -w
```


Once it's running in dual write mode, and everything looks good, let's finalize
```
kubectl annotate kraftmigrationjob kraftmigrationjob \
  platform.confluent.io/kraft-migration-trigger-finalize-to-kraft=true \
  --namespace confluent
```

Remove the lock that the migration job placed on the resources.
```
kubectl annotate kraftmigrationjob kraftmigrationjob \
  platform.confluent.io/kraft-migration-release-cr-lock=true \
  --namespace confluent
```

Remove the Zookeeper nodes
```
kubectl delete -f 05-remove-zookeeper.yaml
```

## Resources

- https://docs.confluent.io/operator/current/co-migrate-kraft.html
