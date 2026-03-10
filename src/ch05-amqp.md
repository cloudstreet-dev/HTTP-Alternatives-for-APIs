# AMQP and Message Brokers

The question that message brokers answer is not "how do I send data from A to B?" That is a networking question. The question is: "how do I decouple A from B such that A can send data even when B is down, B can process at its own rate, multiple Bs can share the work, and no messages are lost in the handoff?"

HTTP cannot answer that question. MQTT can answer parts of it. AMQP — Advanced Message Queuing Protocol — is designed to answer all of it, with formal semantics for routing, acknowledgment, durability, and transactions.

## What AMQP Is

AMQP 0-9-1, published in 2008, is the protocol that RabbitMQ implements. (There is also AMQP 1.0, a different and incompatible standard published in 2011 and ISO-standardized; it is implemented by Apache ActiveMQ Artemis, Azure Service Bus, and others. When engineers say "AMQP" in the context of RabbitMQ, they almost always mean 0-9-1.)

AMQP defines a binary wire protocol with formal semantics for a set of concepts: exchanges, queues, bindings, messages, channels, consumers, and acknowledgments. The protocol is not just a transport — it specifies the behavior of the broker in enough detail that multiple independent implementations should be interchangeable for compliant clients.

In practice, the AMQP 0-9-1 specification had gaps, and broker implementations filled them differently. RabbitMQ added many extensions (publisher confirms, consumer priorities, dead letter exchanges, and others) that are widely used but not part of the base protocol. The pragmatic reality is that AMQP 0-9-1 defines the common vocabulary, and RabbitMQ's specific behavior is what most users actually depend on.

## The Broker Model

AMQP's model has more moving parts than MQTT's, and the parts interact in specific ways.

**Exchanges** receive messages from publishers. The exchange decides how to route a message based on its type and the message's routing key.

- **Direct exchange**: Routes messages to queues whose binding key exactly matches the message's routing key. Used for targeted delivery.
- **Fanout exchange**: Routes messages to all bound queues, ignoring the routing key. Used for broadcast.
- **Topic exchange**: Routes messages to queues whose binding key matches the routing key using wildcard patterns (`*` for a single word, `#` for zero or more words). Used for flexible publish/subscribe with hierarchical routing.
- **Headers exchange**: Routes messages based on header attributes rather than the routing key. Powerful but rarely used.

**Queues** store messages until they are consumed. Queues are durable (survive broker restart), auto-delete (disappear when the last consumer disconnects), or exclusive (private to a single connection). Messages in a durable queue on a durable exchange are persisted to disk, not just memory.

**Bindings** connect exchanges to queues. A queue is bound to an exchange with a binding key (or headers criteria for headers exchanges). The binding is what makes routing happen — without bindings, messages published to an exchange go nowhere.

**Channels** are virtual connections multiplexed over a single TCP connection. Each operation (publishing, consuming, acknowledging) happens on a channel. Channels are cheap to create and destroy; applications typically use one channel per thread or one per logical operation type.

This model gives you flexibility that simpler protocols do not:

A single message published to one exchange can be routed to many queues simultaneously (fanout or topic routing). Multiple services can consume from the same queue in a competing consumer pattern, processing messages in parallel without receiving duplicates (round-robin delivery). A service can consume from multiple queues and process them in priority order.

## Acknowledgments and Durability

Message delivery in AMQP is explicit and acknowledgment-based.

A consumer receives a message and can:
- **Acknowledge** (ack): the broker removes the message from the queue permanently. Delivery succeeded.
- **Negative-acknowledge** (nack): the broker can either requeue the message (for another consumer to attempt) or discard it (for dead-letter routing). Delivery failed; retry or discard.
- **Reject**: similar to nack, but for a single message. The broker requeues or discards based on the requeue flag.

A critical operational concept: messages that are delivered but not yet acknowledged are in an "unacknowledged" state. They cannot be delivered to another consumer while the current consumer holds them. If the consumer connection dies before acknowledging, the broker makes the message available again. This is the guarantee that prevents message loss on consumer failure.

The `prefetch` count (basic.qos) controls how many unacknowledged messages a consumer can hold at once. Setting it to 1 creates a strict round-robin with no buffering at consumers — the broker delivers the next message only after the current one is acknowledged. Setting it higher allows consumers to pipeline processing. The tradeoff is between throughput (higher prefetch) and fair distribution (lower prefetch, avoiding one slow consumer monopolizing the queue).

For durability:
- The exchange must be declared durable
- The queue must be declared durable
- Messages must be published with `delivery_mode: 2` (persistent)

All three conditions must hold. A persistent message on a non-durable queue is lost on restart; a durable queue on a non-persistent exchange is lost on restart. This is a common source of confusion: engineers declare a durable queue and assume durability, not realizing they also need persistent messages.

## Publisher Confirms

Basic AMQP message acknowledgment is consumer-to-broker. Publisher confirms add broker-to-publisher acknowledgment: after the broker has persisted a persistent message to disk (or enqueued a transient one), it sends a confirm to the publisher.

Without publisher confirms, publishing is fire-and-forget. If the broker restarts immediately after receiving a message but before persisting it, the message is lost. With publisher confirms, the publisher can maintain an outstanding set of published-but-unconfirmed messages and resend them if the broker connection drops before they are confirmed.

Publisher confirms are a RabbitMQ extension, not part of the base AMQP 0-9-1 spec, but they are so fundamental to reliable publishing that they are essentially universal in production RabbitMQ deployments.

## Dead Letter Exchanges

When a message is rejected (without requeue), or expires due to TTL, or is dropped because a queue is at its length limit, it can be routed to a dead letter exchange (DLX) rather than simply discarded. The DLX is a normal exchange that routes dead letters to a dead letter queue.

Dead letter queues are where poison messages, processing failures, and expired events go for inspection and remediation. An alert on a growing dead letter queue is one of the most valuable monitoring signals in a message-based system: it tells you that messages are failing to be processed, without losing the messages themselves.

The typical pattern is:
1. Messages are published to the main exchange.
2. Consumers process messages from the main queue. On unrecoverable errors, they nack without requeue.
3. Dead letters route to the DLX, then to the dead letter queue.
4. A separate process (or a human) inspects the dead letter queue, diagnoses the issue, and either republishes to the main queue (if the issue was transient) or archives the messages for manual review.

Without this mechanism, failed messages either disappear (if you discard failures) or cause endless retry loops (if you always requeue failures). The DLX pattern creates a clean separation between the happy path and the failure remediation path.

## TTL, Priority, and Other Queue Properties

AMQP queues support several properties that control message handling:

**Message TTL**: Messages expire after a duration if not consumed. Prevents queue buildup of stale messages. Can be set per-queue (all messages in the queue expire after the same duration) or per-message.

**Queue TTL**: The queue itself expires after being unused for a duration. Useful for ephemeral queues created for temporary consumers.

**Max length and max bytes**: Caps the queue size. When a new message would exceed the cap, the broker either drops it to the DLX or drops the oldest message (depends on overflow setting).

**Priority queues**: Messages carry a priority (0-255). Higher-priority messages are delivered before lower-priority ones. Useful for urgent commands that should skip ahead of routine background work.

**Lazy queues**: Messages are written to disk immediately rather than kept in memory. Reduces memory pressure at the cost of higher latency. Appropriate for queues that may grow very large.

## AMQP Versus Kafka

Any honest treatment of AMQP must address the Kafka comparison, because in many organizations the decision is not "AMQP or HTTP?" but "RabbitMQ or Kafka?"

They are designed for different purposes.

**RabbitMQ** is a message broker: it routes messages from publishers to consumers, tracks acknowledgments, and removes messages when they have been successfully processed. The unit is the message, and the model is delivery — once a consumer acknowledges, the message is gone.

**Kafka** is an event log: messages are written to partitioned, ordered logs, retained for a configurable duration, and consumers maintain their own offsets into the log. Messages are not removed when consumed; the same message can be consumed by many independent consumer groups, each maintaining its own position. Kafka is designed for event sourcing, stream processing, and audit trails.

The practical differences:

- **Replayability**: Kafka consumers can replay from any point in the log. RabbitMQ messages are gone once acknowledged; replay requires application-level storage.
- **Consumer coordination**: Kafka's consumer groups handle partition assignment and rebalancing automatically. RabbitMQ's competing consumers are simpler but less controllable.
- **Ordering**: Kafka guarantees ordering within a partition. RabbitMQ guarantees ordering within a queue for single-consumer setups; competing consumers break ordering.
- **Throughput**: Kafka is designed for very high write throughput (millions of messages per second). RabbitMQ's throughput is lower but sufficient for most applications.
- **Complexity**: Kafka requires a cluster, ZooKeeper (or KRaft in newer versions), careful partition configuration, and schema management for the event log. RabbitMQ is simpler to operate for single-instance or small-cluster deployments.

Use RabbitMQ when you need: task queues with work distribution, complex routing logic (multiple exchanges, binding patterns), transient messages that can be discarded after consumption, and relatively simple operational requirements.

Use Kafka when you need: event sourcing, stream processing with tools like Kafka Streams or Flink, replay and audit capabilities, very high throughput, and you are willing to pay the operational cost.

## When AMQP Is the Right Choice

The core AMQP value proposition is: reliable, ordered, acknowledged, routable message delivery with formal guarantees about what happens when things fail.

This is the right choice for:

**Work queues with multiple consumers.** A queue of image processing jobs, each claimed by exactly one worker, acknowledged when the worker succeeds, requeued when the worker fails. AMQP handles this cleanly. HTTP would require a database-backed job queue, polling, and explicit locking — significant application code to replicate behavior that AMQP provides natively.

**Integration between heterogeneous systems.** An order placed in an e-commerce system needs to trigger fulfillment, inventory updates, and fraud detection. These systems are written by different teams, run at different rates, and have different availability guarantees. An AMQP exchange decouples them: the order system publishes; downstream systems consume at their own pace with their own acknowledgment logic.

**Transient spike absorption.** An email service can process 100 messages per second. An event causes 10,000 messages to arrive in the first minute. A queue absorbs the spike; the email service drains it at its own rate. This is load leveling — a fundamental pattern in reliable distributed systems — and AMQP queues implement it directly.

**Guaranteed delivery where loss is unacceptable.** Financial transactions, order confirmations, audit events — cases where losing a message has real consequences. AMQP's combination of durable queues, persistent messages, and publisher confirms provides a strong delivery guarantee that HTTP request-response does not, because HTTP does not tell you what happens to a request after it is accepted.

The cases where AMQP is not the right choice: ultra-low latency requirements where broker overhead is unacceptable, IoT with constrained devices (MQTT wins there), streaming analytics where the log model matters (Kafka wins there), or simple request-response where the broker is unnecessary complexity.
