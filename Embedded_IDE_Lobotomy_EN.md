# Embedded IDE Lobotomy

2024.10

If you have some experience with programming embedded system from an IDE like Code Composer Studio, Keil, or STM32CubeIDE, you might have experienced the same phase as I did:

You want to start a new project, so you copy and paste the same IDE project template you've been using since day 1. You are not sure what keeps this template running but you know moving a single header file around will definitely cause compilation errors. The dependency management is a mess, and even if you want to import a new library you are not sure where to put it. If the IDE complains a header file is not found, the only fix you know is to change `#include "header.h"` to `#include "../inc/header.h"`, click on `Build`, and hope it would work.

I've been through the same phaes and today I still can't say confidently I've left this phase of ignorance. However, I did put many efforts to find out what is happening behind an embedded IDE. And that, is why I wrote this tutorial:
 **To show Embedded Engineers what is happening behind an Embedded IDE, from source code to a running program on MCU, by replacing every step with Command-line tools**. To write down my findings and share them with others that have the same curiocity.

## Compile from commandline on PC

Computers speak machine code of 0's and 1's and each CPU family speaks a little different machine code. The most common example of this is AMD CPU with ARM architecture vs Intel CPU with x86-64 architecture. If you copy a exe file working on one to the other, it's likely* that it would not work or run very slow (because Windows is doing a real-time translation job).

> *I didn't try it. Some guy on stackoverflow claims this is true.

Writing program in machine code for every CPU ever built is impractical. So people decided to write program in a higher level language (C/C++). If each CPU can have their own translator from this higher level language to their machine code, a program can run on any CPU. This translator is called the "compiler".

> And the ability to run on different CPU families is called "cross platform". Like Python can run on ARM and x86.

#### GCC

`GCC (GNU Compiler Collection)` family is probably the most well-known C/C++ compiler**. To learn how to use `gcc` on a PC is our first step to understand embedded IDE.

> **Actually, most "compilers" today are drivers that call compiler, assembler, or linker depending on the input file and command.

Getting `gcc` on Linux is trivial. If you are on Windows, you can use a Linux virtual machine (I use `vmware player 17` + `Ubuntu 22.04`). Or, use `MinGW-W64` + `Git Bash` (remember to add `mingw64/bin` to system PATH) if you choose this way.

UCLA has a good short tutorial on how to use gcc to compile C/C++ program: `https://web.cs.ucla.edu/classes/fall14/cs143/project/cpp/gcc-intro.html`

#### .exe, .out, and .elf

Let's take a look at what we have.

```
gcc main.c
```

On Windows, this will give you `a.exe`.

On Linux, this will give you `a.out`.

We said compilers translate C to CPU specific machine code. If we are compiling against the same CPU, that means these 2 executables are the same thing, right? ...right?

**No.** Even though they contain the same* machine code, how the machine code is packaged into an executable is different.

> *Not entirely the same if we use external libraries or system calls.

When we try to run `a.out` on Windows, or `a.exe` on Linux, it doesn't work because these exectuables contain operating system specific information besides machine code.

`.exe (Executable)` file is a Microsoft OS specific executable format, while `.out` file is a Unix/Linux specific executable format called `Executable and Linkable Format (.elf)`.

> `.elf` and `.out` files are the same thing with different names. It's called `a.out` for only historical reasons.

So what are this extra OS specific information? Instructions for the OS about how to handle this executable. For example, relocation information: executables can ask OS to put them in a specific** memory address when running.

> **If multiple processes ask to locate at the same memory address, they will not collide because of virtual memory.

Okay but why do we care about these different executables and additional information for OS?

Because when we load programs to embedded MCU without an OS, these OS instructions are not needed. We can't load `.exe` or `.elf` to MCUs. Instead, we load binary files `.hex` or `.bin`, which only contain machine code and data that will loaded to a fixed memory address based on CPU design by a external hardware loader. Program counter of the MCU returns to this address and starts running every time we power up, press reset button, or perform a software reset.

#### Mildly more advanced compiling

Most "compilers" today are drivers that call compiler, assembler, and linker depending on the file input.

When we call:
```
gcc main.c
```

It takes 4 steps to convert this source file into an executable: preprocessing, compiling, assembling, and linking. There are many excellent tutorials on this topic. I would not redo their work. I found this course slide from University of North Carolina at Charlotte, Introduction to Computer Architecture very helpful:
https://passlab.github.io/ITSC3181/notes/lecture02_CompilationAssemblingLinkingProgramExecution.pdf

## How to control peripherals on MCU (Registers, Memory map, and Libraries)
There are always many, many library source files in a embedded IDE project folder, but why do we need this many? The reason comes from to how we control peripherals in embedded systems.

There are 2 ways of controlling peripherals: **memory map** and **port map**. Either way needs hardware in CPU to support it. That's said, when designing a CPU, we need to decide what (and how many) hardware the CPU can access.

For port map, CPU uses special assembly commands to access peripherals. Our focus for this chapter is memory map, which is accessing peripherals with the same assembly commands like a real memory address.

When CPU want to access some memory address, the address will be sent to memory management unit (a hardware inside CPU). Some address are permanently directed to peripherals instead of memories.

For example, if our CPU uses 4 bits to express any address, each address is 1 byte on memory, and the memory chip has only 8 bytes.

Then the memory mapping can be like this:

```
0000b - 0111b 8 bytes memory
1000b - 1111b 8 GPIO pins
```

We decide our CPU should permanently map `1000b - 1111b` to 8 GPIO pins with hardware circuits, so every time CPU reads a memory in this range, memory management unit gives it the values from GPIO pins. CPU doesn't know or care it is reading from real memory or peripherals.

One possible implementation of such hardware circuit, is to wire the most significant bit of the memory address register to a multiplex that select from the return values of memory chip and GPIO.

// TODO finish this chapter

## How to compile lots of source files (makefile and CMake)



- Why PC can compile against MCU (Cross-compile and Toolchain)
- How the MCU knows what code to run (Load and Reset)
- How debugging works (OpenOCD, GDB, SWD vs JTAG, debug probe, gdb server)

## What's inside a loader/debug probe/emulator

To load and debug code in a MCU, we need a "loader". It looks something like this:

<img src="./resource/st-link-v2.jpg" alt="Source: https://www.amazon.com/CANADUINO-Compatible-Circuit-Programmer-Debugger/dp/B07B2K6ZPK" height="200"/>

<img src="./resource/ST-LINK^V2.jpg" alt="Source: https://www.digikey.com/en/products/detail/stmicroelectronics/ST-LINK-V2/2214535" height="200"/>

<br>
<br>

There are many names for them.
- Loader
- Flasher
- FTDI chip
- SWD chip or JTAG chip
- Debug probe
- Emulator

We will explain why each name makes sense very soon in this chapter, but let's first look at how they connect to our hardware.

On the PC side, the connector is usually a USB. On the embedded side, depending on the chip manufacturer, we can have many choices.

- Texas Instruments 20 pin and 14 pin JTAG connector.
- ARM 20 pin JTAG connector.
- STMicroelectronics SWD connector.


#### JTAG

![Source: https://twosixtech.com/blog/running-a-baremetal-beaglebone-black-part-1/](./resource/Ti_arm_jtag.jpg)



JTAG was not invented specificly for MCU debugging. It was designed to detect whether all pins of a IC is properly soldered. This is known as "boundary scan".

JTAG should be able to set values for all registers/pins in the system. If we want to use JTAG to debug, we design the system such that all ICs are chained by the JTAG data line. Then we propagate data for each pin.

The number of bits we propagated should equals the the number of pins in the system when JTAG sets values for all pins of all the ICs in the system. By looking at the result (if the 2 numbers match) we will know if the ICs are properly soldered. When people say JTAG is designed for "boundary scan", this process is what they refer to.

When we say "JTAG", we are refering to the collection of the external debug hardware to propagate JTAG data + the connection from the JTAG hardware to the ICs.

> Of course, JTAG requires ICs to include some JTAG circuits as part of their design to work, which is a small cost compared to debugging cost.

This is the best JTAG introduction I've found: `https://www.fpga4fun.com/JTAG1.html`


#### SWD (vs JTAG)

If you never worked with ARM chips you've probably never heard of SWD (serial wire debug) connector. SWD is modified version JTAG and only used on ARM Cortex chips like STM32.

SWD is built on top of JTAG. So if a chip supports SWD, it also supports JTAG. SWD needs 2 data pins: `SWDIO` and `SWDCLK`. JTAG needs 4 data pins: `TMS`, `TCLK`, `TDO`, and `TDI`. In fact, some of their pins are compatible (`SWDIO == TMS`, `SWDCLK == TCLK`) and that why you may sometimes see connectors that can be used for both SWD and JTAG.

![Source: https://wiki-power.com/en/SWD%E4%B8%8EJTAG%E7%9A%84%E5%8C%BA%E5%88%AB%E4%B8%8E%E8%81%94%E7%B3%BB/](./resource/JTAG_SWD_Connector.png)

For chips supporting both debug ports, they default to JTAG on a cold boot and switch to SWD by sending a 50 clock cycle reset high, a 16 bit command sequence, and another 50 clock cycle reset high.

#### Different names for the same thing...

We discussed in the very beginning of this chapter that there are many names for these loaders. Let's see where they all came from.

**Loader/Flasher**:

This is a straightforward one. Most MCUs, on reset, will start to execute code from a fixed address. For STM32 as an example, the address is `0x80000000`. When loading a program, we load the beginning of the program to the starting address of the MCU. That's why it's called a loader.

Flash is a common storage unit for embedded systems and writing to a flash makes the name "flasher".

**SWD/JTAG chip**:

These are the names of the 2 connectors used for debugging as we discussed in the last part.

**Debug probe/Emulator**

The difference between a debug probe and a emulator is a little tricky. To explain this, we first need to know why loaders are sometimes called **FTDI chips**. FTDI is a chip company widely known for their USB-JTAG chips (FT2232 series) and their name is tightly binded to USB-JTAG chips.

Recall from the previous part on JTAG, we know JTAG can load programs into memory. This is because we know that JTAG can set value for all registers in the system and that includes `memory address register (MAR)` and `memory data register (MDR)`. That's said, JTAG can set any value to any memory address with the right input, and that "any value" in memory can be our code.

 If we only want to load the program to the MCU (or, a loader), a JTAG chip (FTDI chip) and some software drivers on PC is all we need. However, when we look at a loader, it sometimes has more ICs than only a FTDI (JTAG) chip. What are these then?

![Source: https://olimex.wordpress.com/2012/04/26/tms320-xds100-v3-prototypes-are-ready/](./resource/tms320-xds100-v3.jpg)

The picture above is a XDS100v3 loader/debugger for Texas Instruments TMS320  MCU series. The smaller chip in the white sqaure is a FTDI chip (FT2232H). What is this larger chip for? To add breakpoints for debugging.

When a breakpoint is added, we need to watch the value of the program counter, and stop the program when the address equals to the breakpoint address. Although this is an easy task on PC, in embedded systems, you need external hardwares to add these breakpoints. Some MCUs allow you to add breakpoints with internal circuits, but the number of breakpoints you can add is usually very limited.

>For MCU TMS320F2837, the number of breakpoints we can add without any external hardware is 2.

Thus, loaders add another middle MCU between the target MCU and the FTDI chip. When we send JTAG "commands" to the target MCU, the middle MCU capture the command and control the target MCU accordingly. When more breakpoints are required, the middle MCU uses its memory to record all the breakpoint addresses and stop the target MCU when the address in the program counter matches. **Since this middle MCU on the loader is acting as if the JTAG is directly controlling the target MCU (emulating), the loader is called an emulator.**

For some MCU families, the FTDI chip and the middle chip is combined into one. For example, the ST-link for STM32 series.

<img src="./resource/st-link-v2-mini-1_1_5.jpg" alt="Source: https://www.waveshare.com/st-link-v2-mini-stm32.htm" height="200"/>
<br><br>


- Case analysis with `illini-robomaster`
