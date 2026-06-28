---
type: concept
status: refactor
course: stm32-f103-hal
source:
  - local-notes
  - st-reference-manual
  - st-datasheet
  - st-hal-documentation
tags:
  - stm32
  - gpio
  - afio
---
# GPIO 与 AFIO

> 能力定位：正确选择引脚模式，并理解引脚如何交给片上外设。
>
> 原始来源：[[01_准备与基础认知]]、[[02_GPIO与传感器]]。

---

## 本节目标

- 能按电气需求选择 F103 GPIO 模式
- 能完成 LED、按键、DO 传感器和蜂鸣器最小实验
- 能解释 AFIO 重映射与 EXTI 端口选择

## 核心概念

### 知识地图

| 名词 | 工程含义 |
|---|---|
| ODR/BSRR | 输出数据与原子置位/复位寄存器 |
| IDR | 输入数据寄存器 |
| CRL/CRH | F103 引脚模式与速度配置寄存器 |
| AFIO | 复用功能、重映射和 EXTI 端口路由 |
| EXTI | 把引脚边沿变成事件/中断 |

### GPIO 模式

| 模式 | 用途 |
|---|---|
| 输入 | 按键、数字传感器、外部状态 |
| 推挽输出 | 主动输出高低电平，适合 LED 和普通控制信号 |
| 开漏输出 | 只能主动拉低，高电平依赖上拉 |
| 模拟模式 | ADC 输入，关闭不必要的数字输入路径 |
| 复用功能 | 把引脚交给 USART、TIM、SPI、I2C 等外设 |

### 上拉、下拉与浮空

输入引脚必须有稳定的默认电平。内部上拉/下拉是否适合，取决于外部电路；不要让业务输入长期悬空。

- 上拉输入：内部弱上拉提供默认高电平，外部器件通常负责拉低。
- 下拉输入：内部弱下拉提供默认低电平，外部器件通常负责拉高。
- 浮空输入：不提供默认电平，只适合外部电路已可靠驱动的信号。
- 模拟输入：关闭数字输入路径，供 ADC 等模拟外设使用。

### AFIO 与重映射

AFIO 负责部分复用功能和外部中断线路由。重映射不是“换一个 GPIO 名字”，而是把同一外设信号切换到另一组支持的引脚。

### EXTI 与 GPIO

GPIO 负责感知引脚电平；EXTI 负责把特定边沿转换成中断/事件。详细机制见 [[NVIC_EXTI_SysTick]]。

## 常用工程接口

### API 速查

| HAL 函数 / 宏 | 作用 | 常见位置 |
|---|---|---|
| `HAL_GPIO_Init()` | 按 `GPIO_InitTypeDef` 配置一组引脚 | `MX_GPIO_Init()` 或 BSP 初始化 |
| `HAL_GPIO_WritePin()` | 设置普通 GPIO 输出电平 | 主循环、BSP 控制函数 |
| `HAL_GPIO_ReadPin()` | 读取引脚当前输入数据状态 | 按键、DO 传感器、状态检查 |
| `HAL_GPIO_TogglePin()` | 翻转普通输出电平 | LED 心跳、低速调试标记 |
| `HAL_GPIO_EXTI_IRQHandler()` | 处理指定 EXTI 线的挂起标志并进入 HAL 回调 | `stm32f1xx_it.c` 的 EXTI IRQHandler |
| `HAL_GPIO_EXTI_Callback()` | 接收 HAL 分发的 GPIO EXTI 事件 | 用户代码中的回调实现 |
| `HAL_NVIC_SetPriority()` | 设置 EXTI 对应 IRQ 的优先级 | GPIO/EXTI 初始化阶段 |
| `HAL_NVIC_EnableIRQ()` | 允许 NVIC 响应指定 EXTI IRQ | 初始化阶段 |

### 使用注意

| HAL 函数 / 宏 | 前置条件 | 参数重点 | 返回值 / 阻塞与常见坑 |
|---|---|---|---|
| `HAL_GPIO_Init()` | 对应 GPIO 时钟已使能 | `GPIOx`、`Pin/Mode/Pull/Speed` | `void`；配置过程短暂同步执行；忘记开时钟；把输入、模拟或复用模式配错 |
| `HAL_GPIO_WritePin()` | 引脚已配置为普通输出 | 端口、Pin 宏、`GPIO_PIN_SET/RESET` | `void`；非阻塞；不能用它控制已交给 TIM/USART 等外设复用的波形；有效电平可能相反 |
| `HAL_GPIO_ReadPin()` | 输入模式和上下拉正确 | 端口与 Pin 宏 | 返回 `GPIO_PinState`；非阻塞；读的是输入状态，不等于读取输出锁存器；悬空输入结果不可靠 |
| `HAL_GPIO_TogglePin()` | 引脚为普通输出 | 端口与 Pin 宏 | `void`；非阻塞；不适合精确时序或并发修改；不能替代 PWM |
| `HAL_GPIO_EXTI_IRQHandler()` | GPIO 已配置 EXTI，NVIC 已使能 | 触发中断的 `GPIO_Pin` | `void`；中断上下文；IRQHandler 传错 Pin 会导致回调不符合预期 |
| `HAL_GPIO_EXTI_Callback()` | IRQHandler 已调用 `HAL_GPIO_EXTI_IRQHandler()` | `GPIO_Pin`，需自行区分来源 | `void` 弱函数；中断上下文；用户应重写弱函数，不修改 HAL 源码；回调中不要阻塞 |
| `HAL_NVIC_SetPriority()` | 已确定优先级分组 | IRQn、抢占优先级、响应优先级 | `void`；同步配置；把数值大小与优先级高低理解反；多个中断优先级无规划 |
| `HAL_NVIC_EnableIRQ()` | IRQn 与 EXTI 线匹配 | `IRQn_Type` | `void`；同步配置；只配边沿但未使能 NVIC，回调不会进入 |

### 典型调用顺序

```text
CubeMX 选择 GPIO 模式/EXTI 边沿
-> 生成并调用 MX_GPIO_Init()
-> 轮询场景调用 HAL_GPIO_ReadPin/WritePin
-> EXTI 场景由 IRQHandler 调 HAL_GPIO_EXTI_IRQHandler()
-> HAL_GPIO_EXTI_Callback() 中区分 Pin 并设置事件标志
```

### 什么时候用哪个函数？

| 场景 | 推荐函数 |
|---|---|
| 控制普通 LED、片选或使能脚 | `HAL_GPIO_WritePin()` |
| 轮询按键或数字传感器 DO | `HAL_GPIO_ReadPin()` |
| 快速验证主循环/定时任务是否运行 | `HAL_GPIO_TogglePin()` |
| 响应 GPIO 边沿 | EXTI 配置 + `HAL_GPIO_EXTI_IRQHandler()` + `HAL_GPIO_EXTI_Callback()` |
| 输出 PWM、USART、SPI 等外设信号 | 配置复用功能并使用对应外设 HAL，不使用 `HAL_GPIO_WritePin()` 生成波形 |

## 本质理解

GPIO 是芯片和外部电路的边界。代码里的 SET/RESET 必须翻译成真实电压，再翻译成 LED、按键或模块的有效状态。模式选择错误时，业务逻辑写得再对也不会产生正确电气行为。

为什么这样做：先建立硬件和数据流模型，再记 HAL API，才能在 API 变化或板卡变化时仍然定位问题。

## STM32F103 / HAL 中的实现方式

- F103 每个 GPIO 通过 CRL/CRH 选择输入、输出、模拟或复用模式；输出可选推挽/开漏。
- `HAL_GPIO_WritePin` 通常利用 BSRR 原子改变输出，`ReadPin` 读取 IDR。
- USART/TIM/SPI/CAN 常用复用推挽，I2C 常用复用开漏，ADC 引脚应设 Analog。
- AFIO MAPR 管理部分外设重映射，EXTICR 选择某条 EXTI 来自哪个 GPIO 端口。

## CubeMX 配置要点

1. 在 Pinout 点选 GPIO_Output、GPIO_Input、Analog 或外设复用功能。
2. 在 GPIO 参数页设置上下拉、输出初值和速度。
3. 需要 EXTI 时选择边沿并在 NVIC 开启对应中断。
4. 需要 USART/SPI/TIM 重映射时确认新引脚和实际接线同步。

## 最小实验 / 最小框架

### 输出控制

> 代码性质：示例框架，用于理解调用顺序，不能保证直接编译。

```c
static void Buzzer_Set(uint8_t enabled)
{
    HAL_GPIO_WritePin(BUZZER_GPIO_Port, BUZZER_Pin,
                      enabled ? BUZZER_ON_LEVEL : BUZZER_OFF_LEVEL);
}
```

### 输入轮询

> 代码性质：示例框架，用于理解调用顺序，不能保证直接编译。

```c
static uint8_t Key_IsPressed(void)
{
    return HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin) == KEY_ACTIVE_LEVEL;
}

static uint8_t SensorDO_IsActive(void)
{
    GPIO_PinState level = HAL_GPIO_ReadPin(SENSOR_DO_GPIO_Port, SENSOR_DO_Pin);
    return level == SENSOR_DO_ACTIVE_LEVEL;
}

/* Core/Src/main.c：MX_GPIO_Init() 之后进入主循环。 */
while (1)
{
    if (Key_IsPressed())
    {
        HAL_Delay(KEY_DEBOUNCE_MS);
        if (Key_IsPressed())
        {
            HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
            while (Key_IsPressed()) { /* 等待松手；正式项目可改状态机。 */ }
        }
    }

    Buzzer_Set(SensorDO_IsActive());
}
```

### EXTI 事件

> 代码性质：示例框架，用于理解调用顺序，不能保证直接编译。

```c
static volatile uint8_t key_exti_event;

void HAL_GPIO_EXTI_Callback(uint16_t pin)
{
    if (pin == KEY_Pin)
    {
        key_exti_event = 1U;
    }
}

/* Core/Src/main.c 的 while(1) 中处理事件。 */
if (key_exti_event)
{
    key_exti_event = 0U;
    App_OnKeyEvent();
}
```

前提：LED、按键、传感器 DO 和蜂鸣器引脚已由 CubeMX 配置。所有有效电平、上下拉和负载驱动方式以实际开发板原理图为准。

## 调试方法

先确认供电和共地，再检查实际引脚、模式、上下拉、有效电平和复用映射。LED、按键和模块接线以实际开发板原理图和 CubeMX 配置为准。

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| LED 逻辑相反 | 低电平点亮 | 按原理图封装 LED_On/Off |
| PC13 驱动异常 | 该类引脚驱动能力/电路特殊 | 避免直接驱动大负载 |
| 输入随机跳 | 引脚悬空 | 配置上拉/下拉或外部电阻 |
| 按键多次触发 | 机械抖动 | 延时/时间戳/状态机消抖 |
| 重映射后串口不通 | 接线仍在旧引脚 | 同步修改接线与 AFIO |
| EXTI 不进 | 端口路由或 NVIC 错 | 检查 AFIO EXTICR 与中断线 |

## 与其它章节的关系

- [[NVIC_EXTI_SysTick]] 处理 GPIO 边沿中断。
- [[ADC与模拟采样]] 使用 GPIO Analog 模式。
- [[USART与串口协议]] 使用 GPIO 复用和 AFIO 重映射。
- [[数字传感器与LM393]] 和 [[蜂鸣器与简单执行器]] 是 GPIO 模块案例。

## 复习检查清单

- [ ] 能根据外部电路选择输入、推挽、开漏、模拟或复用模式。
- [ ] 能说明上拉、下拉和浮空输入分别会形成什么默认电平。
- [ ] 能根据原理图确认 LED、按键、传感器 DO 和蜂鸣器的有效电平。
- [ ] 能完成按键轮询、消抖、等待松手和数字传感器读取流程。
- [ ] 能解释 AFIO 重映射与修改普通 GPIO 名称的区别。
- [ ] 能画出 GPIO 边沿经 EXTI、IRQHandler 到 HAL 回调的路径。
- [ ] 能按供电、接线、模式、上下拉、复用映射和业务逻辑顺序排错。
