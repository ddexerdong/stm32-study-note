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
  - adc
  - trgo
---
# TIM 触发 ADC

> 能力定位：用硬件事件建立稳定采样节拍，减少软件延时抖动。
>
> 原始来源：[[07_TIM触发ADC采集]]。

## 三个角色

| 角色 | 作用 |
|---|---|
| TIM | 按固定频率产生 Update Event |
| TRGO | 把 TIM 更新事件输出给其他外设 |
| ADC | 把外部触发作为转换启动条件 |

## 启动顺序

```text
配置 TIM TRGO
-> ADC 选择对应外部触发源
-> 先启动 ADC 等待触发
-> 再启动 TIM
-> ADC 按硬件节拍采样
```

## 工程意义

适合固定频率采样、滤波、波形采集和控制环。与 `HAL_Delay()` 轮询相比，采样间隔更稳定，也更容易配合 [[DMA与缓冲区]]。

## 常见坑

- TIM 没有真正启动。
- TRGO 不是 Update Event。
- ADC 触发源选错。
- Continuous 模式和外部触发混用。
- ADC/TIM 启动顺序反了。

## 关联知识

- [[TIM定时器体系]]
- [[ADC与模拟采样]]
- [[传感器采集与显示]]

---

## 本节目标

- 能区分软件触发和硬件触发 ADC
- 能配置 TIM TRGO -> ADC 外部触发链
- 能按正确顺序启动并定位 Poll 卡死

## 知识地图

| 名词 | 工程含义 |
|---|---|
| TRGO | 定时器向其他外设输出的触发信号 |
| Update Event | 常用 TRGO 来源 |
| External Trigger | ADC 转换启动条件 |
| 采样抖动 | 触发时间相对理想周期的偏差 |
| 内部温度传感器 | 片上 ADC 通道案例 |

## 本质理解

软件循环触发会受中断和业务耗时影响；TIM 硬件触发把采样时刻交给外设事件链，CPU 只处理结果，因此节拍更稳定，也便于 DMA 连续采样。

为什么这样做：先建立硬件和数据流模型，再记 HAL API，才能在 API 变化或板卡变化时仍然定位问题。

## STM32F103 / HAL 中的实现方式

- TIM Update Event 可通过 TRGO 输出，ADC 选择匹配的 External Trigger。
- 单次外部触发模式下，每个触发产生一轮转换；Continuous 设置必须和目标行为一致。
- 内部温度传感器适合验证链路，但换算只用于粗略芯片温度参考。

## CubeMX 配置要点

1. 配置 TIM PSC/ARR 形成采样频率。
2. Master Mode Trigger Output 选择 Update Event。
3. ADC 选择对应 Timer Trigger Out，关闭不需要的 Continuous。
4. 配置通道、采样时间和 DMA/中断。
5. 先启动 ADC，再启动 TIM。

## 常用 HAL API

| API / 对象 | 作用 | 常见位置 |
|---|---|---|
| `HAL_ADC_Start` | 让 ADC 等待外部触发 | 启动阶段 |
| `HAL_ADC_Start_DMA` | 外部触发结果搬入数组 | 连续采样 |
| `HAL_TIM_Base_Start` | 启动触发源 | ADC 启动后 |
| `HAL_ADC_PollForConversion` | 等待结果 | 低速验证 |

## 最小实验 / 最小框架

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
HAL_ADC_Start(&hadc1);
HAL_TIM_Base_Start(&htim3);

if (HAL_ADC_PollForConversion(&hadc1, 100) == HAL_OK)
{
    adc_raw = HAL_ADC_GetValue(&hadc1);
}
```

## 调试方法

```text
单独确认 TIM 周期
-> 确认 TRGO=Update
-> 确认 ADC 外部触发源
-> 先启动 ADC 再 TIM
-> 观察 ADC 状态/回调/DMA
-> 最后检查温度或电压换算
```

## 与其它章节的关系

- [[TIM定时器体系]] 生成 Update Event。
- [[ADC与模拟采样]] 解释转换和采样时间。
- [[DMA与缓冲区]] 负责连续数据搬运。

## 复习检查清单

- [ ] 能否不用看代码说清本章数据流和硬件链路？
- [ ] 能否在 CubeMX 中完成最小配置并说明每个关键选项？
- [ ] 能否写出初始化、启动和回调/轮询的最小框架？
- [ ] 能否用原始电平、波形、返回值或寄存器证明外设在工作？
- [ ] 能否按“硬件 -> 时钟/配置 -> Start -> 回调 -> 业务”排查故障？
- [ ] 能否指出本章哪些参数必须按原理图、手册或实测确认？
