# 05 ADC 模数转换

> 覆盖课程：[21-1] [21-2] [21-3] [21-4] [21-5]
>
> 目标：掌握 ADC 单通道/多通道采集，配合 DMA 高效采样，完成传感器电压换算。

---

## 知识地图

| 新名词 | 解释 | 哪里出现 |
|--------|------|---------|
| ADC | Analog-to-Digital Converter，模数转换器。把连续的电压值转成离散的数字量 | [21-1] |
| 分辨率 | ADC 能区分的最小电压变化。12 位 ADC = 4096 个台阶，每台阶 3.3V/4096 ≈ 0.8mV | [21-1] |
| VREF | ADC 的参考电压。F103C8T6 的 VREF+ 接 3.3V（内部连 VDDA），决定了 ADC 最大可测电压 | [21-1] |
| LSB | Least Significant Bit，最低有效位。1 LSB = VREF / 2^N。F103 下 = 3.3V / 4096 ≈ 0.806mV | [21-1] |
| 采样时间 | ADC 对输入信号采样的时长（以 ADC 时钟周期为单位）。时间越长越准，但采样频率越低 | [21-1] |
| 逐次逼近型 ADC | F103 使用的 ADC 架构。通过二分法逐次逼近真实电压值，每次比较需要 1 个 ADC 时钟周期 | [21-1] |
| 扫描模式（Scan Mode） | 多通道时，按 Rank 顺序逐个采集所有通道 | [21-2] |
| 注入组（Injected Group） | 高优先级的 ADC 通道组，可插队采集。最多 4 个通道 | [21-3] |
| 规则组（Regular Group） | 常规 ADC 通道组，最多 16 个通道（F103）。支持 DMA | [21-3] |
| 采样保持电路 | ADC 前级电路：先用电容"冻结"输入电压，再逐次比较。采样时间就是给电容充电稳定用的 | [21-1] |
| 校准（Calibration） | ADC 内部比较器的自调零。F103 上电需校准一次 | [21-1] |
| 内嵌温度传感器 | 芯片内部温度传感，连到 ADC1_IN16。不需要外部接线 | [21-5] |

---

## ADC HAL 函数族总览

| 函数 | 模式 | 适用场景 |
|------|------|---------|
| `HAL_ADC_Start(&hadc)` | 轮询 | 偶尔读一次，用 PollForConversion 等结果 |
| `HAL_ADC_Stop(&hadc)` | — | 停止 ADC |
| `HAL_ADC_PollForConversion(&hadc, Timeout)` | 轮询 | 等待一次转换完成。阻塞 |
| `HAL_ADC_GetValue(&hadc)` | 通用 | 读 ADC 转换结果（返回 uint32_t, 0~4095） |
| `HAL_ADC_Start_IT(&hadc)` | 中断 | 每次转换完触发回调 |
| `HAL_ADC_Stop_IT(&hadc)` | — | 停止中断模式 |
| `HAL_ADC_Start_DMA(&hadc, pData, Length)` | DMA | 多通道或连续采集，效率最高 |
| `HAL_ADC_Stop_DMA(&hadc)` | — | 停止 DMA 模式 |
| `HAL_ADC_ConvCpltCallback(hadc)` | 中断/DMA | 转换完成回调（用户实现） |
| `HAL_ADC_ConvHalfCpltCallback(hadc)` | DMA | DMA 缓冲半满回调 |
| `HAL_ADCEx_Calibration_Start(&hadc)` | — | 校准 ADC（F103 必须） |

---

## ADC 工作原理

### 逐次逼近型 ADC 的转换过程

F103 使用的是 SAR（Successive Approximation Register，逐次逼近寄存器）型 ADC。转换过程：

```
1. 采样阶段：闭合采样开关，输入电压给内部电容充电。持续 N 个 ADC 时钟周期的采样时间
2. 转换阶段：断开采样开关，开始二分法逐次逼近
   - 12 位 ADC → 12 次比较 → 12 个 ADC 时钟周期
   - SAR 寄存器从最高位（第 11 位）开始：猜 1 → 与输入电压比较 → 高了就改 0 → 下一位
3. 转换完成：EOC 标志置位，结果存入 DR 寄存器
```

采样时间的作用：输入信号源有内阻，电容充电需要时间。信号源内阻越大，需要的采样时间越长。F103 支持 1.5 / 7.5 / 13.5 / 28.5 / 41.5 / 55.5 / 71.5 / 239.5 个周期。传感器一般用 239.5 周期（最保守）。

### 为什么 12 位 = 4096 而不是 4095？

```
12 位寄存器可以表示 2^12 = 4096 个不同的值
范围：0 ~ 4095（共 4096 个数）

0x000(0)    → 0V
0xFFF(4095) → 3.3V（理论最大）
每变化 1 个数 = 3.3V / 4096 ≈ 0.806mV

但电压换算公式用 4095 还是 4096？
实际使用 4095：因为满量程读数为 4095，此时对应 3.3V
V = ADC_Value / 4095 × 3.3  （通用近似）
理论严格值：
V = ADC_Value / 4096 × 3.3  （理论值）
差异极小（0.02%），工程上两种都用。本笔记统一用 /4095。
```

### ADC 时钟限制

F103 数据手册规定 ADC 时钟频率范围为 0.6~14MHz。CubeMX 默认配置将 PCLK2（72MHz）6 分频得到 12MHz ADC 时钟（= `RCC_ADCPCLK2_DIV6`）。

一次完整转换时间：
```
总时间 = 采样时间 + 12.5 个 ADC 时钟周期（转换时间）

例：采样时间 239.5 Cycles，ADC 时钟 12MHz
总时间 = (239.5 + 12.5) / 12MHz = 252 / 12MHz = 21μs
最大采样率 ≈ 1 / 21μs ≈ 47.6kHz
```

### 为什么需要校准？

ADC 内部有一个采样保持电容阵列，各电容的容值在制造时有微小差异。校准过程测量这些差异并算出补偿值存入内部校准寄存器。不校准的读数会有几到几十个 LSB 的固定偏移量。

校准函数只需要上电后调用一次：

```c
HAL_ADCEx_Calibration_Start(&hadc1);
```

> 校准的底层原理：内部把 ADC 输入短接到 0V 和 VREF，分别测出实际的零点和满量程对应的读数，据此计算补偿值。这个过程约需 2ms。

---

## [21-1] 单通道轮询采集

### `HAL_ADC_Start` + `HAL_ADC_PollForConversion` 的配合

```c
HAL_StatusTypeDef HAL_ADC_Start(ADC_HandleTypeDef *hadc);
```
- 功能：使能 ADC 模块，如果 Continuous Conversion Mode = Enable，ADC 开始连续转换；如果 Disable，ADC 等待 `Start` 后的第一次触发
- 参数：`hadc` 为 CubeMX 生成的 ADC 句柄（如 `&hadc1`）

```c
HAL_StatusTypeDef HAL_ADC_PollForConversion(ADC_HandleTypeDef *hadc, uint32_t Timeout);
```
- 功能：阻塞等待 EOC（End of Conversion）标志。等到了返回 HAL_OK，超时返回 HAL_TIMEOUT
- `Timeout` 单位为毫秒

```c
uint32_t HAL_ADC_GetValue(ADC_HandleTypeDef *hadc);
```
- 返回当前的 ADC 转换值（0~4095）。没有参数需要指定通道——转换结果寄存器 DR 只有 16 位，存的就是最后一次转换结果

### 轮询采集完整流程

```c
HAL_ADC_Start(&hadc1);                              // 启动 ADC

if (HAL_ADC_PollForConversion(&hadc1, 100) == HAL_OK) // 等转换完成，超时100ms
{
    uint32_t adc_val = HAL_ADC_GetValue(&hadc1);     // 拿结果

    float voltage = (float)adc_val / 4095.0f * 3.3f; // 换算电压
    printf("电压: %.2fV\r\n", voltage);
}

HAL_ADC_Stop(&hadc1);                               // 停止，省电
```

> 注意：如果 CubeMX 配了 Continuous Conversion Mode = Enable，`Start` 后 ADC 会一直转换，`PollForConversion` 等到的只是最近一次的结果。

### CubeMX 配置

1. Pinout → 点目标引脚（如 PA0）→ ADC1_IN0
2. Analog → ADC1 → IN0 → 勾选
3. Continuous Conversion Mode: Enable（持续采集，每次 PollForConversion 能拿到新值）
4. Sampling Time: 239.5 Cycles（传感器用最长时间，最稳）

### 易错点
- 忘了校准 → 读数有固定偏移
- 轮询方式 `PollForConversion` 阻塞 CPU → 只适合偶尔读一次
- 电压换算公式用错（/4095 还是 /4096 差异很小，但初学者容易纠结）

---

## [21-2] 多通道轮询采集

### 扫描模式

CubeMX 启用多个通道后，开 Scan Conversion Mode，ADC 每次启动会按 Rank 1→2→3 顺序扫描所有通道。但**轮询方式下最后一个通道的转换结果会覆盖前几个**：

```
原因：ADC 只有一个 DR（数据寄存器）。每次转换结束，新数据覆盖旧数据。
不使用 DMA 时 PollForConversion 只能读到最后一次转换结果。
```

所以多通道**轮询**实际价值有限——真要测多个通道，用 DMA 方式（[21-4]）。

### Rank 顺序

CubeMX → ADC1 → Regular Conversion 里排 Rank 1/2/3。Rank 1 的数据在 adcBuf[0]，Rank 2 在 adcBuf[1]……这是固定映射关系。

### CubeMX 配置要点

1. ADC1 → 启用多个 Channel（IN0、IN1、IN2...）
2. Scan Conversion Mode = **Enable**
3. Number of Conversion = 通道数
4. Regular Conversion 里给每个 Rank 指定 Channel 和 Sampling Time

---

## [21-3] 注入组 vs 规则组

| | 规则组 Regular | 注入组 Injected |
|--|---------------|----------------|
| 通道数 | 最多 16 个 | 最多 4 个 |
| DMA | ✅ | ❌ |
| 优先级 | 常规 | 高（可以插队） |
| 数据寄存器 | 共用 1 个 DR | 各有独立寄存器（4 个） |
| 典型场景 | 正常多通道采集 | 紧急事件采样（如过流检测） |

注入组的工作原理：规则组正在按顺序扫描时，注入组触发信号到来 → 暂停规则组 → 采集注入通道 → 恢复规则组。注入通道的结果存在各自的独立寄存器中，不占用规则组 DR。

> 注入组在入门阶段很少用到，了解即可。

---

## [21-4] 多通道 DMA 采集（标准方案）

### 原理

DMA 配合 ADC 扫描模式：每次转换完成后，DMA 自动将 DR 寄存器的值搬运到用户数组，然后地址指针自增。一轮扫描完毕后，DMA 触发完成中断 → HAL 调用 `HAL_ADC_ConvCpltCallback`。

### `HAL_ADC_Start_DMA` 详解

```c
HAL_StatusTypeDef HAL_ADC_Start_DMA(ADC_HandleTypeDef *hadc,
                                     uint32_t *pData,
                                     uint32_t Length);
```

- `pData`：目标数组，类型必须是 `uint32_t*`（因为 ADC 结果是 32 位寄存器，即使用 12 位精度）
- `Length`：DMA 搬运次数。多通道且 Continuous 模式下，一轮扫描的搬运次数 = 通道数。如果 `Length = 通道数 × N`，则搬 N 轮后才触发回调

### CubeMX 配置关键项

1. ADC1 → 多个 Channel
2. Continuous Conversion Mode = **Enable**
3. Scan Conversion Mode = **Enable**
4. Number of Conversion = 通道数，排好 Rank
5. DMA Settings → Add → ADC1 → Mode: **Circular**
6. DMA Continuous Requests = **Enable**

> Circular 模式 + DMA Continuous Requests = Enable：ADC 连续扫描 → DMA 自动循环搬运 → 新数据持续更新数组。这是最常用的标准配置。

### 代码

```c
uint32_t adcBuf[3];  // 3 个通道 → 数组长度 = 3

HAL_ADCEx_Calibration_Start(&hadc1);    // ① 校准
HAL_ADC_Start_DMA(&hadc1, adcBuf, 3);  // ② 启动 DMA（3 = 通道数）

void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    if (hadc->Instance == ADC1)
    {
        // 每完成一轮扫描触发一次
        // adcBuf[0] = Rank 1 的值  (如 IN0, 热敏传感器)
        // adcBuf[1] = Rank 2 的值  (如 IN1, 光敏传感器)
        // adcBuf[2] = Rank 3 的值  (如 IN2, 烟雾传感器)
    }
}
```

### 易错点
- Continuous Conversion Mode 没开 → ADC 扫一轮就停
- DMA Continuous Requests 没开 → DMA 搬一轮就停
- 数组大小不等于通道数 → 数据错位或溢出
- Rank 顺序和 buf 索引必须严格对应

---

## [21-5] 传感器换算公式

### 热敏传感器（NTC）

NTC（Negative Temperature Coefficient）热敏电阻的阻值随温度升高而下降。通过分压电路，将电阻变化转为电压变化，再用 ADC 读取。

典型分压结构：VCC → 固定电阻 R_fixed → ADC 引脚 ← NTC 热敏电阻 → GND。

```c
float voltage = (float)adcBuf[0] / 4095.0f * 3.3f;
// 下面是教学简化公式（线性近似），精度约 ± 5℃
float temp = (voltage - 0.5f) / 0.01f;
```

精确计算需要查传感器数据手册的 B 值，使用 Steinhart-Hart 方程（这里不展开，后续 [29] DHT11 等数字温湿度传感器普及后实际项目中很少用裸 NTC）。

### 光敏传感器

光敏电阻阻值随光照变化。不同型号差异巨大，没有通用公式：

```c
float voltage = (float)adcBuf[1] / 4095.0f * 3.3f;
// 自己标定：记录三种光照（遮光/室内/强光）下的 ADC 值，用于设定阈值
```

### MQ 系列烟雾/气体传感器

```c
// 需要预热！MQ 传感器内部加热丝需要达到工作温度
// 上电后等待 2 分钟以上再开始采集
float voltage = (float)adcBuf[2] / 4095.0f * 3.3f;
// 浓度换算需查传感器数据手册的 Rs/R0 曲线（通常是指数关系）
```

### 通用抗噪手段

| 方法 | 实现 | 效果 |
|------|------|------|
| 增大采样时间 | CubeMX → Sampling Time: 239.5 Cycles | 采样电容充电更充分 |
| 硬件滤波 | ADC 输入脚对地接 100nF 电容 | 滤除高频噪声 |
| 软件平均 | 连续采 N 次取平均 | 降低随机噪声 |
| 中值滤波 | 连续采 N 次取中间值 | 消除偶发脉冲干扰 |

### 易错点
- MQ 传感器不预热→前 2 分钟数据严重不准
- NTC 简化线性公式不是通用的（不同 NTC 参数不同）
- ADC 值一直为 0 → 检查校准、引脚映射、参考电压

---

*课程：[21-1] [21-2] [21-3] [21-4] [21-5]*
