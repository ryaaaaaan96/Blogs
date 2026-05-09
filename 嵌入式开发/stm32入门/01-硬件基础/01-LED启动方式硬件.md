# 1 LED的四种点亮电路
LED的驱动方式，常见的有四种。

**方式1**：使用引脚输出3.3V点亮LED，输出0V熄灭LED。

**方式2**：使用引脚拉低到0V点亮LED，输出3.3V熄灭LED。

有的芯片为了省电等原因，其引脚驱动能力不足，这时可以使用三极管驱动。

**方式3**：使用引脚输出1.2V点亮LED，输出0V熄灭LED。

**方式4**：使用引脚输出0V点亮LED，输出1.2V熄灭LED。
![[Pasted image 20230924093125.png]]

由此，主芯片引脚输出高电平/低电平，即可改变LED状态，而无需关注GPIO引脚输出的是3.3V还是1.2V。

所以简称输出1或0：

逻辑1-->高电平

逻辑0-->低电平


# 2 GPIO寄存器操作：

a.     芯片手册一般有相关章节，用来介绍：power/clock

可以设置对应寄存器使能某个GPIO模块(Module)

有些芯片的GPIO是没有使能开头的，即它总是使能的

b.     一个引脚可以用于GPIO、串口、USB或其他功能，

有对应的寄存器来选择引脚的功能

c.     对于已经设置为GPIO功能的引脚，有方向寄存器用来设置它的方向：输出、输入

d.     对于已经设置为GPIO功能的引脚，有数据寄存器用来写、读引脚电平状态

# 3 GPIO寄存器的2种操作方法：

原则：不能影响到其他位

a.     直接读写：读出、修改对应位、写入

要设置bit n：

val = data_reg;

val = val | (1<<n);

data_reg = val;

要清除bit n：

val = data_reg;

val = val & ~(1<<n);

data_reg = val;

b.     **set-and-clear protocol：**

set_reg, clr_reg, data_reg 三个寄存器对应的是同一个物理寄存器,set_reg和clr_reg是通过硬件实现的，高效，但不是所有芯片都实现了这个功能

**要设置bit n：set_reg = (1<<n);

**要清除bit n：clr_reg = (1<<n);


# 4 具体操作手册
1. 打开原理图，搜“LED”，有2个用户LED
![[Pasted image 20230924161957.png]]
2. 芯片手册，先使能PLL4
![[Pasted image 20230924162015.png]]
3. 芯片手册，使能GPIOA
![[Pasted image 20230924162037.png]]
![[Pasted image 20230924162042.png]]
4. 芯片手册，设置PA10，用作输出
![[Pasted image 20230924162105.png]]
5. 芯片手册，设置PA10的输出电平
方法1：读寄存、修改值、写回去(低效)

GPIOA_ODR地址： 0x50002000 + 0x14
![[Pasted image 20230924162128.png]]

方法2：直接写寄存器，一次操作即可，高效

GPIOA_BSRR地址： 0x50002000 + 0x18
![[Pasted image 20230924162145.png]]