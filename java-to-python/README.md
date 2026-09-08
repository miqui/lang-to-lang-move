# What a Java Developer Will Complain About When Moving to Python

A Java developer moving to Python will usually complain less about syntax and more about the loss of default guardrails. Python can support highly disciplined, production-grade engineering, but many of the controls Java developers expect are opt-in rather than built into the language and standard workflow.

Most of this applies regardless of which JDK or Python release you're coming from or moving to — it's language design, not a version detail. Where a specific version changes the picture (e.g., virtual threads only exist from JDK 21 on, Python's free-threaded build only became officially supported in 3.14), the relevant section calls it out explicitly rather than assuming a fixed pair of versions.

## 1. “Why didn’t the compiler catch this?”

This is the largest adjustment.

Java has a compile-time-enforced static type system. If a method expects a `BigDecimal`, passing a `String` is a compiler error:

```java
BigDecimal addTax(BigDecimal amount) {
    return amount.multiply(new BigDecimal("1.07"));
}

addTax("100"); // compile error: incompatible types: String cannot be converted to BigDecimal
```

In Python, type annotations are not enforced by the runtime.

```python
def add_tax(amount: float) -> float:
    return amount * 1.07

add_tax("100")  # Valid at call time; fails only when executed
```

A Java developer sees `float` and expects an early error. Python sees it as optional metadata unless an external type checker is running.

The practical response:

- Use type annotations consistently.
- Enforce `pyright` or `mypy` in CI.
- Configure strict mode.
- Avoid `Any` in application and domain code.
- Validate external data at runtime.

```toml
# pyproject.toml
[tool.pyright]
typeCheckingMode = "strict"
pythonVersion = "3.13"
include = ["src"]
```

```bash
pyright
pytest
ruff check .
ruff format --check .
```

The key mindset is:

> Java is type-safe by language default.  
> Python becomes type-safe through engineering discipline and tooling.

---

## 2. “Why are type hints optional?”

Java gives you no equivalent laxity — a method signature is always fully typed, with no way to omit an annotation:

```java
User getUser(String userId) { // parameter and return types are never optional
    ...
}
```

Python supports type hints:

```python
def get_user(user_id: str) -> "User | None":
    ...
```

But it also permits this:

```python
def get_user(user_id):
    ...
```

And both run.

A Java developer may view optional annotations as an invitation to inconsistent codebases:

- Some modules fully typed.
- Some libraries partially typed.
- Some dependencies exposing `Any`.
- Some projects using no static checking at all.

A well-run Python service should establish a clear policy:

- All public functions must have parameter and return annotations.
- New domain code must pass strict static analysis.
- Third-party untyped APIs are wrapped in typed adapters.
- `Any` is isolated to integration edges.
- Type-checking errors are CI failures.

---

## 3. “Why is `Any` allowed to destroy the type system?”

In Java, type escapes usually look explicit: raw generics, unchecked casts, reflection, or `Object`:

```java
Object payload = fetchPayload();

int userId = (int) payload; // ClassCastException at the cast site if the assumption is wrong
```

In Python, `Any` can quietly disable static safety:

```python
from typing import Any

payload: Any = fetch_payload()

user_id: int = payload["user"]["id"]
payload.does_not_exist()
payload.some_method("wrong", "arguments")
```

A type checker generally accepts all of this because `Any` means “trust me.”

This can spread through a system:

```python
def get_config() -> Any:
    ...

def create_client() -> Any:
    ...

def execute_workflow() -> Any:
    ...
```

Better:

```python
from pydantic import BaseModel

class DatabaseConfig(BaseModel):
    host: str
    port: int
    database: str

def get_config() -> DatabaseConfig:
    return DatabaseConfig.model_validate(load_raw_config())
```

Treat `Any` like Java reflection or an unchecked cast: useful at narrow integration boundaries, dangerous as a normal application type.

---

## 4. “Where is the interface?”

Java developers expect an explicit interface:

```java
public interface EventPublisher {
    void publish(Event event);
}
```

Python commonly relies on duck typing:

```python
def publish_event(publisher, event) -> None:
    publisher.publish(event)
```

Anything with a compatible `publish()` method works.

This is flexible, but can feel invisible. A Java developer may ask:

- What methods are required?
- What is the contract?
- How does an IDE know what this object supports?
- How do I safely substitute implementations?

Use `Protocol` to express interface-like contracts without forcing inheritance:

```python
from typing import Protocol

class EventPublisher(Protocol):
    def publish(self, event: "Event") -> None:
        ...

def publish_event(
    publisher: EventPublisher,
    event: "Event",
) -> None:
    publisher.publish(event)
```

This preserves Python’s structural typing while giving static tooling an explicit contract.

---

## 5. “Private members are not actually private?”

Java has language-enforced access modifiers:

```java
private String apiKey;
protected void initialize();
public User getUser();
```

Python primarily relies on conventions:

```python
class ApiClient:
    def __init__(self, api_key: str) -> None:
        self._api_key = api_key

    def _refresh_token(self) -> None:
        ...
```

A leading underscore means:

> This is internal. Do not use it unless you accept breakage risk.

But nothing stops callers from doing this:

```python
client._refresh_token()
```

Double underscores use name mangling:

```python
class ApiClient:
    def __init__(self) -> None:
        self.__token = "secret"
```

But this is not true privacy or security. It merely reduces accidental collisions.

For a production Python package:

- Treat modules without `_` prefixes as the public API.
- Export supported symbols intentionally through `__init__.py`.
- Use documentation and semantic versioning for API stability.
- Mark internal modules, classes, and functions with `_`.
- Avoid assuming Python privacy conventions are security boundaries.

---

## 6. “Why can an object change shape at runtime?”

In Java, an object’s fields and methods are generally fixed by its class definition:

```java
class User {
    String name;
}

User user = new User();
user.permissions = List.of("admin"); // compile error: cannot find symbol
```

In Python, code can attach attributes dynamically:

```python
class User:
    pass

user = User()
user.name = "Miguel"
user.permissions = ["admin"]
```

It can also replace methods at runtime, monkey patch modules, or introspect and modify classes dynamically.

A Java developer may find this dangerous because object shape becomes less predictable.

For domain objects, prefer constrained models:

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class User:
    id: str
    email: str
    display_name: str
```

This improves safety:

- `frozen=True` prevents mutation after construction.
- `slots=True` prevents arbitrary new attributes.
- Type checkers verify declared field types.
- The model communicates its structure clearly.

For API payloads and untrusted input, use runtime validation:

```python
from pydantic import BaseModel, ConfigDict, EmailStr

class CreateUserRequest(BaseModel):
    model_config = ConfigDict(
        strict=True,
        extra="forbid",
    )

    email: EmailStr
    display_name: str
```

---

## 7. “Where are checked exceptions?”

Java makes some error handling explicit:

```java
public User getUser(String id) throws UserNotFoundException {
    ...
}
```

Python exceptions are unchecked:

```python
def get_user(user_id: str) -> "User":
    ...
```

The function may raise:

- `UserNotFoundError`
- `AuthorizationError`
- `TimeoutError`
- `ConnectionError`
- `ValueError`

Nothing in the signature requires the caller to address them.

The Python approach is to make operational behavior clear through:

- Domain-specific exception types.
- Docstrings for public APIs.
- Tests for expected error cases.
- Structured error responses at service boundaries.
- Narrow `try` blocks.

```python
class UserNotFoundError(Exception):
    pass

def get_user(user_id: str) -> User:
    user = repository.find(user_id)

    if user is None:
        raise UserNotFoundError(f"User not found: {user_id}")

    return user
```

Avoid broad handlers:

```python
try:
    ...
except Exception:
    ...
```

Prefer catching only errors you can meaningfully handle:

```python
try:
    user = get_user(user_id)
except UserNotFoundError:
    return {"error": "user_not_found"}, 404
```

---

## 8. “Why does `None` blow up later?”

Java developers are familiar with `null`, `Optional<T>`, and `NullPointerException`:

```java
Optional<User> findUser(String userId) { ... }

User user = findUser("123").get(); // throws NoSuchElementException if empty — still an unchecked
                                     // failure, but Optional<T> at least makes "might be absent"
                                     // visible in the signature
```

Python has `None`:

```python
def find_user(user_id: str) -> User | None:
    ...
```

Without static checking, this can fail later:

```python
user = find_user("123")
print(user.email)  # May fail if user is None
```

With strict typing, Python tools catch this:

```python
user = find_user("123")

if user is not None:
    print(user.email)
```

Or make absence explicit through an exception:

```python
user = get_user("123")
print(user.email)
```

The important distinction:

- `User | None` means a value may be absent.
- An optional function argument needs a default value.
- `Optional[T]` does not mean the parameter itself can be omitted.

```python
def bad_example(timeout: int | None) -> None:
    ...

def correct_example(timeout: int | None = None) -> None:
    ...
```

---

## 9. “Which package manager is the real one?”

Java engineers usually know the answer:

- Maven.
- Gradle.

```xml
<!-- pom.xml — one dependency file format, one build lifecycle -->
<project>
  <dependencies>
    <dependency>
      <groupId>com.example</groupId>
      <artifactId>some-library</artifactId>
      <version>1.2.3</version>
    </dependency>
  </dependencies>
</project>
```

Python developers may offer:

- `pip`
- `venv`
- `uv`
- Poetry
- PDM
- Hatch
- pip-tools
- Conda

This can create unnecessary decision fatigue.

A modern team should choose one standard per organization or repository. For a backend/API service, a practical baseline is:

- `uv` for environment and dependency management.
- `pyproject.toml` for project metadata and tool configuration.
- A committed lock file for reproducibility.
- `ruff` for linting and formatting.
- `pyright` for static type checking.
- `pytest` for testing.

Example workflow:

```bash
uv venv
uv sync
uv run ruff check .
uv run ruff format --check .
uv run pyright
uv run pytest
```

The complaint is not that Python cannot manage dependencies. It is that the ecosystem exposes more choices than Java developers expect.

---

## 10. “Which Python is running?”

Python environments frequently create confusion:

```bash
python
python3
pip
pip3
python -m pip
python3 -m pip
```

A Java developer expects the project build tool to select the JDK and dependency graph predictably:

```bash
mvn -version    # reports exactly which JDK Maven resolved and is using
./gradlew -version
```

Python requires explicit environment discipline:

```bash
uv venv
source .venv/bin/activate
python --version
python -m pip --version
```

Or, better, avoid activation ambiguity:

```bash
uv run python --version
uv run pytest
uv run pyright
```

Rules for reliable projects:

- Never install application dependencies globally.
- Use one virtual environment per project.
- Pin Python versions in CI and local tooling.
- Commit the dependency lock file.
- Run `pip` as `python -m pip` if using `pip`.
- Prefer task commands that hide environment details.

---

## 11. “Refactoring feels less safe”

Java refactoring benefits from:

- Compiler-enforced contracts.
- Strong IDE awareness of symbols and types.
- Explicit interfaces.
- Static dependency structures.
- Build failures for many incompatible changes.

```java
// renaming or removing this method fails the build immediately, at every call site,
// before a single test runs
public User getUser(String userId) { ... }
```

Python refactoring is safe when a team invests in the safety net:

- Strict type checking.
- High-value unit and integration tests.
- Contract tests for external dependencies.
- Runtime schema validation.
- Linting and formatting.
- CI gates.
- Strong IDE support.

Without that, a renamed keyword argument or removed dictionary field may only surface in an infrequently used runtime path.

One specific gap closed recently: `@typing.override` (PEP 698, Python 3.12+) is the direct analog of Java's `@Override`, and it catches the same refactoring bug — a base-class method renamed while a subclass keeps overriding the old name, silently becoming dead code.

```python
from typing import override

class AdminUser(User):
    @override
    def get_display_name(self) -> str:  # type checker errors if User no longer defines this
        ...
```

Like everything else in this section it's checker-enforced rather than language-enforced, so it only helps if the checker actually runs in CI.

A production baseline:

```bash
ruff check .
ruff format --check .
pyright
pytest --cov=src --cov-report=term-missing
```

---

## 12. “A dictionary is not a model”

Python code often starts like this:

```python
user = {
    "id": "usr_123",
    "email": "miguel@example.com",
    "roles": ["admin"],
}
```

Then gradually becomes this:

```python
user["role"]  # Typo: should be "roles"
user["email"] = 42
user["unknown_flag"] = True
```

Java developers may immediately ask for a class:

```java
record User(String id, String email, List<String> roles) {}
```

Use `TypedDict` for lightweight dictionary structures:

```python
from typing import TypedDict

class UserPayload(TypedDict):
    id: str
    email: str
    roles: list[str]
```

Use dataclasses for trusted domain objects:

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class User:
    id: str
    email: str
    roles: tuple[str, ...]
```

Use Pydantic for untrusted external data:

```python
from pydantic import BaseModel, ConfigDict, EmailStr

class UserRequest(BaseModel):
    model_config = ConfigDict(extra="forbid")

    email: EmailStr
    roles: list[str]
```

Convert raw JSON, YAML, queue messages, and database output into typed models early. Do not allow raw `dict[str, object]` values to leak through business logic.

---

## 13. “Why can I pass anything into a function?”

Python favors duck typing:

```python
def process(item) -> None:
    item.execute()
```

This is powerful, but a Java developer expects either an interface or a generic bound:

```java
interface Executable {
    void execute();
}

void process(Executable item) {
    item.execute();
}
```

Use a protocol:

```python
from typing import Protocol

class Executable(Protocol):
    def execute(self) -> None:
        ...

def process(item: Executable) -> None:
    item.execute()
```

This gives you:

- Python-style structural typing.
- No mandatory inheritance hierarchy.
- Editor autocomplete.
- Static verification.
- Safer substitutions and test doubles.

Where Java would reach for a generic bound rather than an interface, Python 3.12+ has syntax that reads much closer to Java's (PEP 695) — type parameters declared inline, no `TypeVar` import and no `Generic` base class:

```python
def first[T](items: list[T]) -> T:  # Python 3.12+
    return items[0]

class Repository[T]: ...
type MaybeUser = User | None
```

Below the 3.12 floor this is the older `T = TypeVar("T")` plus `Generic[T]` form, which is still valid and still what most existing code looks like. The new form isn't only shorter: its type parameters are properly scoped to the declaration instead of being module-level variables, and variance is inferred rather than declared.

---

## 14. “Why does this test double work but production fail?”

Java's own mocking frameworks (Mockito, and similar) are interface-based, so a mock fails to compile the moment the interface it's mocking changes:

```java
Executable client = mock(Executable.class);
doNothing().when(client).execute(); // fails to COMPILE if execute()'s signature ever changes
```

Python makes mocking easy because objects and functions can be replaced dynamically.

That flexibility can lead to overly permissive tests:

```python
from unittest.mock import Mock

client = Mock()
client.fetch.return_value = {"status": "ok"}
```

If the production interface changes, a weak mock may continue to pass.

Prefer interface-aware mocks:

```python
from unittest.mock import create_autospec

client = create_autospec(ApiClient, instance=True)
client.fetch.return_value = ApiResponse(status="ok")
```

Also prefer fakes and contract tests for important boundaries:

- HTTP clients.
- Queue publishers and consumers.
- Database repositories.
- Cloud SDK adapters.
- LLM gateway clients.
- MCP tool clients and servers.

---

## 15. “Why is there so much magic?”

Python frameworks can use decorators, metaclasses, reflection, import-time behavior, monkey patching, and dynamic registration.

Examples include:

- ORM model discovery.
- Dependency injection through decorators.
- Web-route registration at import time.
- Pydantic model construction.
- Framework configuration through global state.
- Plugin discovery through package metadata.

This can feel opaque to developers accustomed to explicit Java configuration and dependency injection:

```java
@Service
public class UserService {
    private final UserRepository repository;

    public UserService(UserRepository repository) { // constructor injection, wired at startup
        this.repository = repository;
    }
}
```

Use a few practical rules:

- Keep business logic framework-independent.
- Put framework code at application edges.
- Avoid import-time side effects.
- Prefer explicit dependency injection.
- Use typed configuration objects.
- Keep startup composition visible in one place.
- Document extension and plugin points.

Unlike Java, where Spring (or CDI) is the default choice for dependency injection, Python has no single dominant DI framework. Teams typically use plain constructor injection, or a small library like `dependency-injector`, or a web framework's built-in system (e.g., FastAPI's `Depends`). Pick one pattern deliberately rather than accumulating ad hoc wiring.

---

## 16. “Why didn’t adding a thread speed this up?”

Java threads map to OS threads and can run CPU-bound work in true parallel across cores:

```java
Thread t1 = new Thread(UserService::cpuBoundWork);
Thread t2 = new Thread(UserService::cpuBoundWork);
t1.start();
t2.start(); // genuinely runs in parallel on separate cores
```

The standard CPython build has a Global Interpreter Lock (GIL) that allows only one thread to execute Python bytecode at a time, regardless of core count:

```python
import threading

def cpu_bound_work() -> int:
    total = 0
    for i in range(50_000_000):
        total += i
    return total

threads = [threading.Thread(target=cpu_bound_work) for _ in range(4)]
```

On a standard interpreter, this does not run four times faster on four cores — the GIL serializes bytecode execution across threads.

As of Python 3.14, that's no longer the whole story. PEP 779 makes the free-threaded build (`python3.14t`, built on PEP 703) officially supported — on that build, the GIL is compiled out and `threading` *can* give real CPU-bound parallelism. It's still an opt-in build, not the default `python3.14` interpreter, and most published wheels still assume a GIL is present, so treat it as something to adopt deliberately (verify your dependencies support it) rather than a drop-in replacement.

The practical response, on the default (GIL) build:

- Use `threading` for IO-bound work (network calls, file IO) — the GIL is released during blocking IO, so threads still help here.
- Use `multiprocessing` or `concurrent.futures.ProcessPoolExecutor` for CPU-bound work, at the cost of process-level isolation and serialization overhead between processes.
- Use `asyncio` for high-concurrency IO-bound workloads, where cooperative scheduling replaces Java's thread-per-request model.
- Since 3.14, `concurrent.interpreters` (PEP 734) offers a fourth option: multiple isolated interpreters in one process, each with its own GIL. It gives process-like isolation without the pickling/IPC cost of `multiprocessing` — useful for CPU-bound work that needs isolation but not a separate OS process.
- On the free-threaded build, `threading` becomes viable for CPU-bound work too, once you've confirmed your dependencies are free-threading-safe.

One trap worth knowing before you reach for `asyncio.TaskGroup`: when concurrent tasks fail, they fail *together*. A `TaskGroup` raises `ExceptionGroup` (Python 3.11+), and an ordinary `except` clause does not match the exceptions wrapped inside it — you need `except*`:

```python
try:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(fetch_user())
        tg.create_task(fetch_orders())
except* ValueError as eg:      # matches ValueErrors *inside* the group
    handle(eg.exceptions)
```

Writing `except ValueError` here silently fails to catch anything, because the raised object is an `ExceptionGroup`, not a `ValueError`. Java has no equivalent construct — its structured concurrency proposal surfaces the first failure and cancels the rest, rather than handing you a tree of concurrent failures to destructure.

A Java developer coming from JDK 21 or later, used to virtual threads — write blocking-style code, get async-style scalability for free — will notice `asyncio` requires explicit `async`/`await` coloring through the whole call stack; there is no equivalent that hides the concurrency model from calling code. (A JDK 17 developer won't have that comparison point at all — virtual threads finalized in JDK 21. And Java's own structured concurrency, the piece that would pair with virtual threads the way `asyncio.TaskGroup` pairs with `asyncio`, is still in preview as of JDK 27 — JEP 533, seventh preview, with finalization now targeted for JDK 28 — so it isn't a stable comparison to lean on yet either.)

```text
IO-bound, low concurrency:            threading
IO-bound, high concurrency:           asyncio
CPU-bound, process isolation OK:      multiprocessing / ProcessPoolExecutor
CPU-bound, lighter isolation:         concurrent.interpreters (3.14+)
CPU-bound, free-threaded build:       threading
```

---

## 17. “Why did `==` return `False` (or the wrong `True`)?”

Java engineers are trained to reflexively call `.equals()`, because `==` on objects compares references:

```java
List<Integer> a = List.of(1, 2, 3);
List<Integer> b = List.of(1, 2, 3);
a.equals(b); // true — List.equals() defines value equality
a == b;      // false — reference equality, as always

class Point { int x, y; }

Point p1 = new Point();
Point p2 = new Point();
p1.equals(p2); // false — Object.equals() defaults to reference equality unless overridden
```

Python inverts the default: `==` calls `__eq__`, which many built-in types define as value equality, but a plain user-defined class does not:

```python
a = [1, 2, 3]
b = [1, 2, 3]
a == b   # True — list defines value equality
a is b   # False — different objects

class Point:
    def __init__(self, x: int, y: int) -> None:
        self.x = x
        self.y = y

p1 = Point(1, 2)
p2 = Point(1, 2)
p1 == p2   # False — Point has no __eq__, so identity is used
```

A Java developer's instinct ("just use `==`, it's not Java's reference `==`") is right for lists, dicts, strings, and numbers, but wrong for a plain custom class — the opposite failure mode from Java's "forgot to override `equals()`."

The practical response:

- `@dataclass` generates `__eq__` automatically (see §12) — prefer it for value objects.
- Reserve `is` for identity checks that are actually about identity: `is None`, `is True`, sentinel objects, singleton checks.
- Never use `is` to compare values, even for small integers or short strings — CPython's caching of those is an implementation detail, not a guarantee.

---

## 18. “Why does this list keep growing across unrelated calls?”

```python
def add_item(item, basket=[]):
    basket.append(item)
    return basket

add_item("apple")   # ['apple']
add_item("banana")  # ['apple', 'banana']  <- shared across every call
```

Default argument values are evaluated once, at function definition time, not once per call. The same list object is reused on every invocation that doesn't pass `basket` explicitly.

There is no Java equivalent to reach for here — a Java method parameter default (via overloading) is re-evaluated per call, so this behavior has no familiar analog to fall back on:

```java
List<String> addItem(String item, List<String> basket) {
    basket.add(item);
    return basket;
}

List<String> addItem(String item) {
    return addItem(item, new ArrayList<>()); // a fresh list, constructed fresh on every call
}
```

```python
def add_item(item: str, basket: list[str] | None = None) -> list[str]:
    if basket is None:
        basket = []
    basket.append(item)
    return basket
```

Rule of thumb: never use a mutable literal (`[]`, `{}`, `set()`, or a mutable object) as a default argument value. Use `None` and construct the default inside the function body.

---

## 19. “Why does this import work when I run it one way but not another?”

Java resolves imports against the classpath at compile time, and a package maps directly to a directory structure. A fat/uber JAR bundles everything into one deployable artifact:

```java
package com.example.app;

import com.example.app.config.Config; // resolved against the classpath at compile time,
                                        // identically regardless of how the JVM was invoked
```

Python resolves imports against `sys.path` at runtime, and relative imports behave differently depending on how a module was invoked:

```python
# app/worker.py
from . import config

# python -m app.worker   -> works, worker.py is imported as part of the app package
# python app/worker.py   -> ImportError: attempted relative import with no known parent package
```

Circular imports are also more common than in Java, because Python executes a module's top-level code the first time it's imported, rather than resolving a full dependency graph up front.

The practical response:

- Run application code as a module (`python -m package.module`) or through a packaged entry point, not as a loose file path.
- Keep the import graph shallow; break cycles by moving shared types into a leaf module both sides can import.
- There is no single fat-jar equivalent for deployment — pick one deliberately: a locked `uv`/`pip` install baked into a container image, a `.pyz` zipapp, or a wheel published to a private index.

---

## 20. “How do I overload this method?”

Java lets you define multiple methods with the same name and different parameter types; the compiler picks the right one at the call site:

```java
void send(String message) { ... }
void send(String message, int priority) { ... } // the compiler picks the right one at each call site
```

Python allows only one function definition per name in a given scope — a second `def` silently replaces the first:

```python
def send(message: str) -> None: ...
def send(message: str, priority: int) -> None: ...  # this is the only `send` that exists now
```

The practical response:

- Default and keyword arguments cover most cases:

```python
def send(message: str, priority: int = 0) -> None: ...
```

- For genuinely different parameter types, use `functools.singledispatch`:

```python
from functools import singledispatch

@singledispatch
def render(value: object) -> str: ...

@render.register
def _(value: int) -> str:
    return f"int:{value}"

@render.register
def _(value: str) -> str:
    return f"str:{value}"
```

---

## 21. “Wait, a class can inherit from more than one class?”

Java restricts a class to a single superclass and covers the rest with interfaces:

```java
class Event implements Serializable, Timestamped { // multiple interfaces, one superclass only
    ...
}
```

Python permits multiple inheritance directly:

```python
class Serializable:
    def to_json(self) -> str: ...

class Timestamped:
    def touch(self) -> None: ...

class Event(Serializable, Timestamped):
    pass
```

Method resolution follows C3 linearization (the "MRO"), not a simple left-to-right search, which can surprise developers used to Java's single-inheritance-plus-interfaces model — especially with `super()` calls in diamond-shaped hierarchies.

```python
print(Event.__mro__)
```

The practical response:

- Prefer composition or `Protocol` (§4) over multiple inheritance for anything beyond a simple mixin.
- If using mixins, keep them stateless and single-purpose, and avoid diamond-shaped or deep hierarchies.
- Inspect `ClassName.__mro__` when a method's resolution order isn't obvious from reading the class definition.

---

## 22. “Why is deserializing this considered dangerous?”

Java's own default serialization has a well-documented history of deserialization vulnerabilities, so the category isn't unfamiliar:

```java
ObjectInputStream in = new ObjectInputStream(untrustedStream);
Object data = in.readObject(); // can execute arbitrary code during deserialization —
                                 // the same class of risk CVE-driven guidance warns against
```

Python's equivalent is easy to reach for without noticing the risk.

```python
import pickle

data = pickle.loads(untrusted_bytes)  # can execute arbitrary code during unpickling
```

`pickle` reconstructs objects by calling constructors and `__reduce__` hooks while deserializing — it is not a safe format for data crossing a trust boundary.

The practical response:

- Never call `pickle.load`/`pickle.loads` on data from a network request, message queue, or user upload.
- Use `json` or a Pydantic model (§6, §12) for anything crossing a trust boundary.
- Reserve `pickle` for trusted, internal-only use — e.g., caching an object between processes you control.

The adjacent trust-boundary problem is interpolation. An f-string builds a finished string with no record of which parts came from user input, so `f"SELECT * FROM users WHERE id = {user_id}"` is the SQL-injection shape a Java developer avoids with `PreparedStatement`. Template strings (PEP 750, Python 3.14+) add a `t` prefix that produces a `Template` object instead of a `str`, keeping static text and interpolated values separate so a library can escape or parameterize them before assembly:

```python
query = t"SELECT * FROM users WHERE id = {user_id}"
type(query)  # string.templatelib.Template — not a str, so it can't be executed by accident
```

This is new enough that library support is still arriving; parameterized queries through your database driver remain the answer today. Its value for a Java developer is structural — a t-string can't be passed where a finished `str` is required, which is exactly the property `PreparedStatement` relies on.

---

## 23. “Where is the Javadoc equivalent?”

Java compiles Javadoc comments into browsable HTML API docs as a standard part of the build:

```java
/**
 * Fetch a user by id.
 *
 * @param userId the user's id
 * @return the matching user
 * @throws UserNotFoundException if no user matches {@code userId}
 */
User getUser(String userId) { ... }
```

Python's equivalent is docstrings plus a separate documentation generator — there's no single default the way `javadoc` is standard:

```python
def get_user(user_id: str) -> User:
    """Fetch a user by id.

    Raises:
        UserNotFoundError: if no user matches ``user_id``.
    """
    ...
```

The practical response:

- Pick one docstring convention (Google, NumPy, or reST style) and lint it — `ruff`'s `D` rule set enforces docstring presence and format.
- Generate API docs with `mkdocs` + `mkdocstrings`, or `Sphinx` with `autodoc`, and publish them as part of CI rather than leaving docstrings unread.

---

## 24. “Why didn’t it warn me I forgot a case?”

Java developers working with a sealed hierarchy expect the compiler to enforce exhaustiveness. Sealed classes were finalized in JDK 17 and pattern matching for `switch` in JDK 21, so a Java developer on either version reaches for exactly this pattern:

```java
sealed interface Shape permits Circle, Square {}
record Circle(double radius) implements Shape {}
record Square(double side) implements Shape {}

double area(Shape shape) {
    return switch (shape) {
        case Circle c -> Math.PI * c.radius() * c.radius();
        case Square s -> s.side() * s.side();
        // compiler error if a case is missing
    };
}
```

Python's structural pattern matching (`match`, since 3.10) looks similar but has no compiler behind it:

```python
def area(shape: Circle | Square) -> float:
    match shape:
        case Circle(radius=r):
            return 3.14159 * r * r
        case Square(side=s):
            return s * s
        # missing a case is a silent None return, not an error
```

Add a third shape to the union and nothing complains — the function just returns `None` for the case nobody handled.

The practical response is the same static-checking discipline as the rest of this guide, applied to `match`:

```python
from typing import assert_never

def area(shape: Circle | Square) -> float:
    match shape:
        case Circle(radius=r):
            return 3.14159 * r * r
        case Square(side=s):
            return s * s
        case _:
            assert_never(shape)  # pyright/mypy flag this if a variant is unhandled
```

`typing.assert_never` requires Python 3.11+ — on 3.10, import it from `typing_extensions` instead (`from typing_extensions import assert_never`).

`assert_never` is unreachable at runtime as long as every case is handled; if a new variant is added to the union later without a matching `case`, the type checker (not the language) reports the gap at the `assert_never` call.

Two notes worth keeping current on this specific comparison:

- Python's dataclasses (§12) plus `X | Y` unions map onto sealed records reasonably well, but there's no `permits` clause — anything can subclass a "closed" hierarchy unless you also rely on `@final` (§1) and static checking to catch it.
- As of JDK 27, Java itself is still extending pattern matching — primitive types in patterns/`instanceof`/`switch` are in a fifth preview (JEP 532) — so the Java side of this comparison is not fully settled either.

---

## 25. “Why did this object get cleaned up instantly — and why did that other one never get cleaned up at all?”

Java's GC timing is intentionally unspecified — an object becomes eligible for collection once unreachable, but *when* the collector actually reclaims it is never something a Java program can rely on, which is exactly why `Object.finalize()` was deprecated (JDK 9) and removed outright (JDK 18) in favor of `try`-with-resources/`AutoCloseable`.

CPython's primary reclamation mechanism is reference counting, not a JVM-style tracing collector: an object is freed the instant its refcount hits zero, deterministically, in the same statement that dropped the last reference.

```python
class Resource:
    def __del__(self):
        print("closed")

def use():
    r = Resource()
    ...
    # r's refcount hits zero here — __del__ runs immediately, on CPython
```

That determinism is real, but it only covers non-cyclic garbage. A reference cycle never hits a refcount of zero on its own, so CPython layers a separate, generational cyclic collector (the `gc` module) on top, purely to find and break cycles — and that part behaves much more like Java's GC: periodic, non-deterministic in exact timing, and pausing to scan.

```python
class Node:
    def __init__(self):
        self.other = None

a, b = Node(), Node()
a.other, b.other = b, a  # a reference cycle
del a, b                 # refcounts never reach zero — each is still held by the other;
                          # reclaimed only whenever the cyclic collector next runs
```

The practical response:

- Never rely on `__del__` for anything correctness-critical — releasing a lock, flushing a file, closing a socket. Use a context manager (`with`, `contextlib.contextmanager`) instead; that's Python's actual equivalent of `try`-with-resources/`AutoCloseable`, with guaranteed, immediate timing.
- Don't generalize CPython's immediate refcount-based deallocation to "the language." PyPy and other implementations use a tracing GC with no reference counting at all, so `__del__` timing that looks instantaneous on CPython can be arbitrarily delayed elsewhere.
- Reference cycles won't leak permanently in CPython — the generational `gc` module collects them — but they defer reclamation and add periodic full-heap scan pauses in a long-running service. Break cycles explicitly (`weakref` for back-references, like a child pointing back to its parent) in hot paths rather than relying on the cyclic collector to catch it eventually.
- `gc.disable()` is a known trick for shaving latency in short-lived scripts (refcounting alone still frees ordinary garbage without it), but doing that in a long-running server without periodically calling `gc.collect()` lets cyclic garbage accumulate without bound.

---

## 26. “Why did this type annotation crash at import time?”

In Java, a type in a signature is resolved by the compiler across the whole compilation unit before any code runs, so a class referring to itself is unremarkable:

```java
class Node {
    Node parent;
    Node addChild(Node child) { ... } // referring to Node inside Node is fine
}
```

A Python annotation is not metadata the way a Java annotation is — it's an ordinary expression, and through Python 3.13 it was evaluated eagerly, at the moment the `def` or `class` statement executed:

```python
class Node:
    def add_child(self, child: Node) -> Node:  # NameError on 3.13 and earlier —
        ...                                     # Node doesn't exist yet while its own body runs
```

Two workarounds grew up around this. Quoting the name (`-> "Node"`) defers it to a string, which is why quoted forward references appear throughout typed Python — including in §7 of this guide. The broader fix, `from __future__ import annotations` (PEP 563), stringified *every* annotation in the module, which solved import-time crashes but broke the libraries that read annotations at runtime — Pydantic, dataclasses, FastAPI — since they now received strings where they expected type objects.

Python 3.14 changes the default. Under PEP 649 (implemented via PEP 749), annotations are stored in a lazily-evaluated function and computed only on first access, so forward references resolve without quoting and without stringifying anything:

```python
# Python 3.14+
class Node:
    def add_child(self, child: Node) -> Node:  # fine — not evaluated at class-definition time
        ...

from annotationlib import get_annotations, Format

get_annotations(Node.add_child, format=Format.VALUE)      # real objects, may raise NameError
get_annotations(Node.add_child, format=Format.FORWARDREF) # unresolved names become ForwardRef
get_annotations(Node.add_child, format=Format.STRING)     # annotations as strings
```

The practical response:

- On 3.14+, stop adding `from __future__ import annotations` to new modules — forward references work unquoted. The import still behaves as it always did, so existing modules don't need an urgent migration.
- At this guide's 3.10 floor you still need one of the two workarounds. Prefer quoting individual forward references over the module-wide `__future__` import, precisely because the latter is the one that surprises runtime consumers.
- If you write code that *reads* annotations — a DI container, a serializer, the kind of thing you'd do with reflection in Java — use `annotationlib.get_annotations()` with an explicit format rather than reaching into `__annotations__`, which now means different things on different versions.
- Remember that annotations are executable expressions in a way Java's never are: an expensive or side-effecting annotation is real code, and before 3.14 it ran at import.

---

## 27. “Why is this ten times slower than the equivalent Java?”

A hot loop in Java is interpreted briefly, then compiled to native code by HotSpot's tiered JIT, with inlining and loop optimizations applied:

```java
long total = 0;
for (int i = 0; i < 50_000_000; i++) {
    total += i; // C2 compiles this to tight machine code once it runs hot
}
```

The same loop in CPython runs through the bytecode interpreter on every single iteration, and the default build has no optimizing JIT to graduate it to native code:

```python
total = 0
for i in range(50_000_000):
    total += i  # seconds, not milliseconds
```

The gap is real, but it has been narrowing. Python 3.11 shipped the specializing adaptive interpreter (PEP 659), which rewrites hot bytecodes into type-specialized variants — conceptually similar to a JIT's inline caches, though still interpretation rather than native code — and measured about 1.25x faster than 3.10 across the pyperformance suite. Python 3.13 added an experimental copy-and-patch JIT (PEP 744), disabled by default; as of 3.14 it remains experimental, though the official Windows and macOS binaries now ship with support for it compiled in.

The practical response:

- Don't port a hot numeric loop line-by-line from Java and expect the JIT to rescue it. The Python answer is to stop executing Python bytecode in the loop at all — push it into NumPy, Polars, or a native extension. That's an architectural difference, not a tuning exercise.
- Measure before optimizing: for the IO-bound service code that makes up most backends, interpreter speed is rarely the bottleneck, and the network dominates.
- Don't enable the experimental JIT in production expecting HotSpot-like gains — it's off by default and still experimental as of 3.14, with no stability guarantees.
- Keep the interpreter floor moving. Version upgrades have delivered real single-threaded gains recently, in a way that was not true across the 3.x releases a Java developer may remember.
- Single-threaded speed and multi-core scaling are separate problems — §16 covers the GIL and the free-threaded build, which on 3.14 costs roughly 5–10% single-threaded to buy real parallelism.

---

## What Python Gets Right

The friction above shouldn't read as "Python is worse." Several things are genuinely nicer once a Java engineer settles in:

- No compile step: the edit-run loop is immediate, and a REPL is available for exploring an API or reproducing a bug interactively.
- `@dataclass` generates `__init__`, `__repr__`, and `__eq__` for free (see §17) — no IDE-generated boilerplate to keep in sync.
- Comprehensions and built-in `list`/`dict`/`set` operations replace a lot of Java's Stream API ceremony for the common cases.
- A large, consistent standard library covers most day-to-day tasks without reaching for a dependency.
- Duck typing and `Protocol` (§4) let you retrofit an interface onto existing code without a refactor — no need to have planned the abstraction in advance.

None of this replaces the discipline this guide argues for — it's why that discipline is worth adding rather than a reason to skip it.

---

## Recommended Python Baseline for Java Engineers

For production backend, API-platform, cloud, and AI-agent services:

Python has no LTS, so the practical equivalent of a Java team's LTS discipline is "current stable, minus one, once your wheels are ready." Python 3.14 is the current stable release; the pins below sit one release back, which is the conservative choice for dependency and wheel availability. Move to 3.14 deliberately when you want PEP 649 annotation semantics (§26), t-strings (§22), or the officially supported free-threaded build (§16).

```text
Python:       Python 3.13+
Packages:     uv + pyproject.toml + lock file
Formatting:   Ruff format
Linting:      Ruff check
Typing:       Pyright strict mode
Tests:        pytest
Validation:   Pydantic at external boundaries
Models:       dataclasses with frozen=True and slots=True
Interfaces:   typing.Protocol
Security:     pip-audit for dependency vulnerability scanning
Docs:         docstrings + mkdocstrings or Sphinx
CI:           lint + format check + type check + tests + dependency audit
Containers:   pinned Python base image and reproducible lock-based installs
```

Example `pyproject.toml`:

```toml
[project]
name = "api-service"
version = "0.1.0"
requires-python = ">=3.13"
dependencies = [
  "fastapi>=0.115",
  "pydantic>=2.0",
  "uvicorn>=0.30",
]

[dependency-groups]
dev = [
  "pyright>=1.1",
  "pytest>=8.0",
  "pytest-cov>=5.0",
  "ruff>=0.8",
  "pip-audit>=2.7",
]

[tool.pyright]
typeCheckingMode = "strict"
pythonVersion = "3.13"
include = ["src"]

[tool.ruff]
target-version = "py313"
line-length = 100

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-q"
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

      - uses: astral-sh/setup-uv@v5

      - run: uv python install 3.13
      - run: uv sync --locked
      - run: uv run ruff format --check .
      - run: uv run ruff check .
      - run: uv run pyright
      - run: uv run pytest
      - run: uv run pip-audit
```

## Bottom Line

The Java developer’s complaint is usually valid:

> Python does not force the engineering discipline that Java makes difficult to avoid.

But that is also Python’s strength. It lets teams choose the amount of ceremony and enforcement appropriate for the problem.

For serious API and platform engineering, treat Python as a typed, tested, validated language by policy:

1. Use strict static checking.
2. Validate all untrusted data at service boundaries.
3. Model domain data explicitly.
4. Keep dynamic behavior localized.
5. Lock dependencies and standardize project tooling.
6. Enforce quality gates in CI.

With those constraints, Python remains concise and productive while providing a level of safety that feels much more familiar to an experienced Java engineer.