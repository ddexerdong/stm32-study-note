---
type: module
status: refactor
course: stm32-f103-hal
source:
  - local-notes
  - module-datasheet
tags:
  - stm32
  - vl53l0x
  - tof
  - i2c
---
# VL53L0X 激光测距

> 模块定位：学习依赖厂商库的复杂测距模块 bring-up。
>
> 原始来源：[[12_I2C传感器]]。
---
## 本节目标

- 能说明 ToF 测距的数据链和厂商库边界。
- 能按初始化、启动、数据就绪、读取和状态判断排错。
- 能把总线故障、库适配故障和无效测距分开。

## 核心概念

### 知识地图

| 名词 | 工程含义 |
|---|---|
| ToF | 通过光飞行时间估计距离 |
| 初始化库 | 封装复杂寄存器序列和校准 |
| 测距模式 | 精度、速度和量程的取舍 |
| Data Ready | 新结果可读取的状态 |
| Range Status | 结果有效性/错误原因 |

### 模块作用

通过 ToF 模块和官方/课程库读取距离，学习复杂初始化、数据就绪和状态码处理。

### 学习主线

```text
I2C ACK
-> 库初始化
-> 配置测距模式
-> 启动单次/连续测距
-> 等待数据就绪或超时
-> 读取距离和状态
-> 处理异常结果
```

初始化失败、数据未就绪和无效距离必须区分。目标反射率、环境光和测量模式都会影响结果。

> 视频待核对：库来源、地址、初始化接口、测距模式、数据就绪、距离读取、状态码和连续测距周期。

不能凭空补完整寄存器序列。

## 本质理解

VL53L0X 测距不是读一个固定寄存器就结束，而是依赖初始化、启动、数据就绪、结果读取和状态判断组成的测量流程。距离数值必须与 Range Status 和超时结果一起判断，不能只看毫米值。

## 硬件连接关系

- 供电、共地、SCL/SDA
- XSHUT/中断脚是否使用按模块设计
- 多个模块可能需要通过 XSHUT 改地址
- 光学窗口避免遮挡

## 通信 / 控制链路

```text
确认 ACK
-> 绑定库底层读写
-> 调用初始化/校准
-> 选择测距模式
-> 启动测距
-> 等待 Data Ready + 超时
-> 读取距离和状态
-> 清中断/下一次
```

## CubeMX 配置要点

1. 启用 I2C
2. 可选中断脚配置 EXTI
3. 先串口打印库返回码
4. 不手写未知寄存器序列

## 本模块依赖的 HAL 层

| 模块动作 | 依赖 HAL | 说明 |
|---|---|---|
| 确认 I2C 地址 ACK | `HAL_I2C_IsDeviceReady()` | 只验证总线层；不能替代厂商 API 初始化 |
| 适配厂商库的写事务 | `HAL_I2C_Master_Transmit()` 或经验证的 `HAL_I2C_Mem_Write()` | VL53L0X 寄存器/事务格式由所用官方库适配层决定，不默认地址宽度 |
| 适配厂商库的读事务 | `HAL_I2C_Master_Transmit()` + `HAL_I2C_Master_Receive()` | 必须与厂商平台接口和寄存器寻址方式一致 |
| 经验证的寄存器式读取 | `HAL_I2C_Mem_Read()` | 仅在适配层明确匹配该寻址模型时使用 |
| 数据就绪/测距超时 | `HAL_GetTick()` | 毫秒级状态机超时，避免无限等待 |
| 控制 XSHUT | `HAL_GPIO_WritePin()` | 可选；有效电平、连接和上电顺序以模块原理图为准 |

HAL 只承担 I2C/GPIO 适配。初始化表、测距模式、数据就绪状态、距离有效性和错误码属于 ST 官方 VL53L0X API/模块驱动，不应凭 HAL 函数猜测。

### 典型调用顺序

```text
可选 XSHUT 复位/上电
-> MX_I2Cx_Init()
-> HAL_I2C_IsDeviceReady()
-> 调用已确认来源的 VL53L0X 初始化 API
-> 启动测距
-> HAL_GetTick() 保护数据就绪等待
-> 读取距离和状态码
```

### 什么时候用哪个函数？

| 场景 | 推荐函数 |
|---|---|
| 只确认 I2C 电气/地址 | `HAL_I2C_IsDeviceReady()` |
| 厂商库底层平台适配 | 严格按平台接口实现 I2C Tx/Rx，不自行猜寄存器序列 |
| 防止测距状态机卡死 | `HAL_GetTick()` 超时 |
| 多传感器地址管理 | 经原理图确认后用 XSHUT GPIO 逐个启动 |

## Bring-up 顺序

1. 确认供电、I2C 上拉、地址 ACK 和 XSHUT 状态。
2. 接通厂商库适配层，逐步记录初始化返回码。
3. 启动单次或连续测距，并设置数据就绪超时。
4. 读取距离与 Range Status，拒绝无效结果。
5. 固定目标和环境，观察结果稳定性。
6. 最后加入多模块地址管理、显示和业务周期。

## 最小驱动框架

> 代码性质：示例框架，用于理解调用顺序，不能保证直接编译。
> 验证状态：未上板验证，需按 CubeMX 变量名、开发板原理图和课程源码核对。

```c
static uint8_t VL53L0X_InitDevice(VL53L0X_Device_t *device)
{
    if (HAL_I2C_IsDeviceReady(&hi2c1, VL53L0X_ADDR_HAL,
                               VL53L0X_READY_TRIALS,
                               VL53L0X_I2C_TIMEOUT_MS) != HAL_OK)
    {
        return 0U;
    }
    return VL53L0X_Vendor_Init(device) == VL53L0X_VENDOR_OK;
}

static uint8_t VL53L0X_WaitDataReady(VL53L0X_Device_t *device,
                                      uint32_t timeout_ms)
{
    uint32_t start = HAL_GetTick();
    while ((uint32_t)(HAL_GetTick() - start) < timeout_ms)
    {
        uint8_t ready = 0U;
        if (VL53L0X_Vendor_CheckDataReady(device, &ready) != VL53L0X_VENDOR_OK)
            return 0U;
        if (ready) return 1U;
    }
    return 0U;
}

uint8_t VL53L0X_ReadDistance(uint16_t *mm)
{
    VL53L0X_RangeStatus_t range_status;
    if ((mm == NULL) || !VL53L0X_WaitDataReady(&vl53_device,
                                                VL53L0X_TIMEOUT_MS))
        return 0U;
    if (VL53L0X_Vendor_GetDistance(&vl53_device, mm, &range_status) !=
        VL53L0X_VENDOR_OK)
        return 0U;
    return VL53L0X_IsRangeValid(range_status);
}

if (!VL53L0X_InitDevice(&vl53_device) ||
    (VL53L0X_Vendor_StartMeasurement(&vl53_device) != VL53L0X_VENDOR_OK))
{
    Debug_ReportVl53InitFailure();
}
```

`VL53L0X_Vendor_*` 代表来源明确的厂商库接口。设备地址、初始化流程、测距模式、状态码和清中断步骤以实际官方库版本为准，不手写未知寄存器序列。

## 调试方法

```text
确认 I2C ACK 与 XSHUT 状态
-> 逐层打印厂商库初始化返回码
-> 确认测距已经启动
-> 检查数据就绪是否超时
-> 同时打印距离和 Range Status
-> 固定目标排查环境影响
-> 最后检查周期与多模块管理
```

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| 初始化失败 | 库移植/I2C 适配错误 | 逐层打印返回码 |
| 一直未就绪 | 测距未启动/状态清除错误 | 检查调用顺序 |
| 距离为 0/极大 | 状态码无效 | 不要忽略 Range Status |
| 偶发卡死 | 等待无超时 | 所有轮询加超时 |
| 结果漂移 | 目标反射率/环境光 | 固定目标做对照 |

## 待核对项

- [ ] 官方/课程库版本和接口
- [ ] 地址、XSHUT、初始化/校准流程
- [ ] 测距模式、周期、状态码和异常处理

以实际开发板原理图、CubeMX 配置、视频课程源码和模块手册为准。

## 与其它章节的关系

- STM32 侧依赖：[[I2C总线]]、[[传感器采集与显示]]、[[通用调试方法]]。
- 通用 bring-up 和数据显示见 [[传感器采集与显示]]。
- 跨外设排错见 [[通用调试方法]]。

## 复习检查清单

- [ ] 能先确认 I2C 在线，再进入厂商库初始化流程。
- [ ] 能按初始化、启动测量、等待就绪、读取距离和检查状态的顺序调用。
- [ ] 能为数据就绪等待设置超时，避免模块异常时卡死。
- [ ] 能把距离值与 Range Status 一起判断，不只看毫米数值。
- [ ] 能区分 I2C 故障、库适配故障和无效测距状态。
- [ ] 能说明多模块地址管理可能依赖 XSHUT，但具体流程不能猜测。
- [ ] 能指出库版本、初始化/校准、测距模式和状态码必须核对。
