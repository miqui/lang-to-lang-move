# What a Java Developer Will Complain About When Moving to Node.js

A Java developer moving to Node.js will usually complain less about the language itself and more about how much of what the JVM and its ecosystem enforce by default — thread safety, checked types, a single blessed build tool, a sane module system — is either absent or fragmented into competing choices in JavaScript. TypeScript recovers a meaningful chunk of the type-safety argument, but only at compile time; it doesn't change what happens once the code is running.

## 1. “Why did one slow request freeze the whole server?”

Java's servlet/thread-per-request model (or virtual threads, JDK 21+) means one slow request occupies its own thread — other requests keep being served by other threads:

```java
@GetMapping("/report")
String report() {
    return buildLargeReportSync(); // blocks only this request's own thread
}

@GetMapping("/health")
String health() {
    return "ok"; // served by a different thread — unaffected by /report
}
```

Node.js runs application code on a single thread, driven by an event loop. A synchronous, CPU-heavy operation blocks that thread entirely — nothing else runs until it finishes:

```javascript
app.get("/report", (req, res) => {
  const data = buildLargeReportSync(); // CPU-bound, synchronous
  res.json(data);
});

app.get("/health", (req, res) => {
  res.send("ok"); // queues behind /report until it finishes, even for unrelated requests
});
```

A single expensive synchronous handler starves every other in-flight request, including unrelated health checks.

The practical response:

- Keep request handlers I/O-bound (network, disk) and non-blocking — the event loop is built for this and does it well.
- Move genuinely CPU-bound work off the main thread with `worker_threads`, or out of process entirely (a queue and a separate worker process).
- Avoid synchronous stdlib calls in request paths (`fs.readFileSync`, `crypto.pbkdf2Sync`) — use their async counterparts.
- Monitor event-loop lag directly (e.g., `perf_hooks.monitorEventLoopDelay`) — it's the equivalent of watching thread-pool saturation in Java.

---

## 2. “Where are my threads?”

Java gives you `Thread`, thread pools, and (since JDK 21) cheap virtual threads for expressing concurrency directly:

```java
ExecutorService pool = Executors.newFixedThreadPool(4);
Future<Result> future = pool.submit(() -> cpuHeavyTask(input));
Result result = future.get(); // same JVM heap throughout — no copying needed to get it back
```

Node.js has no equivalent default — JavaScript on the main thread is single-threaded by design. Parallelism exists, but you have to reach for it explicitly:

```javascript
const { Worker } = require("node:worker_threads");

const worker = new Worker("./cpu-heavy-task.js", { workerData: input });
worker.on("message", (result) => {
  // handle result
});
```

`worker_threads` gives real parallel execution, but each worker is a separate JS heap — data crosses the boundary by copying (structured clone) or via `SharedArrayBuffer`, not by sharing objects the way Java threads share heap memory.

The practical response:

- Default to Node's async I/O model for concurrency — most backend workloads are I/O-bound and don't need threads at all.
- Reach for `worker_threads` only for genuinely CPU-bound work (image processing, heavy computation), and treat the data crossing the boundary as a serialization boundary, not a shared-memory one.
- For horizontal scaling across cores on one machine, the `cluster` module or a process manager (PM2, or just multiple containers behind a load balancer) is more common than in-process threading.

---

## 3. “Why did this promise just disappear?”

Java's checked exceptions force a caller to acknowledge failure at compile time:

```java
void onUserCreated(User user) throws MessagingException {
    mailer.send(user.getEmail(), "welcome"); // caller must catch this or declare it too
}
```

A rejected Promise with no `.catch()` — or an `async` function called without `await` — fails just as silently as a discarded Go error:

```javascript
async function sendWelcomeEmail(user) {
  await mailer.send(user.email, "welcome");
}

function onUserCreated(user) {
  sendWelcomeEmail(user); // not awaited — a rejection here is unhandled
  return { status: "created" };
}
```

If `mailer.send` rejects, nothing in this code observes it. Since Node 15, an unhandled rejection crashes the process by default (`--unhandled-rejections=throw`) rather than just logging a warning — either way, the caller who wrote `onUserCreated` has no signal at the call site that something can go wrong.

The practical response:

- Treat every `async` function call as something that must be `await`ed, `.catch()`ed, or explicitly and visibly fired-and-forgotten (`void sendWelcomeEmail(user)` in TypeScript documents the intent).
- Lint for it — `@typescript-eslint/no-floating-promises` catches exactly this class of bug at build time.
- Register a top-level `process.on("unhandledRejection", ...)` handler as a safety net, not as the primary handling strategy.
- For work that should be abandoned rather than awaited — a superseded query, a request past its deadline — use `AbortController`/`AbortSignal` (a global since Node 15) and pass the signal into `fetch`, timers, and stream APIs. It's the closest thing to `Future.cancel(true)`, and like Java's interruption it's cooperative: the callee has to actually check the signal.

---

## 4. “Why is `this` undefined inside my callback?”

In Java, `this` always refers to the instance a method was called on — there's no way to lose it:

```java
class UserService {
    private final Map<String, User> cache = new HashMap<>();

    void handleUser(User user) {
        cache.put(user.getId(), user); // `this` is always the UserService instance
    }
}

UserService service = new UserService();
eventEmitter.on("user", service::handleUser); // method reference stays bound to `service`
```

In JavaScript, `this` is determined by how a function is called, not where it's defined. Passing a method as a callback detaches it from its object:

```javascript
class UserService {
  constructor() {
    this.cache = new Map();
  }

  handleUser(user) {
    this.cache.set(user.id, user); // works when called as service.handleUser(...)
  }
}

const service = new UserService();
eventEmitter.on("user", service.handleUser); // this is now undefined inside handleUser
```

The practical response:

- Bind explicitly in the constructor (`this.handleUser = this.handleUser.bind(this)`), or
- Prefer arrow-function class fields, which capture `this` lexically from the surrounding class body:

```javascript
class UserService {
  cache = new Map();

  handleUser = (user) => {
    this.cache.set(user.id, user); // this is always the instance, regardless of call site
  };
}
```

- Lint for unbound method references passed as callbacks (`@typescript-eslint/unbound-method`).

---

## 5. “Why did TypeScript let this through?”

Java's static typing is backed by a runtime check the moment a value crosses a type boundary:

```java
double addTax(double amount) {
    return amount * 1.07;
}

Object raw = parseJson(request.getBody());
addTax((Double) raw); // ClassCastException immediately, right here, if raw isn't actually a Double
```

TypeScript adds a compile-time type checker Java developers will recognize the shape of — but it's erased entirely by the time code runs:

```typescript
function addTax(amount: number): number {
  return amount * 1.07;
}

const raw: any = JSON.parse(request.body);
addTax(raw.amount); // compiles fine; amount could be a string, undefined, anything
```

`any` disables checking the same way it does in Python's type hints — and unlike Java, there's no runtime cast to catch the mismatch when it happens; the function just returns `NaN` or a concatenated string, silently.

The practical response:

- Enable `strict` mode in `tsconfig.json` — it's off by default and catches most of what makes `any` dangerous.
- Treat `any` as a boundary marker, not a convenience type — isolate it to the edges (parsing untrusted input) and convert to a real type immediately.
- Validate external data at runtime with a schema library (`zod`, `valibot`) rather than trusting a type assertion — `JSON.parse(...) as User` compiles but proves nothing at runtime.
- Avoid the non-null assertion operator (`value!`) in application code; it's a compile-time promise with no runtime backing, functionally equivalent to an unchecked cast.

---

## 6. “Why did `==` say `true` here?”

Java's `==` on primitives is unambiguous, and object comparison always requires `.equals()`:

```java
0 == 0;                    // true — plain numeric comparison, no coercion involved
"".equals("0");             // false — String.equals() never coerces between types
Double.NaN == Double.NaN;  // false — same IEEE 754 rule, but reached only via a type-safe comparison
```

JavaScript's `==` performs type coercion, with rules that are widely considered a design mistake even by the language's own community:

```javascript
0 == "";        // true
0 == "0";       // true
"" == "0";      // false
null == undefined; // true
NaN == NaN;     // false
[] == false;    // true
```

The practical response:

- Always use `===`/`!==`. There is no legitimate reason to reach for `==` in modern application code, and linters can enforce this outright (`eslint eqeqeq` rule).
- For `NaN`, use `Number.isNaN(x)` — `x === NaN` is always `false`, even when `x` is `NaN`.
- For deep equality (objects, arrays), neither `==` nor `===` compares structurally — use a library (`fast-deep-equal`, or `assert.deepStrictEqual` in tests) or compare specific fields explicitly.

---

## 7. “Why is this ‘private’ field visible from outside the class?”

Java's `private` is enforced by the compiler and JVM:

```java
class ApiClient {
    private String apiKey;

    ApiClient(String apiKey) { this.apiKey = apiKey; }
}

ApiClient client = new ApiClient("secret");
client.apiKey; // compile error: apiKey has private access in ApiClient
```

JavaScript had no true private state for most of its history — only conventions:

```javascript
class ApiClient {
  constructor(apiKey) {
    this._apiKey = apiKey; // convention: "don't touch this"
  }
}

const client = new ApiClient("secret");
console.log(client._apiKey); // nothing stops this
```

As of ES2022 (Node 12+ for the syntax, broadly supported since), true private fields exist with the `#` prefix:

```javascript
class ApiClient {
  #apiKey;

  constructor(apiKey) {
    this.#apiKey = apiKey;
  }
}

const client = new ApiClient("secret");
client.#apiKey; // SyntaxError: Private field '#apiKey' must be declared in an enclosing class
```

The practical response:

- Use `#field` for genuinely private state in new code — it's enforced by the runtime, not just a lint rule, and is the closest JavaScript gets to Java's `private`.
- A leading underscore (`_field`) still appears throughout the ecosystem in older code and some libraries — treat it as documentation only, never as enforcement, when you don't control the source.
- TypeScript's `private`/`protected` keywords are compile-time only and vanish at runtime — they don't add any protection `#field` doesn't already give you at runtime, but they do work on class methods, where `#` support is more recent.

---

## 8. “Where is my checked exception, and why did this `catch` block get a string?”

Java's `throw` only accepts a `Throwable`:

```java
throw "something went wrong"; // compile error: incompatible types: String cannot be converted to Throwable
throw new RuntimeException("something went wrong"); // the only legal choice, and the only sane one
```

JavaScript's `throw` accepts anything:

```javascript
throw "something went wrong"; // valid
throw { code: 500 };          // valid
throw new Error("something went wrong"); // also valid, and the only sane choice
```

A `catch` block has no guarantee about what it received:

```javascript
try {
  doSomething();
} catch (err) {
  console.log(err.message); // undefined if err isn't an Error instance
}
```

Async code adds a second failure mode: a rejected Promise and a thrown exception inside an `async` function behave the same way at the `await` call site, but errors from callback-style APIs (older Node APIs, some libraries) don't flow through `try`/`catch` at all unless explicitly wrapped.

The practical response:

- Always `throw new Error(...)` (or a subclass) — never throw a bare string or plain object.
- Define domain-specific `Error` subclasses for conditions callers need to branch on, the same pattern used in the other guides in this repository:

```javascript
class UserNotFoundError extends Error {
  constructor(id) {
    super(`User not found: ${id}`);
    this.name = "UserNotFoundError";
  }
}
```

- Narrow `catch` handling with `instanceof` rather than assuming shape:

```javascript
} catch (err) {
  if (err instanceof UserNotFoundError) {
    return { status: 404 };
  }
  throw err; // don't swallow what you don't recognize
}
```

- Chain errors with the ES2022 `cause` option (Node 16.9+) instead of concatenating context into a new message — it's the direct analog of `new RuntimeException("...", e)`, and it preserves the original error and its stack rather than flattening it to a string:

```javascript
} catch (err) {
  throw new Error(`Failed to load user ${id}`, { cause: err });
}
```

- `cause` only preserves something if what you caught was an `Error` to begin with — a thrown string has no stack to keep, which is the same reason the first rule above matters.

---

## 9. “Which package manager, and which Node, is this project actually using?”

Java engineers usually have one answer (Maven or Gradle) and one JDK, pinned by the build tool itself:

```xml
<properties>
    <maven.compiler.release>21</maven.compiler.release>
</properties>
<!-- one file, one build tool, one pinned JDK version — no axis of choice left open -->
```

Node.js offers several axes of choice at once:

```text
Package managers: npm, yarn, pnpm
Version managers:  nvm, volta, fnm
Runtimes:          Node.js, Deno, Bun
```

A repository without an explicit, enforced choice tends to accumulate a mix of lockfiles and inconsistent local Node versions across contributors.

The practical response:

- Pick one package manager per repository and commit its lockfile (`package-lock.json`, `pnpm-lock.yaml`, or `yarn.lock`) — never let more than one lockfile exist at once.
- Pin the Node version with an `.nvmrc` or the `engines` field in `package.json`, and enforce it in CI.
- A practical baseline for a backend service:

```json
{
  "engines": { "node": ">=24.0.0" },
  "packageManager": "npm@10.9.0"
}
```

- Note that `packageManager` is Corepack's pinning mechanism, and Corepack's own status moved: it is no longer distributed with Node 25 and later, though it remains bundled in Node 24 LTS and earlier. If a workflow depends on it, install it explicitly (`npm install -g corepack`) rather than assuming the runtime ships with it.

---

## 10. “Why did `require` vs `import` break the build?”

Java's module system (or classpath, pre-JPMS) doesn't have a comparable split within one ecosystem generation — there's exactly one way to bring a type into scope:

```java
import com.example.express.Router; // always resolved via the classpath — one mechanism, one syntax
```

Node.js has two module systems in active use: CommonJS (`require`/`module.exports`) and ECMAScript Modules (`import`/`export`).

```javascript
// CommonJS
const express = require("express");
module.exports = { handler };

// ESM
import express from "express";
export { handler };
```

A package published as ESM-only cannot be `require()`'d from a CommonJS file — this is the "dual package hazard," and it's a common source of build failures when adding a dependency that made the jump to ESM-only ahead of the rest of a codebase. (As of Node 22.12+, `require()` can load a synchronous ESM-only module — one with no top-level `await` anywhere in its graph — without a flag; the hazard is narrower than it used to be, but still real for any ESM-only package that does use top-level `await`.)

The practical response:

- For new projects, default to ESM (`"type": "module"` in `package.json`) — it's where the ecosystem and the language spec itself are heading.
- If a codebase must stay CommonJS, check a new dependency's module format before adding it; an ESM-only package will need a dynamic `import()` (which returns a Promise) rather than a synchronous `require()`, unless it qualifies for Node 22.12+'s native `require(esm)` support (no top-level `await`).
- Don't mix `require` and `import` syntax within the same file — pick one per file, and one system per package unless there's a specific, documented reason for dual-publishing.

---

## 11. “Why isn't `0.1 + 0.2` equal to `0.3`?”

Java has distinct `int`, `long`, `float`, `double`, and `BigDecimal` types, chosen deliberately based on precision needs:

```java
double sum = 0.1 + 0.2;                                        // 0.30000000000000004 — same IEEE 754 double as JS
BigDecimal precise = new BigDecimal("0.1").add(new BigDecimal("0.2")); // exactly 0.3, by deliberately opting in
```

JavaScript has exactly one `number` type — an IEEE 754 double — for everything:

```javascript
0.1 + 0.2;               // 0.30000000000000004
9007199254740993;        // 9007199254740992 — silently loses precision above 2^53
```

There's no compiler warning when a value exceeds safe integer range or needs decimal precision — it just quietly becomes imprecise.

The practical response:

- Never use floating-point `number` for money — represent currency as integer minor units (cents) or use a decimal library (`decimal.js`, `big.js`).
- Use `BigInt` (a distinct type, not interchangeable with `number`) for integers that may exceed `Number.MAX_SAFE_INTEGER` (2^53 − 1) — database IDs from a 64-bit source are a common case that bites people.
- Compare floating-point values with a tolerance, not `===`, the same way you would in Java when comparing `double`s.

---

## 12. “Why did my date roll over to the wrong day?”

Java's `java.time` package (`LocalDate`, `Instant`, `ZonedDateTime`) is deliberately explicit about calendar vs instant vs timezone:

```java
LocalDate date = LocalDate.of(2024, 1, 15);              // January is 1, not 0 — no off-by-one trap
Instant instant = Instant.parse("2024-01-15T00:00:00Z"); // explicitly UTC, no ambient timezone involved
```

JavaScript's built-in `Date` predates that thinking and has several well-known footguns:

```javascript
new Date(2024, 0, 15); // January 15 — the month argument is zero-indexed
new Date("2024-01-15").getDate(); // can print 14, depending on the machine's local timezone
```

`Date` mixes "a point in time" and "a local calendar date" into one type with an implicit, ambient-timezone-dependent interpretation — precisely the confusion `java.time` was designed to eliminate.

The practical response:

- For anything beyond trivial timestamp arithmetic, use a library (`date-fns`, `luxon`) or the `Temporal` API rather than the built-in `Date`. `Temporal` isn't available at all on Node 22 (this guide's floor) — it's reachable only behind `--harmony-temporal` on Node 24, and ships unflagged starting Node 26. Use the `@js-temporal/polyfill` package until your runtime floor moves that far.
- Store and transmit timestamps as UTC ISO 8601 strings or Unix epoch values; convert to a local timezone only at the presentation layer, mirroring the `Instant` vs `ZonedDateTime` separation in `java.time`.
- Never rely on the server's local timezone for business logic — set `TZ=UTC` in deployment environments to remove the ambient dependency entirely.

---

## 13. “Why did installing one dependency pull in four hundred packages?”

Maven Central has a comparatively high barrier to publishing and a culture of larger, more consolidated libraries:

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-web</artifactId>
    <version>6.1.0</version>
</dependency>
<!-- pulls in a comparatively shallow, well-vetted transitive tree -->
```

npm has near-zero barrier to publishing, and a culture of small, single-purpose packages composed deeply — a single dependency can pull in a very large transitive tree.

```bash
npm install express
# adds express plus dozens of transitive dependencies
```

Every package in that tree is code that runs at install time (via lifecycle scripts) or at runtime, from a publisher you likely haven't vetted — a materially larger supply-chain surface than a typical Maven dependency tree.

The practical response:

- Run `npm audit` (or `pnpm audit`) in CI, and treat high-severity findings as build failures, the same way a Java team would treat an OWASP dependency-check failure.
- Check whether the runtime already covers it before adding a package at all. `fetch` (stable since Node 21) replaces axios and node-fetch, `node --watch` (stable since Node 21) replaces nodemon, `--env-file` (added in Node 20.6, no longer experimental as of 22.21) replaces dotenv, and `node:test` (§14) replaces the test-framework dependency outright. Several of the reflexive "every Node project needs these" packages are now redundant.
- Review new dependencies before adding them — package size, maintenance activity, and transitive dependency count are all visible on the npm registry page before you commit to one.
- Consider disabling install-time lifecycle scripts for third-party packages (`npm install --ignore-scripts`, or pnpm's default-deny) when the added risk isn't worth the convenience.
- Commit the lockfile and treat any lockfile diff in a PR as something to actually read, not skip past.

---

## 14. “Which test framework is ‘the’ framework?”

Java has one dominant default (JUnit) that nearly every project uses:

```java
@Test
void shouldCalculateTotal() {
    assertEquals(30, calculateTotal(10, 20)); // JUnit — the only real choice, ecosystem-wide
}
```

Node.js now ships a runner in the runtime itself — `node:test` plus `node:assert`, stable since Node 20, invoked with `node --test`:

```javascript
import test from "node:test";
import assert from "node:assert/strict";

test("calculates total", () => {
  assert.equal(calculateTotal(10, 20), 30); // no install, no config, no dependency
});
```

That answers "where's my JUnit" more directly than anything Node had before, but it hasn't displaced the ecosystem — most existing projects, and most tutorials and tooling you'll encounter, still assume Jest or Vitest:

```text
Jest       — historically dominant, batteries-included, slower on large suites
Vitest     — fast, Vite-native, Jest-compatible API
Mocha+Chai — older, more assembly required
node:test  — built in, stable since Node 20, zero dependencies, thinner feature surface
```

The practical response:

- For a new project with no existing constraint, `node:test` (built-in, zero dependency) or Vitest (fast, good TypeScript support) are the current pragmatic defaults — pick one and standardize it across the team rather than letting it vary by contributor preference.
- Don't assume the built-in runner is a drop-in for Jest. Mocking, snapshot testing, and the richer matcher library are where it's deliberately thinner — which is exactly where a team used to JUnit plus Mockito and AssertJ will feel the gap.
- Whichever framework is chosen, enforce coverage thresholds in CI the way a Java team would with JaCoCo, rather than leaving coverage advisory-only.

---

## 15. “Why did extending a built-in type do something unexpected?”

Java's `class` keyword means what it says — there's no shared, mutable structure sitting behind every instance of a built-in type:

```java
class ArrayUtils {
    static <T> T last(List<T> list) { return list.get(list.size() - 1); } // a plain static method
}

ArrayUtils.last(List.of(1, 2, 3)); // 3 — java.util.List itself is never touched
```

JavaScript's `class` is syntactic sugar over prototype-based inheritance, and the underlying prototype chain is mutable at runtime — including for built-in types:

```javascript
Array.prototype.last = function () {
  return this[this.length - 1];
};

[1, 2, 3].last(); // 3 — every array in the entire process now has this method
```

Monkey-patching a built-in prototype affects every instance of that type across the whole process, including inside third-party libraries that never opted into it — there's no per-class or per-module scoping the way Java's class model would imply.

The practical response:

- Never modify the prototype of a built-in type (`Array`, `Object`, `String`) in application code — write a standalone utility function instead.
- When subclassing is genuinely needed, extend the built-in directly (`class SortedList extends Array`) rather than patching the shared prototype.
- Be aware that `class` fields and methods live on the prototype by default (shared across instances), while fields assigned in the constructor live on the instance — mixing the two without understanding the difference is a common source of "why do all my instances share this array" bugs.

---

## 16. “Why did one long computation freeze every request, not just the one that triggered it?”

A JVM engineer expects a GC pause to be a local, survivable event: G1 (or ZGC/Shenandoah) mostly collects concurrently with the running program and keeps stop-the-world pauses to single-digit milliseconds — and because the JVM is multi-threaded, other threads keep serving requests while any pause that does happen resolves.

Node.js runs JavaScript on a single thread, and V8's GC runs on that same thread. Its young-generation "Scavenge" collector is fast and mostly unnoticeable, but a major "Mark-Compact" collection is stop-the-world on the only thread available to handle anything else:

```javascript
function summarize(rows) {
  return rows.map(r => transform(r)); // builds enough short-lived garbage that
}                                     // the resulting major GC pause blocks
                                      // every in-flight request, not just this one
```

There's no second request-handling thread to fall back on during that pause — a GC pause here is a pause for the entire process, not a pause for one code path.

The practical response:

- Avoid large synchronous allocations or transformations in the hot path; chunk the work, or move it to a `worker_thread` so a major GC pause there doesn't block the main event loop.
- `--max-old-space-size` caps V8's old-space heap (the rough analog of `-Xmx` for the JVM's old generation), but it only bounds the JS heap — `Buffer`s, `ArrayBuffer`s, and native addon memory sit outside it and won't show up until the process's RSS is already large.
- Watch for closures and `EventEmitter` listeners that outlive their intended scope — a forgotten `removeListener` or an array accumulating inside a captured callback is the most common source of "leaks" in Node, since nothing else will reclaim a reachable closure.
- Use `--trace-gc`, `node --inspect` with Chrome DevTools heap snapshots, or `--heapsnapshot-signal` instead of expecting a direct Node equivalent to JVM tools like `jstat` or GC-log analyzers — the tooling and vocabulary (old space/new space, not young/old generation) are V8-specific.

---

## 17. “Where do I put the request context — there's no thread to hang it on?”

In Java, request-scoped context lives in a `ThreadLocal`. Because a request owns its thread for its whole lifetime, every layer underneath can read that context without being handed it explicitly — SLF4J's MDC is the most familiar instance:

```java
MDC.put("correlationId", request.getHeader("X-Correlation-Id"));
try {
    orderService.process(order); // nothing below takes correlationId as a parameter
} finally {
    MDC.remove("correlationId"); // the logger reads it off the thread automatically
}
```

Node has no thread to attach that to — one thread interleaves every in-flight request, so a module-level variable is shared by all of them, and the naive version leaks one request's identity into another's logs. `AsyncLocalStorage` (stable since Node 16.4) is the real equivalent: it keeps a store alive across the async continuations spawned inside `run()`.

```javascript
import { AsyncLocalStorage } from "node:async_hooks";

const requestContext = new AsyncLocalStorage();

app.use((req, res, next) => {
  requestContext.run({ correlationId: req.get("X-Correlation-Id") }, next);
});

// anywhere further down the await chain, with no parameter threaded through:
const { correlationId } = requestContext.getStore() ?? {};
```

The store follows `await` and native promise chains, so it survives the boundaries a `ThreadLocal` survives in Java — but it's scoped per async context rather than per thread, and it's the only supported way to get this behavior.

The practical response:

- Set the store once at the edge (HTTP middleware, a queue consumer's message handler) and read it wherever you'd have read the MDC. Don't thread a context object through every function signature, and never use a module-level variable — it's shared across concurrent requests.
- Wire it into the logger rather than reading it by hand at each call site; that's what makes it an actual MDC replacement rather than a global with extra steps.
- Expect context loss around older callback-style APIs and custom thenables — Node's own documentation calls this out. `util.promisify()` on a callback API, or `AsyncResource` for a custom thenable, restores propagation.
- It isn't free. Maintaining the store across async boundaries costs more than passing a parameter would, so scope it to context genuinely needed for the whole request (correlation ID, tenant, authenticated user), not as a general-purpose ambient store.

---

## 18. “Wait — do I still need a build step for TypeScript?”

A Java developer treats the compile step as the thing that catches mistakes: `javac` produces the artifact and enforces the types in one pass, and there's no mode in which you get a runnable class file whose types went unchecked.

```java
String total = calculateTotal(10, 20); // compile error: int cannot be converted to String
// no class file is produced — code that fails to typecheck cannot be run
```

Node runs TypeScript files directly now, with no build step and no `ts-node`:

```javascript
// server.ts — run it with: node server.ts
function addTax(amount: number): number {
  return amount * 1.07;
}
```

This is *type stripping*: Node erases the annotations and runs the JavaScript underneath. It arrived behind `--experimental-strip-types` in Node 22.6 and runs unflagged from Node 22.18 and 23.6 onward. The part that matters most coming from Java is what it doesn't do — it never type-checks. Annotations are deleted, not verified, so a file that `tsc` would reject runs perfectly happily.

The practical response:

- Keep `tsc --noEmit` in CI and in your editor. `node server.ts` replaces the build step, not the type checker — nothing else will tell you the types are wrong.
- Type stripping only handles *erasable* syntax. Enums, namespaces containing runtime code, and constructor parameter properties have to generate JavaScript rather than be deleted, so Node rejects them outright with `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX` — see [java-to-typescript](../java-to-typescript/) for why `enum` in particular is a poor fit.
- On the Node 22 line, check the minor version before relying on flagless execution: 22.18+ runs `.ts` directly, earlier 22.x still needs `--experimental-strip-types`.

---

## What Node.js Gets Right

The friction above isn't the whole story — several things are genuinely nicer once a Java engineer settles in:

- The async I/O model, once internalized, handles high-concurrency I/O-bound workloads (the common case for a web backend) with far less ceremony than Java's pre-virtual-threads thread-pool tuning.
- npm's package ecosystem is enormous — most small utility needs already have a well-maintained, single-purpose package, rather than requiring a small utility class written in-house.
- No compile step for plain JavaScript: the edit-run loop is immediate, and startup time is typically far faster than a JVM cold start — a real advantage for CLI tools, serverless functions, and local iteration.
- TypeScript's structural typing (closer to Go's interfaces or Python's `Protocol` than to Java's nominal typing) makes it easy to type existing JavaScript incrementally, without redesigning a class hierarchy first.
- `package.json` plus a lockfile is a lighter-weight project descriptor than a Maven POM or Gradle build script for small-to-medium services — less XML/DSL ceremony to get a working build.

---

## Recommended Node.js Baseline for Java Engineers

For production backend and API services:

```text
Runtime:      Node.js 24+ LTS (24 is Active LTS; 22 dropped to maintenance in Oct 2025)
Language:     TypeScript, strict mode
Packages:     npm (or pnpm) + committed lockfile
Formatting:   Prettier
Linting:      ESLint + @typescript-eslint (no-floating-promises, eqeqeq, unbound-method)
Tests:        node:test or Vitest, with coverage thresholds enforced
Validation:   zod (or equivalent) at external boundaries
Dates:        date-fns / luxon / Temporal (needs @js-temporal/polyfill below Node 26) — never bare Date arithmetic
CI:           lint + typecheck + tests + npm audit
Containers:   pinned Node base image, multi-stage build, non-root user
```

Example `tsconfig.json`:

```json
{
  "compilerOptions": {
    "strict": true,
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "noUncheckedIndexedAccess": true,
    "esModuleInterop": true
  },
  "include": ["src"]
}
```

Example CI checks:

```yaml
name: Verify

on:
  pull_request:
  push:
    branches:
      - main

jobs:
  verify:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "24"
          cache: "npm"

      - run: npm ci
      - run: npx tsc --noEmit
      - run: npx eslint .
      - run: npm test
      - run: npm audit --audit-level=high
```

## Bottom Line

The Java developer's complaint is usually valid:

> JavaScript enforces almost nothing by default — not thread safety, not types, not even what you're allowed to `throw`.

TypeScript closes a real part of that gap, but only up to the compiler boundary; once the code is running, an `any` slipped past review, a floating Promise, or a monkey-patched prototype behaves exactly as if there were no type system at all.

For serious backend and platform engineering, treat Node.js and TypeScript as a stack that needs the same deliberate policy layer as Python or Go:

1. Enable and enforce TypeScript strict mode; treat `any` as a boundary marker, not a convenience.
2. Validate all untrusted data at runtime — the compiler cannot do it for you.
3. Never leave a Promise unawaited or unhandled; lint for it.
4. Keep request handlers non-blocking; move CPU-bound work off the main thread deliberately.
5. Standardize the package manager, Node version, and test framework per repository, and enforce all three in CI.

With those constraints in place, Node.js remains fast to iterate in and lightweight to deploy, while avoiding the sharp edges that trip up engineers arriving with Java's compiler- and JVM-backed guarantees.
