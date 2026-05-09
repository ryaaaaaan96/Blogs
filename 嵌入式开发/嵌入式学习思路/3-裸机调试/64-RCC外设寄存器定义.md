# 64. RCC外设寄存器定义

本节介绍RCC（时钟控制）外设的寄存器结构体定义方法。

- RCC负责系统和外设时钟的使能与复位。
- 例：

```c
typedef struct {
    volatile uint32_t CR;
    volatile uint32_t PLLCFGR;
    volatile uint32_t CFGR;
    // ... 其他寄存器
} RCC_TypeDef;
#define RCC ((RCC_TypeDef*)RCC_BASE)
```

- 通过`RCC->CR`等方式访问时钟控制相关寄存器。
