# MQTT and the IoT World

In 1999, Andy Stanford-Clark at IBM and Arlen Nipper at Arcom designed a protocol to monitor oil pipelines over satellite links — connections that were expensive per byte, unreliable, and intermittent. The protocol they designed used publish/subscribe messaging, fit control packets into a minimum of two bytes, and could maintain a session across disconnections. They called it MQTT (originally Message Queue Telemetry Transport, though the authors later dropped the expanded name).

Twenty-five years later, MQTT is the dominant protocol for IoT devices. It runs on microcontrollers with 32KB of RAM, over cellular connections that lose signal when the device drives through a tunnel, in deployments that range from single-device hobby projects to millions of sensors reporting in simultaneously. If you are building anything with constrained devices, unreliable networks, or fire-and-forget telemetry, you should understand MQTT before you reach for HTTP.

## The Architecture

MQTT is a publish/subscribe protocol built on top of TCP. The architecture has three components:

**Clients** — publishers, subscribers, or both. A client can be a sensor publishing temperature readings, a mobile app subscribing to notifications, or a backend service that both subscribes to events and publishes commands.

**Broker** — the central message hub. Clients do not communicate directly; they connect to the broker, which routes messages between them. Popular brokers include Eclipse Mosquitto (lightweight, open source, runs on a Raspberry Pi), EMQ X (high-throughput, clustering support, HTTP and WebSocket gateways), HiveMQ (enterprise-oriented, cluster-native), and cloud-hosted brokers like AWS IoT Core and Azure IoT Hub.

**Topics** — hierarchical string identifiers that organize the message space. A topic like `sensors/building-a/floor-3/temperature` describes exactly what it sounds like. Topics are not pre-registered; they spring into existence when a client publishes to them.

The publish/subscribe model decouples publishers from subscribers in time and space. A temperature sensor does not know or care how many services are subscribed to its readings. A dashboard service does not know or care how many sensors are contributing to the feed. This decoupling is architecturally valuable and is one of the things HTTP request-response makes difficult.

## The Packet Format

MQTT's packet format reflects its constrained-network origins. Every packet starts with a fixed header of at least two bytes:

```
Byte 1: [Message Type (4 bits)] [Flags (4 bits)]
Byte 2+: Remaining Length (variable-length encoding, 1-4 bytes)
```

The remaining length uses a variable-length encoding borrowed from Protocol Buffers: each byte contributes 7 bits of the length value, and the high bit indicates whether another byte follows. This allows lengths up to 268 MB to be encoded in at most 4 bytes, while short messages (the common case) use a single byte for the length.

Compare this to HTTP: a minimal HTTP/1.1 GET request is typically 250+ bytes just for the method, path, host, and required headers. An MQTT PUBLISH packet for a small payload is around 7-10 bytes of overhead. For a device sending a temperature reading every second over a cellular connection where data costs money per kilobyte, this difference accumulates.

## Quality of Service Levels

MQTT's three QoS levels are one of its most important features, and one of the most frequently misunderstood.

**QoS 0 — At most once (fire and forget).** The broker delivers the message at most once, with no acknowledgment. If the connection drops during delivery, the message is lost. Overhead: one round trip (PUBLISH). Appropriate for high-frequency sensor data where individual readings are not critical — if you lose a temperature reading, the next one arrives in a second.

**QoS 1 — At least once.** The sender stores the message until it receives a PUBACK acknowledgment. If the connection drops before the acknowledgment arrives, the sender retransmits the message (with the DUP flag set). This guarantees delivery but may deliver duplicates. Overhead: two round trips (PUBLISH → PUBACK). The receiver must handle duplicates by making its processing idempotent or by tracking message IDs.

**QoS 2 — Exactly once.** A four-way handshake ensures the message is delivered exactly once. PUBLISH → PUBREC → PUBREL → PUBCOMP. The sender holds the message until PUBCOMP; the broker holds a message ID reservation until PUBREL. Overhead: four round trips. Appropriate for cases where duplication causes problems (commands, financial transactions) but high latency is acceptable.

The choice of QoS level is not just about reliability — it is about the tradeoff between reliability, latency, and overhead. A sensor that sends 100 readings per second probably wants QoS 0. A control command that turns off a piece of industrial equipment probably wants QoS 2.

One important subtlety: QoS levels apply hop-by-hop, not end-to-end. If a publisher sends at QoS 2, the delivery guarantee is between the publisher and the broker. If a subscriber subscribes at QoS 1, the delivery guarantee between the broker and the subscriber is QoS 1 (at most the minimum of the two levels). The guarantee is the minimum across both legs.

## Retained Messages

A retained message is a flag on a PUBLISH packet that tells the broker: "store this message, and deliver it immediately to any client that subscribes to this topic in the future." The broker keeps one retained message per topic.

This solves a common IoT problem: a dashboard connects to the broker and subscribes to `sensors/building-a/floor-3/temperature`. Without retained messages, the dashboard sees nothing until the sensor sends its next reading, which might be 60 seconds away. With a retained message, the broker immediately delivers the last known value, and the dashboard shows current data from the moment it connects.

Retained messages are not message history — the broker stores only the most recent one. For historical data, you need a separate storage tier that a subscriber writes into.

## Last Will and Testament

When a client connects to the broker, it can specify a "will" message: a topic, payload, QoS, and retain flag that the broker should publish if the client disconnects unexpectedly (as opposed to a clean disconnect). This is the Last Will and Testament (LWT) mechanism.

LWT is how MQTT handles device failure notification. A sensor connects and specifies: "if I disconnect unexpectedly, publish `offline` to `sensors/building-a/floor-3/status`." Any monitoring system subscribed to the status topic will see the offline notification within seconds of the sensor losing power or connectivity, without polling or any other explicit failure detection.

This is genuinely elegant — failure detection is built into the connection protocol, not the application layer. HTTP has no equivalent primitive; implementing similar behavior requires heartbeat endpoints, health checks, and monitoring systems.

## MQTT 5.0

MQTT 3.1.1 was the dominant version for many years. MQTT 5.0, published in 2019, added significant capabilities that address practical pain points.

**Reason codes.** MQTT 3.1.1 had limited error reporting — a connection could be refused with a handful of fixed codes. MQTT 5.0 expands reason codes for every packet type, making it possible to understand why an operation failed.

**User properties.** Key-value pairs that can be attached to any packet. This allows application-level metadata to travel with messages without embedding it in the payload — useful for correlation IDs, schema versions, content types, and other routing metadata.

**Message expiry interval.** A time-to-live for messages. The broker discards messages older than the expiry interval rather than delivering stale data. Critical for IoT: a message telling a HVAC system to lower the temperature is worthless if it arrives three hours late.

**Subscription identifiers.** Allows a subscriber to tag its subscriptions with identifiers, which the broker includes when delivering matching messages. Useful when a single client has multiple wildcard subscriptions and needs to know which one matched.

**Shared subscriptions.** Multiple clients can subscribe using the same "shared subscription" group, and the broker load-balances messages across them round-robin. This enables competitive consumers for horizontal scaling — previously, every subscriber received every message, which made it difficult to parallelize processing.

**Response topic and correlation data.** Allows request-response patterns over MQTT. The publisher includes a response topic; the receiver publishes its response to that topic with the same correlation data. This is not native request-response (MQTT is fundamentally pub/sub), but it makes request-response patterns possible without layering a separate protocol.

## Where MQTT Falls Short

MQTT is designed around specific tradeoffs. Understanding them prevents misuse.

**The broker is a single point of failure.** Every client connects to the broker. If the broker goes down, all communication stops. High-availability broker deployments require clustering (EMQX, HiveMQ both support this) or active-passive failover. This is manageable but adds operational complexity.

**No native request-response.** MQTT is publish/subscribe. MQTT 5.0's response topic feature enables the pattern, but it is built on top of pub/sub rather than native to the protocol. If your use case is fundamentally request-response — "send a command, get a reply, know whether it succeeded" — MQTT requires more boilerplate than HTTP or gRPC would.

**Topic design requires care.** Topics are hierarchical strings, and the design of the topic namespace for a large system becomes a significant architectural decision. Poorly designed topic hierarchies become unmanageable; wildcard subscription patterns that match too broadly create performance problems. This is not unique to MQTT, but the lack of tooling around topic schema definition (compared to, say, OpenAPI for HTTP) makes it easy to accumulate technical debt.

**Authorization is coarse.** MQTT's authentication is connection-level (username/password or TLS client certificates). Topic-level authorization — this client may publish to `sensors/+` but may not subscribe to `commands/+` — is supported by most brokers but configured outside the protocol, in broker-specific ways. There is no standard ACL format.

**No message ordering guarantees across sessions.** Within a single persistent session, messages on a given topic are ordered. Across reconnections, with queued messages flushing from multiple connections, ordering is not guaranteed. If your application requires strict message ordering across sessions, MQTT requires application-level sequence numbers.

## MQTT in Practice

The constraint that MQTT is designed for is worth making concrete: an ESP32 microcontroller has 512KB of RAM and no operating system. It connects over WiFi using a small TCP/IP stack implemented in ROM. It needs to report sensor readings, receive commands, and detect when it has lost connectivity. HTTP is possible but heavyweight; establishing a new TLS+HTTP connection for every reading consumes meaningful power on a battery-operated device. MQTT's persistent connection, compact packets, and built-in will messages are not just conveniences — they are what makes the application feasible.

The same attributes that make MQTT right for constrained devices also make it right for large-scale telemetry pipelines in the cloud. AWS IoT Core handles billions of MQTT messages per day, routing sensor data to DynamoDB, Lambda, S3, and other AWS services. The fan-out of a single published message to hundreds of subscribers, with the broker doing all the routing, scales naturally in ways that direct HTTP calls between services do not.

Where MQTT becomes the wrong choice is when your devices are not constrained, your network is reliable, and your communication patterns are fundamentally request-response. In that context, MQTT's pub/sub model requires workarounds (the MQTT 5.0 response topic pattern) to do something that HTTP does natively. The broker adds latency that is unnecessary when the client and server are in the same datacenter. The appropriate choice in that context is one of the protocols covered in other chapters.

## Brokers in Production

Running MQTT at scale means operating a broker. A few practical notes:

**Mosquitto** is the right choice for development and for single-instance deployments where high availability is not required. It is small, fast, well-documented, and widely understood. The operational model is simple.

**EMQX** (formerly EMQ X) is a distributed MQTT broker written in Erlang, designed for high availability and high throughput. It supports clustering natively, exposes a REST API for management, and can handle millions of concurrent connections. It is more complex to operate than Mosquitto.

**HiveMQ** is the enterprise option — commercial support, clustering, enterprise security features. Appropriate for large organizations where support contracts and compliance matter.

**Cloud-managed brokers** (AWS IoT Core, Azure IoT Hub, Google Cloud IoT Core) shift the operational burden to the provider. They handle clustering, availability, and scaling. The tradeoff is vendor lock-in, cost at scale, and limited configuration for edge cases that the managed service does not support.

The choice of broker is largely operational rather than protocol-level. The MQTT protocol works the same way regardless of which broker implements it; clients can switch brokers without code changes (beyond connection configuration).
