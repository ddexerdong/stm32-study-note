# 13 SPI 与存储器传感器

> 覆盖课程：[35] [36] [37-1] [37-2]
>
> 目标：理解 SPI 同步串行通信的工程本质，掌握 CubeMX 中 SPI 主机配置思路，并用 W25Q64 Flash 的读 ID、写使能、页写入、扇区擦除、读回校验建立“按命令协议操作外部芯片”的方法；BMP280、SPL06 只整理为气压计传感器学习框架。
>
> 关联章节：[[11_I2C与OLED|I2C]] · [[12_I2C传感器|传感器 bring-up]] · [[STM32术语表|术语表]]

---

## 本节目标

完成本节后，应能做到：

- 说清楚 SPI 中 SCK、MOSI、MISO、CS 分别在通信里做什么。
- 区分 SPI 的模式、片选、全双工、命令字、地址字节和数据字节。
- 在 CubeMX 中配置 STM32F103 的 SPI 主机模式。
- 按“读 ID -> 写使能 -> 擦除 -> 页写入 -> 等待忙状态 -> 读回校验”的顺序理解 W25Q64 Flash 实验。
- 把 BMP280、SPL06 这类气压计传感器拆成通信确认、读 ID、读校准参数、读原始值、公式换算、显示输出几个阶段。

---

## 核心概念

### 知识地图

| 名词 | 工程含义 | 常见位置 |
|------|----------|----------|
| SPI | 同步串行总线，常用于 Flash、屏幕、传感器等外设 | `[35]` |
| SCK | Serial Clock，主机输出时钟 | SPI 引脚 |
| MOSI | Master Out Slave In，主机发给从机的数据线 | SPI 引脚 |
| MISO | Master In Slave Out，从机回给主机的数据线 | SPI 引脚 |
| CS / NSS | 片选信号，被拉低的设备才响应本次通信 | GPIO 或 SPI NSS |
| CPOL / CPHA | 决定时钟空闲电平和采样边沿 | CubeMX SPI 参数 |
| 全双工 | 主机每发出 1 bit，同时也会收到 1 bit | SPI 收发函数 |
| Dummy Byte | 为了读数据而发送的占位字节 | 读 ID / 读数据 |
| W25Q64 | 串行 NOR Flash 模块，按命令字读写擦除 | `[36]` |
| JEDEC ID | Flash 常见识别码，用来确认通信和芯片型号 | Flash 读 ID |
| 写使能 WEL | 写入或擦除前必须先置位的状态 | Flash 状态寄存器 |
| 忙状态 WIP / BUSY | Flash 内部擦写未完成时的状态 | Flash 状态寄存器 |
| 页写入 | Flash 常见的最小连续编程单位 | Flash 写数据 |
| 扇区擦除 | 写入前常需要先擦除到空态 | Flash 擦除 |
| BMP280 / SPL06 | 气压计传感器，通常需要读校准参数并换算 | `[37-1] [37-2]` |

### 常用 HAL API

| API / 宏 | 作用 | 常见用法 |
|----------|------|----------|
| `HAL_SPI_Init()` | 初始化 SPI 外设 | 通常由 CubeMX 生成 |
| `HAL_SPI_Transmit()` | SPI 发送数据 | 发送命令、地址、待写数据 |
| `HAL_SPI_Receive()` | SPI 接收数据 | 简单接收场景 |
| `HAL_SPI_TransmitReceive()` | SPI 同时发送和接收 | 读 ID、读寄存器、读 Flash |
| `HAL_GPIO_WritePin()` | 手动控制 CS 片选 | 片选拉低/拉高 |
| `HAL_UART_Transmit()` | 打印 ID、状态、原始值 | 调试输出 |

---

## 本质理解

SPI 的本质是：**主机用一根时钟线推动移位寄存器交换数据，再用片选线决定当前和哪个从设备说话。**

```text
STM32 主机
  SCK  ---------------- Flash / 传感器
  MOSI ---------------- Flash / 传感器
  MISO ---------------- Flash / 传感器
  CS1  ---------------- W25Q64
  CS2  ---------------- BMP280 / SPL06
```

I2C 更像“先喊地址，再读写寄存器”；SPI 更像“先拉低片选，再按设备手册发命令字和后续字节”。Flash、气压计、屏幕都可能用 SPI，但它们真正的区别不在 SPI 总线本身，而在各自的命令协议和寄存器含义。

W25Q64 的工程重点不是背 API，而是记住 Flash 的约束：

```text
读：片选 -> 命令 -> 地址 -> 接收数据
写：写使能 -> 页写入 -> 查忙 -> 读回校验
擦：写使能 -> 扇区擦除 -> 查忙 -> 再写
```

BMP280、SPL06 这类气压计不要一上来抄完整公式。我的学习顺序应该是先证明通信通了，再证明 ID 和原始数据对，最后才看校准参数和补偿公式。

W25Q64 的读写擦除流程可由 PDF 的 SPI/串行 Flash 章节支撑；BMP280、SPL06 的课程细节在当前 PDF 和已有笔记里支撑不足。

> 视频待核对：课程使用的 BMP280、SPL06 通信方式、模块地址或片选引脚、寄存器表、校准参数读取位置、补偿公式、库文件来源和显示格式。

以实际开发板原理图和 CubeMX 配置为准。

---

## 工作流程 / 原理

### SPI 一次事务

```text
CS 拉低
-> 发送命令字
-> 发送地址字节或寄存器地址
-> 发送 Dummy Byte 或数据
-> 接收从设备返回
-> CS 拉高
```

片选必须包住一次完整事务。很多 SPI 设备都是在 CS 拉低后开始解析命令，CS 拉高后结束本次命令。

### W25Q64 读 ID

```text
CS 拉低
-> 发送 JEDEC ID 命令
-> 连续发送 Dummy Byte
-> 接收厂家 ID / 存储类型 / 容量信息
-> CS 拉高
-> 串口打印 ID
```

PDF 的串行 Flash 示例中能看到 JEDEC ID、写使能、读状态、页写入、扇区擦除等典型命令流程。命令字虽然有资料支撑，复习时仍要对照课程代码和芯片手册。

### W25Q64 写入和校验

```text
读 ID 确认芯片
-> 写使能
-> 擦除目标扇区
-> 轮询忙状态直到完成
-> 再次写使能
-> 页写入
-> 轮询忙状态直到完成
-> 读回同一地址
-> 对比写入数组和读出数组
```

Flash 常见坑是“没擦就写”“没写使能就写”“写完没等忙状态清零”“跨页写入没拆分”。页大小、扇区大小、地址范围和跨页处理细节：

> 视频待核对：课程 `[36]` 使用的 W25Q64 页大小、扇区大小、命令宏、片选 GPIO、SPI 模式和跨页写入封装。

以实际开发板原理图和 CubeMX 配置为准。

### BMP280 / SPL06 学习框架

```text
确认模块通信方式：I2C 还是 SPI
-> 确认地址或 CS 接线
-> 读芯片 ID
-> 写工作模式 / 采样参数
-> 读取校准参数
-> 读取原始温度和气压
-> 用手册或课程公式换算
-> 串口或 OLED 显示
```

这两个气压计的难点通常不在 HAL SPI 本身，而在补偿算法和校准参数格式。当前阶段只记录学习路径，不写完整驱动教程。

> 视频待核对：BMP280、SPL06 在课程中使用 I2C 还是 SPI；如果使用 SPI，CS 引脚、SPI 模式、读写位规则、寄存器地址处理方式都需要核对。

---

## CubeMX / MDK 配置

### SPI 主机基础配置

1. 选择当前 STM32F103 工程或新建工程。
2. 配置系统时钟，先沿用前面已经验证过的时钟方案。
3. 在 `Connectivity` 中选择课程使用的 `SPI1` 或 `SPI2`。
4. Mode 选择 `Full-Duplex Master`。
5. Data Size 选择 8 bit。
6. First Bit 一般选择 MSB First，具体以外设手册为准。
7. CPOL / CPHA 按 W25Q64 或传感器手册配置，不确定时按课程视频核对。
8. Baud Rate Prescaler 先选保守分频，读 ID 稳定后再提高速度。
9. NSS 建议基础阶段使用 Software，CS 片选单独配置为 GPIO Output。
10. 配置 CS 引脚为推挽输出，默认输出高电平。
11. 基础轮询读写先不启用 NVIC / DMA。
12. 如果要串口打印调试信息，确认 USART 已配置并能输出。
13. 生成 MDK 工程。
14. 用户代码写在 `USER CODE` 区域；W25Q64 驱动建议放在 `BSP/w25q64.c` 和 `BSP/w25q64.h`。

SPI 引脚、CS 引脚、SPI 模式、分频、模块供电：

以实际开发板原理图和 CubeMX 配置为准。

### MDK 文件组织

```text
BSP/
├── w25q64.c
├── w25q64.h
├── barometer_xxx.c
└── barometer_xxx.h
```

1. 新建 `BSP/` 目录。
2. 把驱动 `.c` 文件加入 MDK 工程分组。
3. 把 `BSP/` 加入 Include Path。
4. `main.c` 只做实验流程调用和调试输出。
5. 底层驱动不要直接写业务判断，方便后续迁移到项目里。

---

## 最小代码

以下代码是 SPI Flash 和气压计传感器的最小示例框架，不声称可直接编译运行。W25Q64 命令字参考 PDF 中串行 Flash 的常见流程，实际仍要按课程代码和芯片手册核对。

### `BSP/w25q64.h`

> 代码性质：示例框架，用于理解流程，不能保证直接编译。

```c
#ifndef W25Q64_H
#define W25Q64_H

#include "main.h"

uint32_t W25Q64_ReadId(void);
HAL_StatusTypeDef W25Q64_SectorErase(uint32_t addr);
HAL_StatusTypeDef W25Q64_PageProgram(uint32_t addr, const uint8_t *buf, uint16_t len);
HAL_StatusTypeDef W25Q64_Read(uint32_t addr, uint8_t *buf, uint16_t len);

#endif
```

### `BSP/w25q64.c`

> 代码性质：示例框架，用于理解流程，不能保证直接编译。

```c
#include "w25q64.h"
#include "spi.h"

#define W25Q64_SPI_HANDLE       hspi1

#define W25Q64_CS_GPIO_Port     GPIOA
#define W25Q64_CS_Pin           GPIO_PIN_4

#define W25Q64_CMD_WRITE_ENABLE 0x06
#define W25Q64_CMD_READ_STATUS1 0x05
#define W25Q64_CMD_READ_DATA    0x03
#define W25Q64_CMD_PAGE_PROGRAM 0x02
#define W25Q64_CMD_SECTOR_ERASE 0x20
#define W25Q64_CMD_JEDEC_ID     0x9F

#define W25Q64_STATUS_BUSY      0x01

static void W25Q64_Select(void)
{
    HAL_GPIO_WritePin(W25Q64_CS_GPIO_Port, W25Q64_CS_Pin, GPIO_PIN_RESET);
}

static void W25Q64_Unselect(void)
{
    HAL_GPIO_WritePin(W25Q64_CS_GPIO_Port, W25Q64_CS_Pin, GPIO_PIN_SET);
}

static HAL_StatusTypeDef W25Q64_WriteEnable(void)
{
    uint8_t cmd = W25Q64_CMD_WRITE_ENABLE;
    HAL_StatusTypeDef ret;

    W25Q64_Select();
    ret = HAL_SPI_Transmit(&W25Q64_SPI_HANDLE, &cmd, 1, 100);
    W25Q64_Unselect();

    return ret;
}

static HAL_StatusTypeDef W25Q64_ReadStatus(uint8_t *status)
{
    uint8_t tx[2] = {W25Q64_CMD_READ_STATUS1, 0xFF};
    uint8_t rx[2] = {0};
    HAL_StatusTypeDef ret;

    W25Q64_Select();
    ret = HAL_SPI_TransmitReceive(&W25Q64_SPI_HANDLE, tx, rx, sizeof(tx), 100);
    W25Q64_Unselect();

    *status = rx[1];
    return ret;
}

static HAL_StatusTypeDef W25Q64_WaitBusy(void)
{
    uint8_t status = 0;
    HAL_StatusTypeDef ret;

    do
    {
        ret = W25Q64_ReadStatus(&status);
        if (ret != HAL_OK)
        {
            return ret;
        }
    } while ((status & W25Q64_STATUS_BUSY) != 0);

    return HAL_OK;
}

uint32_t W25Q64_ReadId(void)
{
    uint8_t tx[4] = {W25Q64_CMD_JEDEC_ID, 0xFF, 0xFF, 0xFF};
    uint8_t rx[4] = {0};

    W25Q64_Select();
    (void)HAL_SPI_TransmitReceive(&W25Q64_SPI_HANDLE, tx, rx, sizeof(tx), 100);
    W25Q64_Unselect();

    return ((uint32_t)rx[1] << 16) | ((uint32_t)rx[2] << 8) | rx[3];
}

HAL_StatusTypeDef W25Q64_SectorErase(uint32_t addr)
{
    uint8_t cmd[4];
    HAL_StatusTypeDef ret;

    ret = W25Q64_WriteEnable();
    if (ret != HAL_OK)
    {
        return ret;
    }

    cmd[0] = W25Q64_CMD_SECTOR_ERASE;
    cmd[1] = (uint8_t)(addr >> 16);
    cmd[2] = (uint8_t)(addr >> 8);
    cmd[3] = (uint8_t)addr;

    W25Q64_Select();
    ret = HAL_SPI_Transmit(&W25Q64_SPI_HANDLE, cmd, sizeof(cmd), 100);
    W25Q64_Unselect();

    if (ret != HAL_OK)
    {
        return ret;
    }

    return W25Q64_WaitBusy();
}

HAL_StatusTypeDef W25Q64_PageProgram(uint32_t addr, const uint8_t *buf, uint16_t len)
{
    uint8_t cmd[4];
    HAL_StatusTypeDef ret;

    ret = W25Q64_WriteEnable();
    if (ret != HAL_OK)
    {
        return ret;
    }

    cmd[0] = W25Q64_CMD_PAGE_PROGRAM;
    cmd[1] = (uint8_t)(addr >> 16);
    cmd[2] = (uint8_t)(addr >> 8);
    cmd[3] = (uint8_t)addr;

    W25Q64_Select();
    ret = HAL_SPI_Transmit(&W25Q64_SPI_HANDLE, cmd, sizeof(cmd), 100);
    if (ret == HAL_OK)
    {
        ret = HAL_SPI_Transmit(&W25Q64_SPI_HANDLE, (uint8_t *)buf, len, 100);
    }
    W25Q64_Unselect();

    if (ret != HAL_OK)
    {
        return ret;
    }

    return W25Q64_WaitBusy();
}

HAL_StatusTypeDef W25Q64_Read(uint32_t addr, uint8_t *buf, uint16_t len)
{
    uint8_t cmd[4];
    HAL_StatusTypeDef ret;

    cmd[0] = W25Q64_CMD_READ_DATA;
    cmd[1] = (uint8_t)(addr >> 16);
    cmd[2] = (uint8_t)(addr >> 8);
    cmd[3] = (uint8_t)addr;

    W25Q64_Select();
    ret = HAL_SPI_Transmit(&W25Q64_SPI_HANDLE, cmd, sizeof(cmd), 100);
    if (ret == HAL_OK)
    {
        ret = HAL_SPI_Receive(&W25Q64_SPI_HANDLE, buf, len, 100);
    }
    W25Q64_Unselect();

    return ret;
}
```

### `BSP/barometer_xxx.c`

BMP280、SPL06 只保留气压计传感器框架：

> 代码性质：示例框架，用于理解流程，不能保证直接编译。

```c
#include "spi.h"
#include "main.h"

#define BARO_SPI_HANDLE      hspi1
#define BARO_CS_GPIO_Port    GPIOA
#define BARO_CS_Pin          GPIO_PIN_4

#define BARO_REG_ID          0x00
#define BARO_REG_CALIB_START 0x00
#define BARO_REG_RAW_START   0x00

static void Baro_Select(void)
{
    HAL_GPIO_WritePin(BARO_CS_GPIO_Port, BARO_CS_Pin, GPIO_PIN_RESET);
}

static void Baro_Unselect(void)
{
    HAL_GPIO_WritePin(BARO_CS_GPIO_Port, BARO_CS_Pin, GPIO_PIN_SET);
}

uint8_t Baro_ReadId(void)
{
    uint8_t tx[2] = {BARO_REG_ID, 0xFF};
    uint8_t rx[2] = {0};

    Baro_Select();
    (void)HAL_SPI_TransmitReceive(&BARO_SPI_HANDLE, tx, rx, sizeof(tx), 100);
    Baro_Unselect();

    return rx[1];
}

uint8_t Baro_ReadRaw(uint8_t *buf, uint16_t len)
{
    uint8_t reg = BARO_REG_RAW_START;
    HAL_StatusTypeDef ret;

    Baro_Select();
    ret = HAL_SPI_Transmit(&BARO_SPI_HANDLE, &reg, 1, 100);
    if (ret == HAL_OK)
    {
        ret = HAL_SPI_Receive(&BARO_SPI_HANDLE, buf, len, 100);
    }
    Baro_Unselect();

    return ret == HAL_OK;
}

void Baro_ConvertExample(const uint8_t *raw)
{
    /*
     * 校准参数、原始值拼接方式和补偿公式必须按具体芯片手册或课程代码填写。
     */
    (void)raw;
}
```

### `Core/Src/main.c`

> 代码性质：示例框架，用于理解流程，不能保证直接编译。

```c
#include "main.h"
#include "spi.h"
#include "usart.h"
#include "w25q64.h"

int main(void)
{
    uint32_t flash_id;
    uint8_t write_buf[] = {0x11, 0x22, 0x33, 0x44};
    uint8_t read_buf[sizeof(write_buf)] = {0};

    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_SPI1_Init();
    MX_USART1_UART_Init();

    flash_id = W25Q64_ReadId();
    (void)flash_id;

    (void)W25Q64_SectorErase(0x000000);
    (void)W25Q64_PageProgram(0x000000, write_buf, sizeof(write_buf));
    (void)W25Q64_Read(0x000000, read_buf, sizeof(read_buf));

    while (1)
    {
        /*
         * 对比 write_buf 和 read_buf。
         * 实际项目里建议把结果通过串口打印出来。
         */
    }
}
```

---

## 调试方法

### 供电

- 先确认模块电压等级，不要默认所有模块都能接 5V。
- Flash 或传感器供电不稳时，读 ID 可能偶尔正确、偶尔全 0 或全 0xFF。
- 大电流外设和数字传感器混用时，优先检查电源纹波和模块稳压。

### 共地

- STM32、Flash、传感器、USB 串口工具必须共地。
- 没有共地时，SPI 波形看似有跳变，但从设备不一定能识别电平。

### 接线

- SCK、MOSI、MISO 不要接反。
- 每个 SPI 从设备都需要独立 CS，多个设备共用一个 CS 会互相干扰。
- CS 空闲应保持高电平。
- W25Q64、BMP280、SPL06 的接线方式：以实际开发板原理图和 CubeMX 配置为准。

### CubeMX 配置

- 确认 SPI 是 Master，不是 Slave。
- 确认 CPOL / CPHA 与外设要求一致。
- 确认 NSS 使用软件管理时，CS GPIO 已单独配置为输出。
- 分频先保守，读 ID 稳定后再提高速度。

### HAL API 返回值

- 每个 `HAL_SPI_Transmit()`、`HAL_SPI_Receive()`、`HAL_SPI_TransmitReceive()` 都先检查返回值。
- 返回 `HAL_TIMEOUT` 时重点看时钟、接线、句柄和超时时间。
- 返回 `HAL_BUSY` 时检查是否有前一次通信未结束或 DMA/中断状态未清。

### 逻辑分析仪 / 串口打印

- 逻辑分析仪看 CS 是否包住完整命令。
- 看 SCK 是否有波形、MOSI 命令字是否正确、MISO 是否有响应。
- 串口先打印 ID、状态寄存器、原始读回数组，不要直接打印换算后的结果。

### 外设协议

- Flash 写入前必须写使能。
- 写入或擦除后必须等待忙状态结束。
- 页写入不能随意跨页，跨页时要拆分。
- 气压计必须按手册读取校准参数，再换算温度/气压。

### 业务逻辑

- 如果读 ID 不对，不要继续调页写入或气压换算。
- 如果写入后读回不一致，先检查擦除、写使能、忙状态和地址。
- 如果气压计原始值一直不变，先检查采样模式和数据就绪状态。

---

## 工程意义

SPI Flash 代表的是“外部非易失存储”：可以保存配置、日志、离线数据、字体库、图片资源。掌握 W25Q64 读写擦除后，后面接文件系统、参数保存、数据记录会更容易。

BMP280、SPL06 代表的是“带校准参数的复杂传感器”：通信只是第一层，真正难点是按芯片规则把原始值换算成有物理意义的数据。这个套路也适用于很多 IMU、环境传感器和医疗传感器。

---

## 易混淆点

| 容易混淆 | 正确理解 |
|----------|----------|
| SPI 有地址线 | SPI 本身没有 I2C 那种地址，靠 CS 选择设备 |
| `HAL_SPI_Receive()` 只读不发 | SPI 接收也需要时钟，主机本质上仍在发送占位数据 |
| CS 可以一直拉低 | 有些设备可以，有些命令需要 CS 上升沿结束事务，基础阶段按一次命令一次 CS 处理 |
| Flash 写入前不用擦除 | NOR Flash 通常只能把 1 写成 0，重新写前常要擦除 |
| 写使能一次永久有效 | 写使能通常只对下一次写/擦操作有效 |
| 读到 ID 就等于驱动完成 | ID 只证明通信基本通了，写入、擦除、忙状态、边界处理还要单独验证 |
| 气压计公式可以随便套 | 校准参数格式和补偿公式与芯片强相关，不能混用 |

---

## 我的理解

我现在把 SPI 理解成“STM32 拿时钟推着外设交换字节”。总线本身并不复杂，真正容易错的是每个芯片自己的命令协议。

以后做 SPI Flash，我会先按这个顺序检查：CS 是否正确拉低、读 ID 是否稳定、写使能是否执行、擦除后是否等待忙状态、页写入有没有跨页、最后读回数组是否逐字节一致。

对于 BMP280、SPL06，我不会一开始就纠结气压换算结果准不准。我会先看 ID，再看原始温度/气压有没有变化，再去核对校准参数和公式。

---

## [37-1] BMP280气压计实验

> 来源：PDF 弱对应 + 视频待核对 + 模块资料待核对

> 视频待核对：BMP280 通信方式、I2C 地址或 SPI CS、寄存器地址、校准参数读取方式、补偿公式、库文件来源和显示格式。

### 实验定位

BMP280 是气压计传感器实验。PDF 的 I2C/SPI 章节只能支撑通信底层，不能凭空补完整寄存器表和补偿公式。

### Bring-up 框架

```text
确认通信方式
-> 读设备 ID
-> 读取校准参数
-> 配置采样和模式
-> 读取原始温度/气压
-> 套用补偿公式
-> 串口或 OLED 显示
```

### 最小代码

常见位置：`BSP/bmp280.c`

> 代码性质：示例框架，用于理解流程，不能保证直接编译。

```c
uint8_t BMP280_CheckId(void)
{
    uint8_t id = 0;

    /* 视频待核对：通信方式、设备地址/CS、ID 寄存器和期望值。 */
    (void)id;
    return 0;
}

uint8_t BMP280_ReadRaw(int32_t *raw_temp, int32_t *raw_press)
{
    if (raw_temp == NULL || raw_press == NULL)
    {
        return 0;
    }

    /* 视频待核对：原始温度/气压寄存器和字节拼接方式。 */
    return 0;
}

uint8_t BMP280_Compensate(void)
{
    /* 视频待核对：校准参数和补偿公式。 */
    return 0;
}
```

### 调试方法

- 先读 ID，确认通信方式和片选/地址。
- 再读校准参数，确认不是全 0 或全 `0xFF`。
- 原始值稳定后再接补偿公式。

### 常见坑

- 不确认模块是 I2C 还是 SPI。
- 校准参数字节序读错。
- 把原始气压值直接当物理气压显示。

## [37-2] SPL06气压计实验

> 来源：PDF 弱对应 + 视频待核对 + 模块资料待核对

> 视频待核对：SPL06 通信方式、I2C 地址或 SPI CS、寄存器、校准参数、补偿公式、库来源和课程显示方式。

### 实验定位

SPL06 和 BMP280 同属气压计 bring-up 类课程，但寄存器、校准参数和补偿公式不能互相套用。

### Bring-up 框架

```text
确认通信方式
-> 读 ID
-> 读校准参数
-> 配置温度/气压采样
-> 读取原始温度/气压
-> 按 SPL06 公式补偿
-> 显示气压/温度/高度估算
```

### 最小代码

常见位置：`BSP/spl06.c`

> 代码性质：示例框架，用于理解流程，不能保证直接编译。

```c
uint8_t SPL06_CheckId(void)
{
    /* 视频待核对：ID 寄存器、期望值和通信封装。 */
    return 0;
}

uint8_t SPL06_ReadRaw(int32_t *raw_temp, int32_t *raw_press)
{
    if (raw_temp == NULL || raw_press == NULL)
    {
        return 0;
    }

    /* 视频待核对：原始数据寄存器、符号扩展和字节顺序。 */
    return 0;
}

uint8_t SPL06_Compensate(void)
{
    /* 视频待核对：SPL06 校准参数和补偿公式。 */
    return 0;
}
```

### 调试方法

- 不要把 BMP280 的寄存器表和公式搬到 SPL06。
- 原始值读取、校准参数读取、补偿公式分三步验证。
- 显示高度估算前，先确认温度和气压基本合理。

### 常见坑

- I2C/SPI 底层通了，就误以为气压计驱动完整。
- 补偿公式和校准参数位宽不匹配。
- 未处理负数或符号扩展。

---

## 本章仍需视频核对

- [ ] 课程使用的 BMP280、SPL06 通信方式、模块地址或片选引脚、寄存器表、校准参数读取位置、补偿公式、库文件来源和显示格式。
- [ ] 课程 `[36]` 使用的 W25Q64 页大小、扇区大小、命令宏、片选 GPIO、SPI 模式和跨页写入封装。
- [ ] BMP280、SPL06 在课程中使用 I2C 还是 SPI；如果使用 SPI，CS 引脚、SPI 模式、读写位规则、寄存器地址处理方式都需要核对。
- [ ] BMP280 通信方式、I2C 地址或 SPI CS、寄存器地址、校准参数读取方式、补偿公式、库文件来源和显示格式。
- [ ] SPL06 通信方式、I2C 地址或 SPI CS、寄存器、校准参数、补偿公式、库来源和课程显示方式。

---
## 复习检查清单

- [ ] 能画出 SPI 的 SCK、MOSI、MISO、CS 连接关系。
- [ ] 能解释 CPOL / CPHA 为什么要按外设手册配置。
- [ ] 能说明读 SPI 数据时为什么还要发送 Dummy Byte。
- [ ] 能写出 W25Q64 读 ID 的基本流程。
- [ ] 能写出 W25Q64 写入前为什么要写使能和擦除。
- [ ] 能说明忙状态轮询的作用。
- [ ] 能按“供电 -> 共地 -> 接线 -> CubeMX -> HAL 返回值 -> 波形/串口 -> 协议 -> 业务逻辑”排查 SPI 问题。
- [ ] 能把 BMP280/SPL06 拆成读 ID、读校准参数、读原始值、换算、显示几个阶段。
