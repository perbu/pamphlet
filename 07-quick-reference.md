# Chapter 7: Quick Reference

Cheat sheets, gotchas, and advice for when you're stuck.

## Go → Rust Translation Table

### Syntax

| Go                        | Rust                                          |
| ------------------------- | --------------------------------------------- |
| `var x int = 5`           | `let x: i32 = 5;`                             |
| `x := 5`                  | `let x = 5;`                                  |
| `const X = 5`             | `const X: i32 = 5;`                           |
| `var x int` (zero value)  | Not allowed—must initialize                   |
| `func foo(x int) int`     | `fn foo(x: i32) -> i32`                       |
| `func (t T) method()`     | `impl T { fn method(&self) }`                 |
| `if x > 0 { }`            | `if x > 0 { }` (same)                         |
| `for i := 0; i < 10; i++` | `for i in 0..10`                              |
| `for i, v := range slice` | `for (i, v) in slice.iter().enumerate()`      |
| `for _, v := range slice` | `for v in &slice`                             |
| `switch x { case 1: }`    | `match x { 1 => { } }`                        |
| `x.(Type)`                | `match x { }` or downcast                     |
| `go func() { }()`         | `thread::spawn(\|\| { })`                     |
| `defer cleanup()`         | Automatic via `Drop`                          |
| `make([]T, n)`            | `vec![default; n]` or `Vec::with_capacity(n)` |
| `make(map[K]V)`           | `HashMap::new()`                              |
| `make(chan T)`            | `mpsc::channel()`                             |
| `ch <- v`                 | `tx.send(v)`                                  |
| `v := <-ch`               | `rx.recv()`                                   |
| `close(ch)`               | `drop(tx)`                                    |

### Types

| Go                    | Rust                                      |
| --------------------- | ----------------------------------------- |
| `int`, `int64`        | `i32`, `i64` (explicit size)              |
| `uint`, `uint64`      | `u32`, `u64`                              |
| `float64`             | `f64`                                     |
| `string`              | `String` (owned) or `&str` (borrowed)     |
| `[]T`                 | `Vec<T>` (owned) or `&[T]` (slice)        |
| `[N]T`                | `[T; N]`                                  |
| `map[K]V`             | `HashMap<K, V>`                           |
| `*T`                  | `&T`, `&mut T`, `Box<T>` (depends on use) |
| `interface{}` / `any` | `dyn Any` or generics                     |
| `nil`                 | `None` (in `Option<T>`)                   |
| `error`               | `Result<T, E>`                            |

### Error Handling

| Go                                         | Rust                                     |
| ------------------------------------------ | ---------------------------------------- |
| `val, err := foo()`                        | `let val = foo()?;` or `match foo() { }` |
| `if err != nil { return err }`             | `?`                                      |
| `if err != nil { return fmt.Errorf(...) }` | `.map_err(\|e\| ...)?`                   |
| `errors.Is(err, target)`                   | `matches!(err, ...)` or downcast         |
| `panic("msg")`                             | `panic!("msg")`                          |
| `recover()`                                | `catch_unwind` (rarely used)             |

### Common Operations

| Go                         | Rust                                     |
| -------------------------- | ---------------------------------------- |
| `len(s)`                   | `s.len()`                                |
| `append(slice, item)`      | `vec.push(item)`                         |
| `copy(dst, src)`           | `dst.copy_from_slice(&src)`              |
| `strings.Contains(s, sub)` | `s.contains(sub)`                        |
| `strings.Split(s, sep)`    | `s.split(sep)`                           |
| `strconv.Itoa(n)`          | `n.to_string()`                          |
| `strconv.Atoi(s)`          | `s.parse::<i32>()`                       |
| `fmt.Sprintf("%v", x)`     | `format!("{:?}", x)`                     |
| `fmt.Println(x)`           | `println!("{x}")` or `println!("{x:?}")` |
| `json.Marshal(v)`          | `serde_json::to_string(&v)`              |
| `json.Unmarshal(data, &v)` | `serde_json::from_str(data)`             |

### Concurrency

| Go                | Rust                                       |
| ----------------- | ------------------------------------------ |
| `go func() { }()` | `thread::spawn(\|\| { })`                  |
| `sync.Mutex`      | `Mutex<T>` (wraps the data)                |
| `sync.RWMutex`    | `RwLock<T>`                                |
| `sync.WaitGroup`  | `thread::scope` or join handles            |
| `sync.Once`       | `std::sync::Once` or `once_cell`           |
| `context.Context` | No direct equivalent (see tokio for async) |

## Common Gotchas for Go Developers

### 1. Strings Are Not Indexable by Character

```rust
let s = "hello";
let c = s[0];  // ERROR: cannot index into a string
```

Strings are UTF-8. Indexing by byte could split a character. Use:

```rust
let c = s.chars().nth(0);  // Option<char>
let bytes = s.as_bytes()[0];  // u8 (if you want bytes)
```

### 2. No Implicit Type Conversions

```rust
let x: i32 = 5;
let y: i64 = x;  // ERROR: mismatched types
let y: i64 = x as i64;  // OK: explicit cast
let y: i64 = x.into();  // OK: using Into trait
```

Even `i32` to `i64` requires explicit conversion.

### 3. Methods Don't Have Implicit Self

```rust
impl MyStruct {
    fn method(&self) {
        self.field  // Must use `self.`
    }
}
```

No implicit `this` or receiver access like Go's methods.

### 4. No Struct Field Defaults

```rust
struct Config {
    timeout: u32,
    retries: u32,
}

let c = Config { timeout: 30 };  // ERROR: missing field `retries`
```

Options:

- Implement `Default` trait
- Use builder pattern
- Fill all fields

### 5. Macros Have ! and Are Not Functions

```rust
println("hello");   // ERROR: not a function
println!("hello");  // Correct: it's a macro
```

The `!` is part of the name. Common macros: `println!`, `vec!`, `format!`, `panic!`.

### 6. Iterators Are Lazy

```rust
let v = vec![1, 2, 3];
v.iter().map(|x| x * 2);  // Does nothing! Not consumed

let doubled: Vec<_> = v.iter().map(|x| x * 2).collect();  // Actually runs
```

Call `.collect()`, `.for_each()`, or use in a `for` loop.

### 7. Iterator Ownership: iter vs into_iter

Go's `for range` is simple. Rust has three iterator methods with different ownership:

```rust
let v = vec![1, 2, 3];

for x in &v { }           // x is &i32 (borrows, v still usable)
for x in &mut v { }       // x is &mut i32 (mutable borrow)
for x in v { }            // x is i32 (consumes v, v is gone)
```

Or explicitly:

- `.iter()` → yields `&T` (borrows)
- `.iter_mut()` → yields `&mut T` (mutable borrows)
- `.into_iter()` → yields `T` (takes ownership, consumes collection)

If you use `v` after `for x in v`, the compiler will complain that `v` was moved.

### 8. Shadowing Is Intentional

```rust
let x = "5";
let x = x.parse::<i32>().unwrap();  // Same name, different type
```

This is idiomatic Rust, not a mistake.

### 9. match Arms Must Return Same Type

```rust
let x = if condition { 5 } else { "five" };  // ERROR: different types
```

All branches must return the same type. Use enums if you need variants.

### 10. Unused Variables Warn, Unused Results Error

```rust
let x = 5;  // Warning: unused variable (prefix with _ to silence)
let _x = 5;  // No warning

foo()?;  // If foo returns Result, you must handle it
let _ = foo();  // Explicitly ignoring is OK
```

### 11. Clone Is Explicit

```rust
let s1 = String::from("hello");
let s2 = s1;  // s1 is MOVED, not copied
let s2 = s1.clone();  // Explicit copy
```

No implicit deep copies. You type `.clone()` and you know you're paying for it.

### 12. String vs &str in Function Arguments

```rust
// Inflexible: only accepts String
fn greet(name: String) { }

// Flexible: accepts String, &str, &String
fn greet(name: &str) { }
```

**Pro tip:** Take `&str` as function arguments, even if your source is a `String`. It's like how Go code often takes `[]byte`—more flexible. Return `String` when you're creating new data.

## When You're Fighting the Borrow Checker

### "Cannot borrow as mutable because it's also borrowed as immutable"

You're holding a `&T` and trying to get a `&mut T`.

```rust
let mut v = vec![1, 2, 3];
let first = &v[0];  // Immutable borrow
v.push(4);          // ERROR: mutable borrow while immutable exists
println!("{first}");
```

**Fix:** Finish using the immutable reference first:

```rust
let mut v = vec![1, 2, 3];
let first = v[0];  // Copy the value instead of borrowing
v.push(4);         // OK now
println!("{first}");
```

Or restructure so borrows don't overlap.

### "Borrowed value does not live long enough"

A reference is outliving what it points to.

```rust
fn bad() -> &String {
    let s = String::from("hello");
    &s  // ERROR: s is dropped at end of function
}
```

**Fix:** Return owned data:

```rust
fn good() -> String {
    let s = String::from("hello");
    s  // Move ownership out
}
```

### "Use of moved value"

You used a value after giving away ownership.

```rust
let s = String::from("hello");
let s2 = s;
println!("{s}");  // ERROR: s was moved
```

**Fixes:**

- Clone before the move: `let s2 = s.clone();`
- Borrow instead: `let s2 = &s;`
- Restructure to not need it afterward

### "Cannot move out of borrowed content"

Trying to take ownership through a reference.

```rust
fn take(v: &Vec<String>) -> String {
    v[0]  // ERROR: cannot move out of borrowed content
}
```

**Fixes:**

- Clone: `v[0].clone()`
- Take ownership of the whole thing: `fn take(v: Vec<String>)`
- Return a reference: `fn take(v: &Vec<String>) -> &String`

### General Strategies

1. **Clone liberally while learning.** Optimize later. Getting it working matters more than getting it fast.

2. **Use owned types.** `String` instead of `&str`, `Vec<T>` instead of `&[T]`. Simpler ownership.

3. **Reduce reference scope.** The shorter a borrow lives, the less it conflicts.

4. **Consider if you really need the reference.** Sometimes copying a small value is simpler than borrowing.

5. **Restructure data.** If you're fighting ownership, maybe your data layout is fighting Rust's model.

6. **Rc/Arc when needed.** Shared ownership is legitimate. Don't feel bad about using it.

## Quick Debugging

```rust
// Print anything with Debug trait
println!("{:?}", value);
println!("{value:?}");  // Shorthand

// Pretty print (multi-line)
println!("{:#?}", complex_struct);

// Print with variable name
dbg!(value);  // Prints: [src/main.rs:10] value = ...

// What type is this?
fn print_type<T>(_: &T) {
    println!("{}", std::any::type_name::<T>());
}
print_type(&value);
```

## Cargo Commands You'll Use

```bash
cargo build              # Compile
cargo build --release    # Compile with optimizations
cargo run                # Compile and run
cargo test               # Run tests
cargo test test_name     # Run specific test
cargo check              # Fast type-check (no codegen)
cargo clippy             # Linter (install: rustup component add clippy)
cargo fmt                # Format code
cargo doc --open         # Generate and view docs
cargo add crate_name     # Add dependency
cargo update             # Update dependencies
```

## Essential Crates

| Need           | Crate                               |
| -------------- | ----------------------------------- |
| Serialization  | `serde`, `serde_json`               |
| HTTP client    | `reqwest`                           |
| HTTP server    | `axum`, `actix-web`                 |
| Async runtime  | `tokio`                             |
| CLI parsing    | `clap`                              |
| Error handling | `anyhow` (apps), `thiserror` (libs) |
| Logging        | `tracing`, `log`                    |
| Regex          | `regex`                             |
| Date/time      | `chrono`, `time`                    |
| Random         | `rand`                              |
| Testing        | `proptest`, `mockall`               |

Find more at [crates.io](https://crates.io) and [lib.rs](https://lib.rs).

---

That's it. You know programming. Now you know enough Rust to be dangerous.

Write code. Fight the compiler. It gets easier.
