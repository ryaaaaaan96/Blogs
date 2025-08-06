# 03. Debugging操作介绍

## keil

对于keil来说除了基本的外设寄存器，断点，内存 watch窗口以外还有以下功能

### 虚拟串口

打开调试后打开serial windows，相当于一个无法发消息的串口助手，但省去了硬件接线，片内串口的TX消息会直接打印在词窗口，可以在没有usb转ttl的时候应急用。

### Fault Reports

当在Debug模式下时，可以打开keil自带的Fault Reports

### 离线断点

当一个项目复杂后，可能会遇到一些正常运行时会出现，而进入debug模式后却无法复现的bug，这是由于cortex核的调试属于侵入式调试，一定程度上会影响程序的运行，因此，这是需要使用离线断点的方式实现非侵入式调试。
在cmsis_armcc.h文件（GCC中也有），有一个 __BKPT(value) 函数，其作用相当于debug模式下的断点，当程序运行到此处时，会将程序堵塞。

### 在不影响跑程序下进行调试

debug窗口下的Load Application at Startup：要取消取消勾选Load Application at Startup，取消后进入Debug模式不会加载启动文件，即不会重新开始运行，但同时axf文件不会被编译进keil，想要将axf文件与keil相关联还需额外配置。
在调试器配置界面，取消勾选Reset after Connect
取消上述配置后进入Debug模式，会发现无法从汇编窗口跳转到相应的语句，这是由于先前取消勾选Load Application at Startup，这时需要再keil的串口中运行指令
LOAD %L INCREMENTAL

### breakpoint可以通过设置count进行定次数中断

首先在循环中手动打一个断点
点击Debug菜单，选择BreakPoints
找到需要的断点，双击该断点，会看到Expression会显示该断点的信息，修改Count的值为10，点击Define，然后关闭该窗口。
这里说明一下：
手动设置的断点结尾为\123，表示在main.c文件的123行。
Expression为表达式，即断点的条件,这里支持基本的>、<、==、!=等操作符。
Count为次数，表示运行多少次中断一次，手动设置的断点Count都是1。
Command为命令，表示到达该断点时执行的命令，默认为空。

### 变量匹配断点

将变量添加到Watch窗口，右击选择Set Access BreakPoint at xxx。还是弹出刚才的菜单：
勾选Access方式Read或Write，设置Count值，点击Define。这里选择Write，Count值为4，表示该变量第四次被写入时程序会停止。
此外，同样在Watch窗口，右击变量选择Set Access BreakPoint at xxx。勾选Access方式Read或Write，删除Expression下原来的内容，填写表达式“AD== 10”。点击Define。这样当AD==10时程序会停止。

[变量匹配方案](https://blog.csdn.net/AuroraSmith/article/details/146472306)

### 回溯调用函数

点击call statck + Locals 右键进行函数回溯；

## GCC

GCC 通常与 GDB（GNU Debugger）配合使用（如arm-none-eabi-gdb），搭配 OpenOCD、J-Link GDB Server 等调试服务器，实现类似 Keil 的调试功能：

### 函数调用回溯（对应 Keil 的 Call Stack）

GDB 命令：
bt（backtrace）：打印当前函数调用栈，显示函数调用链（类似 Keil 的 Call Stack）。
bt N：显示前 N 层调用栈；bt -N：显示后 N 层调用栈。
frame N（或f N）：切换到第 N 层栈帧，查看对应函数的局部变量。
info locals：查看当前栈帧中的局部变量（对应 Keil 的 Locals 窗口）。

### 断点功能（条件断点、计数断点等）

基本断点：break 文件名:行号 或 break 函数名。
条件断点：break 位置 if 条件（如 break main.c:50 if x>100），等价于 Keil 的断点表达式。
计数断点：ignore 断点编号 N（忽略前 N 次断点触发，第 N+1 次生效），类似 Keil 的 Count 设置。
变量访问断点：
watch 变量名：当变量被修改时触发断点（Write 断点）。
rwatch 变量名：当变量被读取时触发断点（Read 断点）。
awatch 变量名：当变量被读取或修改时触发断点（Read/Write 断点），对应 Keil 的 “Set Access BreakPoint”。

### 虚拟串口（类似 Keil 的 Serial Windows）

GCC 本身不直接提供虚拟串口，但可通过以下方式实现：

半主机模式（Semihosting）：通过printf重定向到调试器输出，需在 GDB 中启用半主机（monitor arm semihosting enable），编译时添加--specs=rdimon.specs链接选项。
自定义日志输出：将串口数据通过 J-Link 的 RTT（Real-Time Transfer）功能输出到 J-Link RTT Viewer，实现无硬件接线的实时日志查看（效率高于半主机）。

### 离线断点（类似 Keil 的__BKPT）

GCC 中可直接使用汇编指令触发断点：

```c
# define__BKPT(value)  __asm__ volatile ("bkpt %0" : : "i"(value))
```

程序运行到__BKPT(0)时会暂停，配合 GDB 可捕获断点事件，实现非侵入式调试。

### 故障报告（类似 Keil 的 Fault Reports）

GCC 环境下可通过解析 Cortex-M 的故障状态寄存器（如SCB->HFSR、SCB->CFSR）手动定位硬 fault：

```c
void HardFault_Handler(void) {
    uint32_t hfsr = SCB->HFSR;
    uint32_t cfsr = SCB->CFSR;
    // 打印故障寄存器值，通过GDB或RTT输出
}
```

配合gdb的info registers命令查看寄存器状态，定位故障原因（如空指针访问、栈溢出）。
