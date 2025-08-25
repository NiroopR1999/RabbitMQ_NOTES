## Classic Queues — In-Depth Overview (RabbitMQ 4.1 docs)

**Source:** RabbitMQ official: *Classic Queues* documentation ([RabbitMQ][1])

---

### 1. What Are Classic Queues?

* **Definition:** The original, non-replicated FIFO queue type in RabbitMQ, suitable when data safety isn’t critical because message contents aren't replicated across nodes. Classic Queues are the default type unless overridden per vhost. ([RabbitMQ][1])

---

### 2. Key Features & Capabilities

Classic Queues fully support:

* **Queue exclusivity**
* **Message & queue TTL (Time-To-Live)**
* **Queue length limits**
* **Message priority**
* **Consumer priority**
* Adherence to **policy-controlled settings**
* **Dead Letter Exchanges (DLX)** — except for at-least-once DLX behavior, which is *only* supported in Quorum Queues ([RabbitMQ][1])
* Support for **global QoS prefetch** exists but is deprecated — prefer per-consumer QoS. This feature will be removed in RabbitMQ 4.0 ([RabbitMQ][1])
* **Transient queues** are supported but deprecated and discouraged; removal is planned in RabbitMQ 4.0 due to upgrade complications ([RabbitMQ][1])

---

### 3. Persistence & Storage Behavior

* **Index-based storage:** Classic Queues use an on-disk index to store message positions and rely on a message store for content. ([RabbitMQ][1])
* **In-memory retention:** Up to \~2048 messages may be kept in memory, depending on consumption rate. Default behavior is writing to disk unless consumption is fast. ([RabbitMQ][1])
* **Message embedding logic:**

  * *Embeds* small messages (<4 KB including headers) in the queue index
  * *Stores larger messages* in a shared vhost message store for efficiency ([RabbitMQ][1])

---

### 4. Implementation Versions: CQv1 vs CQv2

| Feature            | CQv1                    | CQv2 (RabbitMQ ≥ 3.10)     |
| ------------------ | ----------------------- | -------------------------- |
| Message embedding  | Embedded small messages | No embedding               |
| Message store      | Shared across vhost     | Per-queue message store    |
| Status in RabbitMQ | Deprecated in 4.0       | Default and only supported |

* **Migration:** On startup, RabbitMQ 4.0 automatically migrates CQv1 queues to CQv2, which may temporarily block availability, especially with large queues. ([RabbitMQ][1])

---

### 5. Behavior Similar to Lazy Queues (Default in CQv2)

* Starting with RabbitMQ 3.12, Classic Queues (CQv2) default to "lazy-like" behavior: they store most messages on disk, keeping minimal data in memory for faster consumption, without manual lazy mode config. ([CloudAMQP][2])
* CQv2 offers significantly better performance and more stable memory usage than legacy CQv1. ([CloudAMQP][2])

---

### 6. Priority Queues in Classic Queues

* **Supported** via `x-max-priority` argument (range 1–255, but 1–5 recommended). Higher priorities use more memory/CPU internally. ([RabbitMQ][3])
* Messages get a `priority` property in their headers; higher numbers are delivered first.
* Policies *cannot* enable priority queues — must be declared at queue creation. ([RabbitMQ][3])

---

### 7. Deprecations & Removals in RabbitMQ ≥ 4.0

* **Transient queues** — deprecated and removed in 4.0 ([RabbitMQ][1])
* **Global QoS prefetch** — deprecated, removed in 4.0 ([RabbitMQ][1])
* **Mirrored classic queues** — deprecated; removed in 4.0 in favor of Quorum Queues for HA and replication ([RabbitMQ][1])

---

### 8. Summary of Storage Evolution

* CQv1: Messages embedded in index for small items, others in shared store
* CQv2: Uses per-queue store consistently; no embedding
* Lazy-like behavior: Low-memory footprint via CQv2 default since 3.12
* CQv1 → CQv2 migration done automatically on RabbitMQ 4.0 boot

---

### TL;DR Summary

Classic Queues are flexible, feature-rich, non-replicated queues ideal for general use. But for high availability or replication, consider Quorum Queues or Streams. CQv2 is the modern implementation, offering efficient disk-based storage and default lazy behavior without manual config.

---


### 🔹 General Questions

#### 1. What are Classic Queues in RabbitMQ?

**Answer:**
Classic Queues are the original queue type in RabbitMQ. They are designed for simplicity and efficiency, providing at-least-once message delivery guarantees. Classic queues store messages in memory and/or on disk, depending on configuration, and are best suited for workloads where:

* Low latency and high throughput are needed.
* The queue does not need distributed consensus (unlike quorum queues).
  They have been available since early versions of RabbitMQ, and they remain widely used in production systems.

---

#### 2. How are Classic Queues different from Quorum Queues?

**Answer:**

| Feature            | Classic Queue                          | Quorum Queue                                 |
| ------------------ | -------------------------------------- | -------------------------------------------- |
| Replication        | Optional (via Mirrored Queues)         | Always replicated via Raft consensus         |
| Delivery Guarantee | At-least-once                          | At-least-once                                |
| Throughput         | Higher throughput, lower latency       | Slightly lower throughput due to replication |
| Failover           | Manual mirroring setup                 | Automatic with quorum replication            |
| Use Case           | Low-latency, high-throughput workloads | HA and consistency-critical workloads        |

---

#### 3. What storage strategies do Classic Queues use?

**Answer:**
Classic Queues use **segmented message storage**:

* Messages can be stored **in-memory** or **persisted on disk**.
* Storage is optimized to minimize memory pressure and reduce disk I/O.
* Segments allow RabbitMQ to efficiently handle queues with a large number of messages.

---

### 🔹 Features and Behaviors

#### 4. What are Lazy Queues?

**Answer:**
Lazy Queues are a feature of Classic Queues where messages are written to disk as soon as possible, rather than kept in memory. This:

* Reduces memory consumption.
* Increases I/O operations (slightly slower).
* Is ideal for very large queues that hold millions of messages.

You can enable lazy queues by:

```bash
rabbitmqctl set_policy lazy ".*" '{"queue-mode":"lazy"}' --apply-to queues
```

---

#### 5. How does message persistence work in Classic Queues?

**Answer:**

* Messages can be **persistent** (survive broker restarts) if published as `persistent`.
* Queues themselves can be **durable**, meaning the definition survives restarts.
* RabbitMQ writes messages to disk asynchronously for efficiency but ensures safety by flushing data to disk on important events (like `confirm`).

---

#### 6. How do Classic Queues handle high loads?

**Answer:**

* RabbitMQ uses **flow control** to throttle publishers when the broker experiences resource pressure.
* Messages can be paged out from memory to disk.
* **Consumer prefetch** can help balance throughput and fairness between consumers.
* Segmented storage allows RabbitMQ to handle millions of messages efficiently.

---

#### 7. What happens when queues grow very large?

**Answer:**

* Messages are written to disk to avoid memory exhaustion.
* RabbitMQ may apply backpressure to publishers to slow down message ingress.
* For extremely large queues, lazy mode is recommended to avoid memory bloat.

---

### 🔹 Availability and Failover

#### 8. What is a Mirrored Queue, and how does it work?

**Answer:**
A Mirrored Queue (HA Queue) is a Classic Queue replicated across multiple RabbitMQ nodes:

* One node acts as the master; others are mirrors.
* All messages are replicated to all mirrors.
* If the master node fails, a mirror is promoted to master.
  However, mirrored queues are considered **legacy** and are being replaced by **quorum queues** for HA.

---

#### 9. Why are Quorum Queues recommended over Mirrored Classic Queues for HA?

**Answer:**

* Mirrored queues are harder to manage and can cause inconsistencies during failover.
* Quorum queues use Raft consensus, which is simpler and more reliable.
* RabbitMQ maintainers plan to deprecate mirrored queues in favor of quorum queues.

---

### 🔹 Configuration and Management

#### 10. How do you configure Classic Queues?

**Answer:**
You can configure queues via:

* **Policies:**

```bash
rabbitmqctl set_policy "queue_policy" "^myqueue$" '{"ha-mode":"all"}' --apply-to queues
```

* **Queue arguments:** When declaring queues in your code:

```python
channel.queue_declare(queue='my-queue', durable=True, arguments={"x-queue-mode": "lazy"})
```

* Management UI and `rabbitmqctl` for dynamic changes.

---

#### 11. What is Queue Mode?

**Answer:**
Queue mode determines where messages are stored:

* `default`: Keeps as many messages in memory as possible.
* `lazy`: Writes all messages to disk immediately to reduce memory usage.

---

#### 12. How does RabbitMQ handle message ordering in Classic Queues?

**Answer:**

* Messages are **strictly ordered** (FIFO).
* Order is preserved unless consumers `nack` or `reject` messages with requeueing, which can alter delivery order.

---

### 🔹 Performance Considerations

#### 13. When should you use Classic Queues?

**Answer:**

* When you need **low-latency** and **high throughput** messaging.
* When replication is **not required** (single-node or simple workloads).
* When you want **manual control** over HA setups with mirrored queues.

---

#### 14. When should you avoid Classic Queues?

**Answer:**

* When you need **automated replication** and consistency (use quorum queues instead).
* When running in cloud-native environments where **node failures are frequent**.
* When handling very large backlogs (lazy queues or quorum queues are better).

---

#### 15. How do Classic Queues interact with Publishers and Consumers?

**Answer:**

* Publishers push messages asynchronously; RabbitMQ confirms them if `publisher confirms` are enabled.
* Consumers receive messages in a push-based manner, with **prefetch** controlling how many are sent at once.
* Acknowledgements (ack/nack) ensure messages aren’t lost.

---
