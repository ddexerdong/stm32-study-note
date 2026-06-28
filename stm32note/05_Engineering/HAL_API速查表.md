---
type: engineering
status: refactor
course: stm32-f103-hal
source:
  - local-notes
tags:
  - stm32
  - hal
  - api
---
# HAL API 速查表

> 用法：先从本表定位 API 家族，再进入对应能力笔记理解配置、启动和回调关系。

| 外设 | 初始化后常用启动/操作 | 回调或状态 | 详见 |
|---|---|---|---|
| GPIO | `HAL_GPIO_ReadPin`、`WritePin`、`TogglePin` | `HAL_GPIO_EXTI_Callback` | [[GPIO与AFIO]] |
| TIM Base | `HAL_TIM_Base_Start(_IT)` | `HAL_TIM_PeriodElapsedCallback` | [[TIM定时器体系]] |
| TIM PWM | `HAL_TIM_PWM_Start`、`__HAL_TIM_SET_COMPARE` | 通道状态 | [[TIM_PWM输出]] |
| TIM IC | `HAL_TIM_IC_Start(_IT)`、`ReadCapturedValue` | 捕获回调 | [[TIM输入捕获与编码器]] |
| ADC | `HAL_ADC_Start`、`PollForConversion`、`GetValue` | 转换完成回调 | [[ADC与模拟采样]] |
| DMA | 随外设 `Start_DMA` | Half/Complete | [[DMA与缓冲区]] |
| UART | `Transmit`、`Receive_IT`、`ReceiveToIdle_DMA` | Rx/Tx/Event 回调 | [[USART与串口协议]] |
| I2C | `Master_Transmit`、`Mem_Read/Write` | HAL 返回值 | [[I2C总线]] |
| SPI | `TransmitReceive` | HAL 返回值/完成回调 | [[SPI总线与Flash]] |
| CAN | `HAL_CAN_Start`、`AddTxMessage` | FIFO pending 回调 | [[CAN总线]] |

## 使用提醒

CubeMX 初始化只完成配置；很多外设还需要显式 Start。涉及 `_IT` 或 `_DMA` 时，必须同时检查 NVIC/DMA 配置、缓冲区生命周期和回调重启逻辑。

---

## 本节目标

- 按“初始化 -> 启动 -> 事件/回调 -> 停止”查找常用 HAL API。
- 看见 `_IT`、`_DMA` 时能联想到 NVIC、DMA 和缓冲区生命周期。
- 不把 API 表当作配置和硬件机制的替代品。

## 知识地图

| 后缀/形式 | 含义 | 返回特征 |
|---|---|---|
| 无后缀 | 常见阻塞或普通操作 | 调用期间可能等待 |
| `_IT` | 中断方式 | 快速返回，完成后回调 |
| `_DMA` | DMA 方式 | 快速返回，DMA 搬运 |
| `Start/Stop` | 启动/停止运行态 | 初始化后仍需显式调用 |
| Callback | HAL 弱回调入口 | 用户实现并区分句柄 |

## 常用 API 总表

| API | 所属外设 | 作用 | 常见使用位置 | 注意点 |
|---|---|---|---|---|
| `HAL_GPIO_Init` | GPIO | 配置模式/上下拉/速度 | BSP 初始化 | F103 引脚时钟先使能 |
| `HAL_GPIO_ReadPin` | GPIO | 读取输入 | 按键/DO | 先确认有效电平 |
| `HAL_GPIO_WritePin` | GPIO | 设置输出 | LED/控制脚 | 注意低有效 |
| `HAL_GPIO_EXTI_Callback` | EXTI | 用户边沿回调 | 事件入口 | 回调要短 |
| `HAL_TIM_Base_Start_IT` | TIM Base | 启动更新中断 | 周期任务 | NVIC 必须开启 |
| `HAL_TIM_PWM_Start` | TIM PWM | 启动 PWM 通道 | 初始化后 | 通道/AF/MOE |
| `__HAL_TIM_SET_COMPARE` | TIM PWM | 更新 CCR | 调光/调速 | 不超过周期范围 |
| `HAL_TIM_IC_Start_IT` | TIM IC | 启动捕获中断 | 脉宽测量 | 处理极性/溢出 |
| `HAL_TIM_Encoder_Start` | TIM Encoder | 启动编码器 | 初始化后 | 启动所需通道 |
| `HAL_UART_Transmit` | USART | 阻塞发送 | 日志/最小验证 | timeout 与长度 |
| `HAL_UART_Receive_IT` | USART | 中断接收 | 单字节/定长 | 回调中重启 |
| `HAL_UARTEx_ReceiveToIdle_DMA` | USART/DMA | 不定长接收 | 协议层 | Size/重启/缓冲区 |
| `HAL_ADCEx_Calibration_Start` | ADC | F1 ADC 校准 | 启动前 | 按 HAL 版本签名 |
| `HAL_ADC_Start_DMA` | ADC/DMA | 连续搬运结果 | 多通道采样 | Rank/长度/宽度 |
| `HAL_I2C_IsDeviceReady` | I2C | 探测 ACK | bring-up | 7 位地址左移 |
| `HAL_I2C_Mem_Read` | I2C | 读寄存器 | 传感器 | 地址宽度/长度 |
| `HAL_SPI_TransmitReceive` | SPI | 同步收发 | 寄存器/Flash | CS、模式、Dummy |
| `HAL_CAN_ConfigFilter` | CAN | 配过滤器 | Start 前 | ID/Mask/FIFO |
| `HAL_CAN_AddTxMessage` | CAN | 发送一帧 | 业务发送 | 邮箱和状态 |

## 本质理解

HAL API 只是状态机入口。调用成功不代表外设业务成功：UART 发送成功不代表对方理解协议，I2C ACK 不代表初始化完成，CAN 进邮箱不代表总线得到 ACK。

## 使用流程

```text
CubeMX 生成 Init
-> 检查 HAL 返回值
-> 调 Start / Receive / DMA
-> 等待状态或 Callback
-> 复制数据/置标志
-> 主循环处理业务
```

## 最小回调框架

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        rx_ready = 1;
        HAL_UART_Receive_IT(&huart1, &rx_byte, 1);
    }
}
```

## 调试方法

先检查函数返回值，再检查外设 State/ErrorCode、IRQHandler/DMA 句柄和回调。不要只在业务结果错误时盲目更换 API。

## 常见坑

| 坑 | 正确理解 |
|---|---|
| 只调用 Init | 很多外设还需要 Start/Receive |
| 忽略返回值 | HAL_BUSY/TIMEOUT 能直接提示层级 |
| 回调不分句柄 | 多实例会互相干扰 |
| 回调做重活 | ISR 应只复制/置标志 |
| `_DMA` 用局部数组 | DMA 期间内存必须持续有效 |

## 与其它章节的关系

每个 API 的配置和硬件机制应回到对应 Core、Peripheral 或 Communication 文件学习。

## 复习检查清单

- [ ] 能区分 Init、Start、操作和 Callback。
- [ ] 能解释阻塞、IT、DMA 的差异。
- [ ] 能检查 HAL_StatusTypeDef 和 ErrorCode。
- [ ] 能在回调中区分外设实例。
- [ ] 能说明 DMA buffer 生命周期。
- [ ] 能从 API 反查 CubeMX/NVIC/DMA 配置。
