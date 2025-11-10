# Chapter 01: Hello Crystal (You Already Know This)

Welcome to Crystal! If you're a Ruby developer, you're about to have a familiar yet disorienting experience - like visiting a parallel universe where everything looks right but gravity is slightly different.

## What Is Crystal, Anyway?

Crystal is a statically-typed, compiled programming language with Ruby-inspired syntax. Let me translate: it looks like Ruby, runs like C, and makes you think about types like a responsible adult.

The Crystal team's goal was simple: "What if Ruby, but fast and type-safe?" The result is a language that feels comfortably familiar while introducing concepts that might make you go "wait, what?" at least seventeen times in the first hour.

## Installation

Let's get Crystal installed. The process varies by platform:

### macOS
```bash
brew install crystal
```

### Linux (Ubuntu/Debian)
```bash
curl -fsSL https://crystal-lang.org/install.sh | sudo bash
```

### Other Platforms
Visit [crystal-lang.org/install](https://crystal-lang.org/install) for detailed instructions.

Verify your installation:
```bash
crystal --version
```

You should see something like `Crystal 1.x.x`. If you do, congratulations - you're ready to write some blazingly fast code that looks suspiciously like Ruby.

## Your First Crystal Program

Let's start with the classic. Create a file called `hello.cr`:

```crystal
puts "Hello, Crystal!"
```

Wait, that's it? Yes. That's literally Ruby syntax. Run it:

```bash
crystal run hello.cr
```

You should see `Hello, Crystal!` printed to your terminal.

### But Here's Where It Gets Interesting

That `crystal run` command actually compiled your code before running it. You just didn't notice because Crystal is sneaky like that. To compile it into a standalone executable:

```bash
crystal build hello.cr
```

Now you have a binary file called `hello` that runs without needing Crystal installed. Try it:

```bash
./hello
```

This is your first clue that you're not in Ruby anymore. Ruby interprets. Crystal compiles. This distinction will become very important, and occasionally annoying.

## The Familiar Stuff

Let's explore what works exactly like Ruby:

### Variables and Basic Types

```crystal
# Variables (no declaration needed)
name = "Crystal"
age = 13  # Crystal was created in 2012
price = 99.99
is_awesome = true

# String interpolation
puts "#{name} is #{age} years old"

# Arrays
languages = ["Ruby", "Crystal", "Python"]
puts languages[0]

# Hashes
person = {"name" => "Alice", "age" => 30}
puts person["name"]

# Symbols
status = :active

# Ranges
(1..5).each do |i|
  puts i
end

# Blocks and iterators
[1, 2, 3].each do |num|
  puts num * 2
end

# Multiple line blocks
result = [1, 2, 3].map do |num|
  num * 2
end
```

If you're thinking "this is just Ruby," you're right! Crystal's syntax is deliberately Ruby-like to make you feel at home.

### Methods

```crystal
def greet(name)
  "Hello, #{name}!"
end

puts greet("World")

# Methods with default parameters
def greet_with_enthusiasm(name, enthusiasm = "!")
  "Hello, #{name}#{enthusiasm}"
end

puts greet_with_enthusiasm("Crystal")
puts greet_with_enthusiasm("Crystal", "!!!")

# Methods with multiple return values (using tuples)
def get_coordinates
  {40.7128, -74.0060}
end

lat, lon = get_coordinates
puts "Latitude: #{lat}, Longitude: #{lon}"
```

### Classes

```crystal
class Person
  def initialize(@name : String, @age : Int32)
  end

  def introduce
    "Hi, I'm #{@name} and I'm #{@age} years old"
  end
end

person = Person.new("Alice", 30)
puts person.introduce
```

Wait, what's that `@name : String` and `@age : Int32` stuff? Welcome to your first "it's Ruby but..." moment.

## The "It's Ruby But..." Moments

### Type Annotations

In Ruby, you'd write:
```ruby
def initialize(name, age)
  @name = name
  @age = age
end
```

In Crystal, you can write type annotations:
```crystal
def initialize(@name : String, @age : Int32)
end
```

That `@name : String` is doing three things:
1. Creating an instance variable `@name`
2. Assigning the `name` parameter to it
3. Declaring that it must be a `String`

The type annotations aren't always required (Crystal has type inference), but they help the compiler catch mistakes and make your code self-documenting.

### No `require` at Runtime

In Ruby, you're used to:
```ruby
require 'json'
require_relative 'my_file'
```

In Crystal, you use `require` but it works differently:
```crystal
require "json"
require "./my_file"
```

The difference? Crystal resolves all requires at compile time. If a file is missing, you'll know immediately when you compile, not when your program crashes in production at 3 AM.

### Everything Has a Type

Even though you don't always see them, everything in Crystal has a type. The compiler infers most types for you:

```crystal
x = 42          # x is Int32
y = 3.14        # y is Float64
z = "hello"     # z is String
arr = [1, 2, 3] # arr is Array(Int32)
```

You can check types at compile time:
```crystal
x = 42
typeof(x)  # => Int32
```

### Performance

Create a file called `benchmark.cr`:

```crystal
require "benchmark"

Benchmark.ips do |x|
  x.report("sum") do
    sum = 0
    1_000_000.times do |i|
      sum += i
    end
  end
end
```

Run it:
```bash
crystal run --release benchmark.cr
```

The `--release` flag enables optimizations. You'll see something like millions of iterations per second. Try writing equivalent Ruby code - you'll see why people get excited about Crystal's speed.

## The Mental Model Shift

Here's the key insight for Ruby developers: Crystal feels like Ruby but thinks like a compiled, statically-typed language.

**Ruby's mental model:**
- Everything happens at runtime
- Flexibility is paramount
- Duck typing means "if it quacks like a duck..."
- Metaprogramming can change anything, anytime

**Crystal's mental model:**
- Most things are resolved at compile time
- Type safety prevents entire classes of bugs
- "If it quacks like a duck" needs to be verified before runtime
- Metaprogramming happens via macros at compile time

## Common Gotchas for Ruby Developers

### 1. No `eval` or `instance_eval`
```crystal
# This doesn't exist in Crystal
# eval("puts 'hello'")  # NOPE
```

Why? Because Crystal can't compile arbitrary strings at runtime. The compiler needs to know everything upfront.

### 2. No Method Missing (Sort Of)
```crystal
# Ruby's method_missing doesn't exist
# But Crystal has macros that can do similar things at compile time
```

We'll cover macros in Chapter 09. They're powerful but work very differently.

### 3. Integers Are Sized
```crystal
x = 42        # Int32 (32-bit integer)
y = 42_i64    # Int64 (64-bit integer)
z = 42_u32    # UInt32 (unsigned 32-bit integer)
```

Ruby has arbitrary-precision integers. Crystal has fixed-size integers. This matters for memory usage and performance.

### 4. Division Behavior
```crystal
puts 5 / 2      # => 2 (integer division)
puts 5 / 2.0    # => 2.5 (float division)
puts 5.0 / 2    # => 2.5 (float division)
```

This is closer to C than Ruby. Be aware when dividing integers.

## A More Complete Example

Let's write something slightly more interesting. Create `greeter.cr`:

```crystal
class Greeter
  def initialize(@name : String)
    @greeting_count = 0
  end

  def greet(other_name : String) : String
    @greeting_count += 1
    "Hello #{other_name}, I'm #{@name}! " \
    "I've greeted #{@greeting_count} people today."
  end

  def greeting_count : Int32
    @greeting_count
  end
end

# Create a greeter
bob = Greeter.new("Bob")

# Greet some people
puts bob.greet("Alice")
puts bob.greet("Charlie")
puts bob.greet("Diana")

# Check the count
puts "Total greetings: #{bob.greeting_count}"
```

Run it:
```bash
crystal run greeter.cr
```

Notice:
- Type annotations on method parameters and return types
- Instance variable `@greeting_count` gets initialized and inferred as `Int32`
- The method `greeting_count` returns `Int32`
- It all looks very Ruby-ish but with type hints sprinkled in

## What You've Learned

1. Crystal looks like Ruby but compiles to native code
2. Type annotations are optional but helpful
3. The compiler is your friend (even when it's yelling at you)
4. Everything has a type, even if you don't write it explicitly
5. Crystal is fast - really fast
6. Some Ruby features don't work because of compile-time constraints

## Exercises

1. Write a Crystal program that prints the first 10 Fibonacci numbers
2. Create a class `Calculator` with methods for basic arithmetic
3. Write a program that reads user input and responds (hint: `gets` works like Ruby)
4. Try to use a Ruby feature that doesn't work in Crystal and understand why

## What's Next?

In the next chapter, we'll dive deep into Crystal's type system. You'll learn about type inference, union types, and why the compiler sometimes knows more about your code than you do. Prepare for the plot twist where static typing becomes your superpower instead of your enemy.

[Continue to Chapter 02 - Types: The Plot Twist →](./chapter-02.md)
