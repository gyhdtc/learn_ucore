## 练习三
### 分析 bootloader 进入保护模式的过程
### BIOS 将通过读取硬盘主引导扇区到内存，并转跳到对应内存中的位置执行 bootloader。请分析 bootloader 是如何完成从实模式进入保护模式的。
### 提示：需要阅读小节“保护模式和分段机制”和 lab1/boot/bootasm.S 源码，了解如何从实模式切换到保护模式，需要了解：
- 为何开启 A20，以及如何开启 A20
- 如何初始化 GDT 表
- 如何使能和进入保护模式

----------

## 1. 为何开启 A20，以及如何开启 A20

https://wenku.baidu.com/view/d6efe68fcc22bcd126ff0c00.html
Intel 早期的 8086 CPU 提供了 20 根地址线,可寻址空间范围即 0~2^20(00000H~FFFFFH)的 1MB 内存空间。但 8086 的数据处理位宽位 16 位，无法直接寻址 1MB 内存空间，所以 8086 提供了段地址加偏移地址的地址转换机制。PC 机的寻址结构是 segment:offset，segment 和 offset 都是 16 位的寄存器，最大值是 0ffffh，换算成物理地址的计算方法是把 segment 左移 4 位，再加上 offset，所以 segment:offset 所能表达的寻址空间最大应为 0ffff0h + 0ffffh = 10ffefh（前面的 0ffffh 是 segment=0ffffh 并向左移动 4 位的结果，后面的 0ffffh 是可能的最大 offset），这个计算出的 10ffefh 是多大呢？大约是 1088KB，就是说，segment:offset 的地址表示能力，超过了 20 位地址线的物理寻址能力。所以当寻址到超过 1MB 的内存时，会发生“回卷”（不会发生异常）。但下一代的基于 Intel 80286 CPU 的 PC AT 计算机系统提供了 24 根地址线，这样 CPU 的寻址范围变为 2^24=16M,同时也提供了保护模式，可以访问到 1MB 以上的内存了，此时如果遇到“寻址超过 1MB ”的情况，系统不会再“回卷”了，这就造成了向下不兼容。为了保持完全的向下兼容性，IBM 决定在 PC AT计算机系统上加个硬件逻辑，来模仿以上的回绕特征，于是出现了A20Gate。他们的方法就是把A20地址线控制和键盘控制器的一个输出进行AND操作，这样来控制A20地址线的打开（使能）和关闭（屏蔽\禁止）。  
一开始时A20地址线控制是被屏蔽的（总为0），直到系统软件通过一定的IO操作去打开它（参看bootasm.S）。很显然，在实模式下要访问高端内存区，这个开关必须打开，在保护模式下，由于使用 32 位地址线，如果 A20 恒等于 0，那么系统只能访问奇数兆的内存，即只能访问 0--1M、2-3M、4-5M......，这样无法有效访问所有可用内存。所以在保护模式下，这个开关也必须打开。  
有了这些命令和知识，就可以实现操作 A20 Gate 来从实模式切换到保护模式了。 理论上讲，我们只要操作 8042 芯片的输出端口（64h）的 bit 1，就可以控制 A20 Gate，但实际上，当你准备向 8042 的输入缓冲区里写数据时，可能里面还有其它数据没有处理，所以，我们要首先禁止键盘操作，同时等待数据缓冲区中没有数据以后，才能真正地去操作 8042 打开或者关闭 A20 Gate。  
**打开 A20 Gate 的具体步骤大致如下（参考 bootasm.S）：**
- 等待 8042 Input buffer 为空；
- 发送 Write 8042 Output Port （P2）命令到 8042 Input buffer；
- 等待 8042 Input buffer 为空；
- 将 8042 Output Port（P2）得到字节的第 2 位置 1，然后写入8042 Input buffer；  

**8042 有 4 个寄存器：**  
- 1 个 8-bit 长的 Input buffer；Write-Only；
- 1 个 8-bit 长的 Output buffer； Read-Only；
- 1 个 8-bit 长的 Status Register；Read-Only；
- 1 个 8-bit 长的 Control Register；Read/Write。  

**有两个端口地址：60h 和 64h和cpu通信，有关对它们的读写操作描述如下：**
- 读 60h 端口，读 output buffer
- 写 60h 端口，写 input buffer
- 读 64h 端口，读 Status Register
- 操作 Control Register，首先要向 64h 端口写一个命令（20h 为读命令，60h 为写命令），然后根据命令从 60h 端口读出 Control Register 的数据或者向 60h 端口写入 Control Register 的数据（64h 端口还可以接受许多其它的命令）。  

看一眼每一位得作用：  
|bit |meaning|
| ---- | ---- |
|0	 |output register (60h) 中有数据|
|1	 |input register (60h/64h) 有数据|
|2	 |系统标志（上电复位后被置为0）|
|3	 |data in input register is command (1) or data (0)|
|4	 |1=keyboard enabled, 0=keyboard disabled (via switch)|
|5	 |1=transmit timeout (data transmit not complete)|
|6	 |1=receive timeout (data transmit not complete)|
|7	 |1=even parity rec'd, 0=odd parity rec'd (should be odd)|
代码在boot下，bootasm.S，include了asm.h，调用了bootmain.c：  
https://github.com/chyyuu/ucore_os_lab/blob/master/labcodes_answer/lab1_result/boot/bootasm.S  
启动CPU: 切换到32位保护模式，跳转到C。BIOS将此代码从硬盘的第一个扇区加载到在物理地址0x7c00处的# memory，并开始以实际模式执行#与%cs=0 %ip=7c00。  
```
.set PROT_MODE_CSEG,        0x8                     # kernel code segment selector
.set PROT_MODE_DSEG,        0x10                    # kernel data segment selector
.set CR0_PE_ON,             0x1                     # protected mode enable flag
```
上述三行代码就相当于是#define了，用来表示立即数；  

----
**ps：**& 用来表示端口号，或者立即数；端口号一般是cpu和其他芯片通信的端口；只能用ax、al来 inb或outb它。% 用来表示寄存器。顺带提一句，8024芯片就是利用0x60和0x64端口通信的，不过这款芯片现在大多集成在cpu里了。  
**pps：** cpu里面有EAX寄存器，其中ax寄存器是EAX的低16位；al是ax的低8位，就是EAX的低8位；ah是ax的高8位，就是EAX的【8-15】，8位；

----
## 通用寄存器
**一般寄存器:AX、BX、CX、DX**  
AX:累积暂存器，BX:基底暂存器，CX:计数暂存器，DX:资料暂存器  
**索引暂存器:SI、DI**  
SI:来源索引暂存器，DI:目的索引暂存器  
**堆叠、基底暂存器:SP、BP**  
**SP:堆叠指标暂存器，BP:基底指标暂存器**  
- EAX、ECX、EDX、EBX：为ax,bx,cx,dx的延伸，各为32位元
- ESI、EDI、ESP、EBP：为si,di,sp,bp的延伸，32位元

eax, ebx, ecx, edx, esi, edi, ebp, esp等都是X86 汇编语言中CPU上的通用寄存器的名称，是32位的寄存器。  
这些32位寄存器有多种用途，但每一个都有“专长”，有各自的特别之处。  
- EAX 是"累加器"(accumulator), 它是很多加法乘法指令的缺省寄存器。  
- EBX 是"基地址"(base)寄存器, 在内存寻址时存放基地址。  
- ECX 是计数器(counter), 是重复(REP)前缀指令和LOOP指令的内定计数器。  
- EDX 则总是被用来放整数除法产生的余数。  
- ESI/EDI分别叫做"源/目标索引寄存器"(source/destination index),因为在很多字符串操作指令中, DS:ESI指向源串,而ES:EDI指向目标串。  
EBP是"基址指针"(BASE POINTER), 它最经常被用作高级语言函数调用的"框架指针"(frame pointer). 在破解的时候,经常可以看见一个标准的函数起始代码:  
　　push ebp ;保存当前ebp  
　　mov ebp,esp ;EBP设为当前堆栈指针  
　　sub esp, xxx ;预留xxx字节给函数临时变量.  
　　...  
这样一来,EBP 构成了该函数的一个框架, 在EBP上方分别是原来的EBP, 返回地址和参数. EBP下方则是临时变量. 函数返回时作 mov esp,ebp/pop ebp/ret 即可.

ESP 专门用作堆栈指针，被形象地称为栈顶指针，堆栈的顶部是地址小的区域，压入堆栈的数据越多，ESP也就越来越小。在32位平台上，ESP每次减少4字节。

----
```
.globl start
start:
.code16 # Assemble for 16-bit mode
    cli # Disable interrupts
    cld # String operations increment
```
（1）  
.global关键字用来让一个符号对链接器可见，可以供其他链接对象模块使用。  
.global _start 让_start符号成为可见的标示符，这样链接器就知道跳转到程序中的什么地方并开始执行。linux寻找这个 _start标签作为程序的默认进入点。  
在汇编和C混合编程中，在GNU ARM编译环境下，汇编程序中要使用。  
global伪操作声明汇编程序为全局的函数，意即可被外部函数调用，同时C程序中要使用extern声明要调用的汇编语言程序。  
所以这里start为全局函数。  
（2）  
- 使用了 code16 是告诉编译器：下面的代码要编译成 16 位代码。  
更易于理解地说：是告诉编译器，下面的代码要运行在 16 位模式（实模式）下，你要帮我编译成实模式的代码。  
【在这种情况下，要注意的是：16 位代码的寻址模式比 32 位代码的寻址少很多。】  
16 位下：  
（1）只支持 [si]、[di]、[bp]、[bx] 这些基址寄存器与变址寄存器的间接寻址；  
（2）以及：[bx+si]、[bx+di]、[bp+si]、[bp+di]　基址＋变址的间接寻址

- 使用了 code32 是告诉编译器，下面的代码要编译成 32 位代码。更易于理解地说：是告诉编译器，下面的代码要运行在 32 保护模式下，因此你要帮我编译成 32 位的代码。

（3）cli、cld  
CLI 可以屏蔽中断， STI 恢复中断，于是，两者之间的代码就不会被外部中断打断。所以可以尽量保护代码连续执行。但是对于一些不允许屏蔽的中断以及异常，代码的运行还是会被中断。  
cld 和 std 是一组。cld 让标志位DF = 0，在执行字符串操作时，从低地址开始增加1/2；std 让标志位DF = 1，让它自减。
```
# Set up the important data segment registers (DS, ES, SS).
    xorw %ax, %ax    # Segment number zero
    movw %ax, %ds    # -> Data Segment
    movw %ax, %es    # -> Extra Segment
    movw %ax, %ss    # -> Stack Segment
```
xorw 异或，两个一样的东西异或，就是置0；  
后面都是把一个段寄存器置0而已；
ds 数据段；es 扩展段；ss 堆栈段；  
这个很好，一定去看看：  
https://blog.csdn.net/yxc135/article/details/8753130
```
    # Enable A20:
    #  For backwards compatibility with the earliest PCs, physical
    #  address line 20 is tied low, so that addresses higher than
    #  1MB wrap around to zero by default. This code undoes this.
seta20.1:
    inb $0x64, %al    # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al     # 0xd1 -> port 0x64
    outb %al, $0x64     # 0xd1 means: write data to 8042's P2 port

seta20.2:
    inb $0x64, %al   # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al     # 0xdf -> port 0x60
    outb %al, $0x60     # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
```
inb： al寄存器从端口0x64读取一个字节的数据；  
testb 相当于一个 与& 操作 和判断操作的叠加；首先 0x2 = 0000 0010；它会和al进行 & 操作；然后判断是否为0，为0就置编制为ZF = 1；否则反之；
jnz 就是判断这个ZF标志位。如果为0，就循环回seta20.1。  
当然，为什么呢。根据文档开头所说得介绍：0x64得端口数据，表示了8042键盘控制器得状态，它的得第二位为1，表示端口还有数据没有拿走。所以要循环等待他清空。  
清空后，将立即数0xd1写入al，再写入0x64端口号；这是在向64端口写数据：根据上述对于A20的介绍，这个数据表示，接下来向60端口写入数据的是一个**命令**。  
8042 还有 3 个内部端口：Input Port、Outport Port 和 Test Port，这三个端口的操作都是通过向 64h 发送命令，然后在 60h 进行读写的方式完成，其中本文要操作的 A20 Gate 被定义在 Output Port 的 bit 1 上，所以有必要对 Outport Port 的操作及端口定义做一个说明。  
- 读 Output Port：向 64h 发送 0d0h 命令，然后从 60h 读取 Output Port 的内容
- 写 Output Port：向 64h 发送 0d1h 命令，然后向 60h 写入 Output Port 的数据
- 禁止键盘操作命令：向 64h 发送 0adh
- 打开键盘操作命令：向 64h 发送 0aeh

这里有张图，图的左边是buffer，右边是三个 port，其中更改outport就可以更改 P2X 的值：  
https://learningos.github.io/ucore_os_webdocs/lab1_figs/image012.png  
上书代码最后一段就是给outport赋值 1101 1111，更改了 P2第二位的值为1，现在就可以访问32位地址了。

----

## 2. 如何初始化 GDT 表  
#### 1. 为何要分段？ 2. 如何分段的？
