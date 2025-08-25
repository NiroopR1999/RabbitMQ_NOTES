# RabbitMQ — Consumers (detailed notes)

## 1) What is a Consumer?

A **consumer** is a subscription for message delivery — an entity that registers interest with the broker and receives messages from a queue. In RabbitMQ terminology a consumer is specifically the client-side subscription that the server pushes messages to; the same general idea is sometimes called a “subscription” in other systems. The consumer must be registered (e.g. via `basic.consume`) before deliveries begin, and the application can cancel the subscription when it wants to stop receiving messages. ([RabbitMQ][1])

---

## 2) Two ways to receive messages: `basic.consume` vs `basic.get`

* **`basic.consume` (push model)**
  The server *pushes* messages to your consumer as they become available. This is the common, recommended mode for most workloads: you register a callback/consumer and the broker calls it for each delivery. It’s efficient for continuous processing. ([RabbitMQ][1], [CloudAMQP][2])

* **`basic.get` (pull model / polling)**
  The client explicitly asks for a single message at a time. This is simpler for a synchronous or ad-hoc processing script, but is less efficient and scales poorly compared to push-based consumers. Use `basic.get` only for simple scripts or admin tools. ([CloudAMQP][2])

---

## 3) Registering a consumer (`basic.consume`) — consumer tag & cancellation

When you register a consumer you get a **consumer tag**, an identifier used to cancel or manage the subscription. Applications can cancel a consumer using that tag. The server may also cancel a consumer (for example, queue deletion, exclusive consumer conflict, or administrative actions) — clients must be prepared to handle `consumer.cancel` notifications. Handling cancellation gracefully is a key part of robust consumer design. ([RabbitMQ][1])

---

## 4) Acknowledgements — why they exist and how to use them

* **Purpose:** acknowledgements prevent message loss by letting the broker know when a consumer has successfully processed a message. If a consumer dies or a channel/connection is closed before acking, RabbitMQ will requeue the unacknowledged message for delivery to another consumer. ([RabbitMQ][3])

* **Modes:**

  * `auto_ack=True` (no acks): the broker considers the message delivered immediately — fast, but if a consumer crashes after receiving a message but before processing, that message is lost.
  * Manual acknowledgements (`basic.ack`, `basic.nack`, `basic.reject`): consumer explicitly tells the broker when processing succeeds (or fails and whether to requeue). Manual ack is the default recommended mode for reliability. ([RabbitMQ][3])

* **Multi-ack / bulk ack:** clients can ack multiple messages at once for throughput efficiency (e.g., `multiple=True`), but you must understand the effect on potential re-delivery semantics. ([RabbitMQ][3])

---

## 5) Prefetch / QoS and how consumers are rate-limited

* **Prefetch (basic.qos):** controls how many messages a consumer can have "in-flight" (i.e., that have been delivered but not yet acknowledged). A prefetch of `1` means the server will not deliver another message to that consumer until it acks the previous one — useful for longer tasks to ensure fair distribution. `basic.qos` is applied at the channel or consumer scope depending on the client. ([RabbitMQ][4])

* **Why prefetch matters:** without it, RabbitMQ may send many messages to a fast-connection consumer, starving other workers. Setting prefetch appropriately improves fairness and throughput for worker pools. Prefetch affects throughput and memory; tuning is important. ([RabbitMQ][4])

---

## 6) Consumer behavior and dispatching rules

* **Round-robin per queue:** if multiple consumers subscribe to the same queue, RabbitMQ distributes deliveries among them (effectively round-robin, subject to prefetch and priorities). If you want every consumer to receive every message, use exchanges + multiple queues (e.g., fanout), not multiple consumers on the same queue. ([RabbitMQ][1], [Stack Overflow][5])

* **Consumer priorities:** you can assign priorities to consumers (via `x-priority`); higher priority consumers receive messages first while they are active; when they disconnect or stop, messages go to lower-priority consumers. This helps implement hot-standby or prioritized workers. ([RabbitMQ][6])

* **Exclusive consumers:** a consumer can declare itself *exclusive* for a queue, preventing other consumers from subscribing. This is handy when a queue must be processed by only one consumer at a time.

---

## 7) Consumer cancellation & notifications

The server may cancel consumers (e.g., when a queue is deleted, an exclusive consumer from another connection takes over, or management commands are applied). Clients receive a cancellation notification and must react — usually by cleaning up resources, optionally re-declaring the queue and re-registering the consumer after resolving the cause. Being prepared for cancellation is essential for robust long-running consumers. ([RabbitMQ][1])

---

## 8) Requeueing on failure and unacked messages

If a consumer dies or its channel/connection closes while it has outstanding (unacked) messages, RabbitMQ will requeue those messages (or deliver them to other consumers) so work isn't lost. This is why manual acks plus idempotent processing or deduplication logic in consumers is important. The reliability guide explains these semantics in depth. ([RabbitMQ][7])

---

## 9) Consumer reconnect and recovery

* **Client responsibility:** after a disconnect, clients must re-establish their connection and re-register consumers (unless the client library supports automatic recovery). Many client libraries offer automatic recovery features that re-open channels, re-declare queues/exchanges, and re-subscribe consumers — but you should test and understand exactly how your library performs recovery. ([RabbitMQ][7])

* **Server-side re-registration on failover:** for mirrored/quorum queues, if a node fails, consumers connected to other nodes may be re-registered automatically when a new leader is elected — the broker helps with recovery in that scenario, which reduces client-side complexity. Still, design your application to detect duplicates and be idempotent. ([RabbitMQ][7])

---

## 10) Flow-control, connection blocking and how consumers help

When the broker is under resource pressure it may block publishers (connection-blocked) but *consumers continue to be allowed to receive messages* to help drain queues and free resources. That behavior helps the broker recover without dropping messages. Consumers therefore play a part in broker health management: keeping consumers running and able to process messages is key to avoiding prolonged backpressure. ([RabbitMQ][7])

---

## 11) Practical patterns & best practices for consumers

**a. Use manual acknowledgements for reliable processing.**
Ack after processing completes (not on receipt). If processing fails, `nack`/`reject` and decide whether to requeue or send to a dead-letter exchange. ([RabbitMQ][3])

**b. Tune prefetch to balance throughput & fairness.**
Start with `prefetch_count=1` for long-running tasks; increase for short tasks. Measure and iterate. ([RabbitMQ][4])

**c. Make consumers idempotent.**
Because messages can be redelivered (after consumer failure), design consumers so repeated delivery won’t corrupt system state.

**d. Handle consumer cancellation and disconnects.**
Register callback handlers to clean up and to re-subscribe if appropriate.

**e. Use consumer priorities for graceful takeover.**
Set `x-priority` to prefer local or primary consumers; fallback consumers with lower priority remain ready. ([RabbitMQ][6])

**f. Avoid heavy initialization on message thread.**
Do expensive setup outside the message handler or do it lazily and reuse resources where safe.

**g. Monitor consumer lag and queue lengths.**
If queues grow, consumers are not keeping up. Use Management UI or metrics to detect and scale consumers. ([RabbitMQ][8])

---

## 12) Example: Python consumer (pika, BlockingConnection)

Begin with a simple, reliable consumer pattern: manual ack, graceful shutdown, prefetch tuning.

```python
import pika
import signal
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Ensure queue exists
channel.queue_declare(queue='task_queue', durable=True)

# Prefetch 1 for fair dispatch
channel.basic_qos(prefetch_count=1)

def callback(ch, method, properties, body):
    try:
        print("Received:", body)
        # simulate work
        do_work(body)
        # ack after successful processing
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception as e:
        # optionally requeue or dead-letter
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)

channel.basic_consume(queue='task_queue', on_message_callback=callback)

print('Waiting for messages. To exit press CTRL+C')
try:
    channel.start_consuming()
except KeyboardInterrupt:
    channel.stop_consuming()
    connection.close()
    sys.exit(0)
```

Notes:

* `basic_qos(prefetch_count=1)` keeps one unacked message per consumer, enabling fair distribution.
* `durable=True` on the queue + persistent messages improves survivability across broker restarts. ([RabbitMQ][4])

---

## 13) Advanced consumer features

**Consumer priorities** (`x-priority`): set during `basic.consume` to bias which consumers receive messages when multiple consumers are present. Higher values mean higher priority. Useful for hot/standby scenarios. ([RabbitMQ][6])

**Exclusive consumers:** declare exclusive consumer so that only one consumer can consume from the queue at a time — helpful for singleton processing tasks.

**Prefetch per consumer vs per channel:** Depending on your client and channels, `basic.qos` may apply to the channel; if you consume multiple queues on one channel, be careful with how prefetch is interpreted. ([RabbitMQ][4])

---

## 14) Failure modes you must plan for

* **Consumer crashes before ack:** message redelivered to another consumer.
* **Network partition:** client must reconnect and possibly re-declare consumer.
* **Broker node failure:** consumers connected to failed node must reconnect; for quorum/mirrored queues the broker may re-register consumers automatically on leader change—still design for duplicates. ([RabbitMQ][7])

---

## 15) Monitoring & debugging consumers

Use the Management UI to inspect:

* Consumer counts per queue
* Unacked message counts (indicates consumers have messages in-flight)
* Consumer connection details (node, channel)
* Consumer cancel notifications / server logs for reasons why consumers were cancelled

Track metrics and alert when queue length grows beyond thresholds; that indicates consumers are under-provisioned or slow. ([RabbitMQ][8])

---

## 16) When to use many consumers vs many queues

* **Work queues (load balancing):** one queue + many consumers — messages are split among consumers; good for parallel processing of tasks. ([Stack Overflow][5])
* **Publish/Subscribe (fanout):** one exchange → many queues → consumers each get all messages. Use multiple queues if every consumer must see every message. ([RabbitMQ][1])

---

## 17) Summary: checklist for building solid consumers

1. Use **`basic.consume`** for continuous processing. ([CloudAMQP][2])
2. Prefer **manual acknowledgements** and make processing idempotent. ([RabbitMQ][3])
3. Tune **prefetch** to match task duration and throughput needs. ([RabbitMQ][4])
4. Handle **consumer cancel** and **connection recover** logic. ([RabbitMQ][1])
5. Monitor **unacked messages**, **queue length**, and **consumer count** and scale consumers accordingly. ([RabbitMQ][8])

---

## 18) Links to the official docs (most relevant)

* Consumers (main conceptual page). ([RabbitMQ][1])
* Consumer prefetch (QoS). ([RabbitMQ][4])
* Consumer acknowledgements & publisher confirms (data safety). ([RabbitMQ][3])
* Reliability guide (recovery, re-registration). ([RabbitMQ][7])
* Consumer priorities. ([RabbitMQ][6])

---

## RabbitMQ Consumers — Q\&A (30)

1. **Q: What is a “consumer” in RabbitMQ?**
   **A:** A consumer is the client-side subscription that the broker pushes messages to. You register it (usually with `basic.consume`) and the broker delivers messages from a queue to that subscription.

2. **Q: What’s the difference between `basic.consume` and `basic.get`?**
   **A:** `basic.consume` uses a **push model** (server pushes messages to you—recommended). `basic.get` is a **pull model** (client polls for one message at a time—useful for ad-hoc scripts).

3. **Q: What is a consumer tag?**
   **A:** A consumer tag is an identifier (returned by `basic.consume`) used to manage or cancel that subscription later.

4. **Q: What does `auto_ack=True` (auto-ack) do?**
   **A:** It tells RabbitMQ the message is considered delivered as soon as it’s sent to the consumer — no explicit ack required. Fast but unsafe if the consumer crashes during processing.

5. **Q: What are manual acknowledgements and why use them?**
   **A:** Manual acks (`basic.ack`) let the consumer tell RabbitMQ when processing succeeded. If the consumer dies before acking, the message is requeued — safer for reliable processing.

6. **Q: What’s `basic.nack` and `basic.reject`?**
   **A:** They let consumers reject a message. `basic.nack` supports rejecting multiple messages and can ask RabbitMQ to requeue or drop; `basic.reject` rejects a single message with an option to requeue.

7. **Q: What is “unacked” and why monitor it?**
   **A:** “Unacked” messages are delivered but not yet acknowledged. High unacked counts mean consumers are processing slowly or stuck — a sign to scale or investigate.

8. **Q: What is prefetch (QoS / `basic.qos`)?**
   **A:** Prefetch controls how many messages the server will deliver to a consumer at once before waiting for acks. `prefetch_count=1` is common for long tasks to ensure fairness.

9. **Q: Is prefetch applied per consumer or per channel?**
   **A:** It can be per channel or per consumer depending on the `global` flag of `basic.qos`. Many client libs apply it per consumer/channel—check your client docs.

10. **Q: Why set `prefetch_count=1` for workers?**
    **A:** To ensure a worker only receives one task at a time until it finishes and acks—prevents one fast worker from hogging many messages.

11. **Q: What is an exclusive consumer?**
    **A:** An exclusive consumer is the only consumer allowed on that queue — RabbitMQ will reject any other subscribe attempts for that queue.

12. **Q: What’s the difference between an exclusive consumer and an exclusive queue?**
    **A:** *Exclusive consumer* = only one consumer subscription for the queue. *Exclusive queue* = queue can only be used by the connection that declared it and is deleted when that connection closes.

13. **Q: Can the server cancel a consumer? Why?**
    **A:** Yes. The server can cancel consumers because of queue deletion, exclusive consumer conflicts, admin actions, or resource constraints. Clients must handle cancellation notifications.

14. **Q: What should a consumer do on cancellation?**
    **A:** Clean up local resources, optionally re-declare the queue or subscription (after investigating cause), and log/alert so you can act if it’s unexpected.

15. **Q: What happens to unacked messages if a consumer crashes?**
    **A:** RabbitMQ will requeue them (or deliver to another consumer) so work is not lost.

16. **Q: How do you make consumers idempotent and why?**
    **A:** Design handlers so repeated processing has no adverse effects (use dedup keys, transactional updates). Idempotency is essential because messages can be redelivered.

17. **Q: What is consumer priority (`x-priority`)?**
    **A:** A binding/consumer-level extension that lets higher priority consumers receive messages first while active; lower priority consumers get them only when higher ones are absent.

18. **Q: How should you handle long-running tasks in consumers?**
    **A:** Use `prefetch_count=1` or an appropriate value, ack only after work finishes, offload heavy CPU/IO to worker threads/processes, and avoid blocking the network thread.

19. **Q: What’s the best way to handle failures inside a consumer?**
    **A:** Catch exceptions, decide whether to `nack` with `requeue=True` (retry) or `requeue=False` (send to DLX), log the error, and avoid crashing the consumer process repeatedly.

20. **Q: What is a Dead Letter Exchange (DLX) and how do consumers use it?**
    **A:** A DLX is an exchange where rejected or expired messages are routed. Consumers can `nack` with `requeue=False` to send bad messages to a DLX for analysis or retry workflows.

21. **Q: How should consumers reconnect after network failures?**
    **A:** Use client libraries with automatic recovery if available or implement reconnect/backoff logic that re-establishes connections, re-declares topology and re-subscribes consumers.

22. **Q: What about message ordering — are consumers guaranteed order?**
    **A:** RabbitMQ preserves order *within a single queue*. However, requeueing and multiple consumers can cause reordering.

23. **Q: Do consumers affect broker flow control (connection blocked)?**
    **A:** Consumers are allowed to keep receiving messages when the broker blocks publishers—this helps drain queues and relieve pressure.

24. **Q: How do you scale consumers? Many consumers vs many queues?**
    **A:** For parallel work (load balancing), use one queue with many consumers. For pub/sub (every consumer sees every message), give each consumer its own queue (bound to a fanout/topic exchange).

25. **Q: How do channels relate to consumers?**
    **A:** Consumers typically run on channels (virtual connections). Closing a channel cancels its consumers. Don’t run many long-lived consumers on a single channel if you need isolation.

26. **Q: What should you monitor for consumer health?**
    **A:** Consumer count, connection status, unacked message count, queue depth/length, consumer cancel events, processing latency, and error rates.

27. **Q: When should you use `basic.get` (polling)?**
    **A:** Only for simple administrative scripts, testing, or rare ad-hoc tasks. For production continuous processing use `basic.consume`.

28. **Q: Can consumers process messages in parallel?**
    **A:** Yes—either run multiple consumer processes/threads (each with its own channel/consumer) or handle message processing in worker threads from a single consumer while carefully managing acks.

29. **Q: How do you avoid consumer starvation (one fast consumer getting all work)?**
    **A:** Use prefetch tuning and ensure fair dispatch (e.g., `prefetch_count=1`) so messages are distributed to workers that are ready.

30. **Q: Quick checklist for building robust consumers?**
    **A:** Use manual acks, tune prefetch, make processing idempotent, handle cancels/reconnects, monitor unacked/queue depth, and use DLX for failed messages.

---