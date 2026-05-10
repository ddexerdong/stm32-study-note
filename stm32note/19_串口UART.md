# 19 串口 UART（核心章节）

> 串口是 STM32 和电脑/其他设备通信最常用的方式。调试打印、数据上报、指令控制都靠它。
> 本章是**最重要的通信章节**，建议边看边动手验证。

---

## 硬件怎么接线 ⭐

这是新手接线出错最多的地方，先搞清楚：

```
STM32         USB-TTL 模块（CH340/CP2102）
  TX (PA9)  ────→  RX
  RX (PA10) ────→  TX
  GND       ────→  GND     ← 必须共地！

规则：自己的 TX 接对方的 RX，自己的 RX 接对方的 TX（交叉连接）
```

> 🔴 **电平警告**：STM32 的 GPIO 是 **3.3V 电平**。买 USB-TTL 模块必须确认是 3.3V 版本。5V 电平直接接到 STM32 引脚**可能永久损坏**芯片。有些模块有 3.3V/5V 跳线帽，记得跳到 3.3V。

---

## UART 基础配置（CubeMX）

| 参数 | 常用值 | 说明 |
|------|--------|------|
| Baud Rate | 115200 | 每秒传输 115200 位 |
| Word Length | 8 Bits | 一个数据包 8 位 |
| Parity | None | 不校验（省资源） |
| Stop Bits | 1 | 1 位停止位 |
| Mode | Asynchronous | 异步通信 |

---

## 三种接收方式对比 ⭐

| 方式 | 函数 | CPU 占用 | 适用 |
|------|------|---------|------|
| 阻塞接收 | `HAL_UART_Receive` | CPU 死等 | 简单测试，不要用于正式项目 |
| 中断接收 | `HAL_UART_Receive_IT` | 低 | 已知长度（定长）的数据包 |
| IDLE+DMA | `IDLE 中断 + DMA` | 几乎为零 | ⭐ 实际项目首选，不定长数据 |

---

## 19-1 发送字节

```c
uint8_t data = 'A';
HAL_UART_Transmit(&huart1, &data, 1, 100);  // 第 4 个参数是超时(ms)
```

---

## 19-2 发送字符串 + printf 重定向

```c
// 方法1：发送字符串
char *str = "Hello STM32\r\n";
HAL_UART_Transmit(&huart1, (uint8_t *)str, strlen(str), 100);

// 方法2：printf 重定向（推荐） ⭐
// 加在 main.c 的 /* USER CODE BEGIN 0 */ 区域
#include <stdio.h>

int fputc(int ch, FILE *f)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, 100);
    return ch;
}

// 之后就能直接用：
printf("ADC值: %d\r\n", adcValue);
printf("电压: %.2fV\r\n", voltage);
```

### printf 打印浮点数

如果 `printf("%f")` 打印不出来（显示空白或乱码），需要加链接选项：

> CubeIDE → Project → Properties → C/C++ Build → Settings → MCU GCC Linker → Miscellaneous → Linker flags → 添加 `-u _printf_float`

> ⚠️ 这个选项会让固件体积增加约 12KB。Flash 空间紧张时可以避免打印浮点数（先转成整数再打印）。

---

## 19-4 阻塞接收定长数据

```c
uint8_t rxBuf[10];
HAL_UART_Receive(&huart1, rxBuf, 10, 1000);  // 等接收 10 字节，超时 1000ms
// 函数返回前，程序卡在这里什么都不能干
```

---

## 19-5 中断接收定长数据

```c
uint8_t rxBuf[10];

// main() 里启动一次中断接收
HAL_UART_Receive_IT(&huart1, rxBuf, 10);

// 接收完 10 字节后自动调用此回调（放在 /* USER CODE BEGIN 4 */ 区域）
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        // 处理 rxBuf 中的数据...

        // 🔴 必须重新启动接收，否则只收一次就不再收了
        HAL_UART_Receive_IT(&huart1, rxBuf, 10);
    }
}
```

> 💡 回调里忘了重新调用 `Receive_IT` = 只能收一次，这是新手踩得最多的串口坑。

---

## 19-6 串口控制 LED ⭐

### 协议设计思路

```
命令格式：单字节命令
  'A' → LED1 亮
  'B' → LED1 灭
  'C' → LED2 翻转
  ...
```

```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        switch (rxBuf[0])
        {
            case 'A': HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);   break;
            case 'B': HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET); break;
            case 'C': HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_1);                break;
        }
        HAL_UART_Receive_IT(&huart1, rxBuf, 1);  // 继续接收下一条命令
    }
}
```

---

## 19-7 空闲中断接收不定长数据 ⭐⭐⭐

**这是实际项目中最常用的接收方式**。无论发多长、什么时候发完，都能自动识别帧尾。

### 原理一句话

串口总线空闲超过一个字节的时间，硬件自动产生 IDLE 中断。我们在这个中断里把 DMA 收到的数据一次性取出来。

### CubeMX 配置步骤

1. Connectivity → **USART1** → Mode: **Asynchronous** → 参数按上面配置
2. **DMA Settings** 标签 → Add → **USART1_RX** → Mode: **Circular**
3. **NVIC Settings** 标签 → USART1 global interrupt → **Enable**
4. 生成代码

### 代码

```c
// main.c 全局变量（/* USER CODE BEGIN PV */）
uint8_t  rxBuf[256];
uint16_t rxLen = 0;

// main() 初始化中（/* USER CODE BEGIN 2 */）
HAL_UART_Receive_DMA(&huart1, rxBuf, sizeof(rxBuf));
__HAL_UART_ENABLE_IT(&huart1, UART_IT_IDLE);  // 使能空闲中断

// stm32f1xx_it.c → USART1_IRQHandler 函数中添加：
void USART1_IRQHandler(void)
{
    /* USER CODE BEGIN USART1_IRQn 0 */

    // 检测是否是空闲中断
    if (__HAL_UART_GET_FLAG(&huart1, UART_FLAG_IDLE))
    {
        __HAL_UART_CLEAR_IDLEFLAG(&huart1);  // 先清标志

        // 计算实际接收长度：
        // DMA 在 Circular 模式每搬运 1 字节，NDTR 减 1
        // 原始大小 - 当前剩余 = 已接收字节数
        rxLen = sizeof(rxBuf) - __HAL_DMA_GET_COUNTER(huart1.hdmarx);

        // 处理数据（这里示例只是通过串口发回去做回显测试）
        HAL_UART_Transmit(&huart1, rxBuf, rxLen, 100);

        // 重启 DMA 接收
        HAL_UART_DMAStop(&huart1);
        HAL_UART_Receive_DMA(&huart1, rxBuf, sizeof(rxBuf));
    }

    /* USER CODE END USART1_IRQn 0 */

    HAL_UART_IRQHandler(&huart1);  // 原有的库处理不要删
}
```

> 💡 `__HAL_DMA_GET_COUNTER` 返回 DMA 还剩多少字节没搬。`总长度 - 剩余 = 已接收`。

---

## 19-10 DMA + UART 原理

```
没有 DMA：
  CPU: "来一个字节？我搬。又来一个？我再搬..."
  结果：CPU 一直被打断，做不了别的事

有了 DMA：
  DMA 控制器：字节来了我自动搬到内存，搬完了通知 CPU
  CPU：收到通知才去处理数据，其他时间做自己的事
```

### DMA 配置要点

| 参数 | 值 | 为什么 |
|------|---|--------|
| Direction | Peripheral to Memory | 接收方向：串口→内存 |
| Mode | **Circular** | 循环模式：搬完一轮自动从头开始，不间断接收 |
| Data Width | Byte | 串口数据是字节 |
| Memory Increment | Enable | 内存地址要递增，否则数据全写在同一位置 |

---

## 踩坑记录

- [ ] printf 浮点数不显示 → 加 `-u _printf_float` 链接选项
- [ ] 中断接收只触发一次 → 回调里忘记重新调用 `HAL_UART_Receive_IT`
- [ ] 串口收到乱码 → 波特率不一致、没共地、TTL 电平不匹配
- [ ] IDLE+DMA 收不到 → 忘了 `__HAL_UART_ENABLE_IT(&huart1, UART_IT_IDLE)`
- [ ] TX/RX 接反了 → 自己的 TX 要接对方的 RX

---

*课程：[19-1] [19-2] [19-3] [19-4] [19-5] [19-6] [19-7] [19-8] [19-9] [19-10]*
