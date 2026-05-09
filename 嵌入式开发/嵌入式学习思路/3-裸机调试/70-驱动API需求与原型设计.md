# 70. 驱动API需求与原型设计

本节介绍GPIO驱动API的核心功能需求及其原型设计。

- 常见API：
  - 初始化：void GPIO_Init(GPIO_TypeDef *GPIOx, GPIO_InitTypeDef*initStruct);
  - 写引脚：void GPIO_WritePin(GPIO_TypeDef *GPIOx, uint16_t pin, uint8_t val);
  - 读引脚：uint8_t GPIO_ReadPin(GPIO_TypeDef *GPIOx, uint16_t pin);
  - 配置中断等。
- 头文件中应声明所有API原型。
