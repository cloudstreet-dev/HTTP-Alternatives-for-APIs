# How to Choose — A Decision Framework

Every chapter in this book has ended with some version of "it depends." This chapter does not try to replace judgment with a flowchart. What it does do is make the dependencies explicit — describe what "it depends on" actually means in terms you can evaluate for your specific system.

The framework here is not a decision tree. Decision trees for protocol selection are lies: they look authoritative, they collapse the actual complexity, and they give you someone else's opinion expressed as objective logic. What follows instead is a set of dimensions along which protocols differ, with honest characterizations of where each one fits. Apply these dimensions to your specific context.

## Dimension 1: Communication Pattern

This is usually the most decisive dimension. What is the fundamental shape of the conversation between your components?

**Request-response (client initiates, server answers once):** This is the pattern that REST, gRPC unary, Thrift, Twirp, and JSON-RPC are all designed for. It matches: user actions, data retrieval, commands where you need a synchronous acknowledgment. It mismatches: any pattern where you need the server to push data, where multiple responses are expected from one request, or where the client needs to stream data to the server.

**Server push (server initiates, client receives):** SSE is the simplest implementation. WebSocket server-to-client streaming works. gRPC server streaming works. MQTT with a retained message or subscription works. AMQP consumer works. The question is whether you also need the client to send concurrent data — if not, SSE is often the simplest and most robust choice.

**Full duplex (both sides send, both sides receive simultaneously):** WebSockets, gRPC bidirectional streaming, NATS, ZeroMQ with DEALER/ROUTER. The trigger for this is simultaneous independent messaging — not just request-response in both directions, but cases where A sends to B while B is simultaneously sending to A without coordination.

**Pub/sub (many publishers, many subscribers, decoupled):** MQTT, AMQP topic exchanges, NATS, Kafka. The defining characteristic is that publishers do not know who their subscribers are. This decoupling enables patterns that request-response cannot: broadcasting to many consumers, dynamic consumer topology, subscribers that come and go independently of publishers.

**Pipeline/stream processing (ordered, persistent, replayable):** Kafka, NATS JetStream, AMQP durable queues. The characteristic is that messages are records to be processed, not ephemeral events to be delivered. The consumer's offset into the stream matters; replay from a specific point is needed.

Getting this wrong is expensive. Trying to implement pub/sub over REST (polling) or trying to do complex routing with WebSockets (rolling your own AMQP) both result in large amounts of application code that re-implements protocol features.

## Dimension 2: Delivery Guarantees

What does your system need, and what can it tolerate?

**At-most-once (fire and forget):** Acceptable when: the cost of losing a message is low and the cost of duplicate processing is high, or when throughput requirements make acknowledgment overhead unacceptable. Appropriate for: high-frequency telemetry, real-time game state, logs where missing some entries is acceptable. HTTP without retries is at-most-once. MQTT QoS 0. ZeroMQ's default behavior.

**At-least-once:** Acceptable when: processing is idempotent (the same message processed twice has the same result as processing it once). Appropriate for: most event-driven architectures where messages represent distinct events that should be processed but duplicate processing is harmless. MQTT QoS 1, AMQP with acknowledgment, Kafka with at-least-once semantics, most message queuing systems.

**Exactly-once:** Needed when: processing is not idempotent and duplicates cause real problems (sending an email twice, charging a payment twice). Hard to achieve in distributed systems — requires coordination between the sender, the transport, and the receiver. MQTT QoS 2, AMQP transactions, Kafka exactly-once semantics (with caveats), idempotency keys in REST APIs. Every "exactly-once" guarantee comes with caveats about what "exactly once" means in the face of specific failure modes.

A practical note: exactly-once is more expensive than at-least-once, which is more expensive than at-most-once — in latency, in overhead, and in implementation complexity. Many systems that claim to need exactly-once delivery actually need at-least-once with idempotent processing, which is simpler and cheaper.

## Dimension 3: Latency and Throughput Requirements

These are related but distinct, and they point toward different protocol choices.

**Latency-critical (sub-millisecond, microsecond budgets):** You are in the territory of ZeroMQ, custom binary protocols, or Cap'n Proto. HTTP is not a contender. The broker adds a hop; the hop adds latency. Any protocol with a broker is likely disqualified. The question is how much engineering you are willing to invest in eliminating overhead.

**High-throughput (millions of messages/second, bandwidth-constrained):** Binary serialization (Protocol Buffers, Cap'n Proto, FlatBuffers) matters here. JSON's parse cost is a real fraction of CPU at this scale. Connection pooling, batching, and compression become important. Kafka's design is optimized for this — sequential disk writes, batch transmission, efficient consumer group management.

**Moderate throughput, normal latency (most applications):** You have more freedom. The overhead of JSON is not your bottleneck; REST or gRPC both work. Choose based on other dimensions.

**Low-throughput, constrained bandwidth (IoT, satellite links):** MQTT's compact framing and session management were designed specifically for this. The per-message overhead of any HTTP-based protocol is a real cost at this bandwidth constraint.

## Dimension 4: Client Diversity

What clients will consume this API, and what are their constraints?

**Public API consumed by arbitrary clients:** REST/HTTP is the strong default. The tooling, documentation ecosystem, and universal support for HTTP make any other choice a barrier for external developers. Binary protocols require client library support in every language a consumer might use; HTTP is already supported everywhere.

**Browser clients:** HTTP/REST or gRPC-Web (with proxy overhead) or WebSockets/SSE for push. No client-streaming gRPC. No raw TCP. No MQTT without an HTTP bridge. The browser sandbox is the constraint.

**Mobile clients (iOS, Android):** Most networking options are available (HTTP, WebSockets, gRPC). The relevant constraint is battery: maintaining a persistent connection for push consumes more power than polling. MQTT's LWT and persistent session model was specifically designed for mobile. If battery life matters, the connection management model of your chosen protocol matters.

**IoT / embedded devices:** MQTT dominates for a reason. The compact framing, low connection overhead, and built-in network-failure handling are not niceties — they determine whether the application is feasible on constrained hardware.

**Internal service-to-service:** You control both ends, so any protocol is viable. The question shifts to operational concerns and team capability. gRPC is a strong default for polyglot microservices. AMQP is a strong default for work queues. NATS is a reasonable alternative for teams that want lower operational overhead.

## Dimension 5: Failure Handling and Durability

What happens when things go wrong, and what does your system require?

**Loss of a component is tolerable:** Stateless HTTP fits. If the server that handled your request goes down, you retry against another server. This works because the server held no conversational state.

**Loss of a message is catastrophic:** You need durability — AMQP durable queues with persistent messages, Kafka's replicated logs, or application-level storage with idempotency keys. HTTP is not durable: a 200 response means the server received and processed the request, but if the server crashes immediately after accepting the request and before writing to storage, the work is lost.

**Partial connectivity is expected (mobile, IoT):** Session persistence across disconnections is valuable. MQTT's clean/persistent session, WebSocket reconnection with state replay, or application-level sequence numbers with gap detection. Protocols without reconnection semantics require application code to handle this.

**Fan-out to many consumers:** AMQP fanout exchanges, MQTT subscriptions, NATS subject matching. Naive HTTP requires N requests to N consumers; a pub/sub system requires one publish and the broker (or protocol library) handles fan-out.

## Dimension 6: Operational Complexity

This dimension is often underweighted because it is not visible during the design phase. It becomes very visible at 2 AM when something breaks.

**No infrastructure beyond the application:** REST over HTTP, WebSockets/SSE, gRPC, ZeroMQ, Twirp. The protocol lives in the application; there is no broker to deploy, monitor, or maintain.

**Single broker with straightforward operation:** Mosquitto for MQTT. A single RabbitMQ instance. A single NATS server. These are manageable for small teams. They add a failure domain — the broker can fail — but the operational surface is understandable.

**Clustered broker with operational depth:** Kafka, EMQ X, HiveMQ cluster, RabbitMQ cluster. These require deeper operational knowledge: cluster configuration, partition management, replication factors, consumer group rebalancing, monitoring of broker internals. The capabilities they provide are worth the cost at scale; at small scale, the cost exceeds the benefit.

**Service mesh and proxy infrastructure:** gRPC at scale requires L7 load balancing — Envoy, Istio, or similar. This is not just "deploy a proxy"; it is adopting an infrastructure paradigm that touches every service. The benefits (observability, traffic management, mTLS) are real at large scale. For small deployments, the overhead is not justified.

Be honest about your team's operational capacity. A protocol that is theoretically optimal but operationally complex beyond your team's ability to maintain is a liability, not an asset. The right answer for a two-person startup is different from the right answer for a platform team at a large organization.

## Dimension 7: Schema and Contract

How important is the formal definition of your API's interface?

**Schema-first with code generation:** gRPC + Protocol Buffers, Thrift, Cap'n Proto. The schema is the source of truth; clients and servers are generated from it. Type mismatches are caught at compile time. Schema evolution is governed by the schema format's rules.

**Schema-first with validation only:** OpenAPI/Swagger for REST, JSON Schema for WebSocket payloads. The schema defines the contract but clients are not generated from it (or if they are, the generated clients are less tightly bound than with protobuf-generated code). Drift between schema and implementation is possible.

**Schemaless:** Raw WebSockets, ZeroMQ without a payload schema, MQTT without a payload schema. The contract is in the documentation. Runtime errors, not compile-time errors, reveal mismatches. Appropriate when flexibility outweighs safety (rapidly evolving APIs, heterogeneous clients with different capabilities).

For internal services, schema-first with code generation is almost always worth the investment. The class of bugs it eliminates — mismatched field names, type coercions, missing required fields — is large enough that the upfront cost pays back quickly. For external APIs where clients are outside your control, the tradeoff depends on how many clients you have and how much you want to invest in supporting them.

## Putting It Together

Rather than a decision tree, here are characterizations of common situations:

**Web application with a REST backend and a need for real-time updates:** Add SSE for server push. If you also need client-to-server streaming or complex bidirectional interaction, upgrade to WebSockets on those specific endpoints. Do not replace your REST API; add streaming alongside it.

**Microservices communicating internally in a polyglot environment:** gRPC is the default. The protobuf schema contract and generated clients eliminate entire categories of integration bugs. The operational overhead of L7 load balancing is real, but it is manageable with Kubernetes and a service mesh.

**IoT device telemetry at scale:** MQTT. The hardware constraints, connection model, and QoS semantics were all designed for exactly this.

**Work queue for background processing with reliability guarantees:** AMQP (RabbitMQ). Dead letter exchanges, acknowledgment semantics, and queue durability handle the reliability requirements. If you need replay and high throughput, evaluate Kafka.

**High-frequency inter-process messaging on the same host:** ZeroMQ with `inproc://` or `ipc://` transport, or a shared-memory queue. The network stack adds latency that you do not need.

**Event streaming with downstream consumers at their own pace:** Kafka or NATS JetStream. The persistent log, offset tracking, and consumer groups are what distinguish this from pub/sub.

**Public API for third-party developers:** REST over HTTP with OpenAPI documentation. Devex wins. The marginal performance improvement of anything else is not worth the friction you are adding for your consumers.

## The Decision You Are Actually Making

When you choose a protocol, you are not just choosing a wire format. You are choosing:

- What failure modes you are responsible for handling (vs. what the protocol handles)
- What your monitoring and debugging story looks like
- What your team needs to know to operate the system
- What your clients need to know to consume your service
- What happens when your traffic grows by 10x

The best protocol for your system is the one that matches these constraints, not the one that scores highest on abstract technical criteria. A system built on the right protocol for its constraints is easier to reason about, easier to operate, and more reliable than one built on a technically superior protocol that does not match the team's capabilities or the system's operational context.

The goal of this book has been to give you enough information to make this choice deliberately — to be the engineer who chose REST because REST was right, not because REST was the default; who chose gRPC because the polyglot contract enforcement justified the operational complexity, not because gRPC was what the last job used; who chose MQTT because the device constraints demanded it, not because "IoT uses MQTT."

That deliberate choice — made with full information, owned by the engineers who made it — is the difference between a system you understand and a system you inherited.
