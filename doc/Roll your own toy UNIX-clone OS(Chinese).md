

# Roll your own toy UNIX-clone OS

>英文原文链接：http://www.jamesmolloy.co.uk/tutorial_html/
>
>原作者： **James Molloy**  
>
>译： **Curled**
>
>注：因为译者水平有限，如有谬误，还望指出，谢谢！

[TOC]

## 前言

这一系列教程旨在带领你编写一个简单的x86架构的类`UNIX`操作系统。本教程采用C语言作为首选的编写语言，并混合了大量的汇编代码，目的是让你理解在制作操作系统过程中的一些设计与策略。为了让整个过程更简单，我们的操作系统被设计为一个单一的整体（驱动程序在内核态下加载而非用户态）。

事实上，这一系列教程非常实用。每个部分都给出了相应的原理，但是本教程的大部分内容里主要还是涉及如何实现这些抽象思想和机制。需要注意的是，我们实现的内核只是一个教学级别的内核。我知道这里面所用的算法在空间上并不是最优或者最有效的，选择它们仅仅是为了让读者好理解。这能给你一个正确的方向，一个可以着手码代码的基础。并且这个内核是可拓展的，你也可以为它编写更好的算法。如果你对文中的理论知识理解起来有问题的话，有很多网站都能很好的帮助你（`OSDev`论坛上的大多数问题都与具体的实现有关——“我的`get`函数坏掉了!”救救孩子!”-理论问题真是一股清流;)），链接在页尾。

## 预备知识

为了正确的运行和编译我提供的实例代码需要：`GCC,ld,NASM`以及`GNU` 的 `make`工具。`NASM`是一个开源的x86汇编器，也是大多数x86架构操作系统开发人员的首选。

然而，仅仅只是把代码编译运行一遍是没有什么用的，你要理解所编写的代码的含义。为了做到这一点，你要对C语言有很深入的理解，尤其是关于指针这一部分的内容。同时你要了解一点点汇编知识（本教程中用的是`intel`语体的汇编），比方说你得知道`ebp`寄存器是干啥用的。

## 资源

如果你知道到到哪儿找（谷歌是个好东西）的话，其实资源是很多的。以下是你可能需要的：
- [RTFM!](http://www.intel.com/products/processor/manuals/index.htm)英特尔的手册是天赐之物
- [The osdev.org wiki](http://www.osdev.org/wiki) 以及 [forums](http://www.osdev.org/forum).
- [Osdever.net](http://www.osdever.net/)有很多教程和论文。特别推荐 [Bran's kernel development tutorials](http://www.osdever.net/bkerndev/index.php), 本书的早期代码就是基于这篇教程的. (我自己就是 看这个入门的，里面的代码写得太好了所以这些年我都没有换过)
- [alt.os.development](http://groups.google.co.uk/group/alt.os.development/topics)比较专业的问题可以在这上面提问，入门级别的问题建议还是去`osdever.net`论坛.

## 第一部分 环境搭建

我们需要从设计和制作内核的基础开始。我假设你用的是`*nix`系的操作系统，并且带了相应的`GNU`工具链 。如果你非要用windows系统的话，你可能会用到`cygwin`（一个模拟`*nix`环境的工具集）或者`DJGPP`。不管怎样，如果用`Windows`系统的话，这篇教程的`makefiles`和命令有可能跑不通。

### 1.1.目录结构

我的目录结构如下：

```bash
tutorial
 |
 +-- src
 |
 +-- docs
```

所有的源代码放在src目录下，你写的文档（你写文档吗骚年？）放进docs里面。

### 1.2.编译

这篇教程的实例代码可以被`GNU`工具链`（gcc，ld，gas，etc）`成功编译。实例的汇编代码采用`intel`语体编写，（个人认为）比`GNU`用的`AT&T`语体可读性更强一点。为了成功汇编，你需要[ Netwide Assembler](http://nasm.sourceforge.net/)。

这篇教程不是写`bootloader`的教程。我们会用[GRUB](http://www.gnu.org/software/grub)做我们内核的引导。为了做到这一点，我们需要一个预先装好`GRUB`的软盘。有很多[教程](https://blog.csdn.net/RichardGreenhhh/article/details/78087066) (译者ps：作者的原链接失效了，然后我找了一个还不错的教程)能教你在软盘上装`grub`。更幸运的是，我做了一个现成的镜像，可以从[这里]()下载（译者ps：不幸运的是，原链接已经失效，所以自己动动手吧）。

### 1.3.运行

没有比裸机作为一个测试系统更好的替代品了。不幸的是，裸机在检测代码跑到哪里出问题的能力上就是个弟弟（当然，你马上要第一次写完全没bug的代码了）。进入`bochs`的[官网](http://www.bochs.sourceforge.net/)。`bochs`是是一个开源的x86-64架构模拟器。当哪儿出问题的时候，`bochs`会告诉你，同时把处理器的状态保存在日志文件中，这是非常有用的。

### 1.4.Bochs

为了把`bochs`跑起来，你先得写一个`bochs`的配置文件`(bochs.txt)`。幸运的是，这里有一个示范的文件。

```bash
megs: 32
romimage: file=/usr/share/bochs/BIOS-bochs-latest, address=0xf0000
vgaromimage: /usr/share/bochs/VGABIOS-elpin-2.40
floppya: 1_44=/dev/loop0, status=inserted
boot: a
log: bochsout.txt
mouse: enabled=0
clock: sync=realtime
cpu: ips=500000
```

注意`bios`文件的目录。如果你是直接编译的`bochs`的源代码你很有可能么有这个文件目录。谷歌这些文件名，你就能从官网或者其他地方找全这些文件。

这个配置文件会让`bochs`模拟出一个内存为32MB、始终周期为350MHz的`PentiumⅡ`虚拟机。指令执行速度还能提高——但目前我更希望它慢一点，以便我能在大量文字滚动过去的时候能知道发生了啥。

### 1.5.有用的脚本

#### 1.5.1.Makefile

```makefile
# Makefile for JamesM's kernel tutorials.
# The C and C++ rules are already setup by default.
# The only one that needs changing is the assembler
# rule, as we use nasm instead of GNU as.

SOURCES=boot.o

CFLAGS=
LDFLAGS=-Tlink.ld
ASFLAGS=-felf

all: $(SOURCES) link 

clean:
 »  -rm *.o kernel

link:
 »  ld $(LDFLAGS) -o kernel $(SOURCES)

.s.o:
 »  nasm $(ASFLAGS) $<
```

该`makefile`能够编译`source`中的每个文件，并且能把它们链接成`elf`格式的二进制文件，`kernel`。它用了一个链接器脚本，`link.ld`，像这样：

```link
/* Link.ld -- Linker script for the kernel - ensure everything goes in the */
/*            Correct place.  */
/*            Original file taken from Bran's Kernel Development */
/*            tutorials: http://www.osdever.net/bkerndev/index.php. */

ENTRY(start)
SECTIONS
{
  .text 0x100000 :
  {
    code = .; _code = .; __code = .;
    *(.text)
    . = ALIGN(4096);
  }

  .data :
  {
     data = .; _data = .; __data = .;
     *(.data)
     *(.rodata)
     . = ALIGN(4096);
  }

  .bss :
  {
    bss = .; _bss = .; __bss = .;
    *(.bss)
    . = ALIGN(4096);
  }

  end = .; _end = .; __end = .;
```

该脚本告诉链接器如何组装你的内核镜像。首先它告诉链接器`start`标志是二进制文件的入口地址。接着它告诉链接器，`.text`段（代码存放的地方）应该在最前面，并且开始地址为`0x100000(1MB)`。`.data`段(已初始化的静态数据)和`.bss`段(未初始化的静态数据)应该接在`.text`段后面，而且每个都应该被页对齐`(ALIGN(4096))`。（译者ps：为什么要`ALIGN4096`可以参考[这篇文章]( https://blog.csdn.net/qiushanjushi/article/details/40341807 )）。Linux GCC 也会增加一个额外的段`.rodata`。这是为被初始化的只读数据准备的段，比如`const`常量。为了简化我们把它和`.data`段放一起讨论。

#### 1.5.3.update_image.sh

这是一个能把你的二进制内核数据写入软盘的小脚本（假设你已经创建了一个文件夹`/mnt`）。注意：为了使用`loseup`命令，你需要把`/sbin`文件夹包含在`$PATH`环境变量中。

```bash
#!/bin/bash

sudo losetup /dev/loop0 floppy.img
sudo mount /dev/loop0 /mnt
sudo cp src/kernel /mnt/kernel
sudo umount /dev/loop0
sudo losetup -d /dev/loop0
```

#### 1.5.4.run_bochs.sh

这个脚本会启动一个环回设备，在上面可以跑`bochs`，然断开连接。

```bash
#!/bin/bash

# run_bochs.sh
# mounts the correct loopback device, runs bochs, then unmounts.

sudo /sbin/losetup /dev/loop0 floppy.img
sudo bochs -f bochsrc.txt
sudo /sbin/losetup -d /dev/loop0
```



## 第二部分 正式开始

### 2.1.boot部分代码

是时候写一些代码了。尽管我们的内核主要是用C语言编写的，但还是有一部分的内容必须用汇编语言编写。其中之一就是`boot`代码。

```assembly
;
; boot.s -- Kernel 的起始位置. 同时定义了multiboot 头.
; 基于 Bran的 kernel 开发教程中的文件start.asm
;

MBOOT_PAGE_ALIGN    equ 1<<0    ; 把内核和模块加载到内存页的开始
MBOOT_MEM_INFO      equ 1<<1    ; 为内核提供内存信息
MBOOT_HEADER_MAGIC  equ 0x1BADB002 ; Multiboot 魔术数字
; 注意：我们不采用 MBOOT_AOUT_KLUDGE. 意思是说 GRUB 不会
; 给我们传递一个符号表.
MBOOT_HEADER_FLAGS  equ MBOOT_PAGE_ALIGN | MBOOT_MEM_INFO
MBOOT_CHECKSUM      equ -(MBOOT_HEADER_MAGIC + MBOOT_HEADER_FLAGS)


[BITS 32]                       ; 所有的指令都为32位

[GLOBAL mboot]                  ; 让 'mboot' 能被C代码调用
[EXTERN code]                   ; '.text'段的开始
[EXTERN bss]                    ; '.bss'段的开始
[EXTERN end]                    ; 最后一个可加载的段结束

mboot:
  dd  MBOOT_HEADER_MAGIC        ; GRUB 会在内核文件中按每四个字节的范围搜索这个值
  dd  MBOOT_HEADER_FLAGS        ; GRUB 加载你文件/设置的方式
  dd  MBOOT_CHECKSUM            ; 确保上述的值都是正确的
   
  dd  mboot                     ; 描述符的位置
  dd  code                      ;kernel '.text' 段
  dd  bss                       ;kernel '.data' 段结尾
  dd  end                       ;kernel 结尾位置.
  dd  start                     ; Kernel 入口点(EIP初始值).

[GLOBAL start]                  ; Kernel 入口点.
[EXTERN main]                   ; C代码入口点

start:
  push    ebx                   ; 加载 multiboot header的位置

  ; 运行 kernel:
  cli                         ; 禁止中断.
  call main                   ; 调用 main() 函数.
  jmp $                       ; 进入无限循环，防止kernel运行完后处理器执行内存中的垃圾
```

### 2.2.理解启动代码

事实上那个代码片段中只有几行关键代码：

```assembly
push ebx
cli
call main
jmp $
```

剩下的代码大多都跟`multiboot header`有关。

#### 2.2.1.Multiboot

`Multiboot`是`GRUB`希望`kernel`遵守的标准。

这可以让`bootloader`：

1.在启动时知道`kernel`需要的环境
2.允许`kernel`查询它所在的环境

所以，比方说，如果你的`kernel`需要在特定的`VESA`模式下加载（顺便说一下，这是不推荐的方式），你可以以这种方式通知`bootloader`，剩下的事情它会帮你处理好。

为了让我们的`kernel`的`multi boot`兼容性强，你需要在`kernel`的某个地方添加一个头结构（事实上，这个头必须位于`kernel`的前4KB里面）。有一个很有用的`NASM`伪指令允许我们在代码中嵌入特定的常量——`dd`。代码如下：

```assembly
dd MBOOT_HEADER_MAGIC
dd MBOOT_HEADER_FLAGS
dd MBOOT_CHECKSUM
dd mboot
dd code
dd bss
dd end
dd start
```

以上定义了 `MBOOT_* `类的常量。

**MBOOT_HEADER_MAGIC**

一个魔术数字。标志`kernel`是`multiboot`多兼容的。

**MBOOT_HEADER_FLAGS**

标志字段。我们要求`GRUB`对`kernel`的所有部分进行页面对齐`( MBOOT_PAGE_ALIGN )`，同时`GRUB`给我们一些内存的信息`( MBOOT_MEM_INFO )`。注意到有些教程也用 `MBOOT_AOUT_KLUDGE` 。考虑到我们使用的是`elf`的文件格式，所以没必要做这个修改，并且我们也不需要`GRUB`在启动时给我们一个符号表。

 **MBOOT_CHECKSUM**

 这个字段的定义是这样的：当魔术数字，标志字段和该字段相加的时候，总和必须为零。这一般用于验错。

 **mboot**

我们当前编写的头结构的地址。`GRUB`使用其判断是否需要重定位。

 **code,bss,end,start**

这些符号都是由链接器定义的。我们用它们来告诉`GRUB` `kernel`的不同部分位于何处。

在启动时，`GRUB`会将一个指向另外一个信息的结构体指针赋给`EBX`寄存器。这可以用来查询`GRUB`为我们设置的环境。

#### 2.2.2.再次回到代码……

所以，在启动的时候。这部分汇编代码告诉CPU把`EBX`寄存器压进栈里（别忘了`EBX`里有一个指向`multiboot`信息结构体的指针），禁止中断`（CLI）`，调用C的`main()`函数（目前我们还没写），然后进入一个无限循环。

一切都看起来很棒，但是代码显然是跑不通的，因为我们还没定义`main()`函数鸭！

### 2.3.添加一些C代码

在汇编中内联C代码是极其容易的。你只需要清除用到的调用约定。x86架构上GCC使用`__cdecl`的调用约定：

- 函数所有的参数用栈传递
- 参数压栈的方式为从右往左
- 返回值存储在`EAX`寄存器内

所以像这样的函数调用：

```c
d = func(a, b, c);
```

反汇编后是这样的：

```assembly
push [c]
push [b]
push [a]
call func
mov [d], eax
```

看到没？超级简单。你可以看到在以上的汇编代码中，`push ebx`实际上是把ebx寄存器的值作为参数传递给`main()`函数

#### 2.3.1.C代码

```c
// main.c -- 定义kernel的C代码入口点，调用初始化的例程.
// Made for JamesM's tutorials

int main(struct multiboot *mboot_ptr)
{
  // All our initialisation calls will go in here.
  return 0xDEADBABA;
}
```

这是我们`main()`函数的第一个版本。如你所见，我们向`main()`函数穿了一个参数——一个指向`multiboot`结构体的指针。我们稍后定义它（实际上我们没有必要为即将要编译的代码定义它）。

`main()`函数结束时返回一个常数——`0xDEADBABA`。这是一个不寻常的常数，一会儿它会很显眼的。

### 2.4.编译，链接，运行！

现在我们已经在项目中新建了一个文件，我们同时也要在`makefile`中添加这个文件。把这几行：

```makefile
SOURCES=boot.o
CFLAGS=
```
修改为：

```makefile
SOURCES=boot.o main.o
CFLAGS=-nostdlib -nostdinc -fno-builtin -fno-stack-protector
```

`CFLAGS`的作用是：不让GCC把linux c的库链接进我们的`kernel`里——它跑不起来（目前）。

好的，现在你应该可以编译、链接，然后运行你的kernel了。

```bash
cd src
make clean  # 忽略掉这里的任何错误
make
cd ..
./update_image.sh
./run_bochs.sh  # 可能会让你输root密码
```

这应该可以让你的`bochs`跑起来了，你可以看到`GRUB`启动了几秒钟然后`kernel`将会运行。实际上它啥都没干（main函数里面你也啥都没写啊……）。所以它仅仅只是打印了一句`Starting up`，然后就凝滞住。

如果你打开bochsout.txt文件，在底部你可以看见像这样的信息：

```bash
00074621500i[CPU  ] | EAX=deadbaba  EBX=0002d000  ECX=0001edd0 EDX=00000001
00074621500i[CPU  ] | ESP=00067ec8  EBP=00067ee0  ESI=00053c76 EDI=00053c77
00074621500i[CPU  ] | IOPL=0 id vip vif ac vm rf nt of df if tf sf zf af pf cf
00074621500i[CPU  ] | SEG selector     base    limit G D
00074621500i[CPU  ] | SEG sltr(index|ti|rpl)     base    limit G D
00074621500i[CPU  ] |  CS:0008( 0001| 0|  0) 00000000 000fffff 1 1
00074621500i[CPU  ] |  DS:0010( 0002| 0|  0) 00000000 000fffff 1 1
00074621500i[CPU  ] |  SS:0010( 0002| 0|  0) 00000000 000fffff 1 1
00074621500i[CPU  ] |  ES:0010( 0002| 0|  0) 00000000 000fffff 1 1
00074621500i[CPU  ] |  FS:0010( 0002| 0|  0) 00000000 000fffff 1 1
00074621500i[CPU  ] |  GS:0010( 0002| 0|  0) 00000000 000fffff 1 1
00074621500i[CPU  ] | EIP=00100027 (00100027)
00074621500i[CPU  ] | CR0=0x00000011 CR1=0 CR2=0x00000000
00074621500i[CPU  ] | CR3=0x00000000 CR4=0x00000000
00074621500i[CPU  ] >> jmp .+0xfffffffe (0x00100027) : EBFE
```

![genesis_bochs](png\genesis_bochs.png)

注意到`EAX`寄存器的值了没？`0xDEADBABA`——`main()`函数的返回值。恭喜你，你现在有一个多兼容的`multiboot`的程序模板了，接下来你要做的是打印一些东西到屏幕上了。

这个教程的实例代码在[这里](http://www.jamesmolloy.co.uk/tutorial_html/genesis.tar)。



## 第三部分 屏幕

所以，现在我们有了一个可以运行、运行后进入无限循环状态的`kernel`，是时候在屏幕上显示一些有趣的东西了。在调试的苦战里，串行I/O和显示器将会是你最重要的盟友。

### 3.1.一些先导理论

你的`kernel`通过`GRUB`引导以文本模式（textmode）启动。也就是说，它有一个 `framebuffer`（内存的一块区域）来控制字符（不是像素），宽80，高25。这将是你的`kernel`进入`VESA`（不在本教程范围内）之前的操作。

这块被称为`framebuffer`的内存区域和普通的RAM一样都是可读写的，它的起始地址为`0xB8000`。但要注意的是，它并不仅仅是普通的RAM，它是`VGA` 控制器专用视频内存的一部分，通过硬件将内存映射到线性地址空间。这是两者主要的区别。

`framebuffer`实际上是一个16-bit 一个字的数组，每16-bit值表示一个字符。特定的一个位于x，y位置的字符相对于开始位置的偏移量为：
$$
`(y * 80 + x) * 2`
$$
需要注意到的是`* 2`是因为每一个字符是2个字节（16bit）长。如果说你是在一个16bit元素组成的数组内索引一个值，那么你的数组下标应该是`y*80+x`。

在`ASCII`（文本模式不支持`unicode`编码）编码中，8bit被用来代表一个字符 。而这样我们就有了8bit没用到的位。VGA硬件用他们来指定前景和后景的颜色（每个4bit）。16bit的划分如下图所示：

![the_screen_word_format](png\the_screen_word_format.png)

4bit位可以表示15种颜色（译者ps：也不知道原作者怎么数的，明明是16位啊喂）：

` 0:黑色，1:蓝色，2:绿色，3:青色，4:红色，5:品红，6:棕色，7:浅灰色，8:深灰色，9:浅蓝色，10:浅绿色，11:浅红色，13:浅磁，14:浅棕色，15:白色。 `

VGA控制器在I/O串口上也有一些端口，你可以向它们发送特定的指令。它在0x3D4有一个控制寄存器，在0x3D5有一个数据寄存器。我们可以用它们指挥控制器实时更新光标的位置（一条闪烁的下划线指示下一个字符的位置）。

### 3.2.实践

#### 3.2.1.先做重要的部分

首先，我们需要一些更常用的全局函数。`common.c`和`common.h`包含读写I/O串口的函数，以及一些可以让我们更容易地编写可移植代码的类型定义。 同时，它们也是定义类似`memcpy/memset`函数的地方。我把它们留给你来完成! :)

```c
// common.h -- Defines typedefs and some global functions.
// From JamesM's kernel development tutorials.

#ifndef COMMON_H
#define COMMON_H

// Some nice typedefs, to standardise sizes across platforms.
// These typedefs are written for 32-bit X86.
typedef unsigned int   u32int;
typedef          int   s32int;
typedef unsigned short u16int;
typedef          short s16int;
typedef unsigned char  u8int;
typedef          char  s8int;

void outb(u16int port, u8int value);
u8int inb(u16int port);
u16int inw(u16int port);

#endif
```

```c
// common.c -- Defines some global functions.
// From JamesM's kernel development tutorials.

#include "common.h"

// Write a byte out to the specified port.
void outb(u16int port, u8int value)
{
    asm volatile ("outb %1, %0" : : "dN" (port), "a" (value));
}

u8int inb(u16int port)
{
   u8int ret;
   asm volatile("inb %1, %0" : "=a" (ret) : "dN" (port));
   return ret;
}

u16int inw(u16int port)
{
   u16int ret;
   asm volatile ("inw %1, %0" : "=a" (ret) : "dN" (port));
   return ret;
}
```

#### 3.2.2.显示器代码

一个简单的头文件：

```c
// monitor.h -- Defines the interface for monitor.h
// From JamesM's kernel development tutorials.

#ifndef MONITOR_H
#define MONITOR_H

#include "common.h"

// Write a single character out to the screen.
void monitor_put(char c);

// Clear the screen to all black.
void monitor_clear();

// Output a null-terminated ASCII string to the monitor.
void monitor_write(char *c);

#endif // MONITOR_H
```

##### 3.2.2.1.移动光标

为了移动硬件的光标， 我们必须先计算出xy光标坐标的线性偏移量，我们可以用上文提到的公式来计算。接着，我们把这个偏移量发送给VGA控制器。出于某种原因，它需要一个字节一个字节地接受。我们先向控制器地命令端口`(0x3D4)`发送`14`，告诉它我们先发送高8位的一个字节，然后把那个字节发送到端口`(0x3D5)`。低字节也一样，不过向命令端口发送的是`15`。

```c
// Updates the hardware cursor.
static void move_cursor()
{
   // The screen is 80 characters wide...
   u16int cursorLocation = cursor_y * 80 + cursor_x;
   outb(0x3D4, 14);                  // Tell the VGA board we are setting the high cursor byte.
   outb(0x3D5, cursorLocation >> 8); // Send the high cursor byte.
   outb(0x3D4, 15);                  // Tell the VGA board we are setting the low cursor byte.
   outb(0x3D5, cursorLocation);      // Send the low cursor byte.
}
```

##### 3.2.2.2.屏幕滚动

总有屏幕被代码占满的时候。如果我们能让屏幕像终端一样能一行行滚动就太棒了。事实上，做到这一点并不难：

```c
// Scrolls the text on the screen up by one line.
static void scroll()
{

   // Get a space character with the default colour attributes.
   u8int attributeByte = (0 /*black*/ << 4) | (15 /*white*/ & 0x0F);
   u16int blank = 0x20 /* space */ | (attributeByte << 8);

   // Row 25 is the end, this means we need to scroll up
   if(cursor_y >= 25)
   {
       // Move the current text chunk that makes up the screen
       // back in the buffer by a line
       int i;
       for (i = 0*80; i < 24*80; i++)
       {
           video_memory[i] = video_memory[i+80];
       }

       // The last line should now be blank. Do this by writing
       // 80 spaces to it.
       for (i = 24*80; i < 25*80; i++)
       {
           video_memory[i] = blank;
       }
       // The cursor should now be on the last line.
       cursor_y = 24;
   }
}
```

##### 3.2.2.3.向屏幕写一个字符

现在代码开始变得有一点点复杂了。但是你如果仔细观察的话，你会发现大多数都是在实现‘接下来把光标放哪儿’这一问题——其实并不难。

```c
// Writes a single character out to the screen.
void monitor_put(char c)
{
   // The background colour is black (0), the foreground is white (15).
   u8int backColour = 0;
   u8int foreColour = 15;

   // The attribute byte is made up of two nibbles - the lower being the
   // foreground colour, and the upper the background colour.
   u8int  attributeByte = (backColour << 4) | (foreColour & 0x0F);
   // The attribute byte is the top 8 bits of the word we have to send to the
   // VGA board.
   u16int attribute = attributeByte << 8;
   u16int *location;

   // Handle a backspace, by moving the cursor back one space
   if (c == 0x08 && cursor_x)
   {
       cursor_x--;
   }

   // Handle a tab by increasing the cursor's X, but only to a point
   // where it is divisible by 8.
   else if (c == 0x09)
   {
       cursor_x = (cursor_x+8) & ~(8-1);
   }

   // Handle carriage return
   else if (c == '\r')
   {
       cursor_x = 0;
   }

   // Handle newline by moving cursor back to left and increasing the row
   else if (c == '\n')
   {
       cursor_x = 0;
       cursor_y++;
   }
   // Handle any other printable character.
   else if(c >= ' ')
   {
       location = video_memory + (cursor_y*80 + cursor_x);
       *location = c | attribute;
       cursor_x++;
   }

   // Check if we need to insert a new line because we have reached the end
   // of the screen.
   if (cursor_x >= 80)
   {
       cursor_x = 0;
       cursor_y ++;
   }

   // Scroll the screen if needed.
   scroll();
   // Move the hardware cursor.
   move_cursor();
}
```

看到了没？简单极了！真正负责写字符的代码是这两行：

```c
location = video_memory + (cursor_y*80 + cursor_x);
*location = c | attribute;
```

-  设置`location`指针指向与当前光标位置对应的字的线性地址(参见上面的公式)。 
-  将`location`指向的值设置为待写的字符和`attribute`的逻辑或值。还记得我们把'attribute'左移了8位，所以实际上我们只是把`c`放在`attribute`的低八位字节。 

##### 3.2.2.4.清屏

清屏的实现简单死了。只需要填满空格就行：

```c
// Clears the screen, by copying lots of spaces to the framebuffer.
void monitor_clear()
{
   // Make an attribute byte for the default colours
   u8int attributeByte = (0 /*black*/ << 4) | (15 /*white*/ & 0x0F);
   u16int blank = 0x20 /* space */ | (attributeByte << 8);

   int i;
   for (i = 0; i < 80*25; i++)
   {
       video_memory[i] = blank;
   }

   // Move the hardware cursor back to the start.
   cursor_x = 0;
   cursor_y = 0;
   move_cursor();
}
```

##### 3.2.2.5.写一个字符串

```c
// Outputs a null-terminated ASCII string to the monitor.
void monitor_write(char *c)
{
   int i = 0;
   while (c[i])
   {
       monitor_put(c[i++]);
   }
}
```

#### 3.3.总结

如果你写好了以上代码，你就可以在`main()`函数里面添加几行代码：

```c
monitor_clear();
monitor_write("Hello, world!");
```

好了——一个文本输出函数！写了十几分钟的代码是值得的，不是吗？

#### 3.4.拓展

 除了实现memcpy/memset/strlen/strcmp等功能外，还有一些其他的功能可以让你写起代码来更加轻松。

```c
void monitor_write_hex(u32int n)
{
   // TODO: implement this yourself!
}

void monitor_write_dec(u32int n)
{
   // TODO: implement this yourself!
}
```

函数名应该很好理解—— 如果要检查指针的有效性，确实需要使用十六进制。十进制是可选的，但偶尔看到以10为基数的东西也不错! 

![the_screen_screenshot](png\the_screen_screenshot.png)

你还可以看看linux0.1版本的代码——里面有一个`vsprintf`的实现，非常的整洁。你可以借鉴那个函数，来完成`printf()`函数，这会让你的代码调试起来更加容易。

这一章教程的源代码在[这儿](http://www.jamesmolloy.co.uk/tutorial_html/the_screen.tar.gz)可以下载。



## 第四部分 GDT 和 IDT

GDT和IDT都是描述符表。它们是由标志位和bit值组成的数组，用来描述段式系统的操作（GDT）或者中断向量表（IDT）。

但不幸的是，它们都是偏理论的东西，但熬过这一段就会驾轻就熟了！

### 4.1.全局描述符表（理论部分）

x86架构提供了两种内存保护方法和虚拟内存方法——段式内存和页式内存。

通过段式内存管理的方法，每一次内存访问都要根据段地址求值。也就是说，内存的偏移地址加上段地址，再根据段的长度进行检查。你可以把段当成是进入一片内存空间的窗户——但是进程是不知道它是‘窗户’的，它只会‘看’到一个从零开始一直增长到段长度的线性地址空间。

通过页式内存管理的方法，进程的逻辑地址空间被分成很多页（通常是4KB每页，但也可变更）。每一页都能映射到物理内存中——被映射的物理内存块被称为‘页框’。它也可以被取消映射。通过这种方式就能实现虚拟内存的创建和回收。

这两种方法各有好处，但页式管理更好。段式管理方法虽然也能用，但作为内存保护和虚拟内存的方法来看已经过时了。实际上，x86-64架构要求以平展存储模式 （一个基址为0，不超过0xFFFFFFFF的段）来正确执行它的指令集。

 但是，分段基本上是完全内置在x86体系结构中的，这基本上是绕不过的。所以在这里我将带你构建你自己的`GDT`表。

就像上面提到的那样，我们打算尝试构建一个平展的存储模式。段的‘窗户’应该开始于0x00000000，延伸到0xFFFFFFFF（内存的尽头）。 但是，有一件事是分段可以做而分页不能做到的，那就是设置环的级别。 

环指的是一个特权级别——R0是最高级别，R3是最低级别。运行在R0级别的程序也叫运行在内核态或者管理态，因为它们能执行像sti和cli一类大多数程序不能使用的指令。通常，环1和环2是不使用的。从技术上讲，它们可以访问比环3更大的管理态指令的子集。一些微内核架构使用它们来运行服务进程或驱动程序。

段描述符内置了一个表示它所应用的环的级别的数字。 为了改变环的级别(我们稍后会做)，我们需要同时表示环0和环3的段。

### 4.1.全局描述符表（实践部分）

好的，这真是好大一块理论知识，让我们进入具体实施的环节。

忘提的一件事是GRUB已经为你设置好了一个GDT。 但我们并不知道它在哪儿。所以你如果不小心把它覆盖掉了，你的电脑可能就会多次出错又重启。不太明智。

在x86架构中，我们有6个段寄存器。每一个保存了GDT的偏移量。它们分别是`cs`（代码段寄存器），`ds`（数据段寄存器），`es`（附加段寄存器），`fs`，`gs`，`ss`（栈段寄存器）。 代码段寄存器必须引用设置为“代码段”的描述符。为此在访问字节里面设置有一个标志位。 其余的寄存器都应该引用一个被设置为“数据段”的描述符。 

#### 4.2.1.descriptor_tables.h

描述GDT表项的结构体长这样：

```c
// This structure contains the value of one GDT entry.
// We use the attribute 'packed' to tell GCC not to change
// any of the alignment in the structure.
struct gdt_entry_struct
{
   u16int limit_low;           // The lower 16 bits of the limit.
   u16int base_low;            // The lower 16 bits of the base.
   u8int  base_middle;         // The next 8 bits of the base.
   u8int  access;              // Access flags, determine what ring this segment can be used in.
   u8int  granularity;
   u8int  base_high;           // The last 8 bits of the base.
} __attribute__((packed));
typedef struct gdt_entry_struct gdt_entry_t;
```

![gdt_idt_gdt_format_2](png\gdt_idt_gdt_format_2.png)

这些字段中大多数都是不言自明的。访问字节的格式在上面的图中给出，粒度字节的格式在下面给出：

![gdt_idt_gdt_format_1](png\gdt_idt_gdt_format_1.png)

**P**

​	段是否存在？（存在为1）

**DPL**

​	 描述符特权级别——0~3环。 

**DT**

​	描述符类型

**type**

​	段类型——代码段/数据段

**G**

​	  粒度（0=1字节，1=1k字节）

**D**

​	 操作数大小(0 = 16bit, 1 = 32bit) 

**0**

​	恒为0

**A**

​	系统可用（恒为零）

为了告诉处理器去哪儿找GDT，我们必须提供它一个特殊的指针结构的地址：

```C
struct gdt_ptr_struct
{
   u16int limit;               // The upper 16 bits of all selector limits.
   u32int base;                // The address of the first gdt_entry_t struct.
}
 __attribute__((packed));
typedef struct gdt_ptr_struct gdt_ptr_t;
```

`base`是GDT表的首地址，`limit`为表的大小-1（表中最后一个可用地址）。

这些结构定义应该放在头文件`descriptor_tables.h`中，还应该附带一个函数原型的声明。

```C
// Initialisation function is publicly accessible.
void init_descriptor_tables();
```

#### 4.2.2. descriptor_tables.c

在文件`descriptor_tables.c`中，有一些声明如下：

```C
//
// descriptor_tables.c - Initialises the GDT and IDT, and defines the
// default ISR and IRQ handler.
// Based on code from Bran's kernel development tutorials.
// Rewritten for JamesM's kernel development tutorials.
//

#include "common.h"
#include "descriptor_tables.h"

// Lets us access our ASM functions from our C code.
extern void gdt_flush(u32int);

// Internal function prototypes.
static void init_gdt();
static void gdt_set_gate(s32int,u32int,u32int,u8int,u8int);

gdt_entry_t gdt_entries[5];
gdt_ptr_t   gdt_ptr;
idt_entry_t idt_entries[256];
idt_ptr_t   idt_ptr;
```

注意gdt_flush函数——该函数将在asm文件中定义，并且可以加载我们的GDT指针。

```C
// Initialisation routine - zeroes all the interrupt service routines,
// initialises the GDT and IDT.
void init_descriptor_tables()
{
   // Initialise the global descriptor table.
   init_gdt();
}

static void init_gdt()
{
   gdt_ptr.limit = (sizeof(gdt_entry_t) * 5) - 1;
   gdt_ptr.base  = (u32int)&gdt_entries;

   gdt_set_gate(0, 0, 0, 0, 0);                // Null segment
   gdt_set_gate(1, 0, 0xFFFFFFFF, 0x9A, 0xCF); // Code segment
   gdt_set_gate(2, 0, 0xFFFFFFFF, 0x92, 0xCF); // Data segment
   gdt_set_gate(3, 0, 0xFFFFFFFF, 0xFA, 0xCF); // User mode code segment
   gdt_set_gate(4, 0, 0xFFFFFFFF, 0xF2, 0xCF); // User mode data segment

   gdt_flush((u32int)&gdt_ptr);
}

// Set the value of one GDT entry.
static void gdt_set_gate(s32int num, u32int base, u32int limit, u8int access, u8int gran)
{
   gdt_entries[num].base_low    = (base & 0xFFFF);
   gdt_entries[num].base_middle = (base >> 16) & 0xFF;
   gdt_entries[num].base_high   = (base >> 24) & 0xFF;

   gdt_entries[num].limit_low   = (limit & 0xFFFF);
   gdt_entries[num].granularity = (limit >> 16) & 0x0F;

   gdt_entries[num].granularity |= gran & 0xF0;
   gdt_entries[num].access      = access;
}
```

让我们来分析一下这个代码。首先`init_gdt`初始化gdt指针结构—— `limit`大小为每个gdt表项的大小* 5 ——我们有5个表项。 为啥是5个鸭？因为我们不仅有内核模式下的代码段、数据段描述符，还有用户模式下的代码段数据段描述符，以及一个表项，这个不能少，少了就会出大问题。

通过调用`gdt_set_gate`，`gdt_init`函数设置好了5个描述符。`gdt_set_get`其实只是做一些重要的循环移位（译者ps：关于什么是循环移位（bit-twiddling）可以看[这篇文章]( https://lfwen.site/2017/02/08/bit-hacking/ )）和移位，这从代码就能看出来。注意到4个段描述符之间，唯一变了的是`access`位——0x9A，0x92，0xFA，0xF2。 可以看出，如果将这些位映射出来并与上面的格式图进行比较，其实变了的位是`type`和`DPL`字段。 `DPL`用来表示描述符的特权等级——3为用户态，0为内核态。 `Type`指定该段是代码段还是数据段(处理器会经常检查，这可能会导致很多问题)。 

最后，我们写一个汇编函数来写入GDT指针：

```ASM
[GLOBAL gdt_flush]    ; 允许C代码调用

gdt_flush:
   mov eax, [esp+4]  ; 获取指向GDT的指针，作为参数传递
   lgdt [eax]        ; 装载新的GDT指针

   mov ax, 0x10      ; 0x10 在GDT中是数据段的偏移
   mov ds, ax        ; 加载所有数据段选择器
   mov es, ax
   mov fs, ax
   mov gs, ax
   mov ss, ax
   jmp 0x08:.flush   ; 0x08 是代码段的偏移：远转移
.flush:
   ret
```

该函数接受传递给它的第一个参数（在esp+4的位置），使用指令`LGDT`装载指向GDT的指针，然后为代码段和数据段加载段选择子。注意到`GDT`表中每个表项为8字节，内核代码段的描述符为第二个，所以偏移值是0x08。 同样地，内核数据段描述符是第三个，所以它的偏移量是16 = 0x10。在这里我们把值0x10传送到数据段寄存器ds，es，fd，gs，ss。 改变代码段则略有不同，我们执行了一个远转移，这隐式地改变了CS寄存器的值。

### 4.3.中断描述符表（理论部分）

有时你想要暂时中断处理器。你想让它暂停正在处理的操作，强制执行一些别的操作。 例如，当计时器或键盘中断请求(IRQ)触发时。中断就像POSIX信号一样——它告诉你发生了一些有趣的事情。处理器可以注册处理中断的“信号处理程序”(中断处理程序) ，然后再返回到中断触发前的代码。中断可以在外部通过`IRQ`触发，也可以在内部通过`int n`指令触发。对软件来说，触发中断是很有用的，在后面的章节会提到。

中断向量表就是用来告诉处理器去哪儿找中断处理程序的。跟GDT很类似，它不过是很多表项的数组，每一个对应一个中断号。有256种可能的中断号，所以必须定义256个。如果一个中断产生了在中断向量表中却没有相应的处理程序（哪怕一个空表项也好），处理器就会出大问题然后重启。

#### 4.3.1.错误，陷阱和异常

处理器有时需要向内核发出信号。可能主要是发生了一些比如除数为0，或者出现了页面错误的事情。为了处理这些信号，通常我们使用前32个中断号。因此，同时保证它们被正确地映射并且非空是很重要的。——否则CPU会多次出错然后重启（bochs会抛出一个‘未处理异常’的错误）。

 cpu专用的特殊中断如下所示。 

- 0 - 除零异常
- 1 - 调试异常
- 2 - 不可屏蔽中断
- 3 - 断点异常
- 4 - '检测到溢出'
- 5 - 越界异常
- 6 - 操作码无效异常
- 7 - 无协处理器异常
- 8 - 双重故障（推送错误代码） 
- 9 - 协处理器段超出
- 10 - 错误的TSS(推送错误代码)
- 11 - 段不存在(推送错误代码)
- 12 - 堆栈错误(推送错误代码) 
- 13 - 一般保护故障(推送错误代码)
- 14 - 页面错误(推送错误代码)
- 15 - 未知异常中断
- 16 - 协处理器错误
- 17 - 对齐检查异常
- 18 - 机器检查异常
- 19-31 - 保留

### 4.4.中断描述符表（实践部分）

#### 4.4.1.descriptor_tables.h

我们在`descriptor_tables.h`中添加一些定义：

```C
// A struct describing an interrupt gate.
struct idt_entry_struct
{
   u16int base_lo;             // The lower 16 bits of the address to jump to when this interrupt fires.
   u16int sel;                 // Kernel segment selector.
   u8int  always0;             // This must always be zero.
   u8int  flags;               // More flags. See documentation.
   u16int base_hi;             // The upper 16 bits of the address to jump to.
} __attribute__((packed));
typedef struct idt_entry_struct idt_entry_t;

// A struct describing a pointer to an array of interrupt handlers.
// This is in a format suitable for giving to 'lidt'.
struct idt_ptr_struct
{
   u16int limit;
   u32int base;                // The address of the first element in our idt_entry_t array.
} __attribute__((packed));
typedef struct idt_ptr_struct idt_ptr_t;

// These extern directives let us access the addresses of our ASM ISR handlers.
extern void isr0 ();
...
extern void isr31();
```

看见了吗？跟GDT表项和指针结构体非常像。flag字段如下图所示。低5个字节应该恒为`0b0110`——十进制的14。DPL表示期望被调用的特权等级——在我们的实例中是0，但随着我们编程的进展，我们需要把它设置为3。P标志位代表该表项存在。 如果任意描述符这个位为空都将导致“中断未处理”异常。 

![gdt_idt_idt_format_1](png\gdt_idt_idt_format_1.png)

#### 4.4.2.descriptor_tables.c

我们需要修改这个文件来添加一些新的代码：

```C
...
extern void idt_flush(u32int);
...
static void init_idt();
static void idt_set_gate(u8int,u32int,u16int,u8int);
...
idt_entry_t idt_entries[256];
idt_ptr_t   idt_ptr;
...
void init_descriptor_tables()
{
  init_gdt();
  init_idt();
}
...
static void init_idt()
{
   idt_ptr.limit = sizeof(idt_entry_t) * 256 -1;
   idt_ptr.base  = (u32int)&idt_entries;

   memset(&idt_entries, 0, sizeof(idt_entry_t)*256);

   idt_set_gate( 0, (u32int)isr0 , 0x08, 0x8E);
   idt_set_gate( 1, (u32int)isr1 , 0x08, 0x8E);
   ...
   idt_set_gate(31, (u32int)isr32, 0x08, 0x8E);

   idt_flush((u32int)&idt_ptr);
}

static void idt_set_gate(u8int num, u32int base, u16int sel, u8int flags)
{
   idt_entries[num].base_lo = base & 0xFFFF;
   idt_entries[num].base_hi = (base >> 16) & 0xFFFF;

   idt_entries[num].sel     = sel;
   idt_entries[num].always0 = 0;
   // We must uncomment the OR below when we get to using user-mode.
   // It sets the interrupt gate's privilege level to 3.
   idt_entries[num].flags   = flags /* | 0x60 */;
}
```

以下的加入到`gdt.s`：

```C
[GLOBAL idt_flush]    ; Allows the C code to call idt_flush().

idt_flush:
   mov eax, [esp+4]  ; Get the pointer to the IDT, passed as a parameter.
   lidt [eax]        ; Load the IDT pointer.
   ret
```

#### 4.4.3. interrupt.s

好的！我们现在有代码可以告诉CPU在哪里去寻找我们的中断处理程序了——但我们还没有写。

当处理器接收到一个中断，它将必要的寄存器的内容(指令指针、栈指针、代码和数据段、标志寄存器)保存到栈中。接着它从IDT表中找到中断处理程序的位置然后跳转过去。

现在，就像POSIX信号处理程序一样，当中断处理程序运行时，你不会接收到任何关于触发的中断的信息。所以，不幸的是，我们不仅仅要写一个处理器程序，我们要对每个需要处理的中断都编写相应的处理程序。这有点子烦，因为我们想把代码的重复率降低到最小。我们通过编写许多处理程序来实现这一点，这些处理程序将中断号(在汇编文件中硬编码)压栈，并调用一个公共处理程序函数。

看起来很简单，但不幸的是，我们还有另一个问题——一些中断还会将错误代码压栈。我们不能在不同的堆栈结构下调用相同的函数，所以对于那些没有把错误代码压栈的，我们会压一个假代码，以保证堆栈是一样的。

```ASM
[GLOBAL isr0]
isr0:
  cli                 ; Disable interrupts
  push byte 0         ; Push a dummy error code (if ISR0 doesn't push it's own error code)
  push byte 0         ; Push the interrupt number (0)
  jmp isr_common_stub ; Go to our common handler.
```

这个实例能跑起来，但是把它写32个不同的版本看起来还是有很多代码。我们可以用NASM的宏指令减少代码量，像这样：

```asm
%macro ISR_NOERRCODE 1  ; define a macro, taking one parameter
  [GLOBAL isr%1]        ; %1 accesses the first parameter.
  isr%1:
    cli
    push byte 0
    push byte %1
    jmp isr_common_stub
%endmacro

%macro ISR_ERRCODE 1
  [GLOBAL isr%1]
  isr%1:
    cli
    push byte %1
    jmp isr_common_stub
%endmacro
```

现在我们可以通过这样做来创建一个存根函数：

```
ISR_NOERRCODE 0
ISR_NOERRCODE 1
...
```

减少工作量的事情很值得我们去做。快速浏览一下intel手册就会知道只有8号中断、10-14号中断会把错误码压栈。其余的就用我们的假错误码。

就快好了，我保证！

剩下只有两件事情要做——一个是创建一个汇编处理函数，另一个是写一个更高级的C语言处理函数。

```asm
; In isr.c
[EXTERN isr_handler]

; This is our common ISR stub. It saves the processor state, sets
; up for kernel mode segments, calls the C-level fault handler,
; and finally restores the stack frame.
isr_common_stub:
   pusha                    ; Pushes edi,esi,ebp,esp,ebx,edx,ecx,eax

   mov ax, ds               ; Lower 16-bits of eax = ds.
   push eax                 ; save the data segment descriptor

   mov ax, 0x10  ; load the kernel data segment descriptor
   mov ds, ax
   mov es, ax
   mov fs, ax
   mov gs, ax

   call isr_handler

   pop eax        ; reload the original data segment descriptor
   mov ds, ax
   mov es, ax
   mov fs, ax
   mov gs, ax

   popa                     ; Pops edi,esi,ebp...
   add esp, 8     ; Cleans up the pushed error code and pushed ISR number
   sti
   iret           ; pops 5 things at once: CS, EIP, EFLAGS, SS, and ESP
```

这段代码是我们常用的中断处理程序。首先用`pusha`指令把通用寄存器压栈。最后用`popa`指令恢复它们。他还获取当前段选择器进行压栈，把所有的数据段寄存器用作内核数据段选择器，并在之后恢复它们。现在暂时不会有实际的效果，但是当我们切换到用户态时它会有效果。注意，它还调用了一个更高级的中断处理程序——`isr_handler`。

当中断触发的时候，处理器自动地将处理器状态信息压栈。 代码段寄存器、指令指针寄存器、标志寄存器、栈段寄存器和栈指针寄存器被压进栈里。`IRET`指令 是专门为中断返回而设计的。它将这些值从堆栈中弹出，并将处理器返回到它原来的状态。

#### 4.4.4.isr.c

```c
//
// isr.c -- High level interrupt service routines and interrupt request handlers.
// Part of this code is modified from Bran's kernel development tutorials.
// Rewritten for JamesM's kernel development tutorials.
//

#include "common.h"
#include "isr.h"
#include "monitor.h"

// This gets called from our ASM interrupt handler stub.
void isr_handler(registers_t regs)
{
   monitor_write("recieved interrupt: ");
   monitor_write_dec(regs.int_no);
   monitor_put('\n');
}
```

没啥可以解释的——中断处理程序向屏幕打印了一个带中断号的信息。它用了一个结构体`registers_t`，代表所有压栈的寄存器，在`isr.h`中定义。

#### 4.4.5.isr.h

```c
//
// isr.h -- Interface and structures for high level interrupt service routines.
// Part of this code is modified from Bran's kernel development tutorials.
// Rewritten for JamesM's kernel development tutorials.
//

#include "common.h"

typedef struct registers
{
   u32int ds;                  // Data segment selector
   u32int edi, esi, ebp, esp, ebx, edx, ecx, eax; // Pushed by pusha.
   u32int int_no, err_code;    // Interrupt number and error code (if applicable)
   u32int eip, cs, eflags, useresp, ss; // Pushed by the processor automatically.
} registers_t;
```

#### 4.4.6.测试

wow，这真是好长的一个章节！不要退缩，不是所有章都这么长的。一分耕耘一分收获。

现在我们可以进行测试了！把这些添加到`main()`函数里：（译者ps：除了这两句，别忘了加上`init_descriptor_tables()`函数，否则会出大问题）

```c
asm volatile ("int $0x3");
asm volatile ("int $0x4");
```

![gdt_idt_bochs](png\gdt_idt_bochs.png)

在上图中可以看到，产生了两个软件中断：3号中断和4号中断。（译者ps：这里可能是作者截图截错了，正确的应该显示`recieved interrupt:0x3`和`recieved interrupt:0x4`才对，QAQ）

恭喜！你现在有一个可以产生中断的内核了，并且设置好了自己的段表。

实例代码可以在[这里](http://www.jamesmolloy.co.uk/tutorial_html/gdt_idt.tar.gz)下载。



## 第五部分 IRQs 和 PIT

在这一章节我们将要学习中断请求（IRQs）以及可编程间隔定时器（PIT）。

### 5.1.中断请求（理论部分）

有很多种方式可以实现与外部设备的通信。其中最有用也是最常用的两种方法是轮循和中断。

**轮循**

​	CPU定时发出询问，轮次检查是否有设备处于就绪态，并循环这个过程。

**中断**

​	可以做很多有用的事情。当设备处于就绪状态的时候，触发一个CPU中断，并运行相应的中断处理程序。

从我带有主观倾向性的描述中你可以感受到，中断在很多情况下更好使。轮询有很多用途——有的cpu可能没有中断机制，或者你可能有很多设备，抑或是你不需要太频繁地进行检查，因此没必要使用中断。 不管怎样，中断都是一种非常有用的硬件通信方式。当有按键操作时，键盘会使用中断，可编程间隔计时器(PIT)也会使用中断。

外部中断背后的低级概念并不是很复杂。所有有中断能力的设备都被连接到PIC（可编程中断控制器）。PIC是唯一与CPU中断引脚直接连接的设备。它被用作多路复用器，并且能在中断设备间划分优先级。它本质上是一个8选1多路复用器。某时某地某人突然觉得8条IRQ线不够用，于是通过菊花链的拓扑方式在原来的多路复用器上又连接了一个。于是在现代计算机中，有2个PICs，主和从，可以服务15个带中断功能的设备（还有一根线用来接收从PIC的信号）。

![pics](png\pics.png)

PIC的另一个设计得很聪明的地方是，你可以改变传递给每一个IRQ线的中断号。通常称之为PIC的重映射，这很有用。当计算机启动的时候，默认的中断映射为：

- IRQ 0..7 - INT 0x8..0xF
- IRQ 8..15 - INT 0x70..0x77

这给我们带来了一些麻烦。主IRQ的映射（0x8-0xF）与CPU用来表示异常和错误的中断号冲突了。正常来说我们只需要把PICs重新定位，让IRQs 的0-15映射到ISR的32-47（31是最后一个被CPU使用的ISR（中断服务程序））。

### 5.2.中断请求（实践部分）

PICs通过I/O总线通信，每个PIC都含有一个命令端口和一个数据端口:

- 主PIC——命令端口：0x20，数据端口：0x21
- 从PIC——命令端口：0xA0，数据端口：0xA1

对PICs进行重映射的代码时最难也是最容易混淆的部分。为了实现重映射，你必须对它们进行完全的初始化，这就是为什么代码会很长的原因。如果你感兴趣发生了什么，点击[这里](http://www.osdev.org/wiki/PIC)有一个很好的讲解。

```C
static void init_idt()
{
  ...
  // Remap the irq table.
  outb(0x20, 0x11);
  outb(0xA0, 0x11);
  outb(0x21, 0x20);
  outb(0xA1, 0x28);
  outb(0x21, 0x04);
  outb(0xA1, 0x02);
  outb(0x21, 0x01);
  outb(0xA1, 0x01);
  outb(0x21, 0x0);
  outb(0xA1, 0x0);
  ...
  idt_set_gate(32, (u32int)irq0, 0x08, 0x8E);
  ...
  idt_set_gate(47, (u32int)irq15, 0x08, 0x8E);
}
```

注意到我们也为中断请求处理程序设置了32-47中断号。因此在`interrupt.s`文件中我们也要加上这些存根。所以，我们需要定义一个新的宏，——一个与两个数相关的IRQ stub——IRQ号和中断号。

```asm
; This macro creates a stub for an IRQ - the first parameter is
; the IRQ number, the second is the ISR number it is remapped to.
%macro IRQ 2
  global irq%1
  irq%1:
    cli
    push byte 0
    push byte %2
    jmp irq_common_stub
%endmacro
 
...

IRQ   0,    32
IRQ   1,    33
...
IRQ  15,    47
```

我们同时有一个新的stub——`irq_common_stub`。这是因为IRQ的行为略有不同——在从IRQ处理程序返回之前，必须先通知PIC已经完成，这样它才可以分派下一个(如果有一个处于等待状态)。这又叫EOI(中断结束)，有一点点小复杂。如果主PIC发送了中断请求（0-7号），我们显然也应该发送中断结束信号给PIC。如果是从PIC发送的中断请求（8-15号），那么两个PIC都要收到EOI信号（因为它们之间有菊花链的拓扑结构）。

 首先是汇编文件里的公共存根函数，它几乎与isr_common_stub相同。 

```asm
; In isr.c
[EXTERN irq_handler]

; This is our common IRQ stub. It saves the processor state, sets
; up for kernel mode segments, calls the C-level fault handler,
; and finally restores the stack frame.
irq_common_stub:
   pusha                    ; Pushes edi,esi,ebp,esp,ebx,edx,ecx,eax

   mov ax, ds               ; Lower 16-bits of eax = ds.
   push eax                 ; save the data segment descriptor

   mov ax, 0x10  ; load the kernel data segment descriptor
   mov ds, ax
   mov es, ax
   mov fs, ax
   mov gs, ax

   call irq_handler

   pop ebx        ; reload the original data segment descriptor
   mov ds, bx
   mov es, bx
   mov fs, bx
   mov gs, bx

   popa                     ; Pops edi,esi,ebp...
   add esp, 8     ; Cleans up the pushed error code and pushed ISR number
   sti
   iret           ; pops 5 things at once: CS, EIP, EFLAGS, SS, and ESP
```

接着是C代码（`isr.c`文件）

```c
// This gets called from our ASM interrupt handler stub.
void irq_handler(registers_t regs)
{
   // Send an EOI (end of interrupt) signal to the PICs.
   // If this interrupt involved the slave.
   if (regs.int_no >= 40)
   {
       // Send reset signal to slave.
       outb(0xA0, 0x20);
   }
   // Send reset signal to master. (As well as slave, if necessary).
   outb(0x20, 0x20);

   if (interrupt_handlers[regs.int_no] != 0)
   {
       isr_t handler = interrupt_handlers[regs.int_no];
       handler(regs);
   }
}
```

这很好理解——如果中断请求号大于7（中断号大于40），偶们==我们就同时向主PIC和从PIC的命令端口发送信号。

你可能还注意到，我添加了一个小的自定义处理程序机制，允许你注册自定义中断处理程序。作为一个概念性的技术这很有用，并且能够很好地整合我们的代码。

后面还有一些其他的申明定义。

#### 5.2.1.isr.h

```c
// A few defines to make life a little easier
#define IRQ0 32
...
#define IRQ15 47

// Enables registration of callbacks for interrupts or IRQs.
// For IRQs, to ease confusion, use the #defines above as the
// first parameter.
typedef void (*isr_t)(registers_t);
void register_interrupt_handler(u8int n, isr_t handler);
```

#### 5.2.2.isr.c

```c
isr_t interrupt_handlers[256];

void register_interrupt_handler(u8int n, isr_t handler)
{
  interrupt_handlers[n] = handler;
}
```

好了!我们现在可以处理来自外部设备的中断请求，并将它们分派给自定义处理程序。现在我们只差一些中断请求来处理了!

### 5.3.可编程间隔定时器（理论部分）

可编程间隔定时器（PIT）是一个连接到IRQ0号管脚的芯片。它可以以用户定义的频率（182Hz~1.1931MHz）向CPU发出中断请求。PIT是实现系统时钟的主要方法，也是实现多任务(在中断时切换进程)的唯一方法。

PIT有一个内部时钟，振荡频率约为1.1931MHz。这个时钟信号通过一个[分频器](http://en.wikipedia.org/wiki/Frequency_divider)输入，以调节最终的输出频率。它有3个通道，每个通道都有自己的分频器。

- 通道0是最有用的，它的输出与IRQ0号管脚相连接。
- 1号通道没啥用，在现代硬件中不再被实现。它被用来控制DRAM的刷新率。
- 2号通道用来控制计算机的扬声器。

目前我们只用1号通道。

好的，我们接下来设置PIT让它以特定的间隔（频率f）发出中断信号。我一般把f设置为100Hz（每10ms），但你也可随意。为了做到这一点，我们向PIT发送一个‘除数’。这个数应该等于输入的频率（1.9131MHz）除以我们需要的频率。这很容易做到：
$$
divisor = 1193180 Hz / frequency (in Hz)
$$
同样值得注意的是，PIT在I/O空间中有4个寄存器——0x40-0x42分别是通道0-2的数据端口，0x43是命令端口。

### 5.4.可编程间隔定时器（实践部分）

我们需要新建几个文件。`timer.h`内只有一个声明：

```C
// timer.h -- Defines the interface for all PIT-related functions.
// Written for JamesM's kernel development tutorials.

#ifndef TIMER_H
#define TIMER_H

#include "common.h"

void init_timer(u32int frequency);

#endif
```

`timer.c`中也没有多少：

```C
// timer.c -- Initialises the PIT, and handles clock updates.
// Written for JamesM's kernel development tutorials.

#include "timer.h"
#include "isr.h"
#include "monitor.h"

u32int tick = 0;

static void timer_callback(registers_t regs)
{
   tick++;
   monitor_write("Tick: ");
   monitor_write_dec(tick);
   monitor_write("\n");
}

void init_timer(u32int frequency)
{
   // Firstly, register our timer callback.
   register_interrupt_handler(IRQ0, &timer_callback);

   // The value we send to the PIT is the value to divide it's input clock
   // (1193180 Hz) by, to get our required frequency. Important to note is
   // that the divisor must be small enough to fit into 16-bits.
   u32int divisor = 1193180 / frequency;

   // Send the command byte.
   outb(0x43, 0x36);

   // Divisor has to be sent byte-wise, so split here into upper/lower bytes.
   u8int l = (u8int)(divisor & 0xFF);
   u8int h = (u8int)( (divisor>>8) & 0xFF );

   // Send the frequency divisor.
   outb(0x40, l);
   outb(0x40, h);
}
```

好的，我们来看看这个代码。首先，我们有`init_timer`函数。它告诉我们的中断处理机通过函数`timer_callback`来处理IRQ0。无论何时当我们接收到时钟中断的时候，这个函数将被调用。接着我们计算出除数，发送到PIT（详见理论部分）。接着，我们向PIT的命令端口发送一个字节。这个字节（0x36）把PIT设置为重复模式（当除数计数器为0时，它自动刷新）并且告诉它我们要设置除数值。

接着我们向数据端口发送这个值。注意我们要发送两次，先发送低8位的字节，再发送高8位的字节。

做完这些后，我们要做的就是修改`Makefile`，在`main.c`中添加这一行：

```c
init_timer(50); // Initialise timer to 50Hz
```

![irqs_and_the_pit_bochs](png\irqs_and_the_pit_bochs.png)

编译，运行！你应该得到和右图一样的输出。但是请注意，bochs并没有准确地模拟计时器芯片，因此，尽管您的代码在真机上能以正确的速度运行，但在bochs中可能不会!

这一章节的完整源码从[这里](http://www.jamesmolloy.co.uk/tutorial_html/irqs_and_the_pit.tar.gz)下载。



## 第六部分 分页

在这一部分我们要实现分页管理。分页有双重用途——内存保护和虚拟内存(这两者密不可分)。

### 6.1.虚拟内存（理论部分）

*如果你已经提前理解虚拟内存的原理，你可以跳过这一节*

在linux里，如果你写一个像这样的小程序：

```c
int main(char argc, char **argv)
{
  return 0;
}
```

编译它，然后输入`objdump -f`命令，你可能会看到这样的输出：

```bash
jamesmol@aubergine:~/test> objdump -f a.out

a.out:     file format elf32-i386
architecture: i386, flags 0x00000112:
EXEC_P, HAS_SYMS, D_PAGED
start address 0x080482a0
```

注意到程序的开始地址为`0x080482a0`，大约在地址空间的128MB的位置。但这个程序却可以在RAM小于128MB的机器上运行，这可能听起来很奇怪。

当程序读写内存的时候，它实际上‘看到’的，是一个虚拟地址空间。这些虚拟空间的一部分被映射到物理内存，另外的部分却没有映射。如果您试图访问一个未映射的页面，处理器将引发一个页面错误，操作系统将捕获它，而在POSIX系统中，系统会传递一个SIGSEGV信号，紧随它的是一个SIGKILL信号。

这个抽象概念是很有用的。这意味着编译器可以产生一个程序，它依赖于每次运行时代码在内存中的确切位置。通过虚拟内存，进程认为自己处于`0x08048a20`处，实际上它可能在物理内存地址的`0x1000000`处。不仅如此，进程还不能意外或者有意地越权访问其他进程的代码或者数据。

这种类型的虚拟内存完全依赖于硬件支持。它不能通过软件模拟。幸运的是，x86有这么一个东西，叫做MMU(内存管理单元)，它处理由于分段和分页产生的所有内存映射，在CPU和内存之间形成一个层(实际上，它是CPU的一部分，但这只是实现细节)。

### 6.2.分页是虚拟内存的具体实现

虚拟内存是一种抽象的原理。因此，它需要某种某种系统或者算法把它具象化。分段和分页都是实现虚拟内存的可行方法。正如第三部分提到的，分段式管理逐渐地过时了。而分页式管理对x86架构来说是更新的替代品。

页式管理把虚拟内存空间分割为4kb大小的页，而这些页被映射到页框中——通常是同样大小的物理内存块。

#### 6.2.1.页表项

每一个进程通常都有不同的页面映射，这样才能使虚拟内存空间互相独立。在32位x86架构中每页固定为4kb的大小。每一页都有一个相对应的描述符字段，用来告诉处理器它映射到了哪一页框。注意到因为页面和页框都必须以4KB为界限对齐（0x1000字节），Frame adress隐含的最低的12位恒为零。该架构利用这种描述符字段来存储每一页的信息，如是否存在该页、是否是内核态或用户态等等。该字段的组成如下图：

![paging_pte](png\paging_pte.png)

图中的字段很简单，让我们们迅速过一下。

**P**

​	如果设置，代表页在内存中存在。

**R/W**

​	如果该位被设置，代表该页是可写的。反之该页只读。当处于内核态的时候不生效（除非设置了CR0的标志位）。

**U/S**

​	如果设置，代表这是用户态的页面 。反之则是内核态的页面。从用户态不能读写内核态的页面。

**RSVD**

​	CPU内部保留使用。

**A**

​	如果设置，代表该页可被访问（由CPU设置）

**D**

​	如果设置，代表该页已被写入（脏页）。

**AVAIL**

​	这3位未被使用，可供内核使用。

 **Page frame address** 

​	物理内存中页框地址的高20位

#### 6.2.2.页目录/页表

![page_directory](png\page_directory.png)

可能你已经拿出计算器计算出4GB的地址空间下，每4kb页面使用32bit存储描述符，那大概需要4MB的内存。

4MB看起来是一个不小的开销。如果你有4GB的物理内存，可能4MB看起来没多少。但你如果有一台机器工作在16MB的RAM下，存储这些页的描述符就花掉了四分之一的内存！我们想要的是渐进式的，它将占用与内存大小成比例的空间。

好吧，我们并没有这种东西。但英特尔也确实做出了类似的东西——他们使用一个两层系统。CPU能获取到一个页目录，它是一个4KB的大表，每个条目都指向一个页表。而每个页表也是4KB大小，每个条目都是一个页表项，如上所述。

通过这种方式，可以覆盖整个4GB地址空间。它的优点是如果页表没有条目，则可以释放该页表，并且在页目录中将其标记为未设置。

#### 6.2.3.启用分页

启用分页极其简单。

1. 将页目录的地址复制到CR3寄存器中。当然，这必须是物理地址。
2. 设置CR0寄存器的PG位。你可以通过OR-ing控制器来实现。



### 6.3.页错误

当进程执行了内存管理单元“不太乐意”的操作时，会抛出一个页面错误中断。导致这种情况发生的原因有(不完全):

-  读写未被映射的内存区域（页）（未设置页面条目/表的“present”标志）
- 用户态进程试图写入只读页面
- 用户态进程试图访问内核态页面
- 页表项损坏——保留位被覆写

页错误的中断号是14，在第三部分中我们可以看到它会抛出一个错误码。这个错误代码为我们提供了所发生的错误的大量信息。

**第0位**

​	若未设置，代表页不存在引发的错误。

**第1位**

​	若设置，错误由写引发，反之由读引发。

**第2位**

​	若设置，处理器中断时处于用户态，反之处于内核态。

**第3位**

​	若设置，错误原因为保留位被覆写。

**第4位**

​	若设置，错误发生在取指令期间

处理器还为我们提供了另一块信息——错误地址。存放在CR2寄存器。注意，如果您的页面错误处理程序自身导致了页面错误异常，则此寄存器将被覆盖——因此请尽早保存它。 

### 6.4.付诸实践

我们几乎已经准备好开始执行了。但在此之前我们还得需要一些辅助功能，其中最重要的是内存管理功能。

#### 6.4.1.通过placement malloc实现简单内存管理

如果你有C++使用背景，你可能听说过'placement new'。这是new函数的一个带有参数的版本。与普通调用的malloc不同，它会在指定的地址创建对象。我们即将使用一个与它十分相近的概念。

当内核被成功引导之后，我们会得到一个激活的、可操作的内核堆空间。然而，我们要想使用堆来写代码，通常需要先启用虚拟内存。因此，我们需要一个简单的替代方案来在堆激活之前分配内存。由于我们在内核启动的早期就进行了分配，我们可以假设kmalloc()分配的空间不需要再被kfree()。这就大大简化了操作。我们只需要一个指向空闲内存的指针（placement地址），然后将它传递给需要者随后递增，因此有：

```C
u32int kmalloc(u32int sz)
{
  u32int tmp = placement_address;
  placement_address += sz;
  return tmp;
} 
```

这就足够了。不过，我们还有一个要求。当我们分配页表和页目录时，它们必须是页面对齐的。所以我们可以在里面加上：

```c
u32int kmalloc(u32int sz, int align)
{
  if (align == 1 && (placement_address & 0xFFFFF000)) // 如果地址没有页对齐
  {
    // 对齐操作.
    placement_address &= 0xFFFFF000;
    placement_address += 0x1000;
  }
  u32int tmp = placement_address;
  placement_address += sz;
  return tmp;
} 
```

但不幸的是，在后面的教程开始之前，我还不能向你解释为什么真正需要这个操作。它与我们克隆一个页目录时有关(当fork()进程时)。这时候，分页已经完全启用了，`kmalloc()`会返回一个虚拟地址。但是，我们还需要获得分配的内存的物理地址。暂时相信他吧——反正也没多少代码。

```c
u32int kmalloc(u32int sz, int align, u32int *phys)
{
  if (align == 1 && (placement_address & 0xFFFFF000)) // 如果地址没有页对齐
  {
    //对齐操作
    placement_address &= 0xFFFFF000;
    placement_address += 0x1000;
  }
  if (phys)
  {
    *phys = placement_address;
  }
  u32int tmp = placement_address;
  placement_address += sz;
  return tmp;
} 
```

太好了。这就是简单内存管理所需的全部内容。在我的代码中，为了美观，我将`kmalloc`重命名为`kmalloc_int`(表示kmalloc_internal)。然后我封装了几个函数:

```c
u32int kmalloc_a(u32int sz);  // page aligned.
u32int kmalloc_p(u32int sz, u32int *phys); // returns a physical address.
u32int kmalloc_ap(u32int sz, u32int *phys); // page aligned and returns a physical address.
u32int kmalloc(u32int sz); // vanilla (normal). 
```

我觉得这个接口比每次内核堆分配空间都要指定3个参数要好!这些定义应该放在`kheap.h/kheap.c`里面。

#### 6.4.2.必需的定义

`paging.h`应该包含一些结构体定义，这会让事情变得更容易一些。

```C
#ifndef PAGING_H
#define PAGING_H

#include "common.h"
#include "isr.h"

typedef struct page
{
   u32int present    : 1;   // Page present in memory
   u32int rw         : 1;   // Read-only if clear, readwrite if set
   u32int user       : 1;   // Supervisor level only if clear
   u32int accessed   : 1;   // Has the page been accessed since last refresh?
   u32int dirty      : 1;   // Has the page been written to since last refresh?
   u32int unused     : 7;   // Amalgamation of unused and reserved bits
   u32int frame      : 20;  // Frame address (shifted right 12 bits)
} page_t;

typedef struct page_table
{
   page_t pages[1024];
} page_table_t;

typedef struct page_directory
{
   /**
      Array of pointers to pagetables.
   **/
   page_table_t *tables[1024];
   /**
      Array of pointers to the pagetables above, but gives their *physical*
      location, for loading into the CR3 register.
   **/
   u32int tablesPhysical[1024];
   /**
      The physical address of tablesPhysical. This comes into play
      when we get our kernel heap allocated and the directory
      may be in a different location in virtual memory.
   **/
   u32int physicalAddr;
} page_directory_t;

/**
  Sets up the environment, page directories etc and
  enables paging.
**/
void initialise_paging();

/**
  Causes the specified page directory to be loaded into the
  CR3 register.
**/
void switch_page_directory(page_directory_t *new);

/**
  Retrieves a pointer to the page required.
  If make == 1, if the page-table in which this page should
  reside isn't created, create it!
**/
page_t *get_page(u32int address, int make, page_directory_t *dir);

/**
  Handler for page faults.
**/
void page_fault(registers_t regs); 
```

注意`page_directory`结构体的成员 `tablesPhysical`和` physicalAddr`。他们有什么用呢？

`physicalAddr`成员实际上只在我们克隆页面目录时使用(本教程的后面有讲)。请记住，此时，新页目录表在虚拟内存中的地址将与物理内存中的地址不同。如果我们要切换页目录的话，我们需要告诉CPU物理地址。

`tablesPhysical`成员也是类似的。是为了解决一个问题：怎样访问页表？这看起来很简单，但记住页目录必须包含物理地址，而不是虚拟的。但你唯一可以读写内存的方法就是使用虚拟地址！

解决这个问题一个办法是不直接访问页表,但是映射一个页表指向页目录,以便通过访问内存的某个地址你就可以看到所有你的页表，就像页面一样,并且你所有的页表项就像普通的整数一样。 但在我看来，这种方法有点违反直觉，而且还浪费了256MB的可寻址空间，所以我更喜欢另一种方法。

第二种办法是，对于每个页目录，保留两个数组。一个保存它的页表的物理地址(提供给CPU)，另一个保存虚拟地址(以便我们可以读/写它们)。这只会为每个页目录增加4KB的额外开销，不算太多。

#### 6.4.3.页框分配

如果我们要把一个页面映射到页框，我们需要一些办法来寻找一个空闲的页框。当然，我们可以只维护一个由1和0组成的庞大数组，但那太浪费了——我们不需要32位来保存2种值，我们可以用1位来完成。所以如果我们使用[位集](http://en.wikipedia.org/wiki/Bitset)，我们将少消耗32倍的空间!

如果你不知道啥是位集，你应该看看上面的链接。位集实现的功能只有三个:set、test和clear。我还实现了一个函数来有效地从位图中找到第一个空闲页框。 看一看，并搞明白为什么这样做是有效的。我的实现如下所示。我不打算详细解释它——这是一个一般概念，与内核无关。如果你还有疑惑，谷歌一下bitset的实现，实在不行就去osdev上面找。

```C
// A bitset of frames - used or free.
u32int *frames;
u32int nframes;

// Defined in kheap.c
extern u32int placement_address;

// Macros used in the bitset algorithms.
#define INDEX_FROM_BIT(a) (a/(8*4))
#define OFFSET_FROM_BIT(a) (a%(8*4))

// Static function to set a bit in the frames bitset
static void set_frame(u32int frame_addr)
{
   u32int frame = frame_addr/0x1000;
   u32int idx = INDEX_FROM_BIT(frame);
   u32int off = OFFSET_FROM_BIT(frame);
   frames[idx] |= (0x1 << off);
}

// Static function to clear a bit in the frames bitset
static void clear_frame(u32int frame_addr)
{
   u32int frame = frame_addr/0x1000;
   u32int idx = INDEX_FROM_BIT(frame);
   u32int off = OFFSET_FROM_BIT(frame);
   frames[idx] &= ~(0x1 << off);
}

// Static function to test if a bit is set.
static u32int test_frame(u32int frame_addr)
{
   u32int frame = frame_addr/0x1000;
   u32int idx = INDEX_FROM_BIT(frame);
   u32int off = OFFSET_FROM_BIT(frame);
   return (frames[idx] & (0x1 << off));
}

// Static function to find the first free frame.
static u32int first_frame()
{
   u32int i, j;
   for (i = 0; i < INDEX_FROM_BIT(nframes); i++)
   {
       if (frames[i] != 0xFFFFFFFF) // nothing free, exit early.
       {
           // at least one bit is free here.
           for (j = 0; j < 32; j++)
           {
               u32int toTest = 0x1 << j;
               if ( !(frames[i]&toTest) )
               {
                   return i*4*8+j;
               }
           }
       }
   }
} 
```

希望上面的代码没给你带来太大的惊讶。不过就是一些简单的位操作。接下来我们要完成分配和释放物理帧的函数。既然我们已经有了一个高效的位集，这些函数只需要几行代码就能完成！

```c
// 给页分配页帧的函数
void alloc_frame(page_t *page, int is_kernel, int is_writeable)
{
   if (page->frame != 0)
   {
       return; // 该页已经分配好页帧了，直接返回
   }
   else
   {
       u32int idx = first_frame(); // idx 是第一个空页帧的索引
       if (idx == (u32int)-1)
       {
           // PANIC是一个宏，在屏幕上打印错误信息，然后进入无限循环
           PANIC("No free frames!");
       }
       set_frame(idx*0x1000); // 这一帧现在为我们所用！
       page->present = 1; //标记该页存在.
       page->rw = (is_writeable)?1:0; // 标记该页是否可写
       page->user = (is_kernel)?0:1; // 标记该页是否内核态/用户态
       page->frame = idx;
   }
}
// 释放某页的物理帧
void free_frame(page_t *page)
{
   u32int frame;
   if (!(frame=page->frame))
   {
       return; // 该页没有被分配物理帧
   }
   else
   {
       clear_frame(frame); // 该物理帧已被释放
       page->frame = 0x0; // 现在该页没有分配物理帧
   }
}
```

注意PANIC宏调用了一个全局的函数`panic`，参数包括自定义的打印的信息以及发生错误的`__FILE__`和`__LINE__`。`panic`把这些都打印出来并且进入一个无限循环，停止其他的执行。

#### 6.4.4.分页代码

```c
void initialise_paging()
{
   // 物理内存的大小，此时我们假设它16MB
   u32int mem_end_page = 0x1000000;

   nframes = mem_end_page / 0x1000;//一共有多少页帧
   frames = (u32int*)kmalloc(INDEX_FROM_BIT(nframes));
   memset(frames, 0, INDEX_FROM_BIT(nframes));

   // 创建一个页目录
   kernel_directory = (page_directory_t*)kmalloc_a(sizeof(page_directory_t));
   memset(kernel_directory, 0, sizeof(page_directory_t));
   current_directory = kernel_directory;

   // We need to identity map (phys addr = virt addr) from
   // 0x0 to the end of used memory, so we can access this
   // transparently, as if paging wasn't enabled.
   // NOTE that we use a while loop here deliberately.
   // inside the loop body we actually change placement_address
   // by calling kmalloc(). A while loop causes this to be
   // computed on-the-fly rather than once at the start.
   int i = 0;
   while (i < placement_address)
   {
       // Kernel code is readable but not writeable from userspace.
       alloc_frame( get_page(i, 1, kernel_directory), 0, 0);
       i += 0x1000;
   }
   // Before we enable paging, we must register our page fault handler.
   register_interrupt_handler(14, page_fault);

   // Now, enable paging!
   switch_page_directory(kernel_directory);
}

void switch_page_directory(page_directory_t *dir)
{
   current_directory = dir;
   asm volatile("mov %0, %%cr3":: "r"(&dir->tablesPhysical));
   u32int cr0;
   asm volatile("mov %%cr0, %0": "=r"(cr0));
   cr0 |= 0x80000000; // Enable paging!
   asm volatile("mov %0, %%cr0":: "r"(cr0));
}

page_t *get_page(u32int address, int make, page_directory_t *dir)
{
   // Turn the address into an index.
   address /= 0x1000;
   // Find the page table containing this address.
   u32int table_idx = address / 1024;
   if (dir->tables[table_idx]) // If this table is already assigned
   {
       return &dir->tables[table_idx]->pages[address%1024];
   }
   else if(make)
   {
       u32int tmp;
       dir->tables[table_idx] = (page_table_t*)kmalloc_ap(sizeof(page_table_t), &tmp);
       memset(dir->tables[table_idx], 0, 0x1000);
       dir->tablesPhysical[table_idx] = tmp | 0x7; // PRESENT, RW, US.
       return &dir->tables[table_idx]->pages[address%1024];
   }
   else
   {
       return 0;
   }
}
```

好的，让我们分析一下这段代码。我们先看函数的实现。

`switch_page_directory`的功能和它的字面意思一样。它接受一个页目录的地址，然后切换到该页目录。通过把`tablePhysical`的首地址放入CR3寄存器来做到这一点。然后取得CR0寄存器的值，与0x80000000或运算，然后送入CR0寄存器。分页机制开启，页目录缓存被刷新。

`get_page`通过线性地址返回一个指向页的指针。可以选择传入一个参数`make`，如果make为1，并且请求的页表项所位于的页表尚未创建，则将创建该页表。其他情况下，该函数返回0。若页表已经存在，则查找到页表中的页面并返回它。如果页表不存在且make为1,则创建它。

它使用我们的kmalloc_ap函数来检索一个页面对齐的内存块，并获得给定的物理地址。物理地址存储在`tablesPhysical`里，虚拟地址存储在`table`里。

`initialise_paging`首先创建页帧的位集，然后使用`memset`函数把它们置零。然后为页目录分配空间（页对齐）。之后，它分配帧来确保任何页面访问都将映射到具有相同线性地址的帧，称为标识映射。这是在一小部分地址空间内所完成的，因此内核代码可以继续正常运行。它还为页面错误注册一个中断处理程序，然后启用分页。

#### 6.4.5.页面错误处理程序

```c
void page_fault(registers_t regs)
{
   // A page fault has occurred.
   // The faulting address is stored in the CR2 register.
   u32int faulting_address;
   asm volatile("mov %%cr2, %0" : "=r" (faulting_address));

   // The error code gives us details of what happened.
   int present   = !(regs.err_code & 0x1); // Page not present
   int rw = regs.err_code & 0x2;           // Write operation?
   int us = regs.err_code & 0x4;           // Processor was in user-mode?
   int reserved = regs.err_code & 0x8;     // Overwritten CPU-reserved bits of page entry?
   int id = regs.err_code & 0x10;          // Caused by an instruction fetch?

   // Output an error message.
   monitor_write("Page fault! ( ");
   if (present) {monitor_write("present ");}
   if (rw) {monitor_write("read-only ");}
   if (us) {monitor_write("user-mode ");}
   if (reserved) {monitor_write("reserved ");}
   monitor_write(") at 0x");
   monitor_write_hex(faulting_address);
   monitor_write("\n");
   PANIC("Page fault");
}
```

这个处理程序所做的是很好地打印出了错误信息。它从CR2寄存器中取出错误地址，分析错误码，获取页错误的原因。

#### 6.4.6.测试

太好了！你现在有开启分页机制并处理页错误的代码了！让我们看看它实际运行的效果吧～

```c
//main.c
int main(struct multiboot *mboot_ptr)
{
   // Initialise all the ISRs and segmentation
   init_descriptor_tables();
   // Initialise the screen (by clearing it)
   monitor_clear();

   initialise_paging();
   monitor_write("Hello, paging world!\n");

   u32int *ptr = (u32int*)0xA0000000;
   u32int do_page_fault = *ptr;

   return 0;
}
```

显然，它可以先初始化分页，接着打印字符串确保分页初始化正常。接着访问不存在的页0xA0000000，会看到抛出了页错误。

恭喜你，完成了分页部分，接着我们要完成的工作是——建立一个内核堆。这部分的代码可以从这里[下载](http://www.jamesmolloy.co.uk/tutorial_html/paging.tar.gz)。

