# Quorum Queues — Complete Reference (RabbitMQ 4.1)

## 1. Overview

* **Definition**: Quorum Queues are a modern, **durable**, **replicated** queue type in RabbitMQ, built using the **Raft consensus algorithm**. They are recommended as the **default** choice when highly available, replicated queues are needed ([RabbitMQ][1]).
* **Why Quorum?** Designed to ensure **data safety**, **reliable and fast leader election**, and **high availability**—even during upgrades or failures ([RabbitMQ][1]).
* **Feature Highlights**: Support **poison message handling**, **at-least-once dead-lettering**, and an AMQP-level `modified` outcome ([RabbitMQ][1]).
* **Alternatives**: For use cases focusing on replication and repeatable reads, **streams** may be more suitable than queues ([RabbitMQ][1]).
* **Version Note**: Classic mirrored queues were removed starting with RabbitMQ 4.0 ([RabbitMQ][1]).

---

## 2. Replication & Membership Management

* **Default Replica Count**: Up to **3 replicas** (one per cluster node) are created when a quorum queue is declared ([RabbitMQ][1]).
* **Best Practice**: Use at least **3 nodes** for real quorum guarantees. More than a majority provides no additional availability but consumes more resources ([RabbitMQ][1]).
* **Cluster Growth**: If declared before all nodes are present, RabbitMQ applies as many replicas as nodes available. Increasing later requires operator action ([RabbitMQ][1]).
* **Node Decommissioning**: Use `forget_cluster_node` or `shrink` to remove replicas safely ([RabbitMQ][1]).
* **Safe Replacement Workflow**: Add a new node, grow replicas there, then gracefully decommission the old node ([RabbitMQ][1]).

---

## 3. Leader Node & Rebalancing

* **Queue Leader**: Each quorum queue has a **leader** that handles all operations to maintain FIFO ordering ([RabbitMQ][1]).
* **Leader Locator Strategies**:

  * `client-local` (default): Leader placed on node where client connected.
  * `balanced`: Uniformly distribute leaders across cluster (only if <1000 queues; else random).
  * Configurable via queue argument, policy, or config ([RabbitMQ][1]).
* **Rebalance Command**:

  ```bash
  rabbitmq-queues rebalance quorum
  rabbitmq-queues rebalance quorum --queue-pattern "orders.*"
  rabbitmq-queues rebalance quorum --vhost-pattern "production.*"
  ```

  This redistributes *leaders*, not replicas ([RabbitMQ][1]).

---

## 4. Continuous Membership Reconciliation (CMR)

* **CMR Feature**: Automatically ensures replica sets align with target size.
* Leaders periodically check membership and grow queues if needed. Use in addition to manual replica management ([RabbitMQ][1]).

---

## 5. Configuration Settings

* **Flow Control via `commands_soft_limit`**: Limits unconfirmed messages per channel; tune if single publisher flows are common ([RabbitMQ][1]).
* **Relaxed Redeclaration Checks**:
  Setting `quorum_queue.property_equivalence.relaxed_checks_on_redeclaration = true` allows old clients declaring classic vs quorum queues without failure for compatibility during migrations ([RabbitMQ][1]).

---

## 6. Memory, WAL & Storage

* **Memory Footprint**: Approx. **32 bytes of metadata per message**. \~1 MB used for 30,000 messages ([RabbitMQ][1]).
* **WAL Mechanism**:

  * Shared `Write-Ahead Log` per node.
  * WAL flushed to disk when size (default 512 MiB) is reached.
  * Configurable via `raft.wal_max_size_bytes` (e.g., `64000000` for 64 MiB) ([RabbitMQ][1]).
* **Memory Sizing Recommendations**: RAM ≥ 3× WAL size; 4× is safer for heavy load ([RabbitMQ][1]).
* **Disk & Performance**:

  * Quorum queues need ample spare disk space, especially with large messages.
  * Clean-up (compaction) depends on timely consumption ([RabbitMQ][1]).

---

## 7. Summary: Guarantees & Use Cases

* **Safety & Availability**: Quorum queues guarantee high data safety and failover protection.
* Use them when data durability is vital; classic queues if more features (TTL, priority, lazy) are needed.

---

## 8. Additional Insights & Use Cases

* **AWS/AMQ**: Support Raft-based leader/follower model, `x-delivery-count` header, and `delivery-limit` for poison message handling ([AWS Documentation][2]).
* **Migration & Throughput**:

  * Quorum queues provide significantly higher throughput: up to \~30,000 msg/s for 1KB messages, outperforming classic mirrored queues ([RabbitMQ][3]).
  * However, extreme backlog without consumers can cause degraded performance ([RabbitMQ][3]).
* **Consumer Priorities & SAC (v4.0+)**: Quorum queues honor consumer priority within single active consumer logic, enabling switchover to higher priority consumers when they subscribe ([RabbitMQ][4]).
* **Delivery Limit Default**: Set to 20 attempts by default. Messages dropped if there's no DLX setup; recommended to configure DLX to avoid data loss ([RabbitMQ][4]).
* **Startup Optimization (v4.0+)**: Checkpoint files reduce startup time drastically compared to full WAL replay ([RabbitMQ][4]).

---

## 9. Client Behavior: Consumer Flags

* **Exclusive Consumers Ignored**: Quorum queues ignore the `exclusive` flag. Use **Single Active Consumer (SAC)** instead ([RabbitMQ][5]).
* **SAC + Priority Behavior**:

  * With SAC enabled, the first consumer becomes active.
  * A later consumer with higher `x-priority` takes over once the current consumer ack’s its in-flight messages ([RabbitMQ][4]).
* **Consumer Activity Flag** reflects true active consumer per queue and is visible via management/CLI ([RabbitMQ][5]).

---

## 10. Migration Guidance

* **Up-to-date Migration**: Migrate via blue-green to new vhost, optionally using federation to minimize downtime ([RabbitMQ][6]).
* **Compatibility Considerations**:

  * Feature mismatches (priority queues, TTL, etc.) may require code or policy changes ([RabbitMQ][3]).
  * Tools exist to detect policies using mirrored queues and guide migration.

---

# Quorum Queues — Complete Reference (RabbitMQ 4.1)

## 1. Overview

* **Definition**: Quorum Queues are a modern, **durable**, **replicated** queue type in RabbitMQ, built using the **Raft consensus algorithm**. They are recommended as the **default** choice when highly available, replicated queues are needed ([RabbitMQ][1]).
* **Why Quorum?** Designed to ensure **data safety**, **reliable and fast leader election**, and **high availability**—even during upgrades or failures ([RabbitMQ][1]).
* **Feature Highlights**: Support **poison message handling**, **at-least-once dead-lettering**, and an AMQP-level `modified` outcome ([RabbitMQ][1]).
* **Alternatives**: For use cases focusing on replication and repeatable reads, **streams** may be more suitable than queues ([RabbitMQ][1]).
* **Version Note**: Classic mirrored queues were removed starting with RabbitMQ 4.0 ([RabbitMQ][1]).

---

## 2. Replication & Membership Management

* **Default Replica Count**: Up to **3 replicas** (one per cluster node) are created when a quorum queue is declared ([RabbitMQ][1]).
* **Best Practice**: Use at least **3 nodes** for real quorum guarantees. More than a majority provides no additional availability but consumes more resources ([RabbitMQ][1]).
* **Cluster Growth**: If declared before all nodes are present, RabbitMQ applies as many replicas as nodes available. Increasing later requires operator action ([RabbitMQ][1]).
* **Node Decommissioning**: Use `forget_cluster_node` or `shrink` to remove replicas safely ([RabbitMQ][1]).
* **Safe Replacement Workflow**: Add a new node, grow replicas there, then gracefully decommission the old node ([RabbitMQ][1]).

---

## 3. Leader Node & Rebalancing

* **Queue Leader**: Each quorum queue has a **leader** that handles all operations to maintain FIFO ordering ([RabbitMQ][1]).
* **Leader Locator Strategies**:

  * `client-local` (default): Leader placed on node where client connected.
  * `balanced`: Uniformly distribute leaders across cluster (only if <1000 queues; else random).
  * Configurable via queue argument, policy, or config ([RabbitMQ][1]).
* **Rebalance Command**:

  ```bash
  rabbitmq-queues rebalance quorum
  rabbitmq-queues rebalance quorum --queue-pattern "orders.*"
  rabbitmq-queues rebalance quorum --vhost-pattern "production.*"
  ```

  This redistributes *leaders*, not replicas ([RabbitMQ][1]).

---

## 4. Continuous Membership Reconciliation (CMR)

* **CMR Feature**: Automatically ensures replica sets align with target size.
* Leaders periodically check membership and grow queues if needed. Use in addition to manual replica management ([RabbitMQ][1]).

---

## 5. Configuration Settings

* **Flow Control via `commands_soft_limit`**: Limits unconfirmed messages per channel; tune if single publisher flows are common ([RabbitMQ][1]).
* **Relaxed Redeclaration Checks**:
  Setting `quorum_queue.property_equivalence.relaxed_checks_on_redeclaration = true` allows old clients declaring classic vs quorum queues without failure for compatibility during migrations ([RabbitMQ][1]).

---

## 6. Memory, WAL & Storage

* **Memory Footprint**: Approx. **32 bytes of metadata per message**. \~1 MB used for 30,000 messages ([RabbitMQ][1]).
* **WAL Mechanism**:

  * Shared `Write-Ahead Log` per node.
  * WAL flushed to disk when size (default 512 MiB) is reached.
  * Configurable via `raft.wal_max_size_bytes` (e.g., `64000000` for 64 MiB) ([RabbitMQ][1]).
* **Memory Sizing Recommendations**: RAM ≥ 3× WAL size; 4× is safer for heavy load ([RabbitMQ][1]).
* **Disk & Performance**:

  * Quorum queues need ample spare disk space, especially with large messages.
  * Clean-up (compaction) depends on timely consumption ([RabbitMQ][1]).

---

## 7. Summary: Guarantees & Use Cases

* **Safety & Availability**: Quorum queues guarantee high data safety and failover protection.
* Use them when data durability is vital; classic queues if more features (TTL, priority, lazy) are needed.

---

## 8. Additional Insights & Use Cases

* **AWS/AMQ**: Support Raft-based leader/follower model, `x-delivery-count` header, and `delivery-limit` for poison message handling ([AWS Documentation][2]).
* **Migration & Throughput**:

  * Quorum queues provide significantly higher throughput: up to \~30,000 msg/s for 1KB messages, outperforming classic mirrored queues ([RabbitMQ][3]).
  * However, extreme backlog without consumers can cause degraded performance ([RabbitMQ][3]).
* **Consumer Priorities & SAC (v4.0+)**: Quorum queues honor consumer priority within single active consumer logic, enabling switchover to higher priority consumers when they subscribe ([RabbitMQ][4]).
* **Delivery Limit Default**: Set to 20 attempts by default. Messages dropped if there's no DLX setup; recommended to configure DLX to avoid data loss ([RabbitMQ][4]).
* **Startup Optimization (v4.0+)**: Checkpoint files reduce startup time drastically compared to full WAL replay ([RabbitMQ][4]).

---

## 9. Client Behavior: Consumer Flags

* **Exclusive Consumers Ignored**: Quorum queues ignore the `exclusive` flag. Use **Single Active Consumer (SAC)** instead ([RabbitMQ][5]).
* **SAC + Priority Behavior**:

  * With SAC enabled, the first consumer becomes active.
  * A later consumer with higher `x-priority` takes over once the current consumer ack’s its in-flight messages ([RabbitMQ][4]).
* **Consumer Activity Flag** reflects true active consumer per queue and is visible via management/CLI ([RabbitMQ][5]).

---

## 10. Migration Guidance

* **Up-to-date Migration**: Migrate via blue-green to new vhost, optionally using federation to minimize downtime ([RabbitMQ][6]).
* **Compatibility Considerations**:

  * Feature mismatches (priority queues, TTL, etc.) may require code or policy changes ([RabbitMQ][3]).
  * Tools exist to detect policies using mirrored queues and guide migration.

---
