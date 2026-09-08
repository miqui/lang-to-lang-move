# What a Java Developer Will Complain About When Moving to TypeScript

This guide is about the type system specifically — for the runtime and platform concerns (the event loop, `npm`, module systems, testing frameworks), see [java-to-nodejs](../java-to-nodejs/) in this repository. TypeScript recovers a real amount of what Java developers expect from a type system, but it was grafted onto JavaScript rather than designed alongside a compiler and a VM the way Java's was — so the shape of the type system, and where it stops helping you, is genuinely different, not just stricter or looser.

## 1. “Why is this assignable? These classes have nothing to do with each other.”

Java's type system is nominal — a type is compatible with another only if it's declared to be, through `extends` or `implements`:

```java
class Point { double x, y; }
class Vector2D { double x, y; }

Point p = new Vector2D(); // compile error: incompatible types
```

TypeScript's type system is structural — a type is compatible with another if its shape matches, regardless of declared relationship:

```typescript
class Point {
  constructor(public x: number, public y: number) {}
}

class Vector2D {
  constructor(public x: number, public y: number) {}
}

const p: Point = new Vector2D(); // compiles — Vector2D has the shape Point requires
```

Two types with no declared relationship, even two entirely unrelated domain concepts, are interchangeable if their public members line up.

The practical response:

- For types that are structurally identical but semantically distinct (an `OrderId` and a `UserId` both wrapping a `string`), use a nominal-typing workaround — a "branded" type — since TypeScript has no native nominal typing:

```typescript
type UserId = string & { readonly __brand: "UserId" };
type OrderId = string & { readonly __brand: "OrderId" };
```

- Don't rely on structural compatibility as documentation of intent — two types matching by shape doesn't mean they're supposed to be interchangeable. Reach for `interface`/`type` names and doc comments to say what nominal typing would otherwise say for free.

---

## 2. “Why did this object literal get rejected, but the exact same object through a variable didn't?”

Java has no equivalent hazard, because interfaces are nominal (§1) — you always explicitly declare which class implements an interface, and extra fields on that class are simply never inspected:

```java
interface Options {
    int timeout();
}

class Config implements Options {
    public int timeout() { return 5000; }
    public int retries = 3; // extra field — Java has no concept of "excess" here at all
}

void configure(Options opts) { ... }
configure(new Config()); // always fine; extra fields on an implementer are never flagged
```

TypeScript's structural typing (§1) does try to catch a related mistake, but only in one narrow case:

```typescript
interface Options {
  timeout: number;
}

function configure(opts: Options) { ... }

configure({ timeout: 5000, retries: 3 });
// error: 'retries' does not exist in type 'Options'

const opts = { timeout: 5000, retries: 3 };
configure(opts); // compiles fine — same object, no error
```

This is "excess property checking," and it applies only to object *literals* passed directly at a call site, not to a variable holding the same shape — because once the literal is assigned to `opts`, TypeScript widens it to `{ timeout: number; retries: number }`, which is structurally still assignable to `Options` (§1) even though it has an extra field.

The practical response: don't treat a passing type check as proof a typo-laden object matches the intended shape — excess property checking is a narrow, literal-only safety net, not general protection against extra or misspelled fields. When accepting external data with extra fields that must be rejected, validate at runtime (a schema library) rather than relying on this compile-time-only, call-site-only check.

---

## 3. “Where did the type go at runtime?”

Java generics are erased too, but Java's non-generic types (`instanceof SomeClass`) survive to runtime because the JVM tracks class identity:

```java
class User {
    String id;
}

Object obj = new User();
if (obj instanceof User) { // works fine — the JVM tracks concrete class identity at runtime
    System.out.println("it's a User");
}
```

TypeScript erases essentially everything type-level, including `interface` and `type` declarations, which have no runtime representation at all:

```typescript
interface User {
  id: string;
}

function isUser(x: unknown): x is User {
  return x instanceof User; // error: 'User' only refers to a type, but is being used as a value here
}
```

There's no `User` to check `instanceof` against — it never existed after compilation.

The practical response:

- Use a runtime-checkable construct when you need a real runtime check: a `class` (which does compile to something `instanceof`-checkable), a discriminated union with a literal tag field (§9), or a schema validator (`zod`) that both validates and infers the static type from one definition.
- Treat `interface`/`type` as compiler-only documentation — never write code that assumes they exist once the file is compiled to JavaScript.
- The erasure is near-total, which is a useful mental model: delete every piece of type syntax and you generally have working JavaScript. The exceptions are `enum`, `namespace`, and constructor parameter properties, which emit real runtime code — see §6, and `--erasableSyntaxOnly` (TypeScript 5.8+) for enforcing the rule.

---

## 4. “Why does `unknown` behave differently than `any` — aren't they both ‘could be anything’?”

Java's closest equivalent to "could be anything" is `Object`, which still requires an explicit cast before you can call a specific method on it — the compiler won't let you call `.length()` on an `Object` without casting first:

```java
Object payload = fetchPayload();
payload.length(); // compile error: cannot find symbol — must cast first
String s = (String) payload; // compiles, but throws ClassCastException at runtime if wrong
s.length();
```

TypeScript has two very different "anything" types. `any` disables checking entirely — it behaves like Java's `Object` with the cast already silently applied:

```typescript
function handle(payload: any) {
  payload.whatever();      // compiles, no cast needed, may crash at runtime
  payload.a.b.c.d();       // also compiles
}
```

`unknown` is the sound version — it accepts anything, but forces narrowing before you can do anything with it, closer to what Java's `Object` actually enforces:

```typescript
function handle(payload: unknown) {
  payload.whatever(); // error: 'payload' is of type 'unknown'

  if (typeof payload === "object" && payload !== null && "whatever" in payload) {
    // narrowed enough to proceed safely
  }
}
```

The practical response: default to `unknown` at any boundary where external data enters (parsed JSON, a library with loose types), and never `any`. Lint for it — `@typescript-eslint/no-explicit-any` — and treat every remaining `any` in a codebase as a specific, reviewable exception rather than a default.

---

## 5. “Why did narrowing just stop working?”

Java doesn't have flow-sensitive typing beyond effectively-final local variable capture, so this class of surprise doesn't really exist there — once you've null-checked something in Java, nothing implicit can invalidate that:

```java
String name = user.getName();
if (name != null) {
    Runnable r = () -> System.out.println(name.toUpperCase());
    // safe: `name` is captured as an effectively-final local holding the
    // already-null-checked value — nothing can re-read a changed field later
    r.run();
}
```

TypeScript's narrowing is flow-sensitive and looks like it tracks real invariants, but it only reasons about what's visible in the current function body — it can't see through a function call:

```typescript
function process(user: { name: string | null }) {
  if (user.name !== null) {
    setTimeout(() => {
      console.log(user.name.toUpperCase()); // error: 'user.name' is possibly 'null'
    }, 0);
  }
}
```

Even though the outer `if` checked `user.name !== null`, TypeScript won't carry that narrowing into a closure — the closure could run after `user.name` changes, and the compiler doesn't try to prove otherwise. The same loss of narrowing happens across any function boundary, including calling a helper function that itself does the null check.

The practical response:

- Capture the narrowed value into a local `const` before it needs to cross a function boundary — narrowing on a `const` binding is more durable than narrowing on a property access:

```typescript
if (user.name !== null) {
  const name = user.name; // now a plain string, narrowing survives the closure
  setTimeout(() => console.log(name.toUpperCase()), 0);
}
```

- Don't assume a helper function that "does the null check" communicates that fact to the type checker — it doesn't, unless it's declared as a type predicate (`x is T`) or assertion function (`asserts x is T`).

---

## 6. “Are TypeScript `enum`s actually enums?”

Java's `enum` is a closed, type-safe set of singleton instances — you cannot construct an out-of-range value, and switching over one can be checked for exhaustiveness (§9):

```java
enum Status { ACTIVE, INACTIVE }

void setStatus(Status s) { ... }

setStatus(99); // compile error: incompatible types — only Status constants are valid
```

TypeScript's numeric `enum` looks similar but isn't closed:

```typescript
enum Status {
  Active,
  Inactive,
}

function setStatus(s: Status) { ... }

setStatus(99); // compiles — any number is assignable to a numeric enum
```

Numeric enums also generate a reverse mapping at runtime (`Status[0] === "Active"`), which doubles the generated JavaScript and surprises anyone expecting an enum to be a small, closed value like Java's.

The practical response: prefer a union of string literals over `enum` for new code — it's closed the way Java developers expect, requires no runtime code generation, and narrows and displays better:

```typescript
type Status = "active" | "inactive";

function setStatus(s: Status) { ... }

setStatus("unknown"); // error: not assignable to type 'Status'
```

If you need an actual object grouping the values (for iteration, or a value-to-label mapping), pair the literal union with an `as const` object instead of `enum`:

```typescript
const Status = { Active: "active", Inactive: "inactive" } as const;
type Status = (typeof Status)[keyof typeof Status];
```

There's now a second, harder reason to avoid `enum`. Almost everything in TypeScript is *erasable* — deleting the type syntax leaves valid JavaScript with identical behavior (§3). Three constructs aren't: `enum`, `namespace`, and constructor parameter properties (`constructor(private db: Database)`) all emit real runtime code. That makes them incompatible with runtimes that strip types rather than compile them, which is what Node's native TypeScript execution does — see [java-to-nodejs](../java-to-nodejs/) for that side of it. TypeScript 5.8 added `--erasableSyntaxOnly`, which makes using any of the three a compile error:

```json
{ "compilerOptions": { "erasableSyntaxOnly": true } }
```

For a Java developer this is the reverse of the usual intuition: the constructs that look *most* like their Java counterparts — a real `enum`, a `namespace` resembling a package, constructor parameter properties resembling a concise constructor — are exactly the ones that aren't really part of the type layer at all.

---

## 7. “Why did this overload pick the wrong signature?”

Unlike Python and Go, TypeScript does support function overloading — but the mechanism is different from Java's, and it's a common source of confusion for exactly that reason.

Java resolves an overload at compile time based on the static types of the arguments, and every overload is independently callable and independently implemented:

```java
void send(String message) { ... }
void send(String message, int priority) { ... }
```

TypeScript overloads are a list of call *signatures* checked against a single shared implementation, resolved by picking the first signature in the list that matches — not the most specific one:

```typescript
function format(value: string): string;
function format(value: string | number): string;
function format(value: string | number): string {
  return String(value);
}

format(42); // resolves to the second signature (correct here, but only because
             // the first one — string — doesn't match at all)
```

If two overload signatures could both plausibly match a given call, TypeScript always picks whichever is listed first — there's no "most specific wins" resolution the way there is in Java.

The practical response:

- Order overload signatures from most specific to least specific — the reverse of how it might read naturally.
- The implementation signature (the one with a body) is never visible to callers — only the declared overload signatures are — so it can be broader than any individual overload without affecting what callers can pass.
- If the overloads don't clearly express intent to a reader, a union parameter type with an internal `if`/`typeof` branch is often more readable than an overload list.

---

## 8. “Why did this ‘immutable’ array get mutated?”

Java's `final` prevents reassigning a variable, but doesn't make the referenced object immutable — a Java developer already knows not to expect deep immutability from `final` alone:

```java
final List<String> hosts = new ArrayList<>(List.of("api.example.com"));
hosts.add("evil.example.com"); // compiles and runs fine — `final` only blocks
                                 // reassigning the `hosts` variable itself
System.out.println(hosts); // [api.example.com, evil.example.com]
```

TypeScript's `readonly` has the same shallowness, plus it's compile-time only, which adds a second gap Java's `final` doesn't have:

```typescript
interface Config {
  readonly hosts: string[];
}

function printFirst(config: Config) {
  config.hosts.push("evil.example.com"); // no error — readonly doesn't apply to array contents
  console.log(config.hosts[0]);
}

const config: Config = { hosts: ["api.example.com"] };
printFirst(config);
console.log(config.hosts); // ["api.example.com", "evil.example.com"] — mutated
```

`readonly hosts: string[]` only prevents reassigning `config.hosts` to a *different* array — it says nothing about mutating the array that's already there. And because `readonly` is erased at compile time (§3), nothing stops a caller who ignores (or doesn't have) the type from mutating it anyway.

The practical response:

- Use `ReadonlyArray<T>` (or the `readonly T[]` shorthand) instead of `T[]` for a field that shouldn't be mutated — this at least removes `push`/`pop`/`splice` etc. from the type, closing the specific hole above.
- For real runtime immutability — protection against code that doesn't respect the type, exactly the gap `readonly` can't close — use `Object.freeze()`, the same way you'd reach for an unmodifiable collection wrapper in Java when `final` alone isn't enough.

---

## 9. “Where are my sealed types and switch exhaustiveness?”

This is the direct TypeScript analog to [the same question for Java-to-Python movers](../java-to-python/README.md) — Java developers on JDK 17+ (sealed classes) or JDK 21+ (pattern matching for `switch`) expect the compiler to catch a missing case:

```java
sealed interface Shape permits Circle, Square {}
record Circle(double radius) implements Shape {}
record Square(double side) implements Shape {}

double area(Shape shape) {
    return switch (shape) {
        case Circle c -> Math.PI * c.radius() * c.radius();
        case Square s -> s.side() * s.side();
        // compile error if a case is missing — the compiler tracks the permits list
    };
}
```

TypeScript's answer is the discriminated union plus a `never`-based exhaustiveness check, and it's arguably the closest of any language in this repository to what Java 21 gives you natively:

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
    default: {
      const _exhaustive: never = shape; // compile error if a case is missing
      throw new Error(`unhandled shape: ${JSON.stringify(shape)}`);
    }
  }
}
```

Add a third member to the `Shape` union without adding a matching `case`, and the `default` branch's `shape` is no longer assignable to `never` — the build fails at the exhaustiveness check, not silently at runtime the way an unhandled `match` case in Python does.

The practical response: treat this pattern — a literal `kind`/`type` discriminant field plus a `never` assignment in the `default` case — as the standard shape for any closed set of variants, the direct equivalent of a Java sealed hierarchy. Extract it into a small `assertNever(x: never): never` helper used across a codebase rather than rewriting the `throw` each time.

---

## 10. “Why did declaring the same interface twice not error?”

In Java, declaring two types with the same name in the same scope is always a compile error, full stop:

```java
interface Window {
    String title();
}

interface Window { // compile error: duplicate class: 'Window'
    int height();
}
```

TypeScript `interface` declarations merge automatically when declared more than once in the same scope:

```typescript
interface Window {
  title: string;
}

interface Window {
  height: number;
}

// Window is now { title: string; height: number } — both declarations merged
```

This is a deliberate feature (it's how you augment a third-party library's types, or the global `Window` object, without modifying its source) but it means "redeclaring a type" isn't the safety net in TypeScript that it is in Java — the compiler won't catch an accidental duplicate `interface` the way it would catch an accidental duplicate Java `class`.

The practical response:

- Use `type` instead of `interface` for ordinary application types where you don't want merging — a duplicate `type` *is* an error:

```typescript
type Window = { title: string }; // fine
type Window = { height: number }; // error: duplicate identifier 'Window'
```

- Reserve `interface` (and its merging behavior) specifically for cases where you're intentionally extending a shape you don't own — augmenting a library's or the DOM's types — not as a default habit carried over from "interfaces are for contracts."

---

## 11. “How do I even use this — it's plain JavaScript with no types?”

Java has no equivalent problem: a `.jar` either has the class files or it doesn't, and reflection can always inspect them — a dependency's type information is baked into the bytecode, never optional metadata that might be missing:

```java
import com.example.somelibrary.SomeClass;

SomeClass instance = new SomeClass(); // full type info is always available —
                                        // it's part of the compiled .class file
```

Most of the JavaScript ecosystem predates TypeScript, and plenty of packages still ship no type information at all:

```typescript
import { doSomething } from "some-old-library";
// implicitly typed 'any' — TypeScript has no idea what this library exports
```

The practical response:

- Check for community-maintained types first — `npm install --save-dev @types/some-old-library` (the DefinitelyTyped project) covers a large share of untyped packages.
- When no types exist anywhere, write a minimal ambient declaration file (`some-old-library.d.ts`) describing just the surface you actually call — you don't need to type the whole library, only your usage of it:

```typescript
declare module "some-old-library" {
  export function doSomething(input: string): number;
}
```

- Enable `noImplicitAny` (bundled into `strict`) so a genuinely untyped import is a build error you have to explicitly acknowledge, rather than a silent `any` that quietly disables checking for everything downstream of it.

---

## 12. “Why did `as` let something obviously wrong through?”

Java's cast is checked at runtime — casting an `Object` to the wrong type throws `ClassCastException` immediately, at the cast site:

```java
Object input = "not a number";
Integer value = (Integer) input; // throws ClassCastException immediately, at this line
```

TypeScript's `as` is a compile-time-only assertion with no runtime check at all:

```typescript
const input: unknown = "not a number";
const value = input as number; // compiles — 'as' doesn't verify anything
console.log(value + 1); // "not a number1" at runtime — no exception, just a wrong result
```

Unlike Java, there's no exception to catch here — the "cast" doesn't do anything at runtime except tell the compiler to stop checking. If the assumption is wrong, the mistake surfaces later, wherever the resulting value happens to be used unsafely, not at the cast.

The practical response:

- Treat `as` as an unchecked, unverified promise to the compiler, not a runtime-safe cast — use it only when you have external certainty the type is correct (a value you constructed yourself, or one already validated by a schema library upstream).
- Avoid the "double assertion" escape hatch (`value as unknown as Target`), which exists specifically to bypass TypeScript's own sanity check that the source and target types overlap at all — if you need it, that's a strong signal the value should be validated at runtime instead of asserted.
- For parsing external input, use a schema library that performs a real runtime check and only then narrows the type (`zod`'s `.parse()`), rather than casting a `JSON.parse()` result and hoping.
- When `as` is being used merely to check a value against a shape — not to override the compiler — reach for `satisfies` (TypeScript 4.9+) instead. It verifies the expression matches the target type without widening the inferred type to it, so you keep the precise literal types and still get the error when the shape is wrong:

```typescript
const config = {
  port: 8080,
  host: "localhost",
} satisfies Record<string, string | number>;

config.port.toFixed();  // still `number`, not `string | number` — inference preserved
config.hsot;            // error: typo caught, unlike `as Record<...>` which would hide it
```

There's no Java equivalent — a Java type annotation always widens the variable to the declared type, so "check it against this contract but keep the specific type" isn't expressible.

---

## 13. “Decorators look like annotations, but they aren't quite there yet”

Java's `@Annotation` is a single, stable mechanism, backed by reflection, that's been part of the language since Java 5:

```java
@Component
class UserService { }

// reflection can discover every @Component on the classpath at startup
for (Class<?> c : classpathScanner.findAnnotatedClasses(Component.class)) { ... }
```

TypeScript decorators have had a longer, messier road: the original `experimentalDecorators` flag (still the default many frameworks — Angular, older NestJS — target) predates the TC39 standard and behaves differently from the standardized decorators TypeScript 5.0 added support for. As of TypeScript 5.9, decorator metadata (`Symbol.metadata`) reached stability, letting frameworks read structured metadata off a decorated class without the `reflect-metadata` polyfill many of them previously required — but plenty of existing code and tutorials still assume that older, `experimentalDecorators` + `reflect-metadata` setup.

```typescript
// legacy (experimentalDecorators: true) — parameter decorators, reflect-metadata
@Injectable()
class UserService {
  constructor(@Inject(DATABASE) private db: Database) {}
}

// standard decorators (TypeScript 5.0+, no experimentalDecorators flag) —
// no parameter decorators at all; the shape of what you can decorate differs
```

The practical response:

- Check which decorator mode a framework actually targets before copying an example — `experimentalDecorators: true` and standard decorators are not interchangeable, and mixing patterns from both produces confusing compiler errors.
- If you're writing decorators for a new codebase with no legacy framework constraint, prefer the standard (TC39) decorators — they're the direction the language is moving, and as of 5.9 no longer require pulling in `reflect-metadata` for basic metadata needs.
- Don't expect the reflective introspection Java's annotations give you for free (scanning a classpath for every `@Component`) — TypeScript decorators run at class-definition time in the module that defines the class; there's no built-in equivalent to classpath scanning.

---

## 14. “Why did this generic function accept a callback with the wrong parameter type?”

Java's generics are invariant by default — a `List<Dog>` is not assignable to a `List<Animal>` parameter, precisely to prevent the kind of unsoundness that would let you insert a `Cat` into what's really a `List<Dog>` through an aliased reference:

```java
List<Dog> dogs = new ArrayList<>();
List<Animal> animals = dogs; // compile error: incompatible types
                               // (blocked specifically to stop animals.add(new Cat())
                               // from corrupting what's really a List<Dog>)
```

TypeScript's structural type system, combined with a historical compatibility decision, allows a real soundness gap specifically for method-shorthand function parameters:

```typescript
interface Handler {
  handle(event: MouseEvent): void; // method shorthand syntax
}

function register(h: Handler) { ... }

const clickOnlyHandler: { handle(event: { x: number; y: number }): void } = {
  handle(event) { console.log(event.x, event.y); },
};

register(clickOnlyHandler); // compiles, even though `handle` doesn't really accept a MouseEvent
```

Method-shorthand signatures (`handle(event: T): void`) are checked *bivariantly* for backward compatibility with pre-TypeScript JavaScript patterns, even under `strictFunctionTypes` — only function-*valued properties* (`handle: (event: T) => void`) get the strict, contravariant check `strictFunctionTypes` is supposed to enforce everywhere.

The practical response:

- Prefer the function-property syntax (`handle: (event: T) => void`) over method-shorthand syntax (`handle(event: T): void`) in interfaces and type aliases where parameter safety matters — it opts into the strict check the shorthand form silently skips.
- Don't assume `strictFunctionTypes` alone closes every parameter-variance hole — it's a real improvement over the pre-2.6 default, but this specific gap is a documented, permanent compatibility exception, not a bug that'll eventually be fixed.

---

## 15. “Why did overriding this method silently do nothing?”

Java's `@Override` annotation is optional, but when present, `javac` actually checks it — annotate a method `@Override` with a signature that doesn't match any base class member, and compilation fails:

```java
class Animal {
    String speak() { return "..."; }
}

class Dog extends Animal {
    @Override
    String Speak() { return "Woof"; } // compile error: method does not override a method from its superclass
}
```

TypeScript classes dispatch through JavaScript's prototype chain, so a correctly-named method always overrides — but by default, nothing checks that a method you *meant* to override actually matches a base member's name at all:

```typescript
class Animal {
  speak(): string { return "..."; }
}

class Dog extends Animal {
  Speak(): string { return "Woof"; } // typo'd casing — compiles fine
}

const d: Animal = new Dog();
console.log(d.speak()); // "..." — Dog.Speak() is a completely unrelated new method;
                          // Animal.speak() is still what actually runs
```

`Dog` now has two unrelated methods — `speak` (inherited) and `Speak` (new) — and the type checker never flags it, because from its point of view you simply added a new method to `Dog`, which is entirely legal.

The practical response:

- Add the `override` keyword (TypeScript 4.3+) to every method meant to override a base member:

```typescript
class Dog extends Animal {
  override Speak(): string { return "Woof"; }
  // error: this member cannot have an 'override' modifier because it is not
  // declared in the base class 'Animal'
}
```

- `override` alone only checks methods you've already remembered to mark — it doesn't require marking them in the first place. Enable `noImplicitOverride: true` in `tsconfig.json` to make the compiler reject an *unmarked* method that happens to match a base signature, closing the gap from the other direction.
- Together, `override` plus `noImplicitOverride` get you back to roughly what Java's `@Override` gives you — but unlike Java, where `@Override` has been standard practice since Java 5, both are comparatively recent (2021) and easy to find turned off in an existing `tsconfig.json`.

---

## 16. “Where's my multiple inheritance substitute?”

Both languages restrict a class to a single base class. Java's answer for sharing behavior across otherwise-unrelated hierarchies is a `default` method on an interface:

```java
interface Flyer {
    default String fly() { return "Flying"; }
}

class Bird extends Animal implements Flyer { }
```

TypeScript interfaces are purely structural and erased (§1, §3) — they can't carry a method body at all, so there's no TypeScript equivalent to a Java `default` method. The idiomatic answer to "share behavior across unrelated classes" is instead the mixin pattern: a function that takes a base class and returns a new class extending it:

```typescript
type Constructor<T = {}> = new (...args: any[]) => T;

function Flyer<TBase extends Constructor>(Base: TBase) {
  return class extends Base {
    fly(): string { return "Flying"; }
  };
}

class Bird extends Flyer(Animal) { }

const b = new Bird();
b.fly(); // "Flying" — composed in via the mixin function, not a second base class
```

This still produces a single, ordinary prototype chain under the hood — `Bird` genuinely has only one runtime base class, the anonymous class the `Flyer` function generated — so it doesn't violate JavaScript's single-inheritance model the way "multiple inheritance" sounds like it should. It's function composition wearing class-declaration syntax.

The practical response:

- Reach for a mixin when several unrelated classes need the same piece of behavior and a shared interface with no implementation isn't enough — the same motivation that would lead a Java developer to a `default` interface method.
- Keep mixins small and single-purpose, the same discipline this repository's `java-to-golang` guide recommends for struct embedding — a class built from several stacked mixins can become as hard to trace as a deep, ad hoc inheritance chain.
- Don't reach for a mixin just to avoid writing an interface — if the shared piece is a contract with no shared implementation, a plain `interface` (§1) is simpler and doesn't need the `Constructor` generic boilerplate a mixin requires.

---

## 17. “Where's my try-with-resources?”

Java has had `try`-with-resources since Java 7, and it's the default way to guarantee cleanup — any `AutoCloseable` is closed at the end of the block, in reverse declaration order, even if the body throws:

```java
try (var conn = dataSource.getConnection();
     var stmt = conn.prepareStatement(SQL)) {
    return stmt.executeQuery();
} // stmt.close() then conn.close(), guaranteed — and if close() itself throws,
  // that exception is attached to the primary one via getSuppressed()
```

TypeScript 5.2 added `using` and `await using` declarations, which do the same job with a different shape — disposal is tied to the enclosing scope, so there's no extra block to introduce:

```typescript
function query(sql: string) {
  using conn = pool.getConnection();  // conn[Symbol.dispose]() runs when scope exits
  using stmt = conn.prepare(sql);
  return stmt.execute();
} // stmt disposed, then conn — reverse order, even if execute() throws
```

A "resource" is any object with a `[Symbol.dispose]()` method. `await using` looks for `[Symbol.asyncDispose]()` and awaits it — something Java has no equivalent to at all, since `AutoCloseable.close()` has no asynchronous counterpart.

This isn't a TypeScript-only construct: the TC39 explicit resource management proposal reached Stage 4 in May 2025 and is part of ES2026. Runtime support is newer than the TypeScript support, though — V8 shipped it in Chrome 134, and Node gained native support in Node 24 — the runtime [java-to-nodejs](../java-to-nodejs/) recommends. On Node 22, which that guide treats as its floor, the `Symbol.dispose` symbols exist but the syntax isn't parsed natively; TypeScript downlevels it for you.

The practical response:

- Reach for `using` anywhere you'd write `try`-with-resources in Java — file handles, connections, spans, locks, test fixtures — and `await using` where cleanup is itself asynchronous.
- Set `lib` appropriately (the `esnext.disposable` / ES2026 lib types) so `Disposable` and `AsyncDisposable` resolve; a missing lib entry is the usual cause of "`Symbol.dispose` does not exist."
- Use `DisposableStack`/`AsyncDisposableStack` when disposal is conditional or accumulated at runtime rather than one-resource-per-declaration — the case Java handles by nesting or by manual `finally`.
- Expect `SuppressedError` where Java gives you `getSuppressed()`: if a disposal throws while another error is already propagating, JavaScript wraps both rather than attaching one to the other.

---

## 18. “Where are my bounded wildcards?”

Java's generics are invariant, and use-site wildcards are how you opt out of that, one parameter at a time — the PECS rule every Java developer internalizes:

```java
// producer-extends: read Numbers out of a List of any subtype
double sum(List<? extends Number> values) { ... }

// consumer-super: write Integers into any List that can hold them
void fill(List<? super Integer> target) { ... }

sum(new ArrayList<Integer>());  // fine
fill(new ArrayList<Number>());  // fine
```

TypeScript has no use-site variance at all — no `? extends`, no `? super`. Variance is structural, inferred from how a type parameter is actually used, and assignability just follows from that:

```typescript
function sum(values: readonly number[]): number { ... }

declare const dogs: Dog[];
const animals: Animal[] = dogs; // allowed — arrays are covariant, and unsoundly so
animals.push(new Cat());         // compiles; `dogs` now contains a Cat
```

A Java developer knows this exact hazard from Java's own covariant arrays — but Java at least throws `ArrayStoreException` at runtime. TypeScript has no runtime check whatsoever, so the `Cat` simply lands in the `Dog[]` and surfaces as a confusing failure somewhere else entirely.

TypeScript 4.7 added declaration-site variance annotations, but they don't do what wildcards do:

```typescript
type Getter<out T> = () => T;           // covariant
type Setter<in T> = (value: T) => void; // contravariant
interface State<in out T> { ... }       // invariant
```

These *assert* the variance the checker would otherwise infer structurally — the compiler verifies the annotation is right, and can then skip some structural comparison work. They're an accuracy-and-compile-speed tool aimed at library authors, not a mechanism for a caller to widen or narrow what a function accepts. There is no call-site equivalent of `List<? extends Number>`.

The practical response:

- Use `readonly T[]` (or `ReadonlyArray<T>`) wherever Java would use `List<? extends T>` — read-only-ness is how TypeScript expresses "producer," and it closes the covariance hole for that direction.
- Don't reach for `in`/`out` to fix an assignability error — they describe variance, they don't change what's assignable at a call site. Check whether `readonly`, or the function-property syntax from §14, is what you actually needed.
- Treat every implicit widening like `const animals: Animal[] = dogs` as write-unsafe, and don't rely on a runtime error to catch a bad write the way Java's `ArrayStoreException` would.
- §14 covers the related soundness gap in method-shorthand parameters — same theme, different mechanism.

---

## What TypeScript Gets Right

The friction above isn't the whole story — several things are genuinely nicer once a Java engineer settles in:

- Structural typing (§1) makes it possible to type existing JavaScript incrementally, or to satisfy an interface with a plain object literal, without first designing a class hierarchy the way Java's nominal typing effectively requires.
- Discriminated unions plus exhaustiveness checking (§9) give you sealed-type-and-pattern-matching ergonomics with less ceremony than Java's `sealed`/`permits` — no need to declare the closed set of implementors up front at the base type.
- Utility types (`Partial<T>`, `Pick<T, K>`, `Record<K, V>`, `ReturnType<F>`) let you derive a new type mechanically from an existing one — a category of type-level transformation Java's generics don't attempt, and that would otherwise need a code generator.
- Type inference is pervasive enough that most local variables and return types need no annotation at all, without losing the safety of a fully-typed signature at function boundaries.
- The language service (`tsserver`) is fast and IDE-native, so refactors like "rename this field" or "find every caller" work as well as a Java IDE's, even across a codebase with only partial type coverage.

---

## Recommended TypeScript Baseline for Java Engineers

This complements — and assumes — the runtime/tooling baseline in [java-to-nodejs](../java-to-nodejs/); the settings below are specifically about getting the most out of the type checker itself.

```text
TypeScript:   7.0+ (the Go-native compiler; still installed as `typescript`, still run as `tsc`)
Strictness:   strict, plus noUncheckedIndexedAccess and exactOptionalPropertyTypes
Erasability:  erasableSyntaxOnly — no enum, namespace, or parameter properties
Enums:        string literal unions + `as const`, not `enum`, for new code
Resources:    `using` / `await using` for anything with cleanup (§17)
Decorators:   standard (TC39) decorators for new code; match legacy frameworks explicitly
Validation:   zod (or equivalent) at any boundary where `unknown` data enters
Linting:      @typescript-eslint (no-explicit-any, no-unnecessary-condition)
Declarations: tsc --declaration for anything published as a package
CI:           tsc --noEmit + eslint + tests, same as java-to-nodejs's baseline
```

A note on the version, since the jump is larger than the numbering suggests: TypeScript 6.0 (March 2026) was the last JavaScript-based release, and TypeScript 7.0 (July 2026) is the same compiler ported to Go — roughly 10x faster, with type-checking logic deliberately kept structurally identical, so it enforces the same semantics rather than a new dialect. The package name and the `tsc` binary didn't change (`tsgo` was only the preview-era command name). Two migration details matter: everything 6.0 merely deprecated is a hard error in 7.0, and the programmatic compiler API isn't stable yet — tooling that imports from `typescript` and calls into the compiler directly may not work until 7.1. This guide's examples all assume 5.8+ and hold on any of these releases.

Example `tsconfig.json` additions on top of the java-to-nodejs baseline:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "useUnknownInCatchVariables": true,
    "erasableSyntaxOnly": true
  }
}
```

- `noUncheckedIndexedAccess` makes `arr[i]` return `T | undefined` instead of `T` — without it, an out-of-bounds array or missing map key silently types as present, the closest TypeScript gets to Java's `ArrayIndexOutOfBoundsException`, except without the exception.
- `exactOptionalPropertyTypes` stops `{ retries?: number }` from silently accepting an explicit `undefined` as equivalent to omitting the field — the two are different in Java (there's no field-omission concept) and this flag makes TypeScript treat them as different too.
- `useUnknownInCatchVariables` types a caught exception as `unknown` rather than `any` (§4) — since JavaScript's `throw` accepts anything, a `catch` block genuinely doesn't know what it received.
- `erasableSyntaxOnly` (TypeScript 5.8+) rejects `enum`, `namespace`, and constructor parameter properties (§6) — the only TypeScript constructs that emit runtime code. Turning it on early costs nothing on a codebase already avoiding them, and keeps the door open to running TypeScript directly on a type-stripping runtime.

## Bottom Line

The Java developer's complaint is usually valid:

> TypeScript's type system checks a real, useful subset of what Java's does — but it's structural rather than nominal, erased entirely at runtime, and was designed to fit an existing dynamic language rather than the other way around.

That produces gaps with no Java equivalent — unrelated types that are freely interchangeable (§1), `readonly` and `as` that promise nothing at runtime (§8, §12), a decorator story still mid-migration (§13) — alongside capabilities Java's type system doesn't have at all, like mechanical type transformations and lightweight sealed-union exhaustiveness (§9).

For serious application development, treat those gaps the same way the other guides in this repository treat their language's gaps — with a deliberate policy, not by assuming the compiler covers what it visibly doesn't:

1. Default to `unknown`, never `any`, at any boundary where external data enters — and validate it at runtime.
2. Prefer literal unions and discriminated unions over `enum` and ad hoc object shapes — they're closed, and they support real exhaustiveness checking.
3. Treat `readonly` and `as` as compiler-only signals, not runtime guarantees — reach for `Object.freeze()` and runtime validation when you actually need the guarantee.
4. Know which decorator mode a piece of code targets before copying a pattern into it.
5. Enable `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes` — neither is in `strict` by default, and both close gaps a Java developer will otherwise trip over immediately.

With that policy in place, TypeScript's type system gets close to covering what a Java developer expects a type system to guarantee — closer than any other language in this repository — while still requiring the same honesty about where "the compiler didn't complain" stops meaning "this is safe."
