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
  - i2c
  - bus
---
# I2C 总线

> 能力定位：理解两线共享总线，并能从 ACK 和波形定位通信失败。
>
> 原始来源：[[11_I2C与OLED]]。

## 电气基础

SCL 和 SDA 通常使用开漏输出，由上拉电阻产生高电平。器件共享总线时只能主动拉低，避免输出冲突。

## 基本时序

```text
START -> 设备地址 + R/W -> ACK/NACK -> 数据字节 -> STOP
```

起始条件是 SCL 为高时 SDA 从高变低；停止条件是 SCL 为高时 SDA 从低变高。实际边沿和建立/保持时间由控制器产生，但排错时应能在逻辑分析仪上识别事务边界。

7 位设备地址不包含读写位。HAL 工程常见把 7 位地址左移一位传入 API，必须在驱动中统一，不要混用。

## HAL API 区别

| API | 用途 |
|---|---|
| `HAL_I2C_Master_Transmit` | 发送自定义字节序列 |
| `HAL_I2C_Master_Receive` | 接收自定义字节序列 |
| `HAL_I2C_Mem_Read` | 按设备地址 + 寄存器地址读取 |
| `HAL_I2C_Mem_Write` | 按设备地址 + 寄存器地址写入 |

## 调试抓手

- 地址扫描只能确认有设备 ACK，不能证明初始化正确。
- HAL 返回值要区分 OK、ERROR、BUSY、TIMEOUT。
- BUSY 卡死时检查 SDA/SCL 是否被拉低；必要时先用 GPIO 释放总线，再重新初始化 I2C。
- 用逻辑分析仪确认 START、地址、ACK 和数据方向。

## 关联知识

- [[OLED显示模块]]
- [[MPU6050姿态传感器]]
- [[传感器采集与显示]]

---

## 本节目标

- 能解释开漏、上拉、START/STOP、ACK/NACK
- 能区分设备地址和寄存器地址
- 能用 HAL API 扫描、读写寄存器并恢复 BUSY 总线

## 知识地图

| 名词 | 工程含义 |
|---|---|
| SCL/SDA | 时钟线/双向数据线 |
| 开漏 | 只主动拉低，由上拉产生高电平 |
| START/STOP | 总线事务边界 |
| ACK/NACK | 接收方应答/不应答 |
| 7 位地址 | 设备身份，不包含寄存器地址 |
| BUSY | 控制器或线路处于未结束事务状态 |

## 本质理解

I2C 的共享能力来自“任何设备都不主动输出高电平”。主机通过 START、地址和 ACK 确认对方，再交换寄存器地址与数据。地址扫描只能证明 ACK，不能证明寄存器和初始化正确。

为什么这样做：先建立硬件和数据流模型，再记 HAL API，才能在 API 变化或板卡变化时仍然定位问题。

## STM32F103 / HAL 中的实现方式

- F103 I2C GPIO 使用复用开漏并依赖外部上拉。
- HAL 常见接口要求把 7 位地址左移一位，项目中应统一宏的表达方式。
- `Mem_Read/Write` 适合“设备地址 + 寄存器地址”模型；Master API 适合自定义命令流。
- BUSY 卡死可能来自 SDA 被从机拉低，可先 GPIO 模拟时钟释放，再复位/重初始化控制器。

## CubeMX 配置要点

1. 选择 I2C 模式并确认 SCL/SDA 引脚。
2. 设置 Clock Speed 等参数，以模块手册和总线负载为准。
3. 确认外部上拉和电平兼容。
4. 需要中断/DMA时再启用对应配置，先用阻塞 API 验证。

## 常用 HAL API

### API 速查

| HAL 函数 / 宏 | 作用 | 常见位置 |
|---|---|---|
| `HAL_I2C_Master_Transmit()` | 向设备直接发送命令/数据序列 | OLED 命令、非寄存器协议 |
| `HAL_I2C_Master_Receive()` | 从设备直接接收字节流 | 简单连续读取协议 |
| `HAL_I2C_Mem_Write()` | 向寄存器型设备写配置 | BSP 初始化/配置 |
| `HAL_I2C_Mem_Read()` | 从指定寄存器开始读取 | 读 ID、状态、连续数据 |
| `HAL_I2C_IsDeviceReady()` | 通过 ACK 探测设备是否在线 | bring-up、地址扫描 |
| `HAL_I2C_GetState()` | 查询 HAL I2C 状态机 | BUSY/重启诊断 |
| `HAL_I2C_GetError()` | 读取最近 I2C 错误码 | API 失败后记录 |

### 使用注意

| HAL 函数 / 宏 | 前置条件 | 参数重点 | 返回值 / 阻塞与常见坑 |
|---|---|---|---|
| `HAL_I2C_Master_Transmit()` | I2C、上拉和设备地址正确 | `DevAddress`、buffer、Size、Timeout | 返回 `HAL_StatusTypeDef`；阻塞；把寄存器地址误当设备地址；地址未按 HAL 约定处理 |
| `HAL_I2C_Master_Receive()` | 设备已处于可读状态 | 地址、buffer、长度、超时 | 返回状态；阻塞；省略设备要求的前置命令；读取长度不符合协议 |
| `HAL_I2C_Mem_Write()` | 寄存器地址宽度已确认 | 设备地址、寄存器地址、`MemAddSize`、数据 | 返回状态；阻塞；`I2C_MEMADD_SIZE_8BIT/16BIT` 选错；寄存器和值顺序混淆 |
| `HAL_I2C_Mem_Read()` | 设备支持寄存器寻址 | 地址、寄存器、地址宽度、长度 | 返回状态；阻塞；多字节数据高低字节拼错；寄存器自动递增规则未核对 |
| `HAL_I2C_IsDeviceReady()` | 总线上拉和电平正常 | 地址、Trials、Timeout | 返回状态；阻塞尝试；只证明地址 ACK，不证明寄存器和初始化正确 |
| `HAL_I2C_GetState()` | 句柄已初始化 | I2C 句柄 | 返回 `HAL_I2C_StateTypeDef`；非阻塞；只看软件状态，不量 SDA/SCL 电平 |
| `HAL_I2C_GetError()` | 已发生或怀疑通信错误 | I2C 句柄 | 返回错误位掩码；非阻塞；忽略 API 返回值，事后错误上下文已丢失 |

### 典型调用顺序

```text
确认供电、共地、SCL/SDA 上拉
-> MX_I2Cx_Init()
-> HAL_I2C_IsDeviceReady() 验证地址 ACK
-> HAL_I2C_Mem_Read() 读取设备 ID
-> HAL_I2C_Mem_Write() 写最少配置
-> 连续 HAL_I2C_Mem_Read() 读取原始数据
-> 每步检查 HAL_OK，失败时记录 GetState/GetError
```

### 什么时候用哪个函数？

| 场景 | 推荐函数 |
|---|---|
| 探测某地址是否应答 | `HAL_I2C_IsDeviceReady()` |
| 读写常见传感器寄存器 | `HAL_I2C_Mem_Read()` / `HAL_I2C_Mem_Write()` |
| 发送 OLED 控制字节和数据流 | `HAL_I2C_Master_Transmit()` 或经验证的驱动封装 |
| 设备协议先写命令再独立读取 | `Master_Transmit()` + `Master_Receive()` |
| API 返回 BUSY/ERROR | 先查 SDA/SCL/上拉，再分别看 `HAL_I2C_GetState()`、`HAL_I2C_GetError()` |

## 最小实验 / 最小框架

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
for (uint16_t addr = 1; addr < 128; ++addr)
{
    if (HAL_I2C_IsDeviceReady(&hi2c1, addr << 1, 2, 10) == HAL_OK)
    {
        FoundDevice(addr);
    }
}
```

## 调试方法

```text
确认供电/共地
-> 量 SCL/SDA 空闲是否为高
-> 确认上拉和接线
-> 统一 7 位/左移地址
-> 检查 HAL 返回值
-> 逻辑分析仪看 START/地址/ACK
-> 再查寄存器和业务
```

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| 线路一直低 | 无上拉/短路/从机卡住 | 检查硬件并恢复总线 |
| 设备无 ACK | 地址或供电错误 | 扫描并核对 7 位地址 |
| 读错寄存器 | 设备地址与寄存器地址混淆 | 分开定义两个概念 |
| 地址差一倍 | 左移规则不统一 | 驱动层统一地址宏 |
| SCL/SDA 接反 | 接线错误 | 按模块丝印和原理图确认 |
| 一直 BUSY | 事务未结束/从机拉低 | GPIO 释放 + 重初始化 |

## 与其它章节的关系

- [[OLED显示模块]] 使用命令/数据写入。
- [[传感器采集与显示]] 使用寄存器读写。
- [[通用调试方法]] 说明逻辑分析仪观察。

## 复习检查清单

- [ ] 能否不用看代码说清本章数据流和硬件链路？
- [ ] 能否在 CubeMX 中完成最小配置并说明每个关键选项？
- [ ] 能否写出初始化、启动和回调/轮询的最小框架？
- [ ] 能否用原始电平、波形、返回值或寄存器证明外设在工作？
- [ ] 能否按“硬件 -> 时钟/配置 -> Start -> 回调 -> 业务”排查故障？
- [ ] 能否指出本章哪些参数必须按原理图、手册或实测确认？
