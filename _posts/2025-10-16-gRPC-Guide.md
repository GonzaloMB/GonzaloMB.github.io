---
title: "gRPC | When, Why and How"
date: 2025-10-16
author: GonzaloMB
tags: ["grpc", "microservices"]
---

gRPC is a high‑performance RPC framework built on HTTP/2 and Protocol Buffers. This guide covers when to use it, why to choose it over plain HTTP/REST for service‑to‑service traffic, and includes concise Node.js (JavaScript) examples for unary RPC, streaming, deadlines, errors, security, and best practices.

---

## When To Use gRPC

- Low‑latency service‑to‑service calls inside the backend.
- Strongly typed, evolvable contracts (Proto3, backward compatibility).
- Efficient streaming (server/client/bidirectional) over HTTP/2.
- Connection multiplexing and better tail latencies under load.
- Polyglot interoperability with generated stubs.

When to prefer HTTP/REST:

- Public, browser‑facing or partner APIs (caches/proxies/CDN).
- REST semantics and the broader HTTP ecosystem (gateways, caches, tooling).

---

## Why gRPC

- Efficiency: HTTP/2, header compression, multiplexing, true streaming.
- Contracts: `.proto` as the single source of truth; client/server codegen.
- Safe evolution: optional fields, stable numbering, binary compatibility.
- Observability and control: deadlines, cancellation, standardized status codes.
- Ecosystem: interceptors, load balancing, mTLS, resolvers, health checks.

---

## Patterns vs. HTTP/REST

- Unary RPC ≈ REST request/response, but with lower overhead and strong types.
- Server streaming ≈ long polling/SSE, but binary and efficient.
- Client streaming ≈ uploading batches/chunks without app‑level retries.
- Bidirectional streaming ≈ WebSocket, with backpressure and `.proto` contracts.

You can expose REST externally while using gRPC internally, or place an HTTP/JSON → gRPC gateway in front.

---

## Quick Setup (Node.js)

1. Install dependencies:

```bash
npm i @grpc/grpc-js @grpc/proto-loader
```

2. Define the contract (`protos/hello.proto`):

```proto
syntax = "proto3";
package hello;

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply) {}
  rpc StreamGreetings (HelloRequest) returns (stream HelloReply) {}
}

message HelloRequest { string name = 1; }
message HelloReply { string message = 1; }
```

3. Server (unary + server streaming):

```js
const path = require("node:path");
const grpc = require("@grpc/grpc-js");
const protoLoader = require("@grpc/proto-loader");

const packageDefinition = protoLoader.loadSync(
  path.join(__dirname, "protos/hello.proto"),
  {
    keepCase: false,
    longs: String,
    enums: String,
    defaults: true,
    oneofs: true,
  }
);
const proto = grpc.loadPackageDefinition(packageDefinition).hello;

function sayHello(call, callback) {
  const name = call.request && call.request.name ? call.request.name : "world";
  callback(null, { message: `Hello, ${name}` });
}

async function streamGreetings(call) {
  const name = call.request && call.request.name ? call.request.name : "world";
  for (const i of [1, 2, 3]) {
    call.write({ message: `Hello ${name} #${i}` });
    await new Promise((r) => setTimeout(r, 300));
  }
  call.end();
}

const server = new grpc.Server();
server.addService(proto.Greeter.service, {
  SayHello: sayHello,
  StreamGreetings: streamGreetings,
});
server.bindAsync("0.0.0.0:50051", grpc.ServerCredentials.createInsecure(), () =>
  server.start()
);
console.log("gRPC server on 50051");
```

4. Client (unary + server streaming + deadline):

```js
const path = require("node:path");
const grpc = require("@grpc/grpc-js");
const protoLoader = require("@grpc/proto-loader");

const packageDefinition = protoLoader.loadSync(
  path.join(__dirname, "protos/hello.proto")
);
const proto = grpc.loadPackageDefinition(packageDefinition).hello;
const client = new proto.Greeter(
  "localhost:50051",
  grpc.credentials.createInsecure()
);

// Unary with deadline
const deadline = new Date(Date.now() + 500); // 500ms
client.SayHello({ name: "Ada" }, { deadline }, (err, res) => {
  if (err) return console.error("error:", err.code, err.message);
  console.log("unary:", res.message);
});

// Server streaming
const stream = client.StreamGreetings({ name: "Ada" });
stream.on("data", (m) => console.log("stream:", m.message));
stream.on("end", () => console.log("done"));
```

---

## RPC Types

- Unary: 1 request → 1 response. Use deadlines and handle errors via `ServiceError.code`.
- Server streaming: 1→N responses. Listen to `data` and close with `end`.
- Client streaming: N→1 response. Useful for batches; finalize with `end`.
- Bidirectional: N↔N. Keep message protocols simple and document sequencing.

Tip: avoid huge messages; prefer chunking or references to external object storage.

---

## Protocol Buffers (Safe Evolution)

- Don’t reuse field numbers; mark obsolete ones as `reserved`.
- Prefer `optional` over breaking compatibility when removing fields.
- Model business errors with gRPC status plus structured details when needed.

Evolution example:

```proto
message User {
  string id = 1;
  string email = 2; // reserved 3 (old phone)
  optional string display_name = 4;
}

reserved 3; // legacy field
```

---

## Errors, Deadlines and Retries

- Always use deadlines; avoid unbounded wait queues.
- Propagate cancellations: if the client cancels, stop server work.
- Retry only idempotent codes (`UNAVAILABLE`, `DEADLINE_EXCEEDED`); use exponential backoff with jitter.
- Map domain errors to appropriate codes (`FAILED_PRECONDITION`, `PERMISSION_DENIED`, `NOT_FOUND`).

Snippet (Node.js):

```js
const grpc = require("@grpc/grpc-js");

function withDeadline(ms) {
  return { deadline: new Date(Date.now() + ms) };
}

function shouldRetry(code) {
  return (
    code === grpc.status.UNAVAILABLE || code === grpc.status.DEADLINE_EXCEEDED
  );
}
```

---

## Security

- TLS by default; consider mTLS for the internal mesh.
- Validate SANs/hostnames; rotate certificates.
- App‑layer auth: JWT/OIDC in metadata (`authorization: Bearer ...`).

TLS example (client):

```js
const grpc = require("@grpc/grpc-js");
const { readFileSync } = require("node:fs");

const rootCerts = readFileSync("certs/ca.pem");
const creds = grpc.credentials.createSsl(rootCerts);
// const client = new proto.Greeter("greeter.prod:443", creds);
```

---

## Observability

- Log calls with interceptors (client/server).
- Metrics: per‑method latencies, status codes, concurrency.
- Distributed traces: propagate/extract context in metadata.

Client interceptor (minimal logging):

```js
const grpc = require("@grpc/grpc-js");

const loggingInterceptor = (options, nextCall) => {
  const method = options.method_definition.path;
  const start = Date.now();
  const requester = {
    start: (metadata, listener, next) => {
      const newListener = {
        onReceiveStatus: (status, nextStatus) => {
          console.log(method, status.code, Date.now() - start + "ms");
          nextStatus(status);
        },
      };
      next(metadata, newListener);
    },
  };
  return new grpc.InterceptingCall(nextCall(options), requester);
};
```

---

## Best Practices

- Design APIs around use cases, not tables.
- Document `.proto` contracts clearly; version packages (`package v1...`).
- Keep messages small; use streaming/backpressure where appropriate.
- Apply deadlines by default and consistent end‑to‑end timeouts.
- Add health checks (`grpc.health.v1.Health`) and readiness probes.
- Automate stub generation in CI to prevent drift.

---

## Benefits (TL;DR)

- Lower latency and better CPU/network use vs. REST for S2S.
- Strong, evolvable contracts with Proto3.
- Native streaming and backpressure control.
- Mature tooling: interceptors, security, observability, balancing.

---

## Resources

- Official docs: https://grpc.io/docs/
- Status codes spec: https://grpc.github.io/grpc/core/md_doc_statuscodes.html
- Health Checking: https://github.com/grpc/grpc/blob/master/doc/health-checking.md

---
