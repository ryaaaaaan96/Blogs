# 71. 驱动API参数与返回类型

本节介绍GPIO驱动API的参数设计和返回值规范。

- 参数应尽量通用、可扩展。
- 返回值可用于指示操作成功/失败。
- 例：

```c
int GPIO_WritePin(GPIO_TypeDef *GPIOx, uint16_t pin, uint8_t val);
```

- 设计良好的API有助于驱动的可移植性和健壮性。
