---
title: "Chapter 4 - 使用C工作"
weight: 5
date: 2023-04-28 18:00:00
description: "Chapter 4 - 了解Zig编程语言如何利用C代码。本教程包括C数据类型、FFI、用C构建、translate-c等内容！"
---

Zig的设计从一开始就把C语言的互操作作为一个首要功能。在这一节中，我们将介绍如何工作.

# ABI

ABI *(application binary interface)*是一种标准，与以下方面有关：

- 类型的内存布局（即一个类型的大小、对齐方式、偏移量和它的字段的布局）
- 符号的内存命名（例如，名称混用）
- 函数的调用约定（即函数调用在二进制水平上如何工作）

通过定义这些规则并且不破坏它们，ABI被认为是稳定的，这可以被用来，例如，可靠地连接多个库、可执行文件或单独编译的对象（可能在不同的机器上，或使用不同的编译器）。这允许FFI *（外部函数接口）*的发生，我们可以在编程语言之间共享代码。

Zig原生支持用于 "外部 "事物的C ABI；使用哪种C ABI取决于你所编译的目标（如CPU架构、操作系统）。这允许与不是用Zig编写的代码进行近乎无缝的互操作；使用C ABI是编程语言中的标准。

Zig内部不使用ABI，这意味着在需要可复制和定义的二进制级别行为时，代码应明确地符合C ABI。

# C原始类型

Zig提供了特殊的`c_`前缀类型，以符合C ABI的要求。这些类型没有固定的大小，而是根据所使用的ABI来改变大小.

| Type         | C Equivalent      | Minimum Size (bits) |
|--------------|-------------------|---------------------|
| c_short      | short             | 16                  |
| c_ushort     | unsigned short    | 16                  |
| c_int        | int               | 16                  |
| c_uint       | unsigned int      | 16                  |
| c_long       | long              | 32                  |
| c_ulong      | unsigned long     | 32                  |
| c_longlong   | long long         | 64                  |
| c_ulonglong  | unsigned longlong | 64                  |
| c_longdouble | long double       | N/A                 |
| c_void       | void              | N/A                 |

注意：C的void（和Zig的`c_void`）有一个未知的非零大小。Zig的`void`是一个真正的零尺寸类型.


# 调用约定

调用约定描述了函数的调用方式。这包括如何向函数提供参数（即它们的位置--在寄存器中还是在堆栈中，以及如何），以及如何接收返回值。

在Zig中，"callconv "属性可以被赋予给一个函数。可用的调用约定可以在[std.biltin.CallingConvention](https://ziglang.org/documentation/master/std/#A;std:biltin.CallingConvention)中找到。这里我们使用了cdecl的调用约定 .

```zig
fn add(a: u32, b: u32) callconv(.C) u32 {
    return a + b;
}
```

当你从C语言中调用Zig时，用C语言的调用惯例来标记你的函数是至关重要的.


# 外部结构

Zig中的普通结构没有定义布局；当你希望结构的布局与C ABI的布局相匹配时，就需要`extern`结构。

我们来创建一个extern结构。这个测试应该在`x86_64`和`gnu`ABI下运行，可以用`-target x86_64-native-gnu`完成.

```zig
const expect = @import("std").testing.expect;

const Data = extern struct { a: i32, b: u8, c: f32, d: bool, e: bool };

test "hmm" {
    const x = Data{
        .a = 10005,
        .b = 42,
        .c = -10.5,
        .d = false,
        .e = true,
    };
    const z = @ptrCast([*]const u8, &x);

    try expect(@ptrCast(*const i32, z).* == 10005);
    try expect(@ptrCast(*const u8, z + 4).* == 42);
    try expect(@ptrCast(*const f32, z + 8).* == -10.5);
    try expect(@ptrCast(*const bool, z + 12).* == false);
    try expect(@ptrCast(*const bool, z + 13).* == true);
}
```

这就是我们的`x`值里面的内存的样子.

| Field | a  | a  | a  | a  | b  |    |    |    | c  | c  | c  | c  | d  | e  |    |    |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Bytes | 15 | 27 | 00 | 00 | 2A | 00 | 00 | 00 | 00 | 00 | 28 | C1 | 00 | 01 | 00 | 00 |

请注意中间和末尾的空隙--这被称为 "填充"。这个填充物中的数据是未定义的内存，不会一直为零.

由于我们的 "x "值是一个外部结构，我们可以安全地将它传递给一个期望有 "数据 "的C函数，前提是该C函数也是用相同的 "gnu "ABI和CPU架构编译的.


# 对齐

由于电路的原因，CPU在内存中以一定的倍数访问原始值。例如，这可能意味着一个`f32'值的地址必须是4的倍数，这意味着`f32'的对齐方式是4。这种所谓的原始数据类型的 "自然对齐 "取决于CPU架构。所有的对齐方式都是2的幂。

一个较大的对齐方式的数据也有每个较小的对齐方式；例如，一个对齐方式为16的值也有8、4、2和1的对齐方式。

我们可以通过使用`align(x)`属性来制作特殊对齐的数据。这里我们制作的是具有更大对齐度的数据 .

```zig
const a1: u8 align(8) = 100;
const a2 align(8) = @as(u8, 100);
```
而制作对齐度较小的数据。注意：创建较小对齐方式的数据并不是特别有用.
```zig
const b1: u64 align(1) = 100;
const b2 align(1) = @as(u64, 100);
```

和`const`一样，`align`也是指针的一个属性.
```zig
test "aligned pointers" {
    const a: u32 align(8) = 5;
    try expect(@TypeOf(&a) == *align(8) const u32);
}
```

让我们利用一个期望对齐指针的函数.

```zig
fn total(a: *align(64) const [64]u8) u32 {
    var sum: u32 = 0;
    for (a) |elem| sum += elem;
    return sum;
}

test "passing aligned data" {
    const x align(64) = [_]u8{10} ** 64;
    try expect(total(&x) == 640);
}
```

# 打包的结构

默认情况下，Zig中的所有结构字段都是自然对齐的，即[`@alignOf(FieldType)`](https://ziglang.org/documentation/master/#alignOf)（ABI大小），但没有定义布局。有时你可能希望结构字段的布局与你的C ABI不一致。`packed`结构允许你对结构字段进行极其精确的控制，允许你逐位放置字段。

在打包的结构中，Zig的整数在空间中占用其位宽（即`u12`的[`@bitSizeOf`](https://ziglang.org/documentation/master/#bitSizeOf)为12，意味着它在打包的结构中会占用12位）。傻瓜也占用1位，这意味着你可以很容易地实现位标志.

```zig
const MovementState = packed struct {
    running: bool,
    crouching: bool,
    jumping: bool,
    in_air: bool,
};

test "packed struct size" {
    try expect(@sizeOf(MovementState) == 1);
    try expect(@bitSizeOf(MovementState) == 4);
    const state = MovementState{
        .running = true,
        .crouching = true,
        .jumping = true,
        .in_air = true,
    };
    _ = state;
}
```

目前Zig的打包结构有一些长期存在的编译器错误，目前在很多使用情况下都不能使用.

# 位对齐的指针

与对齐的指针类似，位对齐的指针在其类型中具有额外的信息，告知如何访问数据。当数据不是字节对齐的时候，这些信息是必要的。位对齐信息通常需要用于寻址打包结构中的字段.

```zig
test "bit aligned pointers" {
    var x = MovementState{
        .running = false,
        .crouching = false,
        .jumping = false,
        .in_air = false,
    };

    const running = &x.running;
    running.* = true;

    const crouching = &x.crouching;
    crouching.* = true;

    try expect(@TypeOf(running) == *align(1:0:1) bool);
    try expect(@TypeOf(crouching) == *align(1:1:1) bool);

    try expect(@import("std").meta.eql(x, .{
        .running = true,
        .crouching = true,
        .jumping = false,
        .in_air = false,
    }));
}
```

# C 指针

到现在为止，我们已经使用了以下几种指针：

- 单项指针 - `*T`
- 多项指针 - `[*]T`
- 切片 - `[]T`

与上述指针不同，C语言指针不能处理特别对齐的数据，可以指向地址`0`。C指针在整数之间来回递增，也可以递增到单项和多项指针。当一个值为`0`的C指针被胁迫到一个非选择的指针，这是可以检测到的非法行为。

在自动翻译的C代码之外，使用`[*c]`几乎总是一个坏主意，而且几乎不应该被使用 .

# Translate-C

Zig提供了`zig translate-c`命令，用于自动翻译C源代码。

创建`main.c`文件，内容如下.
```c
#include <stddef.h>

void int_sort(int* array, size_t count) {
    for (int i = 0; i < count - 1; i++) {
        for (int j = 0; j < count - i - 1; j++) {
            if (array[j] > array[j+1]) {
                int temp = array[j];
                array[j] = array[j+1];
                array[j+1] = temp;
            }
        }
    }
}
```

运行命令`zig translate-c main.c`以获得相当于Zig代码的输出到你的控制台（stdout）。你可以用`zig translate-c main.c > int_sort.zig`将其导入一个文件中（对windows用户的警告：在powershell中的管道将产生一个编码不正确的文件--用你的编辑器来纠正）。

在另一个文件中，你可以使用`@import("int_sort.zig")`来使用这个函数。

当前产生的代码可能是不必要的冗长，尽管translate-c成功地将大多数C代码翻译成Zig。你可能希望在将其编辑成更习惯的代码之前使用 translate-c 来生成 Zig 代码；在一个代码库中从 C 逐步转移到 Zig 是一个支持的使用案例 .



# cImport

Zig[`@cImport`](https://ziglang.org/documentation/master/#cImport)内建程序很特别，因为它接收一个表达式，这个表达式只能接收[`@cInclude`](https://ziglang.org/documentation/master/#cInclude)、[`@cDefine`](https://ziglang.org/documentation/master/#cDefine)和[`@cUndef`](https://ziglang.org/documentation/master/#cUndef)。它的工作原理与translate-c类似，将C代码翻译成Zig。

[`@cInclude`](https://ziglang.org/documentation/master/#cInclude)接收一个路径字符串，可以将该路径添加到包含列表中。

[`@cDefine`](https://ziglang.org/documentation/master/#cDefine)和[`@cUndef`](https://ziglang.org/documentation/master/#cUndef)为导入的东西进行定义和取消定义。

这三个函数的工作方式与你期望它们在C代码中的工作方式完全一致。

与 [`@import`](https://ziglang.org/documentation/master/#import) 类似，它返回一个带有声明的结构类型。通常建议在一个应用程序中只使用一个[`@cImport`](https://ziglang.org/documentation/master/#cImport)的实例，以避免符号冲突；在一个cImport中生成的类型将不等同于在另一个中生成的类型。

cImport 仅在链接 libc 时可用。


# 链接 libc

链接 libc 可以通过命令行中的 `-lc` 来完成，或者通过 `build.zig` 使用 `exe.linkLibC();` 来完成。使用的libc是编译目标的libc；Zig为许多目标提供libc .

# Zig cc, Zig c++

Zig的可执行文件中嵌入了Clang，以及为其他操作系统和架构进行交叉编译所需的库和头文件。

这意味着，`zig cc`和`zig c++`不仅可以编译C和C++代码（与Clang兼容的参数），而且还可以在尊重Zig的目标三参数的情况下进行编译；你所安装的单个Zig二进制文件有能力为多个不同的目标进行编译，而无需安装多个版本的编译器或任何附加组件。使用`zig cc`和`zig c++`还可以利用Zig的缓存系统来加速你的工作流程.

使用Zig，人们可以很容易地为使用C和/或C++编译器的语言构建一个交叉编译工具链。


一些野外的例子：

- [使用zig cc将LuaJIT从x86_64-linux交叉编译到aarch64-linux](https://andrewkelley.me/post/zig-cc-powerful-drop-in-replacement-gcc-clang.html)

- [使用zig cc和zig c++与cgo结合，将hugo从arch64-macos交叉编译到x86_64-linux，并进行完全静态链接](https://twitter.com/croloris/status/1349861344330330114)


# 第四章结束

这一章是不完整的。在未来，它将包含诸如以下内容：
- 从Zig调用C代码，反之亦然
- 使用`zig build`混合使用C和Zig代码

欢迎大家提供反馈意见和PR。
