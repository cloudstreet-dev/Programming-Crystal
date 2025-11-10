# Chapter 02: Types: The Plot Twist

Remember in Chapter 01 when Crystal felt comfortably familiar, like Ruby with a few quirks? Well, buckle up. We're about to explore the type system - the feature that makes Crystal fundamentally different from Ruby, and the reason your code runs so fast.

If you're a Ruby developer, you're used to duck typing and the freedom to do whatever you want, whenever you want. Crystal says "that's great, but what if we caught your bugs at compile time instead of production?"

## The Type System: Your New Best Friend

In Ruby, types are implicit and checked at runtime:

```ruby
def add(a, b)
  a + b
end

add(2, 3)        # => 5
add("hello", "world")  # => "helloworld"
add(2, "three")  # => TypeError at runtime (oops)
```

In Crystal, the compiler figures out types and validates them before your code runs:

```crystal
def add(a, b)
  a + b
end

add(2, 3)        # => 5
add("hello", " world")  # => "hello world"
# add(2, "three")  # Won't compile! Type error caught before runtime
```

The magic? Crystal's **type inference**. You didn't write any type annotations, but the compiler figured out what types are valid.

## Type Inference: The Compiler Is Watching

Crystal's compiler is like a really attentive friend who remembers everything you've ever said and calls you out on inconsistencies.

```crystal
x = 42
# Compiler: "Okay, x is an Int32"

x = "hello"
# Compiler: "Wait, you said x was Int32, now it's String? Nope."
```

This will fail to compile with an error like:
```
Error: type must be Int32, not String
```

Once a variable has a type, it keeps that type. No shapeshifting allowed.

### Inference in Action

```crystal
# Simple inference
name = "Crystal"              # String
count = 42                    # Int32
price = 99.99                 # Float64
active = true                 # Bool

# Array inference
numbers = [1, 2, 3]           # Array(Int32)
strings = ["a", "b", "c"]     # Array(String)

# Hash inference
person = {"name" => "Alice", "age" => 30}
# Hash(String, String | Int32)  <- We'll explain this shortly!

# Method return type inference
def double(x)
  x * 2
end
# The compiler infers this returns the same type as x
```

The compiler traces through your entire program, figuring out types and making sure everything is consistent. It's exhausting for the compiler, but convenient for you.

## Explicit Type Annotations

Sometimes you want to be explicit. Maybe for documentation, maybe to catch bugs, maybe to help the compiler:

```crystal
# Explicit variable types
name : String = "Crystal"
age : Int32 = 13
numbers : Array(Int32) = [1, 2, 3]

# Method parameter types
def greet(name : String)
  "Hello, #{name}!"
end

# Method return types
def calculate_score(points : Int32) : Float64
  points * 1.5
end

# Full method signature
def process(input : String, count : Int32) : Array(String)
  Array.new(count, input)
end
```

When should you use explicit types?
- **Public API methods**: Make your interface clear
- **Complex logic**: Help the compiler (and yourself) understand intent
- **Performance-critical code**: Sometimes helps the compiler optimize
- **When inference fails**: Occasionally the compiler needs a hint

## Union Types: Multiple Personalities

Here's where Crystal gets interesting. What if a variable could be multiple types?

```crystal
# This variable could be String OR Int32
value = rand > 0.5 ? "hello" : 42
# Type: (String | Int32)
```

That `|` means "union type" - the variable is String OR Int32. This is Crystal's way of handling situations where Ruby would just wing it.

### Working with Union Types

You can't just use union types directly without checking:

```crystal
value = rand > 0.5 ? "hello" : 42

# This won't compile:
# puts value.upcase
# Error: undefined method 'upcase' for Int32

# You need to check the type first:
if value.is_a?(String)
  puts value.upcase
elsif value.is_a?(Int32)
  puts value * 2
end
```

The compiler is smart enough to know that inside the `if value.is_a?(String)` block, `value` is definitely a String. This is called **type narrowing**.

### Union Types in Hashes and Arrays

```crystal
# Array with mixed types
mixed = [1, "two", 3.0]
# Type: Array(Int32 | String | Float64)

# Hash with mixed values
person = {"name" => "Alice", "age" => 30, "height" => 5.6}
# Type: Hash(String, String | Int32 | Float64)

# Accessing these requires type checking
person["name"].upcase          # Won't compile!

if name = person["name"]
  if name.is_a?(String)
    puts name.upcase
  end
end
```

This might feel tedious compared to Ruby's "just do it" approach, but it prevents an entire class of runtime errors.

## Nil: The Billion Dollar Mistake (Solved)

Tony Hoare, who invented null references, called them his "billion dollar mistake." Crystal tries to fix this.

In Ruby:
```ruby
name = nil
puts name.upcase  # NoMethodError: undefined method `upcase' for nil:NilClass
```

In Crystal, `nil` is part of the type system:

```crystal
# This is actually String | Nil
name : String? = "Alice"
name = nil

# This won't compile:
# puts name.upcase
# Error: undefined method 'upcase' for Nil

# You must handle nil explicitly:
if name
  puts name.upcase  # Safe! name is definitely String here
end
```

The `String?` syntax is shorthand for `String | Nil`. The compiler forces you to handle the nil case.

### The `try` Method

Crystal gives you a convenient method for working with nilable values:

```crystal
name : String? = "Alice"
puts name.try(&.upcase)  # => "ALICE"

name = nil
puts name.try(&.upcase)  # => nil (no error!)
```

The `try` method only calls the block if the value isn't nil. This is similar to Ruby's `&.` operator but built into the type system.

## Type Restrictions and Method Overloading

Crystal allows you to define multiple methods with the same name but different parameter types:

```crystal
def process(value : String)
  "Processing string: #{value}"
end

def process(value : Int32)
  "Processing number: #{value * 2}"
end

def process(value : Array(String))
  "Processing array of #{value.size} strings"
end

puts process("hello")        # => "Processing string: hello"
puts process(42)             # => "Processing number: 84"
puts process(["a", "b"])     # => "Processing array of 2 strings"
```

The compiler picks the right method based on the argument type. This is called **method overloading**, and it's impossible in Ruby.

## Generics: Types That Work with Types

What if you want to write a method that works with any type? Enter generics.

```crystal
# A simple generic method
def first(array : Array(T)) : T forall T
  array[0]
end

numbers = [1, 2, 3]
puts first(numbers)           # => 1 (returns Int32)

strings = ["a", "b", "c"]
puts first(strings)           # => "a" (returns String)
```

That `T forall T` means "T can be any type, figure it out from the array."

### Generic Classes

```crystal
class Box(T)
  def initialize(@value : T)
  end

  def get : T
    @value
  end

  def set(@value : T)
  end
end

# Create a box for integers
int_box = Box(Int32).new(42)
puts int_box.get              # => 42

# Create a box for strings
string_box = Box(String).new("hello")
puts string_box.get           # => "hello"

# This won't compile:
# int_box.set("hello")
# Error: expected Int32, got String
```

Generics let you write reusable code while maintaining type safety.

## Type Aliases: Naming Your Types

For complex types, you can create aliases:

```crystal
# Give a complex type a simple name
alias StringOrInt = String | Int32
alias NameMap = Hash(String, String)
alias IntArray = Array(Int32)

def process(value : StringOrInt)
  # ...
end

names : NameMap = {"first" => "Alice", "last" => "Smith"}
numbers : IntArray = [1, 2, 3]
```

This makes your code more readable and easier to refactor.

## The typeof Operator

You can ask Crystal about types at compile time:

```crystal
x = 42
puts typeof(x)                # => Int32

y = [1, 2, 3]
puts typeof(y)                # => Array(Int32)

z = {"key" => 123}
puts typeof(z)                # => Hash(String, Int32)

# Even works with expressions
puts typeof(1 + 2)            # => Int32
puts typeof("hello" + " world")  # => String
```

This is useful for debugging and understanding what the compiler sees.

## Real-World Example: A Type-Safe Cache

Let's build something practical that shows off Crystal's type system:

```crystal
class Cache(K, V)
  def initialize
    @storage = {} of K => V
  end

  def set(key : K, value : V) : V
    @storage[key] = value
  end

  def get(key : K) : V?
    @storage[key]?
  end

  def has_key?(key : K) : Bool
    @storage.has_key?(key)
  end

  def delete(key : K) : V?
    @storage.delete(key)
  end

  def size : Int32
    @storage.size
  end
end

# Create a cache for string keys and integer values
cache = Cache(String, Int32).new

cache.set("score", 100)
cache.set("level", 5)

if score = cache.get("score")
  puts "Current score: #{score}"
end

# This won't compile:
# cache.set("name", "Alice")  # Error: expected Int32, got String
# cache.set(123, "value")     # Error: expected String, got Int32

puts "Cache size: #{cache.size}"
```

Notice:
- Generic types `K` and `V` for keys and values
- Return type `V?` for methods that might return nil
- Type safety prevents putting wrong types in the cache
- The `{}` of `K => V` syntax initializes an empty typed hash

## Common Type Pitfalls

### 1. Unexpected Union Types

```crystal
def get_value(use_string : Bool)
  if use_string
    "hello"
  else
    42
  end
end

result = get_value(true)
# Type is String | Int32, not String!
# result.upcase  # Won't compile

# Fix with type restriction:
def get_string(use_default : Bool) : String
  if use_default
    "default"
  else
    "custom"
  end
end
```

### 2. Array Type Mismatches

```crystal
# These are different types!
arr1 = [1, 2, 3]                    # Array(Int32)
arr2 = [1, 2, 3] of Int64           # Array(Int64)

# This won't compile:
# arr1.concat(arr2)

# Fix by being explicit:
arr1_64 = [1_i64, 2_i64, 3_i64]     # Array(Int64)
arr1_64.concat(arr2)                # Works!
```

### 3. Type Inference Limitations

```crystal
# This won't compile:
# empty = []
# Error: can't infer type of empty array

# Fix by specifying the type:
empty = [] of String
empty = Array(String).new
```

## Type System Mental Model

Here's how to think about Crystal's type system:

1. **Everything has a type** - The compiler knows the type of every variable and expression
2. **Types are checked at compile time** - Mistakes are caught before your code runs
3. **Type inference is your friend** - You usually don't need to write types explicitly
4. **Union types handle uncertainty** - When a value could be multiple types, unions track all possibilities
5. **Nil is explicit** - `String?` means "String or Nil" - you must handle both cases
6. **Generics enable reusability** - Write code once that works with many types

## Exercises

1. Write a generic `Pair` class that holds two values of potentially different types
2. Create a method that accepts either a string or an array of strings and returns the total character count
3. Build a type-safe `Result` class that can hold either a success value or an error message
4. Write a function that demonstrates method overloading with at least three different type signatures

## What You've Learned

1. Crystal's type system catches bugs at compile time
2. Type inference means you don't always need explicit types
3. Union types (`String | Int32`) handle multiple possibilities
4. Nil is part of the type system (`String?` means `String | Nil`)
5. Method overloading lets you define multiple methods with different types
6. Generics enable type-safe reusable code
7. Type narrowing (`is_a?` checks) tells the compiler about types in specific contexts

## What's Next?

Now that you understand types, we need to talk about compile time versus runtime. Why do some Ruby features not work in Crystal? Why can't you just `eval` a string? In the next chapter, we'll explore the fundamental difference between interpreted and compiled languages, and why it matters.

[Continue to Chapter 03 - Compile Time vs Runtime: The Great Divide →](./chapter-03.md)
