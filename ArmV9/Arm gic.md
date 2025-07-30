[GIC相关知识](https://www.zhihu.com/people/eeknow-53-50)
# 1. 简介
在arm的soc系统中，会有多个外设，均有可能会产生中断发送给arm cpu，等待cpu处理。  
而arm cpu对中断，只提供了2根信号，一个nIRQ，一个是nFIQ。因此就需要有一个中断控制器来作为中间的桥接，收集soc的所有中断信号，然后仲裁选择合适的中断，再发送给CPU，等待CPU处理。中断控制器就是GIC
![[Pasted image 20240308091334.png]]
gic的核心功能，就是对soc中外设的中断源的管理，并且提供给软件，配置以及控制这些中断源。  
当对应的中断源有效时，gic根据该中断源的配置，决定是否将该中断信号，发送给CPU。如果有多个中断源有效，那么gic还会进行仲裁，选择最高优先级中断，发送给CPU。  
当CPU接受到gic发送的中断，通过读取gic的寄存器，就可以知道，中断的来源来自于哪里，从而可以做相应的处理。  
当CPU处理完中断之后，会告诉gic，其实就是访问gic的寄存器，该中断处理完毕。gic接受到该信息后，就将该中断源取消，避免又重新发送该中断给cpu以及允许中断抢占。  
之后，会先介绍下gicv2的相关知识，然后介绍目前主流使用的gicv3。
## 1.1 中断术语
**中断状态** 对于每一个中断而言，有以下4个状态：
- inactive：中断处于无效状态
- pending：中断处于有效状态，但是cpu没有响应该中断
- active：cpu在响应该中断
- active and pending：cpu在响应该中断，但是该中断源又发送中断过来 以下是中断状态的转移图。至于图中的转移条件，在gic架构文档中，有介绍。
![[Pasted image 20240308091941.png]]
>GIC检测中断流程如下：
>1. 当GIC检测到一个中断发生时，会将该中断标记为pending状态(**A1**)。
>2. 对处于pending状态的中断，仲裁单元回确定目标CPU，将中断请求发送到这个CPU上。
>3. 对于每个CPU，仲裁单元会从众多pending状态的中断中选择一个优先级最高的中断，发送到目标CPU的CPU Interface模块上。
>4. CPU Interface会决定这个中断是否可以发送给CPU。如果该终端优先级满足要求，GIC会发生一个中断信号给该CPU。
>5. 当一个CPU进入中断异常后，会去读取`GICC_IAR`寄存器来响应该中断(一般是Linux内核的中断处理程序来读寄存器)。寄存器会返回硬件中断号(hardware interrupt ID)，对于SGI中断来说是返回源CPU的ID。
>6. 当GIC感知到软件读取了该寄存器后，又分为如下情况：
	 如果该中断源是pending状态，那么转改将变成active。**(C)**
	 如果该中断又重新产生，那么pending状态变成active and pending。**(D)**
	 如果该中断是active状态，现在变成active and pending。**(A2)** 
	 当处理器完成中断服务，必须发送一个完成信号EOI(End Of Interrupt)给GIC控制器。软件写GICC_EOIR寄存器，状态变成inactive。**(E1)**
补充：
>7. 对于level triggered类型中断来说，当触发电平消失，状态从active and pending变成active。**(B2)**
>
>常用路径是**A1->D->B2->E1。**

**中断触发方式**
- edge-triggered: 边沿触发，当中断源产生一个边沿，中断有效
- level-sensitive：电平触发，当中断源为指定电平，中断有效
**中断类型**
- PPI：（private peripheral interrupt），私有外设中断，该中断来源于外设，但是该中断只对指定的core有效。
- SPI：（shared peripheral interrupt），共享外设中断，该中断来源于外设，但是该中断可以对所有的core有效。
- SGI：（software-generated interrupt），软中断，软件产生的中断，用于给其他的core发送中断信号
- virtual interrupt：虚拟中断，用于支持虚拟机
**中断响应优先级**
soc中，中断有很多，为了方便对中断的管理，对每个中断，附加了中断优先级。在中断仲裁时，高优先级的中断，会优于低优先级的中断，发送给cpu处理。 当cpu在响应低优先级中断时，如果此时来了高优先级中断，那么高优先级中断会抢占低优先级中断，而被处理器响应。 
**中断号**
为了方便对中断的管理，gic为每个中断，分配了一个中断号，也就是interrupt ID。对于中断号，gic也进行了分配：
- ID0-ID15，分配给SGI
- ID16-ID31，分配给PPI
- ID32-ID1019分配给SPI
- 其他
**中断生命周期**
![[Pasted image 20240308092407.png]]
- start：中断开始
- generate：中断源产生中断，发送给gic
- deliver：gic将中断发送给cpu
- activate：cpu响应该中断
- deactivate： cpu响应完中断，告诉gic，中断处理完毕，gic更新该中断状态
- end：中断结束
**banking**
- 中断 对于PPI和SGI，gic可以有多个中断对应于同一个中断号。比如在soc中，有多个外设的中断，共享同一个中断号。 
- 寄存器 对于同一个gic寄存器地址，在不同的情况下，访问的是不同的寄存器。例如在secure和non-secure状态下，访问同一个gic寄存器，其实是访问的不同的gic的寄存器。 具体，更多的信息，得看gic spec以及arm spec。
# 2. GIC V2
![[Pasted image 20240308092907.png]]
GIC可以分成两个部分：仲裁( Distributor )和 CPU 接口(CPU Interface)。
CPU interface有两种，一种就是和普通processor接口，另外一种是和虚拟机接口的。Virtual CPU interface在本文中不会详细描述。
分发器（Distributor）其实应该叫汇聚器，在 IC 的后端设计中， layout 会把各个模块引过来的中断线（就是上面说的三种中断）混接到 GIC 上，然后把混聚合的中断接到 CPU 的 IRQ 和 FIQ 线上，这样 CPU 就有触觉了。
其中Distribute只有一套，故基地址也只有一个，而Interface有多套，因为每个 CPU 都对应一套 interface 。
其中各个子中断使能，设置触发方式，优先级排序，分发到哪个 CPU 上这些功能由 Distribute 负责，总的中断的使能，状态的维护由 Interface 负责。
## 2.1 Distributor
Distributor的主要的作用是检测各个interrupt source的状态，控制各个interrupt source的行为，分发各个interrupt source产生的中断事件分发到指定的一个或者多个CPU interface上。虽然Distributor可以管理多个interrupt source，但是它总是把优先级最高的那个interrupt请求送往CPU interface。Distributor对中断的控制包括：
- 中断enable或者disable的控制。Distributor对中断的控制分成两个级别。一个是全局中断的控制`GIC_DIST_CTRL`。一旦disable了全局的中断，那么任何的interrupt source产生的interrupt event都不会被传递到CPU interface。另外一个级别是对针对各个interrupt source进行控制`GIC_DIST_ENABLE_CLEAR`，disable某一个interrupt source会导致该interrupt event不会分发到CPU interface，但不影响其他interrupt source产生interrupt event的分发；
- 控制将当前优先级最高的中断事件分发到一个或者一组CPU interface。当一个中断事件分发到多个CPU interface的时候，GIC的内部逻辑应该保证只assert 一个CPU。
- 优先级控制；
- interrupt属性设定。例如是level-sensitive还是edge-triggered；
- interrupt group的设定，例如：分成了group0和group1。使用寄存器GICD_IGROUPRn来对每个中断，设置组 group0-安全中断，由nFIQ驱动，group1-非安全中断，由nIRQ驱动。
Distributor可以管理多个interrupt source，这些interrupt source用ID来标识，我们称之interrupt ID。
## 2.2 CPU interface
CPU interface这个block主要用于和process进行接口。该block的主要功能包括：
- enable或者disable CPU interface向连接的CPU assert中断事件。对于ARM，CPU interface block和CPU之间的中断信号线是nIRQCPU和nFIQCPU。如果disable了中断，那么即便是Distributor分发了一个中断事件到CPU interface，但是也不会assert指定的nIRQ或者nFIQ通知processor。
- ackowledging中断（确认中断）。processor会向CPU interface block应答中断，中断一旦被应答，Distributor就会把该中断的状态从pending状态修改成active。如果没有后续pending的中断，那么CPU interface就会deassert nIRQCPU和nFIQCPU信号线。如果在这个过程中（cpu正在处理中断时）又产生了新的中断，那么Distributor就会把该中断的状态从pending状态修改成pending and active（表示中断请求被记录为待处理状态，并且时活跃状态，需要处理）。这时候，CPU interface仍然会保持nIRQ或者nFIQ信号的asserted状态，也就是向processor signal下一个中断。这意味着向处理器发出下一个中断请求的信号。尽管当前正在处理一个中断，但处理器仍保持对新的中断请求的敏感性，以便及时响应。
- 中断处理完毕的通知。对中断完成，定义了两个部分，一个是优先级重置（priority drop），将当前中断屏蔽的最高优先级进行重置，以便能够响应低优先级中断。（group0中断，通过写GICC_EOIR寄存器，来实现优先级重置，group1中断，通过写 GICC_AEOIR 寄存器，来实现优先级重置），另一个是中断无效（interrupt deactivation）：将中断的状态，设置为inactive状态。通过写 GICC_DIR 寄存器，来实现中断无效。当interrupt handler处理完了一个中断的时候，会向写CPU interface的寄存器从而通知GIC CPU已经处理完该中断。做这个动作一方面是通知Distributor将中断状态修改为deactive，另外一方面，可以允许其他的pending的interrupt向CPU interface提交。
		这里为什么要对中断完成，定义2个stage，其实是有考虑的。对于中断来说，我们是希望中断处理程序越短越好，但是有些中断处理程序，就是比较长，在这种情况下，就会使其他中断得到相应，从而影响实时性。 比如当前cpu在响应优先级为4的中断A，但是这个中断A的中断处理程序比较长，此时如果有优先级为5的中断B到来，那么cpu是不会响应这个中断的。 在软件上，会将中断处理程序分为两部分，分为上半部分，和下半部分。在上半部分，完成中断最紧急的任务，然后就可以通知GIC，降低当前的中断处理优先级，以便其他中断能够得到响应。在下半部分，处理该中断的其他事情。 在这种机制下，低优先级的中断，不用等待高优先级的中断，完全执行完中断处理程序后，就可以被cpu所响应，提高实时性。 为了实现上述机制，就将中断完成分成了2步。还是刚刚的例子，cpu在响应优先级为4的中断A，当中断A的上半部分完成后，通知GIC，优先级重置（drop priority），GIC将当前的最高优先级中断重置，重置到响应中断A之前的优先级，比如优先级6，那么此时优先级为5的中断B，就可以被cpu响应。最后中断A的下半部分完成后，通知GIC，将该中断A的状态，设置为inactive状态，此时中断A就真正的完成了。 当然，也可以不将中断完成分成2步，就1步。通过控制 GICC_CTLR寄存器的EOImode比特，来决定是否将中断完成分成2步。
- 设定priority mask。通过priority mask，可以mask掉一些优先级比较低的中断，这些中断不会通知到CPU。
- 设定preemption的策略（抢占策略）
- 在多个中断事件同时到来的时候，选择一个优先级最高的通知processor
GIC仲裁单元：为每一个中断维护一个状态机，分别是：inactive、pending、active and pending、active。
## 2.3 中断的具体处理流程
- GIC决定每个中断的使能状态，不使能的中断，是不能发送中断的
- 如果某个中断的中断源有效，GIC将该中断的状态设置为pending状态，然后判断该中断的目标core
- 对于每一个core，GIC将当前处于pending状态的优先级最高的中断，发送给该core的cpu interface
- cpu interface接收GIC发送的中断请求，判断优先级是否满足要求，如果满足，就将中断通过nFIQ或nIRQ管脚，发送给core。
- core响应该中断，通过读取 GICC_IAR 寄存器，来认可该中断。读取该寄存器，如果是软中断，返回源处理器ID，否则返回中断号。
- 当core认可该中断后，GIC将该中断的状态，修改为active状态
- 当core完成该中断后，通过写 EOIR （end of interrupt register）来实现优先级重置，写 GICC_DIR 寄存器，来无效该中断
## 2.4 GIC支持抢占
GIC中断控制器支持中断优先级抢占，一个高优先级中断可以抢占一个低优先级且处于active状态的中断，即GIC仲裁单元会记录和比较当前优先级最高的pending状态，然后去抢占当前中断，并且发送这个最高优先级的中断请求给CPU，CPU应答了高优先级中断，暂停低优先级中断服务，进而去处理高优先级中断。
GIC会将pending状态优先级最高的中断请求发送给CPU。
# 3 GIC V3
GICv3架构是GICv2架构的升级版，增加了很多东西。变化在于以下：  
- 使用属性层次（affinity hierarchies），来对core进行标识，使gic支持更多的core
- 将cpu interface独立出来，用户可以将其设计在core内部
- 增加redistributor组件，用来连接distributor和cpu interface
- 增加了LPI，使用ITS来解析
- 对于cpu interface的寄存器，增加系统寄存器访问方式
    ![[Pasted image 20240308105357.png]]
包含了以下的组件：  
- distributor：SPI中断的管理，将中断发送给redistributor
- redistributor：PPI，SGI，LPI中断的管理，将中断发送给cpu interface
- cpu interface：传输中断给core
- ITS：用来解析LPI中断

其中，cpu interface是实现在core内部的，distributor，redistributor，ITS是实现在gic内部的。
cpu interface和gic的redistributor通信，通过AXI-Stream协议，来实现通信。
>PI中断（LPI Interrupt）是指低功耗中断（Low Power Interrupt），它是在ACPI（Advanced Configuration and Power Interface）规范中引入的一种中断机制。
传统的中断机制在系统中使用固定数量的中断线，这些中断线是预先分配给特定设备或事件的。然而，这种分配方式会导致中断线的稀缺性和浪费，尤其在现代系统中，设备数量不断增加且功耗管理变得更加重要的情况下。
为了解决这个问题，ACPI规范引入了LPI中断机制。LPI中断是一种虚拟的中断，它不与特定的设备或事件关联，可以动态地分配和管理。LPI中断通过GIC（Generic Interrupt Controller）实现，并且可以在需要时动态地分配给设备或事件。
使用LPI中断有以下优点：
 1. 灵活性：LPI中断的数量不受硬件限制，可以根据系统的需求动态分配和管理。
>  2. 节约资源：LPI中断不需要预留固定数量的中断线，可以避免中断线资源的浪费。 
>  3. 低功耗：LPI中断可以与系统的低功耗模式和功耗管理机制集成，以实现更有效的功耗管理。
通过使用LPI中断，系统可以更好地适应设备数量的增加和功耗管理的需求，提供更灵活和高效的中断处理机制。
## 3.1 和V2对比变化
**属性层次**
gicv3的一大变化，是对core的标识。对core不在使用单一数字来表示，而是使用属性层次来标识，和arm core，使用MPIDR_EL1系统寄存器来标识core一致。  
每个core，根据属性层次的不同，使用不同的标号来识别。如下图所示，是一个4层结构，那么对于一个core来说，就可以用xxx.xxx.xxx.xxx来识别。 
这种标识方式，和ARMv8架构的使用MPIDR_EL1寄存器，来标识core是一样的。
![[Pasted image 20240308110315.png]]
每个core，连接一个cpu interface，而cpu interface会连接gic中的一个redistributor。redistributor的标识和core的标识一样。
![[Pasted image 20240308110539.png]]
**中断生命周期**(和V2一致)
![[Pasted image 20240308111430.png]]
- generate：外设发起一个中断
- distribute：distributor对收到的中断源进行仲裁，然后发送给对应的cpu interface
- deliver：cpu interface将中断发送给core
- activate：core通过读取 GICC_IAR 寄存器，来对中断进行认可
- priority drop: core通过写 GICC_EOIR 寄存器，来实现优先级重置
- deactivation：core通过写 GICC_DIR 寄存器，实现优先级状态改变
**中断流程**  
下图是gic的中断流程，中断分成2类：  
- 一类是中断要通过distributor，比如SPI中断
- 一类是中断不通过distributor，比如LPI中断
![[Pasted image 20240308111636.png]]
中断要通过distributor的中断流程  
- 外设发起中断，发送给distributor
- distributor将该中断，分发给合适的re-distributor
- re-distributor将中断信息，发送给cpu interface。
- cpu interface产生合适的中断异常给处理器
- 处理器接收该异常，并且软件处理该中断      
LPI中断的中断流程
- 外设发起中断，发送给ITS
- ITS分析中断，决定将来发送的re-distributor
- ITS将中断发送给合适的re-distributor
- re-distributor将中断信息，发送给cpu interface。
- cpu interface产生合适的中断异常给处理器
- 处理器接收该异常，并且软件处理该中断
**中断处理**  
中断处理，分为边沿触发处理和电平触发处理  
边沿触发处理
	![](https://pic4.zhimg.com/80/v2-ee45bf9b38ab7ee8dc503d9374d31b7b_720w.webp)
	外部边沿中断到达，中断状态被置为pending状态。  
	软件读取IAR寄存器值，表示PE认可该中断，中断状态被置为active状态
	软件中断处理完毕后，写EOIR寄存器，表示优先级重置。过一段时间后，写DIR寄存器，中断状态被置为idle状态。
电平触发处理
	![](https://pic3.zhimg.com/80/v2-9d95c1721b487eb012c0ccc4d0f02056_720w.webp)
	外部高电平中断到达，中断状态置为pending状态。  
	软件读取IAR寄存器，表示PE认可该中断。但中断依然为高，中断状态进入pending and active状态。  
	软件中断处理完毕后，写EOIR寄存器，表示优先级重置。过一段时间后，写DIR寄存器，中断状态被置为idle状态。
**寄存器**
gicv3中，多了很多寄存器。而且对寄存器，提供了2种访问方式，一种是memory-mapped的访问，一种是系统寄存器访问：  
memory-mapped访问的寄存器：  
- GICC: cpu interface寄存器
- GICD: distributor寄存器
- GICH: virtual interface控制寄存器，在hypervisor模式访问
- GICR: redistributor寄存器
- GICV: virtual cpu interface寄存器
- GITS: ITS寄存器  
系统寄存器访问的寄存器：
- ICC: 物理 cpu interface 系统寄存器
- ICV: 虚拟 cpu interface 系统寄存器
- ICH： 虚拟 cpu interface 控制系统寄存器  
下图是gicv3中，各个寄存器，所在的位置。
![](https://pic3.zhimg.com/80/v2-d0e9c9064ea6f9c357fe3d158e840e5e_720w.webp)
对于系统寄存器访问方式的gic寄存器，是实现在core内部的。而memory-mapped访问方式的gic寄存器，是在gic内部的。  
gicv3架构中没有强制系统寄存器访问方式的寄存器是不能通过memory-mapped方式访问的。也就是ICC, ICV, ICH寄存器，也是可以实现在gic内部，通过memory-mapped方式去访问。但是一般的实现中，是没有这样的实现的。  
下图是ICC的系统寄存器，和memory-mepped方式寄存器的对应关系的一部分，更多的就要查看gicv3的spec。  

![](https://pic3.zhimg.com/80/v2-0a2f0ff3700ba5c6e81228ff7d1a61aa_720w.webp)
那么，问题来了，gicv3中，为什么选择将cpu interface，从gic中抽离，实现在core内部？为什么要将cpu interface的寄存器，增加系统寄存器访问方式，实现在core的内部？这样做，是有什么好处？  
我认为，gicv3的上述安排，第一是为了软件编写能够简单，通用，第二是为了让中断响应能够更快。  
首先要先了解，在gic的寄存器中，哪一些寄存器，是会频繁被core所访问的，哪一些寄存器，是不会频繁被core所访问的。毫无疑问，cpu interface的寄存器，是会频繁被core所访问的，因为core需要访问cpu interface的寄存器，来认可中断，来中断完成，来无效中断。而其他的寄存器，是配置中断的，只有在core需要去配置中断的时候，才用访问得到。有了这个认识，那么理解之后我所讲述的，就比较容易了。  
在gicv2中，cpu interface的寄存器，是实现在gic内部的，因此当core收到一个中断时，会通过axi总线（假设memory总线是axi总线），去访问cpu interface的寄存器。而中断在一个soc系统中，是会频繁的产生的，这就意味着，core会频繁的去访问gic的寄存器，这样会占用axi总线的带宽，总而会影响中断的实时响应。而且core通过axi总线去访问cpu interface寄存器，延迟，也比较大。  
在gicv3中，将cpu interface从gic中抽离出来，实现在core内部，而不实现在gic中。core对cpu interface的访问，通过系统寄存器方式访问，也就是使用msr，mrs访问，那么core对cpu interface的寄存器访问，就加速了，而且还不占用axi总线带宽。这样core对中断的处理，就加速了。  
cpu interface与gic之间，是通过专用的AXI-stream总线，来传输信息的，这样也不会占用AXI总线的带宽。  
后续会介绍gicv3架构的内容。