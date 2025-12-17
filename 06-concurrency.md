# Chapter 6: Concurrency

Go's concurrency story: "Don't communicate by sharing; share by communicating." Rust's concurrency story: "We'll make data races a compile error."

You know concurrency. This chapter is about how Rust makes it safer.

## Send and Sync: The Foundation

Rust's thread safety is built on two traits:

- **`Send`**: A type can be transferred to another thread
- **`Sync`**: A type can be shared between threads (via references)

Most types are both. The compiler implements these automatically when it's safe. You rarely think about them until something isn't `Send` or `Sync`.

### Types That Aren't Send or Sync

```rust
use std::rc::Rc;
use std::thread;

let data = Rc::new(5);

thread::spawn(move || {
    println!("{data}");  // ERROR: Rc<i32> cannot be sent between threads safely
});
```

`Rc` uses non-atomic reference counting. Sending it to another thread would cause data races on the count. The compiler prevents this.

Fix: use `Arc`:

```rust
use std::sync::Arc;

let data = Arc::new(5);

thread::spawn(move || {
    println!("{data}");  // Fine: Arc is Send
});
```

### Why This Matters

In Go, nothing stops you from sharing a non-thread-safe value:

```go
// Go: compiles fine, races at runtime
data := &NotThreadSafe{}
go func() {
    data.Mutate()  // Race condition
}()
data.Mutate()  // Race condition
```

Rust won't compile equivalent code unless the types are actually safe to share.

## Threads

Rust's threads are OS threads, not green threads like goroutines:

```rust
use std::thread;
use std::time::Duration;

let handle = thread::spawn(|| {
    println!("Hello from thread!");
    thread::sleep(Duration::from_millis(100));
});

handle.join().unwrap();  // Wait for thread to finish
```

### Move Closures

A common pattern—capture variables by moving them into the thread:

```rust
let data = vec![1, 2, 3];

let handle = thread::spawn(move || {
    // This closure owns `data` now
    println!("{:?}", data);
});

// println!("{:?}", data);  // ERROR: data was moved

handle.join().unwrap();
```

The `move` keyword transfers ownership into the closure. Without it:

```rust
let data = vec![1, 2, 3];

thread::spawn(|| {
    println!("{:?}", data);  // ERROR: closure may outlive current function
});
```

The compiler sees that the thread might outlive `data` and rejects it. `move` fixes this by giving the thread ownership.

### Go Comparison

```go
// Go: implicit capture, hope you don't have races
data := []int{1, 2, 3}
go func() {
    fmt.Println(data)  // Captures by reference, might race
}()
data[0] = 99  // Oops
```

Go captures by reference. If you mutate `data` after spawning the goroutine, you have a race. Rust's ownership makes you choose: move it in, or explicitly share it safely.

## Channels

Rust has channels, though they're not as central as in Go:

```rust
use std::sync::mpsc;  // Multi-producer, single-consumer
use std::thread;

let (tx, rx) = mpsc::channel();

thread::spawn(move || {
    tx.send("hello").unwrap();
});

let received = rx.recv().unwrap();
println!("Got: {received}");
```

**Warning for Go devs:** Note that `mpsc` stands for **Multi-Producer, Single-Consumer**. You can clone `tx` (the sender) but not `rx` (the receiver). If you want multiple workers pulling from one channel (MPMC, which Go channels support), you either wrap `rx` in `Arc<Mutex<>>` (see worker pool example below) or use the `crossbeam-channel` or `flume` crates.

### Multiple Producers

```rust
let (tx, rx) = mpsc::channel();

for i in 0..3 {
    let tx = tx.clone();  // Clone the sender
    thread::spawn(move || {
        tx.send(i).unwrap();
    });
}

drop(tx);  // Drop original sender

for received in rx {  // Iterate until all senders dropped
    println!("Got: {received}");
}
```

### Go Comparison

| Go                         | Rust                                       |
| -------------------------- | ------------------------------------------ |
| `ch := make(chan int)`     | `let (tx, rx) = mpsc::channel()`           |
| `ch := make(chan int, 10)` | `let (tx, rx) = mpsc::sync_channel(10)`    |
| `ch <- value`              | `tx.send(value).unwrap()`                  |
| `value := <-ch`            | `let value = rx.recv().unwrap()`           |
| `close(ch)`                | `drop(tx)` (when all senders dropped)      |
| `select { ... }`           | No direct equivalent (see crossbeam crate) |

Rust's standard library channels are simpler than Go's. For `select`-style operations, use the `crossbeam` crate.

### Ownership Through Channels

When you send a value, you transfer ownership:

```rust
let (tx, rx) = mpsc::channel();

let data = vec![1, 2, 3];
tx.send(data).unwrap();

// println!("{:?}", data);  // ERROR: data was moved

let received = rx.recv().unwrap();  // Receiver now owns it
```

Go's channels copy values (or share pointers). Rust's channels move ownership. No accidental sharing.

## Shared State

Sometimes channels aren't the right fit. You need shared mutable state.

### The Arc<Mutex<T>> Pattern

This is Rust's bread and butter for shared state:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let counter = Arc::clone(&counter);
    let handle = thread::spawn(move || {
        let mut num = counter.lock().unwrap();
        *num += 1;
    });
    handles.push(handle);
}

for handle in handles {
    handle.join().unwrap();
}

println!("Result: {}", *counter.lock().unwrap());  // 10
```

Breaking it down:

- `Mutex<T>` - ensures exclusive access to `T`
- `Arc<T>` - allows multiple threads to own the mutex
- `lock()` - acquires the lock, returns a guard
- Guard auto-unlocks when dropped (RAII)

### Mutex Poisoning

Why all the `.unwrap()` calls on `lock()`? If a thread panics while holding a lock, the mutex becomes _poisoned_. The next thread to call `lock()` gets an `Err` instead of the guard.

```rust
let result = counter.lock();  // Returns Result<MutexGuard, PoisonError>
let num = result.unwrap();    // "If previous holder panicked, I'll panic too"
```

This is Rust saying: "The data might be in an inconsistent state." You can recover with `lock().unwrap_or_else(|e| e.into_inner())` if you want to proceed anyway, but usually panicking is correct.

### Go Comparison

```go
// Go
var mu sync.Mutex
var counter int

var wg sync.WaitGroup
for i := 0; i < 10; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        mu.Lock()
        counter++
        mu.Unlock()
    }()
}
wg.Wait()
fmt.Println(counter)
```

Key differences:

1. **In Rust, the mutex owns the data.** You can't access `counter` without going through the mutex. In Go, the mutex is separate—you can forget to lock.

2. **Rust's lock returns the data.** `counter.lock()` gives you access. Go's `mu.Lock()` is a separate operation.

3. **Rust auto-unlocks.** The guard drops at end of scope. In Go, you need `defer mu.Unlock()` or explicit unlock.

### What Happens If You Forget to Lock?

Go:

```go
counter++  // Compiles fine, races at runtime
```

Rust:

```rust
*counter += 1;  // ERROR: cannot deref Arc<Mutex<i32>>
```

You literally cannot access the data without the lock. The type system prevents it.

### Deadlocks

Rust prevents data races. It does not prevent deadlocks:

```rust
let a = Arc::new(Mutex::new(1));
let b = Arc::new(Mutex::new(2));

// Thread 1: locks a, then b
// Thread 2: locks b, then a
// Classic deadlock - Rust won't save you
```

Deadlocks are a logic error, not a memory safety issue. You still need to think about lock ordering.

### RwLock for Read-Heavy Workloads

```rust
use std::sync::RwLock;

let data = Arc::new(RwLock::new(vec![1, 2, 3]));

// Multiple readers simultaneously
let r1 = data.read().unwrap();
let r2 = data.read().unwrap();

// Writers are exclusive
drop(r1);
drop(r2);
let mut w = data.write().unwrap();
w.push(4);
```

Same as Go's `sync.RWMutex`.

## Scoped Threads

Sometimes you want threads that can borrow from the stack:

```rust
use std::thread;

let data = vec![1, 2, 3];

thread::scope(|s| {
    s.spawn(|| {
        println!("{:?}", data);  // Borrows data, no move needed
    });
    s.spawn(|| {
        println!("Length: {}", data.len());  // Also borrows
    });
});  // Scope ensures all threads complete before data goes out of scope

println!("{:?}", data);  // Still valid!
```

The `scope` function guarantees all spawned threads finish before it returns. This lets threads borrow safely.

## A Note on Async

Rust has async/await. It's not like goroutines.

```rust
async fn fetch_data() -> Result<String, Error> {
    let response = reqwest::get("https://api.example.com").await?;
    response.text().await
}
```

Key differences from Go:

| Aspect     | Go                        | Rust async                  |
| ---------- | ------------------------- | --------------------------- |
| Runtime    | Built-in (goroutines)     | External (tokio, async-std) |
| Scheduling | Preemptive                | Cooperative                 |
| Default    | Everything is async-ready | Must explicitly use async   |

Async Rust is powerful but adds complexity:

- You need an async runtime (tokio is most popular)
- Not all code is async-compatible
- Lifetimes in async are tricky
- `Send` bounds get complicated

For many programs, regular threads are simpler. Async shines for high-concurrency I/O (thousands of connections).

This pamphlet won't cover async in depth. It deserves its own guide. Just know it exists and is different from Go's model.

## Common Patterns

### Worker Pool

```rust
use std::sync::mpsc;
use std::thread;

let (tx, rx) = mpsc::channel();
let rx = Arc::new(Mutex::new(rx));  // Share receiver among workers

let mut workers = vec![];
for id in 0..4 {
    let rx = Arc::clone(&rx);
    workers.push(thread::spawn(move || {
        loop {
            let job = rx.lock().unwrap().recv();
            match job {
                Ok(task) => println!("Worker {id} processing: {task}"),
                Err(_) => break,  // Channel closed
            }
        }
    }));
}

// Send jobs
for i in 0..10 {
    tx.send(format!("job-{i}")).unwrap();
}
drop(tx);  // Close channel

for worker in workers {
    worker.join().unwrap();
}
```

### Thread-Local State

```rust
use std::cell::RefCell;

thread_local! {
    static CACHE: RefCell<Vec<i32>> = RefCell::new(vec![]);
}

CACHE.with(|cache| {
    cache.borrow_mut().push(42);
});
```

## Summary

| Concept            | Go                         | Rust                      |
| ------------------ | -------------------------- | ------------------------- |
| Thread primitive   | Goroutine (green thread)   | `std::thread` (OS thread) |
| Safety model       | Convention + race detector | Compile-time enforcement  |
| Sharing data       | Mutexes                    | `Send`/`Sync` traits      |
| Mutex              | Separate from data         | Wraps the data            |
| Forgetting to lock | Runtime race               | Doesn't compile           |
| Channel ownership  | Copy/share                 | Move                      |
| Async              | Built-in                   | External runtime          |

Rust's concurrency model trades convenience for safety. You can't accidentally share mutable state across threads. The compiler catches it.

Go's model is simpler to start with. Rust's is harder to get wrong.

---

Next: Quick reference—cheat sheets, gotchas, and "when in doubt" guidance.
