# What a Java Developer Will Complain About When Moving to C#

C# is the closest language to Java in this repository — both are statically typed, class-based, garbage-collected, and compile to a bytecode-like intermediate representation run by a managed VM. The friction here isn't "where are my guardrails" the way it is moving to Python, Go, or Node.js — most of Java's guardrails are still there, sometimes stronger. The friction is that C# made a series of different design choices at almost every fork in the road, and a Java developer's muscle memory is wrong just often enough to be dangerous.

## 1. “Why did assigning this copy the whole thing?”

Java has exactly one non-primitive kind of type: the reference type. Every `class` instance lives on the heap, and assignment copies the reference, never the object:

```java
Point p1 = new Point(1, 2);
Point p2 = p1;
p2.x = 99;
System.out.println(p1.x); // 99 — p1 and p2 point at the same object
```

C# has a second, equally first-class kind of type: the `struct`, a value type. Assigning a struct copies its contents, not a reference to them:

```csharp
struct Point { public int X, Y; }

Point p1 = new Point { X = 1, Y = 2 };
Point p2 = p1;
p2.X = 99;
Console.WriteLine(p1.X); // 1 — p2 is an independent copy
```

The surprise cuts both ways: a Java developer who assumes every object behaves like `p1`/`p2` above will be confused when a `struct` doesn't share mutations, and — just as often — surprised when passing a large `struct` into a method or a collection turns out to silently copy it repeatedly, with a real performance cost that has no Java equivalent to worry about.

The practical response:

- Default to `class` for anything with identity or non-trivial size — the C# community's own guidance is to reserve `struct` for small (rule of thumb: 16 bytes or less), immutable, value-like data (a coordinate, a money amount, a short-lived DTO).
- Never define a mutable `struct` — the copy semantics above combine with mutability to produce bugs that are genuinely hard to spot in review (mutating `list[i].X` on a `List<Point>` of mutable structs is a classic C# gotcha with no direct Java analog).
- When you do want structs, ask whether it should implement `IEquatable<T>` (§2) — structs get member-wise equality more cheaply than classes, but only if you opt in correctly.

---

## 2. “Why does `==` sometimes mean value equality, sometimes reference — and why can I overload the operator itself?”

Java's `==` rule is simple and fixed: reference equality for objects, value equality for primitives, and there's no way to change what `==` does for a given type — you override `.equals()` instead, and `==` is always the same operator.

C# lets a type overload `==` itself, and different built-in categories default differently:

```csharp
class Point { public int X, Y; }
var p1 = new Point { X = 1, Y = 2 };
var p2 = new Point { X = 1, Y = 2 };
Console.WriteLine(p1 == p2); // False — class default is reference equality, like Java

record Point2(int X, int Y);
var r1 = new Point2(1, 2);
var r2 = new Point2(1, 2);
Console.WriteLine(r1 == r2); // True — record generates value equality, including == , automatically

struct Point3 { public int X, Y; }
var s1 = new Point3 { X = 1, Y = 2 };
var s2 = new Point3 { X = 1, Y = 2 };
Console.WriteLine(s1.Equals(s2)); // True (via boxing + reflection, unless IEquatable<T> is implemented)
Console.WriteLine(s1 == s2); // compile error — struct doesn't get == unless you define it yourself
```

Three different defaults for three kinds of types, and `record` in particular means "does `==` compare values here?" is no longer answerable from the language alone — it depends on the specific declaration.

The practical response:

- Prefer `record`/`record struct` for data-carrying types where value equality is the whole point — it's less to get wrong than hand-rolling `Equals`/`GetHashCode`/`==`/`!=` on a `class`, all of which need to stay consistent with each other.
- For a `struct`, implement `IEquatable<T>` explicitly if you need `Equals` to be fast and correct — the default boxes the argument and uses reflection, which is both slow and occasionally wrong for fields that are themselves reference types.
- When reviewing someone else's `==` overload on a `class`, don't assume it means what Java's `==` would mean, or even what `.Equals()` on the same type means — check that the author kept them consistent.

---

## 3. “Why is the compiler suddenly warning me about `null`?”

Java has no compiler-level null-safety by default — `@Nullable`/`@NonNull` annotations exist, but they're a third-party or IDE-level convention, not enforced by `javac`.

C#'s nullable reference types feature (opt-in per-project, on by default in new project templates since .NET 6) makes the compiler track nullability the way it already tracked it for value types:

```csharp
#nullable enable

string GetName(User user) => user.Name; // ok: User and its Name are non-null by declaration

string? maybeName = user?.Name;
string upper = maybeName.ToUpper(); // warning: Dereference of a possibly null reference
```

The syntax overlaps with Java's `?`/`Optional` mental model but works completely differently underneath — this is compiler flow analysis on ordinary reference types, not a wrapper type, and it's a warning, not a compile error, unless the project is configured to treat warnings as errors.

The practical response:

- Enable `<Nullable>enable</Nullable>` in every project — it's off by default in projects created before .NET 6 or migrated from older templates, and turning it on mid-project surfaces a wave of warnings worth working through deliberately rather than suppressing.
- Treat nullable warnings as build failures in CI (`<WarningsAsErrors>CS8602;CS8600</WarningsAsErrors>` at minimum, or all of them) — as a warning-only feature, it's easy for a team to silently stop reading them.
- Remember this is a static-analysis convention, not a runtime guarantee — a `string` annotated as non-nullable can still be `null` at runtime if it arrives via reflection, deserialization, or a library compiled without nullable annotations. It catches mistakes at the call sites the analyzer can see, not every possible path.

---

## 4. “Why did this LINQ query run twice — with different results each time?”

Java's `Stream` can only be consumed once — calling a terminal operation on an already-consumed stream throws `IllegalStateException`, which is annoying but at least loud.

C#'s LINQ query expressions build an `IEnumerable<T>` that's lazily evaluated and can be re-enumerated freely — with no warning that re-enumerating means re-running the whole query, side effects included:

```csharp
var expensiveQuery = orders.Where(o => {
    Log(o); // a "harmless" side effect
    return o.Total > 100;
});

var count = expensiveQuery.Count();      // runs the query, logs every order
var list = expensiveQuery.ToList();      // runs the query AGAIN, logs every order again
```

If the underlying source can change between enumerations (a database query, a mutable collection), the two enumerations aren't even guaranteed to see the same data — the opposite failure mode from Java's Streams, which fail loudly on reuse instead of silently reproducing work.

The practical response:

- Materialize a query with `.ToList()` or `.ToArray()` as soon as you need to use the result more than once — don't pass a bare `IEnumerable<T>` around expecting it to behave like a realized collection.
- Be wary of a side effect inside a `Select`/`Where` lambda specifically because it might run more than once, or (with `IQueryable<T>` against a database) not run in-process at all.
- When a method's return type is `IEnumerable<T>`, treat that signature as "this may be lazy and may re-run" — the same caution Java developers already apply to a `Stream`-returning method, just without the exception to catch a misuse.

---

## 5. “Where are checked exceptions — and why doesn't C# have any at all?”

This isn't an oversight — C#'s designers looked at Java's checked exceptions specifically and decided against them.  In practice, checked exceptions pushed Java developers toward `catch (Exception e) {}` or blanket `throws Exception` declarations, which defeated the mechanism's own purpose.

```csharp
// no `throws` clause exists in C# at all
public User GetUser(string id)
{
    // may throw UserNotFoundException, or may not — nothing in the signature says so
}
```

Every C# exception is unchecked, full stop — there's no middle ground to opt into the way Java has one.

The practical response is the same discipline the other guides in this repository already push for languages with no compiler-enforced error signaling:

- Document what a public method can throw in its XML doc comment (`/// <exception cref="UserNotFoundException">...</exception>`) — analyzers and IDEs surface this at the call site, which is the closest C# gets to a `throws` clause.
- Define specific exception types for conditions callers need to branch on, rather than throwing or catching the base `Exception`.
- Lint for empty or overly broad catch blocks — Roslyn analyzers (`CA1031: Do not catch general exception types`) catch exactly this pattern.

---

## 6. “Why did `await`-ing this deadlock, in code that clearly isn't broken?”

Java's `CompletableFuture` and (as of JDK 21) virtual threads don't have this failure mode — blocking on a future from another thread doesn't require anything about which thread you started on.

C#'s `async`/`await` — which async/await in JavaScript and Python later borrowed the shape of — captures a `SynchronizationContext` in certain application types (classic ASP.NET, WPF, WinForms) and resumes the continuation on it. Blocking synchronously on an async call from that same context deadlocks:

```csharp
// inside an ASP.NET (classic, non-Core) controller action or a UI event handler
public string GetUserName(string id)
{
    var task = _client.GetUserAsync(id);
    return task.Result; // deadlock: this thread is blocked waiting for the continuation,
                         // but the continuation needs this exact thread's context to run
}
```

The calling thread blocks on `.Result` while holding the only context the continuation needs in order to complete — neither side can proceed. This exact deadlock doesn't happen in ASP.NET Core (no `SynchronizationContext` by default) or in a console app, which is precisely what makes it so confusing: the same code pattern is fine in one host and hangs forever in another.

The practical response:

- Use `await` all the way up the call stack — never call `.Result` or `.Wait()` on a `Task` from code that might run under a captured context.
- Call `.ConfigureAwait(false)` on awaited calls inside library code that doesn't need to resume on the original context — it tells the continuation it's safe to resume on any thread pool thread, sidestepping the deadlock entirely. (ASP.NET Core largely removes the need for this, but library code that might be consumed by a classic ASP.NET or desktop app should still do it.)
- If you inherit code with `.Result`/`.Wait()` calls, treat every one as a potential deadlock waiting for the right caller, not just a style nit.

---

## 7. “How is this method appearing on a type I don't own?”

Java has no way to add a method to an existing type without subclassing it or wrapping it — `String` gets exactly the methods `java.lang.String` declares, forever.

C# extension methods let any static class add what looks like an instance method to any type, including ones you don't own and can't modify:

```csharp
public static class StringExtensions
{
    public static bool IsPalindrome(this string s) =>
        s.SequenceEqual(s.Reverse());
}

"racecar".IsPalindrome(); // calls the extension method above, as if String had this method
```

This is how LINQ itself is implemented — `.Where()`, `.Select()`, `.OrderBy()` are all extension methods on `IEnumerable<T>`, not members of the interface. A Java developer reading unfamiliar C# code will reasonably wonder where a method like this is actually declared, since "go to the type's own source" won't find it.

The practical response:

- Use your IDE's "go to definition" rather than assuming a method must be declared on the type itself — extension methods resolve based on which `using` directives are in scope, which is itself a source of surprise (the same call can resolve to a different extension method, or none at all, depending on what's imported).
- Reach for extension methods yourself for utility functions over a type you don't control, instead of a static helper class with an awkward `StringUtils.isPalindrome(s)` call shape — it's idiomatic C#, not a hack.
- Keep extension methods discoverable — put them in a namespace/class name that signals what they extend (`StringExtensions`, not a generic `Utils`), since there's no `implements`-style declaration pointing back at the type they augment.
- C# 14 (.NET 10) generalizes this into *extension members*: an `extension` block can declare extension properties, operators, and static members on the receiver type, not just instance methods. It groups everything extending one type under a single declaration instead of repeating `this T` on each method. Available only from C# 14 — well above this guide's C# 10 floor, so don't expect it in existing code.

---

## 8. “Wait — generics actually work at runtime here?”

Java generics are erased at compile time — `List<String>` and `List<Integer>` are the exact same class at runtime, `new T()` is illegal, `list instanceof List<String>` doesn't compile, and arrays of generic types need unchecked casts.

C# generics are reified — the CLR knows the real type argument at runtime, for real:

```csharp
List<int> ints = new List<int>();
List<string> strings = new List<string>();
Console.WriteLine(ints.GetType() == strings.GetType()); // False — genuinely different runtime types

void PrintType<T>(T value) => Console.WriteLine(typeof(T)); // typeof(T) just works

T CreateDefault<T>() where T : new() => new T(); // legal, with the `new()` constraint
```

Value-type generics (`List<int>`) also avoid boxing entirely at the CLR level, which has no Java equivalent — `List<Integer>` in Java always boxes every element.

Reification also enables something Java has no answer to at all. Java can't express "a generic method that sums any numeric type" — `Number` has no `+`, so you end up with an overload per primitive. C# 11 (.NET 7) added static abstract interface members, and the BCL's `INumber<TSelf>` builds generic math on top of them:

```csharp
static T Sum<T>(IEnumerable<T> values) where T : INumber<T>
{
    T total = T.Zero;              // a static member, resolved through the type parameter
    foreach (var v in values) total += v;
    return total;
}

Sum(new[] { 1, 2, 3 });        // int
Sum(new[] { 1.5, 2.5 });       // double — one implementation, no overloads
```

The practical response: this is one of the areas where C# is simply less surprising than Java, not more — code that "should obviously work" (checking a generic type at runtime, creating a new instance of a type parameter, storing value types in a generic collection without boxing overhead) generally does. The one adjustment: `where T : new()` only allows a parameterless constructor call — for anything more, pass a factory `Func<T>` instead, since C# still won't let you call an arbitrary constructor generically. And note that generic math needs C# 11 / .NET 7 or later — above this guide's C# 10 floor, so it won't be available in every codebase you touch.

---

## 9. “Why does this class definition keep spilling across files?”

A Java class is defined in exactly one place — always.

C# allows a `class` (or `struct`, `interface`, `record`) to be declared `partial`, splitting its definition across multiple files:

```csharp
// UserService.cs
public partial class UserService
{
    public User GetUser(string id) { ... }
}

// UserService.Generated.cs — often written by a source generator, not a human
public partial class UserService
{
    private readonly ILogger _logger;
}
```

This is deliberate and heavily used by tooling — WinForms/WPF designers put UI layout code in a `.Designer.cs` partial so it doesn't collide with hand-written code in the main file, and source generators (Entity Framework, `System.Text.Json` source generation, `[GeneratedRegex]`) emit a partial counterpart to a class you wrote by hand.

The practical response: when a member seems to be missing from a class's file, check for a `partial` modifier on the class declaration before assuming it doesn't exist — search the whole project for other `partial class ClassName` declarations, since your IDE's "go to definition" may only show you one of several files. Don't split a class across files by hand for organizational reasons alone; reserve `partial` for generated code and framework-mandated splits, where it earns the indirection.

---

## 10. “Which `using` is this — an import, or a resource cleanup?”

Java uses `import` for bringing a type into scope and `try (var resource = ...)` for deterministic cleanup — two different keywords for two different jobs.

C# uses the same keyword, `using`, for both, with meaning determined entirely by position:

```csharp
using System.Text.Json; // "using" as import — brings a namespace into scope

using var connection = new SqlConnection(connectionString); // "using" as a disposal
// declaration — connection.Dispose() runs automatically at the end of the enclosing scope
```

The second form is C#'s answer to Java's try-with-resources (`IDisposable`/`Dispose()` playing the role of `AutoCloseable`/`close()`), but it reads, at a glance, exactly like the import statement above it.

The practical response: read a `using` statement's position, not just the keyword — at the top of a file (or inside a `namespace` block) it's an import; attached to a variable declaration, it's a disposal scope. The `using var x = ...;` form (C# 8+) ties disposal to the enclosing block automatically, without the extra braces Java's try-with-resources requires — functionally equivalent, just without the visual boundary Java gives you for free.

---

## 11. “Why is this visible to code I don't think should see it?”

Java's visibility model has four levels: `private`, package-private (the default), `protected`, and `public` — all scoped to the class or its package.

C# has those four Java-familiar levels (roughly: `private`, `internal`, `protected`, `public`) plus two combinations Java has no equivalent for, both scoped to the assembly (a compiled `.dll`/`.exe`, roughly analogous to a Java module, not a package):

```csharp
internal class Repository { }               // visible anywhere in this assembly, invisible outside it
protected internal void Save() { }           // visible to subclasses OR anywhere in this assembly
private protected void Validate() { }        // visible to subclasses, but ONLY within this assembly
```

`internal` is the default for a top-level `class` if no modifier is given (unlike Java, where the default is package-private) — so an unmarked C# class is more broadly visible within its assembly than an unmarked Java class is within its package, if the assembly spans more code than a single Java package typically would.

The practical response: don't map C# visibility onto Java's package-private mental model directly — a C# assembly is usually a whole project/library, not a single directory, so `internal` grants visibility across far more code than Java's default does. Use `internal` deliberately for genuine library-internals that consumers of a NuGet package shouldn't see, and reach for `private protected` (rather than plain `protected`) when a protected member should be usable by subclasses within your own codebase but not by subclasses in a downstream consumer's assembly.

---

## 12. “Why is this boilerplate suddenly two lines?”

Java's getter/setter convention is compiler-invisible — `private String name; public String getName() { return name; } public void setName(String n) { name = n; }` is three lines of hand-written (or IDE-generated) boilerplate for one logical field, and every caller uses `getName()`/`setName(...)` rather than field-like syntax.

C# properties are a first-class language construct with field-like call-site syntax:

```csharp
public class User
{
    public string Name { get; set; }              // full property, auto-backed
    public string Id { get; init; }                // settable only at construction (C# 9+)
    public int Age { get; private set; }            // publicly readable, privately settable
}

user.Name = "Miguel"; // looks like field access, actually calls the setter
```

Records compress this further, generating properties, a constructor, equality (§2), and a `ToString()` from a single line:

```csharp
public record User(string Id, string Name, int Age);

var u1 = new User("1", "Miguel", 30);
var u2 = u1 with { Age = 31 }; // non-destructive copy with one field changed
```

Object initializers make this construction style pleasant but historically unsafe — nothing forced a caller to set anything, which is why Java developers reach for a builder. `required` (C# 11, .NET 7) closes that gap at the compiler level:

```csharp
public class User
{
    public required string Id { get; init; }   // caller MUST set this
    public required string Name { get; init; }
    public int Age { get; init; }               // optional
}

var u = new User { Id = "1" }; // compile error: required member 'Name' must be set
```

The practical response: default to auto-properties (`{ get; set; }`) over manually-backed ones unless you actually need logic in the getter or setter — and reach for `record` (C# 9+) / `record struct` (C# 10+) for DTOs and value objects the way you'd reach for a Java `record` (JDK 16+), except with the additional `with`-expression non-destructive update Java's own `record` doesn't provide. Where you'd write a Java builder purely to make construction safe, `required` members (C# 11+) usually replace it outright.

One place Java got there first: multi-line string boilerplate. Java text blocks (`"""`) finalized in JDK 15 (JEP 378); C#'s raw string literals — same `"""` delimiter, same goal of embedding JSON/SQL/XML without escaping — arrived in C# 11 (.NET 7), two years later. If you're on the C# 10 floor, you're still writing verbatim strings (`@"..."`) with doubled quotes.

---

## 13. “Can I actually overload an operator here?”

Java forbids operator overloading entirely — `+` means concatenation for `String` and arithmetic for numbers, by fiat from the language, and no user type can participate.

C# allows a type to define what `+`, `==`, `<`, and other operators mean for it, plus custom indexers:

```csharp
public struct Money
{
    public decimal Amount { get; }
    public static Money operator +(Money a, Money b) => new Money(a.Amount + b.Amount);
}

var total = new Money(10) + new Money(5); // calls the overloaded operator

public class Matrix
{
    public double this[int row, int col] { get => _data[row, col]; set => _data[row, col] = value; }
}

matrix[1, 2] = 5.0; // custom indexer, not a built-in language feature
```

The practical response: operator overloading is powerful and easy to abuse — reserve it for types where the operator has an unambiguous, domain-standard meaning (arithmetic on a `Money` or `Vector` type), not as a shorthand for an arbitrary method just because the syntax is available. When reading unfamiliar C#, remember that `+`, `==`, and `[...]` are not guaranteed to mean what they mean on a built-in type — check whether the type in question defines its own.

---

## 14. “Why doesn't the namespace match the folder?”

Java enforces a strict correspondence: a class in package `com.example.orders` must live in a directory path ending `com/example/orders/`, and the compiler checks it.

C# namespaces are conventionally aligned with folder structure by tooling, but the language itself doesn't enforce it — a `namespace` declaration is just a name, unrelated to the file's location on disk:

```csharp
// file: Services/Orders/OrderProcessor.cs
namespace MyCompany.Billing; // file-scoped namespace (C# 10+) — no braces, no requirement to match the path

public class OrderProcessor { }
```

A file can declare any namespace regardless of where it sits in the project, and a project can have multiple unrelated top-level namespaces mixed across the same folder tree without anything failing to compile.

The practical response: don't rely on the compiler to catch a namespace/folder mismatch the way `javac` would catch a package/directory mismatch — most IDEs offer an analyzer warning for this (`IDE0130` in the default .NET SDK analyzers) but it's advisory. Turn it on and enforce it in CI if the team wants Java-like consistency; otherwise treat "does the namespace match the folder" as a code-review convention, not a guarantee.

---

## 15. “Why did calling this through the base-class reference get the base's behavior, not the override?”

This is the single biggest inheritance-semantics gap between the two languages, and it's silent — no compile error, often not even a warning if you're not paying attention.

Java makes every instance method virtual by default — dynamic dispatch is the language's default behavior, and you opt *out* of it explicitly with `final`:

```java
class Animal {
    public String speak() { return "..."; }
}

class Dog extends Animal {
    @Override
    public String speak() { return "Woof"; }
}

Animal a = new Dog();
System.out.println(a.speak()); // "Woof" — always dispatches on the runtime type
```

C# inverts the default: a method is *not* virtual unless the base class explicitly says so, and a derived class must explicitly write `override` to participate in dynamic dispatch — anything else *hides* the base member instead of overriding it:

```csharp
class Animal
{
    public string Speak() => "..."; // not virtual — an ordinary, non-polymorphic method
}

class Dog : Animal
{
    public new string Speak() => "Woof"; // hides Animal.Speak; does NOT override it
}

Animal a = new Dog();
Console.WriteLine(a.Speak()); // "..." — dispatches on the STATIC (compile-time) type, Animal

Dog d = new Dog();
Console.WriteLine(d.Speak()); // "Woof" — static type is Dog here, so this resolves differently
```

Two calls on the exact same object return two different results, purely because of the compile-time type of the reference used to call it — something that can't happen in Java, where `a.speak()` and `d.speak()` above would both return `"Woof"` no matter which variable's declared type you called through.

Dropping the `new` keyword on `Dog.Speak()` doesn't fix this — it produces the identical hiding behavior, plus compiler warning CS0108 ("hides inherited member; use the `new` keyword if hiding was intended"). Neither `new` nor its absence makes `Dog.Speak()` participate in polymorphic dispatch — only declaring `Animal.Speak()` as `virtual` and `Dog.Speak()` as `override` does:

```csharp
class Animal
{
    public virtual string Speak() => "...";
}

class Dog : Animal
{
    public override string Speak() => "Woof";
}

Animal a = new Dog();
Console.WriteLine(a.Speak()); // "Woof" — now genuinely virtual, matching Java's behavior above
```

The practical response:

- Mark a base method `virtual` as a deliberate decision that subclasses may change its behavior polymorphically — treat an unmarked method as sealed by default, the opposite assumption from Java's "everything is virtual unless `final`."
- Never ignore the CS0108 warning — it's the compiler telling you what you wrote looks like an override but isn't one. Add `new` only when hiding is genuinely intended (rare), and `override` (on a `virtual` or `abstract` base member) when polymorphism is the goal.
- The same rule applies to properties and indexers, not just methods — an unmarked property hides rather than overrides under identical logic.
- Java's `final` on an overriding method — sealing it against further overriding down the hierarchy — is `sealed override` in C#. A plain method can't be `sealed` on its own; only an `override` can be, since a non-`virtual` method is already unoverridable by default.
- Covariant return types (an override returning a more derived type than the base declares) work in C# too, but only since C# 9 (.NET 5, 2020) — Java has had this since JDK 5 (2004). If you're reading code written for an older C# target, don't assume it's available.

---

## 16. “Why did the same object answer differently depending on which interface I called it through?”

Java lets a class implement multiple interfaces, but every interface method it implements shares exactly one implementation — if two interfaces declare a method with the same signature, one method body in the class satisfies both. When both interfaces supply conflicting `default` bodies for that signature, Java forces the implementing class to resolve the conflict explicitly, but the resolution is still a single implementation, not two:

```java
interface Flyer { default String move() { return "Flying"; } }
interface Swimmer { default String move() { return "Swimming"; } }

class Duck implements Flyer, Swimmer {
    @Override
    public String move() { // required: must resolve to one implementation
        return Flyer.super.move();
    }
}
```

C# allows *explicit interface implementation*, where a class provides a genuinely distinct implementation per interface for a same-named member — and which one runs depends on the static type of the reference used to call it, not on the object's runtime type alone:

```csharp
interface IFlyer { string Move(); }
interface ISwimmer { string Move(); }

class Duck : IFlyer, ISwimmer
{
    string IFlyer.Move() => "Flying";     // explicit implementation, reachable only via IFlyer
    string ISwimmer.Move() => "Swimming"; // explicit implementation, reachable only via ISwimmer
}

Duck duck = new Duck();
// duck.Move(); // doesn't compile — Move() isn't a public member of Duck itself

IFlyer flyer = duck;
ISwimmer swimmer = duck;
Console.WriteLine(flyer.Move());   // "Flying"
Console.WriteLine(swimmer.Move()); // "Swimming" — same object, different answer
```

Java has no equivalent to this at all — there's no syntax for a class to give two different bodies for what's nominally "the same method," keyed by which interface reference calls it.

The practical response:

- Use explicit interface implementation to resolve a genuine name collision between two unrelated interfaces a type must implement, or to deliberately de-emphasize a member (a common pattern for `IDisposable.Dispose()`, kept out of a type's "primary" public surface while still satisfying the interface contract).
- When reading unfamiliar C#, don't assume `someObject.Member` is the only way to reach a given interface member — check whether the type also carries an explicit implementation reachable only by first casting or assigning to the interface type.
- Don't reach for this as a general technique for giving one type two unrelated behaviors depending on caller perspective — it's a narrow, deliberate escape hatch. If a type genuinely needs two different behaviors, that's usually a sign it's doing two jobs and should be split instead.

---

## 17. “Why did this hot loop start generating garbage the moment I passed the struct through an `object`?”

Every value in Java that isn't a primitive is already a reference type living on the heap, so passing anything into a method that accepts `Object` never allocates beyond ordinary autoboxing of a primitive — and the JIT can often avoid even that for small, cached `Integer` values:

```java
void log(Object value) { System.out.println(value); }
log(42); // autoboxes int -> Integer; values -128..127 are cached, so often no fresh allocation
```

C# structs (§1) are true value types — they live inline, on the stack or embedded within their containing object, specifically to avoid heap allocation and GC involvement. But the moment a struct is converted to `object`, to an interface type, or into a non-generic collection, it's *boxed*: a full heap-allocated copy is made, and that copy is on the GC's books exactly like any class instance.

```csharp
void Log(object value) => Console.WriteLine(value);

struct Point { public int X, Y; }

Point p = new Point { X = 1, Y = 2 };
Log(p); // boxes p — a new heap allocation on every call, garbage the moment Log returns
```

Nothing about the call site `Log(p)` looks like an allocation to someone reading it with Java's uniform-heap model in mind — the type that exists specifically to stay off the heap ends up there anyway, generating Gen0 pressure on every call.

.NET's GC is generational (Gen0/Gen1/Gen2, plus a separate Large Object Heap for allocations ≥ 85,000 bytes) — conceptually closer to the JVM than what most other languages in this repository use — but the defaults and knobs differ.

The practical response:

- Prefer generic APIs and collections (`List<T>`, `IEquatable<T>`, generic constraints) over passing structs through `object` or non-generic types — reified generics (§8) mean a `List<Point>` stores `Point`s unboxed and inline, unlike Java's `List<Integer>`, which always boxes.
- An interface call site can box silently — implementing the generic form of an interface (`IEquatable<T>` rather than relying on `Equals(object)`, `IComparable<T>` rather than `IComparable`) avoids it; profile with `dotnet-trace` or Visual Studio's allocation profiler if a hot path takes a struct as an interface parameter.
- Server-side apps (ASP.NET Core's default) run Server GC, background and concurrent by default, tuned for throughput; latency-sensitive desktop-style apps default to Workstation GC. Set `<ServerGarbageCollection>`/`<ConcurrentGarbageCollection>` in the project file deliberately rather than assuming the default matches the workload — the concept maps to choosing a JVM collector, but the knobs and defaults aren't the same ones.
- Objects ≥ 85,000 bytes land on the Large Object Heap, which historically isn't compacted by default (`GCSettings.LargeObjectHeapCompactionMode` opts in per-collection) — large arrays or buffers in a long-running service can fragment that heap in a way the JVM's compacting collectors wouldn't.

---

## 18. “Why doesn't this primary constructor act like a record?”

A Java `record` declaration generates a great deal from one line — accessors, value equality, `hashCode`, and a `toString`:

```java
public record Person(String name, int age) { }

var p1 = new Person("Miguel", 30);
var p2 = new Person("Miguel", 30);
p1.equals(p2);   // true — value equality, generated
p1.name();       // "Miguel" — accessor, generated
p1.toString();   // "Person[name=Miguel, age=30]", generated
```

C# primary constructors (C# 12, .NET 8) let a `class` or `struct` declare constructor parameters in the header, which reads almost identically — and generates almost none of that:

```csharp
public class Person(string name, int age)
{
    public string Describe() => $"{name} is {age}"; // params stay in scope for the whole class body
}

var p1 = new Person("Miguel", 30);
var p2 = new Person("Miguel", 30);
Console.WriteLine(p1 == p2);       // False — reference equality; no == is generated
// Console.WriteLine(p1.Name);     // doesn't compile — no property is generated
Console.WriteLine(p1.ToString());  // "Person" — the default, not a value-based ToString
```

The parameters become captured private fields, not properties: nothing is exposed publicly, no equality or `ToString` is generated, and there's no deconstruction. Worse for a Java reader's assumptions, those captured parameters are ordinary mutable state in scope for the entire class body — any method can reassign `name`, and it stays reassigned. C#'s actual analog of a Java `record` is a C# `record` (§12), which does generate init-only properties, value equality, `ToString`, and deconstruction.

The practical response:

- Read `class Foo(...)` as "constructor parameters that never go out of scope," not as a record declaration. If you want Java-record semantics, write `record Foo(...)` (C# 9+) or `record struct Foo(...)` (C# 10+).
- If callers need to read a primary constructor parameter, declare the property explicitly (`public string Name { get; } = name;`) — the parameter alone gives you nothing public.
- Treat a captured parameter as mutable unless you deliberately assign it to a `readonly` field; a Java developer expecting record-component immutability won't get it here.
- The same syntax on a `struct` (C# 12) likewise generates no value equality beyond what §1 and §2 describe for structs generally — the header shape is not a signal about equality in either case.

---

## 19. “Where's my zero-copy slice?”

Slicing in Java allocates. `substring` has produced a fresh `String` with its own backing array since JDK 7u6, and `split` allocates both an array and a `String` per element:

```java
String csv = "alpha,beta,gamma";
String[] parts = csv.split(",");     // allocates an array plus a String per element
String first = csv.substring(0, 5);  // allocates a new String with its own char[]
```

Java's zero-copy tools are separate APIs you opt into — `ByteBuffer`, or the Foreign Function & Memory API finalized in JDK 22 (JEP 454) — not something the language threads through ordinary code.

C# has `Span<T>`/`ReadOnlySpan<T>`: stack-only views over contiguous memory that slice without allocating at all.

```csharp
ReadOnlySpan<char> csv = "alpha,beta,gamma";
ReadOnlySpan<char> first = csv.Slice(0, 5); // no allocation — a view over the same memory
Span<byte> buffer = stackalloc byte[256];   // stack-allocated; the GC never sees it
```

These are `ref struct` types, and that's the direct counterweight to §17: where boxing pushes a value type onto the GC's books, a `Span<T>` guarantees it never gets there. The cost is a set of restrictions that feel arbitrary until you see the reason — a `ref struct` can't be a field of a `class` or non-ref struct, can't be boxed, and can't be captured in a lambda or local function. All of it follows from one rule: these types hold managed interior pointers, and the GC only tracks those on the stack. C# 13 (.NET 9) softened two of the sharpest edges — a `ref struct` can now implement an interface (though still never be converted to one, since that would box), the `allows ref struct` anti-constraint lets one be used as a generic type argument, and they're permitted in async methods and iterators as long as they don't sit in the same block as an `await` or a `yield return`.

The practical response:

- Reach for `ReadOnlySpan<char>` over `Substring` in parsing and formatting hot paths — not allocating is the entire point, and most of the BCL's parsing surface (`int.Parse`, `Utf8Formatter`, etc.) has span overloads.
- Use `stackalloc` for small, short-lived buffers only — stay well under about 1KB, since blowing the stack is a process-ending failure, not an exception you can catch.
- Expect the restrictions to bite in ordinary code: no stashing a `Span<T>` in a field, no closing over it, no holding it across an `await`. That's not an arbitrary limitation — it's what makes the stack-only guarantee sound.
- Don't spread it everywhere. `Span<T>` earns its complexity in parsers, serializers, and buffer-shuffling code; in ordinary business logic it buys nothing and costs readability.

---

## 20. “Why did this arithmetic throw instead of wrapping?”

Java's rule for integer overflow is absolute and stated in the language spec: it wraps, silently, always. The opt-in to loud failure is a library call:

```java
int max = Integer.MAX_VALUE;
int wrapped = max + 1;                // -2147483648 — silently wraps, by language rule
int loud = Math.addExact(max, 1);     // ArithmeticException — the explicit opt-in
```

C# wraps by default too — but "by default" is doing real work in that sentence, because the default is a project-level setting:

```csharp
int max = int.MaxValue;
int wrapped = max + 1;         // wraps — unless the project sets CheckForOverflowUnderflow

checked
{
    int boom = max + 1;        // OverflowException
}

int constant = int.MaxValue + 1; // compile ERROR — constant expressions are always checked
```

The same expression can wrap in one project and throw in another, decided by `<CheckForOverflowUnderflow>` in the `.csproj` rather than by anything visible at the call site. Constant expressions are checked at compile time regardless of that setting. And since C# 11, a user-defined operator can supply a separate `checked` variant, so a custom type's `+` may genuinely do two different things depending on the context it's called from.

The practical response:

- Don't carry Java's "overflow always wraps silently" assumption into an unfamiliar C# codebase — check `<CheckForOverflowUnderflow>` before reasoning about arithmetic.
- Turn checking on for financial or domain code where a silent wrap is a correctness bug, and use an explicit `unchecked { }` block for the narrow cases where wrapping is intended (hash computation being the classic one).
- Note that `decimal` throws on overflow regardless of the surrounding context, and floating-point types never throw at all — they produce `Infinity`/`NaN`, matching Java. The configurable behavior applies to the integral types.
- If you define arithmetic operators on your own type (§13), consider supplying the `checked operator` variant (C# 11+) so the type stays consistent in both contexts rather than silently wrapping inside a `checked` block.

---

## What C# Gets Right

The friction above isn't the whole story — several things are genuinely nicer once a Java engineer settles in:

- Reified generics (§8) eliminate an entire category of Java's erasure workarounds — no unchecked casts, no `Class<T>` token-passing to recover runtime type information, no boxing for `List<int>`.
- LINQ's query and method syntax make transformation pipelines more concise than the equivalent Java `Stream` chain, especially for joins and grouping, which read closer to SQL than Java's `Collectors` API does.
- `record` types (§2, §12) collapse what would be a Java `record` plus a hand-written `with`-style copy-and-modify helper into one declaration.
- Nullable reference types (§3), once enabled and enforced, give the compiler a real (if unsound at the edges) opinion about `null` — closer to what Java developers wish `@Nullable` actually did.
- A single official toolchain (`dotnet` CLI, NuGet, MSBuild) covers build, test, package management, and publishing without the tool-selection fatigue this repository's Python and Node.js guides describe.

---

## Recommended C# Baseline for Java Engineers

For production backend and service development:

```text
.NET:         .NET 10 (current LTS)
Language:     C# 14, <LangVersion>latest</LangVersion>
Nullability:  <Nullable>enable</Nullable> in every project, warnings as errors in CI
Formatting:   dotnet format / .editorconfig
Analyzers:    built-in .NET SDK analyzers + Roslyn analyzers (CA rules) enabled
Tests:        xUnit (or NUnit/MSTest), dotnet test
Packages:     NuGet + PackageReference in .csproj, committed packages.lock.json
CI:           dotnet format --verify-no-changes + build (warnings as errors) + dotnet test
Containers:   dotnet publish with a pinned SDK/runtime base image
```

Example `.csproj` settings:

```xml
<PropertyGroup>
  <TargetFramework>net10.0</TargetFramework>
  <Nullable>enable</Nullable>
  <ImplicitUsings>enable</ImplicitUsings>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  <AnalysisLevel>latest</AnalysisLevel>
  <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
</PropertyGroup>
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

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: "10.0.x"

      - run: dotnet format --verify-no-changes
      - run: dotnet build --warnaserror
      - run: dotnet test
```

## Bottom Line

The Java developer's complaint here is different in character from every other guide in this repository:

> C# didn't strip away Java's guardrails — it rebuilt most of them differently, and added a few Java deliberately doesn't have.

Value types with copy semantics (§1), an overloadable `==` (§2), lazily re-run LINQ queries (§4), a real async deadlock hazard (§6), and reified generics that quietly remove problems Java developers have learned to route around (§8) are all cases where "this looks like Java" is exactly what makes the difference dangerous — the two languages are close enough that a wrong assumption feels confident right up until it's wrong.

For serious backend and platform engineering, the practical response is less about adding discipline C# lacks (mostly, it doesn't) and more about not assuming Java's specific defaults:

1. Reserve `struct` for small, immutable, value-like data — never a mutable one.
2. Enable nullable reference types project-wide, and treat the warnings as build failures.
3. Materialize a LINQ query with `.ToList()`/`.ToArray()` before using it more than once.
4. Never call `.Result`/`.Wait()` on a `Task` in code that might run under a captured `SynchronizationContext`.
5. Don't assume `==`, a class's default visibility, or a file's namespace mean what they'd mean in the equivalent Java code — check the specific declaration.

With that adjustment, C# gives a Java engineer nearly everything they already know how to reach for, plus a handful of capabilities — reified generics chief among them — Java still doesn't have.
