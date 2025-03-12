# How to Send and Receive Kafka Messages in [SonataFlow Workflows](https://sonataflow.org)

This guide demonstrates how to send and receive messages from a **Kafka Broker** using the [Quarkus SmallRye Kafka Connector](https://quarkus.io/version/3.8/guides/kafka#configuring-smallrye-kafka-connector) in a **SonataFlow workflow application**. This tutorial covers defining event-driven workflows, configuring Kafka topics, and running tests locally and in a Kubernetes cluster.

---

## 📌 Understanding the Event Definitions

The [workflow specification](callback-flow/src/main/resources/lock.sw.yaml) defines three key events that orchestrate workflow execution:

```yaml
events:
  - type: lock-event
    kind: consumed
    name: lock-event
    source: local
    correlation:
      - contextAttributeName: lockid
  - type: release-event
    kind: consumed
    name: release-event
    source: local
    correlation:
      - contextAttributeName: lockid
  - type: released-event
    kind: produced
    name: released-event
    correlation:
      - contextAttributeName: lockid
```

### 🔹 Event Breakdown
1. **`lock-event`** → Initiates the workflow execution by acquiring a lock.
2. **`release-event`** → Releases the lock, resuming the workflow execution.
3. **`released-event`** → Signals that the workflow execution has completed successfully.

SonataFlow enforces event-driven execution through [CloudEvents](https://cloudevents.io/). Below is an example of a valid **CloudEvent JSON message**:

```json
{
  "specversion": "1.0",
  "id": "db16ff44-5b0b-4abc-88f3-5a71378be171",
  "source": "http://dev.local",
  "type": "lock-event",
  "datacontenttype": "application/json",
  "time": "2025-03-07T15:04:32.327635-05:00",
  "lockid": "03471a81-310a-47f5-8db3-cceebc63961a",
  "data": {
    "name": "The Kraken",
    "id": "03471a81-310a-47f5-8db3-cceebc63961a"
  }
}
```

### 🔹 Correlation in SonataFlow
Each event definition includes `correlation.contextAttributeName`, which enables **event correlation** across workflow instances.

For example, if multiple workflows are waiting for the `release-event`, SonataFlow will match incoming events by their `lockid` attribute, ensuring that only the relevant workflow instance resumes execution.

Kafka topics are automatically mapped to event **types**. By default, topics follow these naming conventions:
- `lock-event`
- `release-event`
- `released-event`

To customize the Kafka topic name, override the default configuration in **`application.properties`**:
```properties
mp.messaging.incoming.lock-event.topic=MySuperFancyTopicName
```

---

## 🏗 Running the Workflow in a Kubernetes Cluster

Deploying the workflow in a Kubernetes cluster allows it to integrate seamlessly with other microservices and event-driven systems. Follow these steps to set up a Kafka instance, configure SonataFlow, and interact with the workflow in a Kubernetes environment.

### 🔹 Deploy Kafka in Kubernetes
Kafka serves as the messaging backbone for the workflow. Deploy a single-node Kafka instance using Bitnami Helm charts:

```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install kafka bitnami/kafka -f kafka/values.yaml
```

Once installed, verify that the Kafka broker is running:

```shell
kubectl get pods
```

Expected output:
```shell
NAME                           READY   STATUS    RESTARTS   AGE
kafka-controller-0             1/1     Running   0          5m
```

### 🔹 Deploy SonataFlowPlatform
SonataFlow requires a platform resource to manage workflow execution. Apply the following Kubernetes manifest to set up the **SonataFlowPlatform**:

```shell
kubectl apply -f manifests/00-sonataflow-platform.yaml
```

This resource defines necessary configurations, including Kafka integration details. Ensure it is running:

```shell
kubectl get sonataflowplatform
```

Expected output:
```shell
NAME                   STATUS    AGE
sonataflow-platform    Ready     2m
```

### 🔹 Deploy the Workflow Application
Deploy the workflow application to the Kubernetes cluster by applying the necessary configuration and workflow manifests:

```shell
kubectl apply -f manifests/01-configmap_lock-flow-props.yaml
kubectl apply -f manifests/02-sonataflow_lock-flow.yaml
```

These manifests define the workflow logic and configure event-based communication between Kafka and the workflow engine.

Monitor the deployment progress:

```shell
kubectl get pods
```

Expected output (once ready):
```shell
NAME                           READY   STATUS      RESTARTS   AGE
kafka-controller-0             1/1     Running     0          10m
lock-flow-78df85ff6b-xyz12     1/1     Running     0          1m
```

### 🔹 Sending Events to the Workflow
With Kafka and the workflow application running, you can now send events and observe the workflow execution in real-time.

#### **Monitor Workflow Logs**
Open a new terminal and follow the logs of the running workflow pod:
```shell
kubectl logs -f <workflow-pod-name>
```

#### **Send a `lock-event` to Trigger Execution**
Send a `lock-event` message to Kafka, initiating a workflow instance:
```shell
cat kafka-messages/lock-event.json | kubectl exec -i kafka-controller-0 -- kafka-console-producer.sh \
  --broker-list localhost:9092 \
  --topic lock-event
```

Once sent, you should see logs confirming workflow execution:
```shell
INFO  [kogito-event-executor-1] Starting new process instance with signal 'lock-event'
INFO  [kogito-event-executor-1] Lock received: The Kraken
INFO  [kogito-event-executor-1] Waiting lock release: The Kraken
```

#### **Send a `release-event` to Resume Workflow Execution**
```shell
cat kafka-messages/release-event.json | kubectl exec -i kafka-controller-0 -- kafka-console-producer.sh \
  --broker-list localhost:9092 \
  --topic release-event
```

Expected logs:
```shell
INFO  [kogito-event-executor-1] Lock The Kraken released
INFO  [kogito-event-executor-1] Workflow completed successfully
```

#### **Verify the `released-event` in Kafka**
To confirm the workflow emitted the expected event, consume messages from the `released-event` topic:
```shell
kubectl exec -it kafka-controller-0 -- kafka-console-consumer.sh \
  --bootstrap-server kafka.default.svc.cluster.local:9092 \
  --topic released-event --from-beginning
```

Example output:
```json
{"specversion":"1.0","id":"173ce9f2-b9d8-46de-815e-1abb48221607","source":"/process/lock-flow","type":"released-event","time":"2025-03-11T23:18:15.123Z","lockid":"03471a81-310a-47f5-8db3-cceebc63961a","data":{"name":"The Kraken","id":"86ebe1ee-9dd2-4e9b-b9a2-38e865ef1792"}}
```

---

This guide provides an end-to-end approach to deploying and testing event-driven workflows with SonataFlow and Kafka in a Kubernetes environment. 🚀

