---
title: "Chapter 3 - 构建系统"
weight: 4
date: 2021-02-12 12:49:00
description: "Chapter 3 - Ziglang构建系统详解."
---

# 构建模式  

Zig提供了四种构建模式，其中调试模式是默认的，因为它能产生最短的编译时间。.

|               | 运行时安全 | 优化 |
|---------------|----------------|---------------|
| Debug         | Yes            | No            |
| ReleaseSafe  | Yes            | Yes, Speed    |
| ReleaseSmall | No             | Yes, Size     |
| ReleaseFast  | No             | Yes, Speed    |

这些可以在`zig run`和`zig test`中启用，参数为`-O ReleaseSafe`、`-O ReleaseSmall`和`-O ReleaseFast`.

建议用户在开发软件时启用运行时安全，尽管它在速度上有小的缺点. 

# 输出一个可执行文件

命令`zig build-exe`、`zig build-lib`和`zig build-obj`可以分别用来输出可执行文件、库和对象。这些命令接收一个源文件和参数。

一些常见的参数：
- `-fsingle-threaded`，它断言二进制文件是单线程的。这将把线程安全措施（如互斥）变成无操作。
- `-fstrip`，从二进制文件中删除调试信息。
- `--dynamic`，与`zig build-lib`一起使用，输出一个动态/共享库。

让我们创建一个小小的hello world。保存为`tiny-hello.zig`，然后运行`zig build-exe .\tiny-hello.zig -O ReleaseSmall -fstrip -fsingle-readed`。目前对于`x86_64-windows`，这将产生一个2.5KiB的可执行文件。

<!--no_test-->
```zig
const std = @import("std");

pub fn main() void {
    std.io.getStdOut().writeAll(
        "Hello World!",
    ) catch unreachable;
}
```

# 交叉编译

默认情况下，Zig将为你的CPU和操作系统的组合进行编译。这可以通过`-target`来重写。让我们把我们的小hello世界编译到64位的arm linux平台上.

`zig build-exe .\tiny-hello.zig -O ReleaseSmall -fstrip -fsingle-threaded -target aarch64-linux`

[QEMU](https://www.qemu.org/) 或类似的东西可以用来方便地测试为国外平台制作的可执行文件.

一些可以交叉编译的CPU架构:
- `x86_64`
- `arm`
- `aarch64`
- `i386`
- `riscv64`
- `wasm32`

一些可以交叉编译的操作系统:
- `linux`
- `macos`
- `windows`
- `freebsd`
- `netbsd`
- `dragonfly`
- `UEFI`

许多其他的目标可以用来编译，但目前还没有经过良好的测试。更多信息请参见[Zig的支持表](https://ziglang.org/learn/overview/#wide-range-of-targets-supported)；经过良好测试的目标列表正在慢慢扩大.

由于Zig默认为您的特定CPU进行编译，这些二进制文件可能无法在其他CPU架构略有不同的计算机上运行。为了提高兼容性，指定一个特定的基线CPU型号可能是有用的。注意：选择一个较早的CPU架构会带来更大的兼容性，但也意味着你会错过较新的CPU指令；这里有一个效率/速度与兼容性的权衡.

让我们为sandybridge CPU（Intel x86_64，2011年左右）编译一个二进制文件，这样我们就可以合理地确定使用x86_64 CPU的人可以运行我们的二进制文件。这里我们可以用`native`来代替我们的CPU或操作系统，以使用我们系统的.

`zig build-exe .\tiny-hello.zig -target x86_64-native -mcpu sandybridge`

关于哪些架构、操作系统、CPU和ABI（关于ABI的细节在下一章），可以通过运行`zig targets`找到。注意：输出结果很长，你可能想把它放到一个文件里，例如`zig targets > targets.json`.

# Zig Build

`zig build`命令允许用户根据`build.zig`文件进行编译。`zig init-exe`和`zig init-lib`可以用来给你一个基线项目.

让我们在一个新文件夹中使用`zig init-exe`。这就是你会发现的.
```
.
├── build.zig
└── src
    └── main.zig
```
`build.zig`包含我们的构建脚本。构建运行器*将使用这个 "pub fn build "函数作为其入口点--这就是你运行 "zig build "时执行的内容.

<!--no_test-->
```zig
const Builder = @import("std").build.Builder;

pub fn build(b: *Builder) void {
    // 标准目标选项允许运行`zig build`的人选择
    // 为哪个目标进行构建。这里我们没有覆盖默认值，这
    // 意味着任何目标都是允许的，默认是本地的。其他选项
    // 可用于限制支持的目标集.
    const target = b.standardTargetOptions(.{});

    // 标准优化选项允许运行`zig build'的人选择
    // 在Debug、ReleaseSafe、ReleaseFast和ReleaseSmall之间选择。这里我们不
    // 设置一个首选的发布模式，允许用户决定如何优化.
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "init-exe",
        .root_source_file = .{ .path = "src/main.zig" },
        .target = target,
        .optimize = optimize,
    });

    // 这宣布了当用户调用 "安装 "步骤时，可执行文件将被安装到标准位置的意图。
    // 当用户调用 "安装 "步骤（运行`zig build`时默认的
    // 运行`zig build`时的默认步骤）.
    b.installArtifact(exe);

    const run_cmd = exe.run();
    run_cmd.step.dependOn(b.getInstallStep());

    const run_step = b.step("run", "Run the app");
    run_step.dependOn(&run_cmd.step);
}
```

`main.zig`包含我们可执行程序的入口点。.

<!--no_test-->
```zig
const std = @import("std");

pub fn main() anyerror!void {
    std.log.info("All your codebase are belong to us.", .{});
}
```

使用`zig build`命令后，可执行文件将出现在安装路径中。这里我们没有指定安装路径，所以可执行文件将被保存在`./zig-out/bin`.

# Builder

Zig的[`std.Build`](https://ziglang.org/documentation/master/std/#A;std:Build)类型包含构建运行器所使用的信息。这包括诸如以下信息：

- 构建目标
- 发布模式
- 库的位置
- 安装路径
- 构建步骤


# CompileStep

`std.build.CompileStep`类型包含构建一个库、可执行文件、对象或测试所需的信息。

让我们利用我们的`Builder'，用`Builder.addExecutable'创建一个`CompileStep'，它接收一个名字和一个源码根的路径.

<!--no_test-->
```zig
const Builder = @import("std").build.Builder;

pub fn build(b: *Builder) void {
    const exe = b.addExecutable(.{
        .name = "init-exe",
        .root_source_file = .{ .path = "src/main.zig" },
    });
    b.installArtifact(exe);
}
```


# 模块

Zig构建系统有一个模块的概念，它是用Zig编写的其他源文件.让我们来使用一个模块.

在一个新的文件夹中，运行以下命令.

```
zig init-exe
mkdir libs
cd libs
git clone https://github.com/Sobeston/table-helper.git
```

目录结构应该是这样的.

```
.
├── build.zig
├── libs
│   └── table-helper
│       ├── example-test.zig
│       ├── README.md
│       ├── table-helper.zig
│       └── zig.mod
└── src
    └── main.zig
```

在你新制作的`build.zig`中，添加以下几行.

<!--no_test-->
```zig
    const table_helper = b.addModule("table-helper", .{
        .source_file = .{ .path = "libs/table-helper/table-helper.zig" }
    });
    exe.addModule("table-helper", table_helper);
```

现在，当通过`zig build`运行时，你的`main.zig`里面的[`@import`](https://ziglang.org/documentation/master/#import)将与 "table-helper "字符串一起工作。这意味着main拥有table-helper包。包（类型为[`std.build.Pkg`](https://ziglang.org/documentation/master/std/#std;build.Pkg)）也有一个类型为`?[]const Pkg`的依赖域，默认为空。这允许你拥有依赖其他软件包的软件包.

在你的`main.zig'中放置以下内容，并运行`zig build run'. 

<!--no_test-->
```zig
const std = @import("std");
const Table = @import("table-helper").Table;

pub fn main() !void {
    try std.io.getStdOut().writer().print("{}\n", .{
        Table(&[_][]const u8{ "Version", "Date" }){
            .data = &[_][2][]const u8{
                .{ "0.7.1", "2020-12-13" },
                .{ "0.7.0", "2020-11-08" },
                .{ "0.6.0", "2020-04-13" },
                .{ "0.5.0", "2019-09-30" },
            },
        },
    });
}
```

这个表打印到你的控制台 .

```
Version Date       
------- ---------- 
0.7.1   2020-12-13 
0.7.0   2020-11-08 
0.6.0   2020-04-13 
0.5.0   2019-09-30 
```

Zig还没有一个官方的软件包管理器。然而一些非官方的实验性软件包管理器确实存在，即[gyro](https://github.com/mattnite/gyro)和[zigmod](https://github.com/nektro/zigmod)。`table-helper`包就是为了支持这两个包而设计的。

一些寻找软件包的好地方包括： [astroabe.pm](https://astrolabe.pm), [zpm](https://zpm.random-projects.net/), [awesome-zig](https://github.com/nrdmn/awesome-zig/), 以及[GitHub上的zig标签](https://github.com/topics/zig).


# 构建步骤

构建步骤是为构建运行器提供任务的一种方式。让我们创建一个构建步骤，并将其作为默认步骤。当你运行`zig build`时，它将输出`Hello!`. 

<!--no_test-->
```zig
const std = @import("std");

pub fn build(b: *std.build.Builder) void {
    const step = b.step("task", "do something");
    step.makeFn = myTask;
    b.default_step = step;
}

fn myTask(self: *std.build.Step, progress: *std.Progress.Node) !void {
    std.debug.print("Hello!\n", .{});
    _ = progress;
    _ = self;
}
```

我们之前调用了`b.installArtifact(exe)`--这增加了一个构建步骤，告诉构建器要构建可执行文件.


# 生成文档

Zig编译器带有自动生成文档的功能。这可以通过在`zig build-{exe, lib, obj}`或`zig run`命令中添加`femit-docs`来调用。这些文档被保存在`./docs`中，作为一个小型静态网站。

Zig的文档生成利用了*doc注释*，它类似于注释，使用`///`而不是`//`，以及前面的globals。

在这里，我们将把它保存为`x.zig'，然后用`zig build-lib -femit-docs x.zig -target native-windows'为它建立文档。这里有一些东西需要注意：
- 只有带有文档注释的公开内容才会出现
- 可以使用空白的文档注释
- 文档注释可以使用markdown的子集
- 只有当编译器分析它们时，事情才会出现在生成的文档中；你可能需要强制分析以获得事情的出现。


<!--no_test-->
```zig
const std = @import("std");
const w = std.os.windows;

///**打开一个进程**, 获得一个句柄. 
///[MSDN](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-openprocess)
pub extern "kernel32" fn OpenProcess(
    ///[进程访问权限](https://docs.microsoft.com/en-us/windows/win32/procthread/process-security-and-access-rights)
    dwDesiredAccess: w.DWORD,
    ///
    bInheritHandle: w.BOOL,
    dwProcessId: w.DWORD,
) callconv(w.WINAPI) ?w.HANDLE;

///spreadsheet 位置
pub const Pos = struct{
    ///row
    x: u32,
    ///column
    y: u32,
};

pub const message = "hello!";

//用于强制分析，因为这些东西没有其他参考价值.
comptime {
    _ = OpenProcess;
    _ = Pos;
    _ = message;
}

//另一种方法是自动强制分析一切，但只在测试构建中:
test "Force analysis" {
    comptime {
        std.testing.refAllDecls(@This());
    }
}
```

使用`build.zig`时，可以通过在`CompileStep`上设置`emit_docs`字段为`.emit`来调用。我们可以创建一个生成文档的编译步骤，如下所示，然后用`$ zig build docs`来调用它.

<!--no_test-->
```zig
const std = @import("std");

pub fn build(b: *std.build.Builder) void {
    const mode = b.standardReleaseOptions();

    const lib = b.addStaticLibrary("x", "src/x.zig");
    lib.setBuildMode(mode);
    lib.install();

    const tests = b.addTest("src/x.zig");
    tests.setBuildMode(mode);

    const test_step = b.step("test", "Run library tests");
    test_step.dependOn(&tests.step);

    //构建步骤来生成文档:
    const docs = b.addTest("src/x.zig");
    docs.setBuildMode(mode);
    docs.emit_docs = .emit;
    
    const docs_step = b.step("docs", "Generate docs");
    docs_step.dependOn(&docs.step);
}
```

这种生成方式是试验性的，在复杂的例子中经常失败。这被[标准库文档](https://ziglang.org/documentation/master/std/)所使用.

当合并错误集时，最左边的错误集的文档字符串优先于右边的。在这种情况下，`C.PathNotFound`的文档注释是`A`中提供的文档注释.

<!--no_test-->
```zig
const A = error{
    NotDir,

    /// A doc comment
    PathNotFound,
};
const B = error{
    OutOfMemory,

    /// B doc comment
    PathNotFound,
};

const C = A || B;
```

# 第三章结束

本章还不完整。在未来，它将包含`zig build'的高级用法。

欢迎大家提出反馈意见和PR。