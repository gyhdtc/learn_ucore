## Lab1 code解析
只讲述challenge的代码  
首先程序一直在 kernal 中运行，就是ring 0；  
我们需要进行两次中断，首先中断到用户态，然后再回来。  
这就遇到了一个问题，在第一次中断时，中断是 ring 0 到 ring 0 的中断，如何修改栈值才能在中断返回后，回到用户态 ring 3 呢？  
在第二次中，中断发生后，是从 ring 3 中断过去的，系统本身就会发生栈的转换，但是在返回时，还是会回到 ring 3，如何修改栈值才能在中断返回后，留在内核态 ring 0 呢？  

----

在内核态发生中断时，堆栈是这样的：  
|高地址| |
|----|----|
|EFLAGS||
|CS||
|EIP||
|ERROR CODE||
|Trap Number||
|DS||
|······||
|ESI||
|EDI|<--- old esp|
|old esp|在返回地址上面，相当于trap函数的参数<br>其值就是上面一大段的地址，接收它的是一个trapframe指针<br>EDI到EFLAGS之间和trap.h文件中定义的trapframe结构体一致|
|return address||
为了让中断返回到用户态，我们还需要额外的东西，在EFLAGS上需要 SS 和 ESP，这表示原栈段和栈地址。  
既然是要插入东西，并且栈是向下生长的，我们无法知道栈的低地址有没有其他的东西，所以还是重新定义一个 trapframe 结构体最为保险。
```
struct trapframe switchk2u, *switchu2k;
```
然后是一些简易的赋值代码：  
```
    case T_SWITCH_TOU:
    	if (tf->tf_cs != USER_CS) {
    		switchk2u = *tf;
    		switchk2u.tf_esp = (uint32_t *)tf + sizeof(struct trapframe) - 8;
    		switchk2u.tf_ss = switchk2u.tf_ds = switchk2u.tf_es = USER_DS;
    		switchk2u.tf_cs = USER_CS;
    		switchk2u.tf_eflags |= FL_IOPL_MASK; // reset flag
    		*((uint32_t *)tf - 1) = (uint32_t)&switchk2u;
    	}
    	break;
```
注意两点：  
- 新结构体中的 esp 的值应该是没有什么用的，但是这里还是将它设置为**中断之前的值**。我改成 - 1 也 ok。  
> *((uint32_t *)tf - 1) = (uint32_t)&switchk2u;  
- 注意最后一行。tf 指向的是EDI，首先把tf存储的地址拿出来，再减一，得到栈里“old esp”的地址，然后给他重新赋值。

----
第二次中断其实也差不多。  
中断发生在用户态，系统本身就是要发生栈的转换：
|原||新|
|----|----|----|
||<--- old esp||
||--->入栈|SS(new)|
||--->入栈|old ESP(new)|
|||EFLAGS|
|||CS|
|||EIP|
|||ERROR CODE|
|||Trap Number|
|||DS|
|||······|
|||ESI|
|||EDI|<--- old esp|
|||old esp|
|||return address|

要想中断返回时留在 ring 0，就需要删除/覆盖掉开头的SS 和 ESP，所以代码中直接定义一个trapframe指针，利用memmove函数，将old esp以下的数据上移8字节。  
```
case T_SWITCH_TOK:
if (tf->tf_cs != KERNEL_CS) {
    tf->tf_cs = KERNEL_CS;
    tf->tf_ds = tf->tf_es = KERNEL_DS;
    tf->tf_eflags &= ~FL_IOPL_MASK;
    switchu2k = (struct trapframe *)(tf->tf_eflags - (sizeof(struct trapframe) - 8));
    memmove(switchu2k, tf, sizeof(struct trapframe) - 8);
    *((uint32_t *)tf - 1) = (uint32_t)switchu2k;
}
break;
```