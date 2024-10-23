# Embedded IDE Lobotomy [Work in Progress]

A Beginner's Guide on Embedded System Programing without IDE.

[English Version Click Here (Not finished)](./Embedded_IDE_Lobotomy_EN.md)

### What's wrong with IDE?

If you have some experience with programming embedded system from an IDE like Code Composer Studio, Keil, or STM32CubeIDE, you might have experienced the same phase as I did:

You want to start a new project, so you copy and paste the same IDE project template you've been using since day 1. You are not sure what keeps this template running but you know moving a single header file around will definitely cause compilation errors. The dependency management is a mess, and even if you want to import a new library you are not sure where to put it. If the IDE complains a header file is not found, the only fix you know is to change `#include "header.h"` to `#include "../inc/header.h"`, click on `Build`, and hope it would work.

I've been through the same phaes and today I still can't say confidently I've left this phase of ignorance. However, I did put many efforts to find out what is happening behind an embedded IDE. And that, is why I wrote this tutorial:
 **To show Embedded Engineers what is happening behind an Embedded IDE, from source code to a running program on MCU, by replacing every step with Command-line tools**. To write down my findings and share them with others that have the same curiocity.


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

I wish this guide can be understood by an electronics hobbist or a undergrad freshman, but I also don't want to re-explain topics with existing well-written tutorials. **So this guide will cite many external resources and they are also part of this guide.**

### Wait...why would I want to do IDE Lobotomy?

I got asked this question once during an interview so I made something up. Here's a list.

- More flexible file structure
- Better compatibility with Git/Github
- Can use whatever editor you prefer
- Easy unit test
- Easy pipeline debug print to other CLI tools
- Doesn't crash
- Open Source

I can't make up more, but I'm sure you know what you are doing if you read this far. Hope you have fun reading this guide because I know I did.

> Removing IDE from you life is just like having Lobotomy: no more pain for your life without absolutely zero side effect!

### Index
- Compile from commandline on a PC
- How to control peripherals on a MCU (Registers, Memory map, and Libraries)
- How to compile lots of source files (makefile and CMake)
- Why PC can compile against MCU (Cross-compile and Toolchain)
- How the MCU knows what code to run (Load and Reset)
- How debugging works (OpenOCD, GDB, SWD vs JTAG, debug probe, gdb server)
- Case analysis with `illini-robomaster`

---

# 嵌入式IDE切除术

一份不用IDE进行嵌入式编程的新手指南。

[中文版点此（未上线）]()

### IDE怎么了？
如果你有一些使用IDE，比如Code Composer Studio, Keil, 或者STM32CubeIDE，进行嵌入式编程的经验，你大概也会和我一样经历这样一个阶段：

你想开一个新项目，于是你复制粘贴了那个祖传IDE工程模板。你不太确定到底是什么东西让这个模板正常运转，但你知道乱动头文件一定会导致编译报错。依赖项管理非常糟糕，你想要导入一个新库也不确定该放哪。如果IDE告诉你头文件未找到，你也只敢把 `#include "header.h"`改成 `#include "../inc/header.h"`然后点击`Build`，希望这次一切正常。

我也经历过这个无知的阶段，应该说，今天我依然不能自信自己已经完全离开了这种无知。不过，我确实做出了很多努力，去探索这些嵌入式IDE之下到底发生了什么。这也是我写这篇指南的初衷：**通过将每一步用命令行工具替换，向嵌入式工程师展示从源码到单片机上的程序，嵌入式IDE里面到底发生了什么。**写下我的这些发现，也希望可以分享给其他怀有同样好奇心的人们。

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

我希望这篇教程可以被电子爱好者和大一新生轻易读懂，但我同时又不愿意重新解释一些已经有非常优秀的其他教程的主题。**所以本指南会引用许多外链，这些外链也是本指南的一部分。**

### IDE切除术……我为啥要这么做？

我在一次面试里面被问过这个问题所以我现编了点，好处如下：

- 更灵活的文件结构
- 更好的Git/Github兼容性
- 想用什么编辑器都可以
- 单元测试更简单
- 调试 print 可以直接接到其他命令行工具上
- 不崩溃
- 支持开源

好我编不下去了，但是你都看到这里了我觉得你是知道自己在做什么的。希望你能在阅读本指南时获得和我探索时一样多的快乐。

> 将 IDE 扔出你的生活就好像接受脑叶切除术：生活不再痛苦，并且没有任何副作用！

### 目录

- 在电脑上从命令行编译
- 单片机怎么控制外设（寄存器，内存映射，与库）
- 如何编译大量的源文件（makefile 和 CMake）
- 电脑为什么可以编译单片机（工具链和交叉编译）
- 单片机怎么知道要运行什么（烧录和复位）
- 调试又是什么原理（OpenOCD，GDB，SWD vs JTAG, 调试器，gdb server）
- 实例分析：以`illini-robomaster`为例
