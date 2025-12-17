# Chapter 4: Traits

Go has interfaces. Rust has traits. They look similar, but the differences matter.

## Traits vs Go Interfaces

In Go, interfaces are satisfied implicitly. This took me quite a long time to really understand. Also, the name `interface` is really a misnomer. It should be called `contract.

```go
type Stringer interface {
    String() string
}

type Person struct {
    Name string
}

// Person implements Stringer without declaring it
func (p Person) String() string {
    return p.Name
}
```

In Rust, _traits_ are implemented explicitly:

```rust
trait Display {
    fn fmt(&self, f: &mut Formatter) -> Result;
}

struct Person {
    name: String,
}

// Explicit: "I am implementing Display for Person"
impl Display for Person {
    fn fmt(&self, f: &mut Formatter) -> Result {
        write!(f, "{}", self.name)
    }
}
```

### Why Explicit?

Go's implicit satisfaction is convenient but has quirks:

- You can accidentally implement an interface
- Changing an interface signature silently breaks implementors
- Hard to find all types implementing an interface

Rust's explicit implementation:

- Clear intent—you meant to implement this trait
- Compiler error if the trait changes
- Tools can easily find all implementations

### The impl Block

Methods in Rust go in `impl` blocks:

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

// Inherent methods (no trait)
impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }

    fn new(width: u32, height: u32) -> Rectangle {
        Rectangle { width, height }
    }
}

// Trait implementation
impl Display for Rectangle {
    fn fmt(&self, f: &mut Formatter) -> Result {
        write!(f, "{}x{}", self.width, self.height)
    }
}
```

Note: `new` isn't special syntax—it's just a convention for constructors.

### self, &self, &mut self

Methods take `self` in different forms:

```rust
impl MyType {
    fn consume(self) {}        // Takes ownership, value is moved
    fn borrow(&self) {}        // Immutable borrow
    fn mutate(&mut self) {}    // Mutable borrow
}
```

Go comparison:

| Rust        | Go              | When to use               |
| ----------- | --------------- | ------------------------- |
| `self`      | (no equivalent) | Method consumes the value |
| `&self`     | `func (t T)`    | Read-only access          |
| `&mut self` | `func (t *T)`   | Need to modify            |

In Go, you choose between value and pointer receivers. In Rust, ownership rules make the choice clearer.

## Defining Traits

```rust
trait Summary {
    fn summarize(&self) -> String;

    // Default implementation
    fn preview(&self) -> String {
        format!("Read more: {}", self.summarize())
    }
}

struct Article {
    title: String,
    content: String,
}

impl Summary for Article {
    fn summarize(&self) -> String {
        self.title.clone()
    }
    // preview() uses the default implementation
}
```

Default implementations can call other trait methods—even ones without defaults.

## Generics with Trait Bounds

Go 1.18 added generics. Rust has had them from the start, and they work with traits.

### Basic Generics

```rust
fn largest<T>(list: &[T]) -> &T {
    // ... but wait, how do we compare items?
}
```

This won't compile. Rust doesn't know if `T` can be compared. You need a _trait bound_:

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

`T: PartialOrd` means "T must implement the PartialOrd trait" (which enables `>` comparison).

### Go Comparison

```go
// Go
func Largest[T constraints.Ordered](list []T) T {
    largest := list[0]
    for _, item := range list {
        if item > largest {
            largest = item
        }
    }
    return largest
}
```

Same idea: constraining what the type parameter can be.

### Multiple Bounds

```rust
fn print_and_clone<T: Display + Clone>(item: T) {
    println!("{}", item);
    let copy = item.clone();
}
```

`T` must implement both `Display` and `Clone`.

### where Clauses

When bounds get complex, use `where`:

```rust
fn complex_function<T, U>(t: T, u: U)
where
    T: Display + Clone,
    U: Debug + Default,
{
    // ...
}
```

Cleaner than cramming it all in the angle brackets.

### Trait Bounds in Struct Definitions

```rust
struct Wrapper<T: Display> {
    value: T,
}
```

Now `Wrapper` can only hold types that implement `Display`.

## Static vs Dynamic Dispatch

Here's where Rust and Go diverge sharply.

### Go: Always Dynamic Dispatch

```go
type Writer interface {
    Write([]byte) (int, error)
}

func writeData(w Writer, data []byte) {
    w.Write(data)  // Runtime lookup: which Write() to call?
}
```

Go interfaces use runtime dispatch. There's a vtable lookup to find the actual method.

### Rust: Static by Default (Monomorphization)

```rust
fn write_data<W: Write>(w: &mut W, data: &[u8]) {
    w.write(data);  // Compiler knows the exact type at compile time
}
```

When you call `write_data::<File>()`, Rust generates a specialized version for `File`. When you call `write_data::<TcpStream>()`, it generates another version for `TcpStream`.

This is _monomorphization_—one function in source, multiple specialized functions in the binary.

**Advantages:**

- Zero runtime overhead
- Enables inlining
- Faster

**Disadvantages:**

- Larger binary (one copy per type)
- Longer compile times

### Dynamic Dispatch with dyn

Sometimes you need runtime dispatch:

```rust
fn write_data(w: &mut dyn Write, data: &[u8]) {
    w.write(data);  // Runtime lookup, like Go
}
```

The `dyn Write` is a _trait object_. It's a "fat pointer": a pointer to data plus a pointer to a vtable.

**Go comparison:** A Go interface value is also two pointers (data + type/vtable), but Go automatically boxes the data (usually on the heap). In Rust, the "fatness" is in the pointer itself—the data stays where it is. With generics, there's no fat pointer at all; with `dyn`, there is.

### When to Use Each

| Situation                 | Use                          | Why                           |
| ------------------------- | ---------------------------- | ----------------------------- |
| Performance-critical code | Generics (static)            | Zero overhead                 |
| Heterogeneous collections | `dyn Trait`                  | Need to store different types |
| Public API flexibility    | Either, but `dyn` is simpler | Generics expose type params   |
| Small hot loops           | Generics                     | Inlining possible             |

### Box<dyn Trait>

To own a trait object (not just borrow it):

```rust
// Can hold any type implementing Animal
let animals: Vec<Box<dyn Animal>> = vec![
    Box::new(Dog { name: "Rex".to_string() }),
    Box::new(Cat { name: "Whiskers".to_string() }),
];

for animal in &animals {
    animal.speak();  // Dynamic dispatch
}
```

This is the closest to Go's interface slices:

```go
animals := []Animal{Dog{Name: "Rex"}, Cat{Name: "Whiskers"}}
```

## Common Traits You'll Use

Rust's standard library defines traits you'll encounter constantly.

### Clone and Copy

```rust
// Clone: explicit duplication
let s1 = String::from("hello");
let s2 = s1.clone();  // Explicit copy

// Copy: implicit duplication (stack-only types)
let x = 5;
let y = x;  // Implicitly copied, x still valid
```

- `Clone`: "I can make a duplicate of myself" (explicit `.clone()`)
- `Copy`: "I'm cheap to copy implicitly" (subset of Clone, no heap data)

### Debug and Display

```rust
#[derive(Debug)]      // Auto-implement Debug
struct Point {
    x: i32,
    y: i32,
}

let p = Point { x: 1, y: 2 };
println!("{:?}", p);  // Debug format: Point { x: 1, y: 2 }
println!("{p:?}");    // Same thing, shorter syntax
```

- `Debug`: programmer-facing output, usually derived
- `Display`: user-facing output, must implement manually

Go comparison: `Debug` is like Go's default `%v`, `Display` is like implementing `String()`.

### Default

```rust
#[derive(Default)]
struct Config {
    timeout: u32,      // Will be 0
    retries: u32,      // Will be 0
    verbose: bool,     // Will be false
}

let config = Config::default();
```

Like Go's zero values, but explicit and customizable.

### From and Into

Type conversions:

```rust
let s: String = String::from("hello");  // &str -> String
let s: String = "hello".into();         // Same thing

// Implement From, get Into for free
impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Fahrenheit {
        Fahrenheit(c.0 * 9.0 / 5.0 + 32.0)
    }
}

let f: Fahrenheit = celsius.into();
```

### PartialEq and Eq

```rust
#[derive(PartialEq, Eq)]
struct Point {
    x: i32,
    y: i32,
}

let p1 = Point { x: 1, y: 2 };
let p2 = Point { x: 1, y: 2 };
assert!(p1 == p2);  // Uses PartialEq
```

- `PartialEq`: enables `==` and `!=`
- `Eq`: marker that equality is reflexive (NaN != NaN, so floats are only PartialEq)

### PartialOrd and Ord

```rust
#[derive(PartialEq, Eq, PartialOrd, Ord)]
struct Score(u32);

let scores = vec![Score(85), Score(92), Score(78)];
let max = scores.iter().max();  // Uses Ord
```

- `PartialOrd`: enables `<`, `>`, `<=`, `>=`
- `Ord`: total ordering (can always compare)

### Derive: Auto-Implementing Traits

You've seen `#[derive(...)]`. It generates implementations automatically:

```rust
#[derive(Debug, Clone, PartialEq, Eq, Default)]
struct User {
    id: u64,
    name: String,
    active: bool,
}
```

This generates sensible implementations based on the struct's fields. Without derive, you'd write pages of boilerplate.

Common derivable traits:

- `Debug`, `Clone`, `Copy` (if all fields are Copy)
- `PartialEq`, `Eq`, `PartialOrd`, `Ord`
- `Default`, `Hash`

## Associated Types

Traits can have associated types—type placeholders that implementors define:

```rust
trait Iterator {
    type Item;  // Associated type

    fn next(&mut self) -> Option<Self::Item>;
}

impl Iterator for Counter {
    type Item = u32;  // Counter yields u32s

    fn next(&mut self) -> Option<Self::Item> {
        // ...
    }
}
```

Why not generics? Compare:

```rust
// With associated type
fn process(iter: impl Iterator<Item = u32>) {}

// If Iterator used generics instead
fn process<T: Iterator<u32>>(iter: T) {}  // Every use needs the type param
```

Associated types are cleaner when there's only one logical type per implementation.

## Summary

| Concept            | Go                    | Rust                                 |
| ------------------ | --------------------- | ------------------------------------ |
| Interface/trait    | Implicit satisfaction | Explicit `impl Trait for Type`       |
| Method receiver    | `(t T)` or `(t *T)`   | `self`, `&self`, `&mut self`         |
| Dispatch           | Always runtime        | Static (generics) or dynamic (`dyn`) |
| Type constraints   | `[T constraints.X]`   | `<T: Trait>` or `where T: Trait`     |
| Auto-implement     | (doesn't exist)       | `#[derive(...)]`                     |
| Hetero collections | `[]Interface`         | `Vec<Box<dyn Trait>>`                |

Traits are more powerful than Go interfaces: default methods, associated types, static dispatch, derivable implementations. The tradeoff is more explicit ceremony.

---

Next: Smart pointers—when `&T` isn't enough.
