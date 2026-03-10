# ZeroMQ — Messaging Without a Broker

Every protocol in the previous two chapters runs through a broker. MQTT requires a broker. AMQP requires a broker. Kafka requires a cluster of brokers. The broker is the hub; clients are spokes; no spoke can reach another spoke directly.

The broker provides important things: durability, routing logic, queue management, acknowledgment tracking. But it also provides something you did not ask for: a centralized piece of infrastructure that can fail, that must be operated, that becomes a bottleneck, and that adds a network hop to every message.

ZeroMQ starts from a different premise. The question ZeroMQ asks is: what if the messaging capabilities lived in the application, not in infrastructure? What if the "broker" was an abstraction implemented by the library itself, peer-to-peer, without any central coordinator?

## What ZeroMQ Is

ZeroMQ (also written ØMQ, ZMQ, or zmq) is a messaging library, not a message broker. You add it as a dependency to your application and get a set of socket types that implement messaging patterns at the library level. There is no server to deploy, no configuration to manage, no cluster to operate. The patterns — publish/subscribe, push/pull, request/reply, dealer/router — are implemented by the library using direct TCP (or IPC, or in-process) connections between participating processes.

This is architecturally unusual and worth thinking about carefully, because the mental model for ZeroMQ is different from every other protocol in this book.

When you create a ZeroMQ PUB socket and bind it to a port, you have created a publisher that accepts incoming connections from subscribers. When a subscriber creates a SUB socket and connects to the publisher's port, a direct TCP connection is established between publisher and subscriber. Messages flow from publisher to subscribers over those direct connections. There is no broker in the middle.

## The Socket Types

ZeroMQ's core abstraction is the socket, but ZeroMQ sockets are nothing like BSD sockets. A ZeroMQ socket encapsulates a messaging pattern, not a connection. Under the hood, a single ZeroMQ socket may manage many simultaneous TCP connections, do internal queuing, and handle connection management automatically. From the application's perspective, you just send and receive messages.

**REQ / REP (Request-Reply)**

The most basic pattern: a REQ socket sends a request and blocks waiting for a reply. A REP socket receives a request and must send exactly one reply before receiving the next request.

```python
# Server
ctx = zmq.Context()
socket = ctx.socket(zmq.REP)
socket.bind("tcp://*:5555")
while True:
    message = socket.recv()
    socket.send(b"pong")

# Client
socket = ctx.socket(zmq.REQ)
socket.connect("tcp://localhost:5555")
socket.send(b"ping")
reply = socket.recv()
```

The strict alternation (send, receive, send, receive) makes REQ/REP fragile in practice: if a reply is never received, the socket is stuck. REQ/REP is useful for simple scripts and tests, but for production use you almost always want DEALER/ROUTER.

**DEALER / ROUTER (Async Request-Reply)**

DEALER is like REQ but non-blocking: it can send multiple messages without waiting for replies. ROUTER is like REP but address-aware: each incoming message is prefixed with the sender's identity, and replies must include the identity to route back to the correct sender.

DEALER/ROUTER is the pattern for building async servers, load balancers, and proxies. A load balancer connects a ROUTER facing clients with a DEALER facing workers, forwarding messages between them in whatever pattern the load-balancing strategy requires.

**PUB / SUB (Publish-Subscribe)**

A PUB socket publishes messages. Any number of SUB sockets can connect and receive copies. Subscribers filter messages by prefix: a subscriber that calls `socket.setsockopt(zmq.SUBSCRIBE, b"sensor:")` receives only messages whose payload begins with `sensor:`.

```python
# Publisher
socket = ctx.socket(zmq.PUB)
socket.bind("tcp://*:5556")
while True:
    socket.send_multipart([b"sensor:temp", b"23.5"])
    socket.send_multipart([b"sensor:humidity", b"67.2"])
```

An important property: PUB/SUB in ZeroMQ is lossy by default. If a subscriber is slow or a subscriber has not connected yet, the publisher does not block and does not buffer messages for that subscriber. Slow subscribers fall behind and lose messages. This is deliberate — the publisher's throughput is not limited by the slowest subscriber.

If you need guaranteed delivery in a PUB/SUB pattern, you need to add reliability on top: a subscriber can request missed messages via a separate REQ/REP channel, or you can add a persistent intermediary.

**PUSH / PULL (Pipeline)**

PUSH sends messages round-robin to connected PULL sockets. PULL receives from any connected PUSH socket. This is the work queue pattern without a broker: a PUSH socket distributes tasks; PULL sockets (workers) claim them.

```
[ Ventilator (PUSH) ] → [ Worker 1 (PULL) ]
                       → [ Worker 2 (PULL) ]
                       → [ Worker 3 (PULL) ]
```

Workers process tasks independently and send results to a PUSH socket connected to a collector's PULL socket. The ventilator–worker–collector pipeline is a classic ZeroMQ pattern and works well for embarrassingly parallel computation.

**PAIR**

PAIR connects exactly two sockets. Useful for intra-process or inter-thread communication where you want the simplicity of a dedicated channel without the overhead of managing connections manually.

## The Transport Layer

ZeroMQ supports multiple transport mechanisms, switchable by changing the address prefix:

- `tcp://` — Standard TCP connections, for inter-process and inter-host communication
- `ipc://` — Unix domain sockets, for inter-process communication on the same host (faster than TCP loopback)
- `inproc://` — In-process communication, for inter-thread messaging without any OS involvement (fastest possible; zero serialization overhead when using the same context)
- `pgm://` and `epgm://` — Pragmatic General Multicast, for UDP multicast (rarely used; requires multicast-capable network infrastructure)

The transport is a concern of the address string, not the socket type. A PUB socket can bind on `tcp://` in one deployment and switch to `ipc://` in another without any code change.

## What ZeroMQ Does Well

**Raw throughput.** ZeroMQ is fast. The library does minimal copying, uses lock-free data structures internally where possible, and batches small messages automatically. Published benchmarks have shown ZeroMQ achieving millions of messages per second on commodity hardware. The overhead per message is measured in microseconds, not milliseconds.

**Flexibility in topology.** ZeroMQ does not impose a topology. You can build star networks, rings, pipelines, trees, meshes — any topology that your application requires. The same socket types work in all of them. This is particularly useful for systems where the communication topology changes based on runtime conditions.

**No infrastructure dependencies.** For a system where every microsecond of operational complexity matters, having no broker to deploy, configure, and maintain is a genuine advantage. The messaging is part of the application; it deploys with the application; it fails with the application rather than independently.

**Language breadth.** ZeroMQ bindings exist for essentially every language: C, C++, Python, Java, Go, Rust, Ruby, Node.js, Erlang, and many more. The core library is C, and the bindings are thin wrappers. Messages cross language boundaries with zero transformation overhead.

**Dynamic topology.** ZeroMQ handles connection management automatically. A PUB socket bound to a port accepts new SUB connections as they arrive; it detects and handles disconnections. A DEALER that connects to multiple ROUTER endpoints and one goes down will route around it when the reconnect timer fires. Topology changes do not require restart or reconfiguration.

## What ZeroMQ Does Not Do

**No persistence.** ZeroMQ does not persist messages to disk. If no SUB socket is connected when a PUB socket sends, the message is gone. If a PULL worker crashes before acknowledging a task, the task is gone. ZeroMQ is purely an in-memory, in-process messaging layer.

**No acknowledgment.** ZeroMQ has no native acknowledgment primitive. Send returns when the message is in the outgoing queue; there is no notification when the receiver processes it. Building at-least-once delivery requires application code that adds acknowledgment messages and resend logic.

**No routing beyond prefix filtering.** PUB/SUB subscription filters are prefix-based string matching. There is no content-based routing, no topic hierarchy with wildcards, no exchange-and-binding model. If you need to route messages based on complex criteria, you implement the routing logic in your application.

**No authorization or encryption in the core.** ZeroMQ's base implementation is unauthenticated and unencrypted. ZAP (ZeroMQ Authentication Protocol) provides an authentication layer, and CURVE is an elliptic-curve encryption and authentication layer. Both work well, but they are not defaults — you must explicitly configure them, and few tutorials show you how.

**Complex patterns require discipline.** The ZeroMQ guide (known as the "zguide") is excellent, but it runs to several hundred pages for a reason. Building reliable messaging on top of ZeroMQ's primitives requires understanding failure modes — what happens when a subscriber falls behind, when a DEALER's connected ROUTER goes down, when the network partitions — and implementing appropriate handling. AMQP's broker handles these failure modes for you; ZeroMQ exposes them to you.

## Nanomsg and NNG

Nanomsg was created by Martin Sústrik, one of ZeroMQ's original authors, as a cleaner reimplementation that addressed some of ZeroMQ's design decisions (the C++ codebase, the license, the architecture of the library). NNG (Nanomsg Next Generation) is a further evolution by Garrett D'Amore.

NNG supports the same socket patterns (PUB/SUB, REQ/REP, PUSH/PULL, and others) with a cleaner C API, more transport options (TLS built in, WebSocket support), and an aio (asynchronous I/O) model. For new projects that want ZeroMQ-style messaging without ZeroMQ's historical baggage, NNG is worth evaluating.

## When ZeroMQ Fits

ZeroMQ is the right choice when:

**You need low-latency messaging between cooperating processes** and the overhead of a broker is unacceptable. Financial systems that route orders between components with microsecond budgets use ZeroMQ. HFT firms use ZeroMQ. Real-time game servers use ZeroMQ.

**You need flexible topology** that would require complex configuration in a broker-based system. If your communication graph changes dynamically or has an unusual shape, ZeroMQ's lack of a central coordinator can simplify the topology rather than complicate it.

**You are in an environment where broker deployment is impractical.** Embedded systems, edge computing, development environments, or systems that must function without infrastructure — these benefit from ZeroMQ's library-level approach.

**You are building the messaging infrastructure** rather than using it. ZeroMQ is the right foundation for building higher-level messaging abstractions because its socket primitives are low enough to compose.

ZeroMQ is the wrong choice when:

**You need durability.** If messages must survive process restarts, power failures, or network partitions, you need a broker with persistent storage.

**You need guaranteed delivery** without building it yourself. At-exactly-once semantics require acknowledgment, storage, and resend logic — all the things a broker provides.

**Your team is unfamiliar with the failure modes.** ZeroMQ's flexibility comes with the responsibility of handling failures that a broker would handle for you. The learning curve is real, and the mistakes are subtle (silent message loss is ZeroMQ's most common surprise).

The choice between ZeroMQ and a broker-based system is fundamentally a choice about where you want the complexity to live: in infrastructure that someone else operates (broker), or in application code that you write and maintain (ZeroMQ). Neither is universally better. The right choice depends on whether your reliability requirements are better met by infrastructure guarantees or by application-level control.
