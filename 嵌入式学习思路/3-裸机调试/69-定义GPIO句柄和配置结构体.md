# 69. 定义GPIO句柄和配置结构体

本节介绍GPIO驱动中句柄结构体和配置结构体的设计方法。

- 句柄结构体用于保存GPIO实例的状态。
- 配置结构体用于描述初始化参数。
- 例：

```c
typedef struct {
    uint32_t Pin;
    uint32_t Mode;
    uint32_t Pull;
    // ...
} GPIO_InitTypeDef;
```
