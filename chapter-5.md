---
title: "Chapter 5 - 异步"
weight: 6
date: 2023-04-28 18:00:00
description: "Chapter 5 - 了解ziglang的异步是如何工作的"
---

警告：当前版本的编译器还不支持异步


# Async

要对Zig的异步进行有效的理解，需要熟悉调用栈的概念。如果你以前没有听说过这个，[请查看wikipedia页面](https://en.wikipedia.org/wiki/Call_stack)。

<！-- TODO：实际上解释一下调用栈？-->

一个传统的函数调用包括三个方面：
1. 用参数启动被调用的函数，推入函数的堆栈框架
2. 将控制权转移到该函数
3. 在函数完成后，将控制权交还给调用者，取回函数的返回值并弹出函数的堆栈框架

有了Zig的异步函数，我们可以做得更多，控制权的转移是一个持续的双向对话（也就是说，我们可以多次将控制权交给函数并收回）。正因为如此，在异步上下文中调用一个函数时必须要有特别的考虑；我们不能再像正常情况下那样推送和弹出堆栈框架（因为堆栈是不稳定的，"在 "当前堆栈框架之上的东西可能被覆盖），而是明确地存储异步函数的框架。虽然大多数人不会使用它的全部功能，但这种Async的风格对于创建更强大的结构（如事件循环）是很有用的。

Zig的async风格可以被描述为可暂停的无堆栈的coroutines。Zig的async与操作系统中的线程有很大的不同，后者有一个堆栈，只能由内核来暂停。此外，Zig的async是为你提供控制流结构和代码生成；async并不意味着并行或使用线程。

# 暂停/恢复 suspend/resume

在上一节中，我们谈到了异步函数如何将控制权交还给调用者，以及异步函数随后如何将控制权收回。这个功能是由关键字[`suspend`, and `resume`]（https://ziglang.org/documentation/master/#Suspend-and-Resume）提供的。当一个函数暂停时，控制流返回到最后恢复的地方；当一个函数通过`async`调用时，这是一个隐式恢复。

这些例子中的注释指出了执行的顺序。这里有几件事需要注意：
* `async`关键字是用来在异步上下文中调用函数的。
* `async func()`返回函数的框架。
* 我们必须存储这个框架。
* `resume`关键字用于框架上，而`suspend`用于被调用的函数上。

这个例子有一个暂停，但没有匹配的恢复。
```zig
const expect = @import("std").testing.expect;

var foo: i32 = 1;

test "suspend with no resume" {
    var frame = async func(); //1
    _ = frame;
    try expect(foo == 2);     //4
}

fn func() void {
    foo += 1;                 //2
    suspend {}                //3
    foo += 1;                 //never reached!
}
```

在良好的代码中，每个暂停都与一个恢复相匹配.

```zig
var bar: i32 = 1;

test "suspend with resume" {
    var frame = async func2();  //1
    resume frame;               //4
    try expect(bar == 3);       //6
}

fn func2() void {
    bar += 1;                   //2
    suspend {}                  //3
    bar += 1;                   //5
}
```

# Async / Await

类似于良好的代码在每个恢复过程中都有一个暂停，每个有返回值的`async`函数调用都必须有一个`await`来匹配。`await`在异步框架上产生的值与函数的返回值相对应。

你可能注意到，这里的`func3`是一个普通的函数（即它没有暂停点--它不是一个异步函数）。尽管如此，当从一个异步调用中调用时，`func3`可以作为一个异步函数工作；`func3`的调用约定不必改为异步 - `func3`可以是任何调用约定 .

```zig
fn func3() u32 {
    return 5;
}

test "async / await" {
    var frame = async func3();
    try expect(await frame == 5);
}
```

在一个可能暂停的函数的异步框架上使用`await`，只有在异步函数中才可能。因此，在一个异步函数的框架上使用`await`的函数也被视为异步函数。如果你能确定潜在的暂停不会发生，`nosuspend await`将阻止这种情况的发生.

# Nosuspend

当调用一个被确定为异步的函数（即它可能暂停）而没有`async`调用时，调用它的函数也被视为异步。当一个具体的（非async）调用约定的函数被确定为有暂停点时，这是一个编译错误，因为async需要它自己的调用约定。这意味着，例如，main不能是异步的。

<!--no_test-->
```zig
pub fn main() !void {
    suspend {}
}
```
(compiled from windows)
```
C:\zig\lib\zig\std\start.zig:165:1: error: function with calling convention 'Stdcall' cannot be async
fn WinStartup() callconv(.Stdcall) noreturn {
^
C:\zig\lib\zig\std\start.zig:173:65: note: async function call here
    std.os.windows.kernel32.ExitProcess(initEventLoopAndCallMain());
                                                                ^
C:\zig\lib\zig\std\start.zig:276:12: note: async function call here
    return @call(.{ .modifier = .always_inline }, callMain, .{});
           ^
C:\zig\lib\zig\std\start.zig:334:37: note: async function call here
            const result = root.main() catch |err| {
                                    ^
.\main.zig:12:5: note: suspends here
    suspend {}
    ^
```

如果你想在不使用`async`调用的情况下调用一个异步函数，并且该函数的调用者也不是异步，`nosuspend`关键字就会派上用场。这允许异步函数的调用者不是异步的，通过断言潜在的暂停不会发生 .

<!--no_test-->
```zig
const std = @import("std");

fn doTicksDuration(ticker: *u32) i64 {
    const start = std.time.milliTimestamp();

    while (ticker.* > 0) {
        suspend {}
        ticker.* -= 1;
    }

    return std.time.milliTimestamp() - start;
}

pub fn main() !void {
    var ticker: u32 = 0;
    const duration = nosuspend doTicksDuration(&ticker);
}
```

在上面的代码中，如果我们把`ticker`的值改为0以上，这就是可检测到的非法行为。如果我们运行这段代码，在安全构建模式下会出现这样的错误。与Zig中的其他非法行为类似，在不安全模式下发生这些行为将导致未定义行为 .

```
async function called in nosuspend scope suspended
.\main.zig:16:47: 0x7ff661dd3414 in main (main.obj)
    const duration = nosuspend doTicksDuration(&ticker);
                                              ^
C:\zig\lib\zig\std\start.zig:173:65: 0x7ff661dd18ce in std.start.WinStartup (main.obj)
    std.os.windows.kernel32.ExitProcess(initEventLoopAndCallMain());
                                                                ^
```

# Async Frames, Suspend Blocks(异步框架，悬空区块)

`@Frame(function)`返回该函数的框架类型。这适用于异步函数，以及没有特定调用约定的函数。

```zig
fn add(a: i32, b: i32) i64 {
    return a + b;
}

test "@Frame" {
    var frame: @Frame(add) = async add(1, 2);
    try expect(await frame == 3);
}
```

[`@frame()`](https://ziglang.org/documentation/master/#frame)返回一个指向当前函数框架的指针。与`suspend`点类似，如果在一个函数中发现这个调用，那么它被推断为是同步的。所有指向框架的指针都会被默认为特殊类型`anyframe'，你可以对其使用`resume'。

这使得我们可以，例如，写一个函数来恢复自己。

```zig
fn double(value: u8) u9 {
    suspend {
        resume @frame();
    }
    return value * 2;
}

test "@frame 1" {
    var f = async double(1);
    try expect(nosuspend await f == 2);
}
```

或者，更有趣的是，我们可以用它来告诉其他函数来恢复我们。在这里，我们要引入**suspend blocks(suspend blocks)**。在进入suspend块时，异步函数已经被认为是暂停了（也就是说，它可以被恢复）。这意味着我们可以让我们的函数被最后的恢复者以外的东西恢复 .

```zig
const std = @import("std");

fn callLater(comptime laterFn: fn () void, ms: u64) void {
    suspend {
        wakeupLater(@frame(), ms);
    }
    laterFn();
}

fn wakeupLater(frame: anyframe, ms: u64) void {
    std.time.sleep(ms * std.time.ns_per_ms);
    resume frame;
}

fn alarm() void {
    std.debug.print("Time's Up!\n", .{});
}

test "@frame 2" {
    nosuspend callLater(alarm, 1000);
}
```

使用`anyframe`数据类型可以被认为是一种类型清除，因为我们不再确定函数或函数框架的具体类型。这很有用，因为它仍然允许我们恢复框架--在很多代码中，我们不关心细节，只是想恢复它。这给了我们一个单一的具体类型，我们可以用它来实现我们的异步逻辑。

`anyframe`的自然缺点是我们失去了类型信息，我们不再知道函数的返回类型是什么。这意味着我们不能等待一个`anyframe`。Zig对此的解决方案是`anyframe->T`类型，其中`T`是框架的返回类型。

```zig
fn zero(comptime x: anytype) x {
    return 0;
}

fn awaiter(x: anyframe->f32) f32 {
    return nosuspend await x;
}

test "anyframe->T" {
    var frame = async zero(f32);
    try expect(awaiter(&frame) == 0);
}
```

# 基本事件循环的实现

事件循环是一种设计模式，其中事件被分派和/或被等待。这将意味着某种服务或运行时在满足条件时恢复暂停的异步框架。这是Zig的async最强大和最有用的用例。

这里我们将实现一个基本的事件循环。这个将允许我们提交任务，在一定的时间内执行。我们将用它来提交成对的任务，这些任务将打印自程序开始以来的时间。下面是一个输出的例子 .

```
[task-pair b] it is now 499 ms since start!
[task-pair a] it is now 1000 ms since start!
[task-pair b] it is now 1819 ms since start!
[task-pair a] it is now 2201 ms since start!
```

以下是具体实现.

<!--no_test-->
```zig
const std = @import("std");

// 获得单调的时间，而不是挂钟时间
var timer: ?std.time.Timer = null;
fn nanotime() u64 {
    if (timer == null) {
        timer = std.time.Timer.start() catch unreachable;
    }
    return timer.?.read();
}

// 持有该帧，以及该帧应该被恢复的纳米时间
const Delay = struct {
    frame: anyframe,
    expires: u64,
};

// 暂停调用者，稍后由事件循环恢复。
fn waitForTime(time_ms: u64) void {
    suspend timer_queue.add(Delay{
        .frame = @frame(),
        .expires = nanotime() + (time_ms * std.time.ns_per_ms),
    }) catch unreachable;
}

fn waitUntilAndPrint(
    time1: u64,
    time2: u64,
    name: []const u8,
) void {
    const start = nanotime();

    // 暂停自我，当time1过后被唤醒
    waitForTime(time1);
    std.debug.print(
        "[{s}] it is now {} ms since start!\n",
        .{ name, (nanotime() - start) / std.time.ns_per_ms },
    );

    // 暂停自我，待time2过后被唤醒 
    waitForTime(time2);
    std.debug.print(
        "[{s}] it is now {} ms since start!\n",
        .{ name, (nanotime() - start) / std.time.ns_per_ms },
    );
}

fn asyncMain() void {
    // 存储我们任务的异步框架
    var tasks = [_]@Frame(waitUntilAndPrint){
        async waitUntilAndPrint(1000, 1200, "task-pair a"),
        async waitUntilAndPrint(500, 1300, "task-pair b"),
    };

    // 使用|*t|，因为|t|是一个*const @Frame(...)不能被等待的
    for (tasks) |*t| await t;
}

// 任务的优先队列
// lower .expires => higher priority => to be executed before
var timer_queue: std.PriorityQueue(Delay, void, cmp) = undefined;
fn cmp(context: void, a: Delay, b: Delay) std.math.Order {
    _ = context;
    return std.math.order(a.expires, b.expires);
}

pub fn main() !void {
    timer_queue = std.PriorityQueue(Delay, void, cmp).init(
        std.heap.page_allocator, undefined
    );
    defer timer_queue.deinit();

    var main_task = async asyncMain();

    // 事件循环的主体
    // 弹出下一个要执行的任务
    while (timer_queue.removeOrNull()) |delay| {
        // 等待，直到执行下一个任务的时间到了  
        const now = nanotime();
        if (now < delay.expires) {
            std.time.sleep(delay.expires - now);
        }
        // 执行下一个任务
        resume delay.frame;
    }

    nosuspend await main_task;
}
```

# 第五章结束

本章还不完整，将来应该包含[`std.event.Loop`](https://ziglang.org/documentation/master/std/#std;event.Loop)的用法，以及事件化IO。

欢迎反馈和PR。

