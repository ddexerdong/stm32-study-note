---
type: peripheral
status: refactor
course: stm32-f103-hal
source:
  - local-notes
  - st-reference-manual
  - st-hal-documentation
tags:
  - stm32
  - tim
  - input-capture
  - encoder
---
# TIM 输入捕获与编码器

> 能力定位：用 TIM 测量外部边沿、脉宽、频率和正交编码器位置。
>
> 原始来源：[[08_TIM通道捕获]]。

## 编码器模式

A/B 两相信号相差约 90 度。TIM 硬件根据相位关系自动让 CNT 加减，CPU 周期性读取 CNT 即可得到位置或增量。

关键点：输入滤波、计数范围、溢出、方向和采样周期。

## 输入捕获与超声波测距

输入捕获会在边沿到来时把 CNT 锁存到 CCR。先捕获上升沿，再捕获下降沿，两次 CCR 差值就是高电平持续计数。

```text
pulse_time = delta_count / counter_freq
distance = pulse_time * sound_speed / 2
```

声速、模块电平和计数频率必须按实际环境和硬件确认。

HC-SR04 一类模块可作为输入捕获练习：MCU 产生 TRIG，模块在 ECHO 输出与往返时间相关的脉宽。触发宽度、ECHO 电平和换算常数必须按具体模块手册与实测确认。

## HAL 主线

- `HAL_TIM_Encoder_Start()`
- `HAL_TIM_IC_Start()` / `HAL_TIM_IC_Start_IT()`
- `HAL_TIM_ReadCapturedValue()`
- `__HAL_TIM_SET_CAPTUREPOLARITY()`

## 调试方法

先看 A/B 相或 ECHO 原始波形，再看 CNT/CCR，最后验证换算公式。所有等待边沿流程都要有超时。

## 关联知识

- [[TIM定时器体系]]
- [[NVIC_EXTI_SysTick]]
- [[通用调试方法]]

---

## 本节目标

- 能解释编码器 A/B 相和 CNT 自动加减
- 能用 CCR 测量脉宽
- 能处理方向、溢出和超时

## 知识地图

| 名词 | 工程含义 |
|---|---|
| Encoder Mode | 用两路相位信号驱动 CNT 加减 |
| 输入捕获 | 边沿到来时把 CNT 锁存到 CCR |
| 极性切换 | 选择上升沿或下降沿捕获 |
| 计数溢出 | CNT 越过 ARR 后回绕 |
| 数字滤波 | 抑制输入毛刺 |

## 本质理解

编码器模式让硬件完成“看相位并计数”；输入捕获让硬件完成“在边沿瞬间拍下时间”。CPU 只读取结果，不需要高速轮询每个边沿。

为什么这样做：先建立硬件和数据流模型，再记 HAL API，才能在 API 变化或板卡变化时仍然定位问题。

## STM32F103 / HAL 中的实现方式

- F103 TIM 编码器接口使用 CH1/CH2 输入，CNT 方向反映相位关系。
- 输入捕获把 CNT 复制到 CCRx；两次捕获差值需考虑计数器回绕。
- 超声波 ECHO 可能超过一个计数周期，应选择足够计数范围或记录溢出次数。

## CubeMX 配置要点

1. 编码器：选择 Encoder Mode，配置两通道极性/滤波/ARR。
2. 输入捕获：选择 Input Capture direct mode，配置边沿和滤波。
3. 需要回调时启用 TIM NVIC。
4. 按 ECHO 电平范围设计分压或电平转换。

## 常用 HAL API

| API / 对象 | 作用 | 常见位置 |
|---|---|---|
| `HAL_TIM_Encoder_Start` | 启动编码器通道 | 初始化后 |
| `__HAL_TIM_GET_COUNTER` | 读取位置计数 | 周期任务 |
| `HAL_TIM_IC_Start_IT` | 启动捕获中断 | 脉宽测量 |
| `HAL_TIM_ReadCapturedValue` | 读取 CCR | 捕获回调 |
| `__HAL_TIM_SET_CAPTUREPOLARITY` | 切换捕获边沿 | 双边沿流程 |

## 最小实验 / 最小框架

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
uint32_t now = __HAL_TIM_GET_COUNTER(&htim3);
int32_t delta = (int16_t)(now - last_count);
last_count = now;

/* 输入捕获回调中读取 CCR，并按计数器范围处理差值。 */
```

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| 编码器为 0 | 未 Start 或引脚错 | 启动并检查 A/B 波形 |
| 方向反 | A/B 相交换 | 交换接线或软件取反 |
| 计数跳动 | 毛刺/机械抖动 | 使用输入滤波并改善接线 |
| 捕获回调不进 | NVIC/Start_IT 缺失 | 检查中断链 |
| 距离偶尔巨大 | 未处理溢出或超时 | 扩展计数范围并拒绝无效帧 |

## 与其它章节的关系

- [[TIM定时器体系]] 提供 CNT/CCR。
- [[NVIC_EXTI_SysTick]] 解释捕获回调。
- [[通用调试方法]] 说明波形优先调试。

## 复习检查清单

- [ ] 能否不用看代码说清本章数据流和硬件链路？
- [ ] 能否在 CubeMX 中完成最小配置并说明每个关键选项？
- [ ] 能否写出初始化、启动和回调/轮询的最小框架？
- [ ] 能否用原始电平、波形、返回值或寄存器证明外设在工作？
- [ ] 能否按“硬件 -> 时钟/配置 -> Start -> 回调 -> 业务”排查故障？
- [ ] 能否指出本章哪些参数必须按原理图、手册或实测确认？
