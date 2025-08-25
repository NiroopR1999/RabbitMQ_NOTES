# RabbitMQ Streams – Offset Tracking (Python)

## Overview

This tutorial builds on the earlier “Hello World” stream tutorial by showing you how to manage **stream offsets** — enabling consumers to continue processing where they left off or stop at a specific marker.

---

## Prerequisites

* RabbitMQ with **stream plugin** enabled and running (default port: 5552).
* Python with the **rstream** library installed.
* Recommended: use Docker to run RabbitMQ locally with stream support:

  ```bash
  docker run -it --rm --name rabbitmq -p 5552:5552 -p 15672:15672 -p 5672:5672 \
    -e RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS='-rabbitmq_stream advertised_host localhost' \
    rabbitmq:4-management  
  docker exec rabbitmq rabbitmq-plugins enable rabbitmq_stream rabbitmq_stream_management
  ```

  ([RabbitMQ][1])

---

## What You'll Build

1. **offset\_tracking\_send.py** — Sends 100 messages to a stream, where the final message is labeled `"marker"`.
2. **offset\_tracking\_receive.py** — Consumes messages from the stream until it encounters the `"marker"`, then stops.
   This demonstrates how consumers can track their progress and resume later.
   ([RabbitMQ][1])

---

## Producer: Sending with a Marker

* Create a `Producer` and declare the stream, specifying retention (e.g., 2 GB).
* Send 100 messages in a loop.
* Use `"marker"` as the body of the last message.
* Use `asyncio.Condition` and a callback (e.g., `_on_publish_confirm_client`) to ensure all messages are confirmed before closing.
  ([RabbitMQ][1])

---

## Consumer: Reading Until Marker

* Create a `Consumer` and declare the same stream.
* Subscribe with an offset starting from the **earliest**.
* Register a callback that processes messages and checks for the `"marker"`; when seen, it stops consuming.
* This approach illustrates how a consumer can **resume from where it left off in a previous run**.
  ([RabbitMQ][1])

---

## Key Takeaways

* **Offset tracking** allows resuming consumption after restarts.
* Use markers and consumer logic to manage stopping points.
* Useful in streaming systems where continuity matters.

---

# Q\&A: Offset Tracking (Python Stream)

1. **Q: What is Offset Tracking?**
   **A:** It’s tracking a consumer’s progress through a stream so the consumer can resume from where it left off.

2. **Q: Why use a marker message?**
   **A:** It signals the consumer where to stop or consider one session complete.

3. **Q: What is `asyncio.Condition` used for in the producer?**
   **A:** To wait until all messages are confirmed as published before closing the connection.

4. **Q: Can a consumer restart from the last position it processed before?**
   **A:** Yes — by keeping track of offsets, it can resume exactly where it left off.

5. **Q: Does the offset tracking persist automatically on the server?**
   **A:** With server-side offset tracking enabled (in newer RabbitMQ versions), yes — offsets are stored within the stream.
   ([RabbitMQ][2])

6. **Q: What client library does this tutorial use?**
   **A:** The `rstream` Python client (`rstream` library).
   ([RabbitMQ][1])

7. **Q: Does the stream persist messages by default?**
   **A:** Yes, streams are always disk-based and preserve messages per retention settings.

8. **Q: How do you trigger stopping the consumer?**
   **A:** By checking message content; when `"marker"` arrives, the consumer breaks out of listening loop.

9. **Q: Is restarting offset tracking required each time?**
   **A:** Only if consumer didn’t finish processing; using the offset tracking feature, you can resume automatically.

10. **Q: Can multiple consumers use offset tracking independently?**
    **A:** Absolutely — each consumer tracks its own offset.

11. **Q: What happens if no marker is found?**
    **A:** The consumer will continue listening until its offset reaches the end of the stream.

12. **Q: Do producer confirmations guarantee delivery to the stream?**
    **A:** Yes — confirmations ensure RabbitMQ received the message before producer exits.

13. **Q: Do markers need unique bodies?**
    **A:** Yes — use a unique marker label to correctly signal stopping conditions.

14. **Q: Is offset tracking exclusive to streams?**
    **A:** Yes — classic queues do not have offset-based replay semantics.

15. **Q: Are offsets human-readable or numeric?**
    **A:** Numeric; they correspond to positions in the stream.

16. **Q: Does offset tracking increase stream size?**
    **A:** Slightly — server-side offsets consume small metadata space per tracking.
    ([RabbitMQ][2])

17. **Q: Why use a tracking consumer vs. manual reset?**
    **A:** Tracking is more efficient and avoids reprocessing messages unnecessarily.

18. **Q: Can consumers skip ahead in the stream?**
    **A:** Yes — by specifying a custom offset type (e.g., last or specific number).

19. **Q: Is offset tracking supported in all RabbitMQ clients?**
    **A:** Only in stream-supported libraries (e.g., `rstream`) and RabbitMQ versions with stream plugin.

20. **Q: Can offset tracking detect duplicates?**
    **A:** Not directly — deduplication uses producer name and publishing ID.
    ([RabbitMQ][2])

---
