# 四、串口与 DMA 篇

> 覆盖课程：[19-1]~[19-10] [20]
>
> 目标：掌握串口通信的所有常用场景——发送、阻塞接收、中断接收、IDLE+DMA 不定长接收、多串口、printf 重定向。

---

## 硬件接线（所有串口实验的前置条件）⭐

```
STM32                  USB-TTL 模块（CH340 / CP2102）
  TX (PA9)  ─────────→  RX
  RX (PA10) ─────────→  TX
  GND       ─────────→  GND       ← 必须共地！

铁律：自己的 TX 接对方的 RX（交叉连接）
```

> 🔴 电平警告：STM32 GPIO 是 3.3V。BTM 模块必须选 3.3V 档（有跳线帽的跳到 3.3V）。5V 直接连 STM32 引脚可能永久损坏芯片。

---

## [19-1] 串口发送之发送字节

### 知识点
- UART 异步串行通信的基本参数
- `HAL_UART_Transmit()` 发送单字节

### CubeMX 配置步骤

1. Connectivity → USART1 → Mode: Asynchronous
2. 参数保持默认：115200 / 8N1
3. 确认 PA9=TX, PA10=RX

### HAL 库代码模板

```c
uint8_t data = 'A';
HAL_UART_Transmit(&huart1, &data, 1, 100);  // 发送 1 字节，超时 100ms
```

### 易错点
- TX/RX 引脚接反
- 波特率两端不一致

---

## [19-2] 串口发送之发送字符串与 printf 重定向

### 知识点
- `HAL_UART_Transmit()` 发送多字节
- `fputc` 重定向实现 printf

### HAL 库代码模板

```c
// 发送字符串
char *str = "Hello STM32\r\n";
HAL_UART_Transmit(&huart1, (uint8_t *)str, strlen(str), 100);

// printf 重定向（加在 main.c 的 /* USER CODE BEGIN 0 */）
#include <stdio.h>

int fputc(int ch, FILE *f)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, 100);
    return ch;
}

// 之后直接用：
printf("ADC 值: %d\r\n", adcValue);
```

### 打印浮点数

如果 `printf("%f")` 不显示，需加链接选项：

> CubeIDE → Project → Properties → C/C++ Build → Settings → MCU GCC Linker → Miscellaneous → Linker flags → 添加 `-u _printf_float`

> ⚠️ 会增大固件约 12KB，Flash 紧张时避免打印浮点数。

### 易错点
- `strlen()` 需要 `#include <string.h>`
- 浮点数不显示 → 没加 `-u _printf_float`
- 注意换行符是 `\r\n`（串口助手里只发 `\n` 可能不换行）

---

## [19-3] 串口发送之更换脚组实验

### 知识点
- USART1 可以重映射到不同引脚组
- 引脚重映射的概念

### 关键原理

一个外设可能对应多组引脚，通过 GPIO 重映射（Remap）或直接选其他脚组来切换：

| USART1 脚组 | TX | RX |
|------------|-----|-----|
| 默认 | PA9 | PA10 |
| 重映射 | PB6 | PB7 |

在 CubeMX 里点 USART1 的 TX/RX 引脚，可以看到可选的其他脚组。CTRL+点击即可切换。

### 易错点
- 重映射后要确认新引脚没有被其他外设占用
- 某些重映射组合可能需要手动开启 AFIO 时钟（F103 系列）

---

## [19-4] 串口接收之阻塞接收定长数据

### 知识点
- `HAL_UART_Receive()` 阻塞接收
- 阻塞方式的优缺点

### HAL 库代码模板

```c
uint8_t rxBuf[10];
HAL_UART_Receive(&huart1, rxBuf, 10, 1000);  // 等收 10 字节，超时 1000ms
// 函数返回前，CPU 卡住不能做任何事
```

### 易错点
- 阻塞接收期间 CPU 完全不能响应其他事件——只适合简单测试
- 一旦对方少发了数据，会一直等到超时

---

## [19-5] 串口接收之中断接收定长数据

### 知识点
- `HAL_UART_Receive_IT()` 非阻塞接收
- 接收完成回调 `HAL_UART_RxCpltCallback`

### 关键原理

中断接收不阻塞 CPU，收满指定字节后硬件自动触发回调。

### HAL 库代码模板

```c
uint8_t rxBuf[10];

// main() 中启动一次
HAL_UART_Receive_IT(&huart1, rxBuf, 10);

// 回调（/* USER CODE BEGIN 4 */）
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        // 处理 rxBuf 数据...

        // 🔴 必须重新启动接收，否则只收一次
        HAL_UART_Receive_IT(&huart1, rxBuf, 10);
    }
}
```

### 易错点
- 回调里忘了重新调 `Receive_IT` → 只收一次就停了
- 数据和长度必须匹配：收了 10 字节才触发回调，如果对方只发 5 字节就永远不触发

---

## [19-6] 串口接收之控制 LED 灯

### 知识点
- 串口指令协议设计（单字节命令）
- 用串口助手发送字符控制硬件

### HAL 库代码模板

```c
uint8_t rxCmd;  // 单字节命令

// main() 中启动
HAL_UART_Receive_IT(&huart1, &rxCmd, 1);

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        switch (rxCmd)
        {
            case 'A': HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);   break;
            case 'B': HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET); break;
            case 'C': HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_1);                break;
        }
        HAL_UART_Receive_IT(&huart1, &rxCmd, 1);  // 继续收下一条
    }
}
```

### 易错点
- 串口助手里要选「HEX 发送」还是「字符发送」——这和你的协议有关
- 字符命令区分大小写（'A' ≠ 'a'）

---

## [19-7] 串口接收之空闲中断接收不定长数据 ⭐⭐⭐

### 知识点
- UART IDLE 空闲中断原理
- DMA 循环模式搬运
- 这是实际项目最常用的接收方式

### 关键原理

```
串口收到一帧数据后，总线空闲超过 1 字节时间
  → 硬件自动产生 IDLE 中断
  → 我们在 IDLE 中断里把 DMA 搬好的数据一次性取出

不需要知道数据是 5 字节还是 200 字节，硬件自动识别帧尾。
```

#### 为什么用 `__HAL_DMA_GET_COUNTER` 算长度？

```
DMA Circular 模式：每搬运 1 字节，NDTR 计数器减 1
已接收字节数 = 缓冲区总大小 - NDTR 剩余计数
```

### CubeMX 配置步骤

1. USART1 → DMA Settings → Add → USART1_RX → Mode: Circular
2. NVIC Settings → USART1 global interrupt → Enable
3. DMA 参数：Direction: Peripheral to Memory, Data Width: Byte, Memory Increment: Enable

### HAL 库代码模板

```c
// 全局变量（/* USER CODE BEGIN PV */）
uint8_t  rxBuf[256];
uint16_t rxLen = 0;

// main() 初始化（/* USER CODE BEGIN 2 */）
HAL_UART_Receive_DMA(&huart1, rxBuf, sizeof(rxBuf));
__HAL_UART_ENABLE_IT(&huart1, UART_IT_IDLE);  // 使能空闲中断

// stm32f1xx_it.c → USART1_IRQHandler 中添加（/* USER CODE BEGIN USART1_IRQn 0 */）
if (__HAL_UART_GET_FLAG(&huart1, UART_FLAG_IDLE))
{
    __HAL_UART_CLEAR_IDLEFLAG(&huart1);  // 先清标志

    rxLen = sizeof(rxBuf) - __HAL_DMA_GET_COUNTER(huart1.hdmarx);  // 算长度

    // 处理数据（示例：回显）
    HAL_UART_Transmit(&huart1, rxBuf, rxLen, 100);

    // 重启 DMA
    HAL_UART_DMAStop(&huart1);
    HAL_UART_Receive_DMA(&huart1, rxBuf, sizeof(rxBuf));
}
// HAL_UART_IRQHandler(&huart1) 不要删，放在后面

// stm32f1xx_it.c 需要 extern 外部变量
extern UART_HandleTypeDef huart1;
extern uint8_t  rxBuf[256];
extern uint16_t rxLen;
```

### 易错点
- 忘了 `__HAL_UART_ENABLE_IT(&huart1, UART_IT_IDLE)` → IDLE 中断不触发
- IDLE 中断处理完忘了重启 DMA → 后续数据不收
- `__HAL_UART_CLEAR_IDLEFLAG` 必须在读 DMA 计数器之前调用
- 跨文件引用变量需要 `extern` 声明

---

## [19-8] 串口之多串口应用

### 知识点
- 同时使用多个 USART
- 各 USART 的引脚映射

### 关键原理

| USART | 所在总线 | 默认 TX | 默认 RX |
|-------|---------|---------|---------|
| USART1 | APB2 | PA9 | PA10 |
| USART2 | APB1 | PA2 | PA3 |
| USART3 | APB1 | PB10 | PB11 |

每个 USART 独立配置，使用方法完全一样。在回调里通过 `huart->Instance` 区分是哪个串口触发的中断。

### HAL 库代码模板

```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        // 处理串口 1 的数据
        HAL_UART_Receive_IT(&huart1, rxBuf1, 10);
    }
    else if (huart->Instance == USART2)
    {
        // 处理串口 2 的数据
        HAL_UART_Receive_IT(&huart2, rxBuf2, 10);
    }
}
```

### 易错点
- USART2/3 在 APB1 上，时钟使能要注意（CubeMX 自动处理，但手动建工程时容易漏）

---

## [19-9] 串口型雷达模块基础应用

### 知识点
- 外部模块通过串口发送数据帧
- 根据协议解析数据帧

### 关键原理

雷达模块（或其他串口传感器）一般按固定帧格式持续发送数据：

```
帧头（如 0x55 0xAA）+ 数据 + 校验 + 帧尾
```

用 [19-7] 的 IDLE+DMA 方式接收整帧，然后按协议解析。

### 易错点
- 不同模块的波特率可能不同，确认后再配
- 模块上电后有初始化时间，上电立即读可能收到乱码

---

## [19-10] DMA 与串口应用

### 知识点
- DMA 的作用：解放 CPU
- DMA 配置要点（Circular 模式、数据宽度、地址自增）

### 关键原理

```
没有 DMA：
  每来 1 字节 → CPU 被中断 → 手动搬到内存 → 1 字节处理完 → 又来...

有了 DMA：
  DMA 控制器自动把串口数据搬到内存
  CPU 只在整帧收完后才介入处理
```

| DMA 参数 | 设置 | 原因 |
|----------|------|------|
| Direction | Peripheral to Memory | 串口→内存（接收） |
| Mode | Circular | 循环搬运，不间断 |
| Data Width | Byte | 串口数据是字节 |
| Memory Increment | Enable | 内存地址递增 |

### 易错点
- 如果选 Normal 模式而非 Circular → 搬完一轮就停了
- DMA 通道和 USART 的对应关系是硬件固定的，CubeMX 会自动匹配

---

## [20] 编码的概念与应用

### 知识点
- 二进制、十进制、十六进制的关系和转换
- ASCII 编码表
- 数据类型对应的字节数

### 关键原理

#### 为什么要学编码

串口通信传的都是二进制数据。你在串口助手里看到的字符，本质上是按照 ASCII 编码表翻译后的结果。理解编码才能理解「串口收到的是什么」。

#### 数制速查

| 数制 | 前缀 | 例 | 十进制 |
|------|------|-----|--------|
| 二进制 | `0b` | `0b1010` | 10 |
| 十进制 | 无 | `10` | 10 |
| 十六进制 | `0x` | `0x0A` | 10 |

#### ASCII 核心

| 字符 | ASCII (HEX) | 十进制 |
|------|-----------|--------|
| '0' | 0x30 | 48 |
| 'A' | 0x41 | 65 |
| 'a' | 0x61 | 97 |
| '\r' (回车) | 0x0D | 13 |
| '\n' (换行) | 0x0A | 10 |
| 空格 | 0x20 | 32 |

#### C 语言数据类型

| 类型 | 字节数 | 范围 |
|------|--------|------|
| uint8_t | 1 | 0 ~ 255 |
| uint16_t | 2 | 0 ~ 65535 |
| uint32_t | 4 | 0 ~ 4,294,967,295 |
| int8_t | 1 | -128 ~ 127 |
| float | 4 | ±3.4×10³⁸ |

### 易错点
- 字符 '1'（ASCII 0x31）和数字 1（0x01）不是一回事
- 串口调试助手里「HEX 显示」和「字符显示」看到的内容不同
- `sizeof(uint8_t)` = 1, `sizeof(uint16_t)` = 2, `sizeof(uint32_t)` = 4 —— 这个在 DMA 配数据宽度时很重要

---

*课程：[19-1]~[19-10] [20]*
