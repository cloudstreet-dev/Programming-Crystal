# Chapter 04: Nil Safety (No More NoMethodError on nil:NilClass)

If you've written Ruby for any length of time, you've seen this error:

```ruby
NoMethodError: undefined method `upcase' for nil:NilClass
```

It's the error that shows up at 3 AM in production. The error that makes you add defensive `if something` checks everywhere. The error that Tony Hoare called his "billion dollar mistake."

Crystal says: what if we made nil explicit? What if the compiler forced you to handle nil cases? What if you could never accidentally call a method on nil?

Welcome to nil safety.

## The Ruby nil Problem

In Ruby, almost anything can be nil:

```ruby
def find_user(id)
  # Might return a user, might return nil
  User.find(id)
end

user = find_user(123)
puts user.name.upcase  # Works... or crashes

# Better:
if user
  puts user.name.upcase
end

# But you have to remember to check!
```

The problem: nothing forces you to check for nil. The method signature doesn't tell you if it returns nil. You have to read the docs, read the code, or wait for the crash.

## Crystal's Solution: Nilable Types

In Crystal, nil is part of the type system:

```crystal
# This can only be String
name : String = "Alice"
name = nil  # Compile error!

# This can be String OR Nil
name : String? = "Alice"
name = nil  # OK!
```

The `?` suffix means "this type or nil". `String?` is shorthand for `String | Nil`.

Now the type tells you: "this might be nil, handle it."

## Working with Nilable Types

You can't just use a nilable value directly:

```crystal
def find_user(id : Int32) : String?
  # Simulate database lookup
  if id > 0
    "User #{id}"
  else
    nil
  end
end

name = find_user(123)
# name is String?

puts name.upcase  # Won't compile!
# Error: undefined method 'upcase' for Nil
```

The compiler knows `name` might be nil, and nil doesn't have an `upcase` method.

### Solution 1: Check with if

```crystal
name = find_user(123)

if name
  # Inside here, compiler knows name is String (not nil)
  puts name.upcase
else
  puts "User not found"
end
```

The compiler is smart about flow control. Inside the `if name` block, it knows `name` can't be nil, so it treats it as `String`.

### Solution 2: Use the `try` Method

```crystal
name = find_user(123)
puts name.try(&.upcase)  # Returns nil if name is nil, otherwise calls upcase
```

The `try` method only executes the block if the value isn't nil. If it is nil, it returns nil.

### Solution 3: Provide a Default

```crystal
name = find_user(123)
puts (name || "Unknown").upcase
```

If `name` is nil, use "Unknown" instead. Now the expression is always a String.

### Solution 4: Use `not_nil!` (Danger Zone)

```crystal
name = find_user(123)
puts name.not_nil!.upcase  # Runtime error if name is nil!
```

The `not_nil!` method tells the compiler "trust me, this isn't nil." If you're wrong, you get a runtime exception. Use sparingly.

## Type Narrowing in Detail

Crystal's compiler tracks types through your code:

```crystal
value : String? = "hello"

# Before the check: value is String?
if value
  # Inside if: value is String
  puts value.upcase
  puts value.size
else
  # Inside else: value is Nil
  puts "Value was nil"
end
# After the check: value is String? again
```

This is called **type narrowing** or **flow-sensitive typing**. The compiler narrows the type based on what it knows at each point.

### Works with is_a? Checks

```crystal
value : String | Int32 = "hello"

if value.is_a?(String)
  # value is String here
  puts value.upcase
elsif value.is_a?(Int32)
  # value is Int32 here
  puts value * 2
end
```

### Works with responds_to?

```crystal
value : String | Int32 = "hello"

if value.responds_to?(:upcase)
  # value responds to upcase here
  puts value.upcase
end
```

### Works with Multiple Conditions

```crystal
user_name : String? = get_user_name()
admin : Bool = is_admin()

if user_name && admin
  # user_name is definitely String here (not nil)
  # admin is definitely true
  puts "Admin: #{user_name.upcase}"
end
```

## Nilable Method Returns

When should a method return a nilable type?

```crystal
# Method that might not find something
def find_config(key : String) : String?
  case key
  when "host"
    "localhost"
  when "port"
    "3000"
  else
    nil  # Unknown config key
  end
end

# Method that always returns something
def get_config(key : String) : String
  find_config(key) || "default"
end

# Usage:
if config = find_config("host")
  puts "Host: #{config}"
end

puts "Port: #{get_config("port")}"  # No nil check needed
```

**Rule of thumb:** If a method might not have a meaningful return value, return a nilable type. If it always returns something, don't use nilable.

## Arrays and Nilable Elements

Array access can return nil:

```crystal
arr = [1, 2, 3]

# [] returns nilable type (might be out of bounds)
first = arr[0]?           # Int32?
puts typeof(first)        # Int32 | Nil

# []without ? raises on out of bounds
definitely_first = arr[0] # Int32
puts typeof(definitely_first)  # Int32

# But this would raise:
# arr[10]  # IndexError
```

Use `[]?` when access might fail, use `[]` when you're certain it won't.

## Hashes and Nilable Values

Hash access also returns nilable:

```crystal
config = {"host" => "localhost", "port" => 3000}

# Hash access returns nilable
host = config["host"]?     # String | Int32 | Nil
puts typeof(host)

# Without ? raises if key doesn't exist
port = config["port"]      # String | Int32
puts typeof(port)

# Better: check before using
if host = config["host"]?
  if host.is_a?(String)
    puts "Connecting to #{host}"
  end
end
```

## The ||= Operator with Nils

The `||=` operator is nil-aware:

```crystal
name : String? = nil
name ||= "Default"
puts name  # => "Default"

name = "Alice"
name ||= "Default"
puts name  # => "Alice"
```

It only assigns if the current value is nil or false.

## Real-World Example: Safe User Lookup

```crystal
class User
  property name : String
  property email : String

  def initialize(@name, @email)
  end
end

class UserRepository
  def initialize
    @users = {} of Int32 => User
    @users[1] = User.new("Alice", "alice@example.com")
    @users[2] = User.new("Bob", "bob@example.com")
  end

  def find(id : Int32) : User?
    @users[id]?
  end

  def find!(id : Int32) : User
    @users[id]? || raise "User #{id} not found"
  end

  def find_email(id : Int32) : String?
    find(id).try(&.email)
  end
end

repo = UserRepository.new

# Safe lookup with explicit handling
if user = repo.find(1)
  puts "Found: #{user.name} (#{user.email})"
else
  puts "User not found"
end

# Using try for chaining
puts "Email: #{repo.find(1).try(&.email) || "unknown"}"

# Using ! version when you're certain
user = repo.find!(1)
puts "Name: #{user.name}"

# This would raise at runtime:
# user = repo.find!(999)
```

Notice the pattern:
- `find` returns `User?` - might return nil
- `find!` returns `User` - raises if not found
- `find_email` returns `String?` - chains with `try`

This pattern is common in Crystal: safe version returns nilable, bang version raises.

## The nil? Method

Check explicitly for nil:

```crystal
value : String? = "hello"

if value.nil?
  puts "It's nil"
else
  # value is String here
  puts value.upcase
end

# Equivalent to:
if !value
  puts "It's nil"
else
  puts value.upcase
end
```

Both work, but `nil?` is more explicit about what you're checking.

## Nilable in Structs and Classes

```crystal
class Person
  property name : String
  property nickname : String?  # Optional
  property age : Int32

  def initialize(@name : String, @age : Int32, @nickname : String? = nil)
  end

  def display_name : String
    nickname || name
  end
end

person = Person.new("Alice Smith", 30)
puts person.display_name  # => "Alice Smith"

person.nickname = "Ally"
puts person.display_name  # => "Ally"
```

Use nilable properties when a field is optional. Provide defaults in the initializer.

## The Billion Dollar Fix

Tony Hoare invented null references in 1965 and later called it his "billion dollar mistake" because of all the crashes it caused. Crystal doesn't eliminate nil, but it makes it impossible to forget about it.

In Ruby:
```ruby
def process(user)
  # Crash waiting to happen
  user.name.upcase
end
```

In Crystal:
```crystal
def process(user : User?)
  # Won't compile unless you handle nil
  user.try(&.name.upcase)
end
```

The compiler forces you to deal with nil. You can't ignore it. You can't forget. You must handle it.

## Common Patterns

### Pattern 1: Early Return

```crystal
def process_user(id : Int32)
  user = find_user(id)
  return unless user  # Exit early if nil

  # user is guaranteed String here
  puts "Processing: #{user.upcase}"
end
```

### Pattern 2: Guard Clause

```crystal
def process_user(id : Int32)
  user = find_user(id)
  unless user
    puts "User not found"
    return
  end

  # user is String here
  puts user.upcase
end
```

### Pattern 3: Default Value

```crystal
def get_name(id : Int32) : String
  find_user(id) || "Guest"
end
```

### Pattern 4: try with Chain

```crystal
# Chain multiple calls safely
result = find_user(123)
  .try(&.profile)
  .try(&.bio)
  .try(&.upcase)

# Returns nil if any step is nil
```

## Exercises

1. Write a `find_by_email` method that returns a nilable User
2. Create a class with both required and optional properties
3. Write a method that safely navigates a nested hash structure
4. Implement a safe array access method with default values

## What You've Learned

1. Nil is part of Crystal's type system via nilable types (`String?`)
2. You must explicitly handle nil cases before using a value
3. Type narrowing lets the compiler track what's nil and what isn't
4. The `try` method safely calls methods on nilable values
5. Methods returning nilable types signal "this might not have a value"
6. The `!` suffix convention indicates "raises if fails"
7. Nil safety prevents an entire class of runtime errors

## What's Next?

You now understand how Crystal handles nil. Next, we need to talk about another fundamental choice: structs vs classes. Crystal gives you value types (structs) and reference types (classes), and choosing the right one affects both performance and behavior. Let's explore when to use each.

[Continue to Chapter 05 - Structs vs Classes: Speed Dating Edition →](./chapter-05.md)
