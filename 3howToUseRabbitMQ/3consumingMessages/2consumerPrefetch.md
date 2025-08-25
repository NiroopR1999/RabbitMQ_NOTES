## 1 — What is prefetch (in plain words)?

`Prefetch` (aka *prefetch count*, controlled with `basic.qos`) limits how many messages the broker will deliver to a consumer that are **unacknowledged** at any one time.
This helps you control memory and work-in-progress per consumer and implement fair work distribution between workers. ([RabbitMQ][1])

---

## 2 — RabbitMQ vs AMQP: the important difference

AMQP 0-9-1 defines `basic.qos` at the **channel** level and treats prefetch as shared across all consumers on that channel. RabbitMQ *deviates slightly* for performance and convenience: by default RabbitMQ applies the `prefetch_count` **separately to each new consumer** on the channel (i.e., per-consumer), which is usually what applications want. This avoids slow coordination between queues and channels and is much faster in clustered environments. ([RabbitMQ][1])

**Implication:** calling `channel.basicQos(10)` then `basicConsume()` results in a consumer that will receive *up to 10 unacked messages at once*. ([RabbitMQ][1])

---

## 3 — The `basic.qos` parameters (practical summary)

`basic.qos` typically has three meaningful pieces (clients may expose slightly different APIs):

* `prefetch_count` — the numeric limit on unacked messages. In RabbitMQ a non-zero value is applied **per consumer** by default. `0` means “no limit” (infinite). ([RabbitMQ][1])
* `prefetch_size` — a byte-size hint (rarely used in practice).
* `global` flag — whether the prefetch applies to the **channel** (global) or the **consumer** (non-global). RabbitMQ supports both: non-global = per-consumer behavior; `global=true` makes it a channel-wide limit. ([RabbitMQ][1])

---

## 4 — Examples

### Java (per-consumer limit = 10)

```java
Channel channel = ...;
channel.basicQos(10);              // applies to each new consumer on the channel
channel.basicConsume("my-queue", false, consumer);
```

This consumer will have at most **10** unacknowledged messages in flight. `0` would mean no limit. ([RabbitMQ][1])

### Python (pika) — typical worker

```python
import pika

conn = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
ch = conn.channel()

# ensure queue exists and persistent settings if needed
ch.queue_declare(queue='task_queue', durable=True)

# limit to 1 unacked message at a time (fair dispatch for long tasks)
ch.basic_qos(prefetch_count=1)

def callback(ch, method, props, body):
    do_work(body)
    ch.basic_ack(delivery_tag=method.delivery_tag)

ch.basic_consume(queue='task_queue', on_message_callback=callback)
ch.start_consuming()
```

`prefetch_count=1` is a common safe default for long-running tasks — it ensures a worker only gets the next message after it acks the previous one.

---

## 5 — Multiple consumers on same channel and combining limits

RabbitMQ allows multiple calls to `basic.qos` with different `global` flags. When you mix per-consumer and per-channel limits, **both limits are enforced**: a consumer will only receive new messages when neither the per-consumer nor the channel-wide limit is exceeded. Example from the docs: set a per-consumer limit of 10 and a channel-wide limit of 15 — each consumer can have up to 10 unacked messages, but the two consumers together will never exceed 15 unacked messages total. This adds coordination overhead and is therefore slower. ([RabbitMQ][1])

---

## 6 — Why prefetch matters (use cases and effects)

* **Fairness (work queues):** with prefetch=1 you get fair dispatch: RabbitMQ gives a new task only to workers that finished (acked) their previous task. Good for long-running jobs.
* **Throughput**: for many small/fast tasks, a higher prefetch can increase throughput by allowing parallel in-flight work (fewer round trips).
* **Memory usage:** higher prefetch increases memory and WIP on consumers—be mindful if messages are large.
* **Backpressure control:** prefetch is a simple flow-control mechanism at consumer side to avoid overwhelming consumers.

---

## 7 — Recommended tuning process

1. **Start safe:** `prefetch_count=1` for long-running tasks.
2. **Measure:** observe throughput, latency, unacked counts, and CPU/memory.
3. **Increase gradually** for short tasks (2, 5, 10, 50...) until you hit diminishing returns or resource pressure.
4. **Consider message size**: large messages => keep prefetch lower.
5. **Test failure scenarios**: consumer crash, slow consumer — see how requeues affect throughput and ordering.
6. **If consuming multiple queues on one channel**, prefer per-consumer prefetch (the RabbitMQ default) or use separate channels for independent tuning.
7. **Avoid huge prefetch numbers** unless you know consumers can hold that many messages in memory and process them quickly.

---

## 8 — Practical pitfalls & gotchas

* **Setting too high**: large prefetch = more memory usage and more messages “in flight” that may be re-delivered if the consumer dies.
* **Mixing global and per-consumer limits**: combining a per-channel limit with per-consumer limits enforces both — extra coordination slows routing. Use only if you need channel-level throttling. ([RabbitMQ][1])
* **Multiple queues on one channel**: RabbitMQ’s per-consumer default helps, but be thoughtful if you consume many queues from the same channel and try to reuse a single prefetch for all.
* **Auto-ack mode**: prefetch has no effect on ensuring safe processing if you use `auto_ack=True` — messages are considered delivered immediately and prefetch semantics are moot for reliability. Always use manual ack for reliability.
* **0 = infinite**: `prefetch_count=0` disables limits (infinite). Use cautiously. ([RabbitMQ][1])

---

## 9 — Monitoring & metrics to watch

* **Unacked message count per queue/consumer** — indicates in-flight work and if consumers are slow or stuck.
* **Consumer count** — are you under-provisioned?
* **Queue length** — messages waiting to be delivered.
* **Processing latency per message** — to decide prefetch size.
* **Memory & CPU of consumer hosts** — ensure they can hold messages if prefetch is high.

---

## 10 — Special server-side default

RabbitMQ allows a configurable **server default prefetch** that will be applied when consumers don’t set one themselves. This is set via `rabbit.default_consumer_prefetch` in the advanced configuration file (`advanced.config`). Example:

```erlang
[
 {rabbit, [
    {default_consumer_prefetch, {false,250}}
 ]}
].
```

This can be useful to enforce sensible defaults across many clients. ([RabbitMQ][1])

---

## 11 — Interaction with other features

* **Acknowledgements:** prefetch only counts *unacknowledged* messages — always pair prefetch tuning with deliberate ack strategy (e.g., ack after successful processing).
* **Consumer Priorities:** prefetch affects which consumers receive messages; combined with consumer priorities, you can create preferred processing pipelines.
* **Exclusive consumers & E2E topologies:** ensure your prefetch settings match the processing model (exclusive consumers often imply single-worker processing so prefetch can be higher or lower depending on jobs).
* **Clustering:** RabbitMQ’s per-consumer behavior is intentionally faster across clusters because it avoids global coordination for every delivery. Use this to your advantage in distributed setups. ([RabbitMQ][1])

---

## 12 — Quick FAQs

* **Q: Does prefetch prevent message loss?**
  **A:** No — prefetch only controls concurrency and memory. Use manual acks + persistence/quorum queues + publisher confirms for durability.

* **Q: Should I set prefetch on connection or channel?**
  **A:** The `global` flag controls that. Default (non-global) = per consumer (recommended). Use channel/global limits only when you need a channel-wide cap.

* **Q: What is a safe initial value?**
  **A:** `1` for long tasks; 10–50 for fast tasks but measure.

* **Q: Can I change prefetch at runtime?**
  **A:** Yes — you can call `basic.qos` again to change limits. If mixing multiple `basic.qos` calls with differing `global` flags, RabbitMQ enforces all applicable limits. ([RabbitMQ][1])

---

## 13 — Troubleshooting checklist

* If tasks are **unevenly distributed**: check prefetch and make sure it’s not too high for a single fast consumer.
* If **memory spikes** on consumers: lower prefetch.
* If **throughput is low** for small tasks: try increasing prefetch incrementally.
* If **re-orders** happen after failover: be aware re-delivery can change order; consider idempotency.
* If you see **channel-wide slowdowns** after setting global prefetch limits: consider using per-consumer limits or separate channels.

---

## 14 — Closing advice (practical rules of thumb)

* **Prefer manual ack + prefetch tuning** for reliable work queues.
* **Start with prefetch=1** for long jobs; increase for short jobs while measuring.
* **Keep per-consumer defaults** unless you have a reason to use channel-wide limits.
* **Monitor unacked & queue depth** to guide tuning decisions.
* Use server-side `default_consumer_prefetch` if you want a cluster-wide sensible default.

---

### Official doc (primary source)

The behavior described above — RabbitMQ’s per-consumer prefetch, zero meaning infinite, mixing per-consumer and channel limits, and the `default_consumer_prefetch` advanced config — is documented on the RabbitMQ site. See: Consumer Prefetch. ([RabbitMQ][1])

---


### 🔥 RabbitMQ Consumer Prefetch – Q\&A

#### **1. What is consumer prefetch in RabbitMQ?**

Consumer prefetch is a mechanism that controls how many messages RabbitMQ will deliver to a consumer before receiving an acknowledgment for previously sent messages. It prevents a single consumer from being overloaded with too many unacknowledged messages.

---

#### **2. Why is prefetch important?**

Prefetch helps maintain **fair load balancing** between consumers by ensuring that no single consumer receives too many messages at once. It also improves throughput and resource utilization by reducing idle time for other consumers.

---

#### **3. What happens if prefetch is not configured?**

By default, RabbitMQ sends as many messages as possible to consumers (unlimited prefetch). This can lead to one consumer being “flooded” with messages, causing performance bottlenecks and uneven message distribution.

---

#### **4. What is the default prefetch value in RabbitMQ?**

The default prefetch value is **unlimited**. This means RabbitMQ will keep sending messages to a consumer until its TCP buffer is full, regardless of acknowledgment status.

---

#### **5. How can prefetch be set in RabbitMQ?**

You can set the prefetch value:

* **Globally per channel** (affects all consumers on that channel).
* **Per consumer** (specific to an individual consumer).
  This is done using the `basic.qos` method in AMQP or equivalent client library APIs.

---

#### **6. What is the difference between global and per-consumer prefetch?**

* **Global prefetch** applies the limit to the entire channel. All consumers on that channel share the same prefetch count.
* **Per-consumer prefetch** applies the limit individually for each consumer, giving finer control.

---

#### **7. How do you configure global prefetch in RabbitMQ?**

Call `basic.qos` with `global=true`.
Example in Python (Pika):

```python
channel.basic_qos(prefetch_count=10, global_qos=True)
```

---

#### **8. How do you configure per-consumer prefetch in RabbitMQ?**

Call `basic.qos` with `global=false` (default behavior):

```python
channel.basic_qos(prefetch_count=10)
```

---

#### **9. What happens when `prefetch_count=1`?**

When set to `1`, RabbitMQ sends only one unacknowledged message at a time to the consumer. Once the consumer acknowledges that message, RabbitMQ sends the next one.

---

#### **10. Is prefetch applied across all channels?**

No, prefetch is **channel-specific**. You need to configure prefetch on every channel where it’s required.

---

#### **11. How does prefetch impact performance?**

* **Low prefetch (e.g., 1)** ensures fair distribution but may reduce throughput because consumers spend more time waiting for messages.
* **High prefetch (e.g., 100 or unlimited)** improves throughput but risks overloading consumers and causing uneven distribution.

---

#### **12. Can prefetch be changed at runtime?**

Yes, prefetch settings can be changed dynamically during runtime without restarting RabbitMQ or the consumers.

---

#### **13. How does prefetch interact with acknowledgments?**

RabbitMQ will not send more messages than the configured prefetch value **without receiving ACKs**. If a consumer does not acknowledge messages quickly, it will not receive new ones until it does.

---

#### **14. What happens if a consumer crashes with unacked messages?**

If a consumer dies without acknowledging messages, RabbitMQ requeues those messages and delivers them to other available consumers. Prefetch ensures that other consumers are not starved.

---

#### **15. What is the ideal prefetch setting?**

There is no one-size-fits-all setting. A common strategy is to start with a **prefetch of 1** for fairness, then increase it gradually (e.g., 10, 50, 100) to find the balance between fairness and throughput.

---

#### **16. How is prefetch different from concurrency scaling?**

* Prefetch controls **message flow per consumer**.
* Concurrency scaling increases the **number of consumers** to handle load.
  They complement each other but solve different problems.

---

#### **17. What happens if prefetch is too high?**

If prefetch is too high, one consumer may hold many unacknowledged messages, causing slow processing, large memory usage, and unfair load distribution.

---

#### **18. Does prefetch work with `auto_ack=True`?**

No, prefetch has no effect if `auto_ack` is enabled because messages are considered acknowledged as soon as they are sent. Prefetch only works with manual acknowledgments.

---

#### **19. How does prefetch affect message requeueing?**

When a consumer cancels or crashes, messages that were delivered but not acknowledged are requeued. Prefetch determines **how many messages may be requeued** at once.

---

#### **20. Can I set different prefetch counts for different consumers?**

Yes, you can set prefetch counts **per consumer** by using separate channels and configuring `basic.qos` individually for each.

---

#### **21. Is prefetch stored in RabbitMQ or the client library?**

Prefetch is a RabbitMQ **broker-side feature**. The broker enforces prefetch limits and decides how many messages to send to a consumer.

---

#### **22. Does prefetch apply to both AMQP 0-9-1 and AMQP 1.0?**

Prefetch is defined in the **AMQP 0-9-1 protocol**. AMQP 1.0 has a similar feature called **link credit**, but its implementation is different.

---

#### **23. How does prefetch interact with `basic.consume`?**

When you call `basic.consume` to create a consumer, RabbitMQ starts delivering messages according to the prefetch limit you’ve set.

---

#### **24. Should prefetch be used with priority queues?**

Yes, prefetch works fine with priority queues. However, if prefetch is high, lower-priority messages may get delivered early because RabbitMQ pushes messages without re-sorting priorities after delivery.

---

#### **25. How do I test different prefetch values?**

You can simulate load with different consumer prefetch counts using:

1. RabbitMQ Management UI to monitor unacked messages.
2. Load testing tools like **PerfTest**.
3. Adjusting `basic.qos` dynamically and observing message processing times.

---
