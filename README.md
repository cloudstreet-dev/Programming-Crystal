# Programming Crystal: A Ruby Developer's Guide

Welcome to **Programming Crystal**, a practical guide to Crystal programming for developers who already know Ruby. This book teaches Crystal with humor, intention, and a healthy appreciation for the fact that you've probably spent years writing `end` keywords.

## Who This Book Is For

This book is written for Ruby developers who want to explore Crystal - a language that looks like Ruby, compiles like C, and runs fast enough to make you question your life choices. If you've ever wished Ruby could be statically typed, compiled, and blazingly fast while still feeling like home, you're in the right place.

## What You'll Learn

Crystal isn't just "fast Ruby" - it's a distinct language with its own philosophy and trade-offs. You'll learn how to:

- Write Crystal code that feels familiar but behaves differently
- Embrace static typing without losing your mind
- Understand when compile-time beats runtime (spoiler: often)
- Handle concurrency with fibers and channels
- Build high-performance applications without sacrificing readability
- Know when to reach for Crystal vs when to stick with Ruby

## Table of Contents

### [Chapter 01 - Hello Crystal (You Already Know This)](./chapter-01.md)
Installation, basic syntax, and the "it's Ruby but..." moment. Your first Crystal program and why it feels both comforting and slightly wrong.

### [Chapter 02 - Types: The Plot Twist](./chapter-02.md)
Static typing, type inference, union types, and why `nil` is now something you have to think about. The type system that Ruby developers didn't know they wanted.

### [Chapter 03 - Compile Time vs Runtime: The Great Divide](./chapter-03.md)
Understanding compilation, the difference between compile-time and runtime, and why some of your favorite Ruby tricks won't work anymore (and what to do instead).

### [Chapter 04 - Nil Safety (No More NoMethodError on nil:NilClass)](./chapter-04.md)
How Crystal's type system saves you from nil-related bugs, handling optional values properly, and the magical `try` method that might save your sanity.

### [Chapter 05 - Structs vs Classes: Speed Dating Edition](./chapter-05.md)
Value types vs reference types, when to use structs, when to use classes, and why this matters more than you think for performance and memory.

### [Chapter 06 - Concurrency: Fibers, Channels, and Why You'll Sleep Better](./chapter-06.md)
Crystal's concurrency model, how fibers differ from threads, channels for communication, and building concurrent applications that don't make you cry.

### [Chapter 07 - Shards: Bundler's Younger, Faster Sibling](./chapter-07.md)
Dependency management with shards, creating your own shards, and navigating the Crystal ecosystem (which is smaller but growing).

### [Chapter 08 - C Bindings: When You Need to Get Down and Dirty](./chapter-08.md)
FFI and lib declarations, binding to C libraries, and accessing the vast world of C code from your Crystal applications.

### [Chapter 09 - Macros: The Metaprogramming You Thought You Lost](./chapter-09.md)
Compile-time code generation, macro methods, and how to achieve Ruby-like metaprogramming magic at compile time instead of runtime.

### [Chapter 10 - Performance: Making Ruby Devs Cry (Tears of Joy)](./chapter-10.md)
Benchmarking, optimization techniques, profiling, and understanding where Crystal's speed comes from. Prepare to be amazed (and slightly annoyed at how slow Ruby suddenly feels).

## Reading This Book

Each chapter builds on the previous ones, but feel free to jump around if you're looking for something specific. Code examples are plentiful, and we'll often compare Crystal approaches to their Ruby equivalents.

All code examples assume you have Crystal installed. If you don't, Chapter 01 will get you set up.

## Contributing

Found a mistake? Have a suggestion? Want to add more terrible puns? Contributions are welcome! This book is open source and lives at [github.com/cloudstreet-dev/Programming-Crystal](https://github.com/cloudstreet-dev/Programming-Crystal).

## About the Author

This book is written by Ruby developers, for Ruby developers, with the understanding that learning a new language is both exciting and occasionally frustrating. We've been there, and we're here to make the journey smoother.

## License

This book is released under the MIT License. See LICENSE for details.

---

**Ready to start?** Head over to [Chapter 01](./chapter-01.md) and let's write some Crystal.
