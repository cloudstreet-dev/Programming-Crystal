# Chapter 08: C Bindings: When You Need to Get Down and Dirty

Ruby has FFI (Foreign Function Interface) for calling C code. It works, but it's not particularly elegant or fast.

Crystal was designed with C interoperability in mind from day one. You can call C functions, use C structs, link to C libraries, and even inline C code. This gives you access to the massive ecosystem of C libraries while writing Crystal code.

Why would you want this? Performance, existing libraries, hardware access, or because sometimes you just need to get close to the metal.

## The lib Keyword

Crystal's C bindings use the `lib` keyword to declare external C libraries:

```crystal
# Binding to standard C library math functions
lib LibC
  fun sqrt(value : Float64) : Float64
end

result = LibC.sqrt(16.0)
puts result  # => 4.0
```

That's it. You declared a C function and called it from Crystal.

## Basic C Function Binding

Let's bind to some standard C library functions:

```crystal
lib LibC
  # String length
  fun strlen(s : UInt8*) : Int32

  # String compare
  fun strcmp(s1 : UInt8*, s2 : UInt8*) : Int32

  # Get current time
  fun time(t : Int64*) : Int64
end

# Use them
str = "Hello"
len = LibC.strlen(str.to_unsafe)
puts len  # => 5

# Get current timestamp
timestamp = LibC.time(nil)
puts timestamp
```

Notice `to_unsafe` - it converts a Crystal String to a C `char*` pointer.

## C Type Mappings

Crystal types map to C types:

| Crystal Type | C Type |
|--------------|--------|
| `Int8` | `int8_t` / `char` |
| `Int16` | `int16_t` / `short` |
| `Int32` | `int32_t` / `int` |
| `Int64` | `int64_t` / `long long` |
| `UInt8` | `uint8_t` / `unsigned char` |
| `UInt16` | `uint16_t` / `unsigned short` |
| `UInt32` | `uint32_t` / `unsigned int` |
| `UInt64` | `uint64_t` / `unsigned long long` |
| `Float32` | `float` |
| `Float64` | `double` |
| `Void` | `void` |
| `Bool` | `bool` (C99+) |
| `UInt8*` | `char*` |
| `Void*` | `void*` |

## Structs

C structs map to Crystal structs:

```crystal
lib LibC
  struct TimeSpec
    tv_sec : Int64   # seconds
    tv_nsec : Int64  # nanoseconds
  end

  fun clock_gettime(clock_id : Int32, tp : TimeSpec*) : Int32
end

# Use the struct
ts = LibC::TimeSpec.new
LibC.clock_gettime(0, pointerof(ts))
puts "Seconds: #{ts.tv_sec}"
puts "Nanoseconds: #{ts.tv_nsec}"
```

Crystal structs and C structs have compatible memory layouts.

## Real-World Example: SQLite

Let's bind to SQLite, a popular C library:

```crystal
@[Link("sqlite3")]
lib LibSQLite3
  type Database = Void*
  type Statement = Void*

  fun open(filename : UInt8*, db : Database*) : Int32
  fun close(db : Database) : Int32
  fun exec(db : Database, sql : UInt8*, callback : Void*, arg : Void*, errmsg : UInt8**) : Int32
  fun prepare_v2(db : Database, sql : UInt8*, nbyte : Int32, stmt : Statement*, tail : UInt8**) : Int32
  fun step(stmt : Statement) : Int32
  fun column_text(stmt : Statement, col : Int32) : UInt8*
  fun finalize(stmt : Statement) : Int32

  SQLITE_OK = 0
  SQLITE_ROW = 100
  SQLITE_DONE = 101
end

class Database
  def initialize(filename : String)
    result = LibSQLite3.open(filename, out @db)
    raise "Failed to open database" unless result == LibSQLite3::SQLITE_OK
  end

  def execute(sql : String)
    result = LibSQLite3.exec(@db, sql, nil, nil, nil)
    raise "Failed to execute SQL" unless result == LibSQLite3::SQLITE_OK
  end

  def query(sql : String)
    results = [] of Array(String)
    result = LibSQLite3.prepare_v2(@db, sql, -1, out stmt, nil)
    raise "Failed to prepare statement" unless result == LibSQLite3::SQLITE_OK

    while LibSQLite3.step(stmt) == LibSQLite3::SQLITE_ROW
      row = [] of String
      col = 0
      loop do
        text = LibSQLite3.column_text(stmt, col)
        break if text.null?
        row << String.new(text)
        col += 1
      end
      results << row
    end

    LibSQLite3.finalize(stmt)
    results
  end

  def close
    LibSQLite3.close(@db)
  end
end

# Usage
db = Database.new(":memory:")
db.execute("CREATE TABLE users (id INTEGER, name TEXT)")
db.execute("INSERT INTO users VALUES (1, 'Alice')")
db.execute("INSERT INTO users VALUES (2, 'Bob')")

results = db.query("SELECT * FROM users")
results.each do |row|
  puts "ID: #{row[0]}, Name: #{row[1]}"
end

db.close
```

We just called SQLite's C API directly from Crystal!

## The @[Link] Annotation

The `@[Link]` annotation tells the compiler which C library to link:

```crystal
@[Link("m")]  # Link libm (math library)
lib LibM
  fun cos(x : Float64) : Float64
  fun sin(x : Float64) : Float64
end

@[Link("ssl")]  # Link OpenSSL
lib LibSSL
  # SSL functions...
end

@[Link("pthread")]  # Link pthreads
lib LibPThread
  # Thread functions...
end
```

The linker will find these libraries in standard system locations.

### Linking to Custom Libraries

```crystal
@[Link(ldflags: "-L/usr/local/lib -lmycustomlib")]
lib LibCustom
  # Custom library functions...
end
```

## Pointers and Memory Management

Crystal has pointers for C interop:

```crystal
# Create a pointer to an Int32
x = 42
ptr = pointerof(x)
puts ptr.value  # => 42

# Allocate memory
ptr = Pointer(Int32).malloc(10)  # 10 Int32s
ptr[0] = 100
ptr[1] = 200
puts ptr[0]  # => 100

# Always free manually allocated memory!
ptr.free
```

### to_unsafe: Crossing the Bridge

Crystal objects → C pointers:

```crystal
# String to C string
str = "Hello"
c_str = str.to_unsafe  # UInt8*

# Array to C array
arr = [1, 2, 3]
c_arr = arr.to_unsafe  # Int32*

# Slice to C pointer
slice = Slice(UInt8).new(10)
c_ptr = slice.to_unsafe  # UInt8*
```

### String.new: C Pointer to Crystal String

C string → Crystal String:

```crystal
lib LibC
  fun getenv(name : UInt8*) : UInt8*
end

# Get environment variable
c_value = LibC.getenv("HOME")
unless c_value.null?
  home = String.new(c_value)
  puts "Home: #{home}"
end
```

## Callbacks: C Calling Crystal

Sometimes C libraries need callbacks. You can pass Crystal procs to C:

```crystal
lib LibC
  fun qsort(base : Void*, nel : UInt64, width : UInt64,
            compar : (Void*, Void*) -> Int32) : Void
end

# Crystal callback
compare = ->(a : Void*, b : Void*) {
  x = a.as(Int32*).value
  y = b.as(Int32*).value
  x <=> y
}

# Use qsort
arr = [5, 2, 8, 1, 9]
LibC.qsort(arr.to_unsafe, arr.size, sizeof(Int32), compare)
puts arr  # => [1, 2, 5, 8, 9]
```

The `->` syntax creates a C-compatible function pointer.

## Unions

C unions (all fields share memory) are supported:

```crystal
lib LibC
  union Value
    i : Int32
    f : Float32
    c : UInt8
  end
end

val = LibC::Value.new
val.i = 42
puts val.i  # => 42

val.f = 3.14_f32
puts val.f  # => 3.14
# val.i is now garbage (overlapping memory)
```

## Enums

C enums become Crystal enums:

```crystal
lib LibC
  enum FileMode
    Read = 1
    Write = 2
    Append = 4
  end

  fun fopen(path : UInt8*, mode : UInt8*) : Void*
end

# Or use @[Flags] for bitwise enums
@[Flags]
enum Permissions
  Read = 1
  Write = 2
  Execute = 4
end

perms = Permissions::Read | Permissions::Write
```

## Real-World Example: cURL

Binding to libcurl for HTTP requests:

```crystal
@[Link("curl")]
lib LibCURL
  type CURL = Void*

  enum Option
    URL = 10002
    WRITEFUNCTION = 20011
    WRITEDATA = 10001
  end

  fun init : CURL
  fun setopt(curl : CURL, option : Option, ...) : Int32
  fun perform(curl : CURL) : Int32
  fun cleanup(curl : CURL)
end

# Callback to collect response
response = String::Builder.new
write_callback = ->(ptr : UInt8*, size : UInt32, nmemb : UInt32, userdata : Void*) {
  actual_size = size * nmemb
  str = String.new(ptr, actual_size)
  userdata.as(String::Builder*).value << str
  actual_size
}

# Make HTTP request
curl = LibCURL.init
LibCURL.setopt(curl, LibCURL::Option::URL, "https://api.github.com")
LibCURL.setopt(curl, LibCURL::Option::WRITEFUNCTION, write_callback)
LibCURL.setopt(curl, LibCURL::Option::WRITEDATA, pointerof(response))
LibCURL.perform(curl)
LibCURL.cleanup(curl)

puts response.to_s
```

(Note: Crystal has built-in HTTP client - this is just to demonstrate C bindings!)

## Using pkg-config

Many C libraries provide pkg-config metadata:

```crystal
@[Link(pkg_config: "libxml-2.0")]
lib LibXML2
  # XML parsing functions...
end
```

The compiler will use pkg-config to get the right flags automatically.

## Creating a Shard with C Bindings

Structure for a shard that wraps a C library:

```
mylib/
├── shard.yml
├── src/
│   ├── mylib.cr           # Crystal API
│   └── mylib/
│       ├── lib.cr         # C bindings
│       └── wrapper.cr     # Safe wrappers
└── spec/
    └── mylib_spec.cr
```

**src/mylib/lib.cr** (C bindings):
```crystal
@[Link("mylib")]
lib LibMyLib
  fun do_something(x : Int32) : Int32
end
```

**src/mylib/wrapper.cr** (Safe Crystal API):
```crystal
module MyLib
  def self.do_something(x : Int32) : Int32
    result = LibMyLib.do_something(x)
    raise "Error!" if result < 0
    result
  end
end
```

**src/mylib.cr** (Public API):
```crystal
require "./mylib/lib"
require "./mylib/wrapper"
```

Users get a safe Crystal API while you handle the unsafe C bindings internally.

## Safety Considerations

C bindings are **unsafe**. You can:
- Segfault
- Corrupt memory
- Leak memory
- Invoke undefined behavior

Best practices:
1. **Wrap C APIs in safe Crystal code**
2. **Validate inputs before passing to C**
3. **Handle errors from C functions**
4. **Document memory ownership**
5. **Test thoroughly**

## Exercises

1. Bind to a simple C library (like zlib for compression)
2. Create a Crystal wrapper that provides a safe API
3. Write tests that exercise C function calls
4. Use pkg-config to link a library
5. Create a callback from C to Crystal

## What You've Learned

1. The `lib` keyword declares C libraries
2. C functions are bound with `fun`
3. C structs map to Crystal structs
4. `@[Link]` specifies which library to link
5. Pointers bridge Crystal and C memory
6. `to_unsafe` converts Crystal objects to C pointers
7. C bindings are powerful but require care

## What's Next?

You've now seen how Crystal interacts with C code. Next, we'll explore macros - Crystal's compile-time metaprogramming system. You'll learn how to generate code, create DSLs, and achieve Ruby-like magic at compile time instead of runtime. This is where Crystal's philosophy of "flexible syntax, fast execution" really shines.

[Continue to Chapter 09 - Macros: The Metaprogramming You Thought You Lost →](./chapter-09.md)
