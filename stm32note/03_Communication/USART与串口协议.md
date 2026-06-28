---
type: bus
status: refactor
course: stm32-f103-hal
source:
  - local-notes
  - st-reference-manual
  - st-hal-documentation
tags:
  - stm32
  - usart
  - uart
  - protocol
---
# USART 与串口协议

> 能力定位：从物理接线到字节流再到应用协议，完整定位串口问题。
>
> 原始来源：[[04_串口与DMA]]。

## 串口工程三层模型

| 层 | 核心问题 |
|---|---|
| 物理层 | TX/RX/GND、电平兼容、是否交叉连接 |
| 字节层 | 波特率、数据位、校验位、停止位（如 8N1） |
| 协议层 | 帧头、长度、命令、字段、校验、超时 |

## 接收方式

- 阻塞：简单但占用 CPU。
- 中断：适合少量或定长数据，每轮完成后通常要重新启动。
- DMA：适合连续搬运。
- IDLE + DMA：适合不定长数据帧，帧结束仍需协议层确认。

## 字节流与字符串

UART 传输的是字节。字符串只是以某种编码组织的字节序列；二进制模块帧不能直接按 C 字符串处理。

ASCII 是字符到字节值的一种编码；串口工具的“ASCII 发送”和“HEX 发送”只是输入表达方式不同。协议帧中的 payload 是业务载荷，可以是文本，也可以是包含零字节的二进制数据。

## 常用 HAL API

- `HAL_UART_Transmit` / `HAL_UART_Receive`
- `HAL_UART_Receive_IT`
- `HAL_UART_Transmit_DMA` / `HAL_UART_Receive_DMA`
- `HAL_UARTEx_ReceiveToIdle_DMA`

## 工程主题

`printf` 重定向用于调试；多串口需要独立句柄和缓冲区；重映射后要同步检查 GPIO 复用和实际接线。

## 关联知识

- [[DMA与缓冲区]]
- [[串口协议解析]]
- [[RS485物理层]]
- [[ESP8266_AT指令]]

---

## 本节目标

- 能从物理层、字节层、协议层排查串口
- 能使用阻塞/中断/DMA/IDLE 接收
- 能设计包含长度、命令、校验和超时的帧

## 知识地图

| 名词 | 工程含义 |
|---|---|
| UART/USART | 异步串口/具同步能力的通用串口 |
| 波特率 | 每秒码元数 |
| 8N1 | 8 数据位、无校验、1 停止位 |
| TTL 电平 | MCU 逻辑电平，不等于 RS232 |
| IDLE | 接收线空闲一个字符时间 |
| 帧 | 协议层定义的完整消息 |

## 本质理解

串口硬件只产生和接收字节。字符串、命令和帧都是软件解释。把物理层、电气/帧格式和协议解析分开，才能避免“收到了字节却业务不工作”的混乱。

为什么这样做：先建立硬件和数据流模型，再记 HAL API，才能在 API 变化或板卡变化时仍然定位问题。

## STM32F103 / HAL 中的实现方式

- F103 USART 引脚通过 GPIO 复用和 AFIO 重映射连接，TX/RX 要交叉并共地。
- 阻塞 API 适合最小验证；中断适合少量定长；DMA 适合连续数据；IDLE 只提供空闲事件，不等于协议帧完整。
- 多串口共享 HAL 回调时按 `huart->Instance` 分流，每个串口独立启动接收。

## CubeMX 配置要点

1. 选择 USART Asynchronous。
2. 配置波特率、Word Length、Parity、Stop Bits。
3. 确认 TX/RX 引脚和重映射。
4. 中断模式启用 USART NVIC；DMA 模式配置 RX/TX DMA 和 DMA NVIC。
5. 生成后分别启动每个串口接收。

## 最小实验 / 最小框架

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
static uint8_t rx_byte;

HAL_UART_Receive_IT(&huart1, &rx_byte, 1);

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        command_byte = rx_byte;
        command_ready = 1;
        HAL_UART_Receive_IT(&huart1, &rx_byte, 1);
    }
}
```

## 调试方法

```text
确认 3.3V TTL 与共地
-> 确认 TX/RX 交叉
-> 核对波特率和 8N1
-> 阻塞发送固定字符串
-> 观察原始 HEX 字节
-> 再启用中断/DMA
-> 最后调帧边界和协议
```

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| 无数据 | TX/RX 未交叉或未共地 | 先查物理连接 |
| 乱码 | 波特率/时钟/电平错误 | 核对 PCLK 和串口助手 |
| 只收一次 | 回调未重启 | 再次调用 Receive_IT |
| 字符串越界 | 接收缓冲无 `\0` | 按长度处理并预留终止符 |
| 半包粘包 | 把一次回调当一帧 | 使用长度/状态机/校验 |
| IDLE 异常 | 标志/重启/Size 处理错 | 按 HAL 版本核对 RxEvent 流程 |

## 与其它章节的关系

- [[DMA与缓冲区]] 解释数据搬运和生命周期。
- [[串口协议解析]] 处理半包、粘包和帧校验。
- [[RS485物理层]] 在 UART 外增加差分收发器。

## 复习检查清单

- [ ] 能否不用看代码说清本章数据流和硬件链路？
- [ ] 能否在 CubeMX 中完成最小配置并说明每个关键选项？
- [ ] 能否写出初始化、启动和回调/轮询的最小框架？
- [ ] 能否用原始电平、波形、返回值或寄存器证明外设在工作？
- [ ] 能否按“硬件 -> 时钟/配置 -> Start -> 回调 -> 业务”排查故障？
- [ ] 能否指出本章哪些参数必须按原理图、手册或实测确认？
