# Chapter 07: Shards: Bundler's Younger, Faster Sibling

If you're coming from Ruby, you know Bundler. You know `Gemfile`, you know `bundle install`, and you know the joy of dependency hell at 2 AM when versions conflict.

Crystal has Shards - similar concept, cleaner execution, fewer headaches. Think of Shards as Bundler's younger sibling who learned from all the mistakes and came out more efficient.

## What Are Shards?

Shards are Crystal's packages - libraries you can add to your project. The tool that manages them is also called `shards` (yes, confusing, but we'll survive).

In Ruby:
- Packages: Gems
- Package manager: Bundler
- Manifest: Gemfile

In Crystal:
- Packages: Shards
- Package manager: shards
- Manifest: shard.yml

## The shard.yml File

Every Crystal project with dependencies needs a `shard.yml` file:

```yaml
name: myapp
version: 0.1.0

authors:
  - Your Name <you@example.com>

crystal: ">= 1.0.0"

dependencies:
  kemal:
    github: kemalcr/kemal
    version: ~> 1.1.0

development_dependencies:
  webmock:
    github: manastech/webmock.cr
    version: ~> 0.14.0

license: MIT
```

This is your `Gemfile` equivalent, but in YAML.

## Installing Shards

Create a `shard.yml` and run:

```bash
shards install
```

This:
1. Reads `shard.yml`
2. Resolves dependencies
3. Downloads shards to `lib/`
4. Creates `shard.lock` (version lock file)

The `shard.lock` file (like `Gemfile.lock`) ensures everyone gets the same versions.

## Finding Shards

### Option 1: CrystalShards.org

Visit [crystalshards.org](https://crystalshards.org) - a searchable directory of Crystal shards.

### Option 2: GitHub Search

Most shards are on GitHub with names ending in `.cr`:
- `kemal.cr` - web framework
- `pg.cr` - PostgreSQL driver
- `redis.cr` - Redis client

### Option 3: Awesome Crystal

Check out [awesome-crystal](https://github.com/veelenga/awesome-crystal) - a curated list of quality shards.

## Adding Dependencies

### From GitHub

Most common - pull from GitHub:

```yaml
dependencies:
  mylib:
    github: username/mylib
    version: ~> 1.0.0
```

Version specifiers:
- `1.0.0` - exact version
- `~> 1.0.0` - >= 1.0.0 and < 1.1.0
- `>= 1.0.0` - any version >= 1.0.0
- `< 2.0.0` - any version < 2.0.0

### From Git URL

```yaml
dependencies:
  mylib:
    git: https://gitlab.com/username/mylib.git
    version: ~> 1.0.0
```

### From Local Path

Useful for development:

```yaml
dependencies:
  mylib:
    path: ../mylib
```

### From Specific Branch or Commit

```yaml
dependencies:
  experimental_lib:
    github: username/experimental_lib
    branch: develop

  stable_lib:
    github: username/stable_lib
    commit: abc123
```

## Using Installed Shards

After running `shards install`, require the shard in your code:

```crystal
# shard.yml has:
# dependencies:
#   kemal:
#     github: kemalcr/kemal

# In your code:
require "kemal"

get "/" do
  "Hello World!"
end

Kemal.run
```

The `require` statement finds the shard in the `lib/` directory.

## Development Dependencies

Dependencies only needed for development or testing:

```yaml
development_dependencies:
  spec-kemal:
    github: kemalcr/spec-kemal
  ameba:
    github: crystal-ameba/ameba
```

These won't be installed when someone uses your shard as a dependency.

## Creating Your Own Shard

### Initialize a New Shard

```bash
crystal init lib myshard
cd myshard
```

This creates:
```
myshard/
├── shard.yml
├── src/
│   ├── myshard.cr
│   └── myshard/
│       └── version.cr
├── spec/
│   ├── spec_helper.cr
│   └── myshard_spec.cr
├── .gitignore
├── LICENSE
└── README.md
```

### The Structure

**src/myshard.cr** - Main entry point:
```crystal
require "./myshard/version"

module Myshard
  # Your public API here
end
```

**src/myshard/version.cr** - Version constant:
```crystal
module Myshard
  VERSION = "0.1.0"
end
```

**spec/myshard_spec.cr** - Tests:
```crystal
require "./spec_helper"

describe Myshard do
  it "works" do
    true.should eq(true)
  end
end
```

### Building Your Shard

Add functionality:

```crystal
# src/myshard.cr
require "./myshard/version"

module Myshard
  def self.greet(name : String) : String
    "Hello, #{name}!"
  end

  def self.calculate(a : Int32, b : Int32) : Int32
    a + b
  end
end
```

Write tests:

```crystal
# spec/myshard_spec.cr
require "./spec_helper"

describe Myshard do
  describe ".greet" do
    it "greets by name" do
      Myshard.greet("Crystal").should eq("Hello, Crystal!")
    end
  end

  describe ".calculate" do
    it "adds numbers" do
      Myshard.calculate(2, 3).should eq(5)
    end
  end
end
```

Run tests:
```bash
crystal spec
```

## Publishing Your Shard

1. **Make sure shard.yml is complete**:
```yaml
name: myshard
version: 0.1.0

authors:
  - Your Name <you@example.com>

description: |
  A short description of what your shard does

crystal: ">= 1.0.0"

license: MIT

repository: https://github.com/yourusername/myshard
```

2. **Tag a release**:
```bash
git tag v0.1.0
git push origin v0.1.0
```

3. **Submit to CrystalShards** (optional):
Once your repo is public and tagged, it should appear automatically on crystalshards.org.

## Shard Best Practices

### Versioning

Follow [Semantic Versioning](https://semver.org/):
- **MAJOR**: Breaking changes (1.0.0 → 2.0.0)
- **MINOR**: New features, backwards compatible (1.0.0 → 1.1.0)
- **PATCH**: Bug fixes (1.0.0 → 1.0.1)

### Documentation

Include good docs:
```crystal
module Myshard
  # Greets a person by name
  #
  # ```
  # Myshard.greet("Alice")  # => "Hello, Alice!"
  # ```
  def self.greet(name : String) : String
    "Hello, #{name}!"
  end
end
```

Generate docs:
```bash
crystal docs
```

View at `docs/index.html`.

### Testing

Write comprehensive specs:
```bash
crystal spec --verbose
```

### README

Include:
- Installation instructions
- Basic usage examples
- Link to documentation
- Contribution guidelines

## Real-World Example: Using Kemal

Kemal is a popular web framework shard. Let's use it:

**shard.yml**:
```yaml
name: webapp
version: 0.1.0

dependencies:
  kemal:
    github: kemalcr/kemal
    version: ~> 1.1.0
```

**Install**:
```bash
shards install
```

**src/webapp.cr**:
```crystal
require "kemal"

get "/" do
  "Welcome to my Crystal web app!"
end

get "/hello/:name" do |env|
  name = env.params.url["name"]
  "Hello, #{name}!"
end

post "/echo" do |env|
  body = env.request.body.try(&.gets_to_end) || ""
  "You said: #{body}"
end

Kemal.run
```

**Run**:
```bash
crystal run src/webapp.cr
```

Visit `http://localhost:3000` and you have a web server!

## Common Shards You Should Know

### Web Development

- **kemal** - Web framework (Sinatra-like)
- **amber** - Full-stack framework (Rails-like)
- **lucky** - Type-safe web framework

### Databases

- **pg** - PostgreSQL
- **mysql** - MySQL
- **sqlite3** - SQLite
- **redis** - Redis

### Testing

- **spec** - Built-in testing (like RSpec)
- **webmock** - HTTP request mocking
- **timecop** - Time manipulation for tests

### Utilities

- **json** - JSON parsing (built-in)
- **yaml** - YAML parsing (built-in)
- **http** - HTTP client/server (built-in)
- **log** - Logging (built-in)

### CLI Tools

- **option_parser** - Command-line parsing (built-in)
- **colorize** - Terminal colors (built-in)
- **commander** - CLI framework

## Troubleshooting

### Dependency Conflicts

```bash
shards install
# Error: Conflicting dependencies...
```

Check which shards require conflicting versions and update your `shard.yml` constraints.

### Outdated Shards

Update to latest versions:
```bash
shards update
```

### Missing Dependencies

If compilation fails with "can't find shard":
```bash
# Make sure dependencies are installed
shards install

# Check that you're requiring correctly
require "shard_name"  # not "lib/shard_name"
```

### Removing Unused Shards

```bash
# Remove from shard.yml, then:
rm -rf lib/
shards install
```

## Comparing to Bundler

| Feature | Bundler | Shards |
|---------|---------|--------|
| Manifest | Gemfile | shard.yml |
| Lock file | Gemfile.lock | shard.lock |
| Install | bundle install | shards install |
| Update | bundle update | shards update |
| Init | bundle init | crystal init lib |
| Local dev | gem build / path | path in shard.yml |

The concepts are nearly identical, so your Bundler knowledge transfers directly.

## Exercises

1. Create a new shard that provides string utility functions
2. Add Kemal as a dependency and build a simple API
3. Write comprehensive specs for your shard
4. Generate documentation for your shard
5. Explore crystalshards.org and try 3 interesting shards

## What You've Learned

1. Shards are Crystal's packages, like Ruby gems
2. `shard.yml` defines dependencies (like Gemfile)
3. `shards install` fetches dependencies
4. `shard.lock` locks versions for reproducibility
5. Creating shards is straightforward with `crystal init lib`
6. The shard ecosystem is smaller than gems but growing
7. Most concepts from Bundler transfer directly

## What's Next?

You now know how to use and create shards. Next, we'll explore one of Crystal's unique capabilities: C bindings. You'll learn how to call C libraries directly from Crystal, giving you access to decades of C code and letting you optimize performance-critical sections by dropping down to C when needed.

[Continue to Chapter 08 - C Bindings: When You Need to Get Down and Dirty →](./chapter-08.md)
