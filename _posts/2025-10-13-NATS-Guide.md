---
title: "NATS | When, Why and How"
date: 2025-10-13
author: GonzaloMB
tags: ["nats", "microservices"]
---

NATS is a high‑performance messaging system for building fast, simple, and resilient services. This guide explains when and why to use NATS, how it compares to HTTP, and shows concise TypeScript/JavaScript examples for Pub/Sub, Request‑Reply, Work Queues, and JetStream persistence.

---

## When To Use NATS

- Low‑latency messaging between microservices where HTTP adds overhead.
- Fan‑out (one publisher → many subscribers) or event‑driven architectures.
- Work queues with competing consumers to process jobs in parallel.
- Request‑Reply RPC without HTTP (service‑to‑service calls over NATS).
- Durable streams, replay, and at‑least‑once delivery via JetStream.
- Multi‑language, multi‑platform systems (NATS clients exist for many stacks).

When to stick with HTTP:

- Public APIs for browsers and partners; caching/proxies matter.
- Strict REST semantics, standard intermediaries, or API gateways required.

---

## Why NATS

- Simplicity: subjects instead of topics/queues/exchanges.
- Speed: lightweight protocol, low latency, high throughput.
- Patterns: Pub/Sub, Request‑Reply, Queue Groups, Key/Value, Object Store.
- Scalability: clustering and leaf nodes; global mesh via JetStream.
- Reliability (JetStream): persistence, replay, consumers with acks.

---

## Communication Patterns vs. HTTP

- Pub/Sub (fire‑and‑forget events) ≈ HTTP webhook broadcasts, but push to many subscribers with backpressure via consumer design.
- Request‑Reply (RPC) ≈ HTTP request/response, but over a single TCP connection with lower latency and built‑in load balancing.
- Queue Groups ≈ HTTP load‑balanced workers; only one consumer in the group receives each message.
- Streams (JetStream) ≈ HTTP + queue + DB for durable event logs, with pull/push consumers and replay.

You can build HTTP‑like semantics on NATS using Request‑Reply, headers, and subject conventions, while avoiding HTTP servers where not needed.

---

## Setup

1. Run NATS locally (Docker):

```bash
docker run -p 4222:4222 -ti nats:2 -js
```

2. Install client:

```bash
npm i nats
```

3. Connect (TS/JS):

```ts
import { connect, StringCodec } from "nats";

async function main() {
  const nc = await connect({ servers: "nats://localhost:4222" });
  const sc = StringCodec();
  console.log("connected:", nc.getServer());

  // Close gracefully
  const done = nc.closed();
  nc.closed().then(() => console.log("connection closed"));
  process.on("SIGINT", async () => {
    await nc.drain();
  });
}
main();
```

---

## Pub/Sub (Broadcast)

Publisher:

```ts
import { connect, StringCodec } from "nats";
const sc = StringCodec();
(async () => {
  const nc = await connect({ servers: "nats://localhost:4222" });
  nc.publish(
    "orders.created",
    sc.encode(JSON.stringify({ id: "o1", total: 42 }))
  );
  await nc.flush();
  await nc.drain();
})();
```

Subscriber:

```ts
import { connect, StringCodec } from "nats";
const sc = StringCodec();
(async () => {
  const nc = await connect({ servers: "nats://localhost:4222" });
  const sub = nc.subscribe("orders.created");
  for await (const m of sub) {
    console.log("event", m.subject, JSON.parse(sc.decode(m.data)));
  }
})();
```

Subject design tip: use dot‑separated nouns/verbs (`orders.created`, `payments.captured`).

---

## Request‑Reply (HTTP‑like RPC)

Service (responder):

```ts
import { connect, StringCodec } from "nats";
const sc = StringCodec();
(async () => {
  const nc = await connect({ servers: "nats://localhost:4222" });
  const sub = nc.subscribe("inventory.get");
  for await (const m of sub) {
    const { sku } = JSON.parse(sc.decode(m.data));
    const qty = 7; // lookup...
    m.respond(sc.encode(JSON.stringify({ sku, qty })));
  }
})();
```

Client (requester):

```ts
import { connect, StringCodec } from "nats";
const sc = StringCodec();
(async () => {
  const nc = await connect({ servers: "nats://localhost:4222" });
  const msg = await nc.request(
    "inventory.get",
    sc.encode(JSON.stringify({ sku: "SKU-123" })),
    { timeout: 1000 }
  );
  console.log("reply:", JSON.parse(sc.decode(msg.data)));
})();
```

Note: You can attach headers (`import { headers }`) to carry metadata like auth, correlation IDs, or content type.

---

## Work Queues (Competing Consumers)

Only one consumer in a queue group receives each message.

Subscriber workers:

```ts
import { connect, StringCodec } from "nats";
const sc = StringCodec();
(async () => {
  const nc = await connect({ servers: "nats://localhost:4222" });
  // queue: "workers"
  const sub = nc.subscribe("images.resize", { queue: "workers" });
  for await (const m of sub) {
    const job = JSON.parse(sc.decode(m.data));
    console.log("processing", job.id);
  }
})();
```

Publisher:

```ts
import { connect, StringCodec } from "nats";
const sc = StringCodec();
(async () => {
  const nc = await connect({ servers: "nats://localhost:4222" });
  for (let i = 1; i <= 10; i++)
    nc.publish("images.resize", sc.encode(JSON.stringify({ id: i })));
  await nc.drain();
})();
```

---

## JetStream (Durable Streams, Replay)

Create a stream, publish, and consume with a durable.

```ts
import { connect, StringCodec, consumerOpts } from "nats";
const sc = StringCodec();
(async () => {
  const nc = await connect({ servers: "nats://localhost:4222" });
  const jsm = await nc.jetstreamManager();

  // idempotent upsert of a stream
  await jsm.streams
    .add({ name: "ORDERS", subjects: ["orders.*"] })
    .catch(() => {});

  const js = nc.jetstream();
  await js.publish("orders.created", sc.encode(JSON.stringify({ id: "o1" })));

  const opts = consumerOpts();
  opts.durable("ORDERS_WORKER");
  opts.manualAck();
  opts.ackExplicit();
  opts.deliverAll(); // or .startSequence / .startTime

  const sub = await js.subscribe("orders.*", opts);
  for await (const m of sub) {
    const ev = JSON.parse(sc.decode(m.data));
    console.log("replayed/event:", ev);
    m.ack(); // at‑least‑once
  }
})();
```

Also available: Key/Value store (`await nc.jetstream().views.kv(...)`) and Object Store for blobs.

---

## Designing HTTP‑Like APIs on NATS

- Subjects as "routes": `svc.user.get`, `svc.user.update`.
- Request‑Reply for sync operations; Pub/Sub for domain events.
- Headers for `authorization`, `x-correlation-id`, `content-type`.
- Service discovery: register multiple responders; NATS balances requests.
- Versioning: prefix subjects (e.g., `v1.svc.user.get`).

Example with headers:

```ts
import { connect, StringCodec, headers } from "nats";
const sc = StringCodec();
(async () => {
  const nc = await connect({ servers: "nats://localhost:4222" });
  const h = headers();
  h.set("authorization", "Bearer <token>");
  h.set("content-type", "application/json");
  const msg = await nc.request(
    "v1.svc.user.get",
    sc.encode(JSON.stringify({ id: "u1" })),
    { headers: h, timeout: 1000 }
  );
  console.log(JSON.parse(sc.decode(msg.data)));
})();
```

---

## Operational Tips

- Use timeouts on `request()` to avoid hanging calls.
- Drain before exit for graceful shutdowns (`await nc.drain()`).
- Standardize subject naming and headers (contracts).
- For durability and backpressure, prefer JetStream consumers with manual acks.
- Use queue groups for horizontal scale without duplicates.

---

## Benefits (TL;DR)

- Lower latency and resource usage vs. HTTP for service‑to‑service.
- Built‑in patterns without extra infra (queues, RPC, streams).
- Horizontal scalability via queue groups and clustering.
- Persistence, replay, and at‑least‑once delivery with JetStream.
- Polyglot clients; simple operational model.

---
