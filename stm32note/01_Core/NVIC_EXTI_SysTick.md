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
  - nvic
  - exti
  - systick
---
# NVIC、EXTI 与 SysTick

> 能力定位：把外部事件和周期事件转换成短小、可控的中断处理。
>
> 原始来源：[[03_中断与定时器]]。

---

## 本节目标

- 能说明向量表、NVIC、EXTI 和回调链
- 能配置按键 EXTI 最小实验
- 能区分 SysTick、TIM 中断和主循环任务

## 核心概念

### 知识地图

| 名词 | 工程含义 |
|---|---|
| NVIC | 使能、挂起、优先级和嵌套管理 |
| EXTI | 外部/内部事件线路 |
| 向量表 | 异常号到入口函数的映射 |
| SysTick | Cortex-M3 24 位系统定时器 |
| HAL tick | HAL 默认毫秒时基 |

### 中断本质

中断让 CPU 暂停当前流程，跳到异常入口处理事件，再返回原流程。ISR 应尽快完成，复杂业务用标志位交给主循环或任务处理。

### 三个角色

| 角色 | 作用 |
|---|---|
| NVIC | 管理中断使能、优先级和嵌套 |
| EXTI | 把 GPIO 边沿或内部事件转换成中断请求 |
| SysTick | Cortex-M 内核定时器，HAL 常用作毫秒时基 |

### 优先级

抢占优先级决定能否打断其他中断；响应优先级决定同级请求的先后顺序。优先级不是越高越好，应按实时性和共享资源设计。

### HAL 回调链

```text
硬件事件 -> IRQHandler -> 对应 HAL IRQHandler（如 HAL_GPIO_EXTI_IRQHandler）-> HAL 回调 -> 设置标志位
```

HAL 回调函数是驱动在统一 IRQHandler 中完成状态处理后留给用户的扩展入口。回调里应先判断外设实例或 Pin，再记录事件并尽快返回。

### 中断中不要做的事

- 长时间阻塞或 `HAL_Delay()`。
- 大量 `printf`。
- 复杂协议解析。
- 不受控地访问共享缓冲区。

共享变量通常需要 `volatile`，并考虑原子性和临界区。

## 本质理解

中断不是另一个线程，而是打断当前上下文执行的特殊控制流。越长的 ISR 越容易阻塞其他事件、破坏实时性，因此回调应采集状态并快速退出。


## STM32F103 / HAL 中的实现方式

- GPIO 边沿先进入 EXTI 线路，再由 NVIC 调度 IRQHandler；HAL IRQHandler 清标志并调用用户回调。
- SysTick 周期中断累加 HAL tick，`HAL_GetTick()` 读取计数，`HAL_Delay()` 轮询时间差。
- 抢占优先级决定能否嵌套，响应优先级决定同抢占级请求的排队顺序。

## CubeMX 配置要点

1. 把按键脚配置为 GPIO_EXTI，并选择上升沿/下降沿。
2. 设置正确上下拉。
3. 在 NVIC 勾选对应 EXTI 线并配置优先级。
4. 生成后在 `HAL_GPIO_EXTI_Callback` 判断 Pin 并置标志位。

## 常用工程接口

### API 速查

| HAL 函数 / 宏 | 作用 | 常见位置 |
|---|---|---|
| `HAL_NVIC_SetPriority()` | 设置某个 IRQ 的抢占/响应优先级 | 外设 MSP 或 CubeMX 生成初始化 |
| `HAL_NVIC_EnableIRQ()` | 使能指定 IRQ | 初始化阶段 |
| `HAL_NVIC_DisableIRQ()` | 临时屏蔽指定 IRQ | 临界维护、停机流程 |
| `HAL_GPIO_EXTI_IRQHandler()` | 清理并分发 GPIO EXTI 中断 | `stm32f1xx_it.c` 对应 IRQHandler |
| `HAL_GPIO_EXTI_Callback()` | 用户处理 EXTI 事件的弱函数入口 | 用户代码 |
| `HAL_GetTick()` | 读取 HAL tick 计数 | 超时、消抖、非阻塞节拍 |
| `HAL_Delay()` | 基于 HAL tick 阻塞等待 | 初始化等待、简单实验 |
| `HAL_IncTick()` | 递增 HAL 内部 tick | 默认由 SysTick/HAL 时基 ISR 调用 |

### 使用注意

| HAL 函数 / 宏 | 前置条件 | 参数重点 | 返回值 / 阻塞与常见坑 |
|---|---|---|---|
| `HAL_NVIC_SetPriority()` | 优先级分组已确定 | `IRQn`、`PreemptPriority`、`SubPriority` | `void`；同步配置；数值越小优先级越高；不要随意让所有中断同级 |
| `HAL_NVIC_EnableIRQ()` | 外设中断源和 IRQn 匹配 | `IRQn_Type` | `void`；同步配置；外设已开中断但 NVIC 未开，IRQHandler 不会执行 |
| `HAL_NVIC_DisableIRQ()` | 清楚屏蔽期间事件处理策略 | `IRQn_Type` | `void`；同步配置；屏蔽后忘记恢复；屏蔽 IRQ 不等于清除外设标志 |
| `HAL_GPIO_EXTI_IRQHandler()` | EXTI 边沿、线路和 NVIC 已配置 | `GPIO_Pin` | `void`；中断上下文；不应绕过它直接只写业务；共享 IRQ 中要处理正确 Pin |
| `HAL_GPIO_EXTI_Callback()` | HAL IRQHandler 调用链完整 | `GPIO_Pin` | `void`；中断上下文；只置标志/记时间，避免 `printf`、长循环和 `HAL_Delay()` |
| `HAL_GetTick()` | HAL tick 正常递增 | 无 | 返回 `uint32_t`；非阻塞；要用无符号差值处理回绕，不比较“永远小于某绝对值” |
| `HAL_Delay()` | tick 中断能运行 | 毫秒延时量 | `void`；阻塞；中断中长时间调用可能因 tick 无法推进而卡死或拖延其它 IRQ |
| `HAL_IncTick()` | 时基已配置 | 无 | `void`；中断上下文；通常不由业务手动调用，否则软件时间会失真 |

### 典型调用顺序

```text
CubeMX 配置 GPIO_EXTI 和边沿
-> 设置 IRQ 优先级并 HAL_NVIC_EnableIRQ()
-> EXTI IRQHandler 进入 HAL_GPIO_EXTI_IRQHandler()
-> HAL_GPIO_EXTI_Callback() 记录事件
-> 主循环用 HAL_GetTick() 做消抖/超时并处理业务
```

### 什么时候用哪个函数？

| 场景 | 推荐函数 |
|---|---|
| 简单毫秒阻塞等待 | `HAL_Delay()`，仅用于允许阻塞的线程/主循环 |
| 非阻塞超时、按键消抖 | `HAL_GetTick()` + 无符号时间差 |
| GPIO 边沿响应 | `HAL_GPIO_EXTI_IRQHandler()` + `HAL_GPIO_EXTI_Callback()` |
| 临时关闭一个 IRQ | `HAL_NVIC_DisableIRQ()`，完成后按设计恢复 |
| 周期性精确定时任务 | TIM 中断，不依赖主循环反复 `HAL_Delay()` |

## 最小实验 / 最小框架

> 代码性质：示例框架，用于理解调用顺序，不能保证直接编译。
> 验证状态：未上板验证，需按 CubeMX 变量名、开发板原理图和课程源码核对。

```c
static volatile uint8_t key_event;
static uint32_t heartbeat_tick;

/* 常见位置：Core/Src/stm32f1xx_it.c。IRQ 名称以 CubeMX 生成结果为准。 */
void EXTI0_IRQHandler(void)
{
    HAL_GPIO_EXTI_IRQHandler(KEY_Pin);
}

void HAL_GPIO_EXTI_Callback(uint16_t pin)
{
    if (pin == KEY_Pin)
    {
        key_event = 1U;
    }
}

/* Core/Src/main.c 的 while(1) 中：用 tick 实现非阻塞周期任务。 */
void App_Task(void)
{
    uint32_t now = HAL_GetTick();

    if ((uint32_t)(now - heartbeat_tick) >= HEARTBEAT_INTERVAL_MS)
    {
        heartbeat_tick = now;
        HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
    }

    if (key_event)
    {
        key_event = 0U;
        App_OnKeyPressed();
    }
}
```

反例：不要在 `HAL_GPIO_EXTI_Callback()` 中长时间调用 `HAL_Delay()`、打印大量日志或等待按键松手。IRQ 名称、Pin 和优先级以 CubeMX 生成文件为准。

## 调试方法

```text
确认 GPIO 电平确实产生边沿
-> 确认 EXTI 端口/线路映射
-> 确认 NVIC 已使能
-> 断点看 IRQHandler
-> 再看 HAL 回调
-> 检查标志位和主循环处理
```

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| 完全不进中断 | NVIC 未开或线路映射错 | 检查 CubeMX NVIC/AFIO |
| 函数名不对 | IRQHandler 与启动文件不匹配 | 按 startup 向量名实现 |
| 反复触发 | 按键抖动或悬空 | 硬件/软件消抖 |
| 系统卡顿 | 回调中延时或打印 | 只置标志位 |
| 优先级混乱 | 未规划抢占关系 | 列出实时性和共享资源后设置 |

## 与其它章节的关系

- [[STM32体系结构与启动流程]] 解释向量表。
- [[GPIO与AFIO]] 提供 EXTI 输入。
- [[TIM定时器体系]] 提供周期性外设中断。
- [[DWT与调试计时]] 提供周期计数和微秒计时。

## 复习检查清单

- [ ] 能从向量表、IRQHandler、HAL IRQHandler 追踪到用户回调。
- [ ] 能说明 NVIC、EXTI 和 GPIO 在一次外部中断中的不同职责。
- [ ] 能解释抢占优先级、响应优先级以及“数值越小优先级越高”。
- [ ] 能在 CubeMX 中配置 EXTI 边沿、上下拉、NVIC 使能和优先级。
- [ ] 能用 `HAL_GetTick()` 写出可处理计数回绕的非阻塞节拍。
- [ ] 能说明为什么中断中不应长延时、大量打印或等待按键松手。
- [ ] 能按 GPIO 电平、EXTI 映射、NVIC、IRQHandler、回调和事件标志逐层排查。
