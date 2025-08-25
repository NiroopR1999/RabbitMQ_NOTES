# Consumer Priority — detailed, well-explained notes

# 1 — Short summary / elevator pitch

**Consumer priority** lets you tell RabbitMQ *which consumers should be preferred* when the broker dispatches messages from a queue. You assign an integer priority (`x-priority`) when you start a consumer; higher integers mean higher priority. The broker will deliver available messages first to the *active consumers with the highest priority*, and only when those consumers are blocked (e.g., due to prefetch limits) will it deliver to lower-priority consumers.

Use cases: favor faster machines, prefer consumers on the queue master node (data locality), have hot standby consumers, prioritize traffic to local services, or implement graceful takeover logic.

---

# 2 — Precise behavior (what happens at dispatch time)

* When multiple consumers are attached to the **same queue**, RabbitMQ chooses *which consumer(s)* get each message.
* With **consumer priority**:

  * The broker computes the **highest priority value** among currently **active** consumers for that queue.
  * It dispatches messages only to consumers whose `x-priority` equals that highest value.
  * If there are several consumers with the same top priority, messages are distributed **round-robin** among those consumers (subject to other constraints like prefetch).
  * **Lower priority** consumers receive deliveries only when **no active higher-priority** consumer can accept more messages.
* A consumer is **active** if it is able to accept deliveries (not blocked by prefetch, not network-saturated, not paused). Prefetch and consumer blocking influence activity.
* If a new higher-priority consumer connects later, RabbitMQ will *stop delivering* to lower-priority consumers and deliver to the higher-priority ones instead—subject to in-flight message completion semantics described below (see SAC/quorum behavior).

---

# 3 — How to declare a consumer priority

You set consumer priority via the `x-priority` consumer argument in the AMQP `basic.consume` call. The exact mechanism depends on the client library; the value is an integer. Consumers that do not provide `x-priority` default to priority `0`.

**Java example**

```java
Map<String, Object> args = new HashMap<>();
args.put("x-priority", 10);
channel.basicConsume("my-queue", false, "", false, false, args, myConsumer);
```

**Python (pika) conceptual example**

```python
# many pika versions accept 'arguments' in basic_consume
arguments = {'x-priority': 10}
channel.basic_consume(queue='my-queue',
                      on_message_callback=callback,
                      auto_ack=False,
                      arguments=arguments)
```

**.NET (RabbitMQ.Client)**

```csharp
var args = new Dictionary<string, object> { { "x-priority", 10 } };
channel.BasicConsume(queue: "my-queue", autoAck: false, consumerTag: "", noLocal: false,
                     exclusive: false, arguments: args, consumer: consumer);
```

Notes:

* Priority value range: can be positive, zero, or negative integers. RabbitMQ compares integers; use a consistent scheme in your system (e.g., `0` base, `10` for high).
* The client library must support passing `arguments` to `basic.consume` — most do.

---

# 4 — Interaction with prefetch / basic.qos

Prefetch (consumer QoS) limits the number of *unacknowledged messages* the broker will deliver to a consumer at once.

Important interactions:

* **A consumer can be high priority but still blocked by prefetch.**
  If a high-priority consumer has reached its prefetch limit (too many unacked messages), it’s considered unable to accept more messages. The broker will then deliver to the next-highest-priority active consumers.

* **Set sensible prefetch per consumer** if you want priority to have immediate effect. For long-running tasks, `prefetch=1` is common: a high-priority worker only receives one message at a time and will get the next once it acks. For very short tasks, you can raise prefetch for throughput.

* **Global vs per-consumer prefetch**: RabbitMQ uses per-consumer prefetch by default (non-global `basic.qos`). If you set a channel-global prefetch (`global=true`), that also interacts—both limits are enforced.

Takeaway: tune `prefetch` and priority together. High priority without enough headroom (prefetch) will not help if you leave prefetch at zero (or too low) for your workload.

---

# 5 — Interaction with Single Active Consumer (SAC) & Quorum queues

RabbitMQ’s Single Active Consumer mode (SAC) and quorum queues introduce special behavior around which consumer is active.

* **SAC**: Exactly one consumer at a time receives messages. Consumer priority influences who becomes the active consumer.

  * If a higher-priority consumer joins while a lower-priority is active, RabbitMQ will **preempt** the active consumer: it will stop delivering new messages to the lower-priority consumer and switch delivery to the higher-priority consumer **after current in-flight messages are acked**.
  * This preemption is safe: it waits for in-flight messages to be processed to avoid duplicating work.
* **Quorum queues**: similar semantics apply for leader changes — consumer priority can be used to prefer consumers connected to the leader node to avoid cross-node traffic.

Operationally: with SAC + priority, expect a momentary shift in which consumer receives messages when a higher-priority consumer connects; in-flight messages are allowed to finish.

---

# 6 — Consumer priority vs message priority — they are different

Do not confuse:

* **Consumer priority** (`x-priority`) — decides *which consumer* receives messages.
* **Message priority** (priority queues, e.g., `x-max-priority`) — decides *which messages* get delivered first within a queue.

They operate on different axes and can be used together, but they are independent mechanisms.

Use consumer priority to prefer particular workers; use message priority to prioritize some messages over others.

---

# 7 — Why use consumer priority? Real use cases

1. **Hardware heterogeneity**
   Prefer faster machines: give them higher priority so they handle most traffic, while slower machines are fallback.

2. **Data locality / cluster topology**
   Prefer consumers connected to the queue’s master node (less replication/network overhead).

3. **Hot standby / graceful takeover**
   Keep lower-priority standby consumers ready; when a high-priority primary comes online it will receive messages first.

4. **Critical vs best-effort consumers**
   Let critical consumers have priority, while best-effort ones process leftover load.

5. **Zero-downtime rolling upgrades**
   Newer version consumers could be given priority; once stable you can demote or remove old ones.

---

# 8 — Implementation details / internals (what RabbitMQ does internally)

* RabbitMQ maintains consumer lists per queue and keeps track of `x-priority` values.
* At dispatch time it computes the highest `x-priority` among *active* consumers and routes deliveries only to those consumers.
* Efficiency: the scheduler selects consumers at dispatch time; the extra scheduling logic is lightweight (integer comparisons + the same round-robin among highest-priority consumers).
* If a consumer goes away or is blocked, the set of active consumers is recomputed and deliveries shift accordingly.

---

# 9 — Performance & scaling considerations

* **Overhead:** negligible for normal numbers of consumers. The decision is a simple integer comparison and selection.
* **Scale limits:** the feature scales well to many queues and consumers, but remember bindings/consumers themselves are resources: thousands of long-lived consumers have memory/cpu cost.
* **Combined with prefetch:** mis-tuned prefetch can mask or negate priority benefits—monitor unacked counts and processing latency.
* **Extreme values:** using extremely large priority ranges is unnecessary; a small integer scheme (e.g., -1, 0, 1..10) is more maintainable.

---

# 10 — Operational guidance: metrics to monitor

When running consumer priority in production, keep an eye on:

* **Consumer counts per queue** — number of consumers and their priorities.
* **Unacked messages per consumer** — reveals if a high-priority consumer is backlogged.
* **Message rate per consumer** — confirms which consumers are actually receiving work.
* **Queue depth** — overall backlog.
* **Consumer cancellations / reconnects** — abrupt changes in consumers could lead to priority shifts.
* **Network and node locality** — if you rely on locality, monitor where consumers are connected.

Management UI: you can see consumer connections and tags; add custom instrumentation to export consumer priority and throughput to Prometheus/Grafana for dashboards.

---

# 11 — Testing and validation checklist (how to verify behavior)

**Basic functional tests**

1. Create a queue.
2. Start two consumers:

   * A: `x-priority = 10`
   * B: `x-priority = 0`
3. Publish a stream of messages. Observe that A receives all messages until it’s blocked (e.g., prefetch reached) then B receives remaining ones.

**Prefetch interplay**

1. Set A prefetch to 1, B prefetch to 1.
2. Publish many messages. Confirm A receives one at a time; B only gets messages if A is slow/blocked.

**Failover test**

1. Start B (low priority) so it gets nothing.
2. Stop A; observe B receives messages (tests fallback).
3. Restart A; verify A takes over again (preemption) after A becomes ready.

**SAC / quorum behavior**

1. Enable Single Active Consumer on queue.
2. Start low-priority consumer; then start high-priority consumer and verify that after in-flight messages are acked, high-priority consumer becomes active.

**Edge tests**

* Simulate network slowness on a high-priority consumer and validate messages go to lower-priority ones when the high-priority consumer cannot keep up.

---

# 12 — Best practices and recommendations

* **Be explicit**: set `x-priority` where it matters. Don’t rely on implicit defaults.
* **Use small, well-documented priority ranges**: e.g., 0 = normal, 10 = preferred, -10 = backup.
* **Tune prefetch per consumer**: higher priority consumers usually should have prefetch tuned to their processing model.
* **Avoid starvation**: ensure lower-priority consumers are not permanently starved if that’s unacceptable for your use case (give them occasional access or use negative priorities carefully).
* **Graceful backoff for re-subscribe**: if many consumers restart simultaneously, use jittered exponential backoff to avoid thundering-herd effects and priority storms.
* **Instrumentation**: export per-consumer metrics (priority, in-flight, rate) to detect anomalies.
* **Idempotency**: always build consumer handlers idempotent since restarts/failover can cause redelivery.
* **Document topology**: consumers & their priorities should be documented so operators know which processes are preferred.

---

# 13 — Pitfalls and gotchas

* **Assuming priority overrides prefetch** — it doesn’t. A high-priority consumer with full prefetch is effectively inactive.
* **Expecting instantaneous takeover on connect** — switch happens after in-flight messages are acked; not instantaneous.
* **Mixing Single Active Consumer and consumer priority without testing** — semantics can be subtle; test leader switches and in-flight behavior.
* **Thinking priority equals absolute throughput** — it shapes dispatch but consumer processing speed, CPU, I/O and prefetch govern real throughput.
* **Not monitoring** — you might misattribute lack of messages to priority rather than consumer crashes/blocks.

---

# 14 — Example scenarios and code snippets

### 14.1 Python (pika) worker with priority and prefetch

```python
import pika
import time

def worker(priority, prefetch):
    params = pika.ConnectionParameters('localhost')
    conn = pika.BlockingConnection(params)
    ch = conn.channel()
    ch.queue_declare(queue='jobs', durable=True)

    # Prefetch tuning: per consumer
    ch.basic_qos(prefetch_count=prefetch)

    args = {'x-priority': priority}

    def callback(ch, method, properties, body):
        print(f"Consumer (priority={priority}) got: {body}")
        time.sleep(1)   # simulate work
        ch.basic_ack(delivery_tag=method.delivery_tag)

    ch.basic_consume(queue='jobs', on_message_callback=callback,
                     auto_ack=False, arguments=args)
    print("Started consumer", priority)
    ch.start_consuming()

# start two processes: worker(10, 1) and worker(0,1)
```

### 14.2 Java snippet (priority argument)

```java
Map<String,Object> args = new HashMap<>();
args.put("x-priority", 10);
channel.basicConsume("jobs", false, "", false, false, args, new DefaultConsumer(channel) {
    @Override
    public void handleDelivery(String consumerTag, Envelope env, AMQP.BasicProperties props, byte[] body) {
        // process and ack
    }
});
```

---

# 15 — How to observe & debug in the Management UI

* Look at **Queues → Consumers**: you can see consumers and their connected channels. Some UIs/plugins will show consumer arguments (including `x-priority`) if the client library sets them.
* Watch **message rates by consumer**: verify the high-priority consumers get the majority of messages.
* Inspect **unacked** counts per consumer to see if a high-priority consumer is blocked.
* Check **broker logs** for consumer cancel, connection drops or queue deletions that might alter dispatch behavior.

---

# 16 — Operational checklist before enabling consumer priority in production

1. Inventory consumers and intended priority mapping.
2. Design consistent priority integer scheme (documented).
3. Tune prefetch for each consumer class.
4. Add monitoring for consumer throughput and unacked messages.
5. Test failover, preemption, and leader-change scenarios in staging.
6. Ensure consumers are idempotent and have backoff/retry logic for re-subscribing.
7. Roll out gradually: enable priority for a small set, validate behavior, then expand.

---

# 17 — When NOT to use consumer priority

* If all consumers have roughly equal capability and you want perfect fairness—use prefetch tuning and equal priority (default).
* If you need strict message ordering across different consumers—priority may change which consumer gets a message and can affect ordering semantics.
* If you rely strongly on predictable per-consumer throughput and wish to avoid any dispatch bias—do not introduce different priorities.

---

# 18 — Quick reference (cheat sheet)

* **Set priority**: `arguments={'x-priority': <int>}` in `basic.consume`.
* **Default priority**: `0` (if not provided).
* **Higher number = higher priority.**
* **Works with:** classic queues, quorum queues, SAC (preemption after in-flight acks).
* **Respects:** prefetch limits (consumer must be able to accept messages to be active).
* **Does not change:** message durability, ack semantics, or message priority.
* **Monitoring:** watch `unacked`, per-consumer rates, consumer counts.

---

# 19 — Additional references & next steps

* Official RabbitMQ docs (consumer priority): [https://www.rabbitmq.com/docs/consumer-priority](https://www.rabbitmq.com/docs/consumer-priority)
* Related docs you should read alongside:

  * Prefetch / `basic.qos` — how to control in-flight messages.
  * Single Active Consumer & Quorum Queues — interactions with leader changes.
  * Exchanges & routing — how queues get messages in the first place.
  * Monitoring & metrics — for production observability.

---

## RabbitMQ — Consumer Priority (Q\&A)

1. **Q:** What is consumer priority?
   **A:** A feature that lets you assign an integer (`x-priority`) to a consumer so RabbitMQ prefers delivering messages to higher-priority consumers attached to the same queue.

2. **Q:** How do you set consumer priority?
   **A:** Pass the `x-priority` argument in the `basic.consume` call (client libraries expose this via `arguments`/options).

3. **Q:** What value do consumers default to if `x-priority` is not provided?
   **A:** Default priority is `0`.

4. **Q:** Do higher numbers mean higher or lower priority?
   **A:** Higher numbers mean **higher** priority (e.g., 10 > 0).

5. **Q:** When RabbitMQ dispatches a message, how does it choose which consumer gets it?
   **A:** It finds the highest priority among *active* consumers and dispatches only to those consumers (round-robin among them).

6. **Q:** What is an “active” consumer?
   **A:** An active consumer is one currently able to accept deliveries (not blocked by prefetch, network backpressure, etc.).

7. **Q:** If two consumers share the same top priority, how are messages distributed?
   **A:** Messages are distributed round-robin between those equal-priority active consumers.

8. **Q:** What happens to lower-priority consumers?
   **A:** They only receive messages when no active consumer with a higher priority can accept more messages.

9. **Q:** Does consumer priority persist if a consumer disconnects and later reconnects?
   **A:** The priority is set at each `basic.consume`. On reconnect you must set it again when you re-register the consumer.

10. **Q:** How does prefetch (`basic.qos`) interact with priority?
    **A:** Prefetch can block a high-priority consumer (too many unacked messages). If blocked, deliveries go to the next highest active consumers. So tune prefetch with priority.

11. **Q:** Can priority cause starvation of low-priority consumers?
    **A:** Yes — if high-priority consumers are always active and able to accept messages, low-priority consumers may get nothing. Design priorities and prefetch to avoid undesired starvation.

12. **Q:** Is consumer priority the same as message priority?
    **A:** No. Consumer priority selects *which consumer* gets messages. Message priority (priority queues) selects *which messages* are delivered first.

13. **Q:** Can consumer priority be negative?
    **A:** Yes — priorities are integers; negative values are valid. Just be consistent in your scheme.

14. **Q:** How do Single Active Consumer (SAC) and quorum queues interact with consumer priority?
    **A:** With SAC, consumer priority can determine which consumer becomes active; if a higher-priority consumer joins, it can preempt the active one (after in-flight messages are acked).

15. **Q:** Does consumer priority add significant performance overhead?
    **A:** No — the dispatch logic is lightweight (integer comparisons and selection). Overhead is minimal.

16. **Q:** What are common use cases for consumer priority?
    **A:** Favoring faster machines, preferring consumers on queue master nodes (data locality), hot-standby consumers, or prioritizing critical services.

17. **Q:** If a high-priority consumer is slow, will messages still go to it?
    **A:** Only if it’s active (i.e., not blocked by prefetch). If it’s blocked, messages go to lower-priority active consumers.

18. **Q:** How do you test consumer priority behavior?
    **A:** Start multiple consumers with different `x-priority` values, publish messages, observe which consumers receive messages; test with prefetch limits, consumer crashes and restarts to validate behavior.

19. **Q:** How should you tune prefetch for priority consumers?
    **A:** Give high-priority consumers a prefetch that matches expected workload (e.g., 1 for long tasks). Ensure they have enough headroom to act on their priority.

20. **Q:** Can consumer priority be set per channel or global?
    **A:** Priority is a consumer argument during `basic.consume` (per consumer). Prefetch has `global` semantics, but `x-priority` is per consumer.

21. **Q:** Do management UI or metrics show consumer priorities?
    **A:** Some management views show consumer arguments; you can also instrument/export custom metrics to include priority information.

22. **Q:** Will a new higher-priority consumer immediately take over shipments?
    **A:** The broker stops delivering new messages to lower-priority consumers and starts to the higher one — but in-flight messages already delivered are allowed to be processed/acked first.

23. **Q:** How does consumer cancellation affect priority?
    **A:** If high-priority consumers are cancelled or disconnected, lower-priority consumers become eligible and will receive messages.

24. **Q:** Can consumer priority be used to implement graceful upgrades?
    **A:** Yes — start new version consumers with higher priority so they receive traffic first; once steady, demote or remove old consumers.

25. **Q:** Is there a recommended priority numbering scheme?
    **A:** Use a small, documented set (e.g., `10` = preferred, `0` = normal, `-10` = backup). Keep it simple for maintainability.

26. **Q:** What are the risks of mis-using consumer priority?
    **A:** Starvation of low-priority consumers, unexpected routing changes, and complexity in understanding who processes what. Monitor and test carefully.

27. **Q:** Does consumer priority affect message durability or ack semantics?
    **A:** No. It only affects dispatch decision; durability and acknowledgments are independent.

28. **Q:** Should consumers be idempotent when using priority?
    **A:** Yes — always design consumers to be idempotent because re-delivery and failover can happen.

29. **Q:** How do you avoid a “resubscribe storm” when many consumers reconnect with priority?
    **A:** Use exponential backoff with jitter on reconnect/resubscribe to avoid thundering-herd problems and priority thrashing.

30. **Q:** Quick checklist before enabling consumer priority in production?
    **A:** Document priority mapping, tune prefetch per consumer type, test failover/preemption, add monitoring (per-consumer rates & unacked), ensure idempotency, and roll out gradually.

---

