# How to Send and Receive Kafka Messages in [SonataFlow Workflows](https://sonataflow.org)

## **📌 Summary**

This guide explains how to integrate **Kafka messaging** into a **SonataFlow workflow application** using the **Quarkus SmallRye Kafka Connector**. It covers:

- Defining and handling workflow events
- Ensuring event correlation within workflow instances
- Configuring Kafka topics and CloudEvents format

---

## **📌 Understanding the Event Definitions**

The [workflow specification](callback-flow/src/main/resources/reproducer.sw.yaml) defines three key events:

```yaml
events:
  - type: lock-event
    kind: consumed
    name: lock-event
    source: local
    correlation:
      - contextAttributeName: lockid
  - type: released-event
    kind: produced
    name: released-event
    correlation:
      - contextAttributeName: lockid
```

### **🔹 Event Breakdown**
1. **`lock-event`** → Triggers the workflow execution.
2. **`released-event`** → Workflow emits this event upon termination.

SonataFlow only accepts messages in the [CloudEvents](https://cloudevents.io/) format. Here’s an example of a **valid CloudEvent JSON message**:

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

### **🔹 Correlation in SonataFlow**
Each event definition contains `correlation.contextAttributeName`, which enables **event correlation** across workflow instances.

For example, if **10 workflows** are waiting for the `release-event`, SonataFlow will match incoming events by their `lockid` attribute to **resume only the relevant workflow instance**.

Additionally, Kafka topics are mapped to event **types**. By default, topics are named based on event types:
- `lock-event`
- `released-event`

To customize the Kafka topic name, override it in **`application.properties`**:
```properties
mp.messaging.incoming.lock-event.topic=MySuperFancyTopicName
```

---
## **🚀 Running the Workflow in Dev Mode**

### 1️⃣ Start a Kafka Instance

**Run the broker:**
```bash
podman run -it --rm --name kafka \
  -p 9092:9092 \
  -e KAFKA_CFG_NODE_ID=0 \
  -e KAFKA_CFG_PROCESS_ROLES=controller,broker \
  -e KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@localhost:9093 \
  -e KAFKA_CFG_LISTENERS=PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093 \
  -e KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092,CONTROLLER://localhost:9093 \
  -e KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT \
  -e KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER \
  -e KAFKA_CFG_INTER_BROKER_LISTENER_NAME=PLAINTEXT \
  bitnami/kafka:latest
```

## **🚀 Running the Workflow in Dev Mode (Common Steps)**

### **2️⃣ Start the Quarkus Application**
```shell
cd callback-flow
mvn clean quarkus:dev
```

### **3️⃣ Send Events to Kafka**
```shell
jq -c . kafka-messages/lock-event.json | podman exec -i kafka kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic lock-event
```

### **4️⃣ Verify Workflow Completion**
Refresh the **SonataFlow DevUI Console** to see the workflow state as completed.

Check the produced event:
```shell
podman exec -i kafka kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic released-event --from-beginning
```

Example output:
```json
{
  "specversion": "1.0",
  "id": "c4cba9e6-2b10-44f6-b2f3-b96a12d875f1",
  "source": "/process/lock-flow",
  "type": "released-event",
  "time": "2025-04-24T08:31:45.67159+03:00",
  "lockid": "03471a81-310a-47f5-8db3-cceebc63961a",
  "kogitoproctype": "SW",
  "kogitoprocinstanceid": "d13ed302-fe09-41ca-abda-d8b3c218a105",
  "kogitoprocist": "Active",
  "kogitoprocversion": "1.0.0",
  "kogitoprocid": "lock-flow",
  "data": {
    "name": "The Kraken",
    "id": "03471a81-310a-47f5-8db3-cceebc63961a"
  }
}
```


### **5️⃣ Stop Kafka**
```shell
podman container stop kafka
```
