# 59. 定义APB1与APB2外设基地址

本节介绍如何定义STM32中APB1和APB2外设的基地址，理解不同外设总线的内存映射。

- APB1和APB2是STM32微控制器的两条外设总线。
- 每个外设在总线上的基地址可通过参考手册查找。
- 通常在头文件中用宏定义：

```c
#define APB1PERIPH_BASE  0x40000000U
#define APB2PERIPH_BASE  0x40010000U
```

- 通过基地址+偏移量可访问具体外设寄存器。
