# Chapter 10: Performance: Making Ruby Devs Cry (Tears of Joy)

Let's talk about the elephant in the room: Ruby is slow. Not "a little slow" - Ruby is often 10x, 50x, sometimes 100x slower than compiled languages. We love Ruby anyway because of productivity, expressiveness, and joy. But sometimes, speed matters.

Crystal promises Ruby's syntax with C's performance. Let's see if it delivers.

## The Speed Comparison

Create `benchmark_ruby.rb`:
```ruby
def fibonacci(n)
  return n if n <= 1
  fibonacci(n - 1) + fibonacci(n - 2)
end

require 'benchmark'
puts Benchmark.measure { puts fibonacci(40) }
```

Create `benchmark_crystal.cr`:
```crystal
def fibonacci(n)
  return n if n <= 1
  fibonacci(n - 1) + fibonacci(n - 2)
end

require "benchmark"
result = Benchmark.measure { fibonacci(40) }
puts result
```

Run them:
```bash
ruby benchmark_ruby.rb
# Real: ~30-40 seconds

crystal run --release benchmark_crystal.cr
# Real: ~0.8 seconds
```

Crystal is approximately **40-50x faster** for this computation. That's not a typo.

## Why Crystal Is Fast

### 1. Compilation to Native Code

Ruby is interpreted (or JIT-compiled in Ruby 3+). Crystal compiles to native machine code via LLVM.

```crystal
def add(a : Int32, b : Int32) : Int32
  a + b
end

# Compiles to ~5 machine instructions
# No method lookup, no type checking, just addition
```

### 2. Static Typing with No Runtime Overhead

```crystal
x = 42  # Compiler knows this is Int32

# Ruby needs to:
# - Check x's type at runtime
# - Look up the + method
# - Handle possible type errors

# Crystal:
# - Knows types at compile time
# - Direct machine instruction for addition
# - No overhead
```

### 3. Struct vs Class (Value vs Reference)

```crystal
struct Point
  property x : Int32
  property y : Int32
  def initialize(@x, @y); end
end

# Stack-allocated, no garbage collection
# Cache-friendly memory layout
# Copies are fast
```

### 4. No Method Lookup

```crystal
class Dog
  def speak
    "Woof"
  end
end

dog = Dog.new
dog.speak  # Direct call, no runtime method lookup
```

Ruby has to look up `speak` at runtime. Crystal knows exactly which method to call at compile time.

### 5. LLVM Optimizations

Crystal uses LLVM, which applies aggressive optimizations:
- Inlining
- Dead code elimination
- Loop unrolling
- Constant propagation
- And hundreds more

## Benchmarking Crystal Code

Use the built-in `Benchmark` module:

```crystal
require "benchmark"

# Measure execution time
elapsed = Benchmark.realtime do
  1_000_000.times { |i| i * 2 }
end
puts "Elapsed: #{elapsed} seconds"

# Compare multiple approaches
Benchmark.ips do |x|
  x.report("string concatenation") { "hello" + " " + "world" }
  x.report("string interpolation") { "hello #{" "} world" }
  x.report("string builder") do
    String.build do |str|
      str << "hello" << " " << "world"
    end
  end
end
```

Output:
```
string concatenation  10.23M (97.73ns) (± 2.45%)
string interpolation   8.91M (112.21ns) (± 1.89%)
string builder        15.44M (64.77ns) (± 1.23%)
```

The output shows iterations per second - higher is better.

## Optimization Techniques

### 1. Use the --release Flag

Always benchmark with `--release`:

```bash
# Debug build (default): slow, with debug info
crystal build myapp.cr

# Release build: optimized
crystal build --release myapp.cr

# For benchmarking, ALWAYS use:
crystal run --release benchmark.cr
```

Release builds can be 5-10x faster than debug builds.

### 2. Use Structs for Small Data

```crystal
# Slow: heap allocation, GC pressure
class Point
  property x : Int32
  property y : Int32
  def initialize(@x, @y); end
end

# Fast: stack allocation, no GC
struct Point
  property x : Int32
  property y : Int32
  def initialize(@x, @y); end
end
```

Benchmark:
```crystal
require "benchmark"

class PointClass
  property x : Int32
  property y : Int32
  def initialize(@x, @y); end
end

struct PointStruct
  property x : Int32
  property y : Int32
  def initialize(@x, @y); end
end

Benchmark.ips do |x|
  x.report("class") { PointClass.new(1, 2) }
  x.report("struct") { PointStruct.new(1, 2) }
end

# struct is typically 3-5x faster for small objects
```

### 3. Avoid Unnecessary Allocations

```crystal
# Slow: creates new array each time
def process
  (1..1000).to_a.sum
end

# Fast: no array allocation
def process
  (1..1000).sum
end

# Even faster: constant folding
RESULT = (1..1000).sum
def process
  RESULT
end
```

### 4. Use String::Builder for Concatenation

```crystal
# Slow: creates new strings
result = ""
1000.times { |i| result += i.to_s }

# Fast: builds string efficiently
result = String.build do |str|
  1000.times { |i| str << i }
end
```

### 5. Leverage Type Information

```crystal
# Generic: slower
def sum(arr)
  total = 0
  arr.each { |x| total += x }
  total
end

# Specific type: faster
def sum(arr : Array(Int32)) : Int32
  total = 0
  arr.each { |x| total += x }
  total
end

# Even better: use built-in
def sum(arr : Array(Int32)) : Int32
  arr.sum  # Optimized implementation
end
```

### 6. Use StaticArray for Fixed-Size Arrays

```crystal
# Heap-allocated
arr = [1, 2, 3, 4, 5]

# Stack-allocated
arr = StaticArray[1, 2, 3, 4, 5]

# Benchmarkable difference for tight loops
Benchmark.ips do |x|
  x.report("Array") do
    arr = [1, 2, 3, 4, 5]
    arr.sum
  end

  x.report("StaticArray") do
    arr = StaticArray[1, 2, 3, 4, 5]
    arr.sum
  end
end
```

### 7. Avoid Exceptions in Hot Paths

```crystal
# Slow: exceptions are expensive
def parse_slow(str : String) : Int32
  begin
    str.to_i
  rescue
    0
  end
end

# Fast: use nilable return
def parse_fast(str : String) : Int32
  str.to_i? || 0
end
```

## Profiling Crystal Code

### Built-in Time Measurement

```crystal
require "benchmark"

def expensive_operation
  sum = 0
  10_000_000.times { |i| sum += i }
  sum
end

elapsed = Benchmark.realtime do
  expensive_operation
end

puts "Took #{elapsed} seconds"
```

### Using Valgrind (Linux/macOS)

Profile cache misses and memory usage:

```bash
crystal build --release myapp.cr
valgrind --tool=callgrind ./myapp
```

### Using Instruments (macOS)

```bash
crystal build --release myapp.cr
instruments -t "Time Profiler" ./myapp
```

### Using perf (Linux)

```bash
crystal build --release myapp.cr
perf record ./myapp
perf report
```

## Real-World Optimization Example

Let's optimize a URL parser:

**Version 1: Naive**
```crystal
def parse_url_v1(url : String)
  parts = url.split("://")
  protocol = parts[0]
  rest = parts[1].split("/")
  domain = rest[0]
  path = "/" + rest[1..].join("/")
  {protocol, domain, path}
end
```

**Version 2: Reduced Allocations**
```crystal
def parse_url_v2(url : String)
  protocol_end = url.index("://") || return nil
  protocol = url[0...protocol_end]

  rest = url[(protocol_end + 3)..]
  path_start = rest.index("/") || rest.size
  domain = rest[0...path_start]
  path = rest[path_start..] || "/"

  {protocol, domain, path}
end
```

**Version 3: No Allocations**
```crystal
struct URL
  getter protocol : String
  getter domain : String
  getter path : String

  def initialize(@protocol, @domain, @path)
  end

  def self.parse(url : String) : URL?
    protocol_end = url.index("://") || return nil
    protocol = url[0...protocol_end]

    rest_start = protocol_end + 3
    path_start = url.index("/", rest_start) || url.size
    domain = url[rest_start...path_start]
    path = path_start < url.size ? url[path_start..] : "/"

    URL.new(protocol, domain, path)
  end
end
```

Benchmark:
```crystal
require "benchmark"

url = "https://example.com/path/to/resource"

Benchmark.ips do |x|
  x.report("v1") { parse_url_v1(url) }
  x.report("v2") { parse_url_v2(url) }
  x.report("v3") { URL.parse(url) }
end

# v1:  800K iterations/sec
# v2:  2M iterations/sec
# v3:  3M iterations/sec
```

## Memory Management

Crystal uses a garbage collector, but you can minimize its impact:

### 1. Reuse Objects

```crystal
# Allocates new string each time
def format_slow(n : Int32)
  "Number: #{n}"
end

# Reuses string builder
builder = String::Builder.new
def format_fast(n : Int32, builder)
  builder.clear
  builder << "Number: " << n
  builder.to_s
end
```

### 2. Use Object Pools

```crystal
class BufferPool
  def initialize(size : Int32)
    @pool = Array.new(size) { String::Builder.new }
    @index = 0
  end

  def acquire : String::Builder
    buffer = @pool[@index % @pool.size]
    @index += 1
    buffer.clear
    buffer
  end
end
```

### 3. Prefer Stack Allocation

```crystal
# Heap-allocated
def create_points_heap(n)
  points = [] of Point
  n.times { points << Point.new(rand(100), rand(100)) }
  points
end

# Stack-allocated struct in array
struct Point
  property x : Int32
  property y : Int32
  def initialize(@x, @y); end
end

def create_points_stack(n)
  points = Array(Point).new(n)
  n.times { points << Point.new(rand(100), rand(100)) }
  points
end
```

## Common Performance Pitfalls

### 1. String Concatenation in Loops

```crystal
# BAD
str = ""
1000.times { |i| str += i.to_s }

# GOOD
str = String.build do |builder|
  1000.times { |i| builder << i }
end
```

### 2. Unnecessary Type Checks

```crystal
# SLOW
def process(value : String | Int32)
  if value.is_a?(String)
    value.upcase
  else
    value * 2
  end
end

# FAST: use method overloading
def process(value : String)
  value.upcase
end

def process(value : Int32)
  value * 2
end
```

### 3. Large Struct Copies

```crystal
# BAD: copies 1KB on each call
struct LargeStruct
  property data : StaticArray(Int32, 256)
end

def process(s : LargeStruct)
  # Expensive copy!
end

# GOOD: use reference
def process(s : LargeStruct*)
  # Pass by reference
end
```

## Comparing Ruby and Crystal Performance

Let's test a real-world scenario - JSON parsing:

**Ruby**:
```ruby
require 'json'
require 'benchmark'

data = {users: (1..1000).map { |i| {id: i, name: "User #{i}"} }}
json = data.to_json

puts Benchmark.measure {
  10_000.times { JSON.parse(json) }
}
```

**Crystal**:
```crystal
require "json"
require "benchmark"

data = {users: (1..1000).map { |i| {id: i, name: "User #{i}"} }}
json = data.to_json

puts Benchmark.measure {
  10_000.times { JSON.parse(json) }
}
```

Typical results:
- Ruby: 8-12 seconds
- Crystal: 0.5-1 seconds

**Crystal is ~10-20x faster** for JSON parsing.

## When Speed Doesn't Matter

Sometimes Ruby is fine:
- **Prototyping** - iterate fast, optimize later
- **I/O bound** - database/network are the bottleneck
- **Small scale** - handling 10 requests/sec vs 10,000
- **Developer time** - if the ecosystem is better in Ruby

Crystal excels when:
- **CPU bound** - heavy computation
- **High throughput** - handling many requests
- **Low latency** - response time matters
- **Resource constrained** - limited memory/CPU

## Exercises

1. Benchmark string operations: concatenation, interpolation, builder
2. Compare class vs struct performance for a small data type
3. Profile a Crystal program with your platform's tools
4. Optimize a recursive function using memoization
5. Write a benchmark comparing Ruby and Crystal for a task you commonly do

## What You've Learned

1. Crystal is typically 10-50x faster than Ruby
2. Compilation, static typing, and LLVM enable the speed
3. Always use `--release` flag for benchmarks
4. `Benchmark.ips` measures iterations per second
5. Structs are faster than classes for small data
6. Avoid allocations in hot paths
7. Profile before optimizing - measure, don't guess
8. Choose Crystal when performance matters, Ruby when productivity matters

## Conclusion: The Journey from Ruby to Crystal

You've completed your journey from Ruby to Crystal. You've learned:

1. **Chapter 01**: Crystal's Ruby-like syntax and compiled nature
2. **Chapter 02**: The type system that prevents bugs
3. **Chapter 03**: Compile time vs runtime and why it matters
4. **Chapter 04**: Nil safety that eliminates NoMethodError
5. **Chapter 05**: Structs vs classes for performance and semantics
6. **Chapter 06**: Concurrency with fibers and channels
7. **Chapter 07**: Shards for dependency management
8. **Chapter 08**: C bindings for accessing C libraries
9. **Chapter 09**: Macros for compile-time metaprogramming
10. **Chapter 10**: Performance optimization and benchmarking

## When to Use Crystal vs Ruby

**Use Ruby when:**
- Ecosystem matters (gems, Rails, community)
- Rapid prototyping and iteration
- Metaprogramming needs are complex
- Team familiarity with Ruby
- Performance is not critical

**Use Crystal when:**
- Performance is critical
- Type safety prevents bugs
- Building CLI tools
- Microservices with high throughput
- Replacing performance-critical Ruby code
- Learning about compiled languages

## The Best of Both Worlds

You don't have to choose. Many teams use both:
- Ruby for rapid web development
- Crystal for performance-critical services
- Ruby for business logic
- Crystal for data processing

## Where to Go from Here

1. **Build something** - The best way to learn is by doing
2. **Read the docs** - [crystal-lang.org/docs](https://crystal-lang.org/reference/)
3. **Explore shards** - Find libraries at [crystalshards.org](https://crystalshards.org)
4. **Join the community** - Crystal forum, Discord, Reddit
5. **Contribute** - Fix bugs, write docs, create shards
6. **Stay updated** - Follow Crystal's development

## Final Thoughts

Crystal isn't trying to replace Ruby - it's offering an alternative when performance matters. The Ruby-like syntax means your knowledge transfers. The compilation and type system mean your code runs fast and catches errors early.

You already know Ruby. Now you know Crystal. You have two powerful tools in your toolkit. Use them wisely.

Welcome to the Crystal community. Now go build something fast.

---

**Thank you for reading!** If you found this book helpful, consider contributing to the Crystal language or creating shards to help the ecosystem grow. The Crystal community is friendly, welcoming, and always looking for contributors.

Happy coding!
