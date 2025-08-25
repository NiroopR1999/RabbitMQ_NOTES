# Negative Acknowledgements (`basic.nack`) — Detailed Notes

## 1 — High-level overview

* AMQP 0-9-1 defines `basic.reject` for rejecting a single delivered message (optionally requeueing it). It **does not** provide for *bulk* negative acknowledgement.
* RabbitMQ adds support for `basic.nack`, which provides the same functionality as `basic.reject` **and** adds the `multiple` flag so a consumer can negatively acknowledge (reject) *many* delivered-but-unacknowledged messages at once. This makes bulk rejection efficient and matches `basic.ack`’s bulk semantics. ([RabbitMQ][1])

---

## 2 — Why `basic.nack` exists (practical motivation)

* When a consumer crashes or detects a batch of messages cannot be processed (poison messages, systematic error), you often need to reject more than one message. `basic.reject` would force one RPC per message; `basic.nack` allows grouping into a single call. This reduces client-broker round trips and simplifies consumer logic for batch-failure handling. ([RabbitMQ][1])

---

## 3 — Method signature & important flags (semantics)

`basic.nack` includes three key parameters you must understand:

1. **`delivery_tag`** — identifies a specific delivery on the channel.
   *Delivery tags are channel-scoped and monotonically increasing for deliveries on that channel.*

2. **`multiple` (boolean)** — if `false`, `basic.nack` targets only the message identified by `delivery_tag`. If `true`, it targets **all unacknowledged deliveries up to and including** `delivery_tag` (mirror of `basic.ack`’s bulk ack behaviour). This is how you perform bulk negative acknowledgements. ([RabbitMQ][1])

3. **`requeue` (boolean)** — if `true` the broker will attempt to requeue the (rejected) message(s); if `false`, the message(s) are not requeued (they will be dropped or routed to a Dead Letter Exchange (DLX) if the queue is configured to DLX such messages).

**Note:** The combination of `multiple=True` and `requeue=True` tells RabbitMQ to requeue a contiguous batch of unacked deliveries. ([RabbitMQ][1])

---

## 4 — How requeueing behaves (position in the queue)

* When a message is requeued, RabbitMQ **tries** to place it back in its original position in the queue.
* If that exact position cannot be restored (because other consumers concurrently consumed and acked messages), the broker will place the requeued message **closer to the queue head** — i.e., near the front so it will be delivered sooner than messages enqueued after it. This is important to know because exact original ordering is not strictly guaranteed in concurrent scenarios. ([RabbitMQ][1])

---

## 5 — Where `basic.nack` works

* `basic.nack` works for both **push** consumers (`basic.consume`) and **polling** (`basic.get`) consumers. You can negative-ack individual gets or bulk-nack delivered messages from a consumer. ([RabbitMQ][1])

---

## 6 — Examples

### Java (from RabbitMQ docs)

Reject a single polled message and requeue it:

```java
GetResponse gr = channel.basicGet("some.queue", false);
channel.basicNack(gr.getEnvelope().getDeliveryTag(), false, true);
```

Reject two polled messages in one call (bulk); `true` is the `multiple` flag:

```java
GetResponse gr1 = channel.basicGet("some.queue", false);
GetResponse gr2 = channel.basicGet("some.queue", false);
channel.basicNack(gr2.getEnvelope().getDeliveryTag(), true, true);
```

(These are the canonical examples from the docs.) ([RabbitMQ][1])

### Python (pika) — consumer callback examples

Single reject, requeue:

```python
def callback(ch, method, properties, body):
    try:
        process(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception:
        # Reject and ask broker to requeue this single message
        ch.basic_nack(delivery_tag=method.delivery_tag, multiple=False, requeue=True)
```

Bulk reject (reject all unacked up to this delivery tag) and send to DLX (no requeue):

```python
# Assume some condition detected that previous N messages failed
ch.basic_nack(delivery_tag=method.delivery_tag, multiple=True, requeue=False)
# If queue has DLX configured, those messages will be routed to DLX; otherwise they'll be dropped.
```

**Note:** API names vary by client library; in pika `basic_nack` exists with named args `delivery_tag`, `multiple`, `requeue`.

---

## 7 — `requeue=False` — what happens

* If you call `basic.nack(..., requeue=False)` the message(s) are **not requeued**. If the queue has a **Dead Letter Exchange (DLX)** configured for dead lettering, the broker will route the message to the DLX (with standard DLX semantics: TTLs, rejection reasons, headers can be added). If no DLX is configured, the broker will drop the message.
* Use `requeue=False` for **poison messages** you don’t want to repeatedly redeliver to the same failing consumer. Combine this with DLX to capture problems for offline inspection or replay. (Also a common pattern: increment an attempt counter in headers and after N attempts `nack(..., requeue=False)`.)

---

## 8 — Redelivery flag and how to detect retries

* When a message is re-delivered (because of consumer crash or requeue), RabbitMQ sets the **`redelivered`** flag on the delivery. Consumers can inspect this (via `method.redelivered` in many clients) to decide whether to retry processing, escalate, or route to DLX.
* Best practice: if a message arrives with `redelivered=True` and processing fails again, consider sending it to a DLX (via `nack(..., requeue=False)`) rather than endlessly requeuing it. This prevents infinite retry loops.

---

## 9 — Interplay with prefetch and in-flight messages

* `basic.nack` only affects **unacknowledged** deliveries. If you `multiple=True`, you must be aware that `delivery_tag` ordering is per-channel — bulk-nacking will include all unacked deliveries on that channel up to the tag.
* If a high prefetch is set, a consumer may have many in-flight messages. Using `multiple=True` allows rejecting them all at once, but beware: requeueing many messages can change queue ordering and cause large rework for other consumers. Use bulk nack intentionally.

---

## 10 — Interaction with other reliability features

* **Dead Letter Exchanges (DLX)**: use `requeue=False` to route bad messages to DLX for inspection or delayed retries.
* **Publisher Confirms**: independent feature — confirms tell producers the broker persisted the message; they don’t affect consumer ack/nack behaviour.
* **Quorum / mirrored queues**: `basic.nack` semantics remain the same; requeue/dropping and DLX behaviour are preserved. Be mindful of replication latency for requeued messages to be visible on other nodes.
* **Consumer Cancel Notifications**: if a consumer is cancelled by the server, unacked deliveries will be requeued and may be redelivered; you don’t need to call `basic.nack` in that failure mode (it’s automatic).

---

## 11 — Common pitfalls & gotchas

1. **Infinite redelivery loops** — if you always `nack(..., requeue=True)` on failure, the same message may be repeatedly redelivered to the same failing consumer. Solution: track attempts (in headers), escalate after N attempts by `requeue=False` → DLX.
2. **Bulk `multiple=True` misuse** — accidentally nacking too many deliveries can requeue messages that were already being processed elsewhere or that you intended to acknowledge. Be careful with delivery\_tag semantics.
3. **Assuming original position is always preserved** — RabbitMQ *tries* to reinsert at original position, but when concurrency occurs it may be placed nearer the head. Don’t rely on absolute ordering guarantees in highly concurrent consumers. ([RabbitMQ][1])
4. **Mixing channels** — delivery tags are channel-scoped. Don’t try to nack delivery tags obtained from another channel.
5. **Not using DLX** — if you `requeue=False` and have no DLX, you may lose messages (they’re dropped). Always configure DLX if you want to inspect or replay failures.

---

## 12 — Best practices & recommended patterns

* **Manual ack + explicit nack**: prefer manual acknowledgements (`auto_ack=False`). `basic.nack` is your tool to reject badly processed messages or batches.
* **Poison message handling**: implement a retry counter (in message header). On failure increment counter; after threshold, `nack(..., requeue=False)` so message goes to DLX for later inspection.
* **Use `multiple` for efficiency** where you know a contiguous range of unacked messages failed (e.g., if a consumer notices a systemic error). But use carefully.
* **Avoid busy-loop retries**: when requeuing messages, add backoff/delays (TTL + DLX or delayed-message plugin) to avoid tight redelivery loops.
* **Idempotent consumers**: always make message processing idempotent; redeliveries can happen and you must avoid side-effect duplication.
* **Monitor `redelivered` flag and unacked counts**: these metrics show problem messages and blocked consumers.

---

## 13 — Troubleshooting checklist

If you see unexpected behaviour after `nack`:

* Check **unacked** counts on the queue and per-consumer.
* Inspect the **redelivered** flag on messages to see how often they come back.
* Confirm **DLX** configuration if `requeue=False` and you expect messages to be captured.
* Verify you are calling `basic_nack` on the **same channel** where the delivery was received (delivery tags are channel-scoped).
* Review broker logs for any warnings related to message routing or resource pressure that could affect requeueing.

---

## 14 — Short reference (cheat sheet)

* `basic.reject`: Reject **one** message (AMQP core). No bulk capability.
* `basic.nack(delivery_tag, multiple, requeue)`: RabbitMQ extension.

  * `multiple=False` → single delivery (delivery\_tag).
  * `multiple=True` → all unacked deliveries up to & including `delivery_tag`.
  * `requeue=True` → attempt to requeue (original position if possible).
  * `requeue=False` → drop or DLX (if configured).
* Use `basic.nack` for bulk rejections and `basic.reject` for single-message simple cases (but `nack` supersedes `reject` in functionality).

(Primary source: RabbitMQ docs on Negative Acknowledgements.) ([RabbitMQ][1])

---

## 15 — Next steps / learning path

* Try small experiments: produce a stream of messages, write a consumer that `nack(..., requeue=True)` on first failure and observe redelivery & `redelivered` flag.
* Implement a poison-message flow: add header `x-attempts`, update it in the consumer, and on threshold `nack(..., requeue=False)` to land in a DLX queue for offline analysis.
* Measure effects of `multiple=True` by delivering a batch and bulk-nacking them to observe broker behaviour and ordering.

---


## `basic.nack` — Q\&A (30)

1. **Q: What is `basic.nack`?**
   **A:** A RabbitMQ extension to AMQP that lets a consumer negatively acknowledge (reject) one or many delivered-but-unacknowledged messages. It supports `multiple` (bulk) and `requeue` flags.

2. **Q: How is `basic.nack` different from `basic.reject`?**
   **A:** `basic.reject` rejects a **single** message only. `basic.nack` can reject one or **multiple** messages (via `multiple=True`) in one call.

3. **Q: What are the parameters of `basic.nack`?**
   **A:** `delivery_tag` (which delivery to target), `multiple` (if `true`, target all unacked deliveries up to that tag), and `requeue` (if `true` requeue, otherwise drop/send to DLX).

4. **Q: What does `requeue=True` do?**
   **A:** Attempts to return the (rejected) message(s) to the head of the queue (ideally original position; not guaranteed under concurrency). It will be redelivered to consumers.

5. **Q: What does `requeue=False` do?**
   **A:** Message(s) are **not** requeued. If the queue has a DLX configured, they go there; otherwise they are dropped.

6. **Q: What does `multiple=True` do?**
   **A:** Bulk-nacks all unacknowledged deliveries on the channel up to and including the given `delivery_tag`.

7. **Q: Are delivery tags global?**
   **A:** No — delivery tags are **channel-scoped**. Only use a delivery tag with the same channel where it was received.

8. **Q: Can I `basic.nack` messages received by `basic.get` (polling)?**
   **A:** Yes — `basic.nack` works for both `basic.get` and `basic.consume` deliveries.

9. **Q: How does RabbitMQ mark a redelivered message?**
   **A:** It sets the `redelivered` flag on the delivery so consumers can detect retries.

10. **Q: Will requeued messages preserve original ordering?**
    **A:** RabbitMQ tries to return them to the original position, but under concurrent activity it may place them nearer the head — exact original ordering is not guaranteed.

11. **Q: What happens to unacked messages if the consumer dies?**
    **A:** RabbitMQ automatically requeues them (or delivers them to other consumers) — you don’t need to call `nack` in that scenario.

12. **Q: If I `nack(..., requeue=True)` on the same failing consumer, won’t it get the same message again immediately?**
    **A:** Possibly — if that consumer remains available it may receive it again. That can cause redelivery loops unless you add logic (attempt counters, DLX) to break the loop.

13. **Q: How should I handle poison messages?**
    **A:** Track attempt counts (headers). After N failures, `nack(..., requeue=False)` so the message goes to a DLX for inspection instead of looping.

14. **Q: Is `basic.nack` atomic for multiple messages?**
    **A:** It bulk-rejects a contiguous range of unacked messages up to `delivery_tag`. It’s treated as a single operation but affects multiple deliveries.

15. **Q: Can I `nack` delivery tags from a different channel or connection?**
    **A:** No — delivery tags are specific to the channel; using one from another channel is invalid.

16. **Q: Does `basic.nack` acknowledge messages?**
    **A:** No — `nack` indicates rejection. It is **not** an ack. You must explicitly `ack` successful messages.

17. **Q: Can `basic.nack` cause message loss?**
    **A:** If `requeue=False` and no DLX is configured, messages are dropped — so yes, if you expect them preserved, configure DLX.

18. **Q: How does `basic.nack` interact with `basic.qos` (prefetch)?**
    **A:** Prefetch controls in-flight unacked messages. Bulk-nacking many messages can suddenly free prefetch slots or flood the queue with requeued messages; tune prefetch carefully.

19. **Q: Is `basic.nack` supported by all clients?**
    **A:** Most modern client libraries provide `basic_nack` (APIs differ). Check your client’s docs.

20. **Q: When should I use `multiple=True`?**
    **A:** When you detect a systemic error affecting a batch of unacked messages and want to reject them efficiently in one call.

21. **Q: If I `nack(..., requeue=False)` on multiple messages and the queue has a DLX, do all messages go to the DLX?**
    **A:** Yes — they will be dead-lettered according to the queue’s DLX configuration.

22. **Q: What signals should consumers check to avoid infinite redelivery?**
    **A:** The `redelivered` flag and an attempt counter (message header) so you can escalate to DLX after retries.

23. **Q: Are there race conditions to be aware of?**
    **A:** Yes — concurrent consumers, acks, and nacks can reorder messages; be careful when bulk-nacking and assume ordering can change.

24. **Q: How do I debug unexpected nacks or drops?**
    **A:** Check broker logs, management UI (unacked counts), consumer logs, and DLX queues. Ensure DLX is set if you expect capture of bad messages.

25. **Q: What’s a common pattern to handle transient errors?**
    **A:** `nack(..., requeue=True)` with exponential backoff/TTL-based delayed retries (TTL + DLX), then `nack(..., requeue=False)` after N attempts.

26. **Q: Will `basic.nack` affect confirmations to producers?**
    **A:** No — publisher confirms are independent. Nacks affect consumer-side delivery only.

27. **Q: Can I `nack` messages when using transactions?**
    **A:** Transactions and acks/nacks are different flows; be careful mixing. Most systems use publisher confirms and manual acks/nacks rather than AMQP transactions.

28. **Q: Does `basic.nack` work on quorum queues and mirrored queues?**
    **A:** Yes — semantics (requeue/DLX) apply; replication may affect timing but behavior is consistent.

29. **Q: Performance: is `nack` expensive?**
    **A:** Single `nack` calls are lightweight. `multiple=True` reduces round-trips and is more efficient than per-message rejects.

30. **Q: Best practices summary for using `basic.nack`?**
    **A:**

    * Use manual acks; `nack` for failures.
    * Use `multiple` for batch failures when appropriate.
    * Prefer `requeue=False` + DLX for poison messages.
    * Track redelivery attempts and avoid tight retry loops.
    * Tune prefetch and make consumers idempotent.
    * Monitor unacked/redelivered counts and test failure scenarios.

---
