# Chapter 5: Smart Pointers

Go has `*T`. That's it. One pointer type, does everything.

Rust has a whole taxonomy:

- `&T` and `&mut T` - references (borrows)
- `Box<T>` - owned heap allocation
- `Rc<T>` - reference counted (single-threaded)
- `Arc<T>` - atomic reference counted (thread-safe)
- `RefCell<T>` - interior mutability
- `Mutex<T>` - thread-safe interior mutability

Why so many? Because Rust makes you be explicit about ownership and thread safety. Each type encodes different guarantees.

## Box<T>: Simple Heap Allocation

`Box<T>` puts a value on the heap with single ownership.

```rust
let b = Box::new(5);  // 5 is on the heap
println!("{b}");      // Deref coercion: use it like a regular value
```

When would you use this over just... putting the value on the stack?

### Use Case 1: Recursive Types

This doesn't compile:

```rust
enum List {
    Cons(i32, List),  // ERROR: recursive type has infinite size
    Nil,
}
```

Rust needs to know the size of types at compile time. A `List` that contains a `List` is infinitely sized.

Fix it with `Box`:

```rust
enum List {
    Cons(i32, Box<List>),  // Box has known size (pointer)
    Nil,
}
```

Now `Cons` holds a pointer to the next node, not the node itself.

### Use Case 2: Large Data

Moving large structs copies bytes. Boxing avoids that:

```rust
struct Huge {
    data: [u8; 1_000_000],
}

// Without Box: copies 1MB on every move
let h1 = Huge { data: [0; 1_000_000] };
let h2 = h1;  // 1MB copy

// With Box: copies 8 bytes (the pointer)
let h1 = Box::new(Huge { data: [0; 1_000_000] });
let h2 = h1;  // 8-byte pointer move
```

### Use Case 3: Trait Objects

```rust
let animal: Box<dyn Animal> = Box::new(Dog { name: "Rex".into() });
```

`dyn Animal` is unsized—you need a pointer to store it.

### Go Comparison

In Go, you'd just use `*T`:

```go
type Node struct {
    Value int
    Next  *Node  // Pointer to break recursion
}
```

Same idea, but Go doesn't make you think about it—everything can be a pointer implicitly.

## Rc<T>: Reference Counting

Sometimes multiple owners need access to the same data. Ownership rules say one owner, so what do you do?

`Rc<T>` (Reference Counted) allows shared ownership:

```rust
use std::rc::Rc;

let data = Rc::new(vec![1, 2, 3]);
let a = Rc::clone(&data);  // Increments ref count
let b = Rc::clone(&data);  // Increments ref count again

println!("count: {}", Rc::strong_count(&data));  // 3
```

When all `Rc` pointers go out of scope, the data is dropped.

### When to Use Rc

Graph structures, caches, or anywhere you genuinely need shared ownership:

```rust
struct Node {
    value: i32,
    children: Vec<Rc<Node>>,  // Children might be shared
}
```

### Limitations

1. **Single-threaded only.** `Rc` is not thread-safe. The compiler will stop you from sending it across threads.

2. **Immutable by default.** `Rc<T>` only gives you `&T`, not `&mut T`. Multiple owners can't all mutate.

### Go Comparison

Go doesn't have reference counting—the GC handles it. The closest mental model is shared pointers where the GC tracks when nothing references the data anymore.

## Arc<T>: Atomic Reference Counting

`Arc<T>` is `Rc<T>` but thread-safe:

```rust
use std::sync::Arc;
use std::thread;

let data = Arc::new(vec![1, 2, 3]);

let handles: Vec<_> = (0..3).map(|i| {
    let data = Arc::clone(&data);
    thread::spawn(move || {
        println!("Thread {i}: {:?}", data);
    })
}).collect();

for handle in handles {
    handle.join().unwrap();
}
```

`Arc` uses atomic operations to update the reference count, making it safe to share across threads.

### Rc vs Arc

|             | Rc<T>                            | Arc<T>                          |
| ----------- | -------------------------------- | ------------------------------- |
| Thread-safe | No                               | Yes                             |
| Performance | Faster                           | Slight overhead                 |
| Use when    | Single-threaded shared ownership | Multi-threaded shared ownership |

Rust makes you choose. Go's GC handles both cases transparently (with corresponding overhead).

## RefCell<T>: Interior Mutability

Normally, you can't mutate through a shared reference:

```rust
let data = Rc::new(vec![1, 2, 3]);
data.push(4);  // ERROR: cannot borrow as mutable
```

`RefCell<T>` moves borrow checking to runtime:

```rust
use std::cell::RefCell;

let data = RefCell::new(vec![1, 2, 3]);

data.borrow_mut().push(4);  // Runtime borrow check
println!("{:?}", data.borrow());  // [1, 2, 3, 4]
```

### How It Works

- `borrow()` - returns `Ref<T>`, like `&T`
- `borrow_mut()` - returns `RefMut<T>`, like `&mut T`

The borrowing rules still apply, but they're checked at runtime:

```rust
let data = RefCell::new(5);

let r1 = data.borrow();
let r2 = data.borrow();      // Fine: multiple immutable borrows

let r3 = data.borrow_mut();  // PANIC: already borrowed immutably
```

Violating borrow rules with `RefCell` causes a runtime panic, not a compile error.

### Combining Rc and RefCell

A common pattern: shared ownership with mutability.

```rust
use std::rc::Rc;
use std::cell::RefCell;

let shared_data = Rc::new(RefCell::new(vec![1, 2, 3]));

let a = Rc::clone(&shared_data);
let b = Rc::clone(&shared_data);

a.borrow_mut().push(4);  // Mutate through a
b.borrow_mut().push(5);  // Mutate through b

println!("{:?}", shared_data.borrow());  // [1, 2, 3, 4, 5]
```

### Go Comparison

Go lets you mutate shared data freely (and unsafely):

```go
data := []int{1, 2, 3}
// Any goroutine with a pointer can mutate. Data races are your problem.
```

Rust's `RefCell` gives you similar flexibility while still enforcing borrowing rules (at runtime).

## Mutex<T>: Thread-Safe Interior Mutability

For multi-threaded interior mutability, use `Mutex<T>`:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0));

let handles: Vec<_> = (0..10).map(|_| {
    let counter = Arc::clone(&counter);
    thread::spawn(move || {
        let mut num = counter.lock().unwrap();
        *num += 1;
    })
}).collect();

for handle in handles {
    handle.join().unwrap();
}

println!("Result: {}", *counter.lock().unwrap());  // 10
```

### The Pattern: Arc<Mutex<T>>

This is Rust's equivalent to Go's mutex pattern:

```go
// Go
type Counter struct {
    mu    sync.Mutex
    value int
}

func (c *Counter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}
```

```rust
// Rust
struct Counter {
    value: Arc<Mutex<i32>>,
}

impl Counter {
    fn inc(&self) {
        let mut num = self.value.lock().unwrap();
        *num += 1;
    }
}
```

Key difference: in Rust, the mutex _contains_ the data. You can't access the data without locking. In Go, the mutex is separate—you can forget to lock.

### RwLock<T>

When reads are more common than writes:

```rust
use std::sync::RwLock;

let lock = RwLock::new(vec![1, 2, 3]);

// Multiple readers OK
let r1 = lock.read().unwrap();
let r2 = lock.read().unwrap();

// Writers are exclusive
let mut w = lock.write().unwrap();
w.push(4);
```

Same as Go's `sync.RWMutex`.

## Choosing the Right Pointer

| Need                            | Use              | Notes                          |
| ------------------------------- | ---------------- | ------------------------------ |
| Heap allocation, single owner   | `Box<T>`         | Simplest                       |
| Shared ownership, single thread | `Rc<T>`          | No mutation                    |
| Shared ownership, multi thread  | `Arc<T>`         | No mutation                    |
| Shared + mutable, single thread | `Rc<RefCell<T>>` | Runtime borrow checks          |
| Shared + mutable, multi thread  | `Arc<Mutex<T>>`  | Locking overhead               |
| Read-heavy shared mutable       | `Arc<RwLock<T>>` | Readers don't block each other |

### Decision Tree

1. **Do you need heap allocation?**
   - No → use regular values or references
   - Yes → continue

2. **Is there a single owner?**
   - Yes → `Box<T>`
   - No → continue

3. **Is it multi-threaded?**
   - No → `Rc<T>` (add `RefCell` if you need mutation)
   - Yes → `Arc<T>` (add `Mutex` if you need mutation)

## Summary

Go gives you `*T` and expects you to use mutexes correctly. Rust gives you a type system that encodes:

- Single vs shared ownership (`Box` vs `Rc`/`Arc`)
- Single vs multi-threaded (`Rc` vs `Arc`)
- Mutable vs immutable (`RefCell`, `Mutex`)

It's more verbose. It's also harder to get wrong.

| Rust         | Go-ish Equivalent          | Key Insight                    |
| ------------ | -------------------------- | ------------------------------ |
| `Box<T>`     | `*T`                       | Single owner, heap allocated   |
| `Rc<T>`      | Shared `*T` (GC tracked)   | Multiple owners, single thread |
| `Arc<T>`     | Shared `*T` (GC tracked)   | Multiple owners, thread-safe   |
| `RefCell<T>` | Mutable shared state       | Runtime borrow checking        |
| `Mutex<T>`   | `sync.Mutex` guarding data | Lock owns the data             |
| `RwLock<T>`  | `sync.RWMutex`             | Multiple readers, one writer   |

---

Next: Concurrency—where Rust's ownership model really shines.
