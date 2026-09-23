# Core Java & OOP

## The four pillars (know these cold, with a real example each)

- **Encapsulation** — bundling state + behavior, hiding internals behind accessors. Interview angle: "why private fields with getters/setters instead of public fields?" → controlled access, validation on set, can change internal representation without breaking callers.
- **Abstraction** — exposing *what* without *how*. Interfaces and abstract classes are the mechanism, not the concept itself.
- **Inheritance** — `extends` for classes (single inheritance), `implements` for interfaces (multiple). Favor composition over inheritance when the relationship isn't a true "is-a".
- **Polymorphism** — compile-time (overloading) vs runtime (overriding via dynamic dispatch/vtable lookup).

A quick concrete example that works for all four (useful when an interviewer asks you to explain OOP to a non-expert): a pen. *Encapsulation* — it writes when pressed to paper, you don't need to know the ink mechanism. *Abstraction* — the blueprint defines what "a pen" does. *Inheritance* — ball pens and gel pens share the base pen behavior. *Polymorphism* — both write, but differently.

### Overloading vs overriding

| | Overloading (compile-time) | Overriding (runtime) |
|---|---|---|
| Resolution | Compile time (static binding) | Runtime (dynamic binding) |
| Signature | Must differ (param count/types) | Must be identical |
| Return type | Can differ freely | Must be same or covariant |
| Inheritance | Not required — same class | Required — subclass overrides superclass |
| Effect | Adds a variant of the behavior | Replaces the behavior |
| Access modifier | Unrestricted | Cannot be more restrictive than the parent's |

## Object relationships: association, aggregation, composition

Frequently asked as a trio — the distinction is about *lifecycle ownership*.

- **Association** — two objects simply know about each other and can interact. No ownership. A `Teacher` and a `Student` reference each other; either can exist without the other.
- **Aggregation** — a "has-a" whole/part relationship where the part can outlive the whole. A `Department` has `Professor`s; delete the department and the professors still exist. Usually modeled as a reference to an object created elsewhere.
- **Composition** — aggregation *plus* lifecycle ownership: the part cannot exist without the whole, and the whole is responsible for creating/destroying it. A `House` has `Room`s; destroy the house and the rooms go with it. Usually the whole constructs the part internally.

The interview one-liner: association is "uses-a", aggregation is "has-a (shared)", composition is "owns-a (exclusive)".

Related principle to raise if it fits: **favor composition over inheritance** — inheritance couples you to the parent's implementation and is fixed at compile time, whereas composition lets you swap behavior at runtime and avoids fragile-base-class problems.

## Interfaces vs abstract classes

| | Abstract class | Interface |
|---|---|---|
| State | Can hold instance fields | Only `static final` constants |
| Constructors | Yes | No |
| Multiple inheritance | No (single class extends) | Yes (implements many) |
| Default methods | Always concrete-able | `default` methods since Java 8 |
| When to use | Shared code + "is-a" hierarchy | Contract/capability, especially across unrelated classes |

Common follow-up: **since Java 8 interfaces can have `default` and `static` methods — so what's still different from an abstract class?** No instance state, no constructors, and a class can implement many interfaces but extend only one abstract class. Diamond conflicts on default methods are resolved by requiring the implementing class to override.

Java 9 added `private` interface methods too — helpers that `default` methods can share without exposing them to implementors.

Two things **not** to say here: that "all interface methods are implicitly abstract" (untrue since Java 8), or that interfaces are slower than abstract classes (obsolete JVM folklore — modern JIT optimizes both dispatch paths; the difference is negligible). Choose on design grounds: shared state and shared code → abstract class; a capability contract across unrelated types → interface.

**When to pick which, practically:**

- Abstract class — closely related classes share both code *and* fields, you need non-public members, or you want a constructor to enforce invariants.
- Interface — unrelated classes need the same capability, you want multiple inheritance of type, or you're defining a contract that implementations vary freely on (the Strategy pattern shape).

## Multiple inheritance

Java allows multiple inheritance of **type** (a class can `implement` many interfaces) but not of **state/implementation** (a class can `extend` only one class). This avoids the classic "diamond problem" — ambiguity about which parent's field/implementation to inherit.

Since Java 8, `default` methods reintroduce a limited diamond risk: if two implemented interfaces provide the same default method, the code won't compile until the implementing class explicitly overrides it (and it can delegate with `InterfaceName.super.method()`).

## Keywords worth being precise about

- **`static`** — belongs to the class, not an instance; accessible without instantiating. Static members are initialized when the JVM loads the class.
  - *Can a static method be overridden?* **No.** Overriding is runtime dispatch on an object; statics are bound at compile time by reference type. A subclass declaring the same static signature **hides** it, it doesn't override it. Private methods likewise can't be overridden (not visible to the subclass).
  - *Can a non-static variable be accessed from a static context?* **No** — statics exist before/without any instance, so there's no `this` to resolve the field against.
- **`final`** — on a class: cannot be subclassed. On a method: cannot be overridden. On a variable: cannot be reassigned (note: a `final` reference to a mutable object still allows mutating that object's contents — `final List` can still be `.add()`-ed to).
- **`transient`** — excludes a field from serialization; on deserialization it comes back as the type's default (`null`/`0`/`false`). Used for secrets, caches, and derived values you don't want persisted.
- **`volatile`** — visibility and reordering guarantees only, **no atomicity and no locking** (see the Concurrency file — this is a commonly-mangled one).

## Static vs dynamic binding

- **Static (early) binding** — resolved at compile time from the *reference type*. Applies to `static`, `private`, and `final` methods, all field access, and all method **overloading**.
- **Dynamic (late) binding** — resolved at runtime from the *actual object type*. Applies to overridden instance methods; this is what makes polymorphism work.

```java
Shape shape = new Rectangle();
shape.getArea();   // dynamic binding → Rectangle's getArea() runs
```

The classic trap combines the two: **fields are not polymorphic.** If both `Shape` and `Rectangle` declare a field `name`, then `shape.name` (with `Shape shape = new Rectangle()`) reads *Shape's* field, because field access uses static binding.

## Wrapper classes, autoboxing & unboxing

- Wrapper classes (`Integer`, `Long`, `Double`, `Boolean`, ...) box primitives as objects so they can be used where objects are required — generics (`List<Integer>`, since generics can't hold primitives), collections, and nullable values.
- **Autoboxing** — the compiler inserts `Integer.valueOf(int)` automatically; **unboxing** inserts `intValue()`.
- Two gotchas worth knowing:
  - **Integer caching.** `Integer.valueOf()` caches −128..127, so `Integer a = 127, b = 127; a == b` is `true`, but at `128` it's `false`. Always compare wrappers with `.equals()`, never `==`.
  - **Unboxing NPEs.** `Integer x = map.get(missingKey); int y = x;` throws `NullPointerException` — the implicit `x.intValue()` on a null reference. A very common production bug.

## Is Java pass-by-value or pass-by-reference?

**Always pass-by-value** — this is a favorite because the honest answer sounds counterintuitive.

- **Primitives:** a copy of the value is pushed onto the method's stack frame. Changes inside the method don't affect the caller.
- **Objects:** Java copies the *reference* (the pointer value) and passes the copy. So:
  - **Mutating** the object through that copied reference (`obj.setName("x")`) **does** affect the caller's object — both references point at the same heap object.
  - **Reassigning** the parameter (`obj = new Thing()`) does **not** affect the caller — you've only repointed the local copy.

That second bullet is the whole answer: Java passes references *by value*, which is not the same as pass-by-reference.

## equals(), hashCode(), toString()

- Contract: if `a.equals(b)` is true, `a.hashCode() == b.hashCode()` must also be true. Violating this breaks hash-based collections (`HashMap`, `HashSet`) silently — objects "disappear" because they're looked up in the wrong bucket.
- Reverse isn't required: equal hashCodes don't imply `equals()` is true (hash collisions are fine).
- Override both together, always. Use `Objects.equals()` / `Objects.hash()` to avoid null-handling bugs.
- `records` (Java 16+) generate `equals`/`hashCode`/`toString` automatically based on components — good to mention if asked about modern Java.

## Immutability & the String pool

- Immutable class recipe: `final` class, `final` fields, no setters, defensive copies of mutable fields in constructor/getter, don't leak `this` during construction.
- `String` is immutable — every "modification" (`concat`, `+`) creates a new object. That's why heavy string building in loops should use `StringBuilder` (mutable, not thread-safe) — `StringBuffer` is the synchronized/thread-safe equivalent, rarely needed now.
- String literals live in the **string constant pool** (part of heap since Java 7+, was PermGen before). `new String("x")` forces a new heap object outside the pool; `.intern()` pulls it back into the pool. `==` compares references — this is the classic "why does `==` sometimes work for strings and sometimes not" interview gotcha.
- Why is `String` immutable at all? Security (used for class names, file paths, network connections — mutation would be a hole), safe hashcode caching (used heavily as `HashMap` keys), thread-safety without synchronization, safe sharing via the pool.

## Exceptions

- **Checked** (extends `Exception`, not `RuntimeException`) — must be declared or caught, compiler-enforced (`IOException`, `SQLException`). **Unchecked** (extends `RuntimeException`) — not enforced (`NullPointerException`, `IllegalArgumentException`).
- `Error` (e.g. `OutOfMemoryError`, `StackOverflowError`) is a separate hierarchy — not meant to be caught/handled, signals JVM-level problems.
- try-with-resources: anything implementing `AutoCloseable` gets `close()` called automatically, even on exception, in reverse order of declaration. Prefer this over manual `finally` blocks.
- Common interview trap: what happens if you `return` in `try` **and** `finally`? The `finally` return wins — swallows the try's return value (and swallows exceptions too if not careful). Don't put `return` in `finally` in real code.
- Custom exceptions: extend `RuntimeException` for most application errors in Spring apps (avoids polluting method signatures with `throws`), map them to HTTP responses via `@ControllerAdvice` (see REST notes).

### throw vs throws

- `throw` — a statement, used *inside* a method body, throws one exception instance: `throw new IllegalArgumentException("...")`.
- `throws` — a clause in the *method signature*, declares which checked exceptions may propagate to the caller; can list several, comma-separated.

### final vs finally vs finalize

Asked constantly because the names collide, and they're completely unrelated:

- **`final`** — keyword. Non-subclassable class / non-overridable method / non-reassignable variable.
- **`finally`** — block. Runs after `try`/`catch` whether or not an exception was thrown; used for cleanup. (Skipped only on `System.exit()` or JVM crash.)
- **`finalize()`** — method on `Object`, historically called by the GC before reclaiming an object. **Deprecated since Java 9** and effectively never the right answer: there's no guarantee it ever runs, it delays collection, and it can resurrect objects. Use `try-with-resources`/`AutoCloseable`, or `java.lang.ref.Cleaner` for native resources.

After an exception is caught and handled, the exception object itself is just an ordinary unreferenced object — eligible for garbage collection like anything else.

## Generics

- Type erasure: generic type info is removed at compile time — `List<String>` and `List<Integer>` are the same class at runtime (`List`). This is why you can't do `new T[]` or `instanceof List<String>`.
- Bounded wildcards — **PECS**: *Producer Extends, Consumer Extends... no — Producer `extends`, Consumer `super`*. If you're reading (producing) `T`s from a structure, use `? extends T`; if you're writing (consuming), use `? super T`.
- `<T extends Comparable<T>>` — bounding a type parameter to require a capability.

## Functional interfaces & lambdas (Java 8+)

- A functional interface has exactly one abstract method (`@FunctionalInterface` is optional but self-documenting/enforced by compiler).
- Built-ins to know: `Function<T,R>`, `Supplier<T>`, `Consumer<T>`, `Predicate<T>`, `BiFunction<T,U,R>`, `Runnable`, `Callable<V>`.
- Lambdas capture *effectively final* local variables (can't reassign a captured variable after the lambda is created).
- Method references: `ClassName::staticMethod`, `instance::method`, `ClassName::instanceMethod`, `ClassName::new` (constructor reference).

## Stream API basics

- Streams are lazy — intermediate ops (`map`, `filter`, `sorted`) don't execute until a terminal op (`collect`, `forEach`, `reduce`, `count`) triggers the pipeline.
- Streams are single-use — consuming one twice throws `IllegalStateException`.
- Common ops to be fluent in: `stream().filter(...).map(...).collect(Collectors.toList())`, `Collectors.groupingBy(...)`, `Collectors.toMap(...)`, `Optional` chaining.
- Parallel streams (`.parallelStream()`) use the common `ForkJoinPool` — good for CPU-bound, side-effect-free work on large datasets; can backfire with I/O-bound tasks or when the pool is shared with other work (starvation).

## Commonly asked

- Why override `equals()` and `hashCode()` together, and what breaks if you don't?
- Why is `String` immutable, and what's the difference between `==` and `.equals()` for strings?
- Checked vs unchecked exceptions — when would you deliberately choose each for a custom exception?
- What is type erasure and what does it prevent you from doing with generics?
- What's the difference between `map` and `flatMap` in the Stream API?
- Explain PECS with a concrete method signature example.
- Is Java pass-by-value or pass-by-reference? Then: why can a method mutate the object you passed in, but not replace it?
- Can a static method be overridden? What actually happens if a subclass declares the same static signature?
- Difference between association, aggregation, and composition — give an example of each.
- What's the difference between `final`, `finally`, and `finalize()`?
- Why does `Integer a = 127, b = 127; a == b` return true but the same code with 128 return false?
- Given `Shape s = new Rectangle()`, why does an overridden *method* call resolve to Rectangle but a *field* access resolve to Shape?
