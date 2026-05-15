# 04 串口与 DMA

> 覆盖课程：[19-1]~[19-10] [20]
>
> 目标：掌握 USART 发送、阻塞接收、中断接收、`printf` 重定向、IDLE + DMA 不定长接收、多串口和编码基础。

---

## 概念

### 知识地图

| 名词 | 工程含义 | 常见位置 |
|------|----------|----------|
| UART / USART | 异步/同步串口外设，日常学习中 USART 常按 UART 使用 | [19-1] |
| 波特率 | 每秒传输的码元数，常见串口中 1 码元约等于 1 bit | [19-1] |
| 8N1 | 8 数据位、无校验、1 停止位 | [19-1] |
| TTL 电平 | STM32 常用 3.3 V TTL，高电平约 3.3 V，低电平 0 V | [19-1] |
| USB-TTL | 把电脑 USB 转成 TTL 串口的模块，如 CH340、CP2102 | [19-1] |
| DMA | Direct Memory Access，硬件自动搬运外设和内存之间的数据 | [19-10] |
| IDLE 中断 | RX 线空闲超过 1 个字节时间后产生的中断 | [19-7] |
| Circular | DMA 循环模式，搬到缓冲区末尾后从头继续搬 | [19-10] |
| NDTR | DMA 剩余传输计数器，用于计算已接收长度 | [19-7] |
| 编码 | 字节和字符之间的映射规则，如 ASCII | [20] |

### 串口 HAL API

| 函数 | 方向 | 模式 | 是否阻塞 | 常见用途 |
|------|------|------|----------|----------|
| `HAL_UART_Transmit()` | 发送 | 轮询 | 是 | 少量调试输出 |
| `HAL_UART_Transmit_IT()` | 发送 | 中断 | 否 | 后台发送 |
| `HAL_UART_Transmit_DMA()` | 发送 | DMA | 否 | 大量数据发送 |
| `HAL_UART_Receive()` | 接收 | 轮询 | 是 | 简单验证 |
| `HAL_UART_Receive_IT()` | 接收 | 中断 | 否 | 定长接收 |
| `HAL_UART_Receive_DMA()` | 接收 | DMA | 否 | 大量或连续接收 |
| `HAL_UART_DMAStop()` | DMA | - | - | 停止串口 DMA |
| `HAL_UART_RxCpltCallback()` | 接收 | 中断/DMA | - | 接收完成回调 |

---

## 本质理解

串口解决的是两个设备之间按位传输字节的问题。

```text
发送端 TX
-> 起始位
-> 数据位（低位先发）
-> 可选校验位
-> 停止位
-> 接收端 RX
```

DMA 解决的是“CPU 不想一个字节一个字节搬数据”的问题。

```text
没有 DMA：
每收到 1 字节 -> 中断一次 -> CPU 读 DR -> CPU 存数组

有 DMA：
每收到 1 字节 -> DMA 自动从 USART_DR 搬到数组
一帧结束或搬满后 -> CPU 再处理整段数据
```

IDLE 中断解决的是“不知道一帧到底多长”的问题。

```text
收到一串字节
-> RX 线空闲超过 1 个字节时间
-> USART 置位 IDLE 标志
-> 认为一帧结束
```

---

## 工作流程 / 原理

### 硬件接线

```text
STM32                         USB-TTL
PA9  USART1_TX  ------------  RX
PA10 USART1_RX  ------------  TX
GND             ------------  GND
```

规则：

- 自己的 `TX` 接对方的 `RX`。
- 自己的 `RX` 接对方的 `TX`。
- **必须共地**。
- USB-TTL 模块要选 3.3 V 逻辑电平，5 V 串口信号可能损坏 STM32。

### UART 帧格式

```text
空闲高电平
-> 起始位 0
-> 数据位 D0~D7（低位先发）
-> 可选校验位
-> 停止位 1
-> 回到空闲高电平
```

常用配置：

| 参数 | 常用值 |
|------|--------|
| Baud Rate | `115200` |
| Word Length | `8 Bits` |
| Parity | `None` |
| Stop Bits | `1` |

### 阻塞、IT、DMA 的选择

| 方式 | 优点 | 缺点 | 适合 |
|------|------|------|------|
| 阻塞 | 最简单 | CPU 等待，不能做别的事 | 入门验证、少量打印 |
| 中断 | 非阻塞 | 高频字节会频繁进中断 | 定长短帧、命令字节 |
| DMA | CPU 负担小 | 配置和缓冲区管理更复杂 | 大量数据、不定长数据 |

### `printf` 重定向

`printf` 最终会调用底层字符输出函数。把这个函数改写成串口发送，`printf` 就能输出到串口助手。

CubeIDE 常用 `__io_putchar()`：

```c
int __io_putchar(int ch)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
```

浮点数打印 `%f` 默认可能被裁剪，需要加链接选项：

```text
Project -> Properties
-> C/C++ Build -> Settings
-> MCU GCC Linker -> Miscellaneous
-> Other flags 添加 -u _printf_float
```

代价：固件体积会增加，STM32F103C8T6 的 64 KB Flash 要注意空间。

### IDLE + DMA 不定长接收

推荐理解流程：

```text
初始化：
1. 启动 UART RX DMA
2. 使能 UART IDLE 中断

接收中：
1. DMA 自动把每个字节写入 rxBuf
2. RX 空闲超过 1 个字节时间
3. IDLE 中断触发
4. 清 IDLE 标志
5. 停止 DMA，冻结 NDTR
6. 用 Size - NDTR 算长度
7. 设置一帧就绪标志
8. 重启 DMA 接收下一帧
9. 主循环处理这一帧
```

为什么不建议在中断里直接 `printf` 或长时间 `HAL_UART_Transmit()`：串口发送是慢速外设，115200 波特下 1 个字符约 87 us，一行字符串可能耗时毫秒级。中断里做这个会拖慢系统响应。

### 编码基础

串口传输的是字节。你在串口助手里看到的字符，是软件按编码显示出来的。

```text
"Hello" 实际字节：
0x48 0x65 0x6C 0x6C 0x6F
```

常见 ASCII：

| 字符 | 十六进制 | 十进制 |
|------|----------|--------|
| `'0'` | `0x30` | 48 |
| `'1'` | `0x31` | 49 |
| `'A'` | `0x41` | 65 |
| `'a'` | `0x61` | 97 |
| `'\r'` | `0x0D` | 13 |
| `'\n'` | `0x0A` | 10 |
| `'\0'` | `0x00` | 0 |

注意：字符 `'1'` 不是数值 `1`，而是字节 `0x31`。

---

## CubeMX 配置

### USART1 基础配置

1. `Connectivity -> USART1`。
2. `Mode` 选择 `Asynchronous`。
3. 参数：
   - `Baud Rate`：`115200`
   - `Word Length`：`8 Bits`
   - `Parity`：`None`
   - `Stop Bits`：`1`
4. 默认引脚：
   - `PA9` = USART1_TX
   - `PA10` = USART1_RX

### USART1 重映射

STM32F103 的 USART1 可重映射：

| 脚组 | TX | RX |
|------|----|----|
| 默认 | PA9 | PA10 |
| 重映射 | PB6 | PB7 |

在 CubeMX Pinout 视图中，可按住 `Ctrl` 点击外设引脚查看可替代引脚。实际是否可用仍要查数据手册的 alternate function 表。

### 中断接收配置

1. USART1 配为 `Asynchronous`。
2. `NVIC Settings` 勾选 `USART1 global interrupt`。
3. 代码中调用：

```c
HAL_UART_Receive_IT(&huart1, rxBuf, 10);
```

### IDLE + DMA 配置

1. USART1 配为 `Asynchronous`。
2. `DMA Settings` 添加 `USART1_RX`。
3. DMA 参数：
   - `Direction`：`Peripheral to Memory`
   - `Mode`：`Normal` 或 `Circular`
   - `Peripheral Increment`：`Disable`
   - `Memory Increment`：`Enable`
   - `Data Width`：`Byte`
4. `NVIC Settings` 勾选 `USART1 global interrupt`。
5. 代码中手动使能 IDLE：

```c
__HAL_UART_ENABLE_IT(&huart1, UART_IT_IDLE);
```

说明：不定长帧入门实现建议先用 `Normal` 模式，收到一帧后停止并重启 DMA，逻辑最清晰。持续环形接收可用 `Circular`，但要额外处理写指针和缓冲区覆盖问题。

---

## HAL 函数 / API

### `HAL_UART_Transmit`

```c
HAL_StatusTypeDef HAL_UART_Transmit(UART_HandleTypeDef *huart,
                                    const uint8_t *pData,
                                    uint16_t Size,
                                    uint32_t Timeout);
```

参数：

- `huart`：串口句柄，如 `&huart1`。
- `pData`：待发送数据首地址。
- `Size`：发送字节数。
- `Timeout`：超时时间，单位 ms。

F1 的 USART 数据寄存器叫 `DR`，不是新系列里常见的 `TDR/RDR` 命名。HAL 会屏蔽这类寄存器差异。

### `HAL_UART_Receive`

```c
HAL_StatusTypeDef HAL_UART_Receive(UART_HandleTypeDef *huart,
                                   uint8_t *pData,
                                   uint16_t Size,
                                   uint32_t Timeout);
```

阻塞等待收到指定字节数或超时，教学验证可以用，工程里避免长时间死等。

### `HAL_UART_Receive_IT`

```c
HAL_StatusTypeDef HAL_UART_Receive_IT(UART_HandleTypeDef *huart,
                                      uint8_t *pData,
                                      uint16_t Size);
```

调用后立即返回。收满 `Size` 字节后进入 `HAL_UART_RxCpltCallback()`。回调里必须重新启动下一次接收。

### `HAL_UART_Receive_DMA`

```c
HAL_StatusTypeDef HAL_UART_Receive_DMA(UART_HandleTypeDef *huart,
                                       uint8_t *pData,
                                       uint16_t Size);
```

启动 DMA 接收。DMA 会自动把串口收到的数据搬到 `pData` 指向的缓冲区。

### `__HAL_DMA_GET_COUNTER`

```c
uint16_t remain = __HAL_DMA_GET_COUNTER(huart1.hdmarx);
```

读取 DMA 剩余搬运次数。已接收长度：

```c
rxLen = RX_BUF_SIZE - __HAL_DMA_GET_COUNTER(huart1.hdmarx);
```

---

## 示例代码

### 发送 1 个字节

```c
uint8_t data = 'A';
HAL_UART_Transmit(&huart1, &data, 1, 100);
```

### 发送字符串

```c
#include <string.h>

char *str = "Hello STM32\r\n";
HAL_UART_Transmit(&huart1, (uint8_t *)str, strlen(str), 100);
```

`strlen()` 不包含末尾 `'\0'`，所以不会把字符串结束符发出去。

### `printf` 重定向

```c
#include <stdio.h>

int __io_putchar(int ch)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
```

也可以用 `fputc()`：

```c
#include <stdio.h>

int fputc(int ch, FILE *f)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
```

二者选一种即可。

### 阻塞接收 10 字节

```c
uint8_t rxBuf[10];

if (HAL_UART_Receive(&huart1, rxBuf, 10, 1000) == HAL_OK)
{
    HAL_UART_Transmit(&huart1, rxBuf, 10, 100);
}
```

只有收到 10 字节或超时，函数才返回。等待期间 CPU 被阻塞。

### 中断接收定长数据

```c
uint8_t rxBuf[10];

/* USER CODE BEGIN 2 */
HAL_UART_Receive_IT(&huart1, rxBuf, sizeof(rxBuf));
/* USER CODE END 2 */

/* USER CODE BEGIN 4 */
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        HAL_UART_Transmit(&huart1, rxBuf, sizeof(rxBuf), 100);

        // 必须重新启动，否则只接收一次
        HAL_UART_Receive_IT(&huart1, rxBuf, sizeof(rxBuf));
    }
}
/* USER CODE END 4 */
```

局限：发送方必须刚好发满 `Size` 字节，回调才会触发。

### 串口控制 LED

```c
uint8_t rxCmd;

/* USER CODE BEGIN 2 */
HAL_UART_Receive_IT(&huart1, &rxCmd, 1);
/* USER CODE END 2 */

/* USER CODE BEGIN 4 */
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        switch (rxCmd)
        {
            case 'A':
                HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);
                break;

            case 'B':
                HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET);
                break;

            case 'C':
                HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_1);
                break;

            default:
                break;
        }

        HAL_UART_Receive_IT(&huart1, &rxCmd, 1);
    }
}
/* USER CODE END 4 */
```

### IDLE + DMA 不定长接收

#### `main.c`

```c
#define RX_BUF_SIZE 256

/* USER CODE BEGIN PV */
uint8_t rxBuf[RX_BUF_SIZE];
uint8_t rxFrame[RX_BUF_SIZE];
volatile uint16_t rxLen = 0;
volatile uint8_t rxFrameReady = 0;
/* USER CODE END PV */

/* USER CODE BEGIN 2 */
HAL_UART_Receive_DMA(&huart1, rxBuf, RX_BUF_SIZE);
__HAL_UART_ENABLE_IT(&huart1, UART_IT_IDLE);
/* USER CODE END 2 */

while (1)
{
    if (rxFrameReady)
    {
        rxFrameReady = 0;

        // 示例：原样回发刚收到的一帧
        HAL_UART_Transmit(&huart1, rxFrame, rxLen, 100);
    }
}
```

#### `stm32f1xx_it.c`

```c
/* USER CODE BEGIN 0 */
#include <string.h>

extern UART_HandleTypeDef huart1;
extern uint8_t rxBuf[256];
extern uint8_t rxFrame[256];
extern volatile uint16_t rxLen;
extern volatile uint8_t rxFrameReady;
/* USER CODE END 0 */

void USART1_IRQHandler(void)
{
    /* USER CODE BEGIN USART1_IRQn 0 */
    if (__HAL_UART_GET_FLAG(&huart1, UART_FLAG_IDLE) != RESET)
    {
        __HAL_UART_CLEAR_IDLEFLAG(&huart1);

        HAL_UART_DMAStop(&huart1);

        rxLen = 256 - __HAL_DMA_GET_COUNTER(huart1.hdmarx);
        if (rxLen > 256)
        {
            rxLen = 256;
        }

        memcpy(rxFrame, rxBuf, rxLen);
        rxFrameReady = 1;

        HAL_UART_Receive_DMA(&huart1, rxBuf, 256);
    }
    /* USER CODE END USART1_IRQn 0 */

    HAL_UART_IRQHandler(&huart1);
}
```

注意：

- `HAL_UART_IRQHandler(&huart1)` 不能删除，HAL 的串口中断框架依赖它。
- `rxFrameReady` 和 `rxLen` 被中断和主循环同时访问，应使用 `volatile`。
- 示例为了教学清晰使用固定长度 `256`，实际工程可把缓冲区大小放到公共头文件里，避免跨文件魔法数字。

### 多串口回调区分

```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        HAL_UART_Receive_IT(&huart1, rxBuf1, sizeof(rxBuf1));
    }
    else if (huart->Instance == USART2)
    {
        HAL_UART_Receive_IT(&huart2, rxBuf2, sizeof(rxBuf2));
    }
}
```

STM32F103C8T6 常见串口：

| USART | 总线 | 默认 TX | 默认 RX | 备注 |
|-------|------|---------|---------|------|
| USART1 | APB2 | PA9 | PA10 | 常用调试串口 |
| USART2 | APB1 | PA2 | PA3 | 常用扩展串口 |
| USART3 | APB1 | PB10 | PB11 | 具体封装需查手册 |

### 串口型模块帧解析

```text
典型帧格式：
[帧头 2B][长度 1B][数据 N B][校验 1B][帧尾 1B]

解析流程：
1. 找帧头
2. 读长度
3. 检查长度是否合理
4. 取数据
5. 校验
6. 按协议翻译
```

GPS、雷达、蓝牙、传感器模块都可能采用类似“帧头 + 长度 + 数据 + 校验”的协议，但具体格式必须查模块手册。

---

## 常见错误

| 问题 | 常见原因 | 解决方法 |
|------|----------|----------|
| 串口不通 | TX 接 TX、RX 接 RX | 交叉连接：TX -> RX，RX -> TX |
| 收到乱码 | 波特率不一致、未共地、5 V 电平 | 核对 115200/8N1、共地、3.3 V |
| `printf` 没输出 | 没实现 `__io_putchar` / `fputc` | 添加重定向函数 |
| `%f` 打不出来 | 未链接浮点打印支持 | 添加 `-u _printf_float` |
| 中断接收只触发一次 | 回调里没重启接收 | 回调末尾重新 `Receive_IT` |
| 定长接收一直不回调 | 没收到足够字节 | 改成单字节命令或 IDLE + DMA |
| IDLE 不触发 | 忘了使能 `UART_IT_IDLE` | 调用 `__HAL_UART_ENABLE_IT` |
| DMA 后续不接收 | 处理完没有重启 DMA | `HAL_UART_DMAStop` 后重新 `Receive_DMA` |
| 跨文件变量未定义 | `main.c` 定义，`it.c` 使用 | 在 `it.c` 里 `extern` 声明 |
| 中断里发送太慢 | 在 IRQ 中大量 `printf` | 中断只置标志，主循环处理 |

---

## 调试方法

1. 用串口助手确认端口号、波特率、8N1。
2. 用万用表确认 USB-TTL 和 STM32 共地。
3. 先做最小发送：上电打印 `"hello\r\n"`。
4. 再做单字节接收：收到 `'A'` 翻转 LED。
5. 再做定长接收。
6. 最后做 IDLE + DMA。

调试优先级：

```text
硬件接线
-> 波特率和电平
-> CubeMX 引脚和 NVIC/DMA
-> API 是否启动
-> 回调是否进入
-> 缓冲区长度和解析逻辑
```

---

## 工程意义

串口是嵌入式工程最常用的调试口和模块接口。

它的价值不只是通信，还包括：

- `printf` 调试。
- 外部传感器和模块接入。
- 参数配置。
- 日志输出。
- Bootloader 或命令行交互。

工程实践中通常按复杂度逐步升级：

```text
阻塞发送
-> printf 重定向
-> 单字节中断命令
-> 定长中断接收
-> IDLE + DMA 不定长接收
-> 协议解析和环形缓冲区
```

---

## 易混淆点

| 容易混淆 | 正确理解 |
|----------|----------|
| UART 和 USART | USART 多同步能力，常规异步串口可当 UART 用 |
| TX/RX 接法 | 自己 TX 接对方 RX |
| 字符 `'1'` 和数值 `1` | `'1'` 的 ASCII 是 `0x31` |
| HEX 显示和字符显示 | 只是串口助手显示方式不同，底层都是字节 |
| 阻塞接收和中断接收 | 阻塞会卡住 CPU，中断立即返回 |
| IDLE 中断和接收完成中断 | IDLE 判断一帧间隔，完成中断判断固定长度 |
| DMA Normal 和 Circular | Normal 一轮就停，Circular 会循环覆盖 |
| 中断里处理数据和主循环处理数据 | 中断里应尽快退出，复杂解析放主循环 |

---

## 我的理解

串口这章的核心不是背 API，而是搞清楚“数据什么时候来、来了放哪里、什么时候算一帧结束”。

我现在的理解是：

- 少量调试输出用阻塞发送就够。
- 命令控制用单字节中断很舒服。
- 长度固定的数据可以用定长中断接收。
- 长度不固定的数据，要靠 IDLE 判断一帧结束。
- 数据量大或连续接收，就要让 DMA 帮 CPU 搬数据。
- 串口问题先查线和波特率，再查代码。

后面做任何串口模块，我会先按这个顺序设计：

```text
确认电平和接线
-> 确认波特率和帧格式
-> 选接收方式
-> 设计缓冲区
-> 判断一帧结束条件
-> 解析协议
-> 做错误处理和超时
```

---

*课程：[19-1]~[19-10] [20]*
