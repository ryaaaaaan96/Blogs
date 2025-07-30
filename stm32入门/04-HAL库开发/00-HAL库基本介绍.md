
# 1. 文件下载
[STM32Cube - Products-HAL](https://www.st.com/en/ecosystems/stm32cube/products.html)
链接中点击Open software page链接

对于keil而言的芯片包下载，[Arm Keil | Devices](https://www.keil.arm.com/devices/)



# 2. 下载文件解析

## 2.1 STM32Cube包整体结构

STM32Cube包的文件结构如下：

```
STM32Cube_FW_F4_Vx.x.x/
├── Drivers/          # 驱动文件夹
├── Middlewares/      # 中间件文件夹  
├── Projects/         # 示例项目
├── Utilities/        # 实用工具
└── Documentation/    # 文档说明
```

### Drivers文件夹
	BSP文件夹
		板级支持包，用于适配ST官方对应的开发板的硬件驱动程序，每一种开发板对应一个文件夹。例如触摸屏， LCD NOR FLASH以及 EEPROM等板载硬件资源等驱动。这些文件针**只匹配特定的开发板使用，不同开发板可能不能直接使用。**
	CMSIS文件夹
		顾名思义就是符合**CMSIS**（Cortex Microcontroller Software Interface Standard）标准的软件抽象层组件相关文件。文件夹内部文件比较多。主要包括DSP库 (DSP_LIB文件夹 Cortex-M内核及其设备文件（ Include文件夹），微控制器专用头文件 /启动代码 /专用系统文件等 (Device文件夹 )。 在新建工程的时候，会使用到这个文件夹内部很多文件。
	STM32F4xx_HAL_Driver文件夹
		这个文件夹非常重要，它包含了所有的STM32F4xx系列 HAL库头文件和源文件，也就是所有底层硬件抽象层 API声明和定义。它的作用是屏蔽了复杂的硬件寄存器操作，统一了外设的接口函数。该文件夹包含 Src和 Inc两个子文件夹，其中Src子文件夹存放的是 .c源文件， Inc子文件夹存放的是与之对应的 .h头文件。每个 .c源文件对应一个 .h头文件。源文件名称基本遵循 stm32f4xx_hal_ppp.c定义格式，头文件名称基本遵循 stm32f4xx_hal_ppp.h定义格式。比如 gpio相关的 API的声明和定义在文件 stm32f4xx_hal_gpio.h和 stm32f4xx_hal_gpio.c中。


Middlewares文件夹
	ST子文件夹
		STemWin文件夹：STemWin工具包，Segger提供。
		STM32_USB_Device_Library文件夹：USB从机设备支持包。
		STM32_USB_Host_Library文件夹：USB主机设备支持包。
	Third_Party子文件夹
		FatFs文件夹：FAT文件系统支持包。采用的FATFS文件系统。
		FreeRTOS文件夹：FreeRTOS实时系统支持包。
		LibJPEG文件夹：基于C语言的 JPEG图形解码支持包。
		LwIP文件夹：LwIP网络通信协议支持包。


## 2.2 CMSIS文件夹关键文件

CMSIS文件夹的结构如下：

```
CMSIS/
├── Include/          # CMSIS标准头文件
├── Device/          # 设备相关文件
├── DSP/             # DSP库文件
└── RTOS/            # RTOS相关文件
```

### CMSIS核心文件说明
CMSIS文件夹中的 Device和 Include这两个文件夹中的文件是我们工程中最常用到的。下
CMSIS文件夹中的 Device和 Include这两个文件夹中的文件是我们工程中最常用到的。下面对这两个文件夹作简单的介绍：

### Include文件夹结构

```
Include/
├── cmsis_armcc.h        # ARM编译器支持
├── cmsis_armclang.h     # ARM Clang编译器支持
├── cmsis_compiler.h     # 编译器抽象层
├── cmsis_version.h      # CMSIS版本信息
├── core_cm4.h          # Cortex-M4内核定义
└── mpu_armv7.h         # 内存保护单元
```

Include文件夹存放了符合 CMSIS标准的 Cortex-M 内核头文件。想要深入学习内核的朋友可以配合内核相关的手册去学习。对于 STM32F4的工程，我们只要把我们需要的添加到工程即可，需要的头文件有： cmsis_armcc.h、 cmsis_armclang.h、cmsis_compiler.h、 cmsis_version.h、core_cm4.h和 mpu_armv7.h。这几个头文件，对比起来，我们会比较多接触的是 core_cm4.h。
core_cm4.h是内核底层的文件，由 ARM公司提供，包含一些 ARM内核指令，如软件复位，开关中断等功能，同时包含了一个重要的头文件 stdint.h。

## 2.3 HAL驱动文件的内容

STM32F407xx_HAL_Driver文件夹下的 Src（Source的简写）文件夹存放的是所有外设的驱动程序源码， Inc（Include的简写 ）文件夹存放的是对应源码的头文件。 Release_Notes.html是HAL库的版本更新信息。最后三个是库的用户手册，这个需要可以去熟悉一下，查阅起来很方便。

### HAL库文件结构

```
STM32F4xx_HAL_Driver/
├── Inc/                 # 头文件目录
│   ├── stm32f4xx_hal.h            # HAL库主头文件
│   ├── stm32f4xx_hal_conf.h       # HAL库配置文件
│   ├── stm32f4xx_hal_gpio.h       # GPIO驱动头文件
│   ├── stm32f4xx_hal_uart.h       # UART驱动头文件
│   └── ...                        # 其他外设头文件
├── Src/                 # 源文件目录
│   ├── stm32f4xx_hal.c            # HAL库主源文件
│   ├── stm32f4xx_hal_gpio.c       # GPIO驱动源文件
│   ├── stm32f4xx_hal_uart.c       # UART驱动源文件
│   └── ...                        # 其他外设源文件
└── Release_Notes.html   # 版本说明
```

# 3 HAL库文件介绍

## 3.1 HAL库文件命名规则

HAL库文件的命名按规则，主要规则如下：

### 文件命名规范

| 文件类型 | 命名格式 | 示例 | 说明 |
|----------|----------|------|------|
| **头文件** | `stm32f4xx_hal_[ppp].h` | `stm32f4xx_hal_gpio.h` | 外设驱动头文件 |
| **源文件** | `stm32f4xx_hal_[ppp].c` | `stm32f4xx_hal_gpio.c` | 外设驱动源文件 |
| **模板文件** | `stm32f4xx_hal_[ppp]_template.c` | `stm32f4xx_hal_msp_template.c` | 用户配置模板 |
| **时基文件** | `stm32f4xx_hal_timebase_[xxx].c` | `stm32f4xx_hal_timebase_tim.c` | 系统时基配置 |

**注意事项**：
- 后缀有template的文件是模板文件，修改后改名，去掉template放入工程中
- 对于timebase文件来说需要将文件进行修改删除，只能用一个不能同时使用
- `[ppp]`代表外设名称缩写，如gpio、uart、spi等
- `[xxx]`代表具体的实现方式

### 常用外设文件对照表

| 外设 | 头文件 | 源文件 | 功能描述 |
|------|--------|--------|----------|
| **GPIO** | `stm32f4xx_hal_gpio.h` | `stm32f4xx_hal_gpio.c` | 通用输入输出 |
| **UART** | `stm32f4xx_hal_uart.h` | `stm32f4xx_hal_uart.c` | 串口通信 |
| **SPI** | `stm32f4xx_hal_spi.h` | `stm32f4xx_hal_spi.c` | SPI通信 |
| **I2C** | `stm32f4xx_hal_i2c.h` | `stm32f4xx_hal_i2c.c` | I2C通信 |
| **TIM** | `stm32f4xx_hal_tim.h` | `stm32f4xx_hal_tim.c` | 定时器 |
| **ADC** | `stm32f4xx_hal_adc.h` | `stm32f4xx_hal_adc.c` | 模数转换 |
| **DMA** | `stm32f4xx_hal_dma.h` | `stm32f4xx_hal_dma.c` | 直接内存访问 |

## 3.2 HAL库命名规范

### 结构体命名规范

```c
// HAL句柄结构体命名格式
typedef struct {
    // 结构体成员
} [PPP]_HandleTypeDef;

// 示例
typedef struct {
    USART_TypeDef    *Instance;    // UART寄存器基地址
    UART_InitTypeDef Init;         // UART初始化参数
    // ... 其他成员
} UART_HandleTypeDef;

// 初始化结构体命名格式  
typedef struct {
    // 初始化参数
} [PPP]_InitTypeDef;

// 示例
typedef struct {
    uint32_t BaudRate;         // 波特率
    uint32_t WordLength;       // 数据位长度
    uint32_t StopBits;         // 停止位
    uint32_t Parity;           // 校验位
    // ... 其他参数
} UART_InitTypeDef;
```

### 函数命名规范

```c
// HAL库函数命名格式
HAL_[PPP]_[Action][_IT/_DMA]

// 基本操作函数
HAL_StatusTypeDef HAL_UART_Init(UART_HandleTypeDef *huart);
HAL_StatusTypeDef HAL_UART_DeInit(UART_HandleTypeDef *huart);

// 数据传输函数
HAL_StatusTypeDef HAL_UART_Transmit(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size, uint32_t Timeout);
HAL_StatusTypeDef HAL_UART_Receive(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size, uint32_t Timeout);

// 中断方式
HAL_StatusTypeDef HAL_UART_Transmit_IT(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size);
HAL_StatusTypeDef HAL_UART_Receive_IT(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size);

// DMA方式
HAL_StatusTypeDef HAL_UART_Transmit_DMA(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size);
HAL_StatusTypeDef HAL_UART_Receive_DMA(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size);
```

### 宏定义命名规范

```c
// 外设基地址宏定义
#define [PPP]                   ((PPP_TypeDef *)PPP_BASE)

// 示例
#define GPIOA                   ((GPIO_TypeDef *)GPIOA_BASE)
#define USART1                  ((USART_TypeDef *)USART1_BASE)

// 功能使能/禁用宏
#define __HAL_[PPP]_[ACTION]    // 具体操作宏

// 示例
#define __HAL_UART_ENABLE(huart)        SET_BIT((huart)->Instance->CR1, USART_CR1_UE)
#define __HAL_UART_DISABLE(huart)       CLEAR_BIT((huart)->Instance->CR1, USART_CR1_UE)
```

## 3.3 HAL库回调函数机制
## 3.3 HAL库回调函数机制

HAL库的回调函数，这部分允许用户重定义，并在其中实现用户自定义的功能。

### 回调函数命名规范

```c
// HAL库回调函数命名格式
void HAL_[PPP]_[Event]Callback([PPP]_HandleTypeDef *h[ppp]);

// 常用回调函数示例
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart);        // 发送完成回调
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart);        // 接收完成回调
void HAL_UART_ErrorCallback(UART_HandleTypeDef *huart);         // 错误回调

void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim);     // 定时器周期回调
void HAL_TIM_IC_CaptureCallback(TIM_HandleTypeDef *htim);        // 输入捕获回调

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin);                 // GPIO外部中断回调
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc);          // ADC转换完成回调
```

### 回调函数使用方法

```c
// 1. 在用户代码中重新定义回调函数
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if(huart->Instance == USART1)
    {
        // 用户自定义处理逻辑
        printf("UART receive completed!\r\n");
        
        // 重新启动接收
        HAL_UART_Receive_IT(&huart1, rx_buffer, 10);
    }
}

// 2. 在定时器中断回调中处理周期性任务
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    if(htim->Instance == TIM2)
    {
        // 每1ms执行一次
        system_tick++;
        
        if(system_tick >= 1000)
        {
            system_tick = 0;
            led_toggle_flag = 1;  // 设置LED切换标志
        }
    }
}

// 3. GPIO外部中断回调函数
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    switch(GPIO_Pin)
    {
        case GPIO_PIN_0:  // 按键1
            button1_pressed = 1;
            break;
            
        case GPIO_PIN_1:  // 按键2  
            button2_pressed = 1;
            break;
            
        default:
            break;
    }
}
```

### 回调函数执行流程

```c
// HAL库内部调用流程示例（以UART为例）
void USART1_IRQHandler(void)
{
    HAL_UART_IRQHandler(&huart1);          // 1. 调用HAL中断处理函数
}

void HAL_UART_IRQHandler(UART_HandleTypeDef *huart)
{
    // 2. HAL库内部处理硬件中断
    if(__HAL_UART_GET_FLAG(huart, UART_FLAG_RXNE))
    {
        // 3. 处理接收数据
        // ...
        
        // 4. 调用用户回调函数
        HAL_UART_RxCpltCallback(huart);
    }
}
```

### 常用回调函数分类

| 回调函数类型 | 函数名格式 | 触发时机 | 应用场景 |
|--------------|------------|----------|----------|
| **完成回调** | `HAL_[PPP]_[Event]CpltCallback` | 操作完成时 | 数据传输完成处理 |
| **错误回调** | `HAL_[PPP]_ErrorCallback` | 发生错误时 | 错误处理和恢复 |
| **状态回调** | `HAL_[PPP]_[State]Callback` | 状态改变时 | 状态机控制 |
| **事件回调** | `HAL_[PPP]_[Event]Callback` | 特定事件时 | 事件响应处理 |

## 3.4 HAL库状态管理

### HAL状态枚举

```c
// HAL库通用状态定义
typedef enum
{
    HAL_OK       = 0x00U,    // 操作成功
    HAL_ERROR    = 0x01U,    // 操作错误
    HAL_BUSY     = 0x02U,    // 设备忙
    HAL_TIMEOUT  = 0x03U     // 操作超时
} HAL_StatusTypeDef;

// 外设状态定义（以UART为例）
typedef enum
{
    HAL_UART_STATE_RESET      = 0x00U,    // 未初始化状态
    HAL_UART_STATE_READY      = 0x20U,    // 就绪状态
    HAL_UART_STATE_BUSY       = 0x24U,    // 忙状态
    HAL_UART_STATE_BUSY_TX    = 0x21U,    // 发送忙
    HAL_UART_STATE_BUSY_RX    = 0x22U,    // 接收忙
    HAL_UART_STATE_BUSY_TX_RX = 0x23U,    // 收发都忙
    HAL_UART_STATE_TIMEOUT    = 0xA0U,    // 超时状态
    HAL_UART_STATE_ERROR      = 0xE0U     // 错误状态
} HAL_UART_StateTypeDef;
```

### 状态检查函数

```c
// 获取外设状态
HAL_UART_StateTypeDef HAL_UART_GetState(UART_HandleTypeDef *huart);
uint32_t HAL_UART_GetError(UART_HandleTypeDef *huart);

// 使用示例
if(HAL_UART_GetState(&huart1) == HAL_UART_STATE_READY)
{
    // UART空闲，可以进行新的操作
    HAL_UART_Transmit_IT(&huart1, tx_data, sizeof(tx_data));
}
```

---

> **HAL库总结**：HAL库提供了统一的API接口，简化了STM32外设的使用。掌握HAL库的文件结构、命名规范和回调机制，是高效开发STM32应用的基础。合理使用回调函数和状态管理，可以构建响应及时、运行稳定的嵌入式系统。


