# 1. ARM汇编
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

# 2. 汇编模拟器
VisUAL