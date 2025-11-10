# Chapter 05: Structs vs Classes: Speed Dating Edition

In Ruby, everything is a class (well, almost). Want to represent a point? Class. A coordinate? Class. A color? Class. Ruby doesn't give you a choice.

Crystal says: "What if small, simple data could be stored differently - faster and more memory-efficient?" Enter structs.

Structs are **value types** (passed by value), while classes are **reference types** (passed by reference). This distinction doesn't exist in Ruby, and it fundamentally changes how you write certain kinds of code.

## Classes: What You Already Know

Classes work mostly like Ruby:

```crystal
class Person
  property name : String
  property age : Int32

  def initialize(@name, @age)
  end

  def greet
    "Hi, I'm #{@name}"
  end
end

alice = Person.new("Alice", 30)
bob = alice  # bob points to the same object as alice

bob.name = "Bob"
puts alice.name  # => "Bob" (same object!)
```

Classes are **reference types**:
- Creating an instance allocates heap memory
- Variables hold references (pointers) to the object
- Assigning one variable to another copies the reference
- Both variables point to the same object
- Changes through one variable affect the other

## Structs: The New Kid

Structs look similar but behave differently:

```crystal
struct Point
  property x : Int32
  property y : Int32

  def initialize(@x, @y)
  end

  def distance_from_origin
    Math.sqrt(x**2 + y**2)
  end
end

p1 = Point.new(3, 4)
p2 = p1  # p2 is a COPY of p1

p2.x = 10
puts p1.x  # => 3 (different object!)
puts p2.x  # => 10
```

Structs are **value types**:
- Stored directly (often on the stack)
- Variables hold the actual value, not a reference
- Assigning one variable to another copies the entire value
- Each variable has its own independent copy
- Changes to one don't affect the other

## The Key Difference: Copy vs Reference

### Classes: Shared Reference

```crystal
class Box
  property value : Int32
  def initialize(@value)
  end
end

box1 = Box.new(42)
box2 = box1        # Both reference same object

box2.value = 100
puts box1.value    # => 100 (same object)
puts box2.value    # => 100
```

### Structs: Independent Copies

```crystal
struct Box
  property value : Int32
  def initialize(@value)
  end
end

box1 = Box.new(42)
box2 = box1        # box2 is a copy

box2.value = 100
puts box1.value    # => 42 (different object)
puts box2.value    # => 100
```

## When to Use Structs

Use structs for:

### 1. Small, Immutable-ish Data

```crystal
struct Color
  getter red : UInt8
  getter green : UInt8
  getter blue : UInt8

  def initialize(@red, @green, @blue)
  end

  def to_hex : String
    "#%02x%02x%02x" % {red, green, blue}
  end
end

color = Color.new(255, 0, 0)
puts color.to_hex  # => "#ff0000"
```

Perfect for RGB colors, coordinates, dimensions, etc.

### 2. Mathematical Value Types

```crystal
struct Vector3
  property x : Float64
  property y : Float64
  property z : Float64

  def initialize(@x, @y, @z)
  end

  def +(other : Vector3) : Vector3
    Vector3.new(x + other.x, y + other.y, z + other.z)
  end

  def magnitude : Float64
    Math.sqrt(x**2 + y**2 + z**2)
  end
end

v1 = Vector3.new(1.0, 2.0, 3.0)
v2 = Vector3.new(4.0, 5.0, 6.0)
v3 = v1 + v2
puts v3.magnitude
```

### 3. Performance-Critical Small Objects

```crystal
# Struct: fast, stack-allocated
struct Position
  property row : Int32
  property col : Int32

  def initialize(@row, @col)
  end
end

# Processing millions of positions? Struct is faster
positions = Array.new(1_000_000) { Position.new(rand(100), rand(100)) }
```

### 4. C Interop (Chapter 08)

Structs map directly to C structs, making them essential for FFI.

## When to Use Classes

Use classes for:

### 1. Objects with Identity

```crystal
class User
  property name : String
  property email : String

  def initialize(@name, @email)
  end
end

# Each user is a unique entity with identity
user1 = User.new("Alice", "alice@example.com")
user2 = User.new("Alice", "alice@example.com")

# Same data, but different objects
puts user1.object_id != user2.object_id  # => true
```

### 2. Mutable State

```crystal
class Counter
  property count : Int32

  def initialize(@count = 0)
  end

  def increment
    @count += 1
  end
end

counter = Counter.new
counter.increment
counter.increment
puts counter.count  # => 2
```

With a struct, this would be awkward because you'd need to reassign:

```crystal
struct Counter  # Don't do this!
  property count : Int32

  def initialize(@count = 0)
  end

  def increment
    @count += 1  # Changes the copy!
  end
end

counter = Counter.new
counter.increment  # Changes counter... but you need to reassign
counter = counter  # Wait, what?
```

### 3. Complex Objects with Behavior

```crystal
class Database
  def initialize(@connection_string : String)
    @connected = false
  end

  def connect
    # Complex connection logic
    @connected = true
  end

  def query(sql : String)
    raise "Not connected" unless @connected
    # Execute query
  end
end
```

### 4. Inheritance Hierarchies

```crystal
abstract class Animal
  abstract def speak : String
end

class Dog < Animal
  def speak : String
    "Woof!"
  end
end

class Cat < Animal
  def speak : String
    "Meow!"
  end
end
```

**Structs cannot inherit!** Only classes can be part of inheritance hierarchies.

## Performance Implications

### Memory Allocation

```crystal
# Struct: often stack-allocated (very fast)
struct Point
  property x : Int32
  property y : Int32
  def initialize(@x, @y); end
end

# Class: heap-allocated (slower, but flexible)
class Point
  property x : Int32
  property y : Int32
  def initialize(@x, @y); end
end
```

Stack allocation is faster because:
- No garbage collector involvement
- Better cache locality
- No allocation overhead

### Copying Overhead

```crystal
# Struct: copied on assignment
struct LargeStruct
  property data : StaticArray(Int32, 1000)
  # 4000 bytes copied on each assignment!
end

# Class: only reference copied
class LargeClass
  property data : Array(Int32)
  # Only 8 bytes (pointer) copied on assignment
end
```

**Rule of thumb:** Keep structs small. If your struct is more than 16-32 bytes, consider using a class.

## Passing Arguments

### Structs Are Passed by Value

```crystal
struct Point
  property x : Int32
  property y : Int32
  def initialize(@x, @y); end
end

def move_point(point : Point)
  point.x += 10  # Modifies the copy
end

p = Point.new(5, 5)
move_point(p)
puts p.x  # => 5 (original unchanged)
```

To modify the original, pass by reference:

```crystal
def move_point(point : Point*)
  point.value.x += 10  # Modifies through pointer
end

p = Point.new(5, 5)
move_point(pointerof(p))
puts p.x  # => 15 (original changed)
```

(Pointers are advanced - you usually won't need them unless doing C interop)

### Classes Are Passed by Reference

```crystal
class Box
  property value : Int32
  def initialize(@value); end
end

def modify_box(box : Box)
  box.value = 100  # Modifies the original
end

b = Box.new(42)
modify_box(b)
puts b.value  # => 100 (original changed)
```

## Equality Semantics

### Structs: Value Equality

```crystal
struct Point
  property x : Int32
  property y : Int32
  def initialize(@x, @y); end
end

p1 = Point.new(3, 4)
p2 = Point.new(3, 4)
puts p1 == p2  # => true (same values)
```

Structs automatically get `==` based on their fields.

### Classes: Reference Equality (By Default)

```crystal
class Point
  property x : Int32
  property y : Int32
  def initialize(@x, @y); end
end

p1 = Point.new(3, 4)
p2 = Point.new(3, 4)
puts p1 == p2  # => false (different objects)

# Need to define == for value equality
class Point
  def ==(other : Point)
    x == other.x && y == other.y
  end
end

p1 = Point.new(3, 4)
p2 = Point.new(3, 4)
puts p1 == p2  # => true (custom equality)
```

## Real-World Example: Game Entities

```crystal
# Value type: position is just data
struct Position
  property x : Float64
  property y : Float64

  def initialize(@x, @y)
  end

  def distance_to(other : Position) : Float64
    dx = x - other.x
    dy = y - other.y
    Math.sqrt(dx**2 + dy**2)
  end
end

# Reference type: entity has identity and state
class Entity
  property position : Position
  property health : Int32
  property name : String

  def initialize(@name, @position, @health = 100)
  end

  def move_to(new_position : Position)
    @position = new_position
  end

  def take_damage(amount : Int32)
    @health -= amount
  end

  def alive? : Bool
    @health > 0
  end
end

# Usage
player = Entity.new("Player", Position.new(0.0, 0.0))
enemy = Entity.new("Enemy", Position.new(10.0, 10.0))

# Position is a struct: cheap to copy and compare
distance = player.position.distance_to(enemy.position)
puts "Distance: #{distance}"

# Entity is a class: maintains identity
player.move_to(Position.new(5.0, 5.0))
player.take_damage(20)
puts "Player health: #{player.health}"
```

Notice:
- `Position` is a struct: small, immutable-ish, just data
- `Entity` is a class: has identity, mutable state, behavior

## Converting Between Struct and Class

Sometimes you start with one and realize you need the other:

```crystal
# Started as struct
struct Point
  property x : Int32
  property y : Int32
  def initialize(@x, @y); end
end

# Realize you need reference semantics
class Point
  property x : Int32
  property y : Int32
  def initialize(@x, @y); end
end
```

Just change `struct` to `class`. The syntax is the same, but the behavior changes.

## Common Pitfalls

### 1. Mutable Struct Fields

```crystal
struct Container
  property items : Array(String)
  def initialize(@items)
  end
end

c1 = Container.new(["apple"])
c2 = c1  # Copy the struct

c2.items << "banana"
puts c1.items  # => ["apple", "banana"] (WAT?)
```

The struct was copied, but the Array is a class (reference type), so both structs point to the same array!

**Lesson:** Structs with reference-type fields still share those references.

### 2. Large Structs

```crystal
struct HugeStruct
  property data1 : StaticArray(Int32, 1000)
  property data2 : StaticArray(Int32, 1000)
  property data3 : StaticArray(Int32, 1000)
  # 12,000 bytes!
end

def process(hs : HugeStruct)  # Copies 12KB on each call!
  # ...
end
```

**Lesson:** Keep structs small or use classes for large data.

### 3. Struct in Array/Hash

```crystal
struct Mutable
  property value : Int32
  def initialize(@value); end
end

arr = [Mutable.new(1), Mutable.new(2)]
arr[0].value = 100  # Modifies a copy!
puts arr[0].value   # => 1 (unchanged)

# To modify in array:
item = arr[0]
item.value = 100
arr[0] = item       # Must reassign
```

**Lesson:** Modifying structs in collections requires reassignment.

## Decision Guide

**Use a struct when:**
- Small data (< 32 bytes)
- Immutable or rarely changes
- Value semantics make sense
- Performance critical
- No inheritance needed
- Pure data, minimal behavior

**Use a class when:**
- Needs identity
- Mutable state
- Inheritance required
- Larger than 32 bytes
- Complex behavior
- Traditional OOP patterns

**When in doubt, use a class.** You can always change to a struct later if profiling shows it's worth it.

## Exercises

1. Implement a `Rectangle` struct with width and height, and methods for area and perimeter
2. Create a `Player` class that has a `Position` struct property
3. Write a benchmark comparing struct vs class for 1 million allocations
4. Identify which should be structs vs classes: Email, ShoppingCart, Coordinate, UserSession, RGB

## What You've Learned

1. Structs are value types (passed by value), classes are reference types (passed by reference)
2. Structs are copied on assignment, classes share references
3. Structs are great for small, data-focused types
4. Classes are better for objects with identity and mutable state
5. Structs can't inherit, classes can
6. Structs get automatic value-based equality
7. Keep structs small for best performance

## What's Next?

You now understand the fundamental types Crystal offers. Next, we'll explore concurrency - one of Crystal's killer features. Crystal's concurrency model uses fibers and channels to make concurrent programming approachable and safe. Get ready to write code that does multiple things at once without the usual thread-related headaches.

[Continue to Chapter 06 - Concurrency: Fibers, Channels, and Why You'll Sleep Better →](./chapter-06.md)
