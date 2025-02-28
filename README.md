# Kafka Load Test

## Pre-requisites

Install the Red Hat Streams for Apache Kafka. This can be done within your target namespace(s) (ie, 'streams''), or globally across all namespaces.

Install the Prometheus operator. This can be done within your target namespace (ie, 'streams'), or globally across all namespaces.

Install the Grafana operator. This can be done within your target namespace (ie, 'streams'), or globally across all namespaces.

## Set up Cluster

__Environment__

```
#
# Set your env variables.
export PROJECT_ROOT="$(pwd)"
export PATH="${PROJECT_ROOT}/bin:${PATH}"
```

__Prometheus/Grafana__

```
#
# Create the metrics project. Do this as a regular user.
cd "${PROJECT_ROOT}/prometheus"
oc new-project streams-metrics

#
# Create/configure the Prometheus server. These steps must be completed as cluster-admin.
cd "${PROJECT_ROOT}/prometheus"
oc -n streams-metrics apply -f ./prometheus-additional-scrape-secret.yaml
oc -n streams-metrics apply -f ./prometheus.yaml
oc -n streams-metrics apply -f ./prometheus-strimzi-pod-monitor.yaml
# End cluster-admin steps.


#
# Create/configure the Grafana server.
cd "${PROJECT_ROOT}/grafana"
oc -n streams-metrics apply -f ./grafana.yaml
oc -n streams-metrics expose service grafana-service
oc -n streams-metrics apply -f ./grafana-datasource.yaml
oc -n streams-metrics apply -f './grafana-*-dashboard.yaml'
```

__Red Hat Streams for Apache Kafka__

```
cd "${PROJECT_ROOT}/kafka"
oc new-project streams
oc -n streams apply -f ./kafka-metrics-configmap.yaml
oc -n streams apply -f ./kafka-cluster.yaml
oc -n streams apply -f ./kafka-topics.yaml
```

To scale the cluster vertically, simply change the `resources` settings in the `kafka.yaml` and reapply. The cluster will roll-restart. To scale the cluster horizontally, it's easiest to delete the existing cluster and recreate using the commands below. That way we don't have to worry about migrating data.

```
cd "${PROJECT_ROOT}/kafka"
oc -n streams delete Kafka my-cluster
oc -n streams delete PersistentVolumeClaim -l 'strimzi.io/cluster=my-cluster'
# Edit the ./kafka-cluster.yaml file with your desired replica count.
oc -n streams apply -f ./kafka-cluster.yaml
```

## Generate some load

```
#
# Create the client project.
cd "${PROJECT_ROOT}/kafka"
oc new-project streams-client


#
# Create/configure the kafka-producer-perf-test.sh deployment.
oc -n streams-client apply -f ./kafka-producer-perf-test-configmap.yaml
oc -n streams-client apply -f ./kafka-producer-perf-test.yaml

#
# Scale the kafka-producer-perf-test.
oc -n streams-client scale deployment kafka-producer-perf-test --replicas=20


#
# Create/configure the kafka-consumer-perf-test.sh deployment.
oc -n streams-client apply -f ./kafka-consumer-perf-test-configmap.yaml
oc -n streams-client apply -f ./kafka-consumer-perf-test.yaml

#
# Scale the kafka-producer-perf-test.
oc -n streams-client scale deployment kafka-consumer-perf-test --replicas=20
```
