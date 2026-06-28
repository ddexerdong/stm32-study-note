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
  - dma
  - buffer
---
# DMA 与缓冲区

> 能力定位：让硬件在外设和内存之间搬运数据，并正确管理缓冲区生命周期。
>
> 原始来源：[[04_串口与DMA]]、[[05_ADC模数转换]]。

## DMA 配置要素

| 要素 | 需要回答的问题 |
|---|---|
| 外设地址 | 数据来自/写向哪个外设寄存器 |
| 内存地址 | 缓冲区在哪里 |
| 方向 | 外设到内存，还是内存到外设 |
| 地址自增 | 外设地址通常固定，数组地址通常递增 |
| 数据宽度 | 字节、半字还是字 |
| 传输长度 | 本轮要搬多少个元素 |
| 模式 | Normal 一轮停止；Circular 循环刷新 |

## 回调与状态

Half Complete 可处理前半缓冲区，Complete 可处理整轮或后半缓冲区。中断中只标记“哪一段可用”，复杂处理放到主循环或任务。

## UART DMA

固定长度可使用普通 RX DMA；不定长数据常结合 USART IDLE。发送 DMA 的源缓冲区在完成前不能失效或被覆盖。

## ADC DMA

ADC 扫描顺序与数组下标按 Rank 对应。DMA 长度必须和通道/采样布局一致。

## Buffer 生命周期

长期 DMA buffer 应使用全局、静态或具有明确长期生命周期的内存。函数局部数组在函数返回后失效，不能作为持续 DMA 的可靠缓冲区。

环形缓冲区、双缓冲和生产者/消费者模型作为后续扩展主题。

## 关联知识

- [[USART与串口协议]]
- [[ADC与模拟采样]]
- [[串口协议解析]]

---

## 本节目标

- 能配置方向、自增、宽度、长度和模式
- 能管理 UART/ADC DMA buffer 生命周期
- 能解释 Half/Complete、IDLE、NDTR 和 ring buffer

## 知识地图

| 名词 | 工程含义 |
|---|---|
| DMA Channel | F103 固定请求映射的搬运通道 |
| Normal/Circular | 单轮停止/循环重装 |
| NDTR | 剩余传输元素数 |
| Half Complete | 半缓冲区可处理事件 |
| Ring Buffer | 生产者/消费者环形队列 |
| 生命周期 | 缓冲区在 DMA 使用期间必须持续有效 |

## 本质理解

DMA 只负责按配置搬数据，不理解帧、通道含义或缓冲区是否被业务覆盖。可靠系统要同时设计搬运参数、完成事件和数据消费速度。

为什么这样做：先建立硬件和数据流模型，再记 HAL API，才能在 API 变化或板卡变化时仍然定位问题。

## STM32F103 / HAL 中的实现方式

- F103 DMA 请求和通道映射由芯片固定，必须查 RM0008/CubeMX。
- 外设地址通常固定，内存数组通常自增；数据宽度应匹配外设寄存器和数组元素。
- UART IDLE + DMA 可用 NDTR 或 RxEvent Size 判断本轮长度；ADC Circular 可持续刷新数组。
- F103 通常没有 Cortex-M7 数据 Cache 一致性问题，但理解 cache 风险有助于迁移到高性能系列。

## CubeMX 配置要点

1. 在外设 DMA Settings 添加 TX/RX 或 ADC 请求。
2. 设置方向、Peripheral/Memory Increment、数据宽度和 Normal/Circular。
3. 需要回调时启用 DMA NVIC。
4. 确认生成的 `__HAL_LINKDMA` 和句柄关联。

## 常用 HAL API

### API 速查

| HAL 函数 / 宏 | 作用 | 常见位置 |
|---|---|---|
| `HAL_DMA_Init()` | 按句柄配置 DMA 通道 | CubeMX MSP 初始化 |
| `HAL_DMA_Start()` | 直接启动一次裸 DMA 搬运 | 特殊内存/外设搬运 |
| `HAL_DMA_Start_IT()` | 启动裸 DMA 并开启完成/错误中断 | 自定义 DMA 驱动 |
| `HAL_DMA_IRQHandler()` | 处理 DMA 中断并分发回调 | `stm32f1xx_it.c` DMA IRQHandler |
| `HAL_DMA_Abort()` | 中止正在运行的 DMA | 超时恢复、模式切换 |
| `HAL_DMA_GetState()` | 查询 DMA 当前状态 | 错误诊断、重启前 |
| `HAL_UART_Receive_DMA()` | 由 UART HAL 配置并启动固定长度 RX DMA | 串口接收启动 |
| `HAL_UARTEx_ReceiveToIdle_DMA()` | 接收至空闲事件或 buffer 满 | 不定长串口帧 |
| `HAL_ADC_Start_DMA()` | 由 ADC HAL 启动转换和 DMA | ADC 连续采样 |
| `__HAL_DMA_GET_COUNTER()` | 读取剩余传输计数 NDTR | 调试、部分接收长度计算 |

### 使用注意

| HAL 函数 / 宏 | 前置条件 | 参数重点 | 返回值 / 阻塞与常见坑 |
|---|---|---|---|
| `HAL_DMA_Init()` | DMA 时钟已使能、通道映射正确 | `DMA_HandleTypeDef *hdma` | 返回 `HAL_StatusTypeDef`；同步配置；一般由 CubeMX/外设 MSP 调用；方向、宽度、自增配错 |
| `HAL_DMA_Start()` | 地址、方向和通道均已配置 | 源地址、目标地址、长度 | 返回状态；非阻塞、无完成中断；新手误用裸 DMA；地址单位和数据宽度不匹配 |
| `HAL_DMA_Start_IT()` | NVIC 和回调已配置 | 源地址、目标地址、长度 | 返回状态；非阻塞；忘记 IRQHandler；多数 UART/ADC 场景应使用外设封装 API |
| `HAL_DMA_IRQHandler()` | DMA NVIC 已使能 | DMA 句柄 | `void`；中断上下文；IRQHandler 绑定错句柄导致外设回调不进入 |
| `HAL_DMA_Abort()` | DMA 句柄有效 | DMA 句柄 | 返回状态；同步中止；中止后未同步外设状态，立即重启仍可能 `HAL_BUSY` |
| `HAL_DMA_GetState()` | DMA 已初始化 | DMA 句柄 | 返回 `HAL_DMA_StateTypeDef`；非阻塞；只看 DMA 状态，不看外设 HAL 状态和错误码 |
| `HAL_UART_Receive_DMA()` | UART RX DMA 已关联 | UART 句柄、buffer、长度 | 返回状态；非阻塞；固定长度没收满不回完成回调；buffer 不能是局部变量 |
| `HAL_UARTEx_ReceiveToIdle_DMA()` | UART/DMA/IDLE 支持和中断链完整 | UART 句柄、buffer、容量 | 返回状态；非阻塞；`RxEventCallback` 的 Size 是本次有效长度，不是 C 字符串长度 |
| `HAL_ADC_Start_DMA()` | ADC DMA 已关联 | ADC 句柄、`uint32_t *` buffer、长度 | 返回状态；非阻塞；Rank 顺序与数组下标不一致；Normal/Circular 选择错误 |
| `__HAL_DMA_GET_COUNTER()` | DMA 通道正在或曾经运行 | DMA 句柄 | 返回剩余元素数；非阻塞；把剩余量当已接收量；并发读取时数值正在变化 |

### 典型调用顺序

```text
CubeMX 选择外设 DMA 请求、方向、宽度、Normal/Circular
-> MX_DMA_Init() 使能 DMA 时钟和 NVIC
-> MX_UART/ADC/SPI_Init() 关联 DMA 句柄
-> 定义全局或静态 buffer
-> 调用 HAL_UART_Receive_DMA / ReceiveToIdle_DMA / HAL_ADC_Start_DMA
-> DMA IRQHandler 调 HAL_DMA_IRQHandler()
-> 外设 Half/Cplt/RxEvent 回调只发布数据事件
```

### 什么时候用哪个函数？

| 场景 | 推荐函数 |
|---|---|
| UART 固定长度块 | `HAL_UART_Receive_DMA()` |
| UART 不定长帧 | `HAL_UARTEx_ReceiveToIdle_DMA()` + ring buffer/协议解析 |
| ADC 连续多通道 | `HAL_ADC_Start_DMA()` + Circular/双半区处理 |
| 特殊裸搬运 | 明确地址与回调模型后使用 `HAL_DMA_Start(_IT)()` |
| 故障恢复 | 停外设 DMA API，必要时 `HAL_DMA_Abort()`，检查状态后重启 |

## 最小实验 / 最小框架

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
static uint8_t rx_dma[128];

HAL_UARTEx_ReceiveToIdle_DMA(&huart1, rx_dma, sizeof(rx_dma));

void HAL_UARTEx_RxEventCallback(UART_HandleTypeDef *huart, uint16_t size)
{
    rx_size = size;
    rx_ready = 1;
}
```

## 调试方法

```text
检查 DMA 请求/通道映射
-> 检查方向和数据宽度
-> 检查 buffer 地址与长度
-> 检查 Start_DMA 返回值
-> 断点看 Half/Complete/RxEvent
-> 确认消费速度与重启时机
-> 制造长数据验证溢出保护
```

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| 回调不进 | NVIC 或 DMA 句柄未关联 | 检查 CubeMX 和 LINKDMA |
| 数据错位 | 宽度/自增设置错误 | 匹配寄存器和数组类型 |
| 局部数组失效 | 函数返回后生命周期结束 | 使用 static/global/长期对象 |
| 丢下一帧 | 重启时机过晚 | 回调快速切换缓冲并置标志 |
| 缓冲区溢出 | 生产快于消费 | ring buffer/流控/丢弃策略 |
| ADC 数组错 | DMA 长度和 Rank 不符 | 按扫描序列映射 |

## 与其它章节的关系

- [[USART与串口协议]] 使用 IDLE + DMA。
- [[ADC与模拟采样]] 使用 Circular DMA。
- [[串口协议解析]] 在 DMA 之上做帧处理。

## 复习检查清单

- [ ] 能否不用看代码说清本章数据流和硬件链路？
- [ ] 能否在 CubeMX 中完成最小配置并说明每个关键选项？
- [ ] 能否写出初始化、启动和回调/轮询的最小框架？
- [ ] 能否用原始电平、波形、返回值或寄存器证明外设在工作？
- [ ] 能否按“硬件 -> 时钟/配置 -> Start -> 回调 -> 业务”排查故障？
- [ ] 能否指出本章哪些参数必须按原理图、手册或实测确认？
