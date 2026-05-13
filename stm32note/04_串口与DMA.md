# 04 串口与 DMA

> 覆盖课程：[19-1]~[19-10] [20]
>
> 目标：掌握串口通信全部场景——发送、三种接收方式、printf 重定向、IDLE+DMA 不定长接收、多串口、编码基础。

---

## 知识地图

| 新名词 | 解释 | 哪里出现 |
|--------|------|---------|
| UART/USART | Universal (Synchronous/)Asynchronous Receiver-Transmitter，通用异步收发器。S 多了同步模式，实际很少用，一般当 UART 用 | [19-1] |
| 波特率（Baud Rate） | 每秒传输的码元数（这里 1 码元 = 1 位），单位 bps。两端波特率必须一致 | [19-1] |
| 起始位/停止位 | 每帧开头一个低电平起始位（1 位），结尾一个高电平停止位（0.5/1/1.5/2 位） | [19-1] |
| 校验位（Parity） | 可选的错误检测位，分奇校验和偶校验。一般不使用（None） | [19-1] |
| TTL 电平 | 串口信号的物理电平标准。STM32 使用 3.3V TTL：高逻辑 = 3.3V，低逻辑 = 0V | [19-1] |
| USB-TTL 模块 | 把 USB 协议转成 TTL 串口的模块（如 CH340G、CP2102）。电脑通过 USB 插上它，串口助手就能和 STM32 通信 | [19-1] |
| DMA | Direct Memory Access，直接存储器访问。硬件控制器自动搬运数据，CPU 不参与 | [19-10] |
| IDLE 中断（空闲中断） | 串口 RX 线上超过 1 个字节时间内没有新数据，硬件自动产生的中断。用来判断「一帧收完了」 | [19-7] |
| Circular 模式 | DMA 的循环模式：搬完最后一笔后自动回到缓冲区开头继续搬，适合持续接收 | [19-10] |
| NDTR | DMA 的 Number of Data Register，记录剩余要搬运的次数。通过 `__HAL_DMA_GET_COUNTER` 读取 | [19-7] |
| 编码 | 用二进制数字表示文字、符号的约定。不同编码下同一个二进制值代表不同字符 | [20] |
| ASCII | American Standard Code for Information Interchange，最常见的字符编码，每个字符占 1 字节 | [20] |

---

## 硬件接线（所有串口实验前提）

```
STM32                         USB-TTL 模块
  PA9  (TX)  ───────────────  RX
  PA10 (RX)  ───────────────  TX
  GND        ───────────────  GND     ← 必须共地！

规则：自己的 TX 接对方的 RX（交叉连接）。
```

> 电平警告：STM32 GPIO 工作在 3.3V。USB-TTL 模块如果有电压选择跳线帽，必须跳到 3.3V。5V 信号直接接 STM32 引脚会造成永久性损坏。

---

## UART HAL 函数族总览

| 函数 | 方向 | 模式 | 阻塞？ | 适用场景 |
|------|------|------|--------|---------|
| `HAL_UART_Transmit(&huart, pData, Size, Timeout)` | 发送 | 阻塞 | 是 | 发少量数据、调试输出 |
| `HAL_UART_Transmit_IT(&huart, pData, Size)` | 发送 | 中断 | 否 | 后台发送，不阻塞 |
| `HAL_UART_Transmit_DMA(&huart, pData, Size)` | 发送 | DMA | 否 | 大批量发送 |
| `HAL_UART_Receive(&huart, pData, Size, Timeout)` | 接收 | 阻塞 | 是 | 简单测试，死等 |
| `HAL_UART_Receive_IT(&huart, pData, Size)` | 接收 | 中断 | 否 | 定长数据接收 |
| `HAL_UART_Receive_DMA(&huart, pData, Size)` | 接收 | DMA | 否 | 大数据接收，常配合 IDLE 中断 |
| `HAL_UART_AbortReceive(&huart)` | — | — | — | 强制中止接收（阻塞） |
| `HAL_UART_AbortReceive_IT(&huart)` | — | — | — | 强制中止接收（非阻塞） |
| `HAL_UART_DMAStop(&huart)` | — | — | — | 停止 DMA 传输 |

---

## [19-1] 发送字节

### UART 帧格式

```
空闲（高电平）
  ↓
起始位  数据位(LSB first)  [校验位]  停止位
 1bit 0  8bit D0-D7        1bit opt  1bit 1
  ↓
空闲（高电平）或下一帧起始位
```

配置参数含义：

| 参数 | 常用值 | 含义 |
|------|--------|------|
| Baud Rate | 115200 | 每秒传 115200 个码元（位） |
| Word Length | 8 Bits | 每个数据帧含 8 位有效数据 |
| Parity | None | 不发送校验位 |
| Stop Bits | 1 | 停止位 1 位 |

### `HAL_UART_Transmit` 详解

```c
HAL_StatusTypeDef HAL_UART_Transmit(UART_HandleTypeDef *huart,
                                     const uint8_t *pData,
                                     uint16_t Size,
                                     uint32_t Timeout);
```

参数：
- `huart`：串口句柄指针。CubeMX 配置 USART1 后生成 `huart1` 全局变量，传 `&huart1`。
- `pData`：指向要发送的数据缓冲区的指针。类型是 `uint8_t*`，如果传字符串需要强制转换：`(uint8_t*)str`。
- `Size`：要发送的字节数。传 1 就是发一个字节。
- `Timeout`：超时时间（毫秒）。每次发送会等 TX 移位寄存器空出来。`100`=等 100ms 没发出去就返回超时错误。`HAL_MAX_DELAY`=不设超时。

返回值：`HAL_OK`（发送成功）、`HAL_TIMEOUT`（超时）、`HAL_BUSY`（外设忙）、`HAL_ERROR`（其他错误）。

函数内部执行过程：用 `for` 循环逐字节写入 TDR（发送数据寄存器），每写一个字节等 TXE（发送寄存器空）标志置位。所以这是个**阻塞函数**：`Size` 越大，卡住时间越长。

### CubeMX 配置

1. Connectivity → USART1 → Mode: Asynchronous
2. 参数保持默认：115200 / 8N1
3. 确认引脚：PA9=TX, PA10=RX

### 代码

```c
uint8_t data = 'A';
HAL_UART_Transmit(&huart1, &data, 1, 100);  // 发 1 字节，超时 100ms
```

### 易错点
- TX 接 RX（交叉），不是同名对接
- 收发两端波特率必须一致

---

## [19-2] 发送字符串与 printf 重定向

### 发送字符串

```c
char *str = "Hello STM32\r\n";
HAL_UART_Transmit(&huart1, (uint8_t *)str, strlen(str), 100);
//                              ^^^^^^^^^^^  ^^^^^^^^^^^
//                              强制转换    需要 <string.h>
```

`strlen()` 返回字符串长度（不含末尾 `\0`）。字符串末尾的 `\0` 不会发送。

### printf 重定向原理

printf 底层调用 `fputc`（或 `__io_putchar`，取决于工具链）输出每个字符。我们要做的就是把这个函数重写为通过串口发出：

```c
// 方式一：fputc（通用性最好）
#include <stdio.h>

int fputc(int ch, FILE *f)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}

// 方式二：__io_putchar（CubeIDE 默认调用这个）
int __io_putchar(int ch)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
```

两种方式功能完全等价，选一个即可。两个都写也不会冲突（链接器只用一个）。

> `HAL_MAX_DELAY` = `0xFFFFFFFF`（约 49.7 天的毫秒数），意思是"不设期限，死等"。在 printf 重定向里用它是因为我们假定串口一定会准备好发送。

### 浮点数打印

printf 默认不支持 `%f`（为了减小代码体积）。需要手动添加链接选项：

CubeIDE → Project → Properties → C/C++ Build → Settings → MCU GCC Linker → Miscellaneous → Other flags → 添加 `-u _printf_float`

> 代价：固件体积增加约 12KB。C8T6 只有 64KB Flash，空间紧张时要掂量。

### 易错点
- `strlen()` 需要 `#include <string.h>`
- `\r\n` 是串口换行的标准写法。只发 `\n` 在某些串口助手里可能只换行不回车
- 打印浮点数之前确认已加 `-u _printf_float`

---

## [19-3] 更换脚组实验

USART1 可以通过重映射（Remap）换到其他引脚。F103 的 USART1 有两组：

| 脚组 | TX | RX | 使能方式 |
|------|----|----|---------|
| 默认 | PA9 | PA10 | 默认 |
| 重映射 | PB6 | PB7 | CubeMX 里 Ctrl+点击 TX/RX 引脚切换 |

在 CubeMX Pinout 视图中，按住 Ctrl 点击 USART1 的 TX 或 RX 脚，会高亮显示可用的替代引脚。点击即可切换。

---

## [19-4] 阻塞接收定长数据

```c
HAL_StatusTypeDef HAL_UART_Receive(UART_HandleTypeDef *huart,
                                    uint8_t *pData,
                                    uint16_t Size,
                                    uint32_t Timeout);
```

参数和 Transmit 类似。区别在于：Receive 是等待接收指定数量的字节，而不是发送。

```c
uint8_t rxBuf[10];
HAL_UART_Receive(&huart1, rxBuf, 10, 1000);  // 等收 10 字节，最多等 1 秒
// 只有收到 10 字节或超时，函数才返回。期间 CPU 卡住。
```

超时时间的计算是逐字节累积的：每收到一个字节重置计时器。所以不是 1000ms 没收到就超时，而是**字节间的间隔**超过 1000ms 才超时。

> 阻塞接收只适合教学验证。实际项目里，CPU 在等数据期间完全不能响应其他事件。

---

## [19-5] 中断接收定长数据

### 原理

`_IT` 版本不阻塞：调用后立即返回，每次收到一个字节硬件自动存到缓冲区。收满 `Size` 个字节后触发 `HAL_UART_RxCpltCallback`。

```c
HAL_StatusTypeDef HAL_UART_Receive_IT(UART_HandleTypeDef *huart,
                                       uint8_t *pData,
                                       uint16_t Size);
```

参数少了 `Timeout`（不需要——非阻塞模式下不需要超时概念）。

### 代码

```c
uint8_t rxBuf[10];

// main() 初始化中启动一次
HAL_UART_Receive_IT(&huart1, rxBuf, 10);

// 回调函数
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        // 此时 rxBuf 中已填满 10 字节，可以处理

        // 关键：重新启动接收。不写这一行 = 只收一次就停
        HAL_UART_Receive_IT(&huart1, rxBuf, 10);
    }
}
```

### 为什么必须重启动？

`HAL_UART_Receive_IT` 内部会设置接收计数器 `RxXferCount = Size`。每收到一个字节，`RxXferCount` 减 1，减到 0 时触发回调，然后**不再接收新数据**。重新调用 `Receive_IT` 才能把计数器重新设为 `Size` 并重新开始。

### 局限性

发送方必须发正好 `Size` 字节，回调才会触发。如果发 5 字节就停了 → 永远凑不够 10 字节 → 回调不触发。

---

## [19-6] 串口控制 LED

利用中断接收单字节命令，根据字符内容控制硬件：

```c
uint8_t  rxCmd;

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
        HAL_UART_Receive_IT(&huart1, &rxCmd, 1);
    }
}
```

单字节命令设计要点：串口助手选「字符发送」模式时，键入 A 回车 = 发 0x41。选「HEX 发送」时，直接发十六进制值。

---

## [19-7] 空闲中断接收不定长数据 ⭐

### 为什么需要这个方式？

中断接收（[19-5]）必须事先知道数据长度。但实际场景中（如 GPS 模块、串口传感器），数据长度是不固定的（可能 5 字节也可能 200 字节）。

IDLE 中断解决了这个问题：硬件自动检测一帧何时结束。

### IDLE 中断的硬件定义

串口接收线（RX）上空闲（持续高电平）超过**一个字节的传输时间**后，硬件自动置位 IDLE 标志（USART_SR 寄存器的 IDLE 位）。这是一个可屏蔽中断源。

```
一帧数据：
[0xAA][0x55][0x01][0x02][0xFF] ── 空闲超过 1 字节时间 →
                                        ↓
                                   IDLE 中断触发
```

一个字节的传输时间 = 10 位（1 起始 + 8 数据 + 1 停止）/ 波特率。
115200 波特下 = 10/115200 ≈ 87μs。即 RX 线空闲超过 87μs，认为一帧结束。

### 为什么要配合 DMA？

串口每收到一个字节，DMA 自动将其搬运到内存缓冲区。CPU 完全不用管理字节级的数据搬运。等 IDLE 中断触发时，整帧数据已经在内存里了。

### DMA 配置参数详解

| 参数 | 设置 | 为什么 |
|------|------|--------|
| Direction | Peripheral to Memory | 数据从串口（外设）流到内存 |
| Mode | Circular | 搬运完最后一字节后自动回到缓冲区开头，覆盖旧数据 |
| Data Width | Byte | 串口数据 1 字节 |
| Memory Increment | Enable | 每搬 1 字节，内存地址 +1 |
| Peripheral Increment | Disable | 外设地址固定（始终读 USART->DR） |

### `__HAL_DMA_GET_COUNTER` 算长度原理

DMA 有一个内部计数器 `NDTR`，初始值为 `Size`（缓冲区总长度）。每搬运一个字节，NDTR 减 1。剩余值 = NDTR。已搬运 = Size - NDTR。

```c
rxLen = sizeof(rxBuf) - __HAL_DMA_GET_COUNTER(huart1.hdmarx);
```

`huart1.hdmarx` 是 `huart1` 内部绑定的接收 DMA 流句柄（CubeMX 自动绑定）。`__HAL_DMA_GET_COUNTER` 宏返回其 NDTR 当前值。

### CubeMX 配置

1. USART1 → DMA Settings → Add → USART1_RX → Mode: Circular
2. NVIC → USART1 global interrupt → Enable
3. DMA 参数确认：Direction = Peripheral to Memory, Data Width = Byte, Memory Increment = Enable

### 完整代码

```c
// ========== main.c ==========

/* USER CODE BEGIN PV */
uint8_t  rxBuf[256];
uint16_t rxLen = 0;
/* USER CODE END PV */

// main() 中：
/* USER CODE BEGIN 2 */
HAL_UART_Receive_DMA(&huart1, rxBuf, sizeof(rxBuf));
__HAL_UART_ENABLE_IT(&huart1, UART_IT_IDLE);  // 使能 IDLE 中断
/* USER CODE END 2 */

// ========== stm32f1xx_it.c ==========

// 文件顶部：
/* USER CODE BEGIN 0 */
extern UART_HandleTypeDef huart1;
extern uint8_t  rxBuf[256];
extern uint16_t rxLen;
/* USER CODE END 0 */

// USART1_IRQHandler 中：
void USART1_IRQHandler(void)
{
    /* USER CODE BEGIN USART1_IRQn 0 */
    if (__HAL_UART_GET_FLAG(&huart1, UART_FLAG_IDLE))
    {
        // 步骤 1：清 IDLE 标志（必须在读 DMA 计数之前）
        __HAL_UART_CLEAR_IDLEFLAG(&huart1);

        // 步骤 2：计算实际接收到的字节数
        rxLen = sizeof(rxBuf) - __HAL_DMA_GET_COUNTER(huart1.hdmarx);

        // 步骤 3：处理数据（示例：原样回发）
        HAL_UART_Transmit(&huart1, rxBuf, rxLen, 100);

        // 步骤 4：停止并重启 DMA（重新初始化 NDTR 计数器）
        HAL_UART_DMAStop(&huart1);
        HAL_UART_Receive_DMA(&huart1, rxBuf, sizeof(rxBuf));
    }
    /* USER CODE END USART1_IRQn 0 */

    HAL_UART_IRQHandler(&huart1);  // 这行不要删！
}
```

### 四个步骤的顺序有严格要求

1. **先清 IDLE 标志**再读 DMA 计数：如果反过来，清理期间可能又有新数据到，计数就变了
2. **先读长度**再重启 DMA：重启 DMA 会重置 NDTR，长度信息就丢了
3. **重新初始化 DMA**：覆盖 Circular 模式下已被使用的全名部分，并放复位计数器

### 易错点
- 忘了 `__HAL_UART_ENABLE_IT(&huart1, UART_IT_IDLE)` → IDLE 永不触发
- IDLE 处理完忘了重启 DMA → 后续数据不收
- 跨文件引用变量（main.c 定义、stm32f1xx_it.c 使用）需要 `extern` 声明
- `HAL_UART_IRQHandler(&huart1)` 不能删 → HAL 库的串口中断框架依赖它

---

## [19-8] 多串口应用

| USART | 总线 | 默认 TX | 默认 RX | 最大波特率 |
|-------|------|---------|---------|----------|
| USART1 | APB2 (72MHz) | PA9 | PA10 | 4.5Mbps |
| USART2 | APB1 (36MHz) | PA2 | PA3 | 2.25Mbps |
| USART3 | APB1 (36MHz) | PB10 | PB11 | 2.25Mbps |

区分不同串口的中断：所有串口共用同一个回调函数 `HAL_UART_RxCpltCallback`，通过 `huart->Instance` 区分：

```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        // 串口 1 的数据
        HAL_UART_Receive_IT(&huart1, rxBuf1, 10);
    }
    else if (huart->Instance == USART2)
    {
        // 串口 2 的数据
        HAL_UART_Receive_IT(&huart2, rxBuf2, 10);
    }
}
```

---

## [19-9] 串口型雷达模块

外部模块通过串口发固定格式的数据帧。通用处理方式：用 IDLE+DMA 接收整帧（[19-7]），然后按帧格式解析：

```
典型帧格式：[帧头 2B][长度 1B][数据 N B][校验 1B][帧尾 1B]

解析步骤：
1. 找帧头 → 2. 读长度 → 3. 取数据 → 4. 验校验 → 5. 按协议翻译
```

不同的模块帧格式不同，具体查模块手册。

---

## [19-10] DMA 详解

### DMA 的工作方式

DMA 控制器是独立于 CPU 的硬件模块。它可以在外设和内存之间搬运数据，全程 CPU 不参与。

```
没有 DMA：
  外设收 1 字节 → 产生中断 → CPU 暂停当前任务
    → 执行 ISR → 读出数据 → 存到内存 → 返回
    → 连续收 256 字节 = CPU 被打断 256 次

有 DMA：
  CPU 初始化 DMA（配置源地址、目标地址、搬运次数）→ 启动
  DMA 自己去外设读数据、写内存，连续搬 256 次
  搬完了 → 产生一次中断通知 CPU
  CPU 只被打断 1 次
```

### DMA 模式对比

| 模式 | 行为 | 适用 |
|------|------|------|
| Normal | 搬完 `Size` 次就停，不再自动重启 | 发一次的场合 |
| Circular | 搬完 `Size` 次后自动回到起始位置继续搬 | 持续接收 |

### 易错点
- 选 Normal 模式做持续接收 → 搬一轮就停，后面的数据丢失
- DMA 通道和外设的绑定关系是硬件固定的（比如 USART1_RX 固定用 DMA1_Channel5），CubeMX 自动匹配

---

## [20] 编码概念

### 为什么需要编码

串口传输的是二进制数据。你在串口助手里看到的 "Hello" 字符串，实际传输的是 `0x48 0x65 0x6C 0x6C 0x6F` 这 5 个字节。字符和字节之间的映射关系，就叫编码。

### 数制

| 数制 | C 语言前缀 | 取值范围 | 位与值的关系 |
|------|----------|--------|------------|
| 二进制 | `0b` | 0~1 | 第 n 位权重 = 2ⁿ |
| 十进制 | 无 | 0~9 | 第 n 位权重 = 10ⁿ |
| 十六进制 | `0x` | 0~F(A=10,B=11...F=15) | 第 n 位权重 = 16ⁿ |

换算示例：
```
0xFF = 15×16¹ + 15×16⁰ = 240 + 15 = 255
0b1010 = 1×2³ + 0×2² + 1×2¹ + 0×2⁰ = 8+0+2+0 = 10
```

### ASCII 核心字符

| 字符 | 十六进制 | 十进制 | 备注 |
|------|---------|--------|------|
| '0' | 0x30 | 48 | 数字 0 的 ASCII 码，不是数值 0 |
| 'A' | 0x41 | 65 | 大写字母从 0x41 开始 |
| 'a' | 0x61 | 97 | 小写字母从 0x61 开始 |
| '\r' | 0x0D | 13 | 回车（Carriage Return） |
| '\n' | 0x0A | 10 | 换行（Line Feed） |
| ' ' (空格) | 0x20 | 32 | |
| 0x00 | 0x00 | 0 | 空字符（C 字符串结束符） |

注意：字符 `'1'` 的 ASCII 码是 `0x31`（十进制 49），不是数值 `1`（0x01）。串口助手里切换「HEX 显示」和「字符显示」就是切换这种映射。

### C 数据类型与字节数

| 类型 | 字节数 | 范围 | 在 DMA 配置中 |
|------|--------|------|-------------|
| `uint8_t` | 1 | 0~255 | Data Width = Byte |
| `uint16_t` | 2 | 0~65535 | Data Width = Half Word |
| `uint32_t` | 4 | 0~4.29×10⁹ | Data Width = Word |
| `int8_t` | 1 | -128~127 | 有符号，串口一般用无符号 |
| `float` | 4 | ±3.4×10³⁸ | 二进制浮点格式（IEEE 754） |

---

*课程：[19-1]~[19-10] [20]*
