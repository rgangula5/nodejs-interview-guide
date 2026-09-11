# Node.js Backend Interview Guide

> Pure backend Node.js & middleware — **no database content**. Concept → answer → code → how to say it in the interview. Optimized for reading on your phone (GitHub renders this natively).

## Contents
1. [Node.js Fundamentals & Architecture](#1-nodejs-fundamentals--architecture)
2. [The Event Loop (the #1 topic)](#2-the-event-loop-the-1-asked-topic)
3. [Async: Callbacks, Promises, async/await](#3-async-callbacks-promises-asyncawait)
4. [Modules (CommonJS vs ESM)](#4-modules-commonjs-vs-esm)
5. [Streams & Buffers](#5-streams--buffers)
6. [EventEmitter](#6-eventemitter)
7. [Error Handling](#7-error-handling)
8. [HTTP & the http Module](#8-http--the-http-module)
9. [Express & MIDDLEWARE (deep dive)](#9-express--middleware-deep-dive)
10. [Building REST APIs](#10-building-rest-apis)
11. [Authentication & Security](#11-authentication--security)
12. [Performance, Scaling, cluster & worker_threads](#12-performance-scaling-cluster--worker_threads)
13. [Testing](#13-testing)
14. [Node Internals (libuv, memory, GC)](#14-node-internals-libuv-memory-gc)
15. [npm / package.json / tooling](#15-npm--packagejson--tooling)
16. [Rapid-Fire Q&A (50+)](#16-rapid-fire-qa-50)
17. [How to Answer ANY Question (framework)](#17-how-to-answer-any-node-question-framework)

---

## 1 · Node.js Fundamentals & Architecture

**What is Node.js?**
A **runtime** to run JavaScript on the server. Built on Chrome's **V8** engine (JS → machine code) plus **libuv** (C library providing the event loop + async non-blocking I/O). Node is **single-threaded** for your JS, **event-driven**, uses **non-blocking I/O** → great for I/O-heavy, high-concurrency work (APIs, real-time), poor for CPU-heavy work.

**Why good for I/O, bad for CPU?**
I/O is delegated to libuv/OS and runs async, so one thread juggles thousands of connections. A CPU-heavy task runs *on* the single main thread and **blocks the event loop** — everything stalls. Fix: `worker_threads`, `child_process`, a queue, or a separate service.

**Single-threaded but uses threads?**
Your **JS** runs on one thread. libuv keeps a **thread pool** (default 4, `UV_THREADPOOL_SIZE`) for things that can't be async at the OS level — file I/O, `dns.lookup`, CPU-bound crypto (`pbkdf2`, `zlib`). Network I/O uses the OS async facilities (epoll/kqueue), not the pool.

> **Say this:** "JS is single-threaded; I/O is offloaded to libuv so Node scales I/O cheaply. The moment you do CPU work on the main thread you block everything — so I move it to a worker thread or a queue."

---

## 2 · The Event Loop (the #1 asked topic)

Lets single-threaded Node do non-blocking I/O. When an async op finishes, its callback is queued and runs when the stack is clear. libuv runs the loop in **phases**, repeating each tick:

- **Timers** — due `setTimeout`/`setInterval` callbacks.
- **Pending callbacks** — some system callbacks (e.g., TCP errors).
- **Poll** — retrieves new I/O events; runs most I/O callbacks. Blocks here if idle.
- **Check** — `setImmediate` callbacks.
- **Close** — close callbacks (e.g., `socket.on('close')`).

**Microtasks vs macrotasks (key nuance).** Between every phase (and after each callback), Node drains two microtask queues in order:
1. **`process.nextTick()`** — highest priority.
2. **Promise queue** — `.then/.catch/finally`, `await` continuations.

**Macrotasks** = timers, setImmediate, I/O callbacks. Microtasks drain fully before moving on.

**Predict the output:**
```js
console.log('1 sync');
setTimeout(() => console.log('2 timeout'), 0);
setImmediate(() => console.log('3 immediate'));
Promise.resolve().then(() => console.log('4 promise'));
process.nextTick(() => console.log('5 nextTick'));
console.log('6 sync');

// 1 sync
// 6 sync
// 5 nextTick   <- microtask, before promises
// 4 promise    <- microtask
// 2 timeout / 3 immediate  (order between these can vary)
```

**setTimeout(fn,0) vs setImmediate vs nextTick:**
- `process.nextTick` — before the loop continues; can starve the loop if recursive.
- Promise `.then` — microtask, right after nextTick.
- `setImmediate` — check phase (after poll).
- `setTimeout(fn,0)` — timers phase; min ~1ms.

> **Say this:** "Microtasks — nextTick then Promises — drain completely between phases, so a Promise callback always runs before a 0ms setTimeout."

---

## 3 · Async: Callbacks, Promises, async/await

**Error-first callbacks / callback hell:**
```js
fs.readFile('a.txt', (err, data) => {
  if (err) return handle(err); // error-first convention
});
```

**Promise combinators:**
- `Promise.all` — waits for all; **rejects fast** if any rejects.
- `Promise.allSettled` — waits for all; never rejects; `{status, value|reason}`.
- `Promise.race` — first to settle (resolve or reject).
- `Promise.any` — first to **fulfill**; rejects only if all reject.

**Sequential vs parallel (common trap):**
```js
// SLOW - sequential
const a = await getA();
const b = await getB();

// FAST - parallel
const [a, b] = await Promise.all([getA(), getB()]);
```

**Async error handling + Express wrapper:**
```js
try { const d = await risky(); } catch (err) { /* handles throws + rejections */ }
const wrap = fn => (req,res,next) => Promise.resolve(fn(req,res,next)).catch(next);
```

> ⚠️ `forEach` does NOT await. Use `for...of` with await (sequential) or `Promise.all(arr.map(...))` (parallel).

---

## 4 · Modules (CommonJS vs ESM)

- **CommonJS** (`require`/`module.exports`) — synchronous; `require` is **cached** (module = singleton).
- **ESM** (`import`/`export`) — standard, async-friendly, tree-shakeable. Enable via `"type":"module"` or `.mjs`.
- Differences: ESM imports are **static/hoisted** (use dynamic `import()` for conditional); no `__dirname`/`__filename` in ESM (use `import.meta.url`); top-level `this` is `undefined` in ESM.

**Module wrapper:** Node wraps each file in a function providing `exports, require, module, __filename, __dirname` — that's why these per-file "globals" exist.

---

## 5 · Streams & Buffers

**Streams** process data in chunks instead of loading it all in memory — essential for large files/payloads.
- **Readable, Writable, Duplex, Transform** (e.g., gzip).
- `pipe()` connects readable → writable and handles **backpressure** automatically.

```js
fs.createReadStream('big.log')
  .pipe(zlib.createGzip())
  .pipe(fs.createWriteStream('big.log.gz'));
```

**Backpressure:** producer faster than consumer; `pipe()`/`pipeline()` pause the source until the destination drains. Prefer `stream.pipeline()` — forwards errors + cleans up.

**Buffer:** fixed-length **binary** data outside the V8 heap (file bytes, packets, crypto). `Buffer.from('hi')`, `buf.toString('utf8')`.

---

## 6 · EventEmitter

Node's pub/sub building block; streams & HTTP servers are EventEmitters.
```js
const { EventEmitter } = require('events');
const bus = new EventEmitter();
bus.on('order', id => console.log('got', id)); // subscribe
bus.emit('order', 42);                          // publish
```
> Emitters are **synchronous** — listeners run in registration order, same tick. An error emitted with no `'error'` listener crashes the process.

---

## 7 · Error Handling

- **Operational errors** (expected: bad input, network) → handle & respond gracefully.
- **Programmer errors** (bugs) → let it crash, fix the code.

```js
process.on('uncaughtException', err => { log(err); process.exit(1); });
process.on('unhandledRejection', err => { log(err); process.exit(1); });
```

**Central error middleware (four args, err first):**
```js
app.use((err, req, res, next) => {
  res.status(err.status || 500).json({ error: err.message });
});
```

> **Say this:** "I separate operational from programmer errors, funnel everything to one error-handling middleware, and keep last-resort handlers that log and exit so a process manager restarts a clean instance."

---

## 8 · HTTP & the http Module

```js
const http = require('http');
http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({ ok: true }));
}).listen(3000);
```
`req` is a Readable stream, `res` a Writable stream. Express is a thin layer over this adding routing + middleware + helpers.

---

## 9 · Express & MIDDLEWARE (deep dive)

**What is middleware?** A **function in the request→response pipeline** with access to `req`, `res`, and **`next`**. It can run code, modify req/res, end the response, or call `next()` to pass control. Express is essentially **"a stack of middleware executed in order."**

```js
function logger(req, res, next) {
  req.startTime = Date.now();        // modify req
  console.log(req.method, req.url);  // run code
  next();                            // pass control
}
app.use(logger);
```

**Signature & `next()`:**
- `(req, res, next)` — normal middleware.
- `(err, req, res, next)` — **error** middleware (Express detects 4 args).
- `next()` → next middleware. `next('route')` → skip to next matching route. `next(err)` → jump to error handler.

> ⚠️ **Most common bug:** forgetting `next()` (or a response) → the request **hangs** forever.

**Order matters:** middleware runs **top-to-bottom in registration order**.
```js
app.use(express.json());              // 1. parse body
app.use(logger);                      // 2. log
app.use('/api', authMiddleware);      // 3. protect
app.use('/api/users', usersRouter);   // 4. routes
app.use((req,res) => res.status(404).json({error:'Not found'})); // 5. 404
app.use(errorHandler);                // 6. errors (last, 4 args)
```

**Types:** application-level, router-level, error-handling, built-in (`express.json/urlencoded/static`), third-party (`cors`, `helmet`, `morgan`, `compression`, `express-rate-limit`, `cookie-parser`).

**Auth middleware (very common ask):**
```js
function auth(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: 'No token' });
  try { req.user = verifyToken(token); next(); }
  catch { return res.status(401).json({ error: 'Invalid token' }); }
}
```

**Middleware factory (parameterized):**
```js
function requireRole(role) {
  return (req, res, next) => {
    if (req.user?.role !== role) return res.status(403).json({error:'Forbidden'});
    next();
  };
}
app.delete('/users/:id', auth, requireRole('admin'), handler);
```

**Async errors in middleware:**
```js
const asyncH = fn => (req,res,next) => Promise.resolve(fn(req,res,next)).catch(next);
app.get('/x', asyncH(async (req,res) => { res.json(await service()); }));
// Express 5 auto-forwards async rejections; Express 4 needs this wrapper.
```

**Onion / pipeline model** — request flows down, response flows back up; middleware can act before *and* after `next()`:
```js
app.use(async (req, res, next) => {
  const start = Date.now();
  await next();
  console.log('took', Date.now() - start, 'ms'); // on the way back
});
```

**Express vs Koa:** Express uses callback `next()`; Koa uses async/await middleware (`await next()`) with a cleaner onion model and a single `ctx`.

> **One-liner:** "Middleware is a `(req,res,next)` function that runs in registered order; it either responds or calls `next()`. Error middleware takes four args and everything funnels to it. Order is everything — parsers first, auth before protected routes, error handler last."

---

## 10 · Building REST APIs

- **Principles:** resources (nouns) as URLs; verbs for actions (GET/POST/PUT/PATCH/DELETE); stateless; standard status codes.
- **Status codes:** 200, 201, 204, 400, 401, 403, 404, 409, 422, 429, 500.
- **Idempotency:** GET/PUT/DELETE safe to retry; POST is not (use idempotency keys for payments).
- **Versioning:** `/api/v1/...`. **Pagination:** `?limit=&offset=` or cursor.
- **Validation:** validate input at the edge (middleware) — reject bad requests before business logic.

---

## 11 · Authentication & Security

**JWT vs sessions:**
- **Sessions** — server stores state, client holds a cookie. Stateful; easy to revoke; needs shared store to scale.
- **JWT** — signed token (header.payload.signature) with claims; **stateless**, server just verifies signature. Scales; hard to revoke (short expiry + refresh). **Not encrypted** — base64; never put secrets in it.

**Refresh tokens:** short-lived access token (~15m) + long-lived refresh token (httpOnly cookie). Exchange refresh for new access when it expires.

**Security checklist:** `helmet` (headers), CORS (origins), rate limiting, input validation/sanitization, bcrypt/argon2 for passwords, secrets in env/secret manager, HTTPS, secure/httpOnly cookies, CSRF for cookie auth, `npm audit`.

---

## 12 · Performance, Scaling, cluster & worker_threads

- **cluster** — forks multiple **processes** (one per core) sharing a port; OS load-balances. Scales across cores (PM2 does this for you).
- **worker_threads** — real **threads** in one process, shared memory (`SharedArrayBuffer`). Best for **CPU-bound** work without blocking the loop.
- **child_process** — spawn separate programs (`spawn`/`exec`/`fork`).

**cluster vs worker_threads?** Processes (isolated memory) for scaling **I/O across cores**; threads (shared memory) for **CPU** computation.

**Other levers:** caching (Redis), `compression` (gzip), keep-alive; avoid blocking the loop (no sync fs/crypto on hot path); stream large responses; PM2 + Nginx/ALB in front.

---

## 13 · Testing

- **Unit** — logic in isolation (Jest, Mocha/Chai, Vitest).
- **Integration** — routes end-to-end with **supertest**.
- **Mocks/stubs/spies** — `jest.mock()`, Sinon.

```js
const request = require('supertest');
it('GET /health -> 200', async () => {
  const res = await request(app).get('/health');
  expect(res.status).toBe(200);
});
```

---

## 14 · Node Internals (libuv, memory, GC)

- **libuv** — C library: event loop, thread pool, cross-OS async I/O.
- **V8 memory** — young + old space; generational GC (Scavenge young, Mark-Sweep-Compact old).
- **Leaks** — growing globals/maps, unremoved listeners, closures holding refs, unbounded caches. Diagnose with `--inspect` + heap snapshots, `process.memoryUsage()`.
- **Heap** ~1.5–2GB default; raise with `--max-old-space-size`.

---

## 15 · npm / package.json / tooling

- **dependencies** (runtime) vs **devDependencies** (build/test).
- **Semver** `MAJOR.MINOR.PATCH`; `^1.2.3` = minor+patch, `~1.2.3` = patch only.
- **package-lock.json** pins the exact tree; `npm ci` installs strictly from it.
- **scripts** via `npm run`; **npx** runs a bin without global install.
- `process.env` for config; `process.argv` for CLI args.

---

## 16 · Rapid-Fire Q&A (50+)

- **Single or multi-threaded?** JS single-threaded; libuv thread pool for some I/O.
- **Event loop in one sentence?** Queues callbacks and runs them when the call stack is empty, enabling non-blocking concurrency.
- **nextTick vs setImmediate?** nextTick before the loop continues (highest microtask); setImmediate in check phase.
- **Promise vs setTimeout(0)?** Promise (microtask) runs first.
- **Callback hell fix?** Promises / async-await / modular functions.
- **all vs allSettled?** all rejects on first failure; allSettled waits for all, never rejects.
- **What is middleware?** `(req,res,next)` function in the pipeline; modify req/res, end response, or `next()`.
- **Error middleware args?** Four: `(err, req, res, next)`.
- **next() vs next(err)?** Next middleware vs jump to error handler.
- **Forget next()?** Request hangs.
- **Why order matters?** Runs in registration order; parsers/auth first, error handler last.
- **express.json()?** Parses JSON body into `req.body`.
- **app.use vs app.get?** All methods/prefix vs GET on a path.
- **Config-able middleware?** A factory returning a middleware.
- **Streams — why?** Chunked processing; avoid loading all in memory.
- **Backpressure?** Producer faster than consumer; pipe pauses source.
- **Buffer?** Fixed-length binary data outside V8 heap.
- **EventEmitter?** on()/emit(); synchronous pub/sub.
- **CommonJS vs ESM?** require (sync, cached) vs import (static, tree-shakeable).
- **Is require cached?** Yes — singleton.
- **__dirname in ESM?** No; use `import.meta.url`.
- **uncaughtException — keep running?** No; log and exit.
- **Operational vs programmer error?** Handle vs let crash & fix.
- **JWT vs session?** Stateless signed token vs server state.
- **Is JWT encrypted?** No — signed & base64; don't put secrets in it.
- **Revoke a JWT?** Short expiry + refresh, or denylist.
- **cluster vs worker_threads?** Processes for I/O scaling vs threads for CPU.
- **Avoid blocking the loop?** No sync CPU work; offload to workers/queues.
- **UV_THREADPOOL_SIZE?** libuv thread pool size (default 4).
- **helmet/cors/rate-limit?** Security headers / origins / throttling — all middleware.
- **Hash passwords?** bcrypt/argon2 with salt; never plaintext.
- **Idempotent methods?** GET, PUT, DELETE; POST is not.
- **Status for validation failure?** 400 or 422.
- **supertest?** Integration-test HTTP endpoints.
- **package-lock.json?** Reproducible installs (`npm ci`).
- **^ vs ~?** minor+patch vs patch only.
- **nextTick starvation?** Recursive nextTick starves the loop; prefer setImmediate.
- **Express vs http?** Framework layer over core http (routing + middleware).
- **Globals?** process, Buffer, __dirname/__filename (CJS), timers, console.
- **process.env?** Environment variables for config/secrets.
- **Graceful shutdown?** On SIGTERM stop new connections, finish in-flight, close resources, exit.
- **exec vs spawn?** exec buffers output; spawn streams it.

---

## 17 · How to Answer ANY Node Question (framework)

1. **One-sentence definition first.**
2. **Then the why / when.**
3. **Then a tiny concrete example** — name a real one you built (JWT authorizer, request validation with Zod, logging middleware).
4. **Mention a trade-off / gotcha** — shows senior depth (order matters / forgetting `next()` hangs the request / blocking the loop).

> **Senior signal:** tie answers to real systems you built — event-driven services, a JWT authorizer, idempotent handlers, structured error middleware. Concrete beats textbook.

**If short on time, master these five (≈80% of backend Node interviews):** Event loop · Middleware & `next()` · async/await + `Promise.all` · Error handling · JWT · cluster vs worker_threads.
