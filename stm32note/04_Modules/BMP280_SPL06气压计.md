---
type: module
status: refactor
course: stm32-f103-hal
source:
  - local-notes
  - module-datasheet
tags:
  - stm32
  - bmp280
  - spl06
  - barometer
---
# BMP280 / SPL06 气压计

> 模块定位：用“ID、校准参数、原始值、补偿”理解气压计驱动。
>
> 原始来源：[[13_SPI与存储器传感器]]。
---
## 本节目标

- 能完成 ID、校准参数、原始温压数据的分层读取。
- 能说明补偿公式必须与具体器件和手册版本一致。
- 能区分 I2C/SPI 通信问题与补偿计算问题。

## 核心概念

### 知识地图

| 名词 | 工程含义 |
|---|---|
| 气压 | 环境绝对压力 |
| 原始温度/气压 | ADC 输出，不能直接作为物理量 |
| 校准参数 | 芯片出厂修正系数 |
| 补偿公式 | 把原始值和系数换成工程单位 |
| 过采样/滤波 | 噪声与响应速度的取舍 |

### 模块作用

读取气压计温度/气压原始值和校准系数，并按对应芯片手册完成补偿。

### 通用 bring-up

```text
确认 I2C 或 SPI
-> 读设备 ID
-> 读校准参数
-> 配置采样
-> 读原始温度/气压
-> 按对应芯片公式补偿
-> 显示结果
```

BMP280 与 SPL06 的寄存器、校准参数位宽和补偿公式不能互相套用。先验证 ID、校准区和原始值，再引入公式。

> 视频待核对：课程使用的通信方式、地址/CS、寄存器、校准参数、补偿公式、库来源和显示单位。

不写未经确认的具体补偿公式。

## 本质理解

总线读通只能证明 STM32 能与气压计交换字节，不能证明温度和气压已经正确。原始值必须与当前芯片的出厂校准参数及补偿公式配套，BMP280 和 SPL06 的寄存器布局、系数与算法不能混用。

## 硬件连接关系

- 模块可能支持 I2C 或 SPI，接口由焊盘/引脚决定
- I2C 地址或 SPI CS 按模块确认
- 供电、电平和上拉/片选正确
- 不要同时错误启用两种接口

## 通信 / 控制链路

```text
确认总线方式
-> 读 ID
-> 读完整校准区
-> 配置采样
-> 读 raw temperature/pressure
-> 符号扩展
-> 按对应手册补偿
-> 与环境参考比较
```

## CubeMX 配置要点

1. 按模块选择 I2C 或 SPI
2. SPI 模式/速率按芯片手册
3. 串口打印 ID、校准参数和 raw
4. 先不启用复杂滤波

## 本模块依赖的 HAL 层

| 模块动作 | 依赖 HAL | 说明 |
|---|---|---|
| I2C 地址探测 | `HAL_I2C_IsDeviceReady()` | 适用于确认模块当前接口和地址是否 ACK |
| I2C 读 ID、校准参数和原始值 | `HAL_I2C_Mem_Read()` | 寄存器地址、长度和字节序分别按 BMP280/SPL06 手册确认 |
| I2C 写模式/过采样配置 | `HAL_I2C_Mem_Write()` | 两款芯片寄存器定义不可混用 |
| SPI 选择器件 | `HAL_GPIO_WritePin()` | 手动控制 CS；默认电平和引脚以原理图为准 |
| SPI 命令和数据交换 | `HAL_SPI_Transmit()` / `HAL_SPI_TransmitReceive()` | 读写位、地址格式和 dummy 要按具体芯片协议 |
| 测量等待/超时 | `HAL_GetTick()` | 用状态机等待，不用无限阻塞 |

HAL 只负责 I2C/SPI 字节传输。校准参数解释、定点/浮点补偿公式、单位和溢出处理属于具体气压计数据手册，BMP280 与 SPL06 不能共用一套未经核对的公式。

### 典型调用顺序

```text
确认模块采用 I2C 还是 SPI
-> 初始化对应 HAL 外设
-> 读取并核对设备 ID
-> 读取全部校准参数并保存
-> 写最少测量配置
-> 读取原始温度/气压
-> 按对应手册补偿
-> 输出 raw、中间量和最终单位
```

### 什么时候用哪个函数？

| 场景 | 推荐函数 |
|---|---|
| I2C 寄存器型访问 | `HAL_I2C_Mem_Read()`、`HAL_I2C_Mem_Write()` |
| SPI 发命令/地址 | 手动 CS + `HAL_SPI_Transmit()` |
| SPI 边发边收 | `HAL_SPI_TransmitReceive()` |
| 等待数据就绪 | 状态寄存器 + `HAL_GetTick()` 超时 |

## Bring-up 顺序

1. 确认模块实际使用 I2C 还是 SPI，并检查上拉或 CS 默认电平。
2. 读取设备 ID，先排除接口模式、地址或片选错误。
3. 原样打印校准区字节，核对长度、字节序和符号。
4. 连续读取原始温度、气压值，确认它们会随环境变化。
5. 使用对应器件手册的补偿入口计算工程量，不混用两款芯片公式。
6. 最后统一单位并接入串口或显示层。

## 最小驱动框架

> 代码性质：示例框架，用于理解调用顺序，不能保证直接编译。

```c
static uint8_t Barometer_ReadId(uint8_t *id)
{
    return Barometer_BusRead(BAROMETER_REG_ID, id, 1U);
}

static uint8_t Barometer_ReadCalibration(Barometer_Calibration_t *cal)
{
    uint8_t raw[BAROMETER_CALIBRATION_BYTES];
    if ((cal == NULL) ||
        !Barometer_BusRead(BAROMETER_REG_CALIBRATION_START,
                           raw, sizeof(raw)))
    {
        return 0U;
    }
    return Barometer_DecodeCalibration(raw, sizeof(raw), cal);
}

static uint8_t Barometer_ReadRaw(Barometer_Raw_t *raw)
{
    uint8_t data[BAROMETER_RAW_BYTES];
    if ((raw == NULL) ||
        !Barometer_BusRead(BAROMETER_REG_RAW_START, data, sizeof(data)))
    {
        return 0U;
    }
    return Barometer_DecodeRaw(data, sizeof(data), raw);
}

static uint8_t Barometer_CompensateFromManual(
    const Barometer_Raw_t *raw,
    const Barometer_Calibration_t *cal,
    Barometer_Data_t *out);

uint8_t Barometer_Read(Barometer_Data_t *out)
{
    Barometer_Raw_t raw;
    if ((out == NULL) || !Barometer_ReadRaw(&raw)) return 0U;
    return Barometer_CompensateFromManual(&raw, &barometer_calibration, out);
}

uint8_t id;
if (!Barometer_ReadId(&id) ||
    !Barometer_IdMatchesSelectedChip(id) ||
    !Barometer_ReadCalibration(&barometer_calibration))
{
    Debug_ReportBarometerInitFailure(id);
}
```

`Barometer_BusRead()` 由 I2C 或 SPI BSP 实现。BMP280 与 SPL06 的 ID、寄存器、校准布局、字节序和补偿函数不同，所有宏及 `Barometer_CompensateFromManual()` 必须逐款依据官方手册实现。

## 调试方法

```text
确认供电与接口模式
-> I2C ACK 或 SPI CS/模式
-> 读取并核对设备 ID
-> 打印校准区原始字节
-> 打印原始温度/气压
-> 检查补偿返回值和单位
-> 最后检查显示结果
```

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| ID 失败 | 接口模式/地址/CS 错 | 先确认 I2C 或 SPI |
| 校准全 0/FF | 读区/字节序错误 | 打印原始校准字节 |
| 数值离谱 | 公式或系数位宽错误 | 逐步验证中间量 |
| BMP/SPL 混用 | 寄存器/公式不同 | 分别维护驱动 |
| 高度不准 | 气压基准和天气变化 | 高度仅作估算并校准 |

## 待核对项

- [ ] BMP280/SPL06 的接口、地址/CS 和 ID
- [ ] 校准参数布局、位宽和符号扩展
- [ ] 补偿公式、单位和课程驱动来源

以实际开发板原理图、CubeMX 配置、视频课程源码和模块手册为准。

## 与其它章节的关系

- STM32 侧依赖：[[I2C总线]]、[[SPI总线与Flash]]、[[传感器采集与显示]]。
- 通用 bring-up 和数据显示见 [[传感器采集与显示]]。
- 跨外设排错见 [[通用调试方法]]。

## 复习检查清单

- [ ] 能先确认模块实际使用 I2C 还是 SPI，并按接口检查上拉或 CS。
- [ ] 能读取并验证设备 ID，不混用 BMP280 与 SPL06 的寄存器定义。
- [ ] 能分别保存校准参数、原始温度和原始气压数据。
- [ ] 能按手册处理校准参数的位宽、符号和字节序。
- [ ] 能说明原始 ADC 值不能直接作为温度或气压。
- [ ] 能把总线故障、原始数据故障和补偿公式故障分层定位。
- [ ] 能指出补偿公式、单位和课程驱动来源仍需手册或视频核对。
