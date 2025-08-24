# 📘 RabbitMQ Tutorial 4: Routing (Beginner-Friendly Notes)

This tutorial builds on previous RabbitMQ tutorials (Hello World, Work Queues, Publish/Subscribe).
We’ll dive deeper into **selective message delivery** using **Direct Exchanges** and **Routing Keys**.

---

## 🧠 The Problem We’re Solving

In the previous tutorial, we used a **Fanout Exchange** which sent every message to **all queues** bound to it.

But what if…

* We want **different consumers** to receive **different types of messages**?
* Example:

  * One consumer should get only **error logs**.
  * Another should get only **info or warning logs**.

If we used a Fanout Exchange, **everyone would get everything**.
This is **inefficient**.

We need **routing**.

---

## 🗺️ Key Concept: Direct Exchange

A **Direct Exchange** sends a message to queues **based on an exact match between a routing key and a binding key**.

Think of it as:

```
Message with routing_key="error" → goes only to queues bound with binding_key="error"
```

---

## 🖼️ Diagram of Direct Exchange

```
    Producer ---> [Direct Exchange] ---> Queue1 (binding: error)
                                       ---> Queue2 (binding: info)
                                       ---> Queue3 (binding: warning)
```

* Messages are tagged with a **routing key**.
* Queues are **bound** to an exchange with a **binding key**.
* If they match → message is delivered.

---

## 📝 Python Code Overview

We’ll write **two scripts**:

1. `emit_log_direct.py` (Producer)
2. `receive_logs_direct.py` (Consumer)

---

### 🔹 1. Producer: `emit_log_direct.py`

```python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Declare a direct exchange
channel.exchange_declare(exchange='direct_logs', exchange_type='direct')

# Get severity from CLI arguments (default 'info')
severity = sys.argv[1] if len(sys.argv) > 1 else 'info'
message = ' '.join(sys.argv[2:]) or 'Hello World!'

# Publish message with routing key = severity
channel.basic_publish(
    exchange='direct_logs',
    routing_key=severity,
    body=message
)

print(f" [x] Sent '{severity}':'{message}'")
connection.close()
```

---

🔍 **How it works**:

1. Connect to RabbitMQ.
2. Declare an exchange named `direct_logs` of type `direct`.
3. Take severity (routing key) and message from CLI.
4. Publish the message with that routing key.
5. Close the connection.

---

### 🔹 2. Consumer: `receive_logs_direct.py`

```python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='direct_logs', exchange_type='direct')

# Create a temporary, exclusive queue (name assigned by RabbitMQ)
result = channel.queue_declare('', exclusive=True)
queue_name = result.method.queue

# Get severities from CLI arguments
severities = sys.argv[1:]
if not severities:
    sys.stderr.write("Usage: %s [info] [warning] [error]\n" % sys.argv[0])
    sys.exit(1)

# Bind the queue to exchange with given severities
for severity in severities:
    channel.queue_bind(exchange='direct_logs', queue=queue_name, routing_key=severity)

print('[*] Waiting for logs. To exit press CTRL+C')

def callback(ch, method, properties, body):
    print(f" [x] {method.routing_key}:{body.decode()}")

channel.basic_consume(queue=queue_name, on_message_callback=callback, auto_ack=True)
channel.start_consuming()
```

---

🔍 **How it works**:

1. Connect and declare the same direct exchange.
2. Declare a **temporary queue** (deleted when consumer disconnects).
3. Get a list of severities (routing keys) from CLI arguments.
4. Bind this queue to the exchange for each severity.
5. Start consuming messages.

---

### 🔹 Example Run

**Terminal 1 (Consumer - error only):**

```bash
python receive_logs_direct.py error
```

**Terminal 2 (Consumer - warning and info):**

```bash
python receive_logs_direct.py warning info
```

**Terminal 3 (Producer):**

```bash
python emit_log_direct.py error "Disk is full"
python emit_log_direct.py info "System booted"
```

**Output:**

```
# Terminal 1
[x] error: Disk is full

# Terminal 2
[x] info: System booted
```

---

## 🔑 Key Takeaways

| Concept                | Meaning                                                                               |
| ---------------------- | ------------------------------------------------------------------------------------- |
| **Direct Exchange**    | Routes messages based on an **exact routing key** match.                              |
| **Routing Key**        | A label/tag used to route messages (e.g., "error", "info").                           |
| **Binding Key**        | A key a queue uses to declare interest in certain messages.                           |
| **Temporary Queue**    | Queue without a name, auto-deleted when the consumer disconnects.                     |
| **Multiple Consumers** | Each consumer can choose **which messages** they want by binding with different keys. |

---

## 🧩 When to Use Direct Exchanges

* You need **filtering** based on simple tags like:

  * Error levels (`info`, `warning`, `error`)
  * Departments (`sales`, `finance`, `dev`)
  * Regions (`us-east`, `eu-west`)
* Good for **selective message delivery**.

---

## 🔍 How It Differs from Fanout Exchange

| Feature           | Fanout Exchange               | Direct Exchange               |
| ----------------- | ----------------------------- | ----------------------------- |
| Message delivery  | Sends to **all queues** bound | Sends to **specific queues**  |
| Routing key usage | Ignored                       | **Used to match binding key** |
| Use case          | Broadcast messages            | Filter messages by category   |

---

## 🏗️ Mental Model (Simple Analogy)

Think of:

* **Exchange** = A **mailbox room** in a post office.
* **Routing Key** = Address on the letter.
* **Binding Key** = Address on the mailbox.
* **Queue** = Mailbox that holds letters.
* If address matches mailbox → letter is delivered there.

---

## 📜 Flow Summary

```
Producer → Send message with routing key → Direct Exchange
Exchange → Look at bindings → Deliver to matching queues
Consumer → Reads from their queue
```

---

# 📝 RabbitMQ Tutorial 4: Routing – Q\&A

---

### ❓ Q1: What is the main purpose of “Routing” in RabbitMQ?

**Answer:**
Routing lets you send messages to **specific consumers** instead of broadcasting to everyone.

* Instead of all queues getting every message (like in `fanout`),
* A **routing key** acts like a label or tag on each message.
* Queues can **bind** to exchanges with a certain routing key, so only queues interested in that key receive the message.
  Think of it like writing a message and putting a “department name” on it — only that department gets it.

---

### ❓ Q2: What is a Routing Key in RabbitMQ?

**Answer:**

* A **routing key** is a string (e.g., `"error"`, `"info"`) attached to each message.
* It tells RabbitMQ where the message should go.
* With a `direct` exchange, **queues must bind with the same routing key** to get messages.
  💡 Analogy: Like writing a destination address on a parcel.

---

### ❓ Q3: What is a Direct Exchange?

**Answer:**

* A `direct` exchange delivers messages to queues **whose binding key exactly matches the routing key** of the message.
* It is perfect for **selective delivery**.
  💡 Example:

  * Queue1 bound to `"error"`.
  * Queue2 bound to `"info"`.
  * A message with routing key `"error"` goes only to Queue1.

---

### ❓ Q4: What is Binding in RabbitMQ (again)?

**Answer:**

* **Binding** is like connecting a queue to an exchange with a filter condition (routing key).
* It’s telling RabbitMQ: “Send me only messages with this label.”
  💡 Analogy: Subscribing to only sports news, not everything.

---

### ❓ Q5: How is this different from the Fanout Exchange (Tutorial 3)?

**Answer:**

| Feature      | Fanout Exchange                   | Direct Exchange                                    |
| ------------ | --------------------------------- | -------------------------------------------------- |
| **Routing**  | Broadcasts messages to all queues | Delivers only to queues with matching routing keys |
| **Use case** | Publish/Subscribe to everyone     | Targeted messaging                                 |
| **Control**  | No filtering                      | Fine-grained filtering by key                      |

---

### ❓ Q6: How does this look in code?

* Producer sets a **routing key** when publishing:

  ```python
  channel.basic_publish(
      exchange='direct_logs',
      routing_key='error',
      body='Something went wrong!'
  )
  ```
* Queue is bound to the exchange with the same key:

  ```python
  channel.queue_bind(exchange='direct_logs', queue=queue_name, routing_key='error')
  ```

Result: Queue only gets `"error"` messages.

---

### ❓ Q7: Can a Queue Bind to Multiple Routing Keys?

**Answer:**
Yes!
You can bind one queue to multiple keys:

```python
channel.queue_bind(exchange='direct_logs', queue=queue_name, routing_key='info')
channel.queue_bind(exchange='direct_logs', queue=queue_name, routing_key='warning')
```

This way, the same queue gets messages for both `"info"` and `"warning"`.

---

### ❓ Q8: What Happens if No Queue Matches the Routing Key?

**Answer:**

* If no queue is bound with that routing key, the message is simply **dropped** (unless you configure an alternate exchange).
  💡 So, always ensure correct bindings.

---

### ❓ Q9: When Would You Use Direct Exchanges?

Use it when:

* You need **selective delivery** (not everyone should see all messages).
* You have **log levels** (info, warning, error).
* You want one app to handle errors, another app to handle info logs.

---

### ❓ Q10: Is This Still a Long-Lived AMQP Connection?

Yes! Both producer and consumer maintain their TCP/AMQP connection to RabbitMQ, just like before. Routing doesn’t change the underlying AMQP model; it just adds filtering.

---

### ❓ Q11: Why Use an Exchange Instead of Sending Directly to a Queue?

* Decouples producers and consumers (producer doesn’t need to know queue names).
* More flexible: you can add/remove queues or change routing without modifying the producer code.
* Scales better in large systems.

---

### ❓ Q12: Can I Have Multiple Consumers per Queue?

Yes!
Even if you bind multiple consumers to the same queue, RabbitMQ will **load balance** messages between them.
💡 Example: Multiple workers handling the same type of task.

---
