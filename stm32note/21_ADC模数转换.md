# 21 ADC 模数转换

## ADC 基础

```
模拟信号（电压）→ ADC → 数字值（0~4095）

精度：12位 → 2^12 = 4096个刻度
范围：0V ~ 3.3V（参考电压VREF）
```

### 电压换算公式 ⭐

```
V = ADC值 / 4095 × VREF
  = ADC值 / 4095 × 3.3

反过来：
ADC值 = V / 3.3 × 4095
```

## HAL ADC 函数

```c
// 轮询方式（阻塞）
HAL_ADC_Start(&hadc1);
HAL_ADC_PollForConversion(&hadc1, 100);  // 等待转换完成，超时100ms
uint32_t adcValue = HAL_ADC_GetValue(&hadc1);
HAL_ADC_Stop(&hadc1);

// 转为电压
float voltage = (float)adcValue / 4095.0f * 3.3f;
printf("电压: %.2fV\r\n", voltage);
```

## 单通道 vs 多通道

| | 单通道 | 多通道扫描 |
|--|--------|----------|
| 配置 | 一个Channel | 多个Channel，开启Scan Mode |
| 采集 | 单次转换 | 自动扫描所有通道 |
| 数据读取 | 直接GetValue | 配合DMA |
| 课程 | [21-1] | [21-2][21-3][21-4] |

## 多通道 + DMA（[21-2]）

```c
// 变量定义（通道数量对应）
uint32_t adcBuf[3];  // 3个通道

// main函数中启动
HAL_ADC_Start_DMA(&hadc1, adcBuf, 3);

// DMA转换完成回调
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc) {
    if (hadc->Instance == ADC1) {
        // adcBuf[0] = 通道1数据
        // adcBuf[1] = 通道2数据
        // adcBuf[2] = 通道3数据
    }
}
```

## 各传感器换算公式

### 热敏传感器（NTC）

```c
// 简化线性换算（实际是非线性，精确需查数据手册）
float voltage = (float)adcBuf[0] / 4095.0f * 3.3f;
float temp = (voltage - 0.5f) / 0.01f;  // 根据传感器特性调整
```

### 光敏传感器

```c
// 光照越强，电阻越小，电压越低
float voltage = (float)adcBuf[1] / 4095.0f * 3.3f;
// 自己标定：记录不同光照下的ADC值
```

### 烟雾传感器（MQ系列）

```c
// 需要预热（上电后等2分钟）
float voltage = (float)adcBuf[2] / 4095.0f * 3.3f;
// 换算为浓度：查数据手册的Rs/R0曲线
```

## CubeMX 配置要点

- Continuous Conversion Mode: Enable（持续采集）
- Scan Conversion Mode: Enable（多通道时开）
- DMA Continuous Requests: Enable（配合DMA）
- 每个通道设置采样时间：传感器一般用 239.5 Cycles（慢但准）

## 踩坑记录

- [ ] 多通道顺序 → 严格按CubeMX里的Rank顺序对应
- [ ] ADC需要校准 → 启动前加 `HAL_ADCEx_Calibration_Start(&hadc1)`
- [ ] 

---
*课程：[21-1] [21-2] [21-3] [21-4] [21-5]*
