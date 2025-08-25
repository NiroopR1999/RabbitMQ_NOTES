# RabbitMQ Streams Tutorial 1: "Hello World" in Python (rstream client)

---

## What Is RabbitMQ Streams?

RabbitMQ Streams is a newer messaging model introduced in RabbitMQ 3.9. Unlike traditional AMQP queues, streams offer a **log-like, append-only data structure** that supports:

* **Non-destructive reads** — multiple consumers can read the same message.
* **Retention policies** — messages persist until they reach a size or time limit.
* **Efficient data distribution** — ideal for streaming, analytics, or event logs.
  ([Akamai][1])

---

## Tutorial Overview

In this “Hello World” guide, you'll write two Python programs:

1. **Producer (`send.py`)** – publishes a single message to a stream.
2. **Consumer (`receive.py`)** – listens to the stream and prints incoming messages.
   ([RabbitMQ][2])

---

## Prerequisites

* **RabbitMQ Server** with the **Stream plugin** enabled (stream port: **5552**).

* Recommended via Docker:

  ```bash
  docker run -it --rm --name rabbitmq -p 5552:5552 -p 15672:15672 -p 5672:5672 \
    -e RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS='-rabbitmq_stream advertised_host localhost' \
    rabbitmq:4-management
  docker exec rabbitmq rabbitmq-plugins enable rabbitmq_stream rabbitmq_stream_management
  ```

  ([RabbitMQ][3])

* Python 3.9+ & `rstream` library installed:

  ```bash
  pip install rstream
  ```

  ([RabbitMQ][2])

---

## Producer: `send.py`

### Steps:

1. Import `Producer` from `rstream` and `asyncio`.
2. Connect to RabbitMQ stream endpoint.
3. Declare a stream (if not existing) with a retention policy.
4. Send one message.
5. Exit.

### Sample Code Snippet:

```python
import asyncio
from rstream import Producer

STREAM_NAME = "hello-python-stream"
STREAM_RETENTION = 5_000_000_000  # ~5 GB

async with Producer(host="localhost", username="guest", password="guest") as producer:
    await producer.create_stream(
        STREAM_NAME,
        exists_ok=True,
        arguments={"MaxLengthBytes": STREAM_RETENTION}
    )
    await producer.send(stream=STREAM_NAME, message=b"Hello, World!")
```

* The stream is **append-only** and only created if it doesn’t already exist.
* `MaxLengthBytes` ensures retention control.
  ([RabbitMQ][2])

---

## Consumer: `receive.py`

### Steps:

1. Import `Consumer`, `AMQPMessage`, etc., from `rstream`.
2. Connect to RabbitMQ and declare the same stream.
3. Subscribe to the stream using an offset (start from earliest).
4. Define a callback to print messages.
5. Keep listening.

### Sample Code Snippet:

```python
import asyncio
from rstream import Consumer, AMQPMessage, MessageContext, ConsumerOffsetSpecification, OffsetType

async with Consumer(host="localhost", username="guest", password="guest") as consumer:
    await consumer.create_stream(
        STREAM_NAME,
        exists_ok=True,
        arguments={"MaxLengthBytes": STREAM_RETENTION}
    )

    await consumer.subscribe(
        STREAM_NAME,
        offset_spec=ConsumerOffsetSpecification(type=OffsetType.EARLIEST),
        on_message=on_message
    )

async def on_message(msg: AMQPMessage, ctx: MessageContext):
    print(f"Got message: {msg.body} from stream {ctx.consumer.get_stream(ctx.subscriber_name)}")
```

* Consumers declare the stream too, so order of start (producer/consumer) doesn't matter.
  ([RabbitMQ][2])

---

## Why This Matters

* Streams allow **multiple readers** without removing messages.
* Handy for facilitating **pub/sub systems**, real-time analytics, or replayable logs.
* Retention policies help manage **storage limits** effectively.
* Skills here extend into more robust patterns like **offset tracking** (next-level tutorial).

---

### 📝 Q\&A: RabbitMQ Tutorial One (Python Stream)

#### 1. **Q: What is the goal of this tutorial?**

**A:** This tutorial teaches how to use **RabbitMQ Streams**, a special type of queue designed for **high-throughput messaging** and **large message histories**. You’ll learn how to publish and consume messages from a **stream** instead of a traditional queue.

---

#### 2. **Q: How is a stream different from a normal RabbitMQ queue?**

**A:**

* **Stream:** Stores messages on disk for a configurable time or size limit, supports **sequential access**, and is optimized for **very high throughput**.
* **Queue:** Optimized for reliable message delivery but doesn’t hold long histories.

---

#### 3. **Q: What makes streams good for large-scale systems?**

**A:**

* They store messages **on disk**, not just in memory.
* They support **fast sequential reads** (like a log).
* They allow **multiple consumers** to read the same data independently.

---

#### 4. **Q: What library is used in this tutorial to interact with RabbitMQ Streams?**

**A:** It uses the **`pika`** library (like previous tutorials) along with the `rabbitmq-stream` plugin to enable streaming capabilities.

---

#### 5. **Q: Do we need to enable anything on RabbitMQ to use streams?**

**A:** Yes! You must enable the **RabbitMQ Streams Plugin** by running:

```bash
rabbitmq-plugins enable rabbitmq_stream
```

---

#### 6. **Q: How do we declare a stream in RabbitMQ?**

**A:** We declare a stream just like a queue but use the `x-queue-type` argument:

```python
channel.queue_declare(queue='mystream', arguments={'x-queue-type': 'stream'})
```

---

#### 7. **Q: Can streams be durable?**

**A:** Yes. Declaring a stream with `durable=True` ensures that its definition survives broker restarts. Messages in a stream are **persisted to disk by default**, which means they survive restarts too.

---

#### 8. **Q: How is publishing to a stream different from publishing to a queue?**

**A:** There is **no difference in basic publishing code**—you still use `basic_publish()`. The difference is in the **queue type**: a stream will store messages differently under the hood.

---

#### 9. **Q: What is the retention policy in streams?**

**A:** A retention policy decides **how long** or **how much data** a stream keeps. For example, you can keep messages for 7 days or up to 1GB of data per stream.

---

#### 10. **Q: Can multiple consumers read from the same stream at different speeds?**

**A:** Yes! Each consumer can **independently track its offset** (position in the stream), so one consumer can be at the latest message, while another replays old messages.

---

#### 11. **Q: What is an offset in RabbitMQ streams?**

**A:** The **offset** is a marker of where a consumer is in the stream. Think of it like a bookmark in a log file—it tells RabbitMQ where to start delivering messages for that consumer.

---

#### 12. **Q: What are the common offset options for consumers?**

**A:**

* `first`: Start from the very first message in the stream.
* `last`: Start from the latest message.
* `next`: Start from the next message after your last consumed one.
* A **specific offset number** if you want to replay from a certain point.

---

#### 13. **Q: Why are streams useful for event sourcing?**

**A:** Because they store **long message histories** and allow consumers to **replay messages** at any time, making them ideal for systems that need to rebuild state or audit historical data.

---

#### 14. **Q: Do streams replace queues in RabbitMQ?**

**A:** No! Streams are a **specialized type of queue** for high-throughput, log-like use cases. Normal queues are still better for simple, point-to-point messaging and task distribution.

---

#### 15. **Q: Are streams FIFO (First In, First Out)?**

**A:** Yes, messages in a stream are always stored and read in strict **publish order**.

---

#### 16. **Q: Can I acknowledge messages in a stream?**

**A:** Yes, you can acknowledge messages just like queues. However, acknowledgments don’t remove messages from the stream—they remain until retention rules delete them.

---

#### 17. **Q: What happens when a stream reaches its retention limit?**

**A:** The broker **automatically deletes old messages** according to the configured retention policy (time or size).

---

#### 18. **Q: Is stream data stored in memory or disk?**

**A:** Streams are **disk-based**. They are designed to store large amounts of data while minimizing memory usage.

---

#### 19. **Q: Can streams be mirrored or clustered?**

**A:** Yes! Streams can be **mirrored across RabbitMQ nodes** for high availability and fault tolerance.

---

#### 20. **Q: What are good use cases for RabbitMQ Streams?**

**A:**

* Event sourcing and audit logs
* Analytics pipelines
* High-throughput telemetry data
* Long-term replayable messaging
* IoT or logging systems

---

#### 21. **Q: How do consumers track their position in a stream?**

**A:** RabbitMQ uses **consumer offsets** stored on the broker or client side. This allows consumers to disconnect and reconnect without losing their place.

---

#### 22. **Q: Are RabbitMQ Streams similar to Apache Kafka?**

**A:** Yes! RabbitMQ Streams provide **Kafka-like features** (persistent logs, replay, offsets) but with RabbitMQ’s flexibility and familiar AMQP model.

---

#### 23. **Q: What is the maximum message size for a stream?**

**A:** Streams can handle messages up to **16 MB** by default, much larger than traditional queues.

---

#### 24. **Q: Can streams support multiple producers?**

**A:** Absolutely. Multiple producers can write messages to the same stream simultaneously, just like a queue.

---

#### 25. **Q: Do streams work with existing RabbitMQ clients?**

**A:** Yes! Most RabbitMQ clients (like `pika`) work with streams, but some **advanced features** may need specialized APIs or the `rabbitmq-stream` client library.

---

#### 26. **Q: Can streams have exchanges?**

**A:** Yes. You can bind a stream to an exchange, allowing messages to be routed just like with normal queues.

---

#### 27. **Q: How are streams created from the CLI?**

**A:**

```bash
rabbitmq-streams add_stream mystream
```

---

#### 28. **Q: Can you replay messages in order?**

**A:** Yes! Streams guarantee strict ordering, and you can replay messages from any offset in the original publish order.

---

#### 29. **Q: Do streams support sharding or partitions?**

**A:** RabbitMQ Streams don’t yet have **native sharding/partitioning** like Kafka, but you can implement a similar pattern by creating multiple streams.

---

#### 30. **Q: Should I always use streams?**

**A:** No. Use streams if you need **replayable, persistent, high-throughput messaging**. If your system only needs **simple, one-time delivery**, normal queues are simpler and lighter.

---

This gives you **30 solid Q\&A points** about Tutorial 1 (Streams) in a beginner-friendly yet technical way.

---
