# Lesser-Known Contenders

The protocols in the preceding chapters are the ones that come up in engineering discussions and appear in job descriptions. But the protocol landscape is wider than any canonical list. This chapter covers the protocols that are real, used in production, and worth understanding — even if they do not dominate conference talks.

## Apache Thrift

Apache Thrift was developed at Facebook in 2007 and open-sourced in 2008. It predates gRPC by several years and solves the same core problem: define a service interface once and generate client and server code in multiple languages.

A Thrift definition file describes services and data types:

```thrift
namespace py payments
namespace java com.example.payments

enum ChargeStatus {
  SUCCESS = 1,
  DECLINED = 2,
  ERROR = 3
}

struct ChargeRequest {
  1: required string idempotency_key,
  2: required i64 amount_cents,
  3: required string currency,
  4: optional string source_token
}

struct ChargeResponse {
  1: required string transaction_id,
  2: required ChargeStatus status
}

service PaymentService {
  ChargeResponse charge(1: ChargeRequest request)
    throws (1: InvalidRequestException invalid, 2: ServiceException error)
}
```

Thrift generates client stubs and server skeletons for a wide range of languages. Where it diverges from gRPC is in its transport and protocol flexibility: Thrift separates transport (TCP, HTTP, in-memory) from protocol (binary, compact binary, JSON) from service (the actual method dispatch). You choose a combination. A development server might use the JSON protocol over HTTP for debuggability; production uses the compact binary protocol over TCP for efficiency.

Thrift's adoption peaked when it was the dominant cross-language RPC framework before gRPC existed. Its continued use today is largely in organizations that built on Thrift before gRPC was available and have too much investment to migrate. Twitter, Evernote, and many others ran on Thrift for years. The framework is mature, battle-tested, and well-understood; it is just no longer the default choice for new systems.

The comparison with gRPC: Thrift has more flexible transport/protocol combinations and no dependency on HTTP/2. gRPC has better ecosystem support today, server reflection, and streaming that is integrated into the protocol rather than added as an extension. For new polyglot RPC projects, gRPC wins on ecosystem; Thrift wins if you specifically need the transport flexibility or are in a Java-heavy organization with existing Thrift infrastructure.

## Apache Avro RPC

Apache Avro is primarily known as a data serialization format used in the Hadoop ecosystem. Its schema language describes data structures in JSON:

```json
{
  "type": "record",
  "name": "ChargeRequest",
  "fields": [
    {"name": "idempotency_key", "type": "string"},
    {"name": "amount_cents", "type": "long"},
    {"name": "currency", "type": "string"},
    {"name": "source_token", "type": ["null", "string"], "default": null}
  ]
}
```

Avro RPC uses these schemas for service definition, with the interesting property that schema exchange happens at connection time: the client and server send each other their schemas during the handshake, and the protocol handles schema evolution by mapping between the writer's schema (the schema the message was written with) and the reader's schema (the schema the reader uses now). This means a reader can evolve its schema independently of the writer, as long as the schemas are compatible.

In practice, Avro RPC's adoption as a general-purpose RPC mechanism is minimal. Its primary use is within the Kafka ecosystem, where Avro is the standard format for Kafka messages paired with the Confluent Schema Registry. The Schema Registry stores schema versions, and Avro consumers use schema resolution to handle producers and consumers that are on different schema versions. If you are using Kafka with Avro, you are using Avro's schema evolution model. If you are not in the Kafka/Hadoop ecosystem, you are probably not using Avro RPC.

## NATS

NATS is a messaging system developed at Apcera (later acquired by Synadia), written in Go, and released as open source. It occupies an interesting position: lighter than Kafka, simpler than RabbitMQ, distributed natively, and fast.

The core model is publish/subscribe over subjects (NATS's term for topics). A publisher sends a message to a subject; subscribers that have expressed interest in that subject receive it. Subject matching supports wildcards: `sensors.>` matches any subject that starts with `sensors.`; `sensors.*` matches exactly one token after `sensors.`.

NATS is unusual in having a request-reply pattern built into the base protocol. A publisher can send a message with a reply-to subject; the recipient publishes its response to that reply-to subject. NATS clients make this pattern ergonomic by automatically generating unique reply-to subjects and handling the subscription lifecycle.

NATS JetStream, added in 2021, extends the base NATS model with persistence, at-least-once and exactly-once delivery, consumer groups (for competing consumers), stream replay, and key-value stores. JetStream transforms NATS from a best-effort messaging system into a full-featured event streaming platform that competes with Kafka in its target use case.

The case for NATS over Kafka: simpler operational model (NATS is a single binary; Kafka requires ZooKeeper or KRaft plus the brokers themselves), lower latency for smaller message volumes, built-in request-reply semantics, and strong Go and Rust library support. The case for Kafka over NATS: higher throughput for very large volumes, more mature stream processing ecosystem (Kafka Streams, Flink integration), and broader enterprise adoption.

NATS is a strong choice for teams that want Kafka-like semantics with less operational overhead. It is particularly popular in Kubernetes environments, where the operational simplicity aligns well with the container deployment model.

## Twirp

Twirp is a minimalist RPC framework developed by Twitch, built on Protocol Buffers and HTTP/1.1 (and HTTP/2) rather than the full gRPC protocol. The service definition uses the same `.proto` format as gRPC, but the generated client/server code communicates over plain HTTP with either binary (protobuf) or JSON content.

The appeal of Twirp over gRPC is exactly the gRPC problems from Chapter 2: Twirp works with standard HTTP/1.1, is compatible with HTTP load balancers without L7 awareness, supports browser clients without a proxy layer, and can be debugged with curl when using the JSON content type.

The tradeoff: Twirp does not support streaming. It is strictly unary RPC — one request, one response. If you need server streaming, client streaming, or bidirectional streaming, you need gRPC.

For teams that want the ergonomics of protobuf code generation and type-safe RPC without the operational complexity of gRPC, Twirp is a reasonable choice. It is a simpler system that is easier to operate, debug, and integrate with standard HTTP infrastructure, at the cost of streaming support.

## JSON-RPC and XML-RPC

JSON-RPC is a stateless remote procedure call protocol that uses JSON for encoding and HTTP (or WebSockets, or any transport) for transmission. A JSON-RPC request is:

```json
{
  "jsonrpc": "2.0",
  "method": "payment.charge",
  "params": {"amount_cents": 1000, "currency": "USD"},
  "id": "req-123"
}
```

The response:

```json
{
  "jsonrpc": "2.0",
  "result": {"transaction_id": "txn_abc123", "status": "SUCCESS"},
  "id": "req-123"
}
```

JSON-RPC is simple enough to implement in an afternoon. It provides a method-based calling convention over JSON, with batch request support and a standard error format. It is used heavily in the Ethereum ecosystem (the standard JSON-RPC API for interacting with Ethereum nodes follows this protocol) and in VS Code's Language Server Protocol.

XML-RPC is the ancestor of JSON-RPC, replacing JSON with XML. It predates both REST and SOAP and is almost entirely confined to legacy systems today. It is worth knowing as history — it demonstrates that non-REST, method-based RPC over HTTP existed long before gRPC — but you would not choose it for new work.

## GraphQL Over WebSocket

GraphQL is primarily associated with HTTP, but its subscription model — real-time data that the server pushes to the client when data changes — is typically implemented over WebSockets using a protocol called graphql-ws (the current standard) or the older subscriptions-transport-ws.

The graphql-ws protocol is a thin framing layer over WebSockets: clients send subscribe/unsubscribe messages with GraphQL subscription documents; servers send data, error, and completion messages. The transport is WebSockets; the query language is GraphQL's subscription syntax.

This is relevant to the protocol discussion because it represents a class of hybrid design: take a query language designed for HTTP request-response, and extend it to server push by swapping the transport for WebSockets. The subscription model fits naturally because GraphQL subscriptions define exactly what data the client wants to receive, and the server evaluates that subscription on every relevant event.

If you are already using GraphQL for your API, the websocket subscription mechanism is usually the right way to add real-time push — it reuses the existing schema, the existing authentication and authorization model, and the existing client libraries. If you are not using GraphQL, this is not a reason to adopt it.

## Honorable Mentions

**gRPC-Web** deserves mention as a distinct protocol from gRPC. It strips out the parts of gRPC that do not work in browsers (trailer-based status codes, client streaming) and encodes messages in a format that browser fetch APIs can handle. It requires a proxy (Envoy or grpc-web-proxy) to translate to real gRPC for the backend. It is not a general replacement for gRPC but a specific bridge for browser clients.

**Cap'n Proto's siblings** — Flatcc and bebop are alternative zero-copy serialization formats with different language-target and performance characteristics. Bebop in particular has very fast generated code for .NET environments.

**OpenTelemetry Protocol (OTLP)** is not a general-purpose RPC protocol but worth knowing: it is a Protocol Buffers-based protocol over HTTP/2 or gRPC used for transmitting telemetry data (metrics, traces, logs). If you are building observability pipelines, OTLP is the emerging standard that replaces custom Jaeger and Zipkin formats.

**Protobuf over HTTP** (sometimes called "proto over REST") is the approach of using Protocol Buffers as the serialization format for HTTP/1.1 endpoints, without the gRPC protocol layer. You get the schema benefits and compact encoding of protobuf, with none of the gRPC operational complexity and full compatibility with standard HTTP infrastructure. Twirp is one formalization of this pattern; hand-rolled implementations are also common. This is underused — many teams that want better-than-JSON serialization reach for gRPC when protobuf-over-HTTP would meet their needs with less overhead.

## The Pattern Across All of These

Looking across the lesser-known contenders, a pattern emerges: most of them are solving one or two specific problems with mainstream protocols, and the solution comes with tradeoffs that explain why they did not displace the mainstream options.

Thrift solved polyglot RPC before gRPC existed, but without HTTP/2's transport and without gRPC's ecosystem investment. NATS solved lightweight pub/sub without Kafka's operational complexity, but without Kafka's throughput. Twirp solved gRPC's browser and load-balancer problems, but without streaming. JSON-RPC added method-dispatch semantics to JSON, but without schemas or types.

The mainstream protocols won by being good enough across many dimensions simultaneously. The lesser-known ones are often better on one specific dimension and worse on others. That is useful information: if the specific dimension where the lesser-known option excels matches your primary constraint, it might be the right tool regardless of its market position.
