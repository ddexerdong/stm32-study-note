# 19 串口 UART（核心章节）

## UART 基础配置（CubeMX）

| 参数 | 常用值 | 说明 |
|------|--------|------|
| Baud Rate | 115200 | 波特率 |
| Word Length | 8 Bits | 数据位 |
| Parity | None | 校验位 |
| Stop Bits | 1 | 停止位 |
| Mode | TX/RX | 收发 |

## 三种接收方式对比 ⭐

| 方式 | 函数 | 特点 | 适用 |
|------|------|------|------|
| 阻塞接收 | `HAL_UART_Receive` | 死等CPU停住 | 简单测试 |
| 中断接收 | `HAL_UART_Receive_IT` | 接完N字节触发回调 | 定长数据 |
| 空闲中断+DMA | IDLE+DMA | 不定长，高效 | ⭐实际项目首选 |

---

## 19-1 发送字节

```c
uint8_t data = 'A';
HAL_UART_Transmit(&huart1, &data, 1, 100);
```

## 19-2 发送字符串 + printf重定向

```c
// 发送字符串
char *str = "Hello STM32\r\n";
HAL_UART_Transmit(&huart1, (uint8_t*)str, strlen(str), 100);

// printf重定向（加在 main.c 里）
#include <stdio.h>
int fputc(int ch, FILE *f) {
    HAL_UART_Transmit(&huart1, (uint8_t*)&ch, 1, 100);
    return ch;
}

// 之后就能直接用
printf("ADC值: %d\r\n", adcValue);
```

> ⚠️ CubeIDE需要在编译选项中添加 `-u _printf_float` 才能打印浮点数

## 19-4 阻塞接收定长数据

```c
uint8_t rxBuf[10];
HAL_UART_Receive(&huart1, rxBuf, 10, 1000);  // 等待接收10字节，超时1000ms
```

## 19-5 中断接收定长数据

```c
uint8_t rxBuf[10];

// main函数中启动一次中断接收
HAL_UART_Receive_IT(&huart1, rxBuf, 10);

// 接收完10字节后自动调用此回调
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    if (huart->Instance == USART1) {
        // 处理 rxBuf 中的数据
        
        // 重新启动接收（否则只接收一次）
        HAL_UART_Receive_IT(&huart1, rxBuf, 10);
    }
}
```

## 19-6 串口控制LED ⭐

### 协议设计思路

```
命令格式：单字节命令
  'A' → LED1亮
  'B' → LED1灭
  'C' → LED2亮
  ...
```

```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    if (huart->Instance == USART1) {
        switch(rxBuf[0]) {
            case 'A': HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);   break;
            case 'B': HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET); break;
            case 'C': HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_1); break;
        }
        HAL_UART_Receive_IT(&huart1, rxBuf, 1);
    }
}
```

## 19-7 空闲中断接收不定长数据 ⭐⭐⭐

这是最重要的接收方式，实际项目必用！

### 配置步骤

1. CubeMX开启 DMA（USART1_RX，Circular模式）
2. 开启 USART1 global interrupt

### 代码

```c
// main.c 变量定义
uint8_t rxBuf[256];
uint16_t rxLen = 0;

// main函数中启动DMA接收
HAL_UART_Receive_DMA(&huart1, rxBuf, sizeof(rxBuf));
__HAL_UART_ENABLE_IT(&huart1, UART_IT_IDLE);  // 使能空闲中断

// stm32f1xx_it.c 中的 USART1_IRQHandler 添加
void USART1_IRQHandler(void) {
    if (__HAL_UART_GET_FLAG(&huart1, UART_FLAG_IDLE)) {
        __HAL_UART_CLEAR_IDLEFLAG(&huart1);
        
        // 计算实际接收长度
        rxLen = sizeof(rxBuf) - __HAL_DMA_GET_COUNTER(huart1.hdmarx);
        
        // 处理数据
        processData(rxBuf, rxLen);
        
        // 重启DMA
        HAL_UART_DMAStop(&huart1);
        HAL_UART_Receive_DMA(&huart1, rxBuf, sizeof(rxBuf));
    }
    HAL_UART_IRQHandler(&huart1);
}
```

## 19-10 DMA + UART 原理

```
没有DMA：CPU → 一个字节一个字节搬运 → 占用大量CPU时间
有了DMA：DMA控制器直接搬运数据 → CPU可以做其他事
         搬完了 → 中断通知CPU
```

### DMA配置要点

| 参数 | 值 |
|------|---|
| Direction | Peripheral to Memory（接收）|
| Mode | Circular（循环，不停接收）|
| Data Width | Byte |
| Memory Increment | Enable（内存地址自增）|

## 踩坑记录

- [ ] printf浮点数不显示 → 加 `-u _printf_float`
- [ ] 中断接收只触发一次 → 回调里忘记重新调用 `Receive_IT`
- [ ] 

---
*课程：[19-1] [19-2] [19-3] [19-4] [19-5] [19-6] [19-7] [19-8] [19-9] [19-10]*
