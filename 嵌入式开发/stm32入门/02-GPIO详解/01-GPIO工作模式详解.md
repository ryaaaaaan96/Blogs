# GPIO 详解与应用

## 📋 概述

GPIO（General Purpose Input/Output）是STM32最基础也是最重要的外设之一。掌握GPIO的配置和使用是学习STM32的第一步。本文将深入解析STM32 GPIO的8种工作模式，并提供丰富的实战示例。

## 🎯 GPIO 基础概念

### 什么是GPIO？

GPIO是通用输入输出端口，可以由软件控制其输入输出状态。STM32的GPIO具有以下特点：

- **多功能复用**：每个引脚可配置为多种功能
- **模式丰富**：支持8种不同工作模式
- **高驱动能力**：每个IO最大驱动电流25mA
- **高速切换**：支持高速数字信号

### STM32 GPIO结构

```
┌─────────────────────────────────────────────────┐
│                GPIO单元结构                      │
├─────────────────────────────────────────────────┤
│  输入部分：                                      │
│  ┌─────┐   ┌─────┐   ┌─────┐                   │
│  │保护 │→│施密特│→│输入 │→ 输入数据寄存器        │
│  │二极管│   │触发器│   │驱动 │   (IDR)            │
│  └─────┘   └─────┘   └─────┘                   │
│                                                 │
│  输出部分：                                      │
│  输出数据寄存器 → ┌─────┐ → ┌─────┐             │
│  (ODR)          │推挽 │   │输出 │→ 引脚        │
│                 │或开漏│   │驱动 │             │
│                 └─────┘   └─────┘             │
│                                                 │
│  复用功能：                                      │
│  片上外设 ←→ 复用功能输入/输出                   │
│                                                 │
│  模拟功能：                                      │
│  ADC/DAC ←→ 模拟输入/输出                       │
└─────────────────────────────────────────────────┘
```

## 🔧 GPIO的8种工作模式

### 1. 输入浮空 (GPIO_MODE_INPUT + GPIO_NOPULL)

**特点**：
- 引脚呈高阻态，电平由外部电路决定
- 无内部上下拉电阻
- 功耗最低

**应用场景**：
- 外部已有上下拉电阻的信号输入
- 模拟信号输入前级

```c
// 配置示例
GPIO_InitTypeDef GPIO_InitStruct = {0};

GPIO_InitStruct.Pin = GPIO_PIN_0;
GPIO_InitStruct.Mode = GPIO_MODE_INPUT;      // 输入模式
GPIO_InitStruct.Pull = GPIO_NOPULL;          // 无上下拉
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW; // 速度等级（输入时无意义）

HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

// 读取引脚状态
GPIO_PinState pin_state = HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0);
```

### 2. 输入上拉 (GPIO_MODE_INPUT + GPIO_PULLUP)

**特点**：
- 内部连接上拉电阻（约40kΩ）
- 默认状态为高电平
- 适合按键等需要上拉的输入

**应用场景**：
- 按键输入（按下为低电平）
- 开关信号检测
- I2C通信线路

```c
// 配置示例 - 按键输入
GPIO_InitTypeDef GPIO_InitStruct = {0};

GPIO_InitStruct.Pin = GPIO_PIN_0;           // 按键连接引脚
GPIO_InitStruct.Mode = GPIO_MODE_INPUT;     // 输入模式
GPIO_InitStruct.Pull = GPIO_PULLUP;         // 内部上拉
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;

HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

// 按键检测函数
uint8_t Key_Scan(void)
{
    static uint8_t key_state = 0;
    
    if(HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)
    {
        if(key_state == 0)
        {
            key_state = 1;
            HAL_Delay(10);  // 消抖延时
            if(HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)
            {
                return 1;   // 按键按下
            }
        }
    }
    else
    {
        key_state = 0;
    }
    return 0;               // 按键未按下
}
```

### 3. 输入下拉 (GPIO_MODE_INPUT + GPIO_PULLDOWN)

**特点**：
- 内部连接下拉电阻（约40kΩ）
- 默认状态为低电平
- 适合高电平有效的输入信号

**应用场景**：
- 高电平有效的信号检测
- 传感器信号输入
- 编码器信号

```c
// 配置示例 - 传感器信号输入
GPIO_InitTypeDef GPIO_InitStruct = {0};

GPIO_InitStruct.Pin = GPIO_PIN_1;           // 传感器信号引脚
GPIO_InitStruct.Mode = GPIO_MODE_INPUT;     // 输入模式
GPIO_InitStruct.Pull = GPIO_PULLDOWN;       // 内部下拉
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;

HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

// 传感器信号检测
uint8_t Sensor_Check(void)
{
    return HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_1);
}
```

### 4. 模拟输入 (GPIO_MODE_ANALOG)

**特点**：
- 关闭数字输入缓冲器
- 引脚直接连接到片上模拟外设
- 最低功耗模式

**应用场景**：
- ADC模拟信号采集
- DAC模拟信号输出
- 比较器输入

```c
// 配置示例 - ADC模拟输入
GPIO_InitTypeDef GPIO_InitStruct = {0};

GPIO_InitStruct.Pin = GPIO_PIN_2;           // ADC输入引脚
GPIO_InitStruct.Mode = GPIO_MODE_ANALOG;    // 模拟模式
GPIO_InitStruct.Pull = GPIO_NOPULL;         // 无上下拉

HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

// ADC配置和使用
// （需要配合ADC外设一起使用）
```

### 5. 开漏输出 (GPIO_MODE_OUTPUT_OD)

**特点**：
- 只能输出低电平或高阻态
- 输出高电平需要外部上拉电阻
- 可实现"线与"逻辑

**应用场景**：
- I2C通信接口
- 1-Wire通信
- 多设备共享信号线

```c
// 配置示例 - I2C引脚配置
GPIO_InitTypeDef GPIO_InitStruct = {0};

// I2C SCL引脚
GPIO_InitStruct.Pin = GPIO_PIN_6;
GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_OD;  // 开漏输出
GPIO_InitStruct.Pull = GPIO_PULLUP;          // 内部上拉
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;

HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

// I2C SDA引脚
GPIO_InitStruct.Pin = GPIO_PIN_7;
HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

// 模拟I2C时序
void I2C_Start(void)
{
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_7, GPIO_PIN_SET);   // SDA高
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_6, GPIO_PIN_SET);   // SCL高
    HAL_Delay(1);
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_7, GPIO_PIN_RESET); // SDA低
    HAL_Delay(1);
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_6, GPIO_PIN_RESET); // SCL低
}
```

### 6. 推挽输出 (GPIO_MODE_OUTPUT_PP)

**特点**：
- 能输出强高电平和强低电平
- 驱动能力强，速度快
- 最常用的输出模式

**应用场景**：
- LED控制
- 继电器驱动
- 数字信号输出

```c
// 配置示例 - LED控制
GPIO_InitTypeDef GPIO_InitStruct = {0};

GPIO_InitStruct.Pin = GPIO_PIN_13;           // LED引脚
GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;  // 推挽输出
GPIO_InitStruct.Pull = GPIO_NOPULL;          // 无上下拉
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW; // 低速

HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);

// LED控制函数
void LED_Control(uint8_t state)
{
    if(state)
    {
        HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET);    // LED点亮
    }
    else
    {
        HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_RESET);  // LED熄灭
    }
}

// LED闪烁
void LED_Blink(uint16_t period)
{
    HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);  // 翻转LED状态
    HAL_Delay(period);
}
```

### 7. 复用功能推挽输出 (GPIO_MODE_AF_PP)

**特点**：
- 引脚控制权交给片上外设
- 推挽输出特性
- 支持高速信号传输

**应用场景**：
- SPI通信接口
- UART发送端
- PWM信号输出

```c
// 配置示例 - SPI引脚配置
GPIO_InitTypeDef GPIO_InitStruct = {0};

// SPI1 SCK引脚 (PA5)
GPIO_InitStruct.Pin = GPIO_PIN_5;
GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;      // 复用推挽输出
GPIO_InitStruct.Pull = GPIO_NOPULL;
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_VERY_HIGH;
GPIO_InitStruct.Alternate = GPIO_AF5_SPI1;   // 复用为SPI1

HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

// SPI1 MOSI引脚 (PA7)
GPIO_InitStruct.Pin = GPIO_PIN_7;
HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

// PWM输出配置示例
// TIM2 CH1 (PA0)
GPIO_InitStruct.Pin = GPIO_PIN_0;
GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;      // 复用推挽输出
GPIO_InitStruct.Pull = GPIO_NOPULL;
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
GPIO_InitStruct.Alternate = GPIO_AF1_TIM2;   // 复用为TIM2

HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
```

### 8. 复用功能开漏输出 (GPIO_MODE_AF_OD)

**特点**：
- 引脚控制权交给片上外设
- 开漏输出特性
- 需要外部上拉电阻

**应用场景**：
- I2C硬件接口
- SMBus通信
- 1-Wire硬件接口

```c
// 配置示例 - I2C硬件接口
GPIO_InitTypeDef GPIO_InitStruct = {0};

// I2C1 SCL引脚 (PB8)
GPIO_InitStruct.Pin = GPIO_PIN_8;
GPIO_InitStruct.Mode = GPIO_MODE_AF_OD;      // 复用开漏输出
GPIO_InitStruct.Pull = GPIO_PULLUP;          // 内部上拉
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_VERY_HIGH;
GPIO_InitStruct.Alternate = GPIO_AF4_I2C1;   // 复用为I2C1

HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

// I2C1 SDA引脚 (PB9)
GPIO_InitStruct.Pin = GPIO_PIN_9;
HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);
```

## ⚡ GPIO速度等级

### 速度等级说明

| 速度等级 | 宏定义 | 最大频率 | 应用场景 |
|----------|--------|----------|----------|
| **低速** | `GPIO_SPEED_FREQ_LOW` | 2MHz | LED、按键、低速信号 |
| **中速** | `GPIO_SPEED_FREQ_MEDIUM` | 25MHz | UART、普通数字信号 |
| **高速** | `GPIO_SPEED_FREQ_HIGH` | 50MHz | SPI、I2C、PWM |
| **超高速** | `GPIO_SPEED_FREQ_VERY_HIGH` | 100MHz | 高速总线、时钟信号 |

### 速度选择原则

```c
// 速度选择示例
GPIO_InitTypeDef GPIO_InitStruct = {0};

// LED控制 - 低速即可
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;

// SPI通信 - 高速传输
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_VERY_HIGH;

// 普通数字信号 - 中速
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_MEDIUM;
```

**注意事项**：
- 速度越高，功耗越大
- 速度越高，EMI干扰越严重
- 选择合适的速度等级，避免过度配置

## 🛠️ GPIO实战应用

### 1. 多LED控制系统

```c
// LED状态定义
typedef enum
{
    LED_OFF = 0,
    LED_ON = 1,
    LED_BLINK_SLOW = 2,
    LED_BLINK_FAST = 3
} LED_State_t;

// LED控制结构体
typedef struct
{
    GPIO_TypeDef* port;
    uint16_t pin;
    LED_State_t state;
    uint32_t last_tick;
    uint16_t period;
} LED_Control_t;

// LED控制数组
LED_Control_t leds[] = {
    {GPIOC, GPIO_PIN_13, LED_OFF, 0, 0},    // 系统状态LED
    {GPIOC, GPIO_PIN_14, LED_OFF, 0, 0},    // 通信状态LED  
    {GPIOC, GPIO_PIN_15, LED_OFF, 0, 0},    // 错误状态LED
};

#define LED_COUNT (sizeof(leds)/sizeof(leds[0]))

// LED初始化
void LED_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    
    __HAL_RCC_GPIOC_CLK_ENABLE();
    
    for(int i = 0; i < LED_COUNT; i++)
    {
        GPIO_InitStruct.Pin = leds[i].pin;
        GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
        GPIO_InitStruct.Pull = GPIO_NOPULL;
        GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
        
        HAL_GPIO_Init(leds[i].port, &GPIO_InitStruct);
        HAL_GPIO_WritePin(leds[i].port, leds[i].pin, GPIO_PIN_RESET);
    }
}

// LED状态设置
void LED_SetState(uint8_t led_num, LED_State_t state)
{
    if(led_num < LED_COUNT)
    {
        leds[led_num].state = state;
        leds[led_num].last_tick = HAL_GetTick();
        
        switch(state)
        {
            case LED_BLINK_SLOW:
                leds[led_num].period = 1000;
                break;
            case LED_BLINK_FAST:
                leds[led_num].period = 200;
                break;
            default:
                break;
        }
    }
}

// LED任务处理（在主循环中调用）
void LED_Task(void)
{
    uint32_t current_tick = HAL_GetTick();
    
    for(int i = 0; i < LED_COUNT; i++)
    {
        switch(leds[i].state)
        {
            case LED_OFF:
                HAL_GPIO_WritePin(leds[i].port, leds[i].pin, GPIO_PIN_RESET);
                break;
                
            case LED_ON:
                HAL_GPIO_WritePin(leds[i].port, leds[i].pin, GPIO_PIN_SET);
                break;
                
            case LED_BLINK_SLOW:
            case LED_BLINK_FAST:
                if(current_tick - leds[i].last_tick >= leds[i].period)
                {
                    HAL_GPIO_TogglePin(leds[i].port, leds[i].pin);
                    leds[i].last_tick = current_tick;
                }
                break;
        }
    }
}
```

### 2. 矩阵按键扫描

```c
// 4x4矩阵按键定义
#define KEY_ROWS    4
#define KEY_COLS    4

// 行引脚配置（输出）
const uint16_t row_pins[KEY_ROWS] = {GPIO_PIN_0, GPIO_PIN_1, GPIO_PIN_2, GPIO_PIN_3};
// 列引脚配置（输入上拉）
const uint16_t col_pins[KEY_COLS] = {GPIO_PIN_4, GPIO_PIN_5, GPIO_PIN_6, GPIO_PIN_7};

// 按键值映射表
const uint8_t key_map[KEY_ROWS][KEY_COLS] = {
    {'1', '2', '3', 'A'},
    {'4', '5', '6', 'B'}, 
    {'7', '8', '9', 'C'},
    {'*', '0', '#', 'D'}
};

// 矩阵按键初始化
void Matrix_Key_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    
    __HAL_RCC_GPIOA_CLK_ENABLE();
    
    // 配置行引脚为推挽输出
    for(int i = 0; i < KEY_ROWS; i++)
    {
        GPIO_InitStruct.Pin = row_pins[i];
        GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
        GPIO_InitStruct.Pull = GPIO_NOPULL;
        GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
        
        HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
        HAL_GPIO_WritePin(GPIOA, row_pins[i], GPIO_PIN_SET);  // 默认高电平
    }
    
    // 配置列引脚为输入上拉
    for(int i = 0; i < KEY_COLS; i++)
    {
        GPIO_InitStruct.Pin = col_pins[i];
        GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
        GPIO_InitStruct.Pull = GPIO_PULLUP;
        
        HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
    }
}

// 矩阵按键扫描
uint8_t Matrix_Key_Scan(void)
{
    uint8_t key_value = 0;
    
    for(int row = 0; row < KEY_ROWS; row++)
    {
        // 当前行输出低电平，其他行输出高电平
        for(int i = 0; i < KEY_ROWS; i++)
        {
            if(i == row)
                HAL_GPIO_WritePin(GPIOA, row_pins[i], GPIO_PIN_RESET);
            else
                HAL_GPIO_WritePin(GPIOA, row_pins[i], GPIO_PIN_SET);
        }
        
        HAL_Delay(1);  // 稳定延时
        
        // 扫描列
        for(int col = 0; col < KEY_COLS; col++)
        {
            if(HAL_GPIO_ReadPin(GPIOA, col_pins[col]) == GPIO_PIN_RESET)
            {
                // 消抖处理
                HAL_Delay(10);
                if(HAL_GPIO_ReadPin(GPIOA, col_pins[col]) == GPIO_PIN_RESET)
                {
                    key_value = key_map[row][col];
                    
                    // 等待按键释放
                    while(HAL_GPIO_ReadPin(GPIOA, col_pins[col]) == GPIO_PIN_RESET);
                    HAL_Delay(10);
                    
                    return key_value;
                }
            }
        }
    }
    
    return 0;  // 无按键按下
}
```

### 3. 步进电机控制

```c
// 步进电机控制结构体
typedef struct
{
    GPIO_TypeDef* port;
    uint16_t pins[4];        // 4相步进电机引脚
    uint8_t step_index;      // 当前步序索引
    uint8_t direction;       // 旋转方向 0-正转 1-反转
    uint16_t speed;          // 步进速度(ms)
    uint32_t last_step_time; // 上次步进时间
} StepMotor_t;

// 步进电机实例
StepMotor_t motor = {
    .port = GPIOB,
    .pins = {GPIO_PIN_12, GPIO_PIN_13, GPIO_PIN_14, GPIO_PIN_15},
    .step_index = 0,
    .direction = 0,
    .speed = 10,
    .last_step_time = 0
};

// 4相8拍步进序列
const uint8_t step_sequence[8][4] = {
    {1, 0, 0, 0},  // A相
    {1, 1, 0, 0},  // AB相
    {0, 1, 0, 0},  // B相
    {0, 1, 1, 0},  // BC相
    {0, 0, 1, 0},  // C相
    {0, 0, 1, 1},  // CD相
    {0, 0, 0, 1},  // D相
    {1, 0, 0, 1}   // DA相
};

// 步进电机初始化
void StepMotor_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    
    __HAL_RCC_GPIOB_CLK_ENABLE();
    
    for(int i = 0; i < 4; i++)
    {
        GPIO_InitStruct.Pin = motor.pins[i];
        GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
        GPIO_InitStruct.Pull = GPIO_NOPULL;
        GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
        
        HAL_GPIO_Init(motor.port, &GPIO_InitStruct);
        HAL_GPIO_WritePin(motor.port, motor.pins[i], GPIO_PIN_RESET);
    }
}

// 执行一步
void StepMotor_OneStep(void)
{
    for(int i = 0; i < 4; i++)
    {
        if(step_sequence[motor.step_index][i])
        {
            HAL_GPIO_WritePin(motor.port, motor.pins[i], GPIO_PIN_SET);
        }
        else
        {
            HAL_GPIO_WritePin(motor.port, motor.pins[i], GPIO_PIN_RESET);
        }
    }
    
    // 更新步序索引
    if(motor.direction == 0)  // 正转
    {
        motor.step_index = (motor.step_index + 1) % 8;
    }
    else  // 反转
    {
        motor.step_index = (motor.step_index == 0) ? 7 : (motor.step_index - 1);
    }
}

// 步进电机任务处理
void StepMotor_Task(void)
{
    uint32_t current_time = HAL_GetTick();
    
    if(current_time - motor.last_step_time >= motor.speed)
    {
        StepMotor_OneStep();
        motor.last_step_time = current_time;
    }
}

// 设置电机参数
void StepMotor_SetSpeed(uint16_t speed)
{
    motor.speed = speed;
}

void StepMotor_SetDirection(uint8_t direction)
{
    motor.direction = direction;
}

// 步进指定步数
void StepMotor_Steps(uint16_t steps, uint8_t direction)
{
    motor.direction = direction;
    
    for(uint16_t i = 0; i < steps; i++)
    {
        StepMotor_OneStep();
        HAL_Delay(motor.speed);
    }
}
```

## 🔍 GPIO调试技巧

### 1. 实时监控GPIO状态

```c
// GPIO状态监控函数
void GPIO_Monitor(void)
{
    printf("GPIO Status Monitor:\r\n");
    printf("GPIOA IDR: 0x%04X\r\n", GPIOA->IDR);
    printf("GPIOA ODR: 0x%04X\r\n", GPIOA->ODR);
    printf("GPIOB IDR: 0x%04X\r\n", GPIOB->IDR);
    printf("GPIOB ODR: 0x%04X\r\n", GPIOB->ODR);
    printf("GPIOC IDR: 0x%04X\r\n", GPIOC->IDR);
    printf("GPIOC ODR: 0x%04X\r\n", GPIOC->ODR);
    printf("------------------------\r\n");
}

// 特定引脚状态检查
void GPIO_PinCheck(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin, const char* pin_name)
{
    GPIO_PinState state = HAL_GPIO_ReadPin(GPIOx, GPIO_Pin);
    printf("%s: %s\r\n", pin_name, state ? "HIGH" : "LOW");
}
```

### 2. GPIO性能测试

```c
// GPIO翻转性能测试
void GPIO_PerformanceTest(void)
{
    uint32_t start_time = HAL_GetTick();
    
    // 测试引脚翻转速度
    for(uint32_t i = 0; i < 100000; i++)
    {
        HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);
    }
    
    uint32_t end_time = HAL_GetTick();
    printf("GPIO Toggle Test: %d ms for 100000 toggles\r\n", end_time - start_time);
    
    // 测试直接寄存器操作速度
    start_time = HAL_GetTick();
    
    for(uint32_t i = 0; i < 100000; i++)
    {
        GPIOC->ODR ^= GPIO_PIN_13;  // 直接寄存器操作
    }
    
    end_time = HAL_GetTick();
    printf("Direct Register Test: %d ms for 100000 toggles\r\n", end_time - start_time);
}
```

### 3. 常见问题诊断

```c
// GPIO配置检查函数
void GPIO_ConfigCheck(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin)
{
    uint32_t pin_pos = 0;
    
    // 查找引脚位置
    while(!(GPIO_Pin & (1 << pin_pos)) && pin_pos < 16)
    {
        pin_pos++;
    }
    
    if(pin_pos >= 16) return;
    
    // 读取配置寄存器
    uint32_t moder = (GPIOx->MODER >> (pin_pos * 2)) & 0x03;
    uint32_t otyper = (GPIOx->OTYPER >> pin_pos) & 0x01;
    uint32_t ospeedr = (GPIOx->OSPEEDR >> (pin_pos * 2)) & 0x03;
    uint32_t pupdr = (GPIOx->PUPDR >> (pin_pos * 2)) & 0x03;
    
    printf("GPIO Pin %d Configuration:\r\n", pin_pos);
    printf("Mode: %d (0:Input, 1:Output, 2:AF, 3:Analog)\r\n", moder);
    printf("Output Type: %d (0:Push-Pull, 1:Open-Drain)\r\n", otyper);
    printf("Speed: %d (0:Low, 1:Medium, 2:High, 3:Very High)\r\n", ospeedr);
    printf("Pull: %d (0:No Pull, 1:Pull-Up, 2:Pull-Down)\r\n", pupdr);
}
```

## 💡 GPIO使用建议

### 1. 配置原则

**选择合适的模式**：
- 根据实际需求选择GPIO模式
- 避免配置错误导致的功能异常
- 注意上下拉电阻的选择

**速度等级选择**：
- 不要盲目选择最高速度
- 考虑EMI和功耗因素
- 根据信号频率合理选择

### 2. 性能优化

**批量操作**：
```c
// 高效的批量GPIO操作
void GPIO_MultiplePins_Set(GPIO_TypeDef* GPIOx, uint16_t pins)
{
    GPIOx->BSRR = pins;  // 原子操作设置多个引脚
}

void GPIO_MultiplePins_Reset(GPIO_TypeDef* GPIOx, uint16_t pins)
{
    GPIOx->BSRR = (uint32_t)pins << 16;  // 原子操作复位多个引脚
}
```

**直接寄存器操作**：
```c
// 对于时间要求严格的场合，直接操作寄存器
#define LED_ON()    (GPIOC->BSRR = GPIO_PIN_13)
#define LED_OFF()   (GPIOC->BSRR = (uint32_t)GPIO_PIN_13 << 16)
#define LED_TOGGLE() (GPIOC->ODR ^= GPIO_PIN_13)
```

### 3. 调试技巧

**使用示波器**：
- 观察GPIO输出波形
- 检查信号时序
- 测量驱动能力

**逻辑分析仪**：
- 多路数字信号分析
- 协议解码功能
- 时序关系分析

**在线调试**：
- 使用debugger观察寄存器
- 实时修改GPIO状态
- 单步调试GPIO操作

---

> **GPIO总结**：GPIO是STM32的基础外设，掌握其8种工作模式是学习STM32的第一步。通过合理的配置和编程，可以实现丰富的硬件控制功能。实际应用中要注意模式选择、性能优化和调试技巧。
