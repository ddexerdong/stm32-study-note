# 05 ADC 模数转换

> 覆盖课程：[21-1] [21-2] [21-3] [21-4] [21-5]
>
> 目标：理解 ADC 采样本质，掌握单通道轮询、多通道扫描、DMA 连续采样和传感器电压换算。

---

## 本节目标

完成本节后，应能做到：

- 理解 ADC 把模拟电压转换成数字值的过程。
- 使用单通道轮询读取传感器电压。
- 使用多通道扫描和 DMA 连续采集。
- 掌握 ADC 原始值、电压值和传感器物理量之间的换算关系。

---

## 核心概念

### 知识地图

| 名词 | 工程含义 | 常见位置 |
|------|----------|----------|
| ADC | Analog-to-Digital Converter，把连续电压转换成离散数字值 | [21-1] |
| 分辨率 | ADC 能分出的等级数量，F103 常用 12 位，即 0~4095 | [21-1] |
| VREF | ADC 参考电压，决定可转换电压上限 | [21-1] |
| LSB | 最小量化单位，约等于 `VREF / 4096` | [21-1] |
| 采样时间 | ADC 前端采样电容连接输入信号的时间 | [21-1] |
| SAR ADC | 逐次逼近型 ADC，F103 使用这种结构 | [21-1] |
| 规则组 | 常规 ADC 采样序列，最多 16 个通道，最常用 | [21-3] |
| 注入组 | 高优先级采样序列，可打断规则组，最多 4 个通道 | [21-3] |
| 扫描模式 | 多通道按 Rank 顺序依次转换 | [21-2] |
| Rank | 多通道扫描时的通道顺序编号 | [21-2] |
| DMA | ADC 转换完成后，硬件自动把结果搬到数组 | [21-4] |
| 校准 | ADC 上电后的内部偏差补偿，F103 建议每次上电做一次 | [21-1] |
| 内部温度传感器 | 芯片内部温度传感器，连接到 ADC1_IN16 | [21-5] |

### ADC HAL API

| 函数 | 模式 | 作用 |
|------|------|------|
| `HAL_ADCEx_Calibration_Start(&hadc)` | 初始化 | ADC 校准，F103 常用 |
| `HAL_ADC_Start(&hadc)` | 轮询/触发 | 启动 ADC |
| `HAL_ADC_Stop(&hadc)` | 通用 | 停止 ADC |
| `HAL_ADC_PollForConversion(&hadc, Timeout)` | 轮询 | 阻塞等待转换完成 |
| `HAL_ADC_GetValue(&hadc)` | 通用 | 读取转换结果 |
| `HAL_ADC_Start_IT(&hadc)` | 中断 | 转换完成后进回调 |
| `HAL_ADC_Start_DMA(&hadc, pData, Length)` | DMA | 启动 ADC + DMA |
| `HAL_ADC_Stop_DMA(&hadc)` | DMA | 停止 ADC DMA |
| `HAL_ADC_ConvCpltCallback(hadc)` | 中断/DMA | 转换完成回调 |
| `HAL_ADC_ConvHalfCpltCallback(hadc)` | DMA | DMA 半满回调 |

---

## 本质理解

ADC 的本质是：把输入电压映射到一个整数。

```text
0 V      -> 0
VREF     -> 4095
中间电压 -> 0~4095 中的某个整数
```

以 3.3 V 参考电压和 12 位 ADC 为例：

```text
数字范围：0~4095
等级数量：4096
1 LSB ≈ 3.3 V / 4096 ≈ 0.806 mV
```

工程中常用换算：

```c
float voltage = (float)adcValue / 4095.0f * 3.3f;
```

严格理论上量化步进常写 `/4096`，但工程显示满量程时通常希望 `4095 -> 3.3 V`，所以本笔记统一用 `/4095`。

---

## 工作流程 / 原理

### 逐次逼近 ADC

STM32F103 使用 SAR（Successive Approximation Register）ADC。

```text
1. 采样阶段：
   输入电压给 ADC 内部采样电容充电。

2. 转换阶段：
   ADC 用二分法从最高位开始猜测电压。
   12 位 ADC 需要约 12.5 个 ADC 时钟周期完成转换。

3. 完成阶段：
   EOC 标志置位，结果写入 DR 数据寄存器。
```

采样时间越长，输入采样电容越容易充到真实电压，读数越稳；但单次转换越慢。传感器源阻抗较大时，优先选择较长采样时间。

F103 常见采样时间：

```text
1.5 / 7.5 / 13.5 / 28.5 / 41.5 / 55.5 / 71.5 / 239.5 ADC cycles
```

传感器采样入门建议先用 `239.5 Cycles`，稳定优先。

### ADC 时钟限制

STM32F103 的 ADC 时钟不能超过 14 MHz。常见配置：

```text
PCLK2 = 72 MHz
ADC Prescaler = /6
ADC_CLK = 12 MHz
```

一次完整转换时间：

```text
T = (采样时间 + 12.5) / ADC_CLK
```

例：

```text
采样时间 = 239.5 cycles
ADC_CLK = 12 MHz

T = (239.5 + 12.5) / 12 MHz
  = 252 / 12 MHz
  = 21 us

最大理论采样率 ≈ 47.6 kHz
```

### 为什么需要校准

ADC 内部比较器、采样电容阵列会有制造偏差。校准会测量并补偿这些偏差，减少固定偏移。

F103 上电后建议调用一次：

```c
HAL_ADCEx_Calibration_Start(&hadc1);
```

### 单通道轮询流程

```text
校准 ADC
-> 启动 ADC
-> 等待转换完成
-> 读取 DR
-> 换算电压
-> 停止或继续下一次
```

适合偶尔读取一次，例如电池电压、旋钮位置、慢速传感器。

### 多通道扫描

多通道扫描时，ADC 按 Rank 顺序转换：

```text
Rank 1 -> Rank 2 -> Rank 3 -> ...
```

但 ADC 只有一个规则组数据寄存器 `DR`。如果不用 DMA，前面通道的结果会被后面通道覆盖，轮询方式很容易只拿到最后一个通道。

因此：**多通道采集优先用 DMA。**

### ADC + DMA 标准方案

```text
ADC 扫描 Rank 1
-> DMA 把 DR 搬到 adcBuf[0]
ADC 扫描 Rank 2
-> DMA 把 DR 搬到 adcBuf[1]
ADC 扫描 Rank 3
-> DMA 把 DR 搬到 adcBuf[2]
-> 一轮完成
```

对应关系：

```text
adcBuf[0] = Rank 1
adcBuf[1] = Rank 2
adcBuf[2] = Rank 3
```

### 规则组和注入组

| 对比项 | 规则组 Regular | 注入组 Injected |
|--------|----------------|-----------------|
| 通道数 | 最多 16 个 | 最多 4 个 |
| DMA | 支持 | 不支持 |
| 优先级 | 常规 | 高，可插队 |
| 数据寄存器 | 共用一个 DR | 有独立寄存器 |
| 典型场景 | 常规采样、多通道 DMA | 过流、过压等紧急采样 |

入门阶段主要使用规则组。

---

## CubeMX 配置

### 单通道轮询

1. `Pinout` 点击目标引脚，例如 `PA0`。
2. 选择 `ADC1_IN0`。
3. `Analog -> ADC1` 中确认通道已加入 Regular Conversion。
4. 建议配置：
   - `Continuous Conversion Mode`：`Enable` 或按需关闭。
   - `Scan Conversion Mode`：`Disable`。
   - `Sampling Time`：传感器建议 `239.5 Cycles`。
5. `RCC` / ADC prescaler 确保 ADC 时钟不超过 14 MHz。

### 多通道 DMA

1. ADC1 启用多个通道。
2. `Scan Conversion Mode` = `Enable`。
3. `Continuous Conversion Mode` = `Enable`。
4. `Number of Conversion` = 通道数。
5. Regular Conversion 中按顺序配置 `Rank`。
6. `DMA Settings` 添加 `ADC1`。
7. DMA 参数：
   - `Mode`：`Circular`
   - `Data Width`：通常 Half Word 或 Word，按 CubeMX/HAL 生成配置保持一致
   - `Memory Increment`：`Enable`
8. `DMA Continuous Requests` = `Enable`。

关键组合：

```text
Scan Conversion Mode = Enable
Continuous Conversion Mode = Enable
DMA Continuous Requests = Enable
DMA Mode = Circular
```

---

## HAL 函数 / API

### `HAL_ADC_Start`

函数：

```c
HAL_StatusTypeDef HAL_ADC_Start(ADC_HandleTypeDef *hadc);
```

作用：

- 调用后，ADC 开始工作。
- 软件触发模式下会开始转换；外部触发模式下会进入等待触发状态。
- 不调用时，ADC 不会产生有效转换结果。
- 适合单通道轮询、外部触发采样和启动 DMA 前的基础理解。

### `HAL_ADC_PollForConversion`

函数：

```c
HAL_StatusTypeDef HAL_ADC_PollForConversion(ADC_HandleTypeDef *hadc,
                                            uint32_t Timeout);
```

作用：

- 调用后，CPU 阻塞等待 EOC 转换完成标志。
- 不调用时，可能在 ADC 尚未转换完成时读取旧值。
- 适合简单轮询，不适合高频采集。

### `HAL_ADC_GetValue`

函数：

```c
uint32_t HAL_ADC_GetValue(ADC_HandleTypeDef *hadc);
```

作用：

- 调用后，读取当前 ADC 转换结果。
- 不调用时，程序无法获得 ADC 数字值。
- F103 12 位 ADC 返回值通常是 `0~4095`。
- 适合电压换算、阈值判断和传感器数据读取。

### `HAL_ADC_Start_DMA`

函数：

```c
HAL_StatusTypeDef HAL_ADC_Start_DMA(ADC_HandleTypeDef *hadc,
                                    uint32_t *pData,
                                    uint32_t Length);
```

作用：

- 调用后，ADC 转换结果由 DMA 自动搬到缓冲区。
- 不调用时，DMA 不会更新 ADC 数组。
- 适合多通道连续采集和高频采样。

参数说明：

- `hadc`：ADC 句柄，例如 `&hadc1`。
- `pData`：目标缓冲区地址。
- `Length`：DMA 搬运次数，通常等于通道数。

说明：F1 HAL 的 API 形参是 `uint32_t *pData`，但实际 ADC 结果只有 12 位。CubeMX 常把 ADC DMA 的外设/内存数据宽度配置为 Half Word，此时工程里更常用 `uint16_t` 缓冲区，并在调用时做一次指针转换。

---

## 示例代码

### 单通道轮询采集

```c
HAL_ADCEx_Calibration_Start(&hadc1);

HAL_ADC_Start(&hadc1);

if (HAL_ADC_PollForConversion(&hadc1, 100) == HAL_OK)
{
    uint32_t adcValue = HAL_ADC_GetValue(&hadc1);
    float voltage = (float)adcValue / 4095.0f * 3.3f;

    printf("ADC=%lu, Voltage=%.2f V\r\n", adcValue, voltage);
}

HAL_ADC_Stop(&hadc1);
```

### 多通道 DMA 采集

```c
/* USER CODE BEGIN PV */
uint16_t adcBuf[3];
/* USER CODE END PV */

/* USER CODE BEGIN 2 */
HAL_ADCEx_Calibration_Start(&hadc1);
HAL_ADC_Start_DMA(&hadc1, (uint32_t *)adcBuf, 3);
/* USER CODE END 2 */

/* USER CODE BEGIN 4 */
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    if (hadc->Instance == ADC1)
    {
        // adcBuf[0] = Rank 1
        // adcBuf[1] = Rank 2
        // adcBuf[2] = Rank 3
    }
}
/* USER CODE END 4 */
```

### 主循环中读取 DMA 最新值

```c
while (1)
{
    float v0 = (float)adcBuf[0] / 4095.0f * 3.3f;
    float v1 = (float)adcBuf[1] / 4095.0f * 3.3f;
    float v2 = (float)adcBuf[2] / 4095.0f * 3.3f;

    printf("CH0=%.2f V, CH1=%.2f V, CH2=%.2f V\r\n", v0, v1, v2);
    HAL_Delay(500);
}
```

### NTC 简化换算

```c
float voltage = (float)adcBuf[0] / 4095.0f * 3.3f;

// 教学简化公式，仅适合演示，不是通用 NTC 精确算法
float temperature = (voltage - 0.5f) / 0.01f;
```

精确 NTC 换算需要查传感器 B 值、分压电阻，并使用 NTC 方程或查表。

### 光敏传感器阈值判断

```c
float lightVoltage = (float)adcBuf[1] / 4095.0f * 3.3f;

if (lightVoltage > 2.0f)
{
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);
}
else
{
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET);
}
```

光敏电阻模块差异很大，阈值应通过实测标定。

### MQ 系列传感器读取

```c
// MQ 系列需要预热，上电后等待 2 分钟以上再相信读数
float mqVoltage = (float)adcBuf[2] / 4095.0f * 3.3f;
printf("MQ voltage = %.2f V\r\n", mqVoltage);
```

浓度换算需查对应型号数据手册的 `Rs/R0` 曲线，不能直接套一个通用线性公式。

---

## 代码执行流程

```text
CubeMX 配置 ADC 通道、采样时间和 DMA
↓
main() 初始化 GPIO、ADC、DMA
↓
F103 上电后先执行 ADC 校准
↓
轮询模式：HAL_ADC_Start()
↓
HAL_ADC_PollForConversion() 等待 EOC
↓
HAL_ADC_GetValue() 读取原始值
↓
DMA 模式：HAL_ADC_Start_DMA() 启动连续搬运
↓
主循环读取缓冲区最新 ADC 值并换算电压/物理量
```

---

## 常见错误

| 问题 | 常见原因 | 解决方法 |
|------|----------|----------|
| ADC 值一直为 0 | 引脚没配 ADC、没启动 ADC、外部电压为 0 | 查 CubeMX 和接线 |
| 读数有固定偏移 | 忘记校准 | 上电后调用 `HAL_ADCEx_Calibration_Start()` |
| 多通道数据错位 | Rank 和数组索引没对应 | 明确 `adcBuf[0]=Rank1` |
| 多通道只读到最后一个通道 | 轮询读 DR 被覆盖 | 多通道用 DMA |
| DMA 搬一轮就停 | Continuous 或 DMA Continuous Requests 未开 | 检查 CubeMX 关键项 |
| 数据抖动大 | 采样时间短、电源噪声、输入阻抗高 | 增大采样时间、加滤波、做平均 |
| 电压超过 3.3 V | 外部输入超出 ADC 范围 | 分压后再接 ADC |
| MQ 上电前几分钟不准 | 传感器未预热 | 上电等待 2 分钟以上 |
| `/4095` 和 `/4096` 纠结 | 工程显示和理论量化口径不同 | 本笔记统一 `/4095` |

---

## 调试方法

1. 先用万用表量 ADC 引脚真实电压。
2. 确认引脚没有接超过 3.3 V 的电压。
3. CubeMX 检查该引脚是否为 `ADCx_INx`，不是普通 GPIO。
4. 单通道轮询先跑通，再上多通道 DMA。
5. 打印原始 ADC 值，不要一上来只看换算后的物理量。
6. 多通道时改变某一个通道输入，确认对应 `adcBuf[i]` 变化。
7. 抖动大时先增大采样时间，再考虑硬件滤波和软件平均。

通用排查链路：

```text
真实电压
-> 引脚映射
-> ADC 时钟
-> ADC 校准
-> 启动方式
-> Rank 顺序
-> DMA 配置
-> 换算公式
```

---

## 工程意义

ADC 是单片机进入真实物理世界的重要入口。它把电压、电位器、温度、光照、压力、气体浓度等模拟信号变成程序能处理的数字。

工程上真正重要的是：

- 知道 ADC 读到的只是电压。
- 知道电压和物理量之间需要传感器模型或标定。
- 知道输入范围不能超过参考电压。
- 知道采样时间、源阻抗、噪声会影响稳定性。
- 知道多通道持续采样应交给 DMA。

---

## 易混淆点

| 容易混淆 | 正确理解 |
|----------|----------|
| 12 位和 4095 | 12 位有 4096 个等级，数值范围是 0~4095 |
| ADC 值和电压 | ADC 值只是数字，需按 VREF 换算成电压 |
| 电压和物理量 | 电压到温度/浓度/光照需要传感器模型 |
| 采样时间和采样周期 | 采样时间只是转换的一部分，总时间还包括 12.5 周期 |
| 规则组和注入组 | 规则组是常规采样，注入组是高优先级插队 |
| Scan 和 Continuous | Scan 是多通道顺序，Continuous 是一轮后是否继续 |
| DMA Circular 和 Continuous Requests | 一个管 DMA 是否循环，一个管 ADC 是否持续请求 DMA |
| `/4095` 和 `/4096` | 工程显示常用 `/4095`，理论量化可见 `/4096` |

---

## 我的理解

ADC 不是“直接测温度/测光照”，它只是在测电压。

我现在的理解是：

- 传感器先把物理量变成电压。
- ADC 再把电压变成数字。
- 程序最后把数字换算成我想看的物理量。
- 如果物理量不准，不能只怪代码，硬件、电源、采样时间、传感器标定都有影响。
- 多通道采样时，Rank 顺序就是数组顺序，不能乱。

以后做 ADC，我会先按这个顺序验证：

```text
万用表量电压
-> 单通道读原始值
-> 换算电压
-> 多通道确认 Rank
-> DMA 连续采样
-> 滤波
-> 物理量标定
```

---

*课程：[21-1] [21-2] [21-3] [21-4] [21-5]*

---

## 复习检查清单

- [ ] 能说明 ADC 只负责把电压转换成数字值，不直接理解物理量。
- [ ] 能写出 12 位 ADC 原始值到电压的大致换算关系。
- [ ] 能区分单通道、多通道、规则组、注入组和 Rank。
- [ ] 能说明轮询、中断和 DMA 采集方式的区别。
- [ ] 能解释多通道 DMA 数组下标和 Rank 顺序的对应关系。
- [ ] 能说明采样时间、源阻抗、电源噪声对 ADC 结果的影响。
- [ ] 能解释传感器公式换算前为什么要先验证原始 ADC 值和电压。
- [ ] 能按万用表电压、CubeMX 通道、启动函数、DMA、换算公式排查 ADC 问题。
