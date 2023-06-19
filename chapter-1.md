---
title: "Chapter 1 - 基础知识"
weight: 2
date: 2023-04-28 18:00:00
description: "Chapter 1 - 这将使你快速掌握几乎所有的Zig编程语言。这一部分的教程应该可以在一小时内完成."
---

# Assignment 赋值

赋值语法: `(const|var) identifier[: type] = value`.

* `const`表示`identifier`是一个**constant 常量**，存储一个不可变的值。
* `var`表示`identifier`是一个**variable 变量**，存储一个可变的值。
* `: type`是对`identifier`的类型注释，如果可以推断出`value`的数据类型，可以省略。

<!--no_test-->
```zig
const constant: i32 = 5;  // signed 32-bit constant
var variable: u32 = 5000; // unsigned 32-bit variable

// @as performs an explicit type coercion
const inferred_constant = @as(i32, 5);
var inferred_variable = @as(u32, 5000);
```

常量和变量*必须*有一个值。如果不能给出已知的值，只要提供一个类型注释，就可以使用[`undefined`](https://ziglang.org/documentation/master/#undefined)值，它可以强制到任何类型 .

<!--no_test-->
```zig
const a: i32 = undefined;
var b: u32 = undefined;
```

在可能的情况下，`const`值比`var`值更受欢迎 .

# Arrays

数组用`[N]T`表示，其中`N`是数组中元素的数量，`T`是这些元素的类型（即数组的子类型）。

对于数组字面，`N`可以用`_`代替，以推断出数组的大小 .

<!--no_test-->
```zig
const a = [5]u8{ 'h', 'e', 'l', 'l', 'o' };
const b = [_]u8{ 'w', 'o', 'r', 'l', 'd' };
```

要获得一个数组的大小，只需访问该数组的`len'字段 .

<!--no_test-->
```zig
const array = [_]u8{ 'h', 'e', 'l', 'l', 'o' };
const length = array.len; // 5
```

# If

Zig的基本if语句很简单，它只接受一个`bool`值（值为`true`或`false`）。没有真值或假值的概念。

这里我们将介绍测试。保存下面的代码，用`zig test file-name.zig`编译+运行它。我们将使用标准库中的[`expect`](https://ziglang.org/documentation/master/std/#std;testing.expect)函数，如果给定的值是`false`，则会导致测试失败。当测试失败时，将显示错误和堆栈跟踪 .

```zig
const expect = @import("std").testing.expect;

test "if statement" {
    const a = true;
    var x: u16 = 0;
    if (a) {
        x += 1;
    } else {
        x += 2;
    }
    try expect(x == 1);
}
```

If语句也可以作为表达式使用.

```zig
test "if statement expression" {
    const a = true;
    var x: u16 = 0;
    x += if (a) 1 else 2;
    try expect(x == 1);
}
```

# While

Zig的while循环有三个部分--一个条件、一个块和一个继续表达式。


没有继续表达式 .
```zig
test "while" {
    var i: u8 = 2;
    while (i < 100) {
        i *= 2;
    }
    try expect(i == 128);
}
```

用`continue`的方式表达.
```zig
test "while with continue expression" {
    var sum: u8 = 0;
    var i: u8 = 1;
    while (i <= 10) : (i += 1) {
        sum += i;
    }
    try expect(sum == 55);
}
```

使用 `continue`.

```zig
test "while with continue" {
    var sum: u8 = 0;
    var i: u8 = 0;
    while (i <= 3) : (i += 1) {
        if (i == 2) continue;
        sum += i;
    }
    try expect(sum == 4);
}
```

使用 `break`.

```zig
test "while with break" {
    var sum: u8 = 0;
    var i: u8 = 0;
    while (i <= 3) : (i += 1) {
        if (i == 2) break;
        sum += i;
    }
    try expect(sum == 1);
}
```

# For
For 循环用于迭代数组（和其他类型，将在后面讨论）。For 循环遵循这个语法。像while一样，for循环可以使用`break`和`continue`。这里我们不得不给`_`赋值，因为Zig不允许我们有未使用的值 .

```zig
test "for" {
    //字符串相当于整数字串
    const string = [_]u8{ 'a', 'b', 'c' };

    for (string, 0..) |character, index| {
        _ = character;
        _ = index;
    }

    for (string) |character| {
        _ = character;
    }

    for (string, 0..) |_, index| {
        _ = index;
    }

    for (string) |_| {}
}
```

# Functions

__所有的函数参数都是不可改变的__ - 如果需要一个副本，用户必须明确地制作一个。与变量的蛇形大小写不同，函数是骆驼大写的。下面是一个声明和调用一个简单函数的例子 .

```zig
fn addFive(x: u32) u32 {
    return x + 5;
}

test "function" {
    const y = addFive(0);
    try expect(@TypeOf(y) == u32);
    try expect(y == 5);
}
```

允许递归:
```zig
fn fibonacci(n: u16) u16 {
    if (n == 0 or n == 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

test "function recursion" {
    const x = fibonacci(10);
    try expect(x == 55);
}
```
当递归发生时，编译器不再能够计算出最大的堆栈大小。这可能会导致不安全的行为--堆栈溢出。关于如何实现安全递归的细节将在今后介绍。


通过使用`_`来代替变量或常量声明，可以忽略数值。这在全局范围内不起作用（也就是说，它只在函数和块内起作用），如果你不需要函数返回的值，这对忽略这些值很有用 .
<!--no_test-->
```zig
_ = 10;
```

# Defer

Defer用于退出当前代码块时执行一个语句 .

```zig
test "defer" {
    var x: i16 = 5;
    {
        defer x += 2;
        try expect(x == 5);
    }
    try expect(x == 7);
}
```
当一个区块中存在多个defer时，将按相反的顺序执行 .

```zig
test "multi defer" {
    var x: f32 = 5;
    {
        defer x += 2;
        defer x /= 2;
    }
    try expect(x == 4.5);
}
```

# Errors

一个错误集就像一个枚举（关于Zig的枚举的细节将在后面介绍），其中每个错误集是一个值。在Zig中没有例外，错误就是值。让我们来创建一个错误集 .

```zig
const FileOpenError = error{
    AccessDenied,
    OutOfMemory,
    FileNotFound,
};
```
错误集强制到它们的超集 .

```zig
const AllocationError = error{OutOfMemory};

test "coerce error from a subset to a superset" {
    const err: FileOpenError = AllocationError.OutOfMemory;
    try expect(err == FileOpenError.OutOfMemory);
}
```

一个错误集类型和一个正常类型可以用`！`操作符结合起来，形成一个错误联合类型。这些类型的值可以是一个错误值，也可以是一个正常类型的值。

让我们来创建一个错误联合类型的值。这里使用了[`catch`](https://ziglang.org/documentation/master/#catch)，它后面是一个表达式，当它前面的值是一个错误时，这个表达式会被评估。这里的catch用来提供一个回退值，但也可以是一个[`noreturn`](https://ziglang.org/documentation/master/#noreturn) - `return`、`while (true)`等的类型 .

```zig
test "error union" {
    const maybe_error: AllocationError!u16 = 10;
    const no_error = maybe_error catch 0;

    try expect(@TypeOf(no_error) == u16);
    try expect(no_error == 10);
}
```

函数经常返回错误的联合体。这里有一个使用catch的，其中`|err|`语法接收错误的值。这被称为__payload capturing__，在很多地方都有类似的用法。我们将在本章后面更详细地讨论它。题外话：有些语言对lambdas使用类似的语法--Zig不是这种情况 .

```zig
fn failingFunction() error{Oops}!void {
    return error.Oops;
}

test "returning an error" {
    failingFunction() catch |err| {
        try expect(err == error.Oops);
        return;
    };
}
```

`try x`是`x catch |err| return err`的快捷方式，常用于不适合处理错误的地方。Zig的[`try`](https://ziglang.org/documentation/master/#try)和[`catch`](https://ziglang.org/documentation/master/#catch)与其他语言的try-catch没有关系 .

```zig
fn failFn() error{Oops}!i32 {
    try failingFunction();
    return 12;
}

test "try" {
    var v = failFn() catch |err| {
        try expect(err == error.Oops);
        return;
    };
    try expect(v == 12); // is never reached
}
```

[`errdefer`](https://ziglang.org/documentation/master/#errdefer)的工作方式与[`defer`](https://ziglang.org/documentation/master/#defer)类似，但只在[`errdefer`](https://ziglang.org/documentation/master/#errdefer)的块内有错误返回时执行 .

```zig
var problems: u32 = 98;

fn failFnCounter() error{Oops}!void {
    errdefer problems += 1;
    try failingFunction();
}

test "errdefer" {
    failFnCounter() catch |err| {
        try expect(err == error.Oops);
        try expect(problems == 99);
        return;
    };
}
```

从一个函数返回的错误联合体可以通过没有明确的错误集来推断其错误集。这个推断的错误集包含了该函数可能返回的所有可能的错误 .

```zig
fn createFile() !void {
    return error.AccessDenied;
}

test "inferred error set" {
    //type coercion successfully takes place
    const x: error{AccessDenied}!void = createFile();

    //Zig does not let us ignore error unions via _ = x;
    //we must unwrap it with "try", "catch", or "if" by any means
    _ = x catch {};
}
```

错误集可以被合并.

```zig
const A = error{ NotDir, PathNotFound };
const B = error{ OutOfMemory, PathNotFound };
const C = A || B;
```

`anyerror`是全局错误集，由于它是所有错误集的超集，可以将任何错误集的错误凝聚成它的一个值。一般应避免使用它 .

# Switch

Zig的 `switch` 工作方式既可以作为一个语句，也可以作为一个表达式。所有分支的类型必须与被切换的类型相一致。所有可能的值都必须有一个相关的分支--值不能被遗漏。案例不能落到其他分支上。

一个切换语句的例子。要求else来满足这个切换的穷举性 .

```zig
test "switch statement" {
    var x: i8 = 10;
    switch (x) {
        -1...1 => {
            x = -x;
        },
        10, 100 => {
            //special considerations must be made
            //when dividing signed integers
            x = @divExact(x, 10);
        },
        else => {},
    }
    try expect(x == 1);
}
```

这里是前者，但作为一个转换表达式 .
```zig
test "switch expression" {
    var x: i8 = 10;
    x = switch (x) {
        -1...1 => -x,
        10, 100 => @divExact(x, 10),
        else => x,
    };
    try expect(x == 1);
}
```

# Runtime Safety

Zig提供了一个安全级别，在执行过程中可能会发现问题。安全性可以被打开，也可以被关闭。Zig有很多所谓的 "可检测的非法行为 "的案例，这意味着非法行为在安全状态下会被发现（导致恐慌），但在安全状态下会导致未定义行为。我们强烈建议用户在开发和测试他们的软件时使用安全功能，尽管其速度会受到影响。

例如，运行时安全可以保护你免受越界指数的影响 .

<!--fail_test-->
```zig
test "out of bounds" {
    const a = [3]u8{ 1, 2, 3 };
    var index: u8 = 5;
    const b = a[index];
    _ = b;
}
```
```
test "out of bounds"...index out of bounds
.\tests.zig:43:14: 0x7ff698cc1b82 in test "out of bounds" (test.obj)
    const b = a[index];
             ^
```

用户可以通过使用内置函数[`@setRuntimeSafety`](https://ziglang.org/documentation/master/#setRuntimeSafety)选择禁用当前块的运行时安全 .

```zig
test "out of bounds, no safety" {
    @setRuntimeSafety(false);
    const a = [3]u8{ 1, 2, 3 };
    var index: u8 = 5;
    const b = a[index];
    _ = b;
}
```

在某些构建模式下，安全是关闭的（稍后讨论）.

# Unreachable

[`unreachable`](https://ziglang.org/documentation/master/#unreachable)是对编译器的一个断言，即这个语句将不会被到达。它可以用来告诉编译器，一个分支是不可能的，然后优化器可以利用它。到达一个[`unreachable`](https://ziglang.org/documentation/master/#unreachable)是可检测的非法行为。

由于它是[`noreturn`](https://ziglang.org/documentation/master/#noreturn)的类型，它与所有其他类型兼容。在这里，它被强制转化为u32 .
<!--fail_test-->
```zig
test "unreachable" {
    const x: i32 = 1;
    const y: u32 = if (x == 2) 5 else unreachable;
    _ = y;
}
```
```
test "unreachable"...reached unreachable code
.\tests.zig:211:39: 0x7ff7e29b2049 in test "unreachable" (test.obj)
    const y: u32 = if (x == 2) 5 else unreachable;
                                      ^
```

这里是在switch中使用 unreachable .
```zig
fn asciiToUpper(x: u8) u8 {
    return switch (x) {
        'a'...'z' => x + 'A' - 'a',
        'A'...'Z' => x,
        else => unreachable,
    };
}

test "unreachable switch" {
    try expect(asciiToUpper('a') == 'A');
    try expect(asciiToUpper('A') == 'A');
}
```

# Pointers

Zig中的普通指针不允许有0或null作为值。它们遵循的语法是`*T`，其中`T`是子类型。

引用是用`&variable`进行的，而取消引用是用`variable.*`进行的 .

```zig
fn increment(num: *u8) void {
    num.* += 1;
}

test "pointers" {
    var x: u8 = 1;
    increment(&x);
    try expect(x == 2);
}
```

试图将一个`*T`设置为0的值是可以检测到的非法行为 .

<!--fail_test-->
```zig
test "naughty pointer" {
    var x: u16 = 0;
    var y: *u8 = @intToPtr(*u8, x);
    _ = y;
}
```
```
test "naughty pointer"...cast causes pointer to be null
.\tests.zig:241:18: 0x7ff69ebb22bd in test "naughty pointer" (test.obj)
    var y: *u8 = @intToPtr(*u8, x);
                 ^
```

Zig也有常态指针，它不能用来修改被引用的数据。引用一个常量变量将产生一个常量指针 .
<!--fail_test-->
```zig
test "const pointers" {
    const x: u8 = 1;
    var y = &x;
    y.* += 1;
}
```
```
error: cannot assign to constant
    y.* += 1;
        ^
```

A `*T` 强制转换成 `*const T`.


# Pointer sized integers 指针大小的整数

`usize`和`isize`是以无符号和有符号的整数形式给出的，与指针大小相同.

```zig
test "usize" {
    try expect(@sizeOf(usize) == @sizeOf(*u8));
    try expect(@sizeOf(isize) == @sizeOf(*u8));
}
```

# Many-Item Pointers 

有时你可能有一个指向未知数量元素的指针。`[*]T`是解决这个问题的方法，它的工作方式与`*T`类似，但也支持索引语法、指针算术和分片。不像`*T`，它不能指向一个没有已知大小的类型。`*T`会强制变成`[*]T`。

这些许多指针可以指向任何数量的元素，包括0和1 .

# Slices

Slices可以被认为是一对`[*]T`（数据的指针）和`usize`（元素的数量）。它们的语法是`[]T`，其中`T`是子类型。当你需要对任意数量的数据进行操作时，Slices在整个Zig中被大量使用。Slices具有与指针相同的属性，这意味着也存在常量切片。For循环也可以在分片上操作。Zig中的字符串字面意思是"[]const u8"。

这里，语法`x[n..m]`被用来从一个数组中创建一个切片。这被称为 __slicing 切片__，创建一个从`x[n]`开始，到`x[m - 1]`结束的元素切片。这个例子使用了一个常量分片，因为分片所指向的值不需要被修改 .

```zig
fn total(values: []const u8) usize {
    var sum: usize = 0;
    for (values) |v| sum += v;
    return sum;
}
test "slices" {
    const array = [_]u8{ 1, 2, 3, 4, 5 };
    const slice = array[0..3];
    try expect(total(slice) == 6);
}
```

当这些' n '和' m '值在编译时都已知时，切片实际上会生成一个指向数组的指针。这不是一个指向数组的指针的问题。' *[N]T '将强制转换为' []T ' .

```zig
test "slices 2" {
    const array = [_]u8{ 1, 2, 3, 4, 5 };
    const slice = array[0..3];
    try expect(@TypeOf(slice) == *const [3]u8);
}
```

语法`x[n..]`可以用于切到最后.

```zig
test "slices 3" {
    var array = [_]u8{ 1, 2, 3, 4, 5 };
    var slice = array[0..];
    _ = slice;
}
```

可用于切片的类型有：arrays 数组、许多指针和 slices 切片。 .

# Enums

Zig枚举允许您定义具有有限命名值集的类型 .

定义一个枚举.
```zig
const Direction = enum { north, south, east, west };
```

枚举类型可以有指定的(整数)标记类型.
```zig
const Value = enum(u2) { zero, one, two };
```

Enum的序数从0开始。它们可以通过内置函数[' @enumToInt '](https://ziglang.org/documentation/master/#enumToInt)访问 .
```zig
test "enum ordinal value" {
    try expect(@enumToInt(Value.zero) == 0);
    try expect(@enumToInt(Value.one) == 1);
    try expect(@enumToInt(Value.two) == 2);
}
```

值可以被覆盖，下一个值从这里继续 .
```zig
const Value2 = enum(u32) {
    hundred = 100,
    thousand = 1000,
    million = 1000000,
    next,
};

test "set enum ordinal value" {
    try expect(@enumToInt(Value2.hundred) == 100);
    try expect(@enumToInt(Value2.thousand) == 1000);
    try expect(@enumToInt(Value2.million) == 1000000);
    try expect(@enumToInt(Value2.next) == 1000001);
}
```

方法可以被赋予枚举。这些方法作为命名的函数，可以用点语法来调用 .

```zig
const Suit = enum {
    clubs,
    spades,
    diamonds,
    hearts,
    pub fn isClubs(self: Suit) bool {
        return self == Suit.clubs;
    }
};

test "enum method" {
    try expect(Suit.spades.isClubs() == Suit.isClubs(.spades));
}
```

枚举也可以被赋予`var`和`const`声明。这些作为命名的球，它们的值与枚举类型的实例无关，也不相连 .

```zig
const Mode = enum {
    var count: u32 = 0;
    on,
    off,
};

test "hmm" {
    Mode.count += 1;
    try expect(Mode.count == 1);
}
```


# Structs

结构是Zig最常见的一种复合数据类型，允许你定义可以存储一组固定的命名字段的类型。Zig不保证结构中字段的内存顺序，也不保证其大小。和数组一样，结构也是用`T{}`语法整齐地构建的。下面是一个声明和填充结构的例子 .
```zig
const Vec3 = struct { x: f32, y: f32, z: f32 };

test "struct usage" {
    const my_vector = Vec3{
        .x = 0,
        .y = 100,
        .z = 50,
    };
    _ = my_vector;
}
```

所有字段必须被赋予一个值.

<!--fail_test-->
```zig
test "missing struct field" {
    const my_vector = Vec3{
        .x = 0,
        .z = 50,
    };
    _ = my_vector;
}
```
```
error: missing field: 'y'
    const my_vector = Vec3{
                        ^
```

字段可能是默认值:
```zig
const Vec4 = struct { x: f32, y: f32, z: f32 = 0, w: f32 = undefined };

test "struct defaults" {
    const my_vector = Vec4{
        .x = 25,
        .y = -50,
    };
    _ = my_vector;
}
```

和枚举一样，结构体也可以包含函数和声明。

结构体有一个独特的特性，即当给定一个指向结构体的指针时，在访问字段时，会自动进行一级的解引用。注意在这个例子中，self.x和self.y是如何在交换函数中被访问的，而不需要对self指针进行脱引 .

```zig
const Stuff = struct {
    x: i32,
    y: i32,
    fn swap(self: *Stuff) void {
        const tmp = self.x;
        self.x = self.y;
        self.y = tmp;
    }
};

test "automatic dereference" {
    var thing = Stuff{ .x = 10, .y = 20 };
    thing.swap();
    try expect(thing.x == 20);
    try expect(thing.y == 10);
}
```

# Unions

Zig的联合体允许你定义类型，在许多可能的类型字段中存储一个值；在同一时间，只有一个字段可能是有效的。

裸联合体类型没有一个保证的内存布局。正因为如此，裸联合体不能被用来重新解释内存。访问一个未激活的联合中的字段是可以检测到的非法行为 .

<!--fail_test-->
```zig
const Result = union {
    int: i64,
    float: f64,
    bool: bool,
};

test "simple union" {
    var result = Result{ .int = 1234 };
    result.float = 12.34;
}
```
```
test "simple union"...access of inactive union field
.\tests.zig:342:12: 0x7ff62c89244a in test "simple union" (test.obj)
    result.float = 12.34;
           ^
```

标签联合体是使用枚举来检测哪个字段处于活动状态的联合体。在这里，我们再次利用有效载荷捕获，在捕获它所包含的值的同时，打开一个联合体的标签类型。这里我们使用 *pointer capture 指针捕获* ；捕获的值是不可改变的，但是使用`|*value|`语法，我们可以捕获一个指向值的指针，而不是值本身。这使得我们可以使用取消引用的方法来改变原始值 .

```zig
const Tag = enum { a, b, c };

const Tagged = union(Tag) { a: u8, b: f32, c: bool };

test "switch on tagged union" {
    var value = Tagged{ .b = 1.5 };
    switch (value) {
        .a => |*byte| byte.* += 1,
        .b => |*float| float.* *= 2,
        .c => |*b| b.* = !b.*,
    }
    try expect(value.b == 3);
}
```

一个有标签的联合体的标签类型也可以被推断出来。这等同于上面的标签类型 .

<!--no_test-->
```zig
const Tagged = union(enum) { a: u8, b: f32, c: bool };
```

`void`成员类型可以在语法中省略其类型。这里，没有一个是`void`类型的 .

```zig
const Tagged2 = union(enum) { a: u8, b: f32, c: bool, none };
```

# Integer Rules 整数规则

Zig支持十六进制、八进制和二进制的整数字 .
```zig
const decimal_int: i32 = 98222;
const hex_int: u8 = 0xff;
const another_hex_int: u8 = 0xFF;
const octal_int: u16 = 0o755;
const binary_int: u8 = 0b11110000;
```
下划线也可以放在数字之间作为视觉分隔符 .
```zig
const one_billion: u64 = 1_000_000_000;
const binary_mask: u64 = 0b1_1111_1111;
const permissions: u64 = 0o7_5_5;
const big_address: u64 = 0xFF80_0000_0000_0000;
```

允许 "整数拓宽"，这意味着一种类型的整数可以胁迫为另一种类型的整数，条件是新类型可以容纳旧类型的所有数值 .

```zig
test "integer widening" {
    const a: u8 = 250;
    const b: u16 = a;
    const c: u32 = b;
    try expect(c == a);
}
```

如果你有一个存储在整数中的值，但不能胁迫到你想要的类型，[`@intCast`](https://ziglang.org/documentation/master/#intCast)可以用来明确地从一种类型转换到另一种。如果给定的值超出了目标类型的范围，这就是可检测的非法行为 .

```zig
test "@intCast" {
    const x: u64 = 200;
    const y = @intCast(u8, x);
    try expect(@TypeOf(y) == u8);
}
```

默认情况下，整数是不允许溢出的。溢出是可检测的非法行为。有时候，能够以一种明确定义的方式溢出整数是人们所希望的行为。对于这种使用情况，Zig提供了溢出操作符 .

| Normal Operator | Wrapping Operator |
|-----------------|-------------------|
| +               | +%                |
| -               | -%                |
| *               | *%                |
| +=              | +%=               |
| -=              | -%=               |
| *=              | *%=               |

```zig
test "well defined overflow" {
    var a: u8 = 255;
    a +%= 1;
    try expect(a == 0);
}
```

# Floats

Zig的浮点数是严格符合IEEE标准的，除非使用[`@setFloatMode(.Optimized)`](https://ziglang.org/documentation/master/#setFloatMode)，这相当于GCC的`-ffast-math`。浮点数与更大的浮点数类型紧密相连 .

```zig
test "float widening" {
    const a: f16 = 0;
    const b: f32 = a;
    const c: f128 = b;
    try expect(c == @as(f128, a));
}
```

Floats 支持多种字面量.
```zig
const floating_point: f64 = 123.0E+77;
const another_float: f64 = 123.0;
const yet_another: f64 = 123.0e+77;

const hex_floating_point: f64 = 0x103.70p-5;
const another_hex_float: f64 = 0x103.70;
const yet_another_hex_float: f64 = 0x103.70P-5;
```
下划线也可以放在数字之间.
```zig
const lightspeed: f64 = 299_792_458.000_000;
const nanosecond: f64 = 0.000_000_001;
const more_hex: f64 = 0x1234_5678.9ABC_CDEFp-10;
```

整数和浮点数可以使用内置函数[`@intToFloat`](https://ziglang.org/documentation/master/#intToFloat)和[`@floatToInt`](https://ziglang.org/documentation/master/#floatToInt)进行转换。[`@intToFloat`](https://ziglang.org/documentation/master/#intToFloat)总是安全的，而[`@floatToInt`](https://ziglang.org/documentation/master/#floatToInt)如果浮点数不能适应整数目标类型，则是可检测的非法行为 .

```zig
test "int-float conversion" {
    const a: i32 = 0;
    const b = @intToFloat(f32, a);
    const c = @floatToInt(i32, b);
    try expect(c == a);
}
```

# Labelled Blocks 标记代码块

Zig中的块是表达式，可以被赋予标签，用来产生数值。这里，我们使用的是一个叫做blk的标签。块产生值，意味着它们可以用来代替一个值。一个空块`{}`的值是一个`void`类型的值 .

```zig
test "labelled blocks" {
    const count = blk: {
        var sum: u32 = 0;
        var i: u32 = 0;
        while (i < 10) : (i += 1) sum += i;
        break :blk sum;
    };
    try expect(count == 45);
    try expect(@TypeOf(count) == u32);
}
```
这可以看作是相当于C的`i++` .
<!--no_test-->
```zig
blk: {
    const tmp = i;
    i += 1;
    break :blk tmp;
}
```

# Labelled Loops 便签循环

循环可以被赋予标签，允许你对外部循环进行 "break "和 "continue" .

```zig
test "nested continue" {
    var count: usize = 0;
    outer: for ([_]i32{ 1, 2, 3, 4, 5, 6, 7, 8 }) |_| {
        for ([_]i32{ 1, 2, 3, 4, 5 }) |_| {
            count += 1;
            continue :outer;
        }
    }
    try expect(count == 8);
}
```

# Loops as expressions 循环作为表达式

与`return`一样，`break`接受一个值。这可以用来从一个循环中获得一个值。Zig中的循环也有一个`else`分支，当循环没有被`break`退出时，该分支会被评估 .

```zig
fn rangeHasNumber(begin: usize, end: usize, number: usize) bool {
    var i = begin;
    return while (i < end) : (i += 1) {
        if (i == number) {
            break true;
        }
    } else false;
}

test "while loop expression" {
    try expect(rangeHasNumber(0, 10, 3));
}
```

# Optionals

Optionals使用语法`?T`，用于存储数据[`null`](https://ziglang.org/documentation/master/#null)，或`T`类型的值 .

```zig
test "optional" {
    var found_index: ?usize = null;
    const data = [_]i32{ 1, 2, 3, 4, 5, 6, 7, 8, 12 };
    for (data, 0..) |v, i| {
        if (v == 10) found_index = i;
    }
    try expect(found_index == null);
}
```

Optionals支持 "orelse "表达式，当选项为[`null`](https://ziglang.org/documentation/master/#null)时，该表达式会发挥作用。这将把可选的东西解包为它的子类型 .

```zig
test "orelse" {
    var a: ?f32 = null;
    var b = a orelse 0;
    try expect(b == 0);
    try expect(@TypeOf(b) == f32);
}
```

`.?`是 "orelse unreachable "的简写。当你知道一个可选的值不可能是空的时候，使用它来解开一个[`null`](https://ziglang.org/documentation/master/#null)值是可以检测到的非法行为 .

```zig
test "orelse unreachable" {
    const a: ?f32 = 5;
    const b = a orelse unreachable;
    const c = a.?;
    try expect(b == c);
    try expect(@TypeOf(c) == f32);
}
```

有效载荷捕获在很多地方都适用于选项，也就是说，在它非空的情况下，我们可以 "捕获 "它的非空值。

这里我们使用`if`可选的有效载荷捕获；a和b在这里是等同的。`if (b) |value|`捕获`b`的值（在`b`不是空的情况下），并将其作为`value`使用。如同在union的例子中，捕获的值是不可变的，但是我们仍然可以使用指针捕获来修改存储在`b`中的值 .

```zig
test "if optional payload capture" {
    const a: ?i32 = 5;
    if (a != null) {
        const value = a.?;
        _ = value;
    }

    var b: ?i32 = 5;
    if (b) |*value| {
        value.* += 1;
    }
    try expect(b.? == 6);
}
```

And with `while`:
```zig
var numbers_left: u32 = 4;
fn eventuallyNullSequence() ?u32 {
    if (numbers_left == 0) return null;
    numbers_left -= 1;
    return numbers_left;
}

test "while null capture" {
    var sum: u32 = 0;
    while (eventuallyNullSequence()) |value| {
        sum += value;
    }
    try expect(sum == 6); // 3 + 2 + 1
}
```

Optional指针和optional slice 类型与非可选的类型相比，不占用任何额外的内存。这是因为在内部它们使用指针的0值作为`null'。

这就是Zig中空指针的工作方式--在取消引用之前，它们必须被解包为非选择型，这可以阻止空指针的取消引用意外发生 .

# Comptime

可以使用[`comptime`](https://ziglang.org/documentation/master/#comptime)关键字在编译时强行执行各块代码。在这个例子中，变量x和y是等价的 .

```zig
test "comptime blocks" {
    var x = comptime fibonacci(10);
    _ = x;

    var y = comptime blk: {
        break :blk fibonacci(10);
    };
    _ = y;
}
```

整数字是 `comptime_int` 的类型。它们的特别之处在于它们没有大小（它们不能在运行时使用！），而且它们有任意的精度。`comptime_int`的值可以与任何可以容纳它们的整数类型相联合。它们也会被胁迫为浮点数。字母是这种类型的 .

```zig
test "comptime_int" {
    const a = 12;
    const b = a + 10;

    const c: u4 = a;
    _ = c;
    const d: f32 = b;
    _ = d;
}
```

也可以使用`comptime_float`，它在内部是`f128`。这些不能被胁迫为整数，即使它们持有一个整数值 .

Zig中的类型是 "type "类型的值。这些在编译时就可以得到。我们之前通过检查[`@TypeOf`](https://ziglang.org/documentation/master/#TypeOf)和与其他类型进行比较来遇到它们，但我们可以做得更多 .

```zig
test "branching on types" {
    const a = 5;
    const b: if (a < 10) f32 else i32 = 5;
    _ = b;
}
```

Zig中的函数参数可以被标记为[`comptime`](https://ziglang.org/documentation/master/#comptime)。这意味着传递给该函数参数的值在编译时必须是已知的。让我们做一个返回类型的函数。注意这个函数是PascalCase，因为它返回一个类型 .

```zig
fn Matrix(
    comptime T: type,
    comptime width: comptime_int,
    comptime height: comptime_int,
) type {
    return [height][width]T;
}

test "returning a type" {
    try expect(Matrix(f32, 4, 4) == [4][4]f32);
}
```

我们可以使用内置的[`@typeInfo`](https://ziglang.org/documentation/master/#typeInfo)来反映类型，它接收一个`类型`并返回一个标记的联合。这个标记的联合类型可以在[`std.buildin.TypeInfo`](https://ziglang.org/documentation/master/std/#std;buildin.TypeInfo)中找到（关于如何利用导入和std的信息将在后面介绍） .

```zig
fn addSmallInts(comptime T: type, a: T, b: T) T {
    return switch (@typeInfo(T)) {
        .ComptimeInt => a + b,
        .Int => |info| if (info.bits <= 16)
            a + b
        else
            @compileError("ints too large"),
        else => @compileError("only ints accepted"),
    };
}

test "typeinfo switch" {
    const x = addSmallInts(u16, 20, 30);
    try expect(@TypeOf(x) == u16);
    try expect(x == 50);
}
```

我们可以使用[`@Type`](https://ziglang.org/documentation/master/#Type)函数从[`@typeInfo`](https://ziglang.org/documentation/master/#typeInfo)创建一个类型。[`@Type`](https://ziglang.org/documentation/master/#Type)对于大多数类型都是实现的，但是对于枚举、联盟、函数和结构来说，显然是没有实现的。

这里匿名结构语法与`.{}`一起使用，因为`T{}`中的`T`可以被推断出来。匿名结构将在后面详细介绍。在这个例子中，如果没有设置`Int`标签，我们会得到一个编译错误 .

```zig
fn GetBiggerInt(comptime T: type) type {
    return @Type(.{
        .Int = .{
            .bits = @typeInfo(T).Int.bits + 1,
            .signedness = @typeInfo(T).Int.signedness,
        },
    });
}

test "@Type" {
    try expect(GetBiggerInt(u8) == u9);
    try expect(GetBiggerInt(i31) == i32);
}
```

返回结构体类型是你在Zig中制作通用数据结构的方式。这里需要使用[`@This`](https://ziglang.org/documentation/master/#This)，它可以获得最内层结构、联盟或枚举的类型。这里还使用了[`std.mem.eql`](https://ziglang.org/documentation/master/std/#std;mem.eql)，用于比较两个slices .

```zig
fn Vec(
    comptime count: comptime_int,
    comptime T: type,
) type {
    return struct {
        data: [count]T,
        const Self = @This();

        fn abs(self: Self) Self {
            var tmp = Self{ .data = undefined };
            for (self.data, 0..) |elem, i| {
                tmp.data[i] = if (elem < 0)
                    -elem
                else
                    elem;
            }
            return tmp;
        }

        fn init(data: [count]T) Self {
            return Self{ .data = data };
        }
    };
}

const eql = @import("std").mem.eql;

test "generic vector" {
    const x = Vec(3, f32).init([_]f32{ 10, -10, 5 });
    const y = x.abs();
    try expect(eql(f32, &y.data, &[_]f32{ 10, 10, 5 }));
}
```

函数参数的类型也可以通过使用`anytype`代替类型来推断。[`@TypeOf`](https://ziglang.org/documentation/master/#TypeOf) 然后可以在参数上使用 .

```zig
fn plusOne(x: anytype) @TypeOf(x) {
    return x + 1;
}

test "inferred function parameter" {
    try expect(plusOne(@as(u32, 1)) == 2);
}
```

Comptime还引入了运算符`++`和`**`，用于连接和重复数组和片断。这些运算符在运行时不工作 .

```zig
test "++" {
    const x: [4]u8 = undefined;
    const y = x[0..];

    const a: [6]u8 = undefined;
    const b = a[0..];

    const new = y ++ b;
    try expect(new.len == 10);
}

test "**" {
    const pattern = [_]u8{ 0xCC, 0xAA };
    const memory = pattern ** 3;
    try expect(eql(u8, &memory, &[_]u8{ 0xCC, 0xAA, 0xCC, 0xAA, 0xCC, 0xAA }));
}
```

# Payload Captures

Payload captures使用语法`|value|`，出现在许多地方，其中一些我们已经看到了。无论它们出现在哪里，它们都是用来 "捕获 "某个东西的值的 .

使用if语句和选项.
```zig
test "optional-if" {
    var maybe_num: ?usize = 10;
    if (maybe_num) |n| {
        try expect(@TypeOf(n) == usize);
        try expect(n == 10);
    } else {
        unreachable;
    }
}
```

用if语句和error unions。这里需要有错误捕获的else .
```zig
test "error union if" {
    var ent_num: error{UnknownEntity}!u32 = 5;
    if (ent_num) |entity| {
        try expect(@TypeOf(entity) == u32);
        try expect(entity == 5);
    } else |err| {
        _ = err catch {};
        unreachable;
    }
}
```

有了 while 循环和选项。这可能有一个else块 .
```zig
test "while optional" {
    var i: ?u32 = 10;
    while (i) |num| : (i.? -= 1) {
        try expect(@TypeOf(num) == u32);
        if (num == 1) {
            i = null;
            break;
        }
    }
    try expect(i == null);
}
```

有了 while 循环和error unions。这里需要有错误捕获的else .

```zig
var numbers_left2: u32 = undefined;

fn eventuallyErrorSequence() !u32 {
    return if (numbers_left2 == 0) error.ReachedZero else blk: {
        numbers_left2 -= 1;
        break :blk numbers_left2;
    };
}

test "while error union capture" {
    var sum: u32 = 0;
    numbers_left2 = 3;
    while (eventuallyErrorSequence()) |value| {
        sum += value;
    } else |err| {
        try expect(err == error.ReachedZero);
    }
}
```

For loops.
```zig
test "for capture" {
    const x = [_]i8{ 1, 5, 120, -5 };
    for (x) |v| try expect(@TypeOf(v) == i8);
}
```

Switch cases on tagged unions.
```zig
const Info = union(enum) {
    a: u32,
    b: []const u8,
    c,
    d: u32,
};

test "switch capture" {
    var b = Info{ .a = 10 };
    const x = switch (b) {
        .b => |str| blk: {
            try expect(@TypeOf(str) == []const u8);
            break :blk 1;
        },
        .c => 2,
        //if these are of the same type, they
        //may be inside the same capture group
        .a, .d => |num| blk: {
            try expect(@TypeOf(num) == u32);
            break :blk num * 2;
        },
    };
    try expect(x == 20);
}
```

正如我们在上面的Union和Optional部分看到的，用`|val|`语法捕获的值是不可变的（类似于函数参数），但我们可以使用指针捕获来修改原始值。这就把值捕获为指针，这些指针本身仍然是不可变的，但由于值现在是一个指针，我们可以通过取消引用来修改原始值： 

```zig
test "for with pointer capture" {
    var data = [_]u8{ 1, 2, 3 };
    for (&data) |*byte| byte.* += 1;
    try expect(eql(u8, &data, &[_]u8{ 2, 3, 4 }));
}
```

# Inline Loops 内联循环

`inline`循环是不滚动的，它允许一些只在编译时发生的事情。这里我们使用一个[`for`](https://ziglang.org/documentation/master/#inline-for)，但[`while`](https://ziglang.org/documentation/master/#inline-while)也有类似的作用 .
```zig
test "inline for" {
    const types = [_]type{ i32, f32, u8, bool };
    var sum: usize = 0;
    inline for (types) |T| sum += @sizeOf(T);
    try expect(sum == 10);
}
```

出于性能的考虑，使用这些是不可取的，除非你已经测试出显式展开的速度更快；编译器在这里往往比你做出更好的决定 .

# Opaque

[`opaque`](https://ziglang.org/documentation/master/#opaque)Zig中的类型有一个未知的（尽管不是零）大小和排列。正因为如此，这些数据类型不能被直接存储。这些是用来维护类型安全的，有指向我们没有信息的类型的指针 .

<!--fail_test-->
```zig
const Window = opaque {};
const Button = opaque {};

extern fn show_window(*Window) callconv(.C) void;

test "opaque" {
    var main_window: *Window = undefined;
    show_window(main_window);

    var ok_button: *Button = undefined;
    show_window(ok_button);
}
```
```
./test-c1.zig:653:17: error: expected type '*Window', found '*Button'
    show_window(ok_button);
                ^
./test-c1.zig:653:17: note: pointer type child 'Button' cannot cast into pointer type child 'Window'
    show_window(ok_button);
                ^
```

Opaque类型在其定义中可以有声明（与结构、枚举和联盟相同） .

<!--no_test-->
```zig
const Window = opaque {
    fn show(self: *Window) void {
        show_window(self);
    }
};

extern fn show_window(*Window) callconv(.C) void;

test "opaque with declarations" {
    var main_window: *Window = undefined;
    main_window.show();
}
```

opaque的典型用例是在与没有暴露完整类型信息的C代码互操作时，维护类型安全

# Anonymous Structs 匿名结构

结构类型可以从结构字中省略。这些字面意义可以联合到其他结构类型。

```zig
test "anonymous struct literal" {
    const Point = struct { x: i32, y: i32 };

    var pt: Point = .{
        .x = 13,
        .y = 67,
    };
    try expect(pt.x == 13);
    try expect(pt.y == 67);
}
```

匿名结构可以是完全匿名的，即没有被胁迫到其他结构类型 .

```zig
test "fully anonymous struct" {
    try dump(.{
        .int = @as(u32, 1234),
        .float = @as(f64, 12.34),
        .b = true,
        .s = "hi",
    });
}

fn dump(args: anytype) !void {
    try expect(args.int == 1234);
    try expect(args.float == 12.34);
    try expect(args.b);
    try expect(args.s[0] == 'h');
    try expect(args.s[1] == 'i');
}
```
<!-- TODO: mention tuple slicing when it's implemented -->

没有字段名的匿名结构可以被创建，并被称为 __tuples__。这些结构有很多和数组一样的属性；图元可以被迭代，可以被索引，可以使用 `++` 和 `**` 操作符，并且有一个 len 域。在内部，这些有编号的字段名从`"0"` 开始，可以用特殊的语法`@"0"`来访问，它作为语法的转义--`@""`里面的东西总是被认为是标识符。

在这里必须使用一个`内联`循环来迭代元组，因为每个元组字段的类型可能不同 .

```zig
test "tuple" {
    const values = .{
        @as(u32, 1234),
        @as(f64, 12.34),
        true,
        "hi",
    } ++ .{false} ** 2;
    try expect(values[0] == 1234);
    try expect(values[4] == false);
    inline for (values, 0..) |v, i| {
        if (i != 2) continue;
        try expect(v);
    }
    try expect(values.len == 6);
    try expect(values.@"3"[0] == 'h');
}
```

# Sentinel Termination 哨兵终结者

数组、切片和许多指针可以由其子类型的值来终止。这被称为哨兵终止。语法是`[N:t]T`, `[:t]T`, 和`[*:t]T`, 其中`t`是子类型`T`的一个值。

一个哨兵终止数组的例子。内置的[`@bitCast`](https://ziglang.org/documentation/master/#bitCast)被用来进行不安全的比特类型转换。这表明数组的最后一个元素后面是一个0字节 .

```zig
test "sentinel termination" {
    const terminated = [3:0]u8{ 3, 2, 1 };
    try expect(terminated.len == 3);
    try expect(@ptrCast(*const [4]u8, &terminated)[3] == 0);
}
```

字符串的类型是`*const [N:0]u8`，其中N是字符串的长度。这允许字符串字面意义上的内容与前哨终止的片断和前哨终止的许多指针相衔接。注意：字符串字词是UTF-8编码的 .

```zig
test "string literal" {
    try expect(@TypeOf("hello") == *const [5:0]u8);
}
```

`[*:0]u8` and `[*:0]const u8` 完美地模拟C的字符串.

```zig
test "C string" {
    const c_string: [*:0]const u8 = "hello";
    var array: [5]u8 = undefined;

    var i: usize = 0;
    while (c_string[i] != 0) : (i += 1) {
        array[i] = c_string[i];
    }
}
```

Sentinel terminated types coerce to their non-sentinel-terminated counterparts.
哨兵终止类型强制转换为非哨兵终止类型。

```zig
test "coercion" {
    var a: [*:0]u8 = undefined;
    const b: [*]u8 = a;
    _ = b;

    var c: [5:0]u8 = undefined;
    const d: [5]u8 = c;
    _ = d;

    var e: [:10]f32 = undefined;
    const f = e;
    _ = f;
}
```

提供了哨兵终止的切片，可以用来创建一个哨兵终止的切片，语法为`x[n..m:t]`，其中`t`是终止值。这样做是来自程序员的断言，即内存在它应该被终止的地方被终止--弄错了就是可检测的非法行为 .

```zig
test "sentinel terminated slicing" {
    var x = [_:0]u8{255} ** 3;
    const y = x[0..3 :0];
    _ = y;
}
```

# Vectors

Zig为SIMD提供了向量类型。这些类型不能与数学意义上的向量相混淆，也不能与C++的std::vector那样的向量相混淆（关于这一点，请看第2章的 "Arraylist"）。向量可以使用我们之前使用的[`@Type`](https://ziglang.org/documentation/master/#Type)内置来创建，[`std.meta.Vector`](https://ziglang.org/documentation/master/std/#std;meta.Vector)提供了一种速记方法。

向量只能有布尔、整数、浮点数和指针等子类型。

具有相同子类型和长度的向量之间可以进行操作。这些操作是对向量中的每个值进行的。[`std.meta.eql`](https://ziglang.org/documentation/master/std/#std;meta.eql)在这里被用来检查两个向量之间是否相等（对其他类型如结构也很有用）.

```zig
const meta = @import("std").meta;
const Vector = meta.Vector;

test "vector add" {
    const x: Vector(4, f32) = .{ 1, -10, 20, -1 };
    const y: Vector(4, f32) = .{ 2, 10, 0, 1 };
    const z = x + y;
    try expect(meta.eql(z, Vector(4, f32){ 3, 0, 20, 0 }));
}
```

向量是可索引的.
```zig
test "vector indexing" {
    const x: Vector(4, u8) = .{ 255, 0, 255, 0 };
    try expect(x[0] == 255);
}
```

内置函数[`@splat`](https://ziglang.org/documentation/master/#splat)可以用来构造一个所有数值都相同的向量。在这里，我们用它将一个向量与一个标量相乘 .

```zig
test "vector * scalar" {
    const x: Vector(3, f32) = .{ 12.5, 37.5, 2.5 };
    const y = x * @splat(3, @as(f32, 2));
    try expect(meta.eql(y, Vector(3, f32){ 25, 75, 5 }));
}
```

向量不像数组那样有一个`len`字段，但是仍然可以循环使用。这里，[`std.mem.len`](https://ziglang.org/documentation/master/std/#std;mem.len)被用作`@typeInfo(@TypeOf(x)).Vector.len`的捷径 .

```zig
const len = @import("std").mem.len;

test "vector looping" {
    const x = Vector(4, u8){ 255, 0, 255, 0 };
    var sum = blk: {
        var tmp: u10 = 0;
        var i: u8 = 0;
        while (i < 4) : (i += 1) tmp += x[i];
        break :blk tmp;
    };
    try expect(sum == 510);
}
```

向量凝聚在各自的数组中 .

```zig
const arr: [4]f32 = @Vector(4, f32){ 1, 2, 3, 4 };
```

值得注意的是，如果你没有做出正确的决定，使用显式向量可能会导致软件速度变慢--编译器的自动向量化在目前来说是相当聪明的 .

# Imports

内置函数 [`@import`](https://ziglang.org/documentation/master/#import) 接收一个文件，并根据该文件给出一个结构类型。所有标记为 `pub`（代表public）的声明最后都会出现在这个结构类型中，可以随时使用。

`@import("std")`是编译器中的一个特殊情况，它让你访问标准库。其他的[`@import`](https://ziglang.org/documentation/master/#import)将接受一个文件路径，或一个包的名称（关于包的更多内容在后面的章节中）。

我们将在后面的章节中探索更多的标准库。

# 第一章结束
在下一章中，我们将介绍标准模式，包括标准库中许多有用的领域。

欢迎大家提供反馈和PR。

