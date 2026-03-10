# gRPC — HTTP/2 in a Convincing Costume

Here is the thing nobody says out loud at conferences: gRPC is not an alternative to HTTP. It is HTTP/2 with a binary framing layer and a code generator bolted on top. Every gRPC call is an HTTP/2 request. The metadata is HTTP/2 headers. The status codes are mapped to HTTP/2 trailers. The multiplexing, flow control, and connection management are all HTTP/2 doing what HTTP/2 does.

This is not a criticism. gRPC is genuinely useful, and the protocol choices behind it are sound. But understanding what gRPC actually is — rather than what the positioning implies — is prerequisite to understanding when it helps and when it does not.

## What gRPC Actually Is

gRPC was open-sourced by Google in 2015 and is now a CNCF project. Google had been running its own internal RPC system called Stubby for years; gRPC was a public version built on the then-emerging HTTP/2 standard rather than a proprietary transport.

The stack has three layers:

1. **Protocol Buffers** for interface definition and serialization. You define your service and message types in `.proto` files. The protoc compiler generates client stubs and server skeletons in your language of choice. On the wire, messages are binary-encoded using the Protocol Buffers encoding, which is compact and fast to parse compared to JSON.

2. **HTTP/2** as the transport. gRPC uses HTTP/2's multiplexing to run multiple concurrent RPCs over a single TCP connection. Request and response metadata travel as HTTP/2 headers. Request and response bodies travel as HTTP/2 DATA frames, with a five-byte length-prefix that gRPC adds for framing within the stream.

3. **The gRPC protocol** which is a thin layer on top of HTTP/2 specifying how to map RPC semantics — method names, status codes, error details, timeouts, deadlines — to HTTP/2 primitives.

The result is a system that is semantically much richer than JSON over HTTP/1.1, but which traffics over port 443 and looks like HTTPS to network infrastructure that does not inspect payload.

## Protocol Buffers: Schema as Contract

Protocol Buffers deserves separate attention because it is doing most of the work that engineers often attribute to gRPC.

A `.proto` file is a machine-readable, language-independent schema:

```protobuf
syntax = "proto3";

package payments;

service PaymentService {
  rpc Charge(ChargeRequest) returns (ChargeResponse);
  rpc Stream(StreamRequest) returns (stream Event);
}

message ChargeRequest {
  string idempotency_key = 1;
  int64 amount_cents = 2;
  string currency = 3;
  string source_token = 4;
}

message ChargeResponse {
  string transaction_id = 1;
  ChargeStatus status = 2;
}

enum ChargeStatus {
  CHARGE_STATUS_UNSPECIFIED = 0;
  CHARGE_STATUS_SUCCESS = 1;
  CHARGE_STATUS_DECLINED = 2;
  CHARGE_STATUS_ERROR = 3;
}
```

From this file, protoc generates:
- A client stub in every supported language (Go, Java, Python, C++, Ruby, C#, Node.js, and many more)
- A server interface that you implement
- Serialization and deserialization code
- Type-safe accessors for all fields

The generated code is not just convenient — it is a compiler-enforced contract. If you remove a field from a message that a client still reads, the build fails on the client side. If you add a required field to a request without updating the server, the server will return an error. The schema is not documentation; it is a build artifact that both sides depend on.

This is qualitatively different from OpenAPI, which generates client code from a spec but cannot prevent the spec from drifting from the server implementation. With protobuf, the schema and the implementation are the same thing.

**Schema evolution** is handled through field numbers, not field names. The binary encoding references fields by their integer tags, not their string names. As long as you never reuse a field number and never change a field's type, you can add new fields and remove old ones without breaking existing clients. Old clients ignore fields they do not know about; new clients get zero values for fields that old servers do not send. This is not magic — it requires discipline — but the mechanism is well-designed.

## The Four Streaming Modes

gRPC supports four interaction patterns, which is one of its genuine advantages over REST:

**Unary RPC** is the standard request-response: one request, one response. This is what REST gives you, mapped to HTTP/2.

```protobuf
rpc Charge(ChargeRequest) returns (ChargeResponse);
```

**Server streaming** allows the server to send a sequence of responses to a single request. Useful for progress updates, large result sets, or push notifications.

```protobuf
rpc WatchEvents(WatchRequest) returns (stream Event);
```

**Client streaming** allows the client to send a sequence of requests before receiving a single response. Useful for bulk uploads or aggregation.

```protobuf
rpc BatchImport(stream ImportRecord) returns (ImportSummary);
```

**Bidirectional streaming** allows both sides to send sequences of messages independently over a single connection. The semantics are not quite arbitrary bidirectional messaging — the server stream is still initiated by a client request — but it is close enough for most purposes.

```protobuf
rpc Chat(stream Message) returns (stream Message);
```

All four modes are built on HTTP/2 streams. The unary mode uses a single HTTP/2 request-response cycle. The streaming modes use HTTP/2 stream multiplexing to keep the logical stream open while messages flow.

## What HTTP/2 Brings

Since gRPC is HTTP/2, it inherits HTTP/2's properties directly:

**Multiplexing.** A single TCP connection carries multiple concurrent RPCs without head-of-line blocking at the HTTP layer. With HTTP/1.1, if you had 100 concurrent requests you needed 100 connections (in practice, six, serialized). With HTTP/2, you need one.

**Header compression.** HTTP/2 HPACK compresses headers using a combination of static and dynamic tables. Repeated headers (Content-Type, method names, common metadata) are transmitted as indexed references rather than repeated strings. For workloads with many small requests, this meaningfully reduces overhead.

**Binary framing.** HTTP/2 frames are binary, not text. This is more efficient to parse and less ambiguous than HTTP/1.1's text-based format, which has been a source of security vulnerabilities (request smuggling, response splitting) that arise from inconsistent parsing.

**Flow control.** HTTP/2 has per-stream and per-connection flow control windows. This prevents a fast sender from overwhelming a slow receiver without dropping the connection.

What HTTP/2 does not help with: **TCP head-of-line blocking**. HTTP/2 solves head-of-line blocking at the HTTP layer, but TCP is still a single ordered stream. If one packet is lost, all streams on that connection stall until the loss is recovered. HTTP/3 over QUIC solves this by using UDP with per-stream loss recovery. gRPC over QUIC exists but is not yet standard.

## The Things That Hurt

The convincing costume starts to slip in a few specific places.

**Load balancers are confused.** Traditional L4 load balancers distribute connections. With HTTP/1.1, where each connection carries a small number of requests, connection-level load balancing is a reasonable proxy for request-level load balancing. With HTTP/2, where a single long-lived connection carries many multiplexed requests, connection-level load balancing fails completely: all traffic from a client will go to whichever backend handled the connection establishment, and other backends will be idle.

gRPC requires L7 load balancing — a proxy that understands the HTTP/2 framing and can route individual requests to different backends. This means you need Envoy, or nginx with HTTP/2 upstream support, or a service mesh, or client-side load balancing with a gRPC-aware client library. These are solvable problems, but they are real operational complexity that does not exist with HTTP/1.1.

**Browsers cannot use gRPC directly.** The browser's fetch API does not expose HTTP/2 trailers, which gRPC uses for status codes. The browser's XMLHttpRequest does not support HTTP/2 at all. The result is that you cannot call a gRPC service from browser JavaScript using the standard gRPC protocol.

gRPC-Web is the workaround. It is a modified protocol that uses HTTP/1.1 or HTTP/2, encodes trailers differently, and works through a translation proxy (typically Envoy). It works, but it requires the proxy, it does not support client streaming, and it adds complexity. If your primary client is a browser, gRPC-Web's operational overhead may not be worth it.

**Debugging requires tools.** curl understands HTTP. It does not speak Protocol Buffers. A gRPC request captured by Wireshark or a logging proxy looks like opaque binary data. You need grpcurl or a gRPC-aware debugging tool to inspect traffic. This is manageable, but it changes the character of debugging and makes ad-hoc exploration harder.

**Error handling is often re-invented.** gRPC has a small set of status codes (OK, CANCELLED, UNKNOWN, INVALID_ARGUMENT, and about a dozen others). These cover the common cases but are coarser than HTTP's rich status code vocabulary. In practice, many gRPC services embed more detailed error information in response messages or in google.rpc.Status details, which means you end up with two error-handling systems anyway.

**Proto evolution requires discipline.** The schema evolution story is good, but it requires that everyone follow the rules: never reuse field numbers, never change field types, always use zero-valued defaults for missing fields. On a small team or a codebase with strong conventions, this is fine. On a large team with many contributors, field number management becomes a coordination problem, and the backward-compatibility guarantee is only as strong as the discipline enforcing it.

## When gRPC Is the Right Choice

Despite the above, gRPC is genuinely excellent in specific contexts.

**Microservice-to-microservice communication** inside a controlled infrastructure is gRPC's strongest use case. You control both ends. Browsers are not involved. The schema contract eliminates the API versioning headaches that plague JSON-based services. Code generation removes the class of bugs where the client and server have drifted. The operational complexity of L7 load balancing is worth it for a system with hundreds of internal services.

**Polyglot environments** benefit enormously from the generated client libraries. If your system has services written in Go, Java, Python, and Rust, the alternative to generated clients is maintaining hand-written clients in all four languages, keeping them in sync as the API evolves. protoc does this for you.

**High-throughput services** where serialization overhead matters. Protocol Buffers encoding is meaningfully faster to serialize and deserialize than JSON, and smaller on the wire. For services processing millions of requests per second, this matters. For services handling a few hundred requests per second, it probably does not.

**Streaming use cases** where you need server push, client streaming, or true bidirectional streaming without the complexity of WebSockets. gRPC streaming is built into the protocol and integrated with the type system; it is not an afterthought.

## The Honest Summary

gRPC is HTTP/2 with a schema language, code generation, and a binary serialization format. These are genuinely valuable additions. The schema language gives you compiler-enforced contracts. The code generation eliminates a class of serialization bugs. The binary format is faster and smaller than JSON.

The transport is still HTTP/2, which means you get HTTP/2's benefits (multiplexing, header compression, binary framing) and HTTP/2's operational requirements (L7 load balancing, TLS). The performance improvements over HTTP/1.1 + JSON are real but often overstated; the operational complexity is real and often understated.

If you are building internal services in a microservice architecture, gRPC is probably the right default. If you are building a public API that browsers consume, you should think carefully before choosing gRPC-Web over a well-designed HTTP/JSON API. If you need transport that is not HTTP at all — for constrained devices, for message queues, for pub/sub semantics — gRPC does not help you and the subsequent chapters will.
