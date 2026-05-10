# 五、ADC 采样篇

> 覆盖课程：[21-1] [21-2] [21-3] [21-4] [21-5]
>
> 目标：掌握 ADC 单通道/多通道采集，配合 DMA 高效采样，能完成各类传感器电压换算。

---

## 前置知识

### ADC 是什么

```
模拟信号（连续变化的电压 0~3.3V）→ ADC → 数字值（0~4095）

F103C8T6 ADC：12 位精度
  2^12 = 4096 个刻度（0~4095）
  分辨率 = 3.3V / 4096 ≈ 0.8mV（1 个 LSB）
```

### 电压换算公式 ⭐

```
V = ADC值 / 4095 × 3.3

例：读到 2048
V = 2048 / 4095 × 3.3 ≈ 1.65V
```

### ADC 校准

F103 系列 ADC 使用前需要校准一次：

```c
HAL_ADCEx_Calibration_Start(&hadc1);  // 启动前校准
```

---

## [21-1] ADC 单通道采集电位器应用（轮询）

### 知识点
- ADC 轮询方式读取
- 电位器分压原理

### 关键原理

电位器 = 可调电阻分压器。拧动旋钮改变中间抽头的电压，ADC 读到 0~3.3V 范围的连续变化值。

### CubeMX 配置步骤

1. Pinout → 点目标引脚（如 PA0）→ 选 ADC1_IN0
2. Analog → ADC1 → Mode: IN0 Single-ended
3. 参数：Continuous Conversion Mode = Enable（持续采集），采样时间 = 239.5 Cycles

### HAL 库代码模板

```c
// 轮询方式（阻塞——每次采集等转换完成）
HAL_ADC_Start(&hadc1);
HAL_ADC_PollForConversion(&hadc1, 100);
uint32_t adcValue = HAL_ADC_GetValue(&hadc1);
HAL_ADC_Stop(&hadc1);

float voltage = (float)adcValue / 4095.0f * 3.3f;
printf("电压: %.2fV\r\n", voltage);
```

### 易错点
- 轮询方式只在偶尔读一次时用。需要连续高速采集 → 用 DMA 方式（[21-4]）
- 忘了 `HAL_ADCEx_Calibration_Start` → 读数偏差大

---

## [21-2] ADC 多通道采集热敏、光敏、反射传感器（轮询）

### 知识点
- 多通道扫描模式
- 通道 Rank 顺序

### 关键原理

多个传感器共用一个 ADC，按 Rank 顺序轮流采集。**Rank 1 = adcBuf[0], Rank 2 = adcBuf[1]……顺序不能错。**

### CubeMX 配置步骤

1. ADC1 → 启用多个 Channel（IN0, IN1, IN2...）
2. Scan Conversion Mode = **Enable**
3. Number of Conversion = 通道数
4. 在 Regular Conversion 里排好 Rank 顺序

### 易错点
- 🔴 **Rank 顺序和 buf 索引的对应是头号坑**：adcBuf[0] 永远对应 Rank 1
- 多通道轮询方式不配合 DMA 则只能拿到最后一个通道的数据

---

## [21-3] ADC 多通道采集雨量、土壤湿度传感器（中断+注入）

### 知识点
- ADC 中断模式
- 注入组和规则组的区别

### 关键原理

| 组 | 特点 | 适用 |
|----|------|------|
| 规则组（Regular） | 常规通道，支持扫描+DMA | 大多数场景 |
| 注入组（Injected） | 可插队，高优先级触发 | 紧急采样 |

> 注入组像 VIP 通道——哪怕规则组正在扫描，注入组触发后立刻插队采集，采完再还给规则组。

### HAL 库代码模板

```c
// 规则组中断回调
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    if (hadc->Instance == ADC1)
    {
        // 数据在 adcBuf 中
    }
}
```

### 易错点
- 注入组用得少，大部分场景规则组 + DMA 就够了

---

## [21-4] ADC 多通道采集空气、烟雾传感器（DMA 传输）

### 知识点
- ADC + DMA 连续采样
- DMA 循环模式

### 关键原理

DMA 自动把每次转换结果搬到数组里，一轮结束触发回调。CPU 完全解放。

### CubeMX 配置步骤

1. ADC1 → 启用多个 Channel
2. Continuous Conversion Mode = Enable
3. Scan Conversion Mode = Enable
4. DMA Settings → Add → ADC1 → Mode: Circular
5. DMA Continuous Requests = Enable

### HAL 库代码模板

```c
// 全局变量
uint32_t adcBuf[3];  // 3 个通道 = 数组大小 3

// main() 初始化中
HAL_ADCEx_Calibration_Start(&hadc1);     // 先校准
HAL_ADC_Start_DMA(&hadc1, adcBuf, 3);   // 启动 DMA 采集

// DMA 转换完成回调
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    if (hadc->Instance == ADC1)
    {
        // adcBuf[0] = Rank 1 通道
        // adcBuf[1] = Rank 2 通道
        // adcBuf[2] = Rank 3 通道
    }
}
```

### 易错点
- forgot Continuous Conversion = Enable → ADC 采一轮就停
- DMA Continuous Requests 没开 → DMA 搬一轮就停
- 数组大小不等于通道数 → 数据错位

---

## [21-5] 空气、烟雾传感器公式换算

### 知识点
- 各类传感器的 ADC 值转物理量
- MQ 系列烟雾传感器的特殊性

### 关键原理

#### 热敏传感器（NTC）

```c
float voltage = (float)adcBuf[0] / 4095.0f * 3.3f;
// ⚠️ 以下为教学简化公式。真实 NTC 是非线性的，需查 B 值用 Steinhart-Hart 方程
float temp = (voltage - 0.5f) / 0.01f;
```

#### 光敏传感器

```c
float voltage = (float)adcBuf[1] / 4095.0f * 3.3f;
// 需要自己标定：分别记录遮光、室内、强光下的 ADC 值，设定阈值
```

#### 烟雾传感器（MQ 系列）

```c
// MQ 传感器上电后需预热 2 分钟以上！内部加热丝需达到工作温度
float voltage = (float)adcBuf[2] / 4095.0f * 3.3f;
// 换算浓度：查数据手册的 Rs/R0 曲线
```

#### 采样时间选择

| 场合 | 采样时间 | 理由 |
|------|---------|------|
| 传感器 | 239.5 Cycles | 慢但准（≈17μs @14MHz ADC 时钟） |
| 高速信号 | 1.5 / 7.5 Cycles | 快但精度略低 |

### 易错点
- MQ 传感器不预热就读 → 前 2 分钟数据严重不准
- NTC 简化公式不是通用的 → 不同传感器参数不同
- 传感器数据抖动大 → 增大采样时间、加电容滤波、或软件多次取平均
- ADC 值一直为 0 → 检查是否忘了校准、引脚是否正确、参考电压是否正常

---

*课程：[21-1] [21-2] [21-3] [21-4] [21-5]*
