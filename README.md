# kafkasource-keda-autoscaler

Sample on Keda integration with the Knative Eventing Source for Apache Kafka

## Prerequisite

* [KEDA](https://keda.sh/docs/2.4/deploy/#yaml)
* [Knative Eventing Keda Autoscaler](https://github.com/knative-sandbox/eventing-autoscaler-keda)

## Apache Kafka Topics

The `KafkaSource` is reading from two topics (`my-topic` and `my-topic2`), created via [Strimzi Operator]():

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: my-topic
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 10
  replicas: 3
  config:
    retention.bytes: 53687091200
    retention.ms: 36000000
```

## The load

For generating some load, a batch like below can be used:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  labels:
    app: kafka-producer-job-notls-noacks
  name: kafka-producer-job-notls-noacks
  namespace: kafka
spec:
  parallelism: 5
  completions: 5
  backoffLimit: 1
  template:
    metadata:
      name: kafka-perf-producer
      labels:
        app: kafka-perf-producer
    spec:
      restartPolicy: Never
      containers:
      - name: kafka-perf-producer
        image: quay.io/strimzi/kafka:0.24.0-kafka-2.7.1
        command: [ "bin/kafka-producer-perf-test.sh" ]
        args: [ "--topic", "my-topic", "--throughput", "10000000", "--num-records", "1000000", "--producer-props", "bootstrap.servers=my-cluster-kafka-bootstrap:9092", "--record-size", "1000" ]
        volumeMounts:
        - name: strimzi-ca
          readOnly: true
          mountPath: "/etc/strimzi"
        env:
        - name: CA_PASSWORD
          valueFrom:
            secretKeyRef:
              name: my-cluster-cluster-ca-cert
              key: ca.password
      volumes:
      - name: strimzi-ca
        secret:
          secretName: my-cluster-cluster-ca-cert
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - kafka-perf-producer
              topologyKey: kubernetes.io/hostname
```

## Scaled consumers

Different objects are scaled via KEDA in this sample

* Eventing Source for Apache Kafka
* Webservers deployed as Knative Serving Services

### Scaled KafkaSource

The sources are using various annotations to define the desired scaling behavior from Knative and Keda:

```yaml
  annotations:
    autoscaling.knative.dev/class: keda.autoscaling.knative.dev
    autoscaling.knative.dev/maxScale: "4"
    autoscaling.knative.dev/minScale: "0"
    autoscaling.knative.dev/target: "1"
    autoscaling.knative.dev/targetUtilizationPercentage: "30"
    keda.autoscaling.knative.dev/pollingInterval: "10"
    keda.autoscaling.knative.dev/cooldownPeriod: "10"
    keda.autoscaling.knative.dev/kafkaLagThreshold: "10"
```

With the load from the above `Job`, we are getting four pods for each source:

```bash
kubectl get pods -l eventing.knative.dev/source

NAME                                                              READY   STATUS    RESTARTS   AGE
kafkasource-kafka-source-2111bdbf-4ba0-4dd8-bdf3-d1b173da15n4dz   1/1     Running   0          33m
kafkasource-kafka-source-2111bdbf-4ba0-4dd8-bdf3-d1b173da19w6h5   1/1     Running   0          33m
kafkasource-kafka-source-2111bdbf-4ba0-4dd8-bdf3-d1b173da1qrglw   1/1     Running   0          34m
kafkasource-kafka-source-2111bdbf-4ba0-4dd8-bdf3-d1b173da1r2l67   1/1     Running   0          33m
kafkasource-kafka-source2-1c2c016f-6aef-4680-bec1-596f13e14wk4b   1/1     Running   0          33m
kafkasource-kafka-source2-1c2c016f-6aef-4680-bec1-596f13e16vl9c   1/1     Running   0          33m
kafkasource-kafka-source2-1c2c016f-6aef-4680-bec1-596f13e1b5b4q   1/1     Running   0          33m
kafkasource-kafka-source2-1c2c016f-6aef-4680-bec1-596f13e1r8txj   1/1     Running   0          33m
```

### Scaled Knative Serving applications

For the receivers of the CloudEvents, comming from the `KafkaSource`s, we are also getting four pods: 

```
kubectl get pods
NAME                                                              READY   STATUS    RESTARTS   AGE
event-display-00002-deployment-c9d5b9647-852d6                    2/2     Running   0          26m
event-display-00002-deployment-c9d5b9647-mhdqx                    2/2     Running   0          26m
event-display-00002-deployment-c9d5b9647-pkzcq                    2/2     Running   0          26m
event-display-00002-deployment-c9d5b9647-rl9x9                    2/2     Running   0          26m
http-to-kafka-processor1-00001-deployment-75fc6cd5c8-qzfpz        2/2     Running   0          34m
http-to-kafka-processor1-00001-deployment-75fc6cd5c8-sk4lc        2/2     Running   0          34m
http-to-kafka-processor1-00001-deployment-75fc6cd5c8-stbvk        2/2     Running   0          34m
http-to-kafka-processor1-00001-deployment-75fc6cd5c8-v4n4z        2/2     Running   0          34m
```

Our consumers here are processors, that receive a CloudEvent and put it back to a configurable topic in Apache Kafka.



_If using a ksvc that has a very short request execution time (like the canonical event-display image (`quay.io/openshift-knative/knative-eventing-sources-event-display`)), we are only getting one pod for that Knative Serving Service:_

```bash
NAME                                                              READY   STATUS    RESTARTS   AGE
event-display-00001-deployment-69cb5ccfd5-6rtvp                   2/2     Running   0          5m15s
http-to-kafka-processor1-00001-deployment-75fc6cd5c8-qzfpz        2/2     Running   0          5m54s
http-to-kafka-processor1-00001-deployment-75fc6cd5c8-sk4lc        2/2     Running   0          5m54s
http-to-kafka-processor1-00001-deployment-75fc6cd5c8-stbvk        2/2     Running   0          5m54s
http-to-kafka-processor1-00001-deployment-75fc6cd5c8-v4n4z        2/2     Running   0          5m54s
```
