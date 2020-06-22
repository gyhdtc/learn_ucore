### 练习 5：实现函数调用堆栈跟踪函数 （需要编程）
#### 我们需要在 lab1 中完成 kdebug.c 中函数 print_stackframe 的实现，可以通过函数 print_stackframe 来跟踪函数调用堆栈中记录的返回地址。在如果能够正确实现此函数，可在 lab1 中执行 “make qemu”后，在 qemu 模拟器中得到类似如下的输出：

```
ebp:0x00007b28 eip:0x00100992 args:0x00010094 0x00010094 0x00007b58 0x00100096
    kern/debug/kdebug.c:305: print_stackframe+22
ebp:0x00007b38 eip:0x00100c79 args:0x00000000 0x00000000 0x00000000 0x00007ba8
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:0x00007b58 eip:0x00100096 args:0x00000000 0x00007b80 0xffff0000 0x00007b84
    kern/init/init.c:48: grade_backtrace2+33
ebp:0x00007b78 eip:0x001000bf args:0x00000000 0xffff0000 0x00007ba4 0x00000029
    kern/init/init.c:53: grade_backtrace1+38
ebp:0x00007b98 eip:0x001000dd args:0x00000000 0x00100000 0xffff0000 0x0000001d
    kern/init/init.c:58: grade_backtrace0+23
ebp:0x00007bb8 eip:0x00100102 args:0x0010353c 0x00103520 0x00001308 0x00000000
    kern/init/init.c:63: grade_backtrace+34
ebp:0x00007be8 eip:0x00100059 args:0x00000000 0x00000000 0x00000000 0x00007c53
    kern/init/init.c:28: kern_init+88
ebp:0x00007bf8 eip:0x00007d73 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
<unknow>: -- 0x00007d72 –
```

请完成实验，看看输出是否与上述显示大致一致，并解释最后一行各个数值的含义。  

提示：可阅读小节“函数堆栈”，了解编译器如何建立函数调用关系的。在完成 lab1 编译后，查看 lab1/obj/bootblock.asm，了解 bootloader 源码与机器码的语句和地址等的对应关系；查看 lab1/obj/kernel.asm，了解 ucore OS 源码与机器码的语句和地址等的对应关系。  

要求完成函数 kern/debug/kdebug.c::print_stackframe 的实现，提交改进后源代码包（可以编译执行），并在实验报告中简要说明实现过程，并写出对上述问题的回答。  

补充材料：  

由于显示完整的栈结构需要解析内核文件中的调试符号，较为复杂和繁琐。代码中有一些辅助函数可以使用。例如可以通过调用 print_debuginfo 函数完成查找对应函数名并打印至屏幕的功能。具体可以参见 kdebug.c 代码中的注释。  

----
一个函数调用动作可分解为：零到多个 PUSH 指令（用于参数入栈），一个 CALL 指令。CALL 指令内部其实还暗含了一个将返回地址（即** CALL 指令下一条指令的地址**）压栈的动作（由硬件完成）。

```
PUSH 2      //参数2压栈，esp-4
PUSH 1      //参数1压栈，esp-4
CALL aaa    //调用函数
```
几乎所有本地编译器都会在每个函数体之前插入类似如下的汇编指令：
```
pushl   %ebp
movl   %esp , %ebp
```
这样在程序执行到一个函数的实际指令前，已经有以下数据顺序入栈：参数、返回地址、ebp 寄存器。由此得到类似如下的栈结构（参数入栈顺序跟调用方式有关，这里以 C 语言默认的 CDECL 为例）：  
| 栈底方向 | 高位地址
| ---- | ---- |
| 参数3 |
| 参数2 |
| 参数1 |
| 返回地址 |
| 上一层[ebp] | <-------- [ebp] |
| 局部变量 | 低位地址 |  

这两条汇编指令的含义是：首先将 ebp 寄存器入栈，然后将栈顶指针 esp 赋值给 ebp。  

“mov ebp esp” 这条指令表面上看是用 esp 覆盖 ebp 原来的值，其实不然。因为给 ebp 赋值之前，原 ebp 值已经被压栈（位于栈顶），而新的 ebp 又恰恰指向栈顶。  
此时 ebp 寄存器就已经处于一个非常重要的地位，该寄存器中存储着栈中的一个地址（原 ebp 入栈后的栈顶），从该地址为基准，向上（栈底方向）能获取返回地址、参数值，向下（栈顶方向）能获取函数局部变量值，**而该地址处又存储着上一层函数调用时的 ebp 值**。  

**PS:** 这里一直没理解。怎么样反朔的？首先 push ebp 只是将ebp的值入栈，ebp只是个寄存器，它还可以放其他的值；然后再将esp —— 栈顶地址赋值给ebp寄存器。这样，就构成了栈顶记录上一个esp的值，当前ebp记录了当前esp的值。  

一般而言，ss:[ebp+4]处为返回地址，ss:[ebp+8]处为第一个参数值（最后一个入栈的参数值，此处假设其占用 4 字节内存），ss:[ebp-4]处为第一个局部变量，ss:[ebp]处为上一层 ebp 值。由于 ebp 中的地址处总是“上一层函数调用时的 ebp 值”，而在每一层函数调用中，都能通过当时的 ebp 值“向上（栈底方向）”能获取返回地址、参数值，“向下（栈顶方向）”能获取函数局部变量值。如此形成递归，直至到达栈底。这就是函数调用栈。

----

这个练习要完成代码，“kern/debug/kdebug.c::print_stackframe” ：
```
void
print_stackframe(void) {
     /* LAB1 YOUR CODE : STEP 1 */
     /* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
      * (2) call read_eip() to get the value of eip. the type is (uint32_t);
      * (3) from 0 .. STACKFRAME_DEPTH
      *    (3.1) printf value of ebp, eip
      *    (3.2) (uint32_t)calling arguments [0..4] = the contents in address (uint32_t)ebp +2 [0..4]
      *    (3.3) cprintf("\n");
      *    (3.4) call print_debuginfo(eip-1) to print the C calling function name and line number, etc.
      *    (3.5) popup a calling stackframe
      *           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
      *                   the calling funciton's ebp = ss:[ebp]
      */
	uint32_t ebp = read_ebp();
	uint32_t eip = read_eip();
	int i, j;
	for (i = 0; ebp != 0 && i < STACKFRAME_DEPTH; i++) {
		cprintf("ebp : 0x%08x \neip : 0x%08x \n", ebp, eip);
		uint32_t *args = (uint32_t *)ebp + 2;
		for (j = 0; j < 4; j++) {
			cprintf("args %d : 0x%08x \n", j, args[j]);
		}
		cprintf("\n");
		print_debuginfo(eip - 1);
		eip = *(((uint32_t *) ebp)+1);
		ebp = *((uint32_t *) ebp);
	}
}
```
这里的代码其实非常简单，其实就是看看ebp和堆栈上下的值；还是很好理解的。  
首先定义两个uint32类型的值，用来获取ebp，esp的值。然后更具地址进行，加减操作得到其他地址。。挺简单的。我更感兴趣他是如何获取ebp的值的。我们一起看看函数 read_ebp();
```
static inline uint32_t
read_ebp(void) {
    uint32_t ebp;
    asm volatile ("movl %%ebp, %0" : "=r" (ebp));
    return ebp;
}
```
其实还是很简单的~，就是内嵌汇编。解释下一些关键字吧。  

> “movl %1,%0”是指令模板；“%0”和“%1”代表指令的操作数，称为占位符，内嵌汇编靠它们将C语言表达式与指令操作数相对应。  
指令模板后面用小括号括起来的是C语言表达式，本例中只有两个：“result”和“input”，他们按照出现的顺序分别与指令操作数“%0”，“%1，”对应；注意对应顺序：第一个C表达式对应“%0”；第二个表达式对应“%1”，依次类推，操作数至多有10个，分别用“%0”，“%1”….“%9，”表示。    

