# Chapter 09: Macros: The Metaprogramming You Thought You Lost

If you're a Ruby developer, you love metaprogramming. `define_method`, `method_missing`, `class_eval`, `instance_eval` - these are the tools that make Ruby feel magical.

Crystal can't do runtime metaprogramming (the compiler isn't around at runtime). But it has something arguably more powerful: **compile-time metaprogramming** through macros.

Macros run during compilation and generate code. They're like having a Ruby script that writes Crystal code for you, and that code becomes part of your compiled program.

## What Are Macros?

Macros are Crystal code that runs at compile time and outputs Crystal code:

```crystal
macro define_property(name)
  def {{name}}
    @{{name}}
  end

  def {{name}}=(value)
    @{{name}} = value
  end
end

class Person
  define_property name
  define_property age
end

person = Person.new
person.name = "Alice"
person.age = 30
puts person.name  # => "Alice"
```

At compile time, `define_property name` expands to:
```crystal
def name
  @name
end

def name=(value)
  @name = value
end
```

## The `{{` `}}` Syntax

Inside macros, `{{ }}` evaluates and inserts code:

```crystal
macro greet(name)
  "Hello, {{name}}!"
end

puts greet("World")  # => "Hello, World!"
```

The `{{name}}` is replaced with the argument at compile time.

## Macro Methods

Methods can have macro bodies:

```crystal
class Record
  macro define_fields(*names)
    {% for name in names %}
      property {{name}}
    {% end %}

    def initialize({{
                     *names.map { |name| "@#{name}".id }
                   }})
    end
  end
end

class User < Record
  define_fields name, email, age
end

user = User.new("Alice", "alice@example.com", 30)
puts user.name   # => "Alice"
puts user.email  # => "alice@example.com"
```

This generates properties and an initializer at compile time!

## The `{% %}` Syntax

`{% %}` runs code at compile time without outputting anything:

```crystal
macro define_methods(names)
  {% for name in names %}
    def {{name}}
      "Method {{name}} called"
    end
  {% end %}
end

class Example
  define_methods [hello, goodbye, welcome]
end

ex = Example.new
puts ex.hello     # => "Method hello called"
puts ex.goodbye   # => "Method goodbye called"
```

Use `{{ }}` to output code, `{% %}` for control flow.

## Macro Variables vs Constants

Inside macros, you can query compile-time information:

```crystal
macro debug_type(value)
  puts "Type: {{ value.class_name }}"
  {{ value }}
end

x = debug_type(42)
# Outputs at compile time: Type: NumberLiteral
# Returns 42 at runtime
```

## Real-World Example: Property Macro

Let's recreate Ruby's `attr_accessor`:

```crystal
macro property(name)
  def {{name}}
    @{{name}}
  end

  def {{name}}=(value)
    @{{name}} = value
  end
end

macro property(*names)
  {% for name in names %}
    property {{name}}
  {% end %}
end

class Person
  property name, age, email

  def initialize(@name : String, @age : Int32, @email : String)
  end
end

person = Person.new("Alice", 30, "alice@example.com")
puts person.name
person.age = 31
puts person.age
```

This is exactly how Crystal's built-in `property` macro works!

## Macro Conditionals

Execute code conditionally at compile time:

```crystal
macro platform_puts(message)
  {% if flag?(:darwin) %}
    puts "macOS: {{message}}"
  {% elsif flag?(:linux) %}
    puts "Linux: {{message}}"
  {% elsif flag?(:win32) %}
    puts "Windows: {{message}}"
  {% else %}
    puts "Unknown: {{message}}"
  {% end %}
end

platform_puts "Hello!"
# On macOS: "macOS: Hello!"
# On Linux: "Linux: Hello!"
```

The code adapts to the platform at compile time!

## Type Information in Macros

Access type information:

```crystal
macro show_type_info(type)
  puts "Type name: {{ type.name }}"
  puts "Superclass: {{ type.superclass.name }}"
  puts "Instance vars: {{ type.instance_vars }}"
end

class Person
  @name : String
  @age : Int32
end

show_type_info(Person)
```

## Macro for DSLs

Macros enable Domain-Specific Languages:

```crystal
class Router
  macro route(method, path)
    def route_{{method.id}}_{{path.id.gsub(/\//, "_")}}
      puts "{{method.id.upcase}} {{path.id}}"
    end
  end

  route get, "/"
  route get, "/users"
  route post, "/users"
end

router = Router.new
router.route_get_           # => "GET /"
router.route_get__users     # => "GET /users"
router.route_post__users    # => "POST /users"
```

## The `getter` and `setter` Macros

Crystal provides built-in convenience macros:

```crystal
class Person
  getter name       # Read-only
  setter age        # Write-only
  property email    # Read-write

  def initialize(@name : String, @age : Int32, @email : String)
  end
end

person = Person.new("Alice", 30, "alice@example.com")
puts person.name        # Works (getter)
# person.name = "Bob"   # Won't compile (no setter)

person.age = 31         # Works (setter)
# puts person.age       # Won't compile (no getter)

person.email = "new@example.com"  # Works (property has both)
puts person.email                 # Works
```

## Method Hooks with Macros

```crystal
class Model
  macro inherited
    def self.table_name
      {{@type.name.downcase}}
    end
  end
end

class User < Model
end

class Product < Model
end

puts User.table_name     # => "user"
puts Product.table_name  # => "product"
```

The `inherited` macro runs when a class is inherited.

## Real-World Example: Serialization

Let's build JSON serialization with macros:

```crystal
require "json"

module Serializable
  macro included
    def to_json(io : IO)
      io << "{"
      {% for ivar, i in @type.instance_vars %}
        {% if i > 0 %}
          io << ","
        {% end %}
        io << {{ivar.name.stringify}} << ":"
        @{{ivar}}.to_json(io)
      {% end %}
      io << "}"
    end

    def self.from_json(json : String)
      parser = JSON::PullParser.new(json)
      from_json(parser)
    end

    def self.from_json(parser : JSON::PullParser)
      instance = allocate

      parser.read_object do |key|
        case key
        {% for ivar in @type.instance_vars %}
        when {{ivar.name.stringify}}
          instance.@{{ivar}} = {{ivar.type}}.from_json(parser)
        {% end %}
        end
      end

      instance
    end
  end
end

class User
  include Serializable

  def initialize(@name : String, @age : Int32, @email : String)
  end
end

user = User.new("Alice", 30, "alice@example.com")
json = user.to_json
puts json  # => {"name":"Alice","age":30,"email":"alice@example.com"}

parsed = User.from_json(json)
puts parsed.name  # => "Alice"
```

The macro generates serialization code for all instance variables automatically!

## The `record` Macro

Crystal provides a `record` macro for simple data classes:

```crystal
record Point, x : Int32, y : Int32

# Expands to:
struct Point
  getter x : Int32
  getter y : Int32

  def initialize(@x : Int32, @y : Int32)
  end

  # Plus ==, hash, copy, to_s, etc.
end

p = Point.new(3, 4)
puts p.x  # => 3
puts p    # => Point(@x=3, @y=4)
```

## Debugging Macros

See what macros generate:

```crystal
macro debug_macro
  {% puts "This prints at compile time!" %}
  puts "This prints at runtime!"
end

debug_macro
```

Or use the `-e` flag to see macro expansion:

```bash
crystal tool expand myfile.cr
```

## Macro Expressions

Macros can perform computations:

```crystal
macro fibonacci(n)
  {% if n <= 1 %}
    {{n}}
  {% else %}
    fibonacci({{n - 1}}) + fibonacci({{n - 2}})
  {% end %}
end

puts fibonacci(10)  # Computed at compile time!
```

Be careful - complex macro recursion can slow compilation.

## The verbatim Macro

Sometimes you need to output code without interpolation:

```crystal
macro make_literal_method
  def get_brackets
    {% verbatim do %}
      "{{ these braces are literal }}"
    {% end %}
  end
end

class Example
  make_literal_method
end

puts Example.new.get_brackets  # => "{{ these braces are literal }}"
```

## Comparing to Ruby Metaprogramming

| Ruby | Crystal |
|------|---------|
| `define_method` (runtime) | `macro` (compile time) |
| `method_missing` (runtime) | Limited support |
| `instance_eval` (runtime) | N/A |
| `class_eval` (runtime) | Macros |
| `const_get` (runtime lookup) | Type info in macros |

The key difference: Ruby metaprogramming happens while your program runs. Crystal metaprogramming happens while your program compiles.

## Advanced: Macro Annotations

Define custom annotations:

```crystal
annotation Required
end

macro validate(object)
  {% for ivar in object.type.instance_vars %}
    {% if ivar.annotation(Required) %}
      raise "{{ivar}} is required" if {{object}}.@{{ivar}}.nil?
    {% end %}
  {% end %}
end

class Form
  @Required
  property name : String?
  property age : Int32?

  def initialize(@name = nil, @age = nil)
  end

  def validate
    validate(self)
  end
end

form = Form.new
form.validate  # Raises: "name is required"
```

## Best Practices

1. **Keep macros simple** - Complex macros slow compilation
2. **Document generated code** - Explain what the macro does
3. **Test macro output** - Ensure generated code is correct
4. **Prefer methods over macros** - Use macros when necessary, not by default
5. **Use `crystal tool expand`** - See what your macros generate

## When to Use Macros

**Use macros for:**
- Generating repetitive code (getters, setters)
- DSLs (routing, configuration)
- Compile-time configuration
- Code that adapts to types
- Zero-cost abstractions

**Don't use macros for:**
- Things normal methods can do
- Complex logic (do it at runtime)
- Anything that makes code hard to understand

## Exercises

1. Write a macro that generates methods for all arithmetic operations
2. Create a `delegate` macro that forwards methods to another object
3. Build a simple validation framework using macros and annotations
4. Implement a `enum_methods` macro that generates a method for each enum value
5. Create a macro that logs method entry/exit at compile time

## What You've Learned

1. Macros run at compile time and generate code
2. `{{ }}` inserts code, `{% %}` runs control flow
3. Macros can access type information
4. `property`, `getter`, `setter` are built-in macros
5. Macros enable DSLs and metaprogramming
6. Crystal's metaprogramming is compile-time, not runtime
7. The `inherited` hook runs when classes are inherited

## What's Next?

You've now mastered Crystal's macro system - the feature that brings Ruby-like flexibility to a compiled language. In the final chapter, we'll tie everything together by exploring performance: benchmarking, profiling, optimization techniques, and understanding why Crystal is so fast. Get ready to see just how much faster than Ruby your code can be.

[Continue to Chapter 10 - Performance: Making Ruby Devs Cry (Tears of Joy) →](./chapter-10.md)
