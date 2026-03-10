# HTTP Alternatives for APIs

A survey of API communication protocols beyond HTTP — gRPC, MQTT, AMQP, ZeroMQ, Cap'n Proto RPC, WebSockets, SSE, and more.

Most API code is written without a protocol decision. Engineers reach for REST over HTTP because it is what they know, what the framework assumes, and what everyone else uses. That is a reasonable default — but it is still a choice, and like any invisible choice, it has costs.

This book surveys the alternatives: what they are, how they work, where they outperform REST, and where they do not.

## Read Online

Published at **https://cloudstreet-dev.github.io/HTTP-Alternatives-for-APIs/**

## Contents

1. The Default We Never Questioned
2. gRPC — HTTP/2 in a Convincing Costume
3. WebSockets and SSE — When You Need the Server to Talk Back
4. MQTT and the IoT World
5. AMQP and Message Brokers
6. ZeroMQ — Messaging Without a Broker
7. Cap'n Proto RPC and the Binary Frontier
8. Lesser-Known Contenders
9. What Nobody Has Tried Yet
10. How to Choose — A Decision Framework

## Building Locally

Requires [mdBook](https://rust-lang.github.io/mdBook/).

```sh
cargo install mdbook
mdbook serve
```

## Acknowledgments

Thanks to **Georgiy Treyvus**, CloudStreet Product Manager, whose idea started this book.
