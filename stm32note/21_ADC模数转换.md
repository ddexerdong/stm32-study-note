# 21 ADC 模数转换

> ADC 把模拟世界的电压变成数字世界的数值。温度、光照、声音、电压……凡是连续变化的物理量，都要经过 ADC 才能被 STM32 处理。

---

## ADC 基础

```
模拟信号（电压 0~3.3V）→ ADC → 数字值（0~4095）

F103C8T6 的 ADC 精度：12 位
  12 位 = 2^12 = 4096 个刻度（编号 0~4095）
  分辨率 = 3.3V / 4096 ≈ 0.8mV（每个刻度代表约 0.8mV）
```

### 电压换算公式 ⭐

```
V = ADC值 / 4095 × VREF
  = ADC值 / 4095 × 3.3

反过来：
ADC值 = V / 3.3 × 4095

例：ADC 读到 2048
V = 2048 / 4095 × 3.3 ≈ 1.65V
```

---

## HAL ADC 轮询方式（最简单入门）

```c
// 轮询方式（阻塞——等待转换完成才返回）
HAL_ADC_Start(&hadc1);                              // 启动 ADC
HAL_ADC_PollForConversion(&hadc1, 100);             // 等待转换完成，超时 100ms
uint32_t adcValue = HAL_ADC_GetValue(&hadc1);       // 读取转换结果
HAL_ADC_Stop(&hadc1);                               // 停止 ADC

// 转为电压
float voltage = (float)adcValue / 4095.0f * 3.3f;
printf("电压: %.2fV\r\n", voltage);
```

> 轮询方式适合偶尔读一次的场景（比如按键触发采集）。需要连续高速采集时用 DMA 方式。

---

## 单通道 vs 多通道

| | 单通道 | 多通道扫描 |
|--|--------|----------|
| CubeMX 配置 | 一个 Channel | 多个 Channel，开启 Scan Conversion Mode |
| 采集方式 | 单次转换 | 自动轮流采集所有通道 |
| 数据读取 | 直接 `GetValue` | 必须配合 DMA，否则只拿得到最后一个通道 |
| 课程 | [21-1] | [21-2] [21-3] [21-4] |

---

## 多通道 + DMA（[21-2]）

### 🔴 通道顺序是头号坑

**`adcBuf[0]` 对应 CubeMX 里 Rank 1 的通道，`adcBuf[1]` 对应 Rank 2……顺序不能搞错。**

在 CubeMX → ADC1 → Regular Conversion 里设置 Rank 顺序，Rank 1 = 第一个采集的通道。

### 代码

```c
// 假设 3 个通道：PA0(Rank1) 热敏，PA1(Rank2) 光敏，PA2(Rank3) 烟雾
uint32_t adcBuf[3];  // 数组大小 = 通道数

// main() 初始化中启动 DMA 转换
HAL_ADC_Start_DMA(&hadc1, adcBuf, 3);

// DMA 转换完成回调（一轮扫描结束触发一次）
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    if (hadc->Instance == ADC1)
    {
        // adcBuf[0] = Rank 1 通道数据（热敏）
        // adcBuf[1] = Rank 2 通道数据（光敏）
        // adcBuf[2] = Rank 3 通道数据（烟雾）
    }
}
```

---

## 各传感器换算公式

### 热敏传感器（NTC）

```c
// ⚠️ 以下为教学用的简化线性换算。真实 NTC 是非线性的。
// 精确应用需查传感器的 B 值（数据手册），用 Steinhart-Hart 方程或查表法。
float voltage = (float)adcBuf[0] / 4095.0f * 3.3f;
float temp = (voltage - 0.5f) / 0.01f;  // 根据你的传感器特性调整系数
```

### 光敏传感器

```c
// 光照越强，光敏电阻越小，ADC 电压越低（常见接法）
float voltage = (float)adcBuf[1] / 4095.0f * 3.3f;
// 需要自己标定：记录遮光/室内/强光下的 ADC 值，设定阈值
```

### 烟雾传感器（MQ 系列）

```c
// MQ 传感器内部有加热丝，上电后需要预热 2 分钟以上让敏感材料达到工作温度
// 否则前几分钟读数严重不准
float voltage = (float)adcBuf[2] / 4095.0f * 3.3f;
// 换算为浓度：查数据手册的 Rs/R0 曲线，通常需要标定
```

---

## CubeMX 配置要点

| 参数 | 设置 | 说明 |
|------|------|------|
| Continuous Conversion Mode | **Enable** | 持续采集，不用手动触发 |
| Scan Conversion Mode | **Enable** | 多通道时开，自动扫描所有 Rank |
| DMA Continuous Requests | **Enable** | DMA 不间断搬运数据 |
| Number of Conversion | 通道数量 | 有几个 Rank 填几 |

### 采样时间

| 场合 | 推荐值 | 理由 |
|------|--------|------|
| 传感器采样 | 239.5 Cycles | 慢但准（约 17μs/次 @14MHz ADC 时钟） |
| 高速信号 | 1.5 / 7.5 Cycles | 快但精度略低 |

---

## 踩坑记录

- [ ] 多通道数据顺序错位 → 检查 CubeMX 的 Rank 顺序和 `adcBuf` 索引是否一致
- [ ] ADC 值一直为 0 → 没调 `HAL_ADCEx_Calibration_Start(&hadc1)`（F103 需要校准）
- [ ] ADC 值抖动大 → 采样时间太短、电源噪声大、模拟信号本身有干扰
- [ ] 多通道 DMA 不更新 → Continuous Conversion 或 DMA Continuous Requests 没打开
- [ ] 传感器读数莫名其妙 → 先单独测试 ADC 通道（接固定的 3.3V/GND 看读数是否正常）

---

*课程：[21-1] [21-2] [21-3] [21-4] [21-5]*
