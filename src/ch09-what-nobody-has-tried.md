# What Nobody Has Tried Yet

Every protocol in this book was a response to a specific constraint: JSON too slow, request-response too limiting, brokers too complex, serialization too expensive. The constraints that drove those designs are well understood. The protocols that will exist in ten years will respond to constraints that are just now becoming visible, or that we are only beginning to articulate clearly.

This chapter is speculative. It is about directions, not products. Some of these ideas are active research areas with prototypes; some are directions that the constraints of distributed systems seem to be pointing toward without anyone yet having synthesized the right design; some are genuine open questions. Treat it as a map of the unexplored territory rather than a guide to specific destinations.

## QUIC-Native Protocols

HTTP/3 runs QUIC under HTTP semantics. But QUIC is a general-purpose transport layer with properties that HTTP does not fully exploit.

QUIC gives you:
- Independent streams that do not suffer head-of-line blocking (a lost packet affects only the stream it belongs to, not others on the same connection)
- 0-RTT connection establishment for resumed connections (no handshake round trip for known servers)
- Connection migration (a QUIC connection can survive the client's IP address changing — when a mobile device transitions from WiFi to cellular, the connection continues)
- Built-in loss detection and recovery per-stream rather than per-connection

gRPC over QUIC is a real project and is being implemented, but it layers gRPC's existing HTTP/2 semantics onto QUIC — it does not redesign the protocol to take advantage of what QUIC enables.

The more interesting possibility is a protocol designed from scratch for QUIC's semantics: one that treats streams as first-class primitives, uses connection migration for seamless handoff between networks, and takes advantage of 0-RTT for latency-critical applications. The mobile use case is compelling: an API protocol where the client transitions from WiFi to cellular mid-session without any connection interruption, reconnection delay, or application-level retry. That is technically possible with QUIC today; no widely-deployed API protocol has been designed specifically for it.

MoQ (Media over QUIC), currently in IETF standardization, is an early example of a non-HTTP QUIC protocol — specifically for live video and game streaming. Its design choices (object-based delivery, subscriber relay model, prioritized delivery within a connection) would not be possible with TCP and are not natural within HTTP's request-response model. MoQ is unlikely to be the final word, but it demonstrates that QUIC-native protocol design is a real space.

## Algebraic Effects as an RPC Model

Contemporary RPC is a remote procedure call: you invoke a function, wait for a result, use the result. The caller blocks (or uses a callback/promise/async/await to avoid blocking the thread) until the callee responds. This is a specific programming model, and it carries specific limitations.

Algebraic effects are a programming language concept (implemented in languages like Koka, Eff, and as a research feature in OCaml) that generalizes exception handling: rather than throwing an exception that unwinds the stack, you "raise" an effect, which is handled by an effect handler further up the call stack that can resume the computation where it left off. Unlike exceptions, effects are resumable.

Applied to RPC, this suggests a different interface: rather than calling a remote function and waiting, you express your computation in terms of effects, and the runtime handles the question of whether those effects are satisfied locally or remotely. A function that needs data from a remote service does not explicitly make an RPC call; it raises an effect, and the effect handler decides how to satisfy it — maybe by making a network call, maybe by returning cached data, maybe by batching multiple effect requests into a single network round trip.

This is not currently implemented at the protocol level. Languages with algebraic effects are research or niche. But the concept points at something real: the current RPC model mixes data access patterns (which data do I need?) with transport concerns (how do I get it?), and that mixing is the source of many of the problems that libraries like DataLoader (for GraphQL N+1 queries) try to solve after the fact.

A protocol built around effect-based semantics would separate these concerns: the application declares its data needs; the runtime decides how to satisfy them efficiently. The protocol layer would need to support batching, merging requests, pipelining, and streaming in ways that the application code does not explicitly orchestrate.

## Local-First and Offline-First APIs

The dominant API model assumes network connectivity. A client calls a server; the server holds the ground truth; the client is a thin display layer that becomes useless when the network is unavailable. This assumption is so deeply embedded in REST's design that it is invisible until you need to violate it.

The local-first software movement (articulated clearly by Kleppmann et al. in their 2019 paper) argues for an architecture where the ground truth lives in local storage on the user's device, and the network is a synchronization mechanism rather than a data source. Offline operation is the default; synchronization happens when connectivity is available.

Technically, this requires:

**Conflict-free Replicated Data Types (CRDTs)**: data structures that can be modified independently on multiple devices and merged without conflicts. Operations are commutative and associative, so merge order does not matter. The challenge is designing CRDT data models that match application semantics rather than just set operations.

**Vector clocks and causal consistency**: tracking causality across replicas so that "A happened before B" relationships are preserved during synchronization, even when the devices communicating them were disconnected when the events occurred.

**Sync protocols**: the actual protocol for exchanging state between replicas. This is an open design space. Existing approaches (rsync-style diffing, Operational Transforms, CRDT-native sync) all have tradeoffs in bandwidth efficiency, convergence guarantees, and implementation complexity.

No standard protocol for local-first sync exists. Applications use custom solutions, or libraries like Automerge and Yjs (which implement specific CRDT-based approaches). The opportunity is for a general-purpose sync protocol that:
- Works over any transport (HTTP, WebSockets, Bluetooth, local network)
- Handles partial connectivity (sync with what's available)
- Provides strong eventual consistency guarantees
- Composes with standard authentication and authorization mechanisms

This is a protocol design problem that nobody has solved in a general way.

## Reactive APIs and Push-Based Truth

The request-response model has the client asking questions and the server answering them. An alternative model has the server asserting truths and the client subscribing to changes.

Differential Dataflow (from Frank McSherry's work) and reactive databases like Materialize implement something close to this: you write a SQL query, and rather than evaluating it once and returning a result, the system evaluates it continuously and pushes incremental updates when the underlying data changes. The client subscribes to a query's result set; the server pushes `+row` and `-row` deltas as the result changes.

This flips the API model: the client does not poll for current state; the current state is pushed to the client continuously as it changes. The bandwidth usage is proportional to the rate of change rather than the frequency of polling. Stale data is not a thing — the client always sees the current state because the server has been updating them in real-time.

The challenge is reconciling this model with existing network infrastructure. WebSockets can carry the delta stream, but the semantics — subscribing to a query result rather than a topic — require server-side query evaluation infrastructure that is not standard. Systems like Supabase's Realtime and Fauna's reactive queries implement specific variants of this model, but there is no protocol standard.

A future protocol in this space might define: a query language (probably something SQL-like), a format for incremental result set deltas, semantics for subscription management and query cancellation, and connection recovery with state reconciliation. It would likely require differentiated server infrastructure — not just a message broker but a reactive query engine.

## P2P API Models

Every protocol in this book assumes a client-server topology. One side serves; the other side consumes. But the rise of edge computing, local-area network applications, and systems built on libp2p (the protocol stack underlying IPFS) suggests that peer-to-peer API models may become more relevant.

In a P2P API model, there is no fixed server. Any node can call methods on any other node; discovery happens through distributed mechanisms (distributed hash tables, gossip protocols, mDNS for local networks). Content addressing (routing to where specific data is, rather than to a specific machine) replaces location addressing.

The practical use cases today are niche: distributed applications, peer-to-peer games, local network device communication, censorship-resistant systems. But the boundary between "edge server" and "powerful client device" is narrowing. A future where a mobile device does computation for its peers in a local mesh, and the concept of "calling an API" means invoking computation on whichever device in your local network has the capability, is not science fiction — it is the direction that WebRTC's data channels and libp2p's protocol stack are pointed at.

The protocol design challenge is significant: discovery, authentication, capability advertisement, load distribution, and failure handling all work differently in a peer-to-peer topology than in a client-server topology.

## The Compression Opportunity

This is less speculative than the others, but important: serialization and compression are separate concerns in current designs, and combining them more tightly has room for improvement.

Protocol Buffers compresses well with general-purpose compressors (gzip, Brotli, zstd) because its binary format lacks the redundancy of JSON text. But general-purpose compressors do not know anything about the data's schema. A schema-aware compressor that knows field types, probable value distributions, and correlation between fields could achieve much better compression ratios than a schema-agnostic one.

Columnar formats (Apache Parquet, Apache Arrow) already exploit schema knowledge for batch data — they store all values for a column together, enabling much better compression because similar values are adjacent. For API protocols, the equivalent would be batching multiple messages and applying columnar compression across the batch — storing all `amount_cents` values together, all `currency` codes together — before transmission.

Some specialized systems do this already. No general-purpose API protocol does. The latency cost of batching (you must accumulate messages before compressing) makes this tradeoff only worthwhile for high-throughput pipes where bandwidth is the bottleneck, but at sufficient scale, the bandwidth savings could justify the latency cost.

## What the Constraints Tell Us

Looking at the direction of these open problems, a few common themes emerge.

**The assumption of reliable, low-latency connectivity will continue to weaken.** Mobile, edge, IoT — all push toward protocol designs that handle intermittent connectivity, disconnected operation, and network transitions as first-class concerns rather than edge cases. Protocols designed around the assumption of a stable data center connection will increasingly be wrong choices for a growing fraction of deployments.

**The distinction between data and computation will blur.** Reactive query subscriptions, local-first sync, and distributed compute all challenge the "dumb pipe with smart endpoints" model. Future protocols may need to carry not just data but queries, continuations, and computation artifacts.

**Security and authorization will need to be more tightly integrated with the protocol layer.** Cap'n Proto's capability model is a hint at what this might look like. As systems become more distributed and peer-to-peer, the assumption that the network boundary is the authorization boundary (which underlies most current API security models) breaks down. The authorization model needs to be carried in the protocol, not added as an application layer on top.

These are constraints, not designs. The protocols that emerge from them do not exist yet. But working engineers who understand what today's protocols are missing are the ones who will be positioned to design the next generation — or at minimum, to recognize and adopt it when someone else does.
