# HTTP Alternatives for APIs

Most API code is written without a protocol decision. Engineers reach for REST over HTTP because it is what they know, what the framework assumes, and what everyone else uses. That is a reasonable default — but it is still a choice, and like any invisible choice, it has costs.

This book surveys the alternatives: what they are, how they work, where they outperform REST, and where they do not. It is organized as a practical tour rather than a reference manual. Each chapter covers one protocol or family of protocols in enough depth to understand the tradeoffs, with the goal of making protocol selection a deliberate decision rather than an accidental one.

## What This Book Covers

**Chapter 1 — The Default We Never Questioned**
Why REST became the baseline, what it costs in performance and expressiveness, and when those costs matter enough to look elsewhere.

**Chapter 2 — gRPC**
Protocol Buffers and HTTP/2 under the hood: strongly-typed contracts, efficient binary serialization, and streaming primitives that REST cannot express.

**Chapter 3 — WebSockets and SSE**
Full-duplex connections and server-sent events: when you need the server to push data rather than wait to be asked.

**Chapter 4 — MQTT and the IoT World**
Publish-subscribe over constrained networks: designed for unreliable connections, minimal overhead, and millions of devices.

**Chapter 5 — AMQP and Message Brokers**
Enterprise messaging with guaranteed delivery, routing, and backpressure — the RabbitMQ ecosystem and when you need a broker in the middle.

**Chapter 6 — ZeroMQ**
Messaging without a broker: low-latency socket patterns for high-throughput systems that cannot afford the overhead of a central queue.

**Chapter 7 — Cap'n Proto RPC and the Binary Frontier**
Zero-copy serialization and capability-based RPC at the extreme end of the performance spectrum.

**Chapter 8 — Lesser-Known Contenders**
Thrift, Avro, Flatbuffers, Nano, NATS, and others that solve real problems in specific contexts.

**Chapter 9 — What Nobody Has Tried Yet**
Emerging directions: QUIC, WebTransport, local-first architectures, and patterns at the edge of current practice.

**Chapter 10 — How to Choose**
A decision framework for matching protocol to use case, with worked examples across common API scenarios.
