# Chapter 1: Setup & Syntax

You know how to install things and use a terminal. Here's what's different.

## Cargo: Your New Best Friend

Cargo is `go` but better. It handles building, dependencies, testing, and more. Notice how I assume that
everything that has anything to do with Rust is superior. This is very common.

```bash
cargo new myproject     # Creates new project (like go mod init, but also creates main.go)
cargo build             # Compiles (like go build)
cargo run               # Compiles and runs
cargo test              # Runs tests
cargo check             # Fast type-check without full compile - use this often
```

Your project structure:

```
myproject/
├── Cargo.toml          # Like go.mod, but also handles build config
└── src/
    └── main.rs         # Entry point
```

Dependencies go in `Cargo.toml`:

```toml
[dependencies]
serde = "1.0"           # Like adding to go.mod, but you edit this directly
```

Run `cargo build` and it fetches dependencies automatically. No `go get`.

## Syntax: A Quick Translation

### Variables

```go
// Go
var x int = 5
x := 5                  // Short declaration, mutable
const y = 10
```

```rust
// Rust
let x: i32 = 5;         // Immutable by default!
let mut x: i32 = 5;     // mut = mutable
const Y: i32 = 10;      // Must have type, SCREAMING_CASE
```

Key difference: Rust variables are immutable by default. You'll type `mut` a lot until you internalize that immutability is the default.

### No Zero Values

Go gives you zero values. Rust doesn't.

```go
// Go - this is fine
var s string            // s is ""
var n int               // n is 0
var p *int              // p is nil
```

```rust
// Rust - none of these compile
let s: String;          // Error: not initialized
let n: i32;             // Error: not initialized
let p: &i32;            // Error: not initialized
```

You must initialize everything. This is one of those situations where Rust is more explicit than Go.

### Semicolons Matter

In Go, semicolons are inserted automatically. In Rust, they change meaning.

```rust
fn five() -> i32 {
    5               // No semicolon = this is the return value
}

fn also_five() -> i32 {
    return 5;       // Explicit return works too
}

fn returns_nothing() {
    5;              // Semicolon = statement, returns () (unit type, like void)
}
```

The last expression without a semicolon is the return value. This is idiomatic:

```rust
fn max(a: i32, b: i32) -> i32 {
    if a > b { a } else { b }   // No semicolon, no return keyword
}
```

### Type Annotations

```go
// Go: type after name, separated by space
func greet(name string) string
var ages map[string]int
```

```rust
// Rust: type after colon
fn greet(name: String) -> String
let ages: HashMap<String, i32>;
```

Generics use `<>` not `[]`:

```go
// Go
func Print[T any](val T)
```

```rust
// Rust
fn print<T>(val: T)
```

### Common Type Mappings

| Go               | Rust                                  |
| ---------------- | ------------------------------------- |
| `int`, `int64`   | `i32`, `i64` (be explicit about size) |
| `uint`, `uint64` | `u32`, `u64`                          |
| `float64`        | `f64`                                 |
| `string`         | `String` (owned) or `&str` (borrowed) |
| `[]T`            | `Vec<T>` (owned) or `&[T]` (borrowed) |
| `[N]T`           | `[T; N]` (note: size is second)       |
| `map[K]V`        | `HashMap<K, V>`                       |
| `bool`           | `bool`                                |

**Slice gotcha:** Go's `[]T` is `(pointer, length, capacity)`. Rust's `&[T]` is only `(pointer, length)`—no capacity. You can't "reslice" back into capacity like you can in Go. If you need growable, use `Vec<T>`.

The `String` vs `&str` distinction will make sense after Chapter 2. For now: use `String` when you're unsure.

## Modules: Not Quite Go Packages

Go has packages tied to directories. Rust has modules that are more flexible.

```rust
// In lib.rs or main.rs
mod utils;              // Looks for utils.rs or utils/mod.rs

// You can also inline modules
mod helpers {
    pub fn help() {}    // pub = exported (like uppercase in Go)
}
```

Visibility:

- Go: uppercase = public, lowercase = private
- Rust: `pub` = public, no modifier = private

```rust
pub struct User {       // Struct is public
    pub name: String,   // Field is public
    age: u32,           // Field is private (even though struct is public!)
}
```

Note: struct fields default to private even if the struct is public. In Go, a public struct exposes all uppercase fields. In Rust, you must mark each field `pub` explicitly.

## Hello World, Compared

```go
// Go
package main

import "fmt"

func main() {
    fmt.Println("Hello, world!")
}
```

```rust
// Rust
fn main() {
    println!("Hello, world!");  // println! is a macro (note the !)
}
```

The `!` means it's a macro, not a function. Macros are common in Rust—`println!`, `vec!`, `format!`. Don't worry about writing macros yet; just know the `!` isn't a typo.

---

That's the basics. If you can `cargo run` and read the syntax, you're ready for the real content: ownership.
