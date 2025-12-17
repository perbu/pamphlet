# Chapter 2: Ownership

This is the chapter. If Rust has a single defining feature, it's ownership. Every confusing compiler error, every "why can't I just do this," every moment of friction—ownership is usually the answer.

The good news: once it clicks, it clicks. The bad news: it takes a while to click.

## Why Ownership Exists

Go has a garbage collector. C has manual memory management. Rust has neither.

In Go, you allocate freely and the GC cleans up. You pay for this with:

- Allocation time - AKA GC pressure
- Memory overhead for GC bookkeeping
- Incompatibility with C

In C, you call `malloc` and `free` manually. You pay for this with:

- Use-after-free bugs
- Double-free bugs
- Memory leaks
- Your sanity

Rust's ownership system is a third way: the compiler tracks who owns each piece of memory and inserts cleanup code at compile time. No runtime GC, no manual `free`, no memory bugs, at least not in _safe_ code.

The cost? You have to satisfy the compiler. That's what this chapter is about.

## The Three Rules

Memorize these:

1. Each value has exactly one owner
2. When the owner goes out of scope, the value is dropped (freed)
3. Ownership can be transferred (moved)

Let's see what these mean in practice.

### Rule 1: One Owner

```rust
let s = String::from("hello");  // s owns the String
```

There's no ambiguity about who's responsible for this memory. It's `s`. Not a shared responsibility, not "whoever frees it first"—just `s`.

### Rule 2: Dropped When Out of Scope

```rust
fn main() {
    let s = String::from("hello");
    // use s...
}   // s goes out of scope here, String is freed automatically
```

This is like Go's `defer` but automatic and tied to scope. Rust calls this "dropping" and the mechanism is the `Drop` trait (which we'll cover later).

In Go terms, imagine if every variable had an invisible `defer cleanup()` that ran when the variable went out of scope. That's Rust, except it's not deferred to function end—it's scoped precisely.

### Rule 3: Ownership Moves

Here's where Go developers get surprised:

```rust
let s1 = String::from("hello");
let s2 = s1;                    // Ownership MOVES from s1 to s2

println!("{}", s1);             // ERROR: s1 is no longer valid
```

In Go, this would copy a pointer or the string header. Both variables would work fine. In Rust, ownership _moved_. `s1` is now invalid—the compiler won't let you use it.

Why? Because if both `s1` and `s2` owned the String, who frees it? Rust's answer: don't allow that situation.

## Move Semantics in Depth

Let's compare Go and Rust on what looks like the same code:

```go
// Go
s1 := "hello"
s2 := s1          // Copies the string header (pointer, length, capacity)
fmt.Println(s1)   // Fine - s1 is still valid
fmt.Println(s2)   // Fine - s2 has its own copy of the header
// Both point to the same underlying bytes, GC handles cleanup
```

```rust
// Rust
let s1 = String::from("hello");
let s2 = s1;      // Moves ownership - s1 is invalidated
println!("{s1}"); // ERROR: borrow of moved value
println!("{s2}"); // Fine - s2 is the owner now
```

The Rust version is tracking ownership at compile time. After the move, `s1` doesn't exist conceptually; it's been consumed.

### Moves Happen in Function Calls Too

```rust
fn take_ownership(s: String) {
    println!("{s}");
}   // s is dropped here

fn main() {
    let s = String::from("hello");
    take_ownership(s);          // s moves into the function
    println!("{s}");            // ERROR: s was moved
}
```

When you pass a value to a function, you're giving it away. The function owns it now. This is very different from Go, where you're always either copying or passing a pointer.

### Getting Values Back

If a function takes ownership, it can give it back:

```rust
fn take_and_give_back(s: String) -> String {
    s   // Return it, transferring ownership to the caller
}

fn main() {
    let s1 = String::from("hello");
    let s2 = take_and_give_back(s1);    // s1 moved in, result moved to s2
    println!("{s2}");                    // Fine
}
```

But this gets tedious. What if you just want to _look_ at a value without taking ownership? That's what borrowing is for.

## The Copy Trait: Types That Don't Move

Some types are cheap to copy and live entirely on the stack. These implement the `Copy` trait and behave more like Go:

```rust
let x = 5;
let y = x;          // Copies, doesn't move
println!("{x}");    // Fine - x is still valid
```

Copy types include:

- All integers (`i32`, `u64`, etc.)
- Floating point (`f32`, `f64`)
- Booleans
- Characters (`char`)
- Tuples of Copy types

`String` is not Copy because it owns heap memory. If it copied implicitly, you'd have two owners of the same heap memory—exactly what ownership prevents.

**Rule of thumb:** if it's stack-only and cheap, it's probably Copy. If it involves the heap, it moves.

## References and Borrowing

Moving ownership everywhere is cumbersome. Usually, you want to let a function _use_ a value without taking ownership. This is borrowing.

```rust
fn calculate_length(s: &String) -> usize {  // s is a reference to a String
    s.len()
}   // s goes out of scope, but since it doesn't own the String, nothing is dropped

fn main() {
    let s1 = String::from("hello");
    let len = calculate_length(&s1);        // &s1 creates a reference
    println!("Length of '{s1}' is {len}");  // s1 still valid!
}
```

`&s1` creates a reference—a pointer that borrows the value without taking ownership. The function can use it, but `s1` remains the owner.

### Go Comparison

In Go, you'd write:

```go
func calculateLength(s *string) int {   // Takes a pointer
    return len(*s)
}

func main() {
    s := "hello"
    length := calculateLength(&s)       // Pass address
    fmt.Printf("Length of '%s' is %d\n", s, length)
}
```

Looks similar, right? But Go's pointer has no rules about aliasing or mutation. You can have ten pointers to the same value, and any of them can mutate it at any time. Rust restricts this.

## The Borrowing Rules

Here's where Rust gets strict:

At any given time, you can have EITHER:

- One mutable reference (`&mut T`), OR
- Any number of immutable references (`&T`)

But never both.

### Immutable References (Shared Borrows)

```rust
let s = String::from("hello");
let r1 = &s;        // Fine
let r2 = &s;        // Fine - multiple immutable refs allowed
let r3 = &s;        // Still fine
println!("{r1}, {r2}, {r3}");
```

Many readers, no writers. Safe.

### Mutable References (Exclusive Borrows)

```rust
let mut s = String::from("hello");
let r = &mut s;     // Mutable borrow
r.push_str(" world");
println!("{r}");
```

Note: the variable must be declared `mut` to create a `&mut` reference.

### You Can't Mix Them

```rust
let mut s = String::from("hello");
let r1 = &s;            // Immutable borrow
let r2 = &s;            // Another immutable borrow - fine
let r3 = &mut s;        // ERROR: cannot borrow as mutable because it's already borrowed as immutable
println!("{r1}, {r2}");
```

Why this rule? Consider what could go wrong:

```rust
let mut data = vec![1, 2, 3];
let first = &data[0];       // Immutable borrow of first element
data.push(4);               // This might reallocate the vector!
println!("{first}");        // first could be a dangling pointer
```

Rust prevents this at compile time. Go would let this happen and you'd get... undefined behavior, probably.

### Scope of Borrows

A borrow lasts until its last use, not until end of scope (this is called Non-Lexical Lifetimes):

```rust
let mut s = String::from("hello");

let r1 = &s;
let r2 = &s;
println!("{r1} and {r2}");
// r1 and r2 are no longer used after this point

let r3 = &mut s;            // Fine - immutable borrows are "done"
println!("{r3}");
```

The compiler is smart about tracking when references are actually used.

## Lifetimes

Every reference has a _lifetime_ - the scope for which it's valid. Most of the time, Rust figures this out automatically. Sometimes, you need to be explicit.

### The Problem

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() { x } else { y }
}
```

This won't compile. Why? The compiler needs to know: how long is the returned reference valid? It depends on whether we return `x` or `y`, and those might have different lifetimes.

### The Solution: Lifetime Annotations

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

The `'a` is a lifetime parameter. This says: "the returned reference will be valid for at least as long as _both_ inputs are valid."

### Reading Lifetime Syntax

- `'a` - a lifetime named "a" (the name is arbitrary, like generic `T`)
- `&'a str` - a string slice that lives for at least lifetime 'a
- `<'a>` - declares a lifetime parameter (like `<T>` for types)

### When You Need Annotations

Most of the time, you don't. Rust has "lifetime elision rules" that fill them in automatically. You need explicit lifetimes when:

1. **Returning references that could come from multiple inputs**
2. **Storing references in structs**

```rust
struct Excerpt<'a> {
    text: &'a str,      // This struct contains a reference
}
```

This says: "an Excerpt cannot outlive the string it references."

### The 'static Lifetime

One special lifetime: `'static` means "lives for the entire program."

```rust
let s: &'static str = "hello";  // String literals are 'static
```

String literals are baked into the binary, so they're always valid.

### Lifetime Advice for Go Developers

When you hit lifetime errors, ask:

1. **Do I really need a reference?** Often you can just own the data.
2. **Can I clone?** `s.clone()` gives you an owned copy. It's not free, but it's simple.
3. **Is the struct holding a reference?** Consider making it own the data instead (`String` vs `&str`).

Lifetime errors often mean your data structure is fighting Rust's model. Sometimes the fix is to restructure, not to add more annotations.

## Common Ownership Patterns

### Pattern 1: Take Ownership When You Need to Store

```rust
struct User {
    name: String,       // Owns the string
}

impl User {
    fn new(name: String) -> User {  // Takes ownership of name
        User { name }
    }
}
```

### Pattern 2: Borrow When You Just Need to Read

```rust
fn print_user(user: &User) {    // Borrows, doesn't take ownership
    println!("{}", user.name);
}
```

### Pattern 3: Mutable Borrow When You Need to Modify

```rust
fn rename_user(user: &mut User, new_name: String) {
    user.name = new_name;
}
```

### Pattern 4: Clone When Ownership Is Tangled

```rust
let s = String::from("hello");
let s_copy = s.clone();         // Explicit deep copy
// Both s and s_copy are valid, independent strings
```

`Clone` is not free—it copies data. But sometimes it's the simplest solution while you're learning.

## Fighting the Borrow Checker

You will fight the borrow checker. Everyone does. Some strategies:

### "Cannot borrow as mutable because it's already borrowed as immutable"

Check if you're holding onto a reference longer than needed. Can you restructure to finish using the immutable reference before mutating?

### "Borrowed value does not live long enough"

The reference is trying to outlive what it points to. Either:

- Return an owned value instead of a reference
- Make sure the source lives long enough
- Consider `clone()`

### "Use of moved value"

You gave ownership away and then tried to use it. Either:

- Borrow instead of move (`&x` instead of `x`)
- Clone before the move
- Restructure so you don't need the value afterward

### When All Else Fails

1. **Clone it.** Seriously. Cloning is fine, especially while learning. Optimize later.
2. **Use owned types.** `String` instead of `&str`, `Vec<T>` instead of `&[T]`.
3. **Ask if your data structure makes sense.** Sometimes the borrow checker is telling you your design has shared mutable state problems that would be bugs in any language.

## Summary

| Concept   | Go Equivalent   | Key Difference                     |
| --------- | --------------- | ---------------------------------- |
| Ownership | (nothing)       | Values have exactly one owner      |
| Move      | (doesn't exist) | Assignment transfers ownership     |
| Copy      | Value types     | Explicit trait, stack-only types   |
| `&T`      | `*T`            | Cannot mutate, can have many       |
| `&mut T`  | `*T`            | Exclusive access, only one allowed |
| Lifetimes | (nothing)       | Compiler tracks reference validity |
| Drop      | `defer`         | Automatic, scope-based             |

The ownership system is Rust's answer to memory safety without garbage collection. It's not just a different way to do the same thing—it's a fundamentally different model that prevents entire categories of bugs at compile time.

Yes, you'll fight the compiler. But every fight is a bug it caught before production.

---

Next: Enums and pattern matching—where Rust gives you tools Go doesn't have.
