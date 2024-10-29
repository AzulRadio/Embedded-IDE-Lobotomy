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

```bash
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
```bash
gcc main.c
```

It takes 4 steps to convert this source file into an executable: preprocessing, compiling, assembling, and linking. There are many excellent tutorials on this topic. I would not redo their work. I found this course slide from University of North Carolina at Charlotte, Introduction to Computer Architecture very helpful:
https://passlab.github.io/ITSC3181/notes/lecture02_CompilationAssemblingLinkingProgramExecution.pdf

## How to control peripherals on MCU (Registers, Memory map, and Libraries)

There are always many, many library source files in a embedded IDE project folder, but why do we need this many? In this chapter we will introduce the hierarchy of library for embedded system.

Let's start with an overview of all 4 layers of libraries of an embedded project. There are 4 layers: **Register level control** and **MCU library (manufacturer library)** are provided by the chip manufacturer; **bsp library** and **non-bsp library (higher level library)** are written by the user (you).

> Disclaimer: This is only the way I use to structure the project and by no means is a universal standard.

![](./resource/bsp_vs_non_bsp.png)

#### Register level control

At the very bottom is the **register level control**. All actions of a MCU is controlled by changing values of some registers. For example, to change the behavior of a GPIO between input mode and output mode, we need to change the control register of input/output multiplexer.

The code for register level control usually look like this:

```C
Blink an LED on MCU STC89C52
sfr and sbit are special data type defined for these MCUs

/*  BYTE Registers  */
sfr P0    = 0x80;
sfr P1    = 0x90;
sfr P2    = 0xA0;
sbit led = P2^0;

int main() {
  while(1) {
    led = 0;
    delay(5000);
    led = 1;
  }
}
```

#### MCU library (manufacturer library)

On top of register layer is what I called the "manufacturer library". When we program at the register level, you always need to know the **memory address** of a register to control it, which is tedious. So many MCU manufacturers write their library and you, the user, don't need to care about these registers. For example, `HAL` for `stm32` family, `Driverlib` for `TI C2000` family.

The code for MCU library level usually look like this:
```C
Blink an LED on MCU STM32F1
...
while (1)
{
  HAL_GPIO_WritePin(GPIOC, GPIO_PIN_9, 0);
  HAL_Delay(100);
  HAL_GPIO_WritePin(GPIOC, GPIO_PIN_9, 1);
}
...
```

Another benefit is many MCUs from the same family can use the same manufacturer library API. Now we are using `STM32F1`. If we want to switch to its close relative `STM32F4`, we can use the same piece of user code to blink LED on our new MCU.

#### Non-bsp library (high level library)

The entire point of bsp library is to make higher level library cross platform, so let's skip the bsp library for now and look at what non-bsp library is trying to do. The high level library's API depends on what embedded system you (the user) want to build. If designed properly, controlling your system with your high level library should be very convenient.

Suppose we want to build an LED array to display numbers and use PWM to control brightness. Sounds very complicated, but with properly designed high level library, our final user code using the high level library might look as simple as this:

```C
Display 4 on an LED array

led_array_t led_array = {some initialization here};
int main() {
  while (1) {
    DisplayNumber(led_array, 4);
    SetBrightness(led_array, 0.5); // Set brightness to 50%
  }
}
```
#### bsp library

Now we return to the bsp library. The bsp library is the bridge layer between MCU library and high level library. We define a common API between higher level library and bsp library. Then implement this common API with MCU library.

For example, if we want to implement the `SetBrightness()` function in the example above, with `stm32`'s `HAL` library, the implementation might be like this:
```C
in led_array.c, high level library
void SetBrightness(led_array_t *dev, float input) {
  SetPWM(dev, input);
}

in bsp_pwm.c, bsp library
// I didn't test this
void SetPWM(led_array_t *dev, float input) {
  uint32_t compare = input * dev->TIME_PERIOD;
  __HAL_TIM_SET_COMPARE(dev->htim, dev->channel, compare);
}
```

With `TI C2000`'s `Driverlib`, the implementation might be like this:
```C
in led_array.c, high level library
void SetBrightness(led_array_t *dev, float input) {
  SetPWM(dev, input);
}

in bsp_pwm.c, bsp library
// I didn't test this
void SetPWM(led_array_t *dev, float input) {
  uint32_t compare = input * dev->TIME_PERIOD;
  EPWM_setCounterCompareValue(dev->pwm_base, dev->compare_id, compare);
}
```

Since we defined the common API `SetPWM()`, The implementation of high level library gets to stay the same across different MCUs. This is what we meant by cross-platform. **If we change to a slightly different MCU or the MCU library changes, the common API between bsp library and high level library allow us to only change bsp library implementation, instead of all the final user code or high level libraries.**

## How to compile lots of source files (makefile and CMake)

With all the libraries we introduced in the last chapter, there will be numerous source files in the project and their number will keep increasing. We can't do this the same way as chapter 1 and we need an automatic way to manage these compilation. So, we introduce `makefile` and `CMake`.

There are many makefile and CMake tutorials but I feel many of them failed to stress on the core idea of **target**. So I'm writing another one as a supplementary to those tutorials. You might want to read this chapter before looking at the other CMake/makefile tutorials.

Suppose we have 2 source files, `hello.c` and `main.c`. We want to compile these 2 source files into an executable. From the first chapter **compiling from commandline** we know how to do this with `gcc`:

```bash
gcc hello.c main.c
```

If we want the output executable to have name `main.elf`, put it in folder `build/` specify that we are using `C11` standard, and add a hidden `#define DEBUG` macro to enable some debug setting in the source code, the command would be:

```bash
gcc hello.c main.c -o build/main.elf -std=c11 -DDEBUG
```

As there are more parameters we would like to add, the command grows longer. **Makefile** is a script language that allows us not to type the long command every time. Now we only need to type:

```bash
make build/main.elf
```

> To do this with MinGW on Windows, you need to rename `mingw32-make.exe` to `make.exe`

`make` will look for a file called `makefile` in the directory it is called and execute the commands in it. Let's look at what `makefile` have in our example:

```makefile
build/main.elf: hello.c main.c
    gcc hello.c main.c -o build/main.elf -std=c11 -DDEBUG
```

`build/main.elf` is the **target** (this is the most important concept for both makefile and CMake). For every target in the makefile script, `make` will check whether it exists and execute the command if the target doesn't exist.

`hello.c` and `main.c` are the **dependencies**. `make` will also execute the command if these files are changed since last build. If we have a dependency tree, only building necessary files will save a lot of time.

Command is just command. Notice it has to be a `tab` before the command and cannot be `whitespace`.

Now we know the 2 conditions for a command to run: target doesn't exist or dependencies are changed. We can write some interesting makefile scripts.

If the target is not created by the command, then this command will run every time we `make` because the target is always missing. For example:

```makefile
clean:
    rm *.out *.elf
```

If we run `make clean`, all `.out` and `.elf` files will be removed in the current directory.

#### CMake

CMake is when makefile is too long, and we need to write another script to generate the makefile script. Many tutorials on CMake are elaborated on usage details which makes them confusing and wordy, but all you need to know is: **CMake is (like makefile) all about targets**.

Remember makefile is all about targets and every line in makefile is about generating one target. CMake generates makefiles, if no targets are defined, no makefile will be generated by this CMake.

There are only 3 functions that defines targets in CMake:

```Python
# Executables that you can generate with GCC or GCC-like compilers
add_executable()

# Libraries (.a/.so files) that you can generate with GCC or GCC-like compilers
add_library()

# Something that generated from other commandline tools
# Example: python script that generate files
# ...Or nothing, remember make clean?
# We can use this to load programs to MCUs
add_custom_target()
```

The first two functions `add_executable()` and `add_library()` will detect if the target is created and if the dependencies are changed as `makefile` does. The third function `add_custom_target()`, however, will execute no matter what. Thus a little trick with `add_custom_target()` is to use it with loader binary like:

```c
add_custom_target(flash-${name}
    COMMAND st-flash --reset write ${BIN_FILE} 0x8000000
    DEPENDS ${name}.elf)
```

Now every time we run `make flash-<your binary name>` like `make flash-led`, the corresponding binary, in this case, `led.elf` will be loaded to your MCU with `st-flash`, a binary loader for `STM32` MCU family.

That's it. That's core idea about CMake. **No matter how many variables you defined or how many `execute_process()` you added, if there is no target to be generated, none of the CMake commands will be executed.**

With that in mind you should be able to understand most of the CMake tutorials available.

> CMake is a very deep rabbit hole and how deep you decide to go in is up to you. I've warned you.

## Why PC can compile against MCU (Cross-compile and Toolchain)

To compile a source file to an executable. We can simply do:

```bash
gcc main.c
```

Recall the very beginning of our **chapter 1: compile from commandline on PC**, we know that:

1. Different MCU/CPUs speak different machine code
2. Executable for one platform (CPU + OS) cannot run on another platform.

These are the two primary reasons for having compilers, to translate source code into corresponding machine code. Wait! When we compile for a embedded system on PCs, how can the program run on the embedded MCU even if it is compiled on a PC? The answer is what we called **cross-compile**.

Let's examine the difference between **compile** and **cross-compile**:

![](./resource/cross_compile.png)

For a source file `.c`, if we want to run it on our PC, we compile it with `gcc`; if we want to run it on our ARM architecture embedded system MCU, we cross-compile it with `arm-gcc`, a cross-compiler on our PC, and then generated executable can run on our embedded system.

`arm-gcc`, the cross-compiler, is one of the **cross-compiling toolchain** provided by the architecture designer. If we are using ARM MCUs, we can get our cross-compiling toolchain from ARM: [ARM Toolchain](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads). This is a GNU Arm Toolchain, meaning it is an variation of the `gcc` family we've been talking about in the previous chapters and compatible to `makefile` and `CMake`. The usage is exactly the same as compiling against our PC.

#### Example:

Suppose our PC is x86-64 Ubuntu and our MCU is ARM architecture STM32F4. This is a **bare-metal target**, which means it doesn't have any operating systems on it. So the cross-compiling toolchain we get is `x86_64 Linux hosted cross toolchains AArch32 bare-metal target (arm-none-eabi)` and our cross-compiler is `arm-none-eabi-gcc`.

To use these in our `CMake` script, add the following commands:

```cmake
set(CMAKE_C_COMPILER arm-none-eabi-gcc)
set(CMAKE_CXX_COMPILER arm-none-eabi-g++)
```


- How the MCU knows what code to run (Load and Reset)
- How debugging works (OpenOCD, GDB, gdb server)

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
