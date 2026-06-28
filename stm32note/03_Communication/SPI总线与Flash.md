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
  - spi
  - flash
  - w25q64
---
# SPI 总线与 Flash

> 能力定位：通过明确的片选和命令序列操作高速同步外设。
>
> 原始来源：[[13_SPI与存储器传感器]]。

## 信号与模式

| 信号 | 作用 |
|---|---|
| SCK | 主机输出时钟 |
| MOSI | 主机发送、从机接收 |
| MISO | 从机发送、主机接收 |
| CS | 选择当前从设备并界定事务 |

CPOL 决定空闲电平，CPHA 决定采样边沿。模式必须和从设备一致。

SPI 全双工意味着发送一个字节的同时也会接收一个字节。只读设备时，主机仍需发送 Dummy Byte 产生时钟。

## 片选

复杂协议通常使用 GPIO 手动控制 CS：拉低开始事务，按顺序发送命令/地址/数据，结束后拉高。

## W25Q64 命令流程

```text
读 JEDEC ID：CS低 -> 命令 -> Dummy/接收 -> CS高
写入：写使能 -> 页写入 -> 等待忙标志清除 -> 读回校验
擦除：写使能 -> 扇区擦除 -> 等待忙标志清除
```

页大小、扇区大小、命令字和跨页处理以实际芯片手册与课程资料为准。

状态寄存器用于判断写使能锁存和内部 BUSY。写使能后应确认状态，写入或擦除后要等待 BUSY 清除，再进行下一次事务或读回校验。

## 常用 HAL API

### API 速查

| HAL 函数 / 宏 | 作用 | 常见位置 |
|---|---|---|
| `HAL_SPI_Transmit()` | 在主机时钟下发送字节 | 命令、地址、写数据 |
| `HAL_SPI_Receive()` | 按当前 SPI 配置接收数据 | 特定接收流程 |
| `HAL_SPI_TransmitReceive()` | 同时发送 dummy/命令并接收返回字节 | 读 ID、状态、全双工设备 |
| `HAL_SPI_Transmit_DMA()` | DMA 发送一块数据 | 大块屏幕/存储写入 |
| `HAL_SPI_Receive_DMA()` | DMA 接收一块数据 | 连续接收场景 |
| `HAL_SPI_GetState()` | 查询 SPI HAL 状态 | 重启前、BUSY 诊断 |
| `HAL_SPI_GetError()` | 读取 SPI 错误码 | API 失败/回调错误后 |

### 使用注意

| HAL 函数 / 宏 | 前置条件 | 参数重点 | 返回值 / 阻塞与常见坑 |
|---|---|---|---|
| `HAL_SPI_Transmit()` | SPI 模式正确，目标 CS 已拉低 | buffer、元素数、Timeout | 返回 `HAL_StatusTypeDef`；阻塞；Size 受 Data Size 影响；事务结束前误拉高 CS |
| `HAL_SPI_Receive()` | 主机必须产生时钟 | buffer、元素数、Timeout | 返回状态；阻塞；忽略全双工读时仍需时钟；复杂命令读优先显式 TxRx |
| `HAL_SPI_TransmitReceive()` | CS 和 CPOL/CPHA 正确 | Tx/Rx buffer、Size、Timeout | 返回状态；阻塞；Tx/Rx 长度或缓冲区重叠处理错误；dummy 值未经协议确认 |
| `HAL_SPI_Transmit_DMA()` | TX DMA 已关联 | 长期有效 buffer、长度 | 返回状态；非阻塞；DMA 完成前拉高 CS 或修改 buffer |
| `HAL_SPI_Receive_DMA()` | RX DMA 已关联且时钟方案明确 | buffer、长度 | 返回状态；非阻塞；主机接收仍需产生 SCK；只开 RX DMA 不代表协议完整 |
| `HAL_SPI_GetState()` | SPI 已初始化 | SPI 句柄 | 返回 `HAL_SPI_StateTypeDef`；非阻塞；只等软件 READY，不检查 CS/时钟和 Flash BUSY |
| `HAL_SPI_GetError()` | 句柄有效 | SPI 句柄 | 返回错误位掩码；非阻塞；不检查返回值，问题发生后才读取已变化状态 |

### 典型调用顺序

```text
MX_SPIx_Init()
-> CS 默认拉高
-> CS 拉低
-> HAL_SPI_Transmit() 发送命令/地址
-> HAL_SPI_TransmitReceive() 发送 dummy 并读取数据
-> CS 拉高
-> 检查 HAL 返回值和设备状态
```

W25Q64 的读 ID、写使能、读状态、页写和扇区擦除是芯片命令序列，不是某一个 HAL 函数。命令字、页/扇区大小和忙位定义以具体 Flash 手册为准。

### 什么时候用哪个函数？

| 场景 | 推荐函数 |
|---|---|
| 发送命令、地址或写数据 | `HAL_SPI_Transmit()` |
| 同一时钟周期边发边收 | `HAL_SPI_TransmitReceive()` |
| 主机读取寄存器/Flash 数据 | 手动 CS + 发送命令 + 显式 dummy/TxRx |
| 大块连续发送 | `HAL_SPI_Transmit_DMA()` |
| 通信失败 | 先查 CS、CPOL/CPHA、SCK/MISO，再看 `GetState/GetError()` |

## 关联知识

- [[DMA与缓冲区]]
- [[BMP280_SPL06气压计]]
- [[通用调试方法]]

---

## 本节目标

- 能解释四线 SPI、CPOL/CPHA 和 Dummy Byte
- 能手动控制 CS 完成完整事务
- 能实现 W25Q64 读 ID、擦写和读回校验流程

## 知识地图

| 名词 | 工程含义 |
|---|---|
| SCK/MOSI/MISO | 时钟/主发/主收信号 |
| CS | 从机选择和事务边界 |
| CPOL/CPHA | 时钟空闲电平和采样相位 |
| Dummy Byte | 只读时用于产生时钟的发送字节 |
| WEL/BUSY | 写使能锁存/内部忙状态 |
| 页/扇区 | 编程与擦除粒度 |

## 本质理解

SPI 只规定同步移位，不规定命令含义。CS 拉低后发送什么命令、地址和数据，完全由从设备协议决定。Flash 可靠写入的关键不是 `HAL_SPI_Transmit`，而是写使能、擦除、跨页和忙等待。


## STM32F103 / HAL 中的实现方式

- F103 SPI 主机通常使用 SCK/MOSI 复用推挽、MISO 输入，CS 可由普通 GPIO 手动控制。
- 全双工意味着接收时仍要发送 Dummy Byte。
- W25Q64 写/擦前需要写使能，操作后轮询状态忙位；页大小、扇区大小和命令字以具体型号手册为准。

## CubeMX 配置要点

1. 选择 SPI Full-Duplex Master。
2. 按器件手册配置 Data Size、First Bit、CPOL、CPHA 和 Prescaler。
3. 配置 SCK/MOSI/MISO 引脚。
4. CS 配置为 GPIO 输出并默认拉高。
5. 先降低速率读 ID，再逐步提高。

## 最小实验 / 最小框架

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
static void Flash_Select(void)
{
    HAL_GPIO_WritePin(FLASH_CS_GPIO_Port, FLASH_CS_Pin, GPIO_PIN_RESET);
}

static void Flash_Deselect(void)
{
    HAL_GPIO_WritePin(FLASH_CS_GPIO_Port, FLASH_CS_Pin, GPIO_PIN_SET);
}

static uint8_t SPI_TransferByte(uint8_t tx, uint8_t *rx)
{
    return HAL_SPI_TransmitReceive(&hspi1, &tx, rx, 1U,
                                   SPI_TIMEOUT_MS) == HAL_OK;
}

static uint8_t Flash_ReadJedecId(uint8_t *id, uint16_t length)
{
    uint8_t command = FLASH_CMD_READ_JEDEC_ID;
    Flash_Select();
    HAL_StatusTypeDef status = HAL_SPI_Transmit(&hspi1, &command, 1U,
                                                SPI_TIMEOUT_MS);
    if (status == HAL_OK)
    {
        status = HAL_SPI_Receive(&hspi1, id, length, SPI_TIMEOUT_MS);
    }
    Flash_Deselect();
    return status == HAL_OK;
}

static uint8_t Flash_WriteEnable(void)
{
    uint8_t command = FLASH_CMD_WRITE_ENABLE;
    Flash_Select();
    HAL_StatusTypeDef status = HAL_SPI_Transmit(&hspi1, &command, 1U,
                                                SPI_TIMEOUT_MS);
    Flash_Deselect();
    return status == HAL_OK;
}

static uint8_t Flash_ReadStatus(uint8_t *status_reg)
{
    uint8_t command = FLASH_CMD_READ_STATUS;
    Flash_Select();
    HAL_StatusTypeDef status = HAL_SPI_Transmit(&hspi1, &command, 1U,
                                                SPI_TIMEOUT_MS);
    if (status == HAL_OK)
    {
        status = HAL_SPI_Receive(&hspi1, status_reg, 1U, SPI_TIMEOUT_MS);
    }
    Flash_Deselect();
    return status == HAL_OK;
}

static uint8_t Flash_WaitReady(uint32_t timeout_ms)
{
    uint32_t start = HAL_GetTick();
    uint8_t status_reg;
    do
    {
        if (!Flash_ReadStatus(&status_reg)) return 0U;
        if ((status_reg & FLASH_STATUS_BUSY_MASK) == 0U) return 1U;
    } while ((uint32_t)(HAL_GetTick() - start) < timeout_ms);
    return 0U;
}

static uint8_t Flash_PageProgramAndVerify(uint32_t address,
                                          const uint8_t *data,
                                          uint16_t length)
{
    uint8_t verify[FLASH_VERIFY_BUFFER_SIZE];
    if ((length > sizeof(verify)) || !Flash_WriteEnable()) return 0U;
    if (!Flash_PageProgram(address, data, length)) return 0U;
    if (!Flash_WaitReady(FLASH_OPERATION_TIMEOUT_MS)) return 0U;
    if (!Flash_Read(address, verify, length)) return 0U;
    return memcmp(data, verify, length) == 0;
}
```

命令字、状态位、地址宽度、页边界、擦除粒度和容量均使用工程宏/驱动函数表示，必须按实际 Flash 型号手册确认。页写前还要由上层保证不跨越器件页边界。

## 调试方法

```text
确认供电/共地
-> 确认 CS 默认高且事务时拉低
-> 核对 CPOL/CPHA
-> 低速读取 JEDEC ID
-> 逻辑分析仪看命令/地址/数据
-> 验证写使能和 BUSY
-> 擦写后读回比较
```

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| 读全 FF | CS 未拉低或 MISO 悬空 | 检查 CS/MISO |
| 读数错位 | CPOL/CPHA 错 | 按手册调整模式 |
| 写不进去 | 忘记 WREN | 每次写/擦前写使能 |
| 数据仍旧 | 未擦除或忙未结束 | 擦除并等待 BUSY |
| 跨页异常 | 页写越界 | 按页拆分 |
| 偶发失败 | 速率过高/接线长 | 降低 SPI 频率并改善布线 |

## 与其它章节的关系

- [[BMP280_SPL06气压计]] 也可能使用 SPI。
- [[DMA与缓冲区]] 可扩展大块收发。
- [[传感器采集与显示]] 复用读 ID/配置/读数据流程。

## 复习检查清单

- [ ] 能否不用看代码说清本章数据流和硬件链路？
- [ ] 能否在 CubeMX 中完成最小配置并说明每个关键选项？
- [ ] 能否写出初始化、启动和回调/轮询的最小框架？
- [ ] 能否用原始电平、波形、返回值或寄存器证明外设在工作？
- [ ] 能否按“硬件 -> 时钟/配置 -> Start -> 回调 -> 业务”排查故障？
- [ ] 能否指出本章哪些参数必须按原理图、手册或实测确认？
