# Embedded IDE Lobotomy [Work in Progress]

A Beginner's Guide on Embedded System Programing without IDE.

[English Version Click Here](./Embedded_IDE_Lobotomy_EN.md)

If you have ever spent some time programming an embedded system, you might agree that working with embedded IDEs is not a very pleasant experience. Code Composer Studio, Keil, STM32CubeIDE... they are clumsy and confusing as they hide everything under a GUI.

To get rid of embedded IDEs, I put many efforts to find out what is happening behind them. Today, I still can't say confidently that I've learnt everything about them, but I went through many interesting topics, interesting enough for me to **replace every part of an IDE with some open sourced command-line tools.**

Then, I met some other people with the same interest, which gave me a reason to write down my findings. That's why I wrote this beginner's guide: **To show Embedded Engineers what is happening behind an Embedded IDE, from source code to a running program on MCU, by replacing every step with Command-line tools**.

Even if you don't want to replace your IDE with command-line tools, I hope you can find something interesting reading this guide.


### Target reader:

This guide is for you if you:

- know how to use Unix/Linux
- know how to write simple C code
- know how to load program into embedded MCUs with JTAG/SWD loader
- roughly know what is machine code and assembly
- roughly know how CPU works (PC, IR) and how it talks with memory (MAR/MDR)
- want to throw away IDEs
- want to replace some parts of IDE with CLI tools
- want to learn about how IDEs work

**For some common topics with many existing excellent tutorials, I cite external resources as part of this guide. For best experience you might also want to read them.**

I hope this guide can be understood by an entry level person like a undergrad ECE freshman. Please tell me if something is poorly explained.

### Why do I want to get an IDE Lobotomy?

I got asked this question once during an interview so I made something up. Here's a list.

- More flexible file structure
- Better compatibility with Git/Github
- Can use whatever editor you prefer
- Easy unit test
- Easy pipeline debug print to other CLI tools
- Doesn't crash
- Open Source

I'm sure you know what you are doing if you read this far. Hope you have fun reading this guide because I know I did.

> Removing IDE from you life is just like having Lobotomy: no more pain for your life without absolutely zero side effect!

### Index
- Compile from commandline on PC
- How to control peripherals on MCU (4 layers of libraries we need to compile)
- How to compile lots of source files (makefile and CMake)
- Why PC can compile against MCU (Cross-compile and Toolchain)
- What's inside a loader/debug probe/emulator
- How debugging works (OpenOCD, GDB, gdb server)
- Case analysis with `illini-robomaster`

---

# 嵌入式IDE切除术

一份不用IDE进行嵌入式编程的新手指南。

[中文版点此（未上线）]()

如果你有一些嵌入式编程的经历，你大概会同意使用IDE并不是一件特别愉快的事。Code Composer Studio, Keil, 或者STM32CubeIDE……他们很不灵活，也很难搞清楚GUI之下到底在发生些什么。

我花了很多精力去研究IDE的运行原理，试图绕过IDE。应该说，今天我依然不能自信自己已经完全搞明白了嵌入式IDE。不过，我学到了很多有趣的知识，有趣到可以让我**把整个IDE换成开源的命令行工具**。

很快，我遇到了抱有同样兴趣的其他人。于是我就有了一个去把我的发现写下来的理由，也是我写这篇指南的初衷：**通过将每一步用命令行工具替换，向嵌入式工程师展示从源码到单片机上的程序，嵌入式IDE里面到底发生了什么**。

即便你不打算把IDE换成命令行工具，我也希望你能在这篇指南中找到一些有趣的东西。

### 目标读者：

这篇指南适合下列读者：
- 会用 Unix/Linux
- 会写基本的 C
- 知道怎么给单片机用 JTAG/SWD 烧录器刷程序
- 大概知道什么是机器码和汇编
- 大概知道CPU怎么工作（PC, IR）怎么读内存（MAR, MDR）
- 想把 IDE 扔出窗外
- 想用命令行工具替换一部分的 IDE
- 想学习IDE是怎么工作的

**对于一些已有许多优秀教程的主题，我在本指南中直接引用了他们的外链。为了最佳体验，请一并阅读他们**。

> 本指南最早是英语写的，有些外链会是英文

我希望这篇教程可以让真正的入门者，比如大一计算机/电子新生读懂。如果有东西解释的很糟糕，请告诉我。


### IDE切除术……我为啥要这么做？

我在一次面试里面被问过这个问题所以我现编了点回答，好处如下：

- 更灵活的文件结构
- 更好的Git/Github兼容性
- 想用什么编辑器都可以
- 单元测试更简单
- 调试 print 可以直接接到其他命令行工具上
- 不崩溃
- 支持开源

好我编不下去了，但是你都看到这里了我觉得你是知道自己在做什么的。希望你能在阅读本指南时和我一样获得快乐。

> 将 IDE 扔出你的生活就好像接受脑叶切除术：生活不再痛苦，并且没有任何副作用！

### 目录

- 在电脑上从命令行编译
- 单片机怎么控制外设（4层需要编译的库）
- 如何编译大量的源文件（makefile 和 CMake）
- 电脑为什么可以编译单片机（工具链和交叉编译）
- 烧录器/仿真器/调试器里面有什么
- 调试是什么原理（OpenOCD，GDB，gdb server）
- 实例分析：以`illini-robomaster`为例
