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

## 中断本质

中断让 CPU 暂停当前流程，跳到异常入口处理事件，再返回原流程。ISR 应尽快完成，复杂业务用标志位交给主循环或任务处理。

## 三个角色

| 角色 | 作用 |
|---|---|
| NVIC | 管理中断使能、优先级和嵌套 |
| EXTI | 把 GPIO 边沿或内部事件转换成中断请求 |
| SysTick | Cortex-M 内核定时器，HAL 常用作毫秒时基 |

## 优先级

抢占优先级决定能否打断其他中断；响应优先级决定同级请求的先后顺序。优先级不是越高越好，应按实时性和共享资源设计。

## HAL 回调链

```text
硬件事件 -> IRQHandler -> HAL_xxx_IRQHandler -> HAL 回调 -> 设置标志位
```

HAL 回调函数是驱动在统一 IRQHandler 中完成状态处理后留给用户的扩展入口。回调里应先判断外设实例或 Pin，再记录事件并尽快返回。

## 中断中不要做的事

- 长时间阻塞或 `HAL_Delay()`。
- 大量 `printf`。
- 复杂协议解析。
- 不受控地访问共享缓冲区。

共享变量通常需要 `volatile`，并考虑原子性和临界区。

## 关联知识

- [[GPIO与AFIO]]
- [[TIM定时器体系]]
- [[DWT与调试计时]]

---

## 本节目标

- 能说明向量表、NVIC、EXTI 和回调链
- 能配置按键 EXTI 最小实验
- 能区分 SysTick、TIM 中断和主循环任务

## 知识地图

| 名词 | 工程含义 |
|---|---|
| NVIC | 使能、挂起、优先级和嵌套管理 |
| EXTI | 外部/内部事件线路 |
| 向量表 | 异常号到入口函数的映射 |
| SysTick | Cortex-M3 24 位系统定时器 |
| HAL tick | HAL 默认毫秒时基 |

## 本质理解

中断不是另一个线程，而是打断当前上下文执行的特殊控制流。越长的 ISR 越容易阻塞其他事件、破坏实时性，因此回调应采集状态并快速退出。

为什么这样做：先建立硬件和数据流模型，再记 HAL API，才能在 API 变化或板卡变化时仍然定位问题。

## STM32F103 / HAL 中的实现方式

- GPIO 边沿先进入 EXTI 线路，再由 NVIC 调度 IRQHandler；HAL IRQHandler 清标志并调用用户回调。
- SysTick 周期中断累加 HAL tick，`HAL_GetTick()` 读取计数，`HAL_Delay()` 轮询时间差。
- 抢占优先级决定能否嵌套，响应优先级决定同抢占级请求的排队顺序。

## CubeMX 配置要点

1. 把按键脚配置为 GPIO_EXTI，并选择上升沿/下降沿。
2. 设置正确上下拉。
3. 在 NVIC 勾选对应 EXTI 线并配置优先级。
4. 生成后在 `HAL_GPIO_EXTI_Callback` 判断 Pin 并置标志位。

## 常用 HAL API

| API / 对象 | 作用 | 常见位置 |
|---|---|---|
| `HAL_NVIC_SetPriority` | 设置 NVIC 优先级 | 初始化 |
| `HAL_NVIC_EnableIRQ` | 使能 IRQ | 初始化 |
| `HAL_GPIO_EXTI_IRQHandler` | HAL EXTI 处理入口 | IRQHandler |
| `HAL_GPIO_EXTI_Callback` | 用户回调 | 业务事件入口 |
| `HAL_GetTick` | 读取毫秒 tick | 超时/消抖 |
| `HAL_Delay` | 阻塞延时 | 非 ISR 场景 |

## 最小实验 / 最小框架

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
volatile uint8_t key_event;

void HAL_GPIO_EXTI_Callback(uint16_t pin)
{
    if (pin == KEY_Pin)
    {
        key_event = 1;
    }
}
```

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

## 复习检查清单

- [ ] 能否不用看代码说清本章数据流和硬件链路？
- [ ] 能否在 CubeMX 中完成最小配置并说明每个关键选项？
- [ ] 能否写出初始化、启动和回调/轮询的最小框架？
- [ ] 能否用原始电平、波形、返回值或寄存器证明外设在工作？
- [ ] 能否按“硬件 -> 时钟/配置 -> Start -> 回调 -> 业务”排查故障？
- [ ] 能否指出本章哪些参数必须按原理图、手册或实测确认？
