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
  - dwt
  - debug
---
# DWT 与调试计时

> 能力定位：用 CPU 周期计数测量程序耗时，并提供微秒级计时基础。
>
> 原始来源：[[10_DWT与单总线传感器]]。

## 核心概念

DWT 是 Cortex-M 调试组件，`CYCCNT` 可记录 CPU 周期数。已知 `SystemCoreClock` 后，可以把周期差换算成时间。

```text
time_us = cycles * 1000000 / SystemCoreClock
```

## 工程用途

- 比较两种算法的执行时间。
- 定位中断或临界代码耗时。
- 构建短时间微秒延时。
- 观察优化等级对运行时间的影响。

与 `HAL_Delay()` 的区别：`HAL_Delay()` 依赖毫秒 tick 并阻塞等待，适合普通毫秒级流程；DWT 直接读取 CPU 周期，适合短时间测量和微秒级等待，但同样会占用 CPU。

## 使用边界

- DWT 依赖内核和调试相关寄存器配置。
- 微秒忙等会占用 CPU，不适合长时间等待。
- 系统时钟变化后必须更新换算依据。
- 单步调试会干扰被测时间，应使用断点或连续运行测量。

> 视频待核对：课程的 DWT 初始化封装、Debug 观察步骤和不同编译环境差异。

## 关联知识

- [[RCC时钟树]]
- [[DHT11单总线温湿度]]
- [[通用调试方法]]

---

## 本节目标

- 能初始化 CYCCNT 并换算微秒
- 能测量函数运行时间
- 能判断 DWT 忙等是否适合当前场景

## 知识地图

| 名词 | 工程含义 |
|---|---|
| DWT | Cortex-M 调试与跟踪组件 |
| CYCCNT | CPU 周期计数器 |
| SystemCoreClock | 周期到时间换算基准 |
| 忙等 | CPU 持续轮询直到时间到 |
| 测量扰动 | 断点/单步对运行时间的影响 |

## 本质理解

DWT 给出的是 CPU 走过多少周期，而不是抽象的“延时”。这使它适合短时间测量和微秒级等待，但它仍占用 CPU，不能替代定时器、DMA 或任务调度。

为什么这样做：先建立硬件和数据流模型，再记 HAL API，才能在 API 变化或板卡变化时仍然定位问题。

## STM32F103 / HAL 中的实现方式

- Cortex-M3 的 DWT 需开启跟踪和 CYCCNT 计数；不同内核、低功耗和调试配置支持情况需核对。
- 时间差应使用无符号减法，允许计数器回绕。
- DHT11 等微秒时序可使用 DWT；WS2813E 的纳秒窗口通常更适合 TIM/DMA 或经过严格验证的实现。

## CubeMX 配置要点

1. DWT 不需要 CubeMX 外设实例。
2. 先确认 `SystemCoreClock` 与真实主频一致。
3. Debug 时在 start/stop/delta 变量上观察，不要把单步停顿算入测量。

## 常用 HAL API

| API / 对象 | 作用 | 常见位置 |
|---|---|---|
| `CoreDebug->DEMCR` | 开启跟踪组件 | DWT 初始化 |
| `DWT->CTRL` | 使能 CYCCNT | DWT 初始化 |
| `DWT->CYCCNT` | 读取周期数 | 时间测量 |
| `SystemCoreClock` | 换算周期 | 系统时钟变量 |

## 最小实验 / 最小框架

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
uint32_t start = DWT->CYCCNT;
Target_Function();
uint32_t cycles = DWT->CYCCNT - start;
float time_us = (float)cycles * 1000000.0f / SystemCoreClock;
```

## 调试方法

```text
确认 DWT 计数器在递增
-> 确认 SystemCoreClock
-> 连续运行到断点而非逐行单步
-> 重复测量并比较波动
-> 关闭不相关中断验证干扰来源
```

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| CYCCNT 一直为 0 | 跟踪或计数未使能 | 检查 DEMCR/CTRL |
| 微秒换算错误 | 主频变量错误 | 更新 SystemCoreClock |
| 测量时间巨大 | 断点停顿被计入 | 改为连续运行 |
| 系统响应变差 | 长时间忙等 | 改用 TIM/中断 |
| 低功耗后异常 | 计数支持或时钟变化 | 按内核/低功耗状态核对 |

## 与其它章节的关系

- [[RCC时钟树]] 提供主频依据。
- [[DHT11单总线温湿度]] 使用微秒级时序。
- [[通用调试方法]] 说明 Debug 与波形工具选择。

## 复习检查清单

- [ ] 能否不用看代码说清本章数据流和硬件链路？
- [ ] 能否在 CubeMX 中完成最小配置并说明每个关键选项？
- [ ] 能否写出初始化、启动和回调/轮询的最小框架？
- [ ] 能否用原始电平、波形、返回值或寄存器证明外设在工作？
- [ ] 能否按“硬件 -> 时钟/配置 -> Start -> 回调 -> 业务”排查故障？
- [ ] 能否指出本章哪些参数必须按原理图、手册或实测确认？
