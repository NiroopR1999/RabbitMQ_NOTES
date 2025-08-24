# RabbitMQ Tutorial 5: Topics (Beginner-Friendly Notes)

---

## What Is This Tutorial About?

In the previous tutorial, we learned how **direct exchanges** route messages based on a single exact match key. While useful, this can be limiting.

With **topic exchanges**, we can route messages using **pattern-based routing** across multiple criteria—like combining severity (`info`, `error`) and source (`kern`, `cron`) in logs. This allows powerful and flexible subscription patterns.

---

## Key Concepts Explained

### 1. **Topic Exchange**

* Routes messages to queues whose binding keys **match a pattern**, using special wildcards:

  * `*` (star) matches **exactly one word**
  * `#` (hash) matches **zero or more words**

For example, if we bind a queue with `*.critical`, it receives messages with two-word routing keys like `kern.critical` or `cron.critical`.

---

### 2. **Routing Key Format**

* Must be a string with words separated by dots:
  e.g. `facility.severity` or `quick.orange.rabbit`
* Each word defines a dimension of the message, allowing fine-grained routing.

---

### 3. **Wildcard Rules**

* `*` (star): matches exactly one word.
  Example: `kern.*` matches `kern.info` but not `kern.critical.error`.

* `#` (hash): matches zero or more words.
  Example: `lazy.#` matches `lazy.orange.rabbit`, `lazy.fox`, and `lazy`.

---

## Example Pattern Binding and Routing

Let's say we bind:

* Queue Q1 → `*.orange.*`
* Queue Q2 → `*.*.rabbit` and `lazy.#`

| Routing Key            | Delivered to Q1? | Delivered to Q2? |
| ---------------------- | ---------------- | ---------------- |
| `quick.orange.rabbit`  | Yes              | Yes              |
| `lazy.orange.elephant` | Yes              | Yes              |
| `quick.orange.fox`     | Yes              | No               |
| `lazy.brown.fox`       | No               | Yes              |
| `lazy.pink.rabbit`     | No               | Yes (once)       |
| `quick.brown.fox`      | No               | No               |

—Provides advanced filtering capabilities.
([rabbitmq.com][1])

---

## Putting It All Together: Complete Python Code

### Producer: `emit_log_topic.py`

```python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='topic_logs', exchange_type='topic')

routing_key = sys.argv[1] if len(sys.argv) > 2 else 'anonymous.info'
message = ' '.join(sys.argv[2:]) or 'Hello World!'

channel.basic_publish(
    exchange='topic_logs',
    routing_key=routing_key,
    body=message
)

print(f" [x] Sent {routing_key}:{message}")
connection.close()
```

* Declares a **topic exchange** named `topic_logs`.
* Sends messages tagged with a routing key like `kern.critical`.

---

### Consumer: `receive_logs_topic.py`

```python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='topic_logs', exchange_type='topic')

result = channel.queue_declare('', exclusive=True)
queue_name = result.method.queue

binding_keys = sys.argv[1:]
if not binding_keys:
    sys.stderr.write("Usage: %s [binding_key]...\n" % sys.argv[0])
    sys.exit(1)

for binding_key in binding_keys:
    channel.queue_bind(exchange='topic_logs', queue=queue_name, routing_key=binding_key)

print(' [*] Waiting for logs. To exit press CTRL+C')

def callback(ch, method, properties, body):
    print(f" [x] {method.routing_key}:{body}")

channel.basic_consume(queue=queue_name, on_message_callback=callback, auto_ack=True)
channel.start_consuming()
```

* Creates a temporary queue and binds it to the `topic_logs` exchange using one or more patterns passed via command line.
* Listens and prints all matching messages.

---

## Sample Usage Patterns

* **Receive all logs:**

  ```
  python receive_logs_topic.py "#"
  ```

* **Only critical logs from `kern`:**

  ```
  python receive_logs_topic.py "kern.critical"
  ```

* **All logs from `kern`, or any critical logs:**

  ```
  python receive_logs_topic.py "kern.*" "*.critical"
  ```

* **Emit a message to combine facility and severity:**

  ```
  python emit_log_topic.py kern.critical "A critical kernel error"
  ```

([rabbitmq.com][1])

---

## Why This is Powerful

* We can **subscribe using patterns** such as `kern.#`, `*.warning`, or `kern.*`.
* Consumers can listen to a broad or very specific set of logs without changes to the publisher.
* It mimics filtering functions often found in real-world systems like logging frameworks (e.g. syslog).

---

# 📝 Q\&A for RabbitMQ Tutorial 5: Topics

---

### **Q1: What is a Topic Exchange in RabbitMQ?**

**Answer:**
A **Topic Exchange** is a type of exchange that routes messages to queues **based on wildcard patterns in routing keys**.

* Unlike **fanout** (broadcast to all) and **direct** exchanges (exact match), topic exchanges allow **flexible routing** using patterns.
* Example:

  * Routing key: `sensor.temperature.kitchen`
  * Binding key: `sensor.*.kitchen` (matches because `*` matches one word)

This is very useful for **complex message filtering** like logs, sensor data, or events with multiple attributes.

---

### **Q2: What are routing keys in Topic Exchanges?**

**Answer:**

* A **routing key** is a string (words separated by dots) attached to every published message.
* Example: `logs.error.backend`
* It tells RabbitMQ how to deliver the message.
* In topic exchanges, the routing key is compared with **binding patterns** using wildcards (`*` and `#`).

---

### **Q3: What are binding keys and how do they work in topic exchanges?**

**Answer:**

* A **binding key** is a pattern used when a queue is bound to an exchange.
* Wildcards allowed:

  * `*` (star): Matches **exactly one word**.
    Example: `sensor.*.kitchen` matches `sensor.temp.kitchen` but not `sensor.temp.indoor.kitchen`.
  * `#` (hash): Matches **zero or more words**.
    Example: `sensor.#` matches `sensor.temp.kitchen`, `sensor.temp`, `sensor`.

---

### **Q4: How does message delivery happen in a Topic Exchange?**

**Answer:**

1. The producer sends a message with a **routing key** to the exchange.
2. The topic exchange looks at all queues bound to it.
3. It checks each queue’s **binding key pattern** against the message’s routing key.
4. If they match, the message is delivered to that queue.
5. Multiple queues can get the same message if multiple binding patterns match.

---

### **Q5: What’s the difference between Topic, Direct, and Fanout Exchanges?**

**Answer:**

| Feature             | Fanout Exchange         | Direct Exchange       | Topic Exchange                        |
| ------------------- | ----------------------- | --------------------- | ------------------------------------- |
| Routing style       | Sends to **all queues** | Exact key match       | Matches **patterns** with `*` and `#` |
| Use case            | Broadcasting            | Simple, exact routing | Complex filtering & selective routing |
| Example routing key | Ignored                 | `info`                | `logs.error.backend`                  |
| Binding flexibility | None                    | Exact only            | Wildcards allow flexible subscription |

---

### **Q6: Why is this useful?**

**Answer:**
Topic exchanges are powerful because:

* They allow **dynamic routing** of messages.
* Consumers can subscribe to **only the messages they care about** (rather than all).
* Scales well for **microservices, logs, IoT sensors, and event-driven systems**.

Example:

* Microservice A listens to `order.*.created`.
* Microservice B listens to `order.#`.
  Both can get relevant messages without unnecessary traffic.

---

### **Q7: What is an example scenario for Topic Exchange?**

**Answer:**
Imagine a **logging system**:

* Producers send logs with routing keys like `logs.error.backend`, `logs.info.frontend`.
* Consumers bind queues to patterns:

  * `logs.error.*` → For teams only interested in error logs.
  * `logs.#` → For full system log collection.

---

### **Q8: What happens if no queue matches the routing key?**

**Answer:**
If a topic exchange doesn’t find any matching queue, the message is **discarded** by default (RabbitMQ doesn’t store unbound messages unless you enable features like **mandatory flag** or **alternate exchanges**).

---

### **Q9: Can a queue have multiple binding patterns?**

**Answer:**
Yes!
A single queue can be bound to a topic exchange with **multiple binding keys**.
Example:

```python
channel.queue_bind(queue_name, exchange_name, "logs.error.#")
channel.queue_bind(queue_name, exchange_name, "logs.warning.#")
```

This allows one queue to receive multiple kinds of messages.

---

### **Q10: Does order of words in routing keys matter?**

**Answer:**
Yes, routing keys are **position-based**.

* `sensor.*.kitchen` matches `sensor.temp.kitchen` but not `sensor.kitchen.temp`.
* Always plan routing keys in a **hierarchical structure** like `system.service.action`.

---

### **Q11: How many words can routing keys have?**

**Answer:**

* Routing keys are typically **dot-separated words**.
* The max length is **255 bytes**.
* RabbitMQ doesn’t limit the number of words, but keep it **reasonable for performance**.

---

### **Q12: Can topic exchanges be used for broadcasting?**

**Answer:**
Yes, if you bind all queues with `#` as the binding key, every message will match all queues. This effectively makes the topic exchange act like a **fanout exchange**.

---

### **Q13: What if RabbitMQ goes down, do bindings stay?**

**Answer:**
Yes.
If you declare **durable exchanges and queues**, RabbitMQ saves the metadata (including bindings) so they survive a restart.
Messages, however, need to be marked **persistent** to survive a crash.

---
