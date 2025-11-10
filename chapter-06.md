# Chapter 06: Concurrency: Fibers, Channels, and Why You'll Sleep Better

If you've done concurrency in Ruby, you've probably dealt with threads, mutexes, race conditions, and the occasional 3 AM debugging session wondering why your code works fine on your laptop but deadlocks in production.

Crystal takes a different approach. Instead of traditional threads, Crystal uses **fibers** - lightweight concurrent units that are easier to reason about. Instead of shared memory and locks, Crystal encourages **channels** for communication. The result? Concurrency that doesn't make you cry.

## The Ruby Concurrency Situation

In Ruby, you have threads:

```ruby
threads = []
10.times do |i|
  threads << Thread.new do
    puts "Thread #{i} starting"
    sleep rand
    puts "Thread #{i} done"
  end
end

threads.each(&:join)
```

This works, but:
- Threads are heavy (each needs its own stack, ~1MB)
- The GIL (Global Interpreter Lock) limits true parallelism in MRI
- Shared memory requires careful synchronization
- Race conditions are easy to introduce

## Crystal's Fibers: Concurrency Made Light

Crystal uses fibers - lightweight concurrent execution units:

```crystal
10.times do |i|
  spawn do
    puts "Fiber #{i} starting"
    sleep rand
    puts "Fiber #{i} done"
  end
end

# Wait for all fibers to complete
Fiber.yield
sleep 2  # Give fibers time to run
```

Fibers are:
- **Lightweight**: Thousands can run simultaneously
- **Cooperative**: They yield control, not preempted
- **Fast**: Minimal context-switching overhead
- **Single-threaded (by default)**: No data races!

## The spawn Keyword

`spawn` creates a new fiber:

```crystal
# Spawn a fiber
spawn do
  puts "Hello from fiber!"
end

# Spawn with arguments
def process(n : Int32)
  spawn do
    puts "Processing #{n}"
    sleep 1
    puts "Done with #{n}"
  end
end

5.times { |i| process(i) }

# Keep main fiber alive
sleep 2
```

The spawned code runs concurrently with the rest of your program.

## Channels: Communication Without Shared Memory

Instead of shared memory with locks, Crystal provides channels - typed queues for communication:

```crystal
# Create a channel
channel = Channel(Int32).new

# Send to channel from a fiber
spawn do
  5.times do |i|
    channel.send(i)
    puts "Sent #{i}"
  end
  channel.close
end

# Receive from channel
while value = channel.receive?
  puts "Received #{value}"
end
```

Channels are:
- **Typed**: `Channel(T)` holds values of type T
- **Blocking**: Send/receive block until ready
- **Thread-safe**: Safe to use from multiple fibers
- **Closable**: Signal when no more values coming

### The Mantra

> "Do not communicate by sharing memory; share memory by communicating."

This is Crystal's concurrency philosophy (borrowed from Go).

## Channel Patterns

### Pattern 1: Producer-Consumer

```crystal
def producer(channel : Channel(Int32))
  spawn do
    10.times do |i|
      sleep 0.1
      channel.send(i)
      puts "Produced: #{i}"
    end
    channel.close
  end
end

def consumer(channel : Channel(Int32))
  spawn do
    while value = channel.receive?
      puts "Consumed: #{value}"
      sleep 0.2
    end
  end
end

channel = Channel(Int32).new
producer(channel)
consumer(channel)

sleep 3  # Let them run
```

### Pattern 2: Worker Pool

```crystal
def worker(id : Int32, jobs : Channel(Int32), results : Channel(Int32))
  spawn do
    while job = jobs.receive?
      puts "Worker #{id} processing job #{job}"
      sleep rand(0.5)
      results.send(job * 2)
    end
  end
end

jobs = Channel(Int32).new
results = Channel(Int32).new

# Start 3 workers
3.times { |i| worker(i, jobs, results) }

# Send jobs
spawn do
  10.times do |i|
    jobs.send(i)
  end
  jobs.close
end

# Collect results
count = 0
while result = results.receive?
  puts "Result: #{result}"
  count += 1
  break if count == 10
end
```

### Pattern 3: Fan-Out/Fan-In

```crystal
def process_data(input : Int32) : String
  sleep rand(0.5)
  "Processed #{input}"
end

def fan_out(inputs : Array(Int32)) : Channel(String)
  output = Channel(String).new(inputs.size)

  inputs.each do |input|
    spawn do
      result = process_data(input)
      output.send(result)
    end
  end

  output
end

inputs = [1, 2, 3, 4, 5]
results = fan_out(inputs)

# Collect all results
inputs.size.times do
  puts results.receive
end
```

## Buffered Channels

Channels can have a buffer:

```crystal
# Unbuffered: send blocks until receive
unbuffered = Channel(Int32).new

# Buffered: send doesn't block until buffer is full
buffered = Channel(Int32).new(10)

spawn do
  10.times do |i|
    buffered.send(i)
    puts "Sent #{i} (didn't block!)"
  end
end

sleep 1
10.times { puts buffered.receive }
```

Use buffered channels when:
- You want to decouple send/receive timing
- You're okay with some latency
- You want to smooth out bursts of data

## Select: Waiting on Multiple Channels

The `select` statement waits on multiple channels:

```crystal
ch1 = Channel(String).new
ch2 = Channel(String).new

spawn do
  sleep 1
  ch1.send("from ch1")
end

spawn do
  sleep 0.5
  ch2.send("from ch2")
end

2.times do
  select
  when value = ch1.receive
    puts "Got: #{value}"
  when value = ch2.receive
    puts "Got: #{value}"
  end
end
```

Select blocks until one channel is ready, then executes that branch.

### Select with Timeout

```crystal
channel = Channel(String).new

spawn do
  sleep 2
  channel.send("data")
end

select
when value = channel.receive
  puts "Received: #{value}"
when timeout(1.second)
  puts "Timed out waiting for data"
end
```

## Fiber Coordination

### WaitGroup Pattern

```crystal
class WaitGroup
  def initialize(@count : Int32 = 0)
    @channel = Channel(Nil).new
  end

  def add(count : Int32 = 1)
    @count += count
  end

  def done
    @count -= 1
    @channel.send(nil) if @count == 0
  end

  def wait
    @channel.receive if @count > 0
  end
end

wg = WaitGroup.new
wg.add(5)

5.times do |i|
  spawn do
    puts "Task #{i} starting"
    sleep rand
    puts "Task #{i} done"
    wg.done
  end
end

wg.wait
puts "All tasks complete"
```

### Semaphore Pattern

```crystal
class Semaphore
  def initialize(size : Int32)
    @channel = Channel(Nil).new(size)
    size.times { @channel.send(nil) }
  end

  def acquire
    @channel.receive
  end

  def release
    @channel.send(nil)
  end

  def synchronize(&block)
    acquire
    begin
      yield
    ensure
      release
    end
  end
end

sem = Semaphore.new(3)  # Max 3 concurrent

10.times do |i|
  spawn do
    sem.synchronize do
      puts "Task #{i} running (max 3 concurrent)"
      sleep 1
    end
  end
end

sleep 5
```

## Real-World Example: Web Scraper

```crystal
require "http/client"

def fetch_url(url : String, results : Channel(String))
  spawn do
    begin
      response = HTTP::Client.get(url)
      results.send("#{url}: #{response.status_code}")
    rescue ex
      results.send("#{url}: Error - #{ex.message}")
    end
  end
end

urls = [
  "https://example.com",
  "https://example.org",
  "https://example.net",
]

results = Channel(String).new

# Fetch all URLs concurrently
urls.each { |url| fetch_url(url, results) }

# Collect results
urls.size.times do
  puts results.receive
end
```

All URLs are fetched concurrently, without threads or complicated synchronization!

## Parallel Execution with Multi-Threading

By default, Crystal runs in a single thread. For true parallelism, enable multi-threading:

```crystal
# Compile with -D preview_mt flag
# crystal build --release -D preview_mt myapp.cr

# Then your fibers can run on multiple CPU cores
spawn do
  # This could run on thread 1
  heavy_computation()
end

spawn do
  # This could run on thread 2
  another_heavy_computation()
end
```

Note: Multi-threading in Crystal is still being refined. Check the docs for current status.

## Common Patterns for Ruby Developers

### Instead of Thread + Mutex

Ruby:
```ruby
mutex = Mutex.new
counter = 0

threads = 10.times.map do
  Thread.new do
    1000.times do
      mutex.synchronize { counter += 1 }
    end
  end
end

threads.each(&:join)
puts counter
```

Crystal:
```crystal
channel = Channel(Int32).new(1)
channel.send(0)  # Initial value

10.times do
  spawn do
    1000.times do
      value = channel.receive
      channel.send(value + 1)
    end
  end
end

sleep 1
puts channel.receive
```

Or better yet, use atomic operations:
```crystal
counter = Atomic(Int32).new(0)

10.times do
  spawn do
    1000.times do
      counter.add(1)
    end
  end
end

sleep 1
puts counter.get
```

### Instead of Thread Pool

Ruby:
```ruby
require 'thread'

queue = Queue.new
threads = 3.times.map do
  Thread.new do
    while job = queue.pop
      process(job)
    end
  end
end

10.times { |i| queue << i }
```

Crystal:
```crystal
jobs = Channel(Int32).new

# Workers
3.times do
  spawn do
    while job = jobs.receive?
      process(job)
    end
  end
end

# Submit jobs
10.times { |i| jobs.send(i) }
jobs.close
```

## Avoiding Common Pitfalls

### 1. Forgetting to Keep Main Fiber Alive

```crystal
# BAD: main fiber exits immediately
spawn do
  puts "Hello"
  sleep 1
end
# Program exits before fiber runs!

# GOOD: wait for fibers
spawn do
  puts "Hello"
  sleep 1
end
sleep 2  # Or use channels, or Fiber.yield
```

### 2. Closing Channels Too Early

```crystal
# BAD
channel = Channel(Int32).new
channel.close  # Closed before use!
# channel.send(42)  # Raises!

# GOOD
channel = Channel(Int32).new
spawn do
  5.times { |i| channel.send(i) }
  channel.close  # Close when done
end
```

### 3. Deadlocks

```crystal
# BAD: deadlock
channel = Channel(Int32).new
channel.send(42)  # Blocks forever (no receiver!)

# GOOD: receiver exists
channel = Channel(Int32).new
spawn { puts channel.receive }
channel.send(42)
```

## Performance Tips

1. **Use buffered channels** for producer-consumer patterns
2. **Batch work** instead of spawning per tiny task
3. **Pool fibers** for CPU-intensive work
4. **Profile** before optimizing
5. **Consider multi-threading** for CPU-bound work (with -D preview_mt)

## Exercises

1. Write a parallel prime number finder using multiple fibers
2. Implement a rate-limiter using channels
3. Create a pipeline that processes data through multiple stages
4. Build a simple job queue with worker fibers

## What You've Learned

1. Crystal uses fibers for lightweight concurrency
2. `spawn` creates new fibers
3. Channels enable safe communication between fibers
4. Select waits on multiple channels
5. Buffered channels decouple producers and consumers
6. Crystal's concurrency avoids most threading pitfalls
7. Multi-threading is available for true parallelism

## What's Next?

You now understand Crystal's concurrency model. Next, we'll explore the Crystal ecosystem and how to manage dependencies with Shards - Crystal's package manager. You'll learn how to find, use, and create libraries to avoid reinventing the wheel.

[Continue to Chapter 07 - Shards: Bundler's Younger, Faster Sibling →](./chapter-07.md)
