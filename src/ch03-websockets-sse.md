# WebSockets and SSE — When You Need the Server to Talk Back

The request-response model has a fundamental asymmetry: the client always initiates. The server can respond richly, but it cannot reach out first. For most of human history with the web, this was fine — users clicked things, servers answered. But modern applications broke this assumption. Chat applications need instant message delivery. Dashboards need live data. Multiplayer games need continuous state synchronization. Collaborative editors need sub-second propagation of every keystroke.

Two standards emerged from this need: WebSockets and Server-Sent Events. They solve the same surface-level problem — getting data from server to client without polling — but they solve it differently and are suited for different applications. Choosing between them is not obvious, and the "just use WebSockets for everything" instinct that many engineers carry is often wrong.

## The Polling Era and Why It Was Terrible

Before persistent connections, the two approaches were polling and long-polling.

**Polling** is the naive solution: the client sends a request every N seconds asking "anything new?" This is simple, stateless, and compatible with every HTTP infrastructure ever built. It is also wasteful in both directions — the server does work answering "no, nothing" requests, and the client experiences latency up to N seconds between an event occurring and it being delivered.

**Long-polling** is the clever hack: the client sends a request, and the server holds the connection open until it has something to say (or until a timeout forces a response). The client immediately re-connects after receiving a response. This delivers events with much lower latency than regular polling, but it still opens a new connection for each event cycle, burning TCP handshake overhead, and it creates subtle problems with connection limits, proxy timeouts, and load balancer behavior.

Both of these work. Companies ran real-time applications on long-polling for years. But they are workarounds for a protocol constraint, and the constraints are visible in the engineering required to make them reliable.

## WebSockets

WebSockets, standardized as RFC 6455 in 2011, replace the HTTP request-response model with a persistent, full-duplex, bidirectional channel. Either side can send a message to the other at any time, without the overhead of initiating a new request.

### The Handshake

A WebSocket connection starts as an HTTP/1.1 request with an `Upgrade` header:

```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

The server accepts the upgrade:

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

After the 101 response, the HTTP protocol is abandoned and the connection becomes a WebSocket connection. The TCP socket remains open, but the framing, multiplexing, and message boundary semantics are all defined by the WebSocket protocol rather than HTTP.

The `Sec-WebSocket-Key` / `Sec-WebSocket-Accept` exchange is not security — it is a sanity check to prevent the connection from being accidentally established by infrastructure that does not understand WebSockets. The server concatenates the client's key with a fixed GUID, hashes the result with SHA-1, and base64-encodes it. This proves that the server read the header, which is enough to prevent most accidental upgrades.

### The Frame Format

WebSocket messages are sent as frames. Each frame has a header of 2-14 bytes followed by the payload:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
```

Key fields:
- **FIN** indicates whether this is the last frame in a message (messages can be fragmented)
- **Opcode** identifies the frame type: 0x1 (text), 0x2 (binary), 0x8 (close), 0x9 (ping), 0xA (pong)
- **MASK** indicates whether the payload is masked. Client-to-server frames must be masked; server-to-client frames must not be
- **Payload len** encodes the payload length with a compact three-case encoding: 0-125 directly, 126 means the next 2 bytes hold the length, 127 means the next 8 bytes hold the length

The masking requirement for client-to-server frames is a security measure against proxy cache poisoning: a malicious client could craft WebSocket messages that, when interpreted as HTTP responses, poison an intermediary's cache. Random masking prevents this attack.

### What WebSockets Give You

Full duplex means both sides can send and receive simultaneously on the same connection. A chat server can push new messages to a client while the client is in the middle of composing a message, without any coordination required.

The message abstraction is useful. Unlike raw TCP, WebSockets have a notion of a complete message (which may span multiple frames). Your application code receives complete messages, not partial payloads.

No request overhead on each message. After the initial handshake, sending a message is just TCP write + WebSocket frame header. No HTTP headers, no method, no URL, no status code.

The protocol is also application-agnostic. You can carry any payload: JSON, Protocol Buffers, MessagePack, raw binary, whatever your application uses. WebSockets do not care about the content; they just deliver frames.

### What WebSockets Do Not Give You

WebSockets do not give you a message format, a serialization protocol, or any application-level semantics. You get a bidirectional byte pipe with message boundaries. What you do with it is entirely up to you.

This means every WebSocket application invents its own sub-protocol for things like:
- How does the server indicate which type of message this is?
- How does the client subscribe to specific event streams?
- How does request-response get modeled over a bidirectional channel?
- What happens when the client reconnects after a dropped connection?
- How are undelivered messages replayed?

None of this is specified. Some applications use JSON with a `type` field and a discriminated union pattern. Some use libraries like Socket.IO that layer an application protocol on top of WebSockets (and fall back to long-polling if WebSockets are unavailable). Some roll their own binary framing. The result is that WebSocket interoperability requires both sides to agree on an application protocol, which is typically documented nowhere formally.

### Scaling WebSockets

This is where most WebSocket-naive implementations encounter their first serious surprise.

HTTP is stateless. Any request can be handled by any server behind a load balancer. WebSockets are stateful: the connection is persistent and the server holds per-connection state. If a client is connected to server A, all messages to that client must be delivered through server A — or server A must be able to route messages to the client, which requires a pub/sub layer across servers.

The standard approach is a Redis pub/sub layer (or equivalent). When server A needs to deliver a message to a client connected to server B, it publishes to a channel; server B is subscribed to that channel and delivers the message over its connection. This works, but it adds operational complexity and a hop.

Load balancers must use sticky sessions (IP hash or cookie-based affinity) or understand WebSockets well enough to route by connection. AWS ALB, nginx, and HAProxy all handle WebSocket connections correctly with appropriate configuration; the issue is usually that default configurations do not.

Connection counts matter differently than with HTTP. A REST server handling 10,000 requests per second can do so over a pool of connections that is much smaller than 10,000, because connections are short-lived. A WebSocket server may have 100,000 simultaneously open connections, each representing a client that has not disconnected. Operating system limits on file descriptors, memory per connection, and TCP state become relevant at scale.

## Server-Sent Events

Server-Sent Events (SSE) is the less famous sibling: a standard for server-to-client streaming over plain HTTP, defined in the HTML specification. Where WebSockets are full-duplex and transport-layer, SSE is unidirectional and application-layer.

### The SSE Model

An SSE connection starts as a standard HTTP GET request. The server responds with `Content-Type: text/event-stream` and keeps the connection open, sending text-formatted events as they occur:

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

data: {"type": "price_update", "symbol": "AAPL", "price": 182.47}

data: {"type": "price_update", "symbol": "AAPL", "price": 182.51}

id: 42
event: alert
data: {"message": "Circuit breaker triggered", "symbol": "GOOG"}

```

The event format is simple:
- `data:` lines carry the payload. Multiple `data:` lines for one event are concatenated with newlines.
- `id:` sets the event ID, which the client uses for reconnection.
- `event:` sets a named event type (optional; default is `message`).
- `retry:` suggests a reconnection delay in milliseconds.
- A blank line terminates an event.

That is the entire protocol. It is so simple you could implement it in a few lines of any language.

### Auto-Reconnection

SSE has built-in reconnection semantics that WebSockets do not. The browser's `EventSource` API automatically reconnects when the connection drops, sending the `Last-Event-ID` header with the ID of the last received event. The server can use this to replay missed events from where the client left off.

This is enormously useful for reliability. A WebSocket application must implement its own reconnection logic, including deciding how to detect that the connection is dead (ping/pong, application-level heartbeats), how to track the last received state, and how to replay messages after reconnection. SSE gives you this for free.

### The Browser EventSource API

```javascript
const source = new EventSource('/api/events');

source.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('Received:', data);
};

source.addEventListener('alert', (event) => {
  showAlert(JSON.parse(event.data));
});

source.onerror = (error) => {
  // EventSource will automatically reconnect; this handler fires on errors
  console.error('SSE error:', error);
};
```

The EventSource API does not expose the reconnection logic; it just handles it. The application code sees a stream of events.

### SSE's Actual Limitations

**HTTP/1.1 connection limit.** Browsers limit concurrent connections to the same origin to six (HTTP/1.1) or unlimited with connection coalescing (HTTP/2). SSE opens a long-lived connection, which uses one of those six slots. If a page opens multiple SSE connections, it can exhaust the limit. HTTP/2 alleviates this significantly since all streams share a connection.

**No binary support in the browser API.** The EventSource API only handles text. If you want to send binary data, you must base64-encode it or use another mechanism. This is a browser-API limitation, not a protocol limitation — the SSE protocol is text-based anyway.

**Unidirectional.** SSE carries data from server to client only. The client sends requests through normal HTTP. For most push-notification use cases, this is fine; for cases where the client needs to send a stream of data to the server, SSE cannot help.

**Some proxies buffer the response.** Nginx's default buffering will collect the SSE response body before forwarding it to the client, which breaks the streaming semantics entirely. You must configure `proxy_buffering off` for SSE endpoints. Similar issues exist with other proxy software.

## Choosing Between WebSockets and SSE

The choice is often clearer than it appears.

**Use SSE when:**
- You primarily need server-to-client push (notifications, live feeds, dashboards)
- You want built-in reconnection with state recovery
- You want to use standard HTTP infrastructure (CDNs, proxies, load balancers without sticky sessions)
- Your clients are browsers and you value simplicity
- You do not need to send a stream of data from client to server

**Use WebSockets when:**
- You need true bidirectional messaging (chat, collaborative editing, multiplayer games)
- You have high message volume in both directions
- You need binary streaming (audio, video, game state)
- You are not limited to browsers and want more control over the framing

The common failure mode is choosing WebSockets when SSE would have been sufficient. WebSockets are more complex to scale, require sticky sessions or a pub/sub layer, have no built-in reconnection, and require you to design your own application protocol. If you are building a live dashboard that pushes server state to passive viewers, SSE is half the operational complexity and twice the reliability for exactly that use case.

WebSockets win when the server and client are participants in a symmetric conversation — both sides initiating messages, both sides listening. SSE wins when the relationship is fundamentally asymmetric: the server has information; the client wants to receive it.

## A Note on Socket.IO

Socket.IO is frequently described as a WebSocket library but is better understood as an abstraction layer over multiple transports, with WebSockets being the preferred transport and long-polling as a fallback. It adds rooms, namespaces, automatic reconnection, acknowledgments, and broadcasting to the raw WebSocket model.

Socket.IO is not the WebSocket protocol. A standard WebSocket client cannot connect to a Socket.IO server; a Socket.IO client cannot connect to a standard WebSocket server. This matters if you are designing an API that non-browser clients will consume — you are committing to Socket.IO's protocol on both sides.

The library is a reasonable choice for browser-heavy applications where the application semantics it provides (rooms, broadcasts, namespaced channels) match your needs. It is a poor choice if you want protocol interoperability or if you are running in an environment where the fallback to long-polling is not needed (most modern mobile clients support WebSockets natively).

## Operational Reality

Both WebSockets and SSE are harder to operate than REST endpoints in certain specific ways.

Connection state means your autoscaling story changes. With stateless HTTP, adding or removing servers is transparent. With persistent connections, removing a server means dropping all its connections. Clients will reconnect, but they will experience an interruption. Blue/green deployments require graceful connection draining rather than hard cutover.

Both protocols interact oddly with TLS termination at load balancers. WebSockets require the load balancer to forward the upgraded connection, not just proxy HTTP/1.1. SSE requires the response not to be buffered. These are configured per-server, and the defaults are usually wrong.

Monitoring connection counts, message rates, and connection lifetimes requires tooling that most generic HTTP monitoring systems do not provide. You will need to instrument your application code.

None of these problems are unsolvable. They are just problems that do not exist with stateless REST, and you should plan for them before they surprise you in production at 2 AM.
