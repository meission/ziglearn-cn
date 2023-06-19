---
title: "Chapter 0 - 入门"
weight: 1
date: 2023-04-28 18:00:00
description: "Ziglearn --Zig编程语言指南/教程。在这里安装并开始使用ziglang."
---

# 欢迎

[Zig](https://ziglang.org) 是一种通用编程语言和工具链，用于— __维护健壮__、 __优化__ 和  __可重用__ 的软件。

警告：最新的主要版本是0.10.1 - Zig仍然是1.0之前的版本；仍然不建议在生产中使用，你可能会遇到编译器的错误。.

遵循本指南，我们假设你有：
   * 先前的编程经验
   * 对低级别的编程概念有一定了解

了解一种语言，如C、C++、Rust、Go、Pascal或类似的语言，将有助于遵循本指南。你应该有一个编辑器、终端和互联网连接。本指南是非官方的，与Zig软件基金会没有关系，旨在从头开始按顺序阅读.

# 安装

**本指南假定你使用的是Zig的主版本**，而不是最新的主要版本，这意味着从网站上下载二进制文件或从源代码编译；**你的软件包管理器中的Zig版本可能已经过时**。本指南不支持Zig 0.10.1。

1.  从以下网站下载并提取Zig的预建主二进制文件:
```
https://ziglang.org/download/
```

2. 增加Zig配置 path
   - linux, macos, bsd

      将Zig二进制文件的位置添加到你的`PATH`环境变量中。在安装时，将`export PATH=$PATH:~/zig`或类似的内容添加到你的`/etc/profile`（全系统）或`$HOME/.profile`。如果这些变化没有立即应用，请从你的shell中运行这一行。

   - windows

      a) 系统级别 (admin powershell)

      ```powershell
      [Environment]::SetEnvironmentVariable(
         "Path",
         [Environment]::GetEnvironmentVariable("Path", "Machine") + ";C:\your-path\zig-windows-x86_64-your-version",
         "Machine"
      )
      ```

      b) 用户级 (powershell)

      ```powershell
      [Environment]::SetEnvironmentVariable(
         "Path",
         [Environment]::GetEnvironmentVariable("Path", "User") + ";C:\your-path\zig-windows-x86_64-your-version",
         "User"
      )
      ```

      关闭你的终端并创建一个新的终端.

3. 用 `zig version`验证你安装的zig. 输出结果应该是这样的
```
$ zig version
0.11.0-dev.2777+b95cdf0ae
```

4. (可选，第三方）为了在你的编辑器中进行补全和去定义，请安装Zig语言服务器，从:
```
https://github.com/zigtools/zls/
```
5. (可选) 加入 [Zig community](https://github.com/ziglang/zig/wiki/Community).

# Hello World

创建 `main.zig`, 其内容:

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("Hello, {s}!\n", .{"World"});
}
```
###### (注意：确保你的文件使用空格缩进，LF行尾和UTF-8编码!)

使用`zig run main.zig`来构建和运行它。在这个例子中，`Hello, World!`将被写入stderr，并假定永远不会失败。
