## 中断（特权转换时）的堆栈变化
### 1. 同级堆栈的变化
发生同级中断时候，在同一个堆栈下进行。  
此时由于不会切换栈，就不用保存SS和ESP，但是由于在处理完中断程序后还是要返回源程序中继续执行的，所以，我们的CS EIP寄存器还是要保存的。  
![](https://img-blog.csdnimg.cn/20181213140614162.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3NDE0NDA1,size_16,color_FFFFFF,t_70)  
### 2. 不同级堆栈的变化  
这里我们讨论的是CPL特权级比DPL低的情况，即数值上CPL > DPL.
这表示我们要往高特权级栈上转移，也意味着我们最后需要恢复旧栈,所以  
1. 处理器先临时保存一下旧栈的SS和ESP（SS是堆栈段寄存器，因为换了一个栈，所以其也要变，ESP相当于在栈上的索引），然后加载新的特权级和DPL相同的段，将其加载到SS和ESP中，然后将之前保存的旧栈的SS和ESP压到新栈中  
![](https://img-blog.csdnimg.cn/20181213134919382.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3NDE0NDA1,size_16,color_FFFFFF,t_70)  
由于SS堆段寄存器是16位寄存器，所以为了对齐，将其0拓展后压栈。
2. 然后在新栈中压入EFLAGS标志寄存器、cs和eip寄存器、Error_code，这里的过程其实就和同级变化一样了。  
![](https://img-blog.csdnimg.cn/20181213140232523.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3NDE0NDA1,size_16,color_FFFFFF,t_70)  
3. 其实在Error_code之后，处理器还要将一大堆寄存器的值放入堆栈，以确保可以顺利从中断返回，参考下图：  
![](https://segmentfault.com/img/remote/1460000009552356)  
对应LAB中的如下代码：  
- Error code、Trap Number
```
.globl vector2
vector2:
  pushl $0
  pushl $2
  jmp __alltraps
  ```
ps：0是error code(对于没有error code的中断，ISR会压入0作为error code；如果中断有error code，这里就不会压入0)，2是中断向量号。注意，在这之前，CPU已经压入了EFLAGS，CS，EIP和Error Code(如果有的话)。在压入error code和中断向量号后，CPU跳到__alltraps，__alltraps会将所有中断需要保存的寄存器存到内核栈，然后将此时栈顶的地址($esp)作为参数传给trap()，trap()会将此时栈中压入的各种寄存器整体当成trapframe来处理。trap()会会根据trapframe中的内容，对中断进行相应的处理。  
- DS -> GS
```
.text
.globl __alltraps
__alltraps:
    # push registers to build a trap frame
    # therefore make the stack look like a struct trapframe
    pushl %ds
    pushl %es
    pushl %fs
    pushl %gs
```
- EAX -> EDI
```
    pushal
```
- 将此时的**数据段**和**附加段**设置为内核的数据段(ISR是位于kernel的)  
```
 # load GD_KDATA into %ds and %es to set up data segments for kernel
    movl $GD_KDATA, %eax
    movw %ax, %ds
    movw %ax, %es
```
- 这段代码先将%esp的值压入内核栈，%esp的值将作为函数trap()的参数，然后我们再call trap。通过向栈中压入各种寄存器的信息并且将栈顶的地址作为trapframe的地址，我们完成了对trapframe的赋值。trap()函数接收到trapframe后就可以根据中断类型做出相应处理了。  
```
 # push %esp to pass a pointer to the trapframe as an argument to trap()
    pushl %esp

    # call trap(tf), where tf=%esp
    call trap
```
----
#### 有两个疑问：1. 堆栈最开始的 EFLAGS、CS、EIP 是谁入的；2. 堆栈转换时候，需要的 SS、ESP 怎么办。  
1. 在调用中断命令 int 执行中：依次入栈：EFLAGS（IF = 0，TF = 0）、CS、EIP；  
上述东西入栈之后，int 命令还有最后一步，设置 IP = n * 4，CS = n * 4 + 2；  
2. 新的SS和ESP在**TSS任务段**中，偏移4到偏移27，有三组，用来保存SS和ESP。**（只是特权级从低到高，在TSS中获得；特权级从高到低，从其他途径获得）**  
![](https://img-blog.csdn.net/20160924135346461#pic_center)  
- 参考：https://blog.csdn.net/qq_40627648/article/details/85001660  
梳理一下过程：
    1. 根据目标代码段的DPL(新的CPL)从TSS中选择切换到的对应的ss和esp。

    2. 从TSS中读取新的ss和esp。在这个过程中如果发现ss、esp或者TSS界限错误都会导致无效TSS异常（#TS）。

    3. 对ss描述符进行检验，如果发生错误，同样产生错误，同样产生#TS异常。

    4. 暂时性地保存当前ss和esp的值。

    5. 加载新的ss和esp。

    6. 将刚刚保存起来的ss和esp的值压入新栈。

    7. 从调用者堆栈中将参数复制到被调用者堆栈（新堆栈）中，复制参数的数目有调用门中Param Count一项来决定。如果Param Count是零的话，将不会复制参数。

    8. 将当前的cs和eip压栈。

    9. 加载调用门中指定的新的cs和eip，开始执行被调用者过程。

 - 那么，正如call指令对应ret，调用门也面临返回的问题。ret基本上就是call的反过程，只是带参数的ret指令会同时释放事先被压栈的参数。由被调用者到调用者的返回过程中，处理器的工作包括一下步骤：
    1. 检查保存的cs中的RPL以判断返回时是否要变换特权级。

    2. 加载被调用者堆栈上的cs和eip（此时会进行代码段描述符和选择子类型和特权级检查）。

    3. 如果ret指令含有参数，则增加esp的值以跳过参数，然后esp指向被保存过的调用者ss和esp。注意，ret的参数必须对应调用门中的Param Count的值。

    4. 加载ss和esp，切换到调用者堆栈，被调用者的ss和esp被丢弃。在这里将会进行ss描述符、esp以及ss段描述符的检查。

    5. 如果ret指令含有参数，增加esp的值以跳过参数（此时已经在调用者堆栈中）。

    6. 检查ds、es、fs、gs的值，如果其中哪一个寄存器指向的段的DPL小于CPL（此规则不适用于一致代码段），那么一个空描述符会被加载到该寄存器。