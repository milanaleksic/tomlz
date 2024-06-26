# tomlz

A TOML parser for Zig targeting TOML v1.0.0, an easy API and safety.
Also supports encoding/serializing values(implemented by @0x5a4)!

```zig
const std = @import("std")
const tomlz = @import("tomlz")

var gpa = std.heap.page_allocator;
var table = try tomlz.parse(gpa,
    \\foo = 1
    \\bar = 2
);
defer table.deinit(gpa);

table.getInteger("foo").?; // 1


// --- or ---
const S = struct { foo: i64, bar: i64 };
const s = try tomlz.decode(S, gpa,
  \\foo = 1
  \\bar = 2
);

// serialize a value like this (also see the examples)
try tomlz.serialize(
    gpa,
    std.io.getStdout.writer()
    s,
);
// foo = 1
// bar = 2
```

## Current status

All types other than datetimes are supported. We pass 321/334 of the
[toml tests](https://github.com/BurntSushi/toml-test) 11 of those are due to not
having datetime support and the other two are minor lexing issues (allowing
whitespace between the square brackets of an array header).

We allow both parsing into a special TOML type, a `Table`, but also support
decoding into a struct directly - including types that must be allocated like
arrays and strings.

The Serializer allows encoding every kind of zig type, overwriting it's default behaviour
by implementing a function called `tomlzSerialize`, has the option to work
without an allocator and can therefore even work at `comptime`!
Note that for some types like `std.HashMap` its not possible to just encode
all their fields, so custom logic is needed. We can't provide this, but it's not
too difficult to implement it yourself(See examples).

## Installation

tomlz supports being included as a module.

Create a file called `build.zig.zon` if you do not already have one, and add `tomlz` as a dependency

```
.{
    .name = "myproject",
    .version = "0.1.0",
    .dependencies = .{
        .tomlz = .{
            .url = "https://github.com/mattyhall/tomlz/archive/<commit-hash>.tar.gz",
            .hash = "12206cf9e90462ee6e14f593ea6e0802b9fe434429ba10992a1451e32900f741005c",
        },
    }
}
```

You'll have to replace the `<commit-hash>` part with an actual, recent commit-hash.
The hash also needs changing, but `zig build` will complain and give you the correct one.

In your `build.zig` file create and use the dependency

```
pub fn build(b: *std.Build) void {
    // ... setup ...

    const tomlz = b.dependency("tomlz", .{
        .target = target,
        .optimize = optimize,
    });

    // add the tomlz module
    exe.root_module.addImport("tomlz", tomlz.module("tomlz"));

    // .. continue ...
}
```

## Usage

### Table

We currently provide a single entry point for parsing which returns a toml
`Table` type. This type has helper methods for getting values out:

```zig
const std = @import("std");
const tomlz = @import("tomlz");

var gpa = std.heap.page_allocator;
var table = try tomlz.parse(gpa,
    \\int = 1
    \\float = 2.0
    \\boolean = true
    \\string = "hello, world"
    \\array = [1, 2, 3]
    \\table = { subvalue = 1, we.can.nest.keys = 2 }
);
defer table.deinit(gpa);

table.getInteger("int");
table.getFloat("float");
table.getBool("boolean");
table.getString("string");
table.getArray("array");
table.getTable("table");
```

A simple example is
[provided](https://github.com/mattyhall/tomlz/tree/main/examples/simple/).

### Decode

```zig
const std = @import("std");
const tomlz = @import("tomlz");

var gpa = std.heap.page_allocator;
const TripleCrowns = struct { worlds: i64, masters: i64, uks: i64 };

const Player = struct {
    name: []const u8,
    age: i64,
    hobbies: []const []const u8,
    triplecrowns: TripleCrowns,

    const Self = @This();

    pub fn deinit(self: *Self, gpa: std.mem.Allocator) void {
        gpa.free(self.name);

        for (self.hobbies) |hobby| {
            gpa.free(hobby);
        }
        gpa.free(self.hobbies);
    }
};

const Game = struct {
    name: []const u8,
    goat: Player,

    const Self = @This();

    pub fn deinit(self: *Self, gpa: std.mem.Allocator) void {
        gpa.free(self.name);
        self.goat.deinit(gpa);
    }
};

var s = try tomlz.decode(Game, gpa,
    \\name = "snooker"
    \\
    \\[goat]
    \\name = "Ronnie o' Sullivan"
    \\age = 46 # as of Nov 2022
    \\hobbies = ["running", "hustling at pool"]
    \\
    \\[goat.triplecrowns]
    \\worlds = 7
    \\masters = 7
    \\uks = 7
);
defer s.deinit(gpa);
```

### Encode

Have a look at [the example](examples/serialize/src/main.zig).

## Goals and non-goals

Goals and non-goals are subject to change based on how the project is used and
my own time constraints. If you feel a goal or non-goal isn't quite right please
open an issue and we can discuss it.

### Goals

- TOML v1.0.0. The datetime portion of this is probably going to be
  unachievable until Zig gets a good standard library type for it or a library
  gets dominance. Other than that however we should pass all the
  [tests](https://github.com/BurntSushi/toml-test)
- A nice API. Getting values from the `Value` type should be painless as
  possible and we should also provide deserialising a `Table` into a struct,
  similarly to how `std.json` does it
- Easy installation. We should try to make using the library as a vendored
  dependency and as a package - on e.g. [gyro](https://github.com/mattnite/gyro)
  \- as easy as possible
- Safety. The parser should never crash no matter the input. To achieve this we
  should run fuzzing against it
- Support Zig master and the latest tagged release until Zig v1.0. This will be
  done by having the main branch track Zig master and a branch for each Zig
  release. Any improvements should be backported to the most recent release
  branch
- Good error messages

### Non-goals

- Super duper performance. We want to be as performant as possible without
  making the code harder to read. It is unlikely that parsing a TOML file is
  going to be the bottleneck in your application so "good" performance should be
  sufficient
- Previous versions of TOML
