# Module 1 — The Node.js runtime before Express

[← Course home](../README.md) · [Roadmap](00-stack-and-roadmap.md) · Next: [HTTP and raw Node server →](02-http-node-http.md)

**Baseline:** Node.js 24.21.0 (Active LTS), official API references from the Node 24 line. Features that are experimental or absent on Node 24 will be labeled; do not assume a Node 26 Current API is automatically a production baseline.

## Concept — what Node.js actually is

Node.js is a JavaScript runtime built on V8, plus platform/runtime components that make JavaScript useful for command-line programs and network services. It is not Express, not a web server by itself, and not simply “JavaScript with a server.” Use the Node API page for the selected major: **Stability 2 = Stable** is the normal baseline; **Experimental** APIs can change and are not capstone dependencies; **Deprecated** APIs should be migrated away from; **Legacy** APIs may remain for compatibility but should not be the default design. Stability labels are per API, not per whole module.

A simplified request path:

```text
Browser
  ↓ HTTP bytes
Operating system (network interface, socket buffers, TCP)
  ↓ readable socket / events
Node.js runtime (V8 + libuv + native bindings + Node APIs)
  ↓ JavaScript callback / promise continuation
Application code
  ↓ write response bytes
Node HTTP server → operating system → network → browser
```

### Runtime pieces

- **V8** parses and compiles JavaScript and executes it in an isolate. JavaScript tasks run one at a time on a given event-loop thread unless work is explicitly moved to another worker/isolate.
- **Node core APIs** expose capabilities such as `node:http`, `node:fs`, `node:crypto`, and streams.
- **Native bindings** connect JavaScript-facing APIs to C/C++ and platform facilities.
- **libuv** supplies the event loop and cross-platform asynchronous infrastructure. Some operations use OS readiness/completion mechanisms; selected work such as many filesystem operations and some crypto/DNS operations can use a bounded libuv thread pool.
- **Operating system** owns sockets, file descriptors, scheduling, TCP, TLS primitives, and other resources.
- **Worker Threads** run JavaScript in separate isolates/threads and are useful for CPU-heavy computation. They do not make shared mutable application state safe by magic.
- **Child processes** are separate OS processes. They have stronger process isolation and are useful for external programs or separate process roles, but need lifecycle, IPC, and security handling.

> **Correct mental model:** Node.js is highly concurrent for I/O, while JavaScript execution inside one isolate is generally serialized. “Single-threaded” is a useful narrow description of a particular JS execution lane, but a misleading description of the whole runtime and its work.

## Mental model — event loop, queues, and callback execution

```text
                    ┌────────────────────────────┐
                    │ JavaScript call stack      │
                    │ one callback runs to end   │
                    └─────────────┬──────────────┘
                                  │ yields / returns
         ┌────────────────────────┼────────────────────────┐
         │                        │                        │
  nextTick queue          Promise microtask queue     event-loop phases
  process.nextTick()      .then / await continuation timers → poll → check
                                                        → close callbacks
         │                        │                        │
         └────────────── event loop picks ready work ──────┘
```

Node event-loop phase names are useful—**timers, pending callbacks, idle/prepare, poll, check, close callbacks**—but do not treat the loop as a fixed, globally deterministic list that runs every callback every turn. The poll phase processes I/O; `setImmediate` callbacks run in check; timers are eligible after thresholds, not guaranteed wall-clock deadlines. `process.nextTick` is a Node-specific queue with priority over ordinary Promise microtasks in Node scheduling; recursively filling it can starve I/O.

### The ordering exercise

```js
console.log("A");

setTimeout(() => console.log("B"), 0);
Promise.resolve().then(() => console.log("C"));
process.nextTick(() => console.log("D"));
```

For this top-level CommonJS script on the baseline Node line, the expected order is **A, D, C, B**: synchronous code first, then Node's next-tick queue, then Promise microtasks, then timer work. Top-level ESM evaluation can change the relative observation of `process.nextTick` and Promise microtasks because module evaluation itself participates in microtask processing; do not teach a single order detached from context. `setTimeout(fn, 0)` is not “run immediately,” and `setImmediate` versus a timer can depend on whether they are scheduled from top-level code or an I/O callback.

**Exercise:** run as both `.cjs` and `.mjs`; add a `setImmediate`; move scheduling into a `node:fs` callback; write down the ordering you actually observe and explain which ordering guarantees are strong versus incidental.

## Async programming: waiting, concurrency, cancellation

Callbacks predate Promises and remain common in event APIs; Promises model one eventual result; `async`/`await` makes Promise control flow readable. An `await` pauses the current async function—not the entire event loop.

```ts
// BAD: independent I/O forced to happen one after another.
const customer = await getCustomer(customerId);
const recommendations = await getRecommendations(customerId);

// Better: start independent work together.
const [customer, recommendations] = await Promise.all([
  getCustomer(customerId),
  getRecommendations(customerId),
]);
```

`Promise.all` rejects on the first rejection but does not cancel sibling work. Use `Promise.allSettled` when all outcomes matter; `Promise.race` for “first settled” (which does not cancel losers); `Promise.any` for “first fulfilled.” Bound concurrency for large collections—launching 50,000 requests at once is not a good plan. Use an `AbortController` signal with APIs that accept one:

```ts
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 2_000);

try {
  const response = await fetch("https://inventory.example.test/items/42", {
    signal: controller.signal,
    headers: { accept: "application/json" },
  });
  if (!response.ok) throw new Error(`Inventory returned ${response.status}`);
  const raw: unknown = await response.json();
  // A later module validates raw data before trusting it.
} finally {
  clearTimeout(timeout);
}
```

Cancellation means “stop work where supported,” not “undo a database commit.” Propagate signals intentionally and define an end-to-end deadline across downstream calls.

## Node core APIs backend engineers actually use

| API | Practical backend job |
|---|---|
| `node:http`, `node:https` | Raw HTTP servers/clients, headers, sockets, upgrade events; build understanding before a framework. |
| `node:fs/promises`, `node:path`, `node:url` | Async file access, safe path construction, URL parsing; avoid sync file calls in request handlers. |
| `node:crypto` | Random IDs, hashes, HMAC/signatures, constant-time comparisons; password hashing should use a maintained Argon2/bcrypt library rather than a homegrown scheme. |
| `node:stream`, `node:stream/promises`, `node:buffer` | Incremental data handling, pipelines, binary data and backpressure. |
| `node:events`, `node:util`, `node:timers/promises` | Event APIs, promisification/inspection, cancellable timer promises. |
| `node:os`, `node:process`, `node:perf_hooks` | Runtime/host inspection, signals/env/exit, timing/event-loop observations. |
| `node:child_process` | Spawn external tools safely with argument arrays and resource limits. Shell interpolation is a command-injection risk. |
| `node:worker_threads` | CPU-heavy transformations in a worker pool; not a substitute for efficient algorithms or a durable job queue. |
| `node:test`, `node:assert/strict` | Built-in test runner and assertions; baseline course tool, no framework dependency. |

Node also supplies web-platform APIs such as `fetch`, `Request`, `Response`, `Headers`, `FormData`, `URL`, `URLSearchParams`, `AbortController`, `Blob`, and Web Streams. These are web API shapes implemented for Node—not the browser DOM. Node streams and Web Streams interoperate via `Readable.toWeb()` / `Readable.fromWeb()` and related helpers; learn the API type at each boundary.

## Code — observe CPU blocking and worker isolation

**BAD: synchronous CPU work blocks this process's event loop.** While this loop runs, other request callbacks in this isolate cannot run.

```js
function terribleHashLoop(rounds) {
  let value = 0;
  for (let i = 0; i < rounds; i++) value = (value * 33 + i) | 0;
  return value;
}
```

Before reaching for a worker: choose a better algorithm, stream/chunk work if possible, or put durable/report workloads on a job worker. If it is genuinely CPU-bound, use a reusable worker pool (not one new thread per request) and set queue/concurrency limits.

```ts
// worker-example.ts — illustrating the boundary, not a complete pool
import { Worker } from "node:worker_threads";

const worker = new Worker(new URL("./cpu-task.js", import.meta.url), {
  workerData: { input: "bounded input" },
  resourceLimits: { maxOldGenerationSizeMb: 128 },
});

worker.once("message", (result: unknown) => {
  // Validate worker output too; same process does not make it trusted by type.
});
worker.once("error", (error) => {
  // Map to an operational failure, log safely, and release pool capacity.
});
```

## Streams, buffers, and backpressure preview

A Buffer is a byte sequence. Encoding transforms bytes to/from text (`utf8`, `base64`, `hex`). Never assume arbitrary uploaded/network bytes are UTF-8. Streams allow incremental processing. Backpressure is the flow-control response when a downstream consumer is slower: honor `write()`'s `false` return / use `pipeline()` so a fast producer does not accumulate unbounded chunks in memory. Module 9 develops this with uploads and exports.

## Processes and resources

A Node process is not just an HTTP listener: it owns heap memory, sockets, file descriptors, timers, database and Redis pools, queue consumers, and event listeners. Multiply configured pool sizes by replicas before deployment; 8 API replicas × 20 DB connections already consume 160 connections before workers and admin connections.

- Use `spawn`/`execFile` with fixed executable and argument arrays for external commands. Avoid `exec("tool " + userInput)` and shell expansion.
- Handle `SIGTERM` by stopping new work and draining before closing resources (Module 11).
- Use `process.env` as string input, parse and validate it at startup.
- Avoid retaining every request object in globals, unbounded maps, listeners, timers, or caches; this is a common memory-leak shape.

## Bad example → production reasoning

**BAD:** “Node is single-threaded, so every async operation is non-blocking.” A synchronous regex, JSON parse of a huge body, accidental `fs.readFileSync`, or expensive serialization can monopolize the event loop. Conversely, scheduling more `Promise`s does not create more CPU capacity.

**Production approach:** measure request latency, event-loop delay/utilization, CPU and heap; cap payloads and fan-out; stream large inputs/outputs; use the libuv pool intentionally; use worker threads or a job process for CPU-heavy work; use timeouts, cancellation, and bounded concurrency around external calls.

## Where it runs / when to use / when not to

- Use Node for network services with substantial I/O concurrency and shared JavaScript skills.
- Keep HTTP handler work short and non-blocking. Move durable long tasks to a queue; move CPU-intensive JS to workers or another compute service.
- Do not use Worker Threads for ordinary PostgreSQL calls or every request. The client library and OS already handle I/O; worker coordination adds cost.
- Do not use `cluster` as a default container-scale plan. One process per container plus replicas is usually easier to isolate and orchestrate. Know the module's purpose for legacy/self-managed hosts.

## Common mistakes

1. Saying async means parallel CPU execution.
2. Treating `setTimeout(..., 0)` as an exact timer.
3. Recursively using `process.nextTick` and starving I/O.
4. Calling `Promise.all` over an unbounded user-controlled list.
5. Forgetting that `Promise.race` leaves losing requests running.
6. Parsing giant JSON or using sync filesystem/crypto in a request path.
7. Treating Node Web APIs as browser DOM APIs.
8. Thinking TypeScript types validate `unknown` messages from workers or external services.
9. Forgetting worker pools, DB pools, sockets, and child processes need shutdown and limits.

## Security notes

- Every process boundary and network result is untrusted data until validated.
- Avoid shell commands built with input; validate paths and prevent traversal; set upload/request size limits.
- Bound CPU, memory, concurrency, and queue depth to resist denial of service.
- Use constant-time comparison for secret MAC/token comparisons where applicable; do not invent a password-hash algorithm.
- Do not log secrets, authorization headers, cookies, or full request bodies.

## Performance and debugging

Use `node --inspect` / Chrome DevTools CPU and heap profiles, `node:perf_hooks` for timing, `process.memoryUsage()`, `monitorEventLoopDelay()`, and `worker.performance.eventLoopUtilization()` when appropriate. Correlate spikes with route, query count, payload size, downstream latency, GC, and pool wait. A CPU profile distinguishes compute from network waiting; a heap snapshot identifies retained objects. Avoid diagnosing from one local benchmark.

## Exercises

- **Beginner:** run the ordering exercise in CommonJS and ESM; explain every logged line.
- **Intermediate:** compare two sequential `fetch` calls with `Promise.all`; then cap fan-out at 5 and propagate abort on request cancellation.
- **Production:** design a CSV export for 2 GB of rows without building a giant array. Name stream types, backpressure point, cancellation and error path.
- **Debugging:** reproduce a delayed `/health` response while a CPU loop runs. Capture evidence, then compare optimizing the algorithm, worker offload, and queueing.
- **Architecture:** for image processing, password verification, SQL I/O, email delivery, and a 300 MB export, choose event loop, libuv, worker thread, child process, or external worker—and defend each choice.

## Build this yourself before looking at a solution

Create a small script that schedules sync, nextTick, Promise, timer, immediate, and file I/O callbacks. Add an abortable `fetch`. Then introduce a deliberately blocking loop and prove that timers/I/O are delayed. **Expected architecture:** one main JS lane; explicit I/O operations; bounded expensive work; measurement before worker offload. **Review:** identify all assumptions that depend on top-level context and all operations that lack cancellation/limits.

## Official documentation (Node 24 baseline)

- [Node releases and status](https://nodejs.org/en/about/previous-releases)
- [Node.js 24 API documentation](https://nodejs.org/docs/latest-v24.x/api/)
- [Event loop guide (libuv)](https://docs.libuv.org/en/v1.x/design.html)
- [Timers](https://nodejs.org/docs/latest-v24.x/api/timers.html) · [process](https://nodejs.org/docs/latest-v24.x/api/process.html) · [worker threads](https://nodejs.org/docs/latest-v24.x/api/worker_threads.html)
- [streams](https://nodejs.org/docs/latest-v24.x/api/stream.html) · [Web Streams](https://nodejs.org/docs/latest-v24.x/api/webstreams.html) · [Buffer](https://nodejs.org/docs/latest-v24.x/api/buffer.html)
- [Node test runner](https://nodejs.org/docs/latest-v24.x/api/test.html) · [child processes](https://nodejs.org/docs/latest-v24.x/api/child_process.html)

**Stability note:** rely on each API's Stability label in the chosen Node major. Do not confuse “available in Node Current docs” with “supported on the course's Active LTS baseline.”

## What I should know before continuing

You can explain the event-loop thread versus libuv/OS/worker work; identify why sync CPU work blocks other callbacks; distinguish Promise concurrency from CPU parallelism; describe backpressure at a high level; choose the correct place for CPU work; and find the Node 24 API stability label. Next: [Module 2 — HTTP and a raw Node server](02-http-node-http.md).
