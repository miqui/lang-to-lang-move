# What a Java Developer Will Complain About When Moving to Go

A Java developer moving to Go will usually complain less about missing features and more about missing abstractions. Go deliberately omits exceptions, inheritance, method overloading, and (until recently) generics, in favor of a small set of primitives — structs, interfaces, error values, goroutines — and expects engineers to build discipline on top of that rather than lean on the language.

## 1. “Where did my exceptions go?”

Java signals failure by throwing:

```java
public User getUser(String id) throws UserNotFoundException {
    ...
}
```

Go signals failure with an ordinary return value:

```go
func GetUser(id string) (*User, error) {
    user, err := repository.Find(id)
    if err != nil {
        return nil, fmt.Errorf("get user %s: %w", id, err)
    }
    if user == nil {
        return nil, errors.New("user not found")
    }
    return user, nil
}
```

Nothing forces a caller to handle it — an ignored error just gets discarded:

```go
user, _ := GetUser("123") // err silently dropped
fmt.Println(user.Email)   // nil pointer dereference if the lookup failed
```

Unlike a Java `throws` clause, which at least makes the compiler nag about a checked exception, Go's `error` is just another return value. Discarding it with `_` compiles cleanly.

The practical response:

- Never discard an error return with `_` except in narrow, deliberate, commented cases.
- Lint for it — `errcheck` (bundled in `golangci-lint`) flags ignored errors.
- Wrap errors with context using `%w`, and use `errors.Is`/`errors.As` for callers that need to branch on error identity.
- Define sentinel errors (`var ErrNotFound = errors.New(...)`) or custom error types for conditions callers are expected to handle.

---

## 2. “Why is there no `class`?”

Java models behavior through inheritance:

```java
class Animal {
    String name;
    String speak() { return name + " makes a sound"; }
}

class Dog extends Animal {
    @Override
    String speak() { return name + " barks"; }
}
```

Go has no classes and no inheritance. It has structs with methods, and struct embedding, which looks similar but behaves differently:

```go
type Animal struct {
    Name string
}

func (a Animal) Speak() string {
    return a.Name + " makes a sound"
}

type Dog struct {
    Animal
}

func (d Dog) Speak() string {
    return d.Name + " barks"
}
```

Embedding promotes `Animal`'s fields and methods onto `Dog`, but there is no virtual dispatch. Nothing that operates on an `Animal` will ever see `Dog`'s override:

```go
func describe(a Animal) string {
    return a.Speak() // always Animal.Speak, even when the caller holds a Dog
}

d := Dog{Animal{Name: "Rex"}}
describe(d.Animal) // "Rex makes a sound" — no polymorphism through embedding
```

The practical response: use embedding for code reuse (composition), not for polymorphism. When you need substitutable behavior, define an interface and dispatch through it:

```go
type Speaker interface {
    Speak() string
}

func describe(s Speaker) string {
    return s.Speak()
}

describe(Dog{Animal{Name: "Rex"}}) // "Rex barks" — interface dispatch, not embedding
```

---

## 3. “Why didn’t my method mutate the struct?”

Java object fields are always accessed through a reference — a method on an object mutates that object.

Go structs are value types by default. A method with a value receiver operates on a copy:

```go
type Counter struct {
    Count int
}

func (c Counter) Increment() {
    c.Count++
}

c := Counter{}
c.Increment()
fmt.Println(c.Count) // 0 — Increment ran on a copy of c
```

The fix is a pointer receiver:

```go
func (c *Counter) Increment() {
    c.Count++
}
```

The practical response:

- Default to pointer receivers for any type with mutable state, or that's expensive to copy.
- Be consistent within a type — mixing value and pointer receivers on the same type is a common source of subtle bugs, and `go vet` flags some but not all of it.
- Passing a struct by value elsewhere (into a function, a channel, a slice append) copies it — know when that's intended.

---

## 4. “Where is the interface implementation declared?”

Java makes the relationship explicit:

```java
public interface Writer {
    int write(byte[] p);
}

public class FileLogger implements Writer {
    public int write(byte[] p) { ... }
}
```

Go interfaces are satisfied implicitly — structurally, at compile time, with no keyword linking a type to an interface:

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}

type FileLogger struct{}

func (FileLogger) Write(p []byte) (int, error) {
    // ...
    return len(p), nil
}

var w Writer = FileLogger{} // satisfies Writer; nothing says so anywhere near FileLogger
```

A Java developer may reasonably ask: what implements this? What's the contract? How would I even find out without an IDE?

The practical response:

- Keep interfaces small — the Go idiom is one to three methods (`io.Reader`, `io.Writer`) — so structural satisfaction is easy to verify by eye.
- Add a compile-time assertion where the relationship matters, so it's visible in the source and breaks the build if it stops holding:

```go
var _ Writer = FileLogger{}
```

- Lean on tooling (`gopls`'s "find implementations") to recover what Java's `implements` keyword gives you for free in a code review.

---

## 5. “Why is this nil check not catching a nil?”

Go's `nil` looks like Java's `null`, until an interface is involved. An interface value is nil only when both its underlying type and value are nil — a nil pointer wrapped in a non-nil interface is itself non-nil:

```go
type MyError struct{}

func (*MyError) Error() string { return "boom" }

func doWork() *MyError {
    return nil // no error occurred
}

func run() error {
    var err *MyError = doWork()
    return err // returns a non-nil error interface, even though err is nil
}

if run() != nil {
    fmt.Println("error!") // prints — despite doWork() reporting no error
}
```

This is one of Go's most infamous gotchas, and it has no Java analog — Java's `null` doesn't get wrapped by anything on the way out of a method.

The practical response:

- Return the interface type directly (`error`) and hand back a literal `nil` for the no-error case, rather than a typed `nil` pointer.
- Never write `var err *MyError; return err` when the caller expects to compare the result against `nil` as an `error`.
- `go vet` and `staticcheck` catch some instances of this, but not all — it's worth an explicit code-review habit.

---

## 6. “Why does this struct have data I never set?”

Java's `null` at least signals "nothing here yet." Go structs start life fully populated with zero values:

```go
type Config struct {
    Timeout time.Duration
    Retries int
    Enabled bool
}

var c Config
fmt.Println(c.Timeout) // 0s — indistinguishable from "explicitly set to zero"
```

There's no way to tell, from the struct alone, whether `Enabled` was deliberately set to `false` or just never touched.

The practical response:

- Use pointer fields (`*int`, `*bool`) or an explicit sentinel when "unset" must be distinguishable from "zero."
- For external payloads, decode into a type with an explicit presence story (generated OpenAPI types, or manual presence checks against `json.RawMessage`) rather than trusting the zero value.
- For configuration, prefer an explicit constructor or options pattern (§7) over `var c Config` plus field assignment, so required fields can't be silently skipped.

---

## 7. “How do I overload this function?”

Go allows exactly one function or method per name in a scope — no overloading, no default argument values:

```go
func Send(message string) { ... }
func Send(message string, priority int) { ... } // compile error: Send redeclared
```

The practical response is the functional options pattern:

```go
type sendConfig struct {
    priority int
}

type SendOption func(*sendConfig)

func WithPriority(p int) SendOption {
    return func(c *sendConfig) { c.priority = p }
}

func Send(message string, opts ...SendOption) {
    cfg := &sendConfig{}
    for _, opt := range opts {
        opt(cfg)
    }
    // ...
}

Send("hello")
Send("hello", WithPriority(1))
```

It's more ceremony than a Java overload, but it keeps the call site self-documenting (`WithPriority(1)` versus a bare positional `1`).

---

## 8. “Why did modifying this slice change one I never touched?”

Java collections are reference types — slicing isn't a built-in concept the way it is in Go. A Go slice is a view over a backing array, and two slices can share that array without either side being obviously aware of it:

```go
original := []int{1, 2, 3, 4, 5}
sub := original[1:3]  // [2, 3] — shares original's backing array
sub = append(sub, 99) // capacity allows it to write into original's array
fmt.Println(original) // [1 2 3 99 5] — a mutation through a slice nobody named "original"
```

The practical response:

- Use `copy()` (or the `slices` package's `slices.Clone`, Go 1.21+) to detach a slice from its source when the caller needs independence, not just a view.
- Be deliberate about capacity when a function receives a slice it might append to — reslicing (`s[:n]`) does not imply ownership.
- When a function's contract is "returns an independent slice," say so in the doc comment — Go's type system won't say it for you.

---

## 9. “Why does map iteration print in a different order every run?”

Java's `HashMap` has technically-undefined order too, but it's usually stable within a JVM run, and `LinkedHashMap`/`TreeMap` are available when order matters. Go actively randomizes map iteration order on every run, specifically to stop code from depending on it:

```go
m := map[string]int{"a": 1, "b": 2, "c": 3}
for k, v := range m {
    fmt.Println(k, v) // order differs across runs, on purpose
}
```

The practical response: never rely on map iteration order. When output needs to be deterministic — logs, test assertions, API responses — collect and sort the keys explicitly:

```go
keys := make([]string, 0, len(m))
for k := range m {
    keys = append(keys, k)
}
sort.Strings(keys)
```

---

## 10. “Where are try/catch/finally?”

Go has `panic`, `recover`, and `defer`, but they aren't a drop-in replacement for Java's exception model — `panic`/`recover` is reserved for programmer errors and truly unrecoverable conditions, not routine failure handling:

```go
func process() (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered: %v", r)
        }
    }()
    // ...
    return nil
}
```

`defer` covers the cleanup role Java gives `finally` (or try-with-resources):

```go
f, err := os.Open(path)
if err != nil {
    return err
}
defer f.Close()
```

The practical response: use `error` returns (§1) for anything an ordinary caller is expected to handle. Reserve `panic` for conditions that indicate a bug (an index genuinely out of range, an invariant violated) rather than for expected failure paths like "file not found" or "invalid input."

---

## 11. “Why is `private` per-package instead of per-class?”

Java has `private`, `protected`, package-private, and `public`, scoped to the class and hierarchy. Go has exactly one visibility signal — capitalization — scoped to the package, not the type:

```go
package user

type User struct {
    ID    string // exported: capitalized
    email string // unexported: any code in package user can read/write this,
                  // regardless of which file or type declared it
}
```

There is no `protected`, and no per-type privacy — two types in the same package see each other's unexported fields freely.

The practical response: use package boundaries, not type boundaries, to enforce encapsulation — split code into separate packages along the line you actually want to protect. Use an `internal/` directory to prevent packages outside your module from importing something at all, which has no real Java equivalent (it's enforced by the toolchain, not just convention).

---

## 12. “Why did my program silently accumulate stuck goroutines?”

Goroutines are cheap, much like Java's virtual threads (JDK 21+) — but Go has no structured-concurrency stdlib equivalent to clean them up automatically (Java's own structured concurrency is itself still in preview as of JDK 26 — JEP 525, sixth preview — so neither ecosystem has fully settled this yet).

```go
func fetchAll(urls []string) []string {
    results := make(chan string)
    for _, url := range urls {
        go func(u string) {
            results <- fetch(u) // blocks forever if nobody ever reads
        }(url)
    }
    out := make([]string, 0, len(urls))
    for range urls {
        out = append(out, <-results)
    }
    return out
}
```

If the caller returns early — say, on the first error — before draining `results`, every remaining goroutine blocks on its send forever. No crash, no error, just a goroutine that never gets garbage collected because Go's GC doesn't reclaim blocked goroutines.

The practical response:

- Thread `context.Context` through anything that can be cancelled, and make goroutines select on `ctx.Done()` rather than blocking unconditionally on a channel.
- Use `golang.org/x/sync/errgroup` for fan-out/fan-in work — it propagates cancellation and the first error automatically, closer to what Java's `ExecutorService`/structured concurrency gives you.
- Close channels from the sender side only, never the receiver.
- Run `go test -race` routinely — goroutine and channel bugs are exactly the failure class the race detector is built to catch.

---

## 13. “Generics exist, but why do they feel bolted on?”

Go added generics in 1.18 — much later than Java, and more restricted. There's no operator overloading beyond what a type constraint explicitly permits, no variance, and inference is more limited than Java's:

```go
type Number interface {
    ~int | ~int64 | ~float64
}

func Sum[T Number](values []T) T {
    var total T
    for _, v := range values {
        total += v
    }
    return total
}
```

The practical response: use generics for container-like utility code (collections, algorithms) where they clearly pay off. Don't expect them everywhere — large parts of the standard library and ecosystem predate 1.18 and still use `any` plus type assertions or type switches, which is the closest Go analog to Java's pre-generics `Object`-based collections.

---

## 14. “Where's the dependency injection framework?”

Java engineers used to Spring (or CDI) expect a container to wire the application together via annotations and reflection. Go culture actively avoids that kind of magic:

```go
func main() {
    db := postgres.Connect(cfg.DatabaseURL)
    userRepo := repository.NewUserRepository(db)
    userService := service.NewUserService(userRepo)
    handler := http.NewUserHandler(userService)
    // wiring is explicit, visible, and grep-able — no container, no reflection
}
```

The practical response: treat explicit constructor wiring in `main()` as the idiomatic default, not a workaround. If wiring grows unwieldy, `google/wire` generates equivalent code at compile time from a small set of provider functions — it removes the boilerplate without introducing a runtime reflection-based container.

---

## 15. “Why does testing feel so manual?”

Java engineers expect JUnit-style assertions and annotations out of the box. Go's standard `testing` package gives you `t.Run` and manual comparisons — nothing more:

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive", 2, 3, 5},
        {"negative", -1, -1, -2},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            if got := Add(tt.a, tt.b); got != tt.expected {
                t.Errorf("Add(%d, %d) = %d, want %d", tt.a, tt.b, got, tt.expected)
            }
        })
    }
}
```

The table-driven form above is idiomatic Go, not a workaround for a missing framework — most Go style guides treat it as the default shape for a test function.

The practical response:

- Add `testify`'s `assert`/`require` if the team wants assertion helpers, but decide deliberately — some teams skip it intentionally so failures read as plain Go control flow.
- Generate mocks from interfaces with `mockgen` (`gomock`) rather than hand-rolling them; because interfaces are structural (§4), a hand-written fake struct is often simpler than a mock in the first place.

---

## What Go Gets Right

The friction above isn't the whole story — several things a Java engineer will genuinely appreciate:

- A single statically-linked binary as the build output — no JVM version, classpath, or fat-jar to reason about at deploy time.
- `gofmt`: one canonical formatting, enforced by the toolchain — not a matter of team style-guide debate.
- Fast compilation that stays fast as the codebase grows, with no incremental-build tuning required.
- `go vet` and the race detector (`-race`) ship in the standard toolchain, not as bolted-on third-party plugins.
- Cross-compilation is a flag (`GOOS=linux GOARCH=arm64 go build`), not a separate build matrix or JVM-per-platform story.
- Goroutines make concurrent code approachable without first learning a thread-pool/executor API surface — the gotchas in §12 are real, but the entry cost is much lower than Java's pre-virtual-threads concurrency APIs.

---

## Recommended Go Baseline for Java Engineers

For production backend, API-platform, and cloud services:

```text
Go:           Go 1.23+ (or current stable)
Formatting:   gofmt / goimports
Linting:      golangci-lint (bundles errcheck, staticcheck, govet, gosimple)
Vet:          go vet
Tests:        go test -race -cover
Modules:      go.mod + go.sum, committed
Wiring:       explicit constructors in main(), or google/wire if generated
Mocking:      mockgen (gomock) from interfaces
CI:           format check + vet + lint + race-enabled tests
Containers:   multi-stage build to a scratch/distroless static binary
```

Example `go.mod`:

```text
module github.com/org/api-service

go 1.23

require (
	github.com/google/uuid v1.6.0
	golang.org/x/sync v0.8.0
)
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

      - uses: actions/setup-go@v5
        with:
          go-version: "1.23"

      - run: gofmt -l . | tee /dev/stderr | (! grep .)
      - run: go vet ./...

      - uses: golangci/golangci-lint-action@v6

      - run: go test -race -cover ./...
```

## Bottom Line

The Java developer's complaint is usually valid:

> Go does not give you the OOP toolbox or exception-based error handling Java trained you to reach for.

Errors are values you can silently drop, interfaces are implicit and structural, inheritance doesn't exist, and there's no reflection-based framework to wire an application together. But that's Go's design bet: fewer abstractions, explicit control flow at every call site, and a toolchain that stays out of the way.

For serious backend and platform engineering, treat Go's safety as coming from disciplined habits rather than OOP guardrails:

1. Never discard an error return value silently.
2. Prefer small, explicit interfaces over embedding when you need polymorphism.
3. Default to pointer receivers for mutable state, and be consistent within a type.
4. Thread `context.Context` through cancellable work, and use `errgroup` for structured fan-out.
5. Lint aggressively — `golangci-lint` catches a good portion of what the Java compiler gives you for free.

With those habits, Go stays simple and fast while avoiding the sharp edges that trip up engineers arriving with Java's OOP and exception-handling instincts.
