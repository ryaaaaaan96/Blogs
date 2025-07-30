# 1. ARM体系结构
指令集有四种：复杂指令集（CISC）、精简指令集（RISC）、显示并行指令集（EPIC）、超长指令字指令集（LVIW）

ARM芯片属于精简指令集计算机(RISC：Reduced Instruction Set Computing)，它所用的指令比较简单，有如下特点：
① 对内存只有读、写指令
② 对于数据的运算是在CPU内部实现，不能直接处理内存中的数据，先将内存中的数据加载到寄存器中才能操作，然后再存到寄存器里面
③ 倾向于使用更多的寄存器来存储数据，而不是使用内存中的堆栈，效率更高 
④ 使用RISC指令的CPU复杂度小一点，易于设计，RISC指令的指令长度是固定的，*并且是单周期指令*

x86属于复杂指令集计算机(CISC：Complex Instruction Set Computing)，它所用的指令比较复杂，比如某些复杂的指令，它是通过“微程序”来实现的。

**ARM指令集的特点**
1. 桶型移位寄存器，单周期内可以完成各种移位操作
2. 并不是所有指令都是单周期的
3. 有16为Thumb指令集，是32位指令的压缩形式
4. 通过指令组合，减少了分支的数目，提高了代码的密度
5. 增加了DSP、SIMD/NEON等指令


对于图所示的乘法运算a = a * b，
![[Pasted image 20230929110039.png]]

在RISC中要使用4条汇编指令：
① 读内存a
② 读内存b
③ 计算a\*b
④ 把结果写入内存

![[Pasted image 20231001094247.png]]
在CISC，实际上会去执行一个“微程序”，在“微程序”里，一样是去执行这4操作：

① 读内存a
② 读内存b
③ 计算a\*b
④ 把结果写入内存

# 2. RISC指令和CISC指令对比
CISC的指令能力强，单多数指令使用率低却增加了CPU的复杂度，指令是可变长格式；
RISC的指令大部分为单周期指令，指令长度固定，操作寄存器，对于内存只有Load/Store操作
CISC支持多种寻址方式；RISC支持的寻址方式
CISC通过微程序控制技术实现；
RISC增加了通用寄存器，硬布线逻辑控制为主，采用流水线
CISC的研制周期长
RISC优化编译，有效支持高级语言



# 3. ARM的寄存器
无论是cortex-M3/M4，还是cortex-A7，CPU内部都有R0、R1、……、R15寄存器；
它们可以用来“暂存”数据。
对于R13、R14、R15，还另有用途：
R13：别名SP(Stack Pointer)，栈指针
R14：别名LR(Link Register)，用来保存返回地址
R15：别名PC(Program Counter)，程序计数器，表示当前指令地址，写入新值即可跳转

![[Pasted image 20231001094918.png]]


## 3.1 xPSR
**对于cortex-M3/M4来说**
xPSR实际上对应3个寄存器：
① APSR：Application PSR，应用PSR
② IPSR：Interrupt PSR，中断PSR
③ EPSR：Exectution PSR，执行PSR
这3个寄存器的含义如图所示
这3个寄存器，可以单独访问：
```
MRS  R0, APSR  ;读APSR
MRS  R0, IPSR    ;读IPSR
MSR  APSR, R0   ;写APSR
```

这3个寄存器，也可以一次性访问：
```
MRS  R0,  PSR  ; 读组合程序状态
MSR  PSR, R0   ; 写组合程序状态
```

所谓组合程序状态，入下图所示：
![[Pasted image 20231001095311.png]]
![[Pasted image 20231001095351.png]]
![[Pasted image 20231001095514.png]]

**对于cortex-A7**
还要一个Current Program Status Register
![[Pasted image 20231001095833.png]]

9-5位不同


# 4. 不同的工作模式
![[Pasted image 20231001100002.png]]



快中断：FIQ，不使用别的寄存器，在处理的过程中不需要保存现场，速度更快


# 5. ARM汇编
一开始，ARM公司发布两类指令集：
① ARM指令集，这是32位的，每条指令占据32位，高效，但是太占空间
② Thumb指令集，这是16位的，每条指令占据16位，节省空间
要节省空间时用Thumb指令，要效率时用ARM指令。
一个CPU既可以运行Thumb指令，也能运行ARM指令。
怎么区分当前指令是Thumb还是ARM指令呢？
**程序状态寄存器中有一位，名为“T”，它等于1时表示当前运行的是Thumb指令。**
>假设函数A是使用Thumb指令写的，函数B是使用ARM指令写的，怎么调用A/B？
>>我们可以往PC寄存器里写入函数A或B的地址，就可以调用A或B，但是怎么让CPU在执行A函数是进入Thumb状态，在执行B函数时进入ARM状态？
做个手脚：
调用函数A时，让PC寄存器的BIT0等于1，即：PC=函数A地址+(1<<0)；
调用函数B时，让PC寄存器的BIT0等于0:，即：PC=函数B地址
麻烦吧？
麻烦！
引入Thumb2指令集，
它支持16位指令、32位指令混合编程。


ARM公司推出了： Unified Assembly Language
UAL，统一汇编语言，你不需要去区分这些指令集。
在程序前面用CODE32/CODE16/THUMB表示指令集：ARM/Thumb/Thumb2
日常工作中，只需要这么几条汇编指令，从名字就可以猜出含义：
MOV
LDR/STR
LDM/STM
AND/OR
ADD/SUB
B/BL
DCD
ADR/LDR
CMP
后面再练习这些汇编指令
![[Pasted image 20231001101313.png]]
Operation表示各类汇编指令，比如ADD、MOV；
cond表示conditon，即该指令执行的条件；
S表示该指令执行后，会去修改程序状态寄存器；
Rd为目的寄存器，用来存储运算的结果；
Rn、Operand2是两个源操作数
![[Pasted image 20231001101427.png]]


*ARM和GCC语法差异*
![[Pasted image 20231001101538.png]]

