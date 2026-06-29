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
---
## 本节目标

- 能从物理层、字节层、协议层排查串口
- 能使用阻塞/中断/DMA/IDLE 接收
- 能设计包含长度、命令、校验和超时的帧

## 核心概念

### 知识地图

| 名词 | 工程含义 |
|---|---|
| UART/USART | 异步串口/具同步能力的通用串口 |
| 波特率 | 每秒码元数 |
| 8N1 | 8 数据位、无校验、1 停止位 |
| TTL 电平 | MCU 逻辑电平，不等于 RS232 |
| IDLE | 接收线空闲一个字符时间 |
| 帧 | 协议层定义的完整消息 |

### 串口工程三层模型

| 层 | 核心问题 |
|---|---|
| 物理层 | TX/RX/GND、电平兼容、是否交叉连接 |
| 字节层 | 波特率、数据位、校验位、停止位（如 8N1） |
| 协议层 | 帧头、长度、命令、字段、校验、超时 |

### 接收方式

- 阻塞：简单但占用 CPU。
- 中断：适合少量或定长数据，每轮完成后通常要重新启动。
- DMA：适合连续搬运。
- IDLE + DMA：适合不定长数据帧，帧结束仍需协议层确认。

#### 串口接收模型图

```mermaid
flowchart LR
    A[RX 连续字节流] --> B[USART 接收]
    B --> C[IT / DMA 缓冲区]
    C --> D[ring buffer]
    D --> E[帧解析状态机]
    E --> F[完整业务消息]
```

### 字节流与字符串

UART 传输的是字节。字符串只是以某种编码组织的字节序列；二进制模块帧不能直接按 C 字符串处理。

ASCII 是字符到字节值的一种编码；串口工具的“ASCII 发送”和“HEX 发送”只是输入表达方式不同。协议帧中的 payload 是业务载荷，可以是文本，也可以是包含零字节的二进制数据。

### 工程主题

`printf` 重定向用于调试；多串口需要独立句柄和缓冲区；重映射后要同步检查 GPIO 复用和实际接线。

## 本质理解

串口硬件只产生和接收字节。字符串、命令和帧都是软件解释。把物理层、电气/帧格式和协议解析分开，才能避免“收到了字节却业务不工作”的混乱。

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

## 常用 HAL API

### API 速查

| HAL 函数 / 宏 | 作用 | 常见位置 |
|---|---|---|
| `HAL_UART_Transmit()` | 轮询发送指定长度字节 | 调试打印、短命令发送 |
| `HAL_UART_Receive()` | 轮询接收固定长度字节 | 最小通信验证 |
| `HAL_UART_Transmit_IT()` | 中断方式发送一块数据 | 不希望主循环等待的短消息 |
| `HAL_UART_Receive_IT()` | 中断方式接收固定长度数据 | 单字节命令、固定小帧 |
| `HAL_UART_Transmit_DMA()` | DMA 发送一块数据 | 大块或低 CPU 占用发送 |
| `HAL_UART_Receive_DMA()` | DMA 接收固定长度数据 | 连续固定块接收 |
| `HAL_UARTEx_ReceiveToIdle_DMA()` | 接收至空闲事件或 buffer 满 | 不定长命令/协议帧 |
| `HAL_UART_RxCpltCallback()` | 固定长度 IT/DMA 接收完成通知 | 用户回调 |
| `HAL_UARTEx_RxEventCallback()` | Receive-to-Idle 接收事件通知 | 用户回调 |
| `HAL_UART_ErrorCallback()` | 接收 UART 错误事件 | 错误记录与恢复入口 |
| `HAL_UART_DMAStop()` | 停止 UART 关联的 DMA 传输 | 超时、切换模式、重启前 |

### 使用注意

| HAL 函数 / 宏 | 前置条件 | 参数重点 | 返回值 / 阻塞与常见坑 |
|---|---|---|---|
| `HAL_UART_Transmit()` | UART 已初始化 | buffer、字节数、Timeout | 返回 `HAL_StatusTypeDef`；阻塞到完成或超时；Timeout 是等待上限；不要在高频回调中大量发送 |
| `HAL_UART_Receive()` | RX 接线与帧格式正确 | buffer、长度、Timeout | 返回状态；阻塞到收满或超时；不定长数据会频繁超时；接收字符串需自行补 `\0` |
| `HAL_UART_Transmit_IT()` | UART NVIC 已启用 | buffer 在完成前保持有效 | 返回状态；非阻塞；发送未完成就修改/释放 buffer；忽略 `HAL_BUSY` |
| `HAL_UART_Receive_IT()` | UART NVIC 已启用 | buffer、期望长度 | 返回状态；非阻塞；完成后通常要再次调用；收不到指定长度就不进完成回调 |
| `HAL_UART_Transmit_DMA()` | TX DMA 已关联并开中断 | buffer、长度 | 返回状态；非阻塞；buffer 在完成前被改写；错误恢复时未停 DMA |
| `HAL_UART_Receive_DMA()` | RX DMA 已关联 | 长期有效 buffer、长度 | 返回状态；非阻塞；固定长度未收满不回完成回调；Normal 模式结束后需重启 |
| `HAL_UARTEx_ReceiveToIdle_DMA()` | RX DMA 与 UART 中断链完整 | buffer 容量，不是期望帧长 | 返回状态；非阻塞；IDLE 不是协议帧边界；应使用 `RxEventCallback` 的 Size |
| `HAL_UART_RxCpltCallback()` | 使用固定长度 Receive IT/DMA | UART 句柄 | `void`；中断上下文；和 `RxEventCallback` 混淆；回调后忘记重新启动接收 |
| `HAL_UARTEx_RxEventCallback()` | 使用 `ReceiveToIdle_IT/DMA` | UART 句柄、本次有效 `Size` | `void`；中断上下文；忽略半满/满/IDLE 等事件差异；直接用 `strlen` 处理二进制 |
| `HAL_UART_ErrorCallback()` | UART 中断已启用 | UART 句柄，可进一步取错误码 | `void`；中断上下文；只清日志不重建接收链；在回调中阻塞重试 |
| `HAL_UART_DMAStop()` | UART DMA 正在或可能正在运行 | UART 句柄 | 返回状态；同步停止；停止后未重置协议/buffer 状态；无条件频繁调用 |

### 典型调用顺序

```text
固定长度 IT：MX_USARTx_UART_Init()
-> 定义静态/全局接收 buffer
-> HAL_UART_Receive_IT()
-> HAL_UART_RxCpltCallback()
-> 复制/发布数据
-> 再次 HAL_UART_Receive_IT()

不定长 DMA：MX_DMA_Init()
-> MX_USARTx_UART_Init()
-> HAL_UARTEx_ReceiveToIdle_DMA()
-> HAL_UARTEx_RxEventCallback(huart, Size)
-> 将 Size 个字节送入 ring buffer/解析器
-> 按所用 DMA 模式确认是否需要重启
```

### 什么时候用哪个函数？

| 场景 | 推荐函数 |
|---|---|
| 简单调试打印 | `HAL_UART_Transmit()` 或基于它的 `printf` 重定向 |
| 最小阻塞收发验证 | `HAL_UART_Transmit()` / `HAL_UART_Receive()` |
| 接收固定长度少量数据 | `HAL_UART_Receive_IT()` |
| 固定长度大块数据 | `HAL_UART_Receive_DMA()` |
| 接收不定长数据帧 | `HAL_UARTEx_ReceiveToIdle_DMA()` + ring buffer/状态机 |
| 发生 DMA 接收错误需重建链路 | 记录错误 -> `HAL_UART_DMAStop()` -> 清业务状态 -> 重新启动 |

> CubeF1 版本提醒：旧版 HAL 可能没有 `HAL_UARTEx_ReceiveToIdle_DMA()`。使用前先查本地 `stm32f1xx_hal_uart.h`；没有该接口时，用普通 DMA + IDLE 中断或中断接收状态机替代。

## 最小实验 / 最小框架

> 代码性质：示例框架，用于理解调用顺序，不能保证直接编译。
> 验证状态：未上板验证，需按 CubeMX 变量名、开发板原理图和课程源码核对。

```c
static uint8_t rx_byte;
static uint8_t rx_dma[UART_DMA_CAPACITY + 1U];
static volatile uint8_t rx_byte_ready;
static volatile uint16_t rx_dma_size;

static HAL_StatusTypeDef UART_SendBlocking(const uint8_t *data, uint16_t length)
{
    return HAL_UART_Transmit(&huart1, data, length, UART_TX_TIMEOUT_MS);
}

/* printf 重定向方式随编译器/运行库而异。 */
int __io_putchar(int ch)
{
    uint8_t byte = (uint8_t)ch;
    return (HAL_UART_Transmit(&huart1, &byte, 1U,
                              UART_TX_TIMEOUT_MS) == HAL_OK) ? ch : -1;
}

static uint8_t UART_StartReceive(void)
{
#if UART_HAL_HAS_RECEIVE_TO_IDLE
    return HAL_UARTEx_ReceiveToIdle_DMA(&huart1, rx_dma,
                                        UART_DMA_CAPACITY) == HAL_OK;
#else
    if (HAL_UART_Receive_DMA(&huart1, rx_dma, UART_DMA_CAPACITY) != HAL_OK)
    {
        return 0U;
    }
    __HAL_UART_ENABLE_IT(&huart1, UART_IT_IDLE);
    return 1U;
#endif
}

static void UART_ParseHexFrame(const uint8_t *data, uint16_t length)
{
    /* 二进制帧必须显式传长度；帧头/长度/校验交给协议解析器。 */
    Protocol_PushBytes(data, length);
}

void HAL_UARTEx_RxEventCallback(UART_HandleTypeDef *huart, uint16_t size)
{
    if ((huart->Instance == USART1) && (size <= UART_DMA_CAPACITY))
    {
        rx_dma[size] = '\0'; /* 仅便于文本调试；二进制仍使用 size。 */
        rx_dma_size = size;
        UART_ParseHexFrame(rx_dma, size);
    }
}

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        rx_byte_ready = 1U;
        (void)HAL_UART_Receive_IT(&huart1, &rx_byte, 1U);
    }
}

#if !UART_HAL_HAS_RECEIVE_TO_IDLE
/* 在 USARTx_IRQHandler 调 HAL_UART_IRQHandler() 前后接入，位置按工程核对。 */
static void UART_LegacyIdleHandler(void)
{
    if ((__HAL_UART_GET_FLAG(&huart1, UART_FLAG_IDLE) != RESET) &&
        (__HAL_UART_GET_IT_SOURCE(&huart1, UART_IT_IDLE) != RESET))
    {
        __HAL_UART_CLEAR_IDLEFLAG(&huart1);
        uint16_t size = UART_DMA_CAPACITY -
                        (uint16_t)__HAL_DMA_GET_COUNTER(huart1.hdmarx);
        (void)HAL_UART_DMAStop(&huart1);
        UART_ParseHexFrame(rx_dma, size);
        (void)UART_StartReceive();
    }
}
#endif

if (HAL_UART_Receive_IT(&huart1, &rx_byte, 1U) != HAL_OK ||
    !UART_StartReceive())
{
    Error_Handler();
}

static const uint8_t hello[] = "uart ready\r\n";
(void)UART_SendBlocking(hello, sizeof(hello) - 1U);
```

`UART_HAL_HAS_RECEIVE_TO_IDLE` 由工程按本地 `stm32f1xx_hal_uart.h` 定义。旧版 HAL 替代框架的 IRQ 接入位置、清 IDLE 顺序和 DMA 重启必须结合本地驱动源码验证。

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
- [[ESP8266_AT指令]] 使用串口传输 AT 命令与异步响应。

## 复习检查清单

- [ ] 能区分 UART 电平、波特率/帧格式和上层协议三个层次。
- [ ] 能按阻塞、IT、DMA、IDLE + DMA 的代价选择接收方式。
- [ ] 能区分 `RxCpltCallback` 与 `RxEventCallback` 的触发条件和有效长度。
- [ ] 能在旧版 CubeF1 缺少 `HAL_UARTEx_ReceiveToIdle_DMA()` 时选择替代方案。
- [ ] 能为文本接收预留 `\0`，并对二进制 HEX 帧始终使用显式长度。
- [ ] 能说明一次回调不等于一帧，并用状态机处理半包、粘包和错误恢复。
- [ ] 能按供电/共地、TX/RX、波特率、原始字节、接收方式和协议逐层排查。
