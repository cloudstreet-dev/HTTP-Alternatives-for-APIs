# Cap'n Proto RPC and the Binary Frontier

In 2013, Kenton Varda — one of the primary authors of Protocol Buffers while at Google — published Cap'n Proto. His stated goal was to fix what he saw as Protocol Buffers' fundamental design mistake: encoding. Protocol Buffers serializes data by traversing a data structure and writing fields sequentially to a byte buffer. Cap'n Proto does not serialize at all. The wire format is the in-memory format.

This is not a marketing claim. It is a specific design decision with measurable consequences, and understanding it changes how you think about the entire category of binary serialization protocols.

## The Zero-Copy Design

When you decode a Protocol Buffers message, the library reads bytes from the wire and constructs objects in memory — copying data, allocating memory, parsing varint-encoded integers. When you're done with the objects, they're garbage collected. Serialization is the reverse: traverse the objects, emit bytes. Every message crossing a process boundary involves at minimum two full traversals and two copies.

Cap'n Proto's wire format is a memory layout that can be mmap'd and used directly. A Cap'n Proto message arriving from the network is a sequence of bytes that is already a valid in-memory representation. Accessing a field in a Cap'n Proto message is a pointer dereference, not a parse. There is no decode step.

This is possible because Cap'n Proto uses a flat, pointer-based representation:

- Structs are laid out as a fixed-size data section (for primitive fields) followed by a pointer section (for variable-length fields, lists, and nested structs)
- Pointers are relative offsets, so the structure is position-independent (can be moved in memory without fixing up pointers)
- Lists are contiguous arrays with a list pointer header specifying element type and count
- The entire message is a sequence of "segments" that can be transmitted across different memory allocations or memory-mapped from files

The result: reading a field from a received Cap'n Proto message has zero allocation overhead and requires only a bounds-checked pointer arithmetic operation. For read-heavy workloads — parsing configuration, deserializing cache entries, reading from a file format — this is a substantial performance difference.

## The Schema Language

Cap'n Proto schemas have a superficial resemblance to Protocol Buffers but differ in several important ways:

```capnp
@0xd5d8c89b5e00d43c;  # Unique file ID

struct PaymentRequest {
  idempotencyKey @0 :Text;
  amountCents @1 :Int64;
  currency @2 :Text;
  sourceToken @3 :Text;
  metadata @4 :List(KeyValue);
}

struct KeyValue {
  key @0 :Text;
  value @1 :Text;
}

interface PaymentService {
  charge @0 (request :PaymentRequest) -> (response :PaymentResponse);
  streamEvents @1 (filter :EventFilter) -> (stream :StreamHandle);
}
```

**Field numbering is explicit and permanent.** Like Protocol Buffers, fields are identified by integers, enabling backward-compatible evolution. Unlike Protocol Buffers, the field ordering affects the memory layout — higher-numbered fields cannot always be efficiently added without considering the layout.

**Unique file IDs.** The `@0xd5d8c89b5e00d43c` at the top is a unique identifier for this schema file. It enables schema-aware tools (like the Cap'n Proto reflection API) to identify schemas without relying on file paths.

**Interface types.** Cap'n Proto schemas can define interface types (RPC services), not just data types. This unifies the schema and the RPC definition — the same file describes both the data format and the service interface.

## Promise Pipelining — The Unusual Part

Cap'n Proto RPC includes a feature called promise pipelining (or time-travel RPC) that has no equivalent in gRPC, Thrift, or any other mainstream RPC framework.

In conventional RPC, calling a method that returns an object and then calling a method on that object requires two round trips:

```
Client → Server: getUser(id)
Server → Client: User { profile_capability: <capability_ref> }
Client → Server: profile_capability.getProfile()
Server → Client: Profile { ... }
```

With Cap'n Proto promise pipelining, you can chain calls before the first one completes:

```
Client → Server: getUser(id)
Client → Server: [promise from getUser].profile_capability.getProfile()
```

The second call is sent immediately, referring to the result of the first call by a "promise reference." The server receives both messages and can pipeline them — it processes the first, and when the result is ready, processes the second using that result, without requiring another client-server round trip.

For systems with high latency between components (cross-datacenter RPCs, satellite links), this can reduce latency proportional to the number of chained calls. A chain of five sequential calls that would require five round trips requires only one round trip with promise pipelining.

This is implemented through Cap'n Proto's capability system. Rather than returning raw data that the client must use to construct the next request, Cap'n Proto methods return capabilities — unforgeable references to objects that the client can call methods on. The capability table in a Cap'n Proto message is what enables promise pipelining at the protocol level.

## Capabilities as Security Primitives

Cap'n Proto's capability model has implications beyond performance.

A capability is a reference that grants authority to invoke operations on the referenced object. You cannot forge a capability — you can only acquire one by receiving it from someone who already has it. This is the object-capability model, and it is a formal security model with properties that ACL-based systems do not have.

In practice: if a Cap'n Proto server returns a capability to a resource, the client can invoke that capability. If the server does not return a capability, the client has no way to manufacture one. Least-privilege access control falls out naturally from the capability model — you give clients access to exactly what they need by giving them capabilities to exactly those objects.

This sounds theoretical but has practical consequences. A service that uses Cap'n Proto capabilities can ensure that clients can only access objects they have been explicitly granted access to, without maintaining a separate authorization table. The capability is the authorization; holding the capability proves authorization.

## FlatBuffers: Google's Answer

FlatBuffers was developed at Google, also pursuing zero-copy serialization. The design is similar to Cap'n Proto in the key respect: the wire format is directly usable as an in-memory format without a decode step.

The differences are in the design details:

- FlatBuffers does not have a native RPC system (Cap'n Proto does)
- FlatBuffers has better support for table modifications (adding fields with forward compatibility) without rewriting the entire buffer
- FlatBuffers is more permissive about mutating values in-place after creation
- FlatBuffers uses a builder pattern that results in back-to-front construction (objects are written in reverse, and the root object is at the end of the buffer); Cap'n Proto uses a more direct construction model
- Cap'n Proto has better support for very large messages that don't fit in a single allocation (through its segment model)

FlatBuffers is widely used in game development (it was created for game asset serialization) and in latency-critical financial systems. Cap'n Proto is used in systems where the RPC layer and capability security model matter, most notably Cloudflare Workers (which uses Cap'n Proto for internal RPC between worker processes).

## MessagePack

MessagePack is a different point in the design space: not zero-copy, but compact binary JSON. It uses a format that closely mirrors JSON's type system (null, boolean, integer, float, string, array, map) but encodes values in binary rather than text.

The key property: MessagePack messages are smaller than equivalent JSON and faster to parse than JSON, but there is no schema — the structure is ad-hoc, like JSON. MessagePack is appropriate when you want the flexibility of JSON with better performance, and you do not want to commit to a schema. It is not appropriate when you need the type safety, validation, or performance of a schema-based format.

Redis uses MessagePack internally. Many game backends use it for client-server communication where the flexibility of a schemaless format outweighs the performance cost.

## The Adoption Reality

Cap'n Proto is technically impressive. The zero-copy design, promise pipelining, and capability model are all genuine innovations. The adoption, however, is limited.

The reasons are practical:
- Language support is narrow. C++ and Rust have mature implementations. Python and Java have workable implementations. Other languages are hit-or-miss.
- The tooling ecosystem is thin compared to Protocol Buffers.
- Promise pipelining requires both client and server to be aware of it; mixed deployments with gRPC backends cannot benefit from it.
- The capability model is powerful but requires adopting a programming model that is unfamiliar to most engineers.

Cap'n Proto's primary real-world adoption is at Cloudflare (where it was developed in-house for Workers) and in systems where the zero-copy performance matters more than the library ecosystem. For most teams, Protocol Buffers offers a better tradeoff between performance and ecosystem.

## The Binary Frontier in Context

The protocols in this chapter — Cap'n Proto, FlatBuffers, MessagePack — represent a category of choice: binary serialization formats that are faster and more compact than JSON. They differ in whether they include an RPC layer (Cap'n Proto does; FlatBuffers and MessagePack do not), whether they require a schema (Cap'n Proto and FlatBuffers do; MessagePack does not), and how they handle encoding (zero-copy versus full decode).

The practical question is not "which binary format is theoretically best?" It is "what does my system need, and what can my team operate?"

If you are at the scale where JSON serialization overhead is a measurable cost — millions of messages per second, latency budgets in microseconds, processes that spend meaningful CPU time in JSON parse — binary serialization is worth the investment. Protocol Buffers is the safe default with the widest language support. FlatBuffers is the right choice if zero-copy matters and your language has a mature FlatBuffers library. Cap'n Proto is the right choice if you want the RPC layer and the capability model and you are committed to the C++ or Rust ecosystem.

If you are not at that scale, the performance of JSON is probably not your bottleneck, and the operational simplicity of human-readable payloads is worth more than a few microseconds of parse time.
