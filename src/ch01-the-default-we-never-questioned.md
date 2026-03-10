# The Default We Never Questioned

There is a particular kind of decision that engineers make without making it. Nobody sits down and evaluates REST against the alternatives, weighs the tradeoffs, and concludes that REST is the right choice for their use case. They just write a controller, annotate it with `@GetMapping` or decorate it with `app.get(...)`, and move on. REST has won by becoming invisible — not by being optimal, but by being the assumed baseline from which all other choices must justify themselves.

This book is about questioning that assumption. Not because REST is bad — it is often perfectly adequate — but because "adequate" and "optimal" are different things, and the gap between them has real costs. Once you know what else exists and when it matters, you can make an actual decision instead of an accidental one.

## How REST Got Here

REST — Representational State Transfer — was described by Roy Fielding in his 2000 dissertation as an architectural style for distributed hypermedia systems. The core insight was elegant: treat everything as a resource, use uniform interfaces to manipulate it, keep interactions stateless, and let the web's existing infrastructure (caches, proxies, load balancers) do useful work.

It was a description of the web as it already worked, not a prescription for building APIs. APIs came later, and they borrowed REST's vocabulary without always inheriting its constraints. What most people call "REST APIs" today are better described as HTTP APIs with JSON bodies — they use HTTP verbs and status codes, but they frequently violate Fielding's constraints, omit HATEOAS entirely, and treat URLs as a naming convention rather than a hypermedia control system. That is not necessarily wrong; it is just not what Fielding described.

The practical story is simpler: HTTP was everywhere. Every language had HTTP client libraries. Every firewall understood HTTP. Every operations team knew how to proxy and load-balance it. JSON emerged as a readable, flexible data format that worked in browsers without a parsing step. The combination was frictionless in a way that nothing else was, and frictionless wins — especially when you are moving fast and the infrastructure costs of something more specialized are real.

## The Costs Nobody Charges You For

REST's ubiquity has a way of making its costs invisible. They are real, but they are baked into the baseline, so you pay them without noticing.

**Request-response is the only primitive.** HTTP is a synchronous request-response protocol. The client asks; the server answers. This maps naturally to fetching data, but it maps poorly to many other communication patterns: server-initiated events, streaming data, fire-and-forget notifications, bidirectional negotiation. Every time you need something other than ask-and-answer, you are working around the protocol: polling loops, long-polling hacks, webhook callbacks, SSE bolted on as an afterthought.

**Connections are expensive and reused poorly.** HTTP/1.1 pipelining was supposed to help with connection overhead, but browser and proxy implementations were so buggy that it was effectively abandoned. The practical result was that browsers opened six connections per host and serialized requests across them, burning three-way handshake overhead on every request to a cold connection. HTTP/2 multiplexing helped significantly, but most internal services — running over TCP between components that control both ends — are still paying connection overhead that custom protocols would not incur.

**Text is readable, but not efficient.** JSON's human-readability is genuinely valuable during development. It is also genuinely wasteful at scale. Parsing JSON is not free: a service processing a million requests per second is spending a measurable fraction of its CPU budget deserializing JSON that it will immediately re-serialize when forwarding to the next tier. More subtly, JSON has no schema at the wire level, which means the recipient must validate structure at runtime and the compiler cannot help you when you add a field on one side without updating the other.

**URLs are a weak contract.** The URL space is not typed, not versioned, and not machine-verifiable. You can document it with OpenAPI and generate clients from that documentation, but the schema is one artifact among many rather than a compiler-enforced contract. The gap between the OpenAPI spec and the actual server behavior is bridged by tests, vigilance, and prayer. This is manageable, but it is friction that other protocols eliminate.

**HTTP semantics are frequently abused.** The HTTP specification defines clear semantics for GET (safe, idempotent), POST (non-idempotent create or action), PUT (idempotent replace), PATCH (partial update), DELETE (remove). In practice, many APIs use POST for everything non-trivial, return 200 for errors, and embed status information in JSON response bodies because actually reading HTTP status codes from JavaScript requires effort. The protocol's semantic richness is largely unused.

## The Myth of Universal Tooling

The usual argument for REST is tooling: curl, Postman, browsers, logging proxies, load balancers — everything understands HTTP. This is true, and it matters. But the argument is weakening.

gRPC has server reflection and grpcurl. MQTT has tools like MQTT Explorer and mosquitto_pub. Kafka has an entire ecosystem. The days when choosing a non-HTTP protocol meant building your own tooling from scratch are largely over. The tooling argument is still a real consideration — HTTP's tooling ecosystem is larger and more mature than any alternative — but it should be weighed honestly against other factors rather than treated as a trump card.

The argument also inverts strangely when applied internally. For service-to-service communication inside a microservice architecture, the "every firewall understands HTTP" argument is irrelevant — there is no firewall between your services. The "debugging with curl" argument matters less when your real debugging workflow involves distributed traces and structured logs. Internal services have different constraints than public APIs, and they are often better served by protocols optimized for internal use.

## When REST Actually Is Right

This book is not an argument against REST. It is an argument against REST by default. The distinction matters.

REST over HTTP/JSON is a strong choice when:

- **You are building a public API.** Developers consuming your API should not need special client libraries. JSON over HTTP is universal, readable, and explorable. The documentation burden of a structured binary protocol is not worth it for a public surface.

- **Your clients are browsers.** Browsers speak HTTP natively. WebSockets and SSE extend HTTP. Everything else requires a gateway or a workaround.

- **Caching matters.** HTTP has a sophisticated, well-understood, widely-implemented caching model. GET requests are cacheable by CDNs, reverse proxies, and browsers without any additional work. Few other protocols have anything comparable.

- **Your team is small and moving fast.** The operational overhead of running MQTT brokers, learning Protocol Buffers, or debugging ZeroMQ socket state is real. For a small team with limited operational capacity, REST's simplicity is not just convenience — it is a strategic choice that keeps the system maintainable.

- **Your traffic patterns fit request-response.** If users are clicking buttons and waiting for results, and your data model is resources-on-a-server, REST is doing exactly what it was designed to do. Do not optimize for problems you do not have.

REST's costs appear when these conditions do not hold: when you are building for constrained devices, when you need true bidirectionality, when you are processing millions of events per second, when you need guaranteed delivery semantics, when your system is a distributed pipeline rather than a collection of resources. These are the cases where the alternatives covered in this book offer something that REST cannot.

## Reading This Book

The chapters that follow cover each major alternative in depth: what it actually is (not just what the marketing says), how it works mechanically, where it fits, and where it does not. Some of these — gRPC in particular — are widely called alternatives to HTTP while actually being built on top of it. The costume is convincing enough to merit its own chapter.

The goal is not to give you a flowchart that tells you which protocol to pick. Protocol selection is a judgment call that depends on context that no flowchart can encode. The goal is to make sure that when you make the call, you are making it with full information rather than with the vague sense that REST is what everyone does.

Everyone doing something does not make it optimal. It just makes it the default.
