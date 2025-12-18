# Rust for Go Developers

A condensed guide to Rust for programmers with Go experience.

You know how to program. You know Go. This pamphlet teaches you what's _different_ about Rust. I'm not gonna explain what a variable is, but why the compiler won't let you use one after you've moved it.

I've taken inspiration, structure and examples from the Rust book by Steve Klabnik, Carol Nichols, and Chris Krycho.

## Chapters

1. **[Setup & Syntax](01-setup-and-syntax.md)** - Cargo, syntax differences, type mappings
2. **[Ownership](02-ownership.md)** - The big one. Move semantics, borrowing, lifetimes
3. **[Enums and Pattern Matching](03-enums-and-pattern-matching.md)** - Option, Result, the ? operator
4. **[Traits](04-traits.md)** - Rust's interfaces, static vs dynamic dispatch
5. **[Smart Pointers](05-smart-pointers.md)** - Box, Rc, Arc, RefCell, Mutex
6. **[Concurrency](06-concurrency.md)** - Send/Sync, fearless concurrency
7. **[Quick Reference](07-quick-reference.md)** - Cheat sheets, gotchas, survival guide

## Who This Is For

Go developers wanting to learn Rust. Rust might not be very productive compared to Go, but I've found it to be a lot of fun to program.

## Who This Is Not For

People with time and attention span to read the entire book. [The Book](https://doc.rust-lang.org/book/) covers it better.

## Building

This is plain Markdown.

## License

[CC0 1.0](LICENSE) - Public domain. Do whatever you want with it.
