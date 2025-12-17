# Chapter 3: Enums and Pattern Matching

Go doesn't have enums. `const` and `iota` are used to simulate enums.

```go
type Status int
const (
    Pending Status = iota
    Active
    Closed
)
```

Rust has real enums. And they're one of the language's best features.

## Enums with Data

Rust enums aren't just named constants. Each variant can hold different data:

```rust
enum Message {
    Quit,                       // No data
    Move { x: i32, y: i32 },    // Named fields (like a struct)
    Write(String),              // Single value
    ChangeColor(i32, i32, i32), // Multiple values (tuple-like)
}
```

One type, four variants, each with different shapes. Quite powerful.

This is called an _algebraic data type_ (specifically, a _sum type_). If you've used TypeScript's discriminated unions, it's the same idea but with compiler enforcement.

### Why This Matters

In Go, you'd model this with an interface and multiple structs:

```go
type Message interface {
    isMessage()
}

type QuitMessage struct{}
func (QuitMessage) isMessage() {}

type MoveMessage struct { X, Y int }
func (MoveMessage) isMessage() {}

type WriteMessage struct { Text string }
func (WriteMessage) isMessage() {}

// ... and so on
```

Verbose, right? And nothing stops you from forgetting to handle a case. Rust enums are:

- More concise
- Exhaustively checked (the compiler ensures you handle every variant)
- Impossible to create an "invalid" variant

## Option<T>: Goodbye nil

Here's Rust's most important enum:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

That's it. A value is either `Some(thing)` or `None`. The absence of `nil` is quite something.

### The Problem with nil

In Go:

```go
func findUser(id int) *User {
    // ... might return nil
}

user := findUser(42)
fmt.Println(user.Name)  // If user is nil: panic
```

You have to remember to check. The compiler doesn't care.

### The Solution

In Rust:

```rust
fn find_user(id: i32) -> Option<User> {
    // ... returns Some(user) or None
}

let user = find_user(42);
println!("{}", user.name);  // ERROR: Option<User> has no field `name`
```

You can't use the value without handling the `None` case. The compiler forces you.

### Working with Option

```rust
let maybe_number: Option<i32> = Some(5);

// Pattern matching - most explicit
match maybe_number {
    Some(n) => println!("Got: {n}"),
    None => println!("Got nothing"),
}

// if let - when you only care about one variant
if let Some(n) = maybe_number {
    println!("Got: {n}");
}

// unwrap - get the value or panic (use sparingly)
let n = maybe_number.unwrap();  // Panics if None

// unwrap_or - get the value or a default
let n = maybe_number.unwrap_or(0);

// map - transform the inner value
let doubled = maybe_number.map(|n| n * 2);  // Some(10)
```

### Go Translation

| Go pattern                   | Rust pattern                         |
| ---------------------------- | ------------------------------------ |
| `*T` that might be nil       | `Option<T>`                          |
| `val, ok := map[key]`        | `map.get(&key)` returns `Option<&V>` |
| Return `nil` for "not found" | Return `None`                        |

## Result<T, E>: Errors Done Right

Go's error handling:

```go
result, err := doSomething()
if err != nil {
    return err
}
// use result
```

Rust's error handling:

```rust
let result: Result<T, E> = do_something();
match result {
    Ok(value) => // use value,
    Err(e) => // handle error,
}
```

The `Result` enum:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

### Why Result is Better

In Go, you can ignore the error:

```go
result, _ := doSomething()  // Oops, ignored the error
```

In Rust, you must handle it:

```rust
let result = do_something();  // Result<T, E>
let value = result;           // Still a Result, not the value
```

You cannot accidentally use the value without handling the error case.

### The ? Operator

This is Rust's killer feature for error handling. Compare:

```go
// Go
func processFile(path string) error {
    file, err := os.Open(path)
    if err != nil {
        return err
    }
    defer file.Close()

    content, err := io.ReadAll(file)
    if err != nil {
        return err
    }

    result, err := parse(content)
    if err != nil {
        return err
    }

    return save(result)
}
```

```rust
// Rust
fn process_file(path: &str) -> Result<(), Error> {
    let file = File::open(path)?;
    let content = std::io::read_to_string(file)?;
    let result = parse(&content)?;
    save(result)?;
    Ok(())
}
```

The `?` operator:

1. If the Result is `Ok(value)`, unwrap it and continue
2. If the Result is `Err(e)`, return early with that error

It's `if err != nil { return err }` condensed to a single character.

### unwrap and expect

For quick scripts or when you're certain an operation won't fail:

```rust
// Panics with generic message if Err
let file = File::open("config.txt").unwrap();

// Panics with your message if Err
let file = File::open("config.txt").expect("config.txt must exist");
```

Use these sparingly in production code. They're fine for:

- Examples and prototypes
- Cases where failure is truly unrecoverable
- Test code

## Pattern Matching

Pattern matching is how you work with enums (and much more).

### match: Exhaustive Checking

```rust
enum Direction {
    North,
    South,
    East,
    West,
}

fn describe(dir: Direction) -> &'static str {
    match dir {
        Direction::North => "Going up",
        Direction::South => "Going down",
        Direction::East => "Going right",
        Direction::West => "Going left",
    }
}
```

If you forget a variant:

```rust
fn describe(dir: Direction) -> &'static str {
    match dir {
        Direction::North => "Going up",
        Direction::South => "Going down",
        // ERROR: non-exhaustive patterns: `East` and `West` not covered
    }
}
```

The compiler catches it. No silent bugs from unhandled cases.

### Destructuring

Pattern matching can extract data from variants:

```rust
enum Message {
    Move { x: i32, y: i32 },
    Write(String),
    Quit,
}

fn handle(msg: Message) {
    match msg {
        Message::Move { x, y } => println!("Move to ({x}, {y})"),
        Message::Write(text) => println!("Write: {text}"),
        Message::Quit => println!("Quit"),
    }
}
```

### Catch-All Patterns

When you don't care about some cases:

```rust
let number = 7;

match number {
    1 => println!("One"),
    2 => println!("Two"),
    _ => println!("Something else"),  // _ matches anything
}
```

Or bind the value:

```rust
match number {
    1 => println!("One"),
    other => println!("Got: {other}"),  // `other` binds the value
}
```

### if let: When You Only Care About One Case

```rust
let maybe_value: Option<i32> = Some(42);

// Full match
match maybe_value {
    Some(v) => println!("Got {v}"),
    None => {},  // Do nothing
}

// Simpler with if let
if let Some(v) = maybe_value {
    println!("Got {v}");
}
```

There's also `while let` for loops:

```rust
let mut stack = vec![1, 2, 3];

while let Some(top) = stack.pop() {
    println!("{top}");
}
```

### let else: Early Returns

New in recent Rust—great for validation:

```rust
fn process(value: Option<i32>) -> i32 {
    let Some(v) = value else {
        return 0;  // Must diverge (return, panic, etc.)
    };

    v * 2
}
```

This is like Go's:

```go
if value == nil {
    return 0
}
v := *value
```

### Matching Multiple Patterns

```rust
let number = 5;

match number {
    1 | 2 | 3 => println!("Small"),
    4..=6 => println!("Medium"),    // Range pattern (inclusive)
    _ => println!("Large"),
}
```

### Guards

Add conditions to patterns:

```rust
let pair = (2, -2);

match pair {
    (x, y) if x == y => println!("Equal"),
    (x, y) if x + y == 0 => println!("Sum to zero"),
    _ => println!("Something else"),
}
```

## Putting It Together

Here's a more realistic example combining everything:

```rust
enum Request {
    Get { path: String },
    Post { path: String, body: String },
    Delete { path: String },
}

fn handle_request(req: Request) -> Result<String, String> {
    match req {
        Request::Get { path } => {
            if path == "/health" {
                Ok("OK".to_string())
            } else {
                fetch_resource(&path)
            }
        }
        Request::Post { path, body } => {
            save_resource(&path, &body)?;  // Propagate errors with ?
            Ok("Created".to_string())
        }
        Request::Delete { path } => {
            delete_resource(&path)?;
            Ok("Deleted".to_string())
        }
    }
}
```

Compare to how you'd write this in Go with interfaces, type switches, and explicit error checking. The Rust version is:

- More concise
- Exhaustively checked
- Clear about error handling

## Summary

| Concept         | Go Equivalent                  | Key Difference                    |
| --------------- | ------------------------------ | --------------------------------- |
| Enums with data | Interfaces + structs           | Single type, compiler-checked     |
| `Option<T>`     | `*T` or `val, ok`              | Cannot ignore the None case       |
| `Result<T, E>`  | `(val, error)`                 | Cannot ignore the error           |
| `?` operator    | `if err != nil { return err }` | One character                     |
| `match`         | Type switch                    | Exhaustive, pattern destructuring |
| `if let`        | Type assertion                 | Concise single-pattern match      |

Enums and pattern matching change how you model data. Instead of interfaces and type switches, you have a closed set of variants that the compiler fully understands.

---

Next: Traits—Rust's answer to interfaces, but with compile-time dispatch.
