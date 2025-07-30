
# 1.下载


# 2. 破解


# 3. 移植

# 4. 使用
## 4.1 连接
使用jlink连接板卡之后，在Trace目录下，点击`open live stream tool` 或者 `open snapshot tool` 进行连接。

`take Snapshot`可以使用GDB（暂时不清楚配置方式），Jlink，STlink（显示是个`!`,但应该能使用），以及API Connection（暂时不清除含义）进行连接。

在setting里面可以对各种连接参数的设置，目前仅仅测试了Jlink的`SEGGER RTT`模式，
设置有两个关键点，一个是`Target Connection`设置为`SEGGER RTT`,另外一个是`Jlink setting`里设置Jlink的连接参数。

连接成功后显示为
![[企业微信截图_17037558057365.png]]
## 4.2 查看参数
![[企业微信截图_1703755991761.png]]
### 4.2.1 Veiws
Veiws中是比较常用的一些窗口
#### 4.2.1.1 Trace Veiw
各种任务运行的时序图

#### 4.2.1.2 Actor Instance
任务的实例图


#### 4.2.1.3 statistics Report
查看任务运行统计，主要是时间上的

#### 4.2.1.4 CPU Load graph
不同任务占用cpu的图

#### 4.2.1.5 Context Switch Intensity
上下文任务切换强度，纵坐标是次数

#### 4.2.1.6 Priority Changes
（需测试）优先级变化记录

#### 4.2.1.7 service call block
服务调用阻塞（不清楚具体使用）？


#### 4.2.1.8  service call Intensity
任务调用次数？

#### 4.2.1.9  communication Flow
通讯调用流图，在任务运行过程中队列，锁，信号量，事件，任务通知等信息流图

#### 4.2.1.10  message Recive Time


#### 4.2.1.11  object History
对象的历史，等待队列，锁，信号量，事件，任务通知等信息的历史

#### 4.2.1.11  object list

#### 4.2.1.12 object utilization（利用率）

#### 4.2.1.13 User Event plot


#### 4.2.1.14 Interval plot

#### 4.2.1.15 Interval Timeline

#### 4.2.1.16 Intervals and State Machines


#### 4.2.1.17 New State Machine

#### 4.2.1.18 State Machines Graph




#### 4.2.1.19 Memory Heap Utilization
堆的利用率

#### 4.2.1.20 Stack Usage
栈的利用率

#### 4.2.1.21 Quick Plot


#### 4.2.1.22 I/O Channel Graphs

#### 4.2.1.23 l/O Channel plots


#### 4.2.1.24 Event Log
#### 4.2.1.25 Event Intensity



#### 4.2.1.26 Interval Coverage


#### 4.2.1.27 Syscall plot


#### 4.2.1.28 Signals and Syscalls Explorer


#### 4.2.1.29 View Port Overview

#### 4.2.1.23 l/O Channel plots