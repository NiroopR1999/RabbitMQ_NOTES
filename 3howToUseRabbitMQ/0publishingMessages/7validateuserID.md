## 🛡️ What Is Validated User-ID?

The **Validated User-ID** feature in RabbitMQ ensures that if a publisher sets the `user-id` property in a message, RabbitMQ will validate that this `user-id` matches the authenticated username of the connection. This helps confirm the identity of the message publisher.&#x20;

---

## 🔐 How It Works

* **When the `user-id` is set**: RabbitMQ compares the `user-id` property in the message with the username of the connection. If they match, the message is accepted; if not, it is rejected.&#x20;

* **When the `user-id` is not set**: If the `user-id` is not set by the publisher, RabbitMQ does not perform any validation, and the publisher's identity remains private.&#x20;

---

## 🧪 Example in Java

Here's how you can set the `user-id` in a Java publisher:

```java
AMQP.BasicProperties properties = new AMQP.BasicProperties();
properties.setUserId("guest");
channel.basicPublish("amq.fanout", "", properties, "test".getBytes());
```



This message will only be published successfully if the authenticated user is "guest".&#x20;

---

## 🛡️ Additional Layer of Authentication

If security is a serious concern, it's recommended to combine the use of this feature with **TLS-enabled connections**, possibly with **peer certificate chain verification** of clients performed by the server. ([RabbitMQ][1])

---

## 👤 Special Cases: the Impersonator Tag

Occasionally, it may be useful to allow an application to forge a `user-id`. To permit this, the publishing user can have its `impersonator` tag set. By default, no users have this tag set. In particular, the `administrator` tag does not allow this. ([RabbitMQ][1])

---

## 🌐 Federation Interactions

The federation plugin can deliver messages from an upstream on which the `user-id` property is set. By default, it will clear this property (since it has no way to know whether the upstream broker is trustworthy). If the `trust-user-id` property on an upstream is set, then it will pass the `user-id` property through from the upstream broker, assuming it to have been validated there. ([RabbitMQ][1])

---

### **1. What is the `validated-user-id` feature in RabbitMQ?**

✅ **Answer:**
It ensures that if a publisher sets the `user-id` property in a message, RabbitMQ checks that it matches the authenticated username of the connection. If it matches, the message is accepted; otherwise, it is rejected.

---

### **2. Why is `validated-user-id` important?**

✅ **Answer:**
It adds an **extra layer of security** to confirm that messages are really coming from the user they claim to be from.

---

### **3. What happens if the `user-id` property is not set in the message?**

✅ **Answer:**
RabbitMQ does not validate anything; the publisher's identity remains private, and the message is accepted without `user-id` verification.

---

### **4. Can any user forge a `user-id`?**

✅ **Answer:**
No. Only users with the **impersonator tag** can set a `user-id` that doesn’t match their username. By default, no users have this tag.

---

### **5. Does the `administrator` tag allow impersonation?**

✅ **Answer:**
No. The `administrator` tag does not grant permission to forge a `user-id`.

---

### **6. How do you set the `user-id` in a message (example in Java)?**

✅ **Answer:**

```java
AMQP.BasicProperties properties = new AMQP.BasicProperties();
properties.setUserId("guest");
channel.basicPublish("amq.fanout", "", properties, "test".getBytes());
```

The message will only be accepted if the authenticated connection is also `guest`.

---

### **7. What happens if the `user-id` doesn’t match the connection username?**

✅ **Answer:**
RabbitMQ rejects the message and it is **not delivered** to the exchange or queues.

---

### **8. Can this feature be used with TLS?**

✅ **Answer:**
Yes. For better security, `validated-user-id` can be combined with **TLS connections** and peer certificate verification.

---

### **9. How does federation affect `user-id`?**

✅ **Answer:**
Federation plugins usually clear the `user-id` property from upstream messages.
If the upstream is trusted (`trust-user-id=true`), the property is passed along assuming validation happened upstream.

---

### **10. Is `validated-user-id` enabled by default?**

✅ **Answer:**
No. You must explicitly set the `user-id` property in messages. RabbitMQ only validates it when it exists.

---

### **11. Does this feature apply to consumers?**

✅ **Answer:**
No. It only affects **publishers** (message producers).

---

### **12. Can this feature prevent message spoofing?**

✅ **Answer:**
Yes, it prevents unauthorized users from sending messages claiming to be someone else.

---

### **13. What if I want multiple users to publish on behalf of one identity?**

✅ **Answer:**
You can use the **impersonator tag** for those users to allow it.

---

### **14. Does RabbitMQ modify the `user-id` automatically?**

✅ **Answer:**
No. RabbitMQ only **validates** it; it does not modify it unless using federation plugins with `trust-user-id`.

---

### **15. Can persistent messages use `validated-user-id`?**

✅ **Answer:**
Yes. Message persistence is independent; validation still applies to all messages.

---

### **16. How do I enable `validated-user-id` in Python using `pika`?**

✅ **Answer:**

```python
import pika

properties = pika.BasicProperties(user_id='guest')
channel.basic_publish(exchange='logs', routing_key='', body='Hello', properties=properties)
```

RabbitMQ checks that the connection username is also `guest`.

---

### **17. What if a message fails validation?**

✅ **Answer:**
The message is **rejected**, and it is **not routed** to any queues or exchanges.

---

### **18. Why not just trust the `user-id` header from clients?**

✅ **Answer:**
Client headers can be **spoofed**. Validation ensures the `user-id` truly matches the authenticated connection.

---

### **19. Does this feature affect performance?**

✅ **Answer:**
Slightly, because RabbitMQ checks each message’s `user-id` against the connection username.

---

### **20. Is `validated-user-id` useful in multi-tenant systems?**

✅ **Answer:**
Yes. It ensures tenants cannot send messages claiming to be another tenant.

---

### **21. How does RabbitMQ handle `user-id` in clustered setups?**

✅ **Answer:**
Validation is performed **per node** on the connection handling the publish. Each node enforces the check.

---

### **22. Can this feature be bypassed by admin users?**

✅ **Answer:**
No. Even administrators cannot forge `user-id` unless explicitly given the impersonator tag.

---

### **23. Can you use `validated-user-id` with fanout exchanges?**

✅ **Answer:**
Yes. It validates the publisher identity **before the message is sent** to all bound queues.

---

### **24. Is `validated-user-id` compatible with AMQP 0-9-1 clients?**

✅ **Answer:**
Yes, as long as the client library supports setting the `user-id` property in `BasicProperties`.

---

### **25. Can the feature be used with auto-generated queues?**

✅ **Answer:**
Yes. Queue type doesn’t matter. Validation happens at publish time.

---

### **26. Does RabbitMQ store the `user-id` in the queue?**

✅ **Answer:**
No. It only validates on publish. Queues just receive the message.

---

### **27. Can the publisher see validation failures?**

✅ **Answer:**
Yes. The client library typically raises an exception or error if the message is rejected.

---

### **28. Can `user-id` validation help with auditing?**

✅ **Answer:**
Yes. It ensures that messages in logs or queues truly originate from the claimed user.

---

### **29. Is it safe to use without TLS?**

✅ **Answer:**
It adds some protection, but without TLS, the connection can be intercepted, so validation is more secure when combined with TLS.

---

### **30. Summary**

✅ **Answer:**

* `validated-user-id` ensures publisher identity matches connection username.
* Prevents spoofing and adds security.
* Can be combined with impersonator tag and federation.
* Works with all exchanges and queues, independent of message persistence.

---
