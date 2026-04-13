# Message Brokers

This directory contains Podman configurations for three messaging systems: **RabbitMQ**, **Apache Kafka**, and the **Google Cloud PubSub emulator**.

## Table of Contents

- [Overview](#overview)
- [Services](#services)
  - [RabbitMQ 3.12](#rabbitmq-312)
  - [Apache Kafka 7.5](#apache-kafka-75)
  - [Google PubSub Emulator](#google-pubsub-emulator)
- [Quick Start](#quick-start)
- [Choosing a Message Broker](#choosing-a-message-broker)
- [Ports Reference](#ports-reference)

---

## Overview

| Broker         | Protocol   | Model           | Best For                              |
|----------------|------------|-----------------|---------------------------------------|
| RabbitMQ       | AMQP       | Push (broker-driven) | Task queues, RPC, routing patterns |
| Kafka          | Binary TCP | Pull (consumer-driven) | Event streaming, log aggregation, high throughput |
| Google PubSub  | HTTP/gRPC  | Push + Pull     | GCP-native event streaming           |

---

## Services

### RabbitMQ 3.12

[RabbitMQ](https://www.rabbitmq.com/) is a mature message broker supporting AMQP, MQTT, STOMP, and more.

- **Management UI**: http://localhost:15672
- **Login**: `admin` / `adminpassword`
- **AMQP Port**: `5672`

**Send a message (Python):**
```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters(
        host="localhost",
        credentials=pika.PlainCredentials("admin", "adminpassword")
    )
)
channel = connection.channel()
channel.queue_declare(queue="tasks", durable=True)
channel.basic_publish(
    exchange="",
    routing_key="tasks",
    body='{"job": "send_email", "to": "user@example.com"}',
    properties=pika.BasicProperties(delivery_mode=2)  # persistent
)
connection.close()
```

**Consume messages (Python):**
```python
def callback(ch, method, properties, body):
    print(f"Received: {body}")
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_qos(prefetch_count=1)
channel.basic_consume(queue="tasks", on_message_callback=callback)
channel.start_consuming()
```

**Exchange types:**
| Type    | Description                                    |
|---------|------------------------------------------------|
| direct  | Route by routing key (default)                 |
| topic   | Route by pattern matching (`order.*`)          |
| fanout  | Broadcast to all bound queues                  |
| headers | Route by message headers                       |

**Useful RabbitMQ CLI commands:**
```bash
# List queues
podman exec rabbitmq rabbitmqctl list_queues name messages

# List exchanges
podman exec rabbitmq rabbitmqctl list_exchanges

# Add a user
podman exec rabbitmq rabbitmqctl add_user newuser password
podman exec rabbitmq rabbitmqctl set_permissions -p / newuser ".*" ".*" ".*"
```

---

### Apache Kafka 7.5

[Apache Kafka](https://kafka.apache.org/) is a distributed event streaming platform capable of handling trillions of events per day.

**Architecture:**
- `zookeeper` — coordinates Kafka broker metadata (port 2181)
- `kafka` — the broker itself (port 9092 external, 29092 internal)

**Configuration highlights:**
- 1 broker, 3 default partitions
- Auto topic creation enabled
- 7-day log retention

**Create a topic:**
```bash
podman exec kafka kafka-topics \
  --bootstrap-server localhost:9092 \
  --create \
  --topic orders \
  --partitions 3 \
  --replication-factor 1
```

**List topics:**
```bash
podman exec kafka kafka-topics \
  --bootstrap-server localhost:9092 \
  --list
```

**Produce messages:**
```bash
podman exec -it kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic orders
> {"order_id":1,"amount":99.99}
> {"order_id":2,"amount":49.00}
```

**Consume messages:**
```bash
podman exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic orders \
  --from-beginning
```

**Python producer (confluent-kafka):**
```python
from confluent_kafka import Producer

producer = Producer({"bootstrap.servers": "localhost:9092"})

def delivery_report(err, msg):
    if err:
        print(f"Delivery failed: {err}")
    else:
        print(f"Delivered to {msg.topic()} [{msg.partition()}]")

producer.produce(
    "orders",
    key="order-1",
    value='{"order_id":1,"amount":99.99}',
    callback=delivery_report
)
producer.flush()
```

**Python consumer:**
```python
from confluent_kafka import Consumer

consumer = Consumer({
    "bootstrap.servers": "localhost:9092",
    "group.id": "order-processor",
    "auto.offset.reset": "earliest"
})
consumer.subscribe(["orders"])

while True:
    msg = consumer.poll(1.0)
    if msg and not msg.error():
        print(f"Received: {msg.value().decode('utf-8')}")
```

---

### Google PubSub Emulator

The [Google Cloud PubSub Emulator](https://cloud.google.com/pubsub/docs/emulator) allows local development without a GCP account.

- **Emulator endpoint**: `localhost:8085`
- **Project**: `local-project`

**Set environment variable:**
```bash
export PUBSUB_EMULATOR_HOST=localhost:8085
export PUBSUB_PROJECT_ID=local-project
```

**Create a topic and subscription (Python):**
```python
import os
os.environ["PUBSUB_EMULATOR_HOST"] = "localhost:8085"

from google.cloud import pubsub_v1

project_id = "local-project"
topic_id = "my-topic"
subscription_id = "my-subscription"

# Create topic
publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path(project_id, topic_id)
publisher.create_topic(request={"name": topic_path})

# Create subscription
subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path(project_id, subscription_id)
subscriber.create_subscription(request={"name": subscription_path, "topic": topic_path})

# Publish
future = publisher.publish(topic_path, b"Hello, PubSub!")
print(f"Published message ID: {future.result()}")

# Pull messages
response = subscriber.pull(request={"subscription": subscription_path, "max_messages": 10})
for msg in response.received_messages:
    print(f"Received: {msg.message.data}")
    subscriber.acknowledge(request={"subscription": subscription_path, "ack_ids": [msg.ack_id]})
```

**Using gcloud CLI:**
```bash
# Set emulator host
export PUBSUB_EMULATOR_HOST=localhost:8085

# Create topic
gcloud pubsub topics create my-topic --project=local-project

# Publish
gcloud pubsub topics publish my-topic --message="hello" --project=local-project

# Create subscription and pull
gcloud pubsub subscriptions create my-sub --topic=my-topic --project=local-project
gcloud pubsub subscriptions pull my-sub --auto-ack --project=local-project
```

---

## Quick Start

```bash
# Start all message brokers
podman-compose up -d

# Check services
podman-compose ps

# Wait for Kafka to be ready
podman-compose logs -f kafka

# Stop
podman-compose down
```

---

## Choosing a Message Broker

| Criterion             | RabbitMQ         | Kafka               | Google PubSub       |
|-----------------------|------------------|---------------------|---------------------|
| Message ordering      | Per-queue        | Per-partition        | Best-effort         |
| Throughput            | High (~50k msg/s)| Very high (1M+ msg/s) | Very high (GCP-managed) |
| Message retention     | Until consumed   | Configurable (days)  | Configurable        |
| Replay old messages   | ❌               | ✅                  | ✅ (with snapshots) |
| Complex routing       | ✅ (exchanges)   | Limited (topic naming) | Limited           |
| Consumer groups       | Competing consumers | ✅ Consumer groups | ✅ Subscriptions   |
| Schema registry       | ❌ (use plugins) | ✅ (Confluent Schema Registry) | ✅ Pub/Sub schemas |
| Local dev             | ✅ Easy          | ✅ (needs ZK)       | ✅ (emulator)       |

---

## Ports Reference

| Service          | Port  | Protocol | Description                        |
|------------------|-------|----------|------------------------------------|
| RabbitMQ         | 5672  | AMQP     | Message protocol                   |
| RabbitMQ         | 15672 | HTTP     | Management UI                      |
| Zookeeper        | 2181  | TCP      | ZooKeeper client port              |
| Kafka            | 9092  | TCP      | Kafka broker (external/host access)|
| Kafka            | 29092 | TCP      | Kafka broker (internal container)  |
| PubSub Emulator  | 8085  | HTTP/gRPC| Google PubSub emulator             |
