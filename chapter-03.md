# Chapter 03: Compile Time vs Runtime: The Great Divide

You've been writing Crystal code that looks like Ruby, but underneath there's a fundamental difference: Ruby interprets your code at runtime, while Crystal compiles it beforehand. This distinction affects everything from performance to what features are possible.

Think of it this way: Ruby is like jazz improvisation - flexible, dynamic, making decisions in the moment. Crystal is like a symphony - planned, rehearsed, and executed flawlessly because everything was figured out in advance.

## The Two Phases of Crystal Programs

Every Crystal program lives in two distinct worlds:

### Compile Time: The Planning Phase

This is when the Crystal compiler:
- Reads your source code
- Checks all types
- Resolves all method calls
- Expands all macros
- Optimizes everything
- Generates machine code

**Nothing from your program actually runs yet.** The compiler is just analyzing and transforming your code.

### Runtime: The Execution Phase

This is when your compiled binary:
- Actually executes
- Processes data
- Responds to input
- Does the work you wanted

By this point, all decisions have been made. There's no compiler, no type checking, just fast machine code running.

## What This Means for Ruby Developers

In Ruby, everything happens at runtime:

```ruby
# Ruby decides at runtime what + means for these objects
def add(a, b)
  a + b
end

# Ruby looks up the method at runtime
user.send(:save)

# Ruby evaluates this string as code at runtime
eval("puts 'hello'")

# Ruby can define methods at runtime
define_method(:greet) { puts "hi" }
```

In Crystal, most things are decided at compile time:

```crystal
# Crystal knows at compile time what + means
def add(a, b)
  a + b
end

# Crystal has already figured out which method to call
user.save

# NO eval - compiler isn't available at runtime
# eval("puts 'hello'")  # Doesn't exist!

# Macros define methods at compile time (more on this later)
macro define_greeter
  def greet
    puts "hi"
  end
end
```

## What You Lose from Ruby

Let's be honest about what Crystal can't do because of compile-time constraints:

### 1. No eval or instance_eval

```ruby
# Ruby: Evaluate arbitrary code at runtime
code = "2 + 2"
result = eval(code)  # => 4

# Crystal: NOPE
# The compiler isn't around at runtime to compile new code
```

### 2. No method_missing (Mostly)

```ruby
# Ruby: Handle unknown methods at runtime
class Magic
  def method_missing(name, *args)
    "You called #{name} with #{args}"
  end
end

Magic.new.anything(1, 2, 3)  # => "You called anything with [1, 2, 3]"
```

Crystal has `method_missing`, but it works differently because it must be resolved at compile time. It's much more limited than Ruby's version.

### 3. No Dynamic Method Definition at Runtime

```ruby
# Ruby: Define methods while program is running
class User
  [:name, :email, :age].each do |attr|
    define_method(attr) do
      instance_variable_get("@#{attr}")
    end
  end
end
```

Crystal must know all methods at compile time. But don't worry - macros can do similar things (Chapter 09).

### 4. No Monkey Patching (Safely)

```ruby
# Ruby: Modify core classes at runtime
class String
  def shout
    upcase + "!!!"
  end
end

"hello".shout  # => "HELLO!!!"
```

Crystal allows reopening classes, but changes apply to the entire compile. You can't conditionally modify classes at runtime.

## What You Gain

The compile-time approach gives you superpowers:

### 1. Type Safety Without Overhead

The compiler checks types, so runtime doesn't need to:

```crystal
def calculate(x : Int32) : Int32
  x * 2
end

# Compile time: Compiler verifies x is Int32, return is Int32
# Runtime: Just multiply and return, no type checking
```

### 2. Dead Code Elimination

The compiler removes code that can never run:

```crystal
def helper
  if false
    expensive_operation()
  end
end

# The compiler sees this is never called and removes it from the binary
```

### 3. Method Dispatch Optimization

Crystal knows exactly which method to call:

```crystal
class Animal
  def speak
    "..."
  end
end

class Dog < Animal
  def speak
    "Woof!"
  end
end

dog = Dog.new
dog.speak  # Compiler knows this is Dog#speak, no lookup needed!
```

In Ruby, this requires a method lookup at runtime. In Crystal, it's often optimized to a direct call.

### 4. Aggressive Inlining

Small methods can be inlined for performance:

```crystal
def square(x)
  x * x
end

result = square(5)

# The compiler might turn this into:
# result = 5 * 5
# No method call overhead!
```

## Understanding Compile-Time Errors

When the compiler complains, it's trying to help. Let's decode common errors:

### Type Mismatch

```crystal
def greet(name : String)
  "Hello, #{name}"
end

greet(42)
```

Error:
```
Error: no overload matches 'greet' with type Int32
```

Translation: "You said greet takes a String, but you gave me an Int32."

### Can't Infer Type

```crystal
arr = []
arr << "hello"
```

Error:
```
Error: for empty arrays use '[] of ElementType'
```

Translation: "I don't know what type this array should hold. Tell me explicitly."

Fix:
```crystal
arr = [] of String
arr << "hello"
```

### Undefined Method

```crystal
value = rand > 0.5 ? "hello" : 42
value.upcase
```

Error:
```
Error: undefined method 'upcase' for Int32
```

Translation: "value could be Int32, and Int32 doesn't have upcase. Handle both cases."

Fix:
```crystal
value = rand > 0.5 ? "hello" : 42
if value.is_a?(String)
  value.upcase
end
```

## The Macro System: Compile-Time Metaprogramming

Crystal can't do runtime metaprogramming, but it has something arguably more powerful: **compile-time metaprogramming** through macros.

A macro is code that runs at compile time and generates more code:

```crystal
macro define_getter(name)
  def {{name}}
    @{{name}}
  end
end

class Person
  def initialize(@name : String, @age : Int32)
  end

  define_getter name
  define_getter age
end

person = Person.new("Alice", 30)
puts person.name  # => "Alice"
puts person.age   # => 30
```

The `define_getter` macro runs at compile time and generates the `name` and `age` methods. By runtime, they're just normal methods.

We'll explore macros deeply in Chapter 09, but for now, know they're Crystal's answer to Ruby's runtime metaprogramming.

## Compile-Time Decisions: A Practical Example

Let's see how compile-time decisions enable performance:

```crystal
# Ruby version: Method lookup at runtime
class Calculator
  def process(operation, a, b)
    case operation
    when :add then a + b
    when :subtract then a - b
    when :multiply then a * b
    when :divide then a / b
    end
  end
end

calc = Calculator.new
result = calc.process(:add, 5, 3)
```

```crystal
# Crystal version: Compiler can optimize based on types
class Calculator
  def add(a : Int32, b : Int32) : Int32
    a + b
  end

  def subtract(a : Int32, b : Int32) : Int32
    a - b
  end

  def multiply(a : Int32, b : Int32) : Int32
    a * b
  end

  def divide(a : Int32, b : Int32) : Int32
    a / b
  end
end

calc = Calculator.new
result = calc.add(5, 3)  # Direct call, no case statement overhead
```

The Crystal version is more verbose but the compiler can optimize it aggressively because it knows exactly what's happening.

## Working With the Compiler

Think of the Crystal compiler as a very picky but helpful teammate:

### Be Explicit When Needed

```crystal
# Compiler struggles
def parse(input)
  input.to_i? || 0  # Return type is Int32 | Int32, simplified to Int32
end

# Help the compiler
def parse(input : String) : Int32
  input.to_i? || 0
end
```

### Use Type Restrictions for Overloading

```crystal
# Define specific behavior for different types
def format(value : Int32) : String
  value.to_s
end

def format(value : Float64) : String
  "%.2f" % value
end

def format(value : String) : String
  "\"#{value}\""
end

puts format(42)        # => "42"
puts format(3.14159)   # => "3.14"
puts format("hello")   # => "\"hello\""
```

The compiler picks the right method at compile time based on the argument type.

### Embrace Type Unions When Appropriate

```crystal
# Sometimes union types are the right answer
def get_config(key : String) : String | Int32 | Bool
  case key
  when "name"
    "MyApp"
  when "port"
    3000
  when "debug"
    true
  else
    ""
  end
end
```

## Real-World Example: A Compile-Time Optimized Parser

```crystal
# A simple expression evaluator that the compiler can optimize
class Expression
  def self.parse(expr : String) : Int32?
    parts = expr.split(' ')
    return nil if parts.size != 3

    left = parts[0].to_i?
    operator = parts[1]
    right = parts[2].to_i?

    return nil if left.nil? || right.nil?

    case operator
    when "+" then left + right
    when "-" then left - right
    when "*" then left * right
    when "/" then right != 0 ? left / right : nil
    else nil
    end
  end
end

# At compile time, Crystal:
# - Knows all types
# - Can inline small methods
# - Can optimize the case statement
# - Can remove impossible code paths

result = Expression.parse("5 + 3")
if result
  puts "Result: #{result}"
end
```

The compiler can optimize this heavily because everything is known at compile time.

## Debugging Compile-Time Issues

### Use `typeof` to Check Types

```crystal
x = [1, 2, 3]
puts typeof(x)  # => Array(Int32)

y = [1, "two", 3.0]
puts typeof(y)  # => Array(Int32 | String | Float64)
```

### Use `-D` Flags for Debug Output

```bash
# See what the compiler is doing
crystal build --debug myprogram.cr

# Print type information
crystal tool types myprogram.cr
```

### Read Compiler Errors Carefully

Compiler errors include:
- What went wrong
- What types were involved
- Where the error occurred
- Sometimes suggestions for fixes

Don't just glance at errors - read them. The compiler is trying to help.

## Mental Model: Two Worlds

**Compile Time World:**
- Types are checked
- Macros expand
- Code is analyzed
- Nothing executes
- All decisions are made

**Runtime World:**
- Code executes
- Data flows
- Results are computed
- No type checking
- No compiler

Your code must satisfy the compile time world before it can enter the runtime world.

## Exercises

1. Write a method that only works with specific types using method overloading
2. Try to create code that would work in Ruby but fails to compile in Crystal - understand why
3. Use `typeof` to explore the types of various expressions
4. Write a simple macro (preview of Chapter 09) that generates getter methods

## What You've Learned

1. Crystal programs have two phases: compile time and runtime
2. Type checking happens at compile time, not runtime
3. Many Ruby features (eval, dynamic method definition) don't work because they require runtime compilation
4. Compile-time decisions enable significant performance optimizations
5. Macros provide compile-time metaprogramming
6. The compiler is strict but makes your programs faster and safer
7. Understanding compile time vs runtime explains many of Crystal's design decisions

## What's Next?

Now that you understand compile time vs runtime, let's tackle one of the most common sources of bugs in any language: nil values. In the next chapter, we'll explore how Crystal's type system makes nil explicit and forces you to handle it properly - saving you from countless `NoMethodError` surprises.

[Continue to Chapter 04 - Nil Safety (No More NoMethodError on nil:NilClass) →](./chapter-04.md)
