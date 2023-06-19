---
title: "Chapter 2 - 标准模式"
weight: 3
date: 2023-04-28 18:00:00
description: "Chapter 2 - 本节教程将详细介绍Zig编程语言的标准库."
---

自动生成的标准库文档可以在[这里](https://ziglang.org/documentation/master/std/)找到。安装[ZLS](https://github.com/zigtools/zls/)也可以帮助你探索标准库，它为你提供了完成度。

# Allocators 分配器

Zig标准库提供了一个分配内存的模式，它允许程序员准确地选择在标准库中如何进行内存分配--在标准库中没有分配在你背后发生。

最基本的分配器是[`std.heap.page_allocator`]（https://ziglang.org/documentation/master/std/#A;std:heap.page_allocator）。每当这个分配器进行分配时，它都会向你的操作系统索取整页的内存；一个单一字节的分配可能会保留多个kibibytes。由于向操作系统索取内存需要一个系统调用，这对速度来说也是非常低效的。

在这里，我们分配了100个字节作为`[]u8`。注意defer是如何与free结合使用的--这是Zig中内存管理的一个常见模式 .

```zig
const std = @import("std");
const expect = std.testing.expect;

test "allocation" {
    const allocator = std.heap.page_allocator;

    const memory = try allocator.alloc(u8, 100);
    defer allocator.free(memory);

    try expect(memory.len == 100);
    try expect(@TypeOf(memory) == []u8);
}
```

[`std.heap.FixedBufferAllocator`](https://ziglang.org/documentation/master/std/#A;std:heap.FixedBufferAllocator)是一个分配器，将内存分配到一个固定的缓冲区，而不进行任何堆分配。当不希望使用堆时，例如在编写内核时，这很有用。出于性能方面的考虑，也可以考虑这样做。如果它的字节数用完了，它将给你一个错误`OutOfMemory`。

```zig
test "fixed buffer allocator" {
    var buffer: [1000]u8 = undefined;
    var fba = std.heap.FixedBufferAllocator.init(&buffer);
    const allocator = fba.allocator();

    const memory = try allocator.alloc(u8, 100);
    defer allocator.free(memory);

    try expect(memory.len == 100);
    try expect(@TypeOf(memory) == []u8);
}
```

[`std.heap.ArenaAllocator`](https://ziglang.org/documentation/master/std/#A;std:heap.ArenaAllocator)吸收了一个子分配器，并允许你多次分配，只释放一次。在这里，`.deinit()`被调用到竞技场上，释放了所有的内存。在这个例子中使用`allocator.free`将是一个no-op（即，什么都不做） .

```zig
test "arena allocator" {
    var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
    defer arena.deinit();
    const allocator = arena.allocator();

    _ = try allocator.alloc(u8, 1);
    _ = try allocator.alloc(u8, 10);
    _ = try allocator.alloc(u8, 100);
}
```

`alloc`和`free`用于分片。对于单个项目，考虑使用`create`和`destroy` .

```zig
test "allocator create/destroy" {
    const byte = try std.heap.page_allocator.create(u8);
    defer std.heap.page_allocator.destroy(byte);
    byte.* = 128;
}
```

Zig标准库也有一个通用的分配器。这是一个安全的分配器，可以防止双重自由、使用后自由，并可以检测泄漏。安全检查和线程安全可以通过其配置结构关闭（下面留空）。Zig的GPA是为安全而非性能而设计的，但仍可能比page_allocator快很多倍 .

```zig
test "GPA" {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    const allocator = gpa.allocator();
    defer {
        const deinit_status = gpa.deinit();
        //fail test; can't try in defer as defer is executed after we return
        if (deinit_status == .leak) expect(false) catch @panic("TEST FAIL");
    }

    const bytes = try allocator.alloc(u8, 100);
    defer allocator.free(bytes);
}
```

为了获得高性能（但安全功能非常少！），可以考虑使用[std.heap.c_allocator](https://ziglang.org/documentation/master/std/#A;std:heap.c_allocator)。然而这有一个缺点，就是需要链接Libc，这可以用`-lc`来完成。

Benjamin Feng的讲座 [*What's a Memory Allocator Anyway?*](https://www.youtube.com/watch?v=vHWiDx_l4V0)对这个话题进行了更详细的介绍，并涵盖了分配器的实现。

# Arraylist

[`std.ArrayList`](https://ziglang.org/documentation/master/std/#A;std:ArrayList)在整个Zig中普遍使用，作为一个缓冲区，其大小可以改变。`std.ArrayList(T)`类似于C++的`std::vector<T>`和Rust的`Vec<T>`。`deinit()`方法释放了ArrayList的所有内存。该内存可以通过它的分片字段--`.items`来读和写。

这里我们将介绍测试分配器的用法。这是一个特殊的分配器，只在测试中工作，可以检测内存泄漏。在你的代码中，使用任何合适的分配器 .

```zig
const eql = std.mem.eql;
const ArrayList = std.ArrayList;
const test_allocator = std.testing.allocator;

test "arraylist" {
    var list = ArrayList(u8).init(test_allocator);
    defer list.deinit();
    try list.append('H');
    try list.append('e');
    try list.append('l');
    try list.append('l');
    try list.append('o');
    try list.appendSlice(" World!");

    try expect(eql(u8, list.items, "Hello World!"));
}
```

# Filesystem

让我们在当前工作目录下创建并打开一个文件，向其中写入内容，然后从其中读取。在这里，我们必须使用`.seekTo`，以便在读取我们所写的内容之前回到文件的起点 .

```zig
test "createFile, write, seekTo, read" {
    const file = try std.fs.cwd().createFile(
        "junk_file.txt",
        .{ .read = true },
    );
    defer file.close();

    const bytes_written = try file.writeAll("Hello File!");
    _ = bytes_written;

    var buffer: [100]u8 = undefined;
    try file.seekTo(0);
    const bytes_read = try file.readAll(&buffer);

    try expect(eql(u8, buffer[0..bytes_read], "Hello File!"));
}
```

函数[`std.fs.openFileAbsolute`](https://ziglang.org/documentation/master/std/#A;std:fs.openFileAbsolute)和类似的绝对函数存在，但我们不会在这里测试它们。

我们可以通过对文件使用`.stat()`来获得有关文件的各种信息。`Stat`也包含了.inode和.mode的字段，但是这里不对它们进行测试，因为它们依赖于当前操作系统的类型。

```zig
test "file stat" {
    const file = try std.fs.cwd().createFile(
        "junk_file2.txt",
        .{ .read = true },
    );
    defer file.close();
    const stat = try file.stat();
    try expect(stat.size == 0);
    try expect(stat.kind == .File);
    try expect(stat.ctime <= std.time.nanoTimestamp());
    try expect(stat.mtime <= std.time.nanoTimestamp());
    try expect(stat.atime <= std.time.nanoTimestamp());
}
```

我们可以制作目录并对其内容进行迭代。这里我们将使用一个迭代器（稍后讨论）。这个目录（和它的内容）将在这个测试结束后被删除 .

```zig
test "make dir" {
    try std.fs.cwd().makeDir("test-tmp");
    const iter_dir = try std.fs.cwd().openIterableDir(
        "test-tmp",
        .{},
    );
    defer {
        std.fs.cwd().deleteTree("test-tmp") catch unreachable;
    }

    _ = try iter_dir.dir.createFile("x", .{});
    _ = try iter_dir.dir.createFile("y", .{});
    _ = try iter_dir.dir.createFile("z", .{});

    var file_count: usize = 0;
    var iter = iter_dir.iterate();
    while (try iter.next()) |entry| {
        if (entry.kind == .File) file_count += 1;
    }

    try expect(file_count == 3);
}
```

# Readers and Writers

[`std.io.Writer`](https://ziglang.org/documentation/master/std/#A;std:io.Writer)和[`std.io.Reader`](https://ziglang.org/documentation/master/std/#A;std:io.Reader)提供使用IO的标准方法。`std.ArrayList(u8)`有一个`writer`方法，给了我们一个写入器。让我们来使用它 .

```zig
test "io writer usage" {
    var list = ArrayList(u8).init(test_allocator);
    defer list.deinit();
    const bytes_written = try list.writer().write(
        "Hello World!",
    );
    try expect(bytes_written == 12);
    try expect(eql(u8, list.items, "Hello World!"));
}
```

这里我们将使用一个阅读器将文件的内容复制到分配的缓冲区。[`readAllAlloc`](https://ziglang.org/documentation/master/std/#A;std:io.Reader.readAllAlloc)的第二个参数是它可能分配的最大尺寸；如果文件大于这个尺寸，它将返回`error.StreamTooLong` .

```zig
test "io reader usage" {
    const message = "Hello File!";

    const file = try std.fs.cwd().createFile(
        "junk_file2.txt",
        .{ .read = true },
    );
    defer file.close();

    try file.writeAll(message);
    try file.seekTo(0);

    const contents = try file.reader().readAllAlloc(
        test_allocator,
        message.len,
    );
    defer test_allocator.free(contents);

    try expect(eql(u8, contents, message));
}
```

读取器的一个常见用例是读取到下一行（例如，用户输入）。这里我们将用[`std.io.getStdIn()`](https://ziglang.org/documentation/master/std/#A;std:io.getStdIn)文件做到这一点 .

```zig
fn nextLine(reader: anytype, buffer: []u8) !?[]const u8 {
    var line = (try reader.readUntilDelimiterOrEof(
        buffer,
        '\n',
    )) orelse return null;
    // trim annoying windows-only carriage return character
    if (@import("builtin").os.tag == .windows) {
        return std.mem.trimRight(u8, line, "\r");
    } else {
        return line;
    }
}

test "read until next line" {
    const stdout = std.io.getStdOut();
    const stdin = std.io.getStdIn();

    try stdout.writeAll(
        \\ Enter your name:
    );

    var buffer: [100]u8 = undefined;
    const input = (try nextLine(stdin.reader(), &buffer)).?;
    try stdout.writer().print(
        "Your name is: \"{s}\"\n",
        .{input},
    );
}
```

一个[`std.io.Writer`](https://ziglang.org/documentation/master/std/#A;std:io.Writer)类型由一个上下文类型、错误集和一个写函数组成。写入函数必须接收上下文类型和一个字节片。写入函数还必须返回Writer类型的错误集和写入字节数的错误联合。让我们创建一个实现写程序的类型 .

```zig
// Don't create a type like this! Use an
// arraylist with a fixed buffer allocator
const MyByteList = struct {
    data: [100]u8 = undefined,
    items: []u8 = &[_]u8{},

    const Writer = std.io.Writer(
        *MyByteList,
        error{EndOfBuffer},
        appendWrite,
    );

    fn appendWrite(
        self: *MyByteList,
        data: []const u8,
    ) error{EndOfBuffer}!usize {
        if (self.items.len + data.len > self.data.len) {
            return error.EndOfBuffer;
        }
        std.mem.copy(
            u8,
            self.data[self.items.len..],
            data,
        );
        self.items = self.data[0 .. self.items.len + data.len];
        return data.len;
    }

    fn writer(self: *MyByteList) Writer {
        return .{ .context = self };
    }
};

test "custom writer" {
    var bytes = MyByteList{};
    _ = try bytes.writer().write("Hello");
    _ = try bytes.writer().write(" Writer!");
    try expect(eql(u8, bytes.items, "Hello Writer!"));
}
```

# Formatting 格式化

[`std.fmt`](https://ziglang.org/documentation/master/std/#A;std:fmt)提供了将数据格式化为字符串和从字符串格式化的方法。

一个创建格式化字符串的基本例子。格式化字符串必须是编译时已知的。这里的`d`表示我们想要一个十进制的数字。

```zig
test "fmt" {
    const string = try std.fmt.allocPrint(
        test_allocator,
        "{d} + {d} = {d}",
        .{ 9, 10, 19 },
    );
    defer test_allocator.free(string);

    try expect(eql(u8, string, "9 + 10 = 19"));
}
```

写作器有一个`print`方法，其工作原理与此类似。

```zig
test "print" {
    var list = std.ArrayList(u8).init(test_allocator);
    defer list.deinit();
    try list.writer().print(
        "{} + {} = {}",
        .{ 9, 10, 19 },
    );
    try expect(eql(u8, list.items, "9 + 10 = 19"));
}
```

花点时间欣赏一下，你现在从上到下知道了打印hello world的工作原理。[`std.debug.print`](https://ziglang.org/documentation/master/std/#A;std:debug.print)的工作原理是一样的，只是它写到了stderr，并且受到mutex的保护 .

```zig
test "hello world" {
    const out_file = std.io.getStdOut();
    try out_file.writer().print(
        "Hello, {s}!\n",
        .{"World"},
    );
}
```

在这之前，我们一直使用`{s}`格式指定器来打印字符串。这里我们将使用`{any}`，它为我们提供了默认的格式化。

```zig
test "array printing" {
    const string = try std.fmt.allocPrint(
        test_allocator,
        "{any} + {any} = {any}",
        .{
            @as([]const u8, &[_]u8{ 1, 4 }),
            @as([]const u8, &[_]u8{ 2, 5 }),
            @as([]const u8, &[_]u8{ 3, 9 }),
        },
    );
    defer test_allocator.free(string);

    try expect(eql(
        u8,
        string,
        "{ 1, 4 } + { 2, 5 } = { 3, 9 }",
    ));
}
```

让我们通过给它一个`format`函数来创建一个具有自定义格式的类型。这个函数必须被标记为 "pub"，以便std.fmt可以访问它（后面会有更多关于包的内容）。你可能会注意到`{s}`的使用，而不是`{}`--这是字符串的格式指定器（后面有更多关于格式指定器的内容）。这里使用的是`{}`，因为`{}`默认为数组打印而不是字符串打印 .

```zig
const Person = struct {
    name: []const u8,
    birth_year: i32,
    death_year: ?i32,
    pub fn format(
        self: Person,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;

        try writer.print("{s} ({}-", .{
            self.name, self.birth_year,
        });

        if (self.death_year) |year| {
            try writer.print("{}", .{year});
        }

        try writer.writeAll(")");
    }
};

test "custom fmt" {
    const john = Person{
        .name = "John Carmack",
        .birth_year = 1970,
        .death_year = null,
    };

    const john_string = try std.fmt.allocPrint(
        test_allocator,
        "{s}",
        .{john},
    );
    defer test_allocator.free(john_string);

    try expect(eql(
        u8,
        john_string,
        "John Carmack (1970-)",
    ));

    const claude = Person{
        .name = "Claude Shannon",
        .birth_year = 1916,
        .death_year = 2001,
    };

    const claude_string = try std.fmt.allocPrint(
        test_allocator,
        "{s}",
        .{claude},
    );
    defer test_allocator.free(claude_string);

    try expect(eql(
        u8,
        claude_string,
        "Claude Shannon (1916-2001)",
    ));
}
```

# JSON

让我们使用流式解析器将一个json字符串解析成一个结构类型 .

```zig
const Place = struct { lat: f32, long: f32 };

test "json parse" {
    var stream = std.json.TokenStream.init(
        \\{ "lat": 40.684540, "long": -74.401422 }
    );
    const x = try std.json.parse(Place, &stream, .{});

    try expect(x.lat == 40.684540);
    try expect(x.long == -74.401422);
}
```

并使用stringify将任意的数据变成一个字符串.

```zig
test "json stringify" {
    const x = Place{
        .lat = 51.997664,
        .long = -0.740687,
    };

    var buf: [100]u8 = undefined;
    var fba = std.heap.FixedBufferAllocator.init(&buf);
    var string = std.ArrayList(u8).init(fba.allocator());
    try std.json.stringify(x, .{}, string.writer());

    try expect(eql(u8, string.items,
        \\{"lat":5.19976654e+01,"long":-7.40687012e-01}
    ));
}
```

json解析器需要一个用于javascript的字符串、数组和地图类型的分配器。这个内存可以用[`std.json.parseFree`](https://ziglang.org/documentation/master/std/#A;std:json.parseFree)释放 .

```zig
test "json parse with strings" {
    var stream = std.json.TokenStream.init(
        \\{ "name": "Joe", "age": 25 }
    );

    const User = struct { name: []u8, age: u16 };

    const x = try std.json.parse(
        User,
        &stream,
        .{ .allocator = test_allocator },
    );

    defer std.json.parseFree(
        User,
        x,
        .{ .allocator = test_allocator },
    );

    try expect(eql(u8, x.name, "Joe"));
    try expect(x.age == 25);
}
```

# Random Numbers

这里我们使用一个64位的随机种子创建一个新的prng。a、b、c和d通过这个prng被赋予随机值。给予c和d值的表达式是等价的。`DefaultPrng`是`Xoroshiro128`；在std.rand中还有其他prngs可用 .

```zig
test "random numbers" {
    var prng = std.rand.DefaultPrng.init(blk: {
        var seed: u64 = undefined;
        try std.os.getrandom(std.mem.asBytes(&seed));
        break :blk seed;
    });
    const rand = prng.random();

    const a = rand.float(f32);
    const b = rand.boolean();
    const c = rand.int(u8);
    const d = rand.intRangeAtMost(u8, 0, 255);

    //suppress unused constant compile error
    _ = .{ a, b, c, d };
}
```

加密安全的随机性也是可用的 .

```zig
test "crypto random numbers" {
    const rand = std.crypto.random;

    const a = rand.float(f32);
    const b = rand.boolean();
    const c = rand.int(u8);
    const d = rand.intRangeAtMost(u8, 0, 255);

    //suppress unused constant compile error
    _ = .{ a, b, c, d };
}
```

# Crypto

[`std.crypto`](https://ziglang.org/documentation/master/std/#A;std:crypto)包括许多加密实用程序，包括：
- AES (Aes128, Aes256)
- Diffie-Hellman密钥交换(x25519)
- 椭圆曲线运算（curve25519, edwards25519, ristretto255)
- 密码安全散列（blake2, Blake3, Gimli, Md5, sha1, sha2, sha3)
- MAC函数（Ghash, Poly1305）
- 流密码（ChaCha20IETF, ChaCha20With64BitNonce, XChaCha20IETF, Salsa20, XSalsa20)

这个列表是不完整的。如需更深入的信息，请尝试[A tour of std.crypto in Zig 0.7.0 - Frank Denis]（https://www.youtube.com/watch?v=9t6Y7KoCvyk）。



# Threads 线程

虽然Zig提供了更多编写并发和并行代码的高级方法，但[`std.Thread`](https://ziglang.org/documentation/master/std/#A;std:Thread)可用于利用操作系统线程。让我们来使用一个操作系统线程 .

```zig
fn ticker(step: u8) void {
    while (true) {
        std.time.sleep(1 * std.time.ns_per_s);
        tick += @as(isize, step);
    }
}

var tick: isize = 0;

test "threading" {
    var thread = try std.Thread.spawn(.{}, ticker, .{@as(u8, 1)});
    _ = thread;
    try expect(tick == 0);
    std.time.sleep(3 * std.time.ns_per_s / 2);
    try expect(tick == 1);
}
```

然而，如果没有线程安全的策略，线程并不是特别有用.

# Hash Maps

标准库提供了[`std.AutoHashMap`](https://ziglang.org/documentation/master/std/#A;std:AutoHashMap)，它可以让你轻松地从一个键类型和一个值类型创建一个哈希图类型。这些必须用分配器来启动。

让我们把一些值放在一个哈希图中。

```zig
test "hashing" {
    const Point = struct { x: i32, y: i32 };

    var map = std.AutoHashMap(u32, Point).init(
        test_allocator,
    );
    defer map.deinit();

    try map.put(1525, .{ .x = 1, .y = -4 });
    try map.put(1550, .{ .x = 2, .y = -3 });
    try map.put(1575, .{ .x = 3, .y = -2 });
    try map.put(1600, .{ .x = 4, .y = -1 });

    try expect(map.count() == 4);

    var sum = Point{ .x = 0, .y = 0 };
    var iterator = map.iterator();

    while (iterator.next()) |entry| {
        sum.x += entry.value_ptr.x;
        sum.y += entry.value_ptr.y;
    }

    try expect(sum.x == 10);
    try expect(sum.y == -10);
}
```

`.fetchPut`在哈希图中放入一个值，如果之前有一个该键的值，则返回一个值 .

```zig
test "fetchPut" {
    var map = std.AutoHashMap(u8, f32).init(
        test_allocator,
    );
    defer map.deinit();

    try map.put(255, 10);
    const old = try map.fetchPut(255, 100);

    try expect(old.?.value == 10);
    try expect(map.get(255).? == 100);
}
```

[`std.StringHashMap`](https://ziglang.org/documentation/master/std/#A;std:StringHashMap)也是为你需要字符串作为键值时提供 .

```zig
test "string hashmap" {
    var map = std.StringHashMap(enum { cool, uncool }).init(
        test_allocator,
    );
    defer map.deinit();

    try map.put("loris", .uncool);
    try map.put("me", .cool);

    try expect(map.get("me").? == .cool);
    try expect(map.get("loris").? == .uncool);
}
```

[`std.StringHashMap`](https://ziglang.org/documentation/master/std/#A;std:StringHashMap)和[`std.AutoHashMap`](https://ziglang.org/documentation/master/std/#A;std:AutoHashMap)只是[`std.HashMap`](https://ziglang.org/documentation/master/std/#A;std:HashMap)的包装器。如果这两者不能满足你的需求，直接使用[`std.HashMap`](https://ziglang.org/documentation/master/std/#A;std:HashMap)可以给你更多的控制。

如果你想让你的元素有一个数组的支持，可以试试[`std.ArrayHashMap`](https://ziglang.org/documentation/master/std/#A;std:ArrayHashMap)及其包装器[`std.AutoArrayHashMap`](https://ziglang.org/documentation/master/std/#A;std:AutoArrayHashMap)。

# Stacks

[`std.ArrayList`](https://ziglang.org/documentation/master/std/#A;std:ArrayList)提供了将其作为堆栈的必要方法。下面是一个创建匹配括号列表的例子 .

```zig
test "stack" {
    const string = "(()())";
    var stack = std.ArrayList(usize).init(
        test_allocator,
    );
    defer stack.deinit();

    const Pair = struct { open: usize, close: usize };
    var pairs = std.ArrayList(Pair).init(
        test_allocator,
    );
    defer pairs.deinit();

    for (string, 0..) |char, i| {
        if (char == '(') try stack.append(i);
        if (char == ')')
            try pairs.append(.{
                .open = stack.pop(),
                .close = i,
            });
    }

    for (pairs.items, 0..) |pair, i| {
        try expect(std.meta.eql(pair, switch (i) {
            0 => Pair{ .open = 1, .close = 2 },
            1 => Pair{ .open = 3, .close = 4 },
            2 => Pair{ .open = 0, .close = 5 },
            else => unreachable,
        }));
    }
}
```

# Sorting 排序

标准库提供了用于原地排序片的实用程序。其基本用法如下 .

```zig
test "sorting" {
    var data = [_]u8{ 10, 240, 0, 0, 10, 5 };
    std.sort.sort(u8, &data, {}, comptime std.sort.asc(u8));
    try expect(eql(u8, &data, &[_]u8{ 0, 0, 5, 10, 10, 240 }));
    std.sort.sort(u8, &data, {}, comptime std.sort.desc(u8));
    try expect(eql(u8, &data, &[_]u8{ 240, 10, 10, 5, 0, 0 }));
}
```

[`std.sort.asc`](https://ziglang.org/documentation/master/std/#A;std:sort.asc)和[`.desc`](https://ziglang.org/documentation/master/std/#A;std:sort.desc)在comptime为指定类型创建一个比较函数；如果非数字类型应该被排序，用户必须提供自己的比较函数。

[`std.sort.sort`](https://ziglang.org/documentation/master/std/#A;std:sort.sort)的最佳情况是O(n)，平均和最差情况是O(n*log(n)) .

# Iterators 迭代器

一个结构类型有一个`next`函数，其返回类型是可选的，这是一个常见的习惯，这样函数可以返回一个null，表示迭代结束 .

[`std.mem.SplitIterator`](https://ziglang.org/documentation/master/std/#A;std:mem.SplitIterator)（和有细微差别的[`std.mem.TokenIterator`](https://ziglang.org/documentation/master/std/#A;std:mem.TokenIterator)）是这种模式的一个例子 .

```zig
test "split iterator" {
    const text = "robust, optimal, reusable, maintainable, ";
    var iter = std.mem.split(u8, text, ", ");
    try expect(eql(u8, iter.next().?, "robust"));
    try expect(eql(u8, iter.next().?, "optimal"));
    try expect(eql(u8, iter.next().?, "reusable"));
    try expect(eql(u8, iter.next().?, "maintainable"));
    try expect(eql(u8, iter.next().?, ""));
    try expect(iter.next() == null);
}
```

一些迭代器有一个`！?T`的返回类型，而不是?T。`!?T`要求我们在可选的之前解开错误联合，这意味着为进入下一个迭代所做的工作可能会出错。下面是一个用循环做这个的例子。[`cwd`](https://ziglang.org/documentation/master/std/#std;fs.cwd)必须以迭代权限打开，以使目录迭代器工作 .

```zig
test "iterator looping" {
    var iter = (try std.fs.cwd().openIterableDir(
        ".",
        .{},
    )).iterate();

    var file_count: usize = 0;
    while (try iter.next()) |entry| {
        if (entry.kind == .File) file_count += 1;
    }

    try expect(file_count > 0);
}
```

这里我们将实现一个自定义的迭代器。这将在一个字符串切片上进行迭代，产生包含一个给定字符串的字符串 .

```zig
const ContainsIterator = struct {
    strings: []const []const u8,
    needle: []const u8,
    index: usize = 0,
    fn next(self: *ContainsIterator) ?[]const u8 {
        const index = self.index;
        for (self.strings[index..]) |string| {
            self.index += 1;
            if (std.mem.indexOf(u8, string, self.needle)) |_| {
                return string;
            }
        }
        return null;
    }
};

test "custom iterator" {
    var iter = ContainsIterator{
        .strings = &[_][]const u8{ "one", "two", "three" },
        .needle = "e",
    };

    try expect(eql(u8, iter.next().?, "one"));
    try expect(eql(u8, iter.next().?, "three"));
    try expect(iter.next() == null);
}
```

# Formatting specifiers 格式化指定器

[`std.fmt`](https://ziglang.org/documentation/master/std/#std;fmt)提供格式化各种数据类型的选项。

`std.fmt.fmtSliceHexLower`和`std.fmt.fmtSliceHexUpper`为字符串提供十六进制格式，以及为ints提供`{x}`和`{X}`。

```zig
const bufPrint = std.fmt.bufPrint;

test "hex" {
    var b: [8]u8 = undefined;

    _ = try bufPrint(&b, "{X}", .{4294967294});
    try expect(eql(u8, &b, "FFFFFFFE"));

    _ = try bufPrint(&b, "{x}", .{4294967294});
    try expect(eql(u8, &b, "fffffffe"));

    _ = try bufPrint(&b, "{}", .{std.fmt.fmtSliceHexLower("Zig!")});
    try expect(eql(u8, &b, "5a696721"));
}
```

`{d}` 对数字类型执行十进制格式化.

```zig
test "decimal float" {
    var b: [4]u8 = undefined;
    try expect(eql(
        u8,
        try bufPrint(&b, "{d}", .{16.5}),
        "16.5",
    ));
}
```

`{c}` 将一个字节格式化为一个ascii字符.
```zig
test "ascii fmt" {
    var b: [1]u8 = undefined;
    _ = try bufPrint(&b, "{c}", .{66});
    try expect(eql(u8, &b, "B"));
}
```

`std.fmt.fmtIntSizeDec`和`std.fmt.fmtIntSizeBin`以公制(1000)和二次方(1024)的符号输出内存大小 .

```zig
test "B Bi" {
    var b: [32]u8 = undefined;

    try expect(eql(u8, try bufPrint(&b, "{}", .{std.fmt.fmtIntSizeDec(1)}), "1B"));
    try expect(eql(u8, try bufPrint(&b, "{}", .{std.fmt.fmtIntSizeBin(1)}), "1B"));

    try expect(eql(u8, try bufPrint(&b, "{}", .{std.fmt.fmtIntSizeDec(1024)}), "1.024kB"));
    try expect(eql(u8, try bufPrint(&b, "{}", .{std.fmt.fmtIntSizeBin(1024)}), "1KiB"));

    try expect(eql(
        u8,
        try bufPrint(&b, "{}", .{std.fmt.fmtIntSizeDec(1024 * 1024 * 1024)}),
        "1.073741824GB",
    ));
    try expect(eql(
        u8,
        try bufPrint(&b, "{}", .{std.fmt.fmtIntSizeBin(1024 * 1024 * 1024)}),
        "1GiB",
    ));
}
```

`{b}` and `{o}` 以二进制和八进制格式输出整数.

```zig
test "binary, octal fmt" {
    var b: [8]u8 = undefined;

    try expect(eql(
        u8,
        try bufPrint(&b, "{b}", .{254}),
        "11111110",
    ));

    try expect(eql(
        u8,
        try bufPrint(&b, "{o}", .{254}),
        "376",
    ));
}
```

`{*}` 执行指针格式化，打印地址而不是数值.
```zig
test "pointer fmt" {
    var b: [16]u8 = undefined;
    try expect(eql(
        u8,
        try bufPrint(&b, "{*}", .{@intToPtr(*u8, 0xDEADBEEF)}),
        "u8@deadbeef",
    ));
}
```

`{e}` 以科学记数法输出浮点数.
```zig
test "scientific" {
    var b: [16]u8 = undefined;

    try expect(eql(
        u8,
        try bufPrint(&b, "{e}", .{3.14159}),
        "3.14159e+00",
    ));
}
```

`{s}` 输出字符串.
```zig
test "string fmt" {
    var b: [6]u8 = undefined;
    const hello: [*:0]const u8 = "hello!";

    try expect(eql(
        u8,
        try bufPrint(&b, "{s}", .{hello}),
        "hello!",
    ));
}
```

此清单并非详尽无遗 .

# Advanced Formatting 高级格式化

到目前为止，我们只涉及了格式化指定器。格式化字符串实际上遵循这种格式，在每一对方括号之间是一个参数，你必须用一些东西来替换 .

`{[position][specifier]:[fill][alignment][width].[precision]}`

| Name      | Meaning                                                                                 |
|-----------|-----------------------------------------------------------------------------------------|
| Position  | 要插入的参数的索引                                       |
| Specifier | 与类型相关的格式化选项                                                      |
| Fill      | 用于填充的单个字符                                                     |
| Alignment | 三个字符“<”、“^”或“>”之一;这些是左中右对齐以上翻译结果来自有道神经网络翻译（YNMT）· 通用场景    |
| Width     | 字段的总宽度(字符)                                              |
| Precision | 一个格式化的数字应该有多少位小数                                        |


Position usage.
```zig
test "position" {
    var b: [3]u8 = undefined;
    try expect(eql(
        u8,
        try bufPrint(&b, "{0s}{0s}{1s}", .{ "a", "b" }),
        "aab",
    ));
}
```

Fill, alignment and width being used.
```zig
test "fill, alignment, width" {
    var b: [6]u8 = undefined;

    try expect(eql(
        u8,
        try bufPrint(&b, "{s: <5}", .{"hi!"}),
        "hi!  ",
    ));

    try expect(eql(
        u8,
        try bufPrint(&b, "{s:_^6}", .{"hi!"}),
        "_hi!__",
    ));

    try expect(eql(
        u8,
        try bufPrint(&b, "{s:!>4}", .{"hi!"}),
        "!hi!",
    ));
}
```

Using a specifier with precision.
```zig
test "precision" {
    var b: [4]u8 = undefined;
    try expect(eql(
        u8,
        try bufPrint(&b, "{d:.2}", .{3.14159}),
        "3.14",
    ));
}
```

# End of Chapter 2

这一章是不完整的。在未来，它将包含诸如以下内容：:

- Arbitrary Precision Maths 任意精度的数学
- Linked Lists  链表
- Queues        队列
- Mutexes       互斥程序
- Atomics       原子
- Searching     搜索
- Logging       记录

我们欢迎反馈和PRs.