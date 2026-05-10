# [24] TIM 触发 ADC 采集

> 当前播放课程
>
> 目标：用定时器硬件触发 ADC 采样，实现精确定时采集，CPU 完全解放。

---

## 知识点
- TIM 触发 ADC 的硬件连接（内部信号，不需要外部接线）
- 定时器 TRGO 输出
- ADC 外部触发源选择
- 精确定时采样 vs 软件触发的区别

---

## 关键原理

### 为什么要用 TIM 触发 ADC？

软件触发的问题：

```
while(1)
{
    HAL_ADC_Start...   ← 采集间隔不准（while 循环时间不固定）
    ...
    HAL_Delay(10)      ← 阻塞，且定时精度差
}
```

TIM 硬件触发的优势：

```
TIM 定时器 → TRGO 信号 → ADC 触发输入（内部直连）
  ↓                           ↓
精确等间隔                   自动开始转换
  ↓                           ↓
每 1ms（完全精确）→          一个采样 → DMA 搬到内存

CPU 全程不参与触发过程！
```

### 硬件信号链路

```
TIMx 计数器溢出（Update Event）
    ↓ TRGO（Trigger Output）
ADCx 外部触发源选择 TIMx_TRGO
    ↓
ADC 自动开始一次/一轮转换
    ↓
DMA 把结果搬到内存
    ↓
采集完预设次数 → 中断通知 CPU
```

### CubeMX 里的设置关系

| TIM 侧 | ADC 侧 |
|--------|--------|
| TIMx → Trigger Output (TRGO) → Update Event | ADCx → External Trigger Conversion Source → Timer x Trigger Out |
| TIMx 的 PSC/ARR 决定采样间隔 | ADC 的采样时间决定每次转换耗时 |

### 采样频率计算

```
采样频率 = TIMx_CLK / [(PSC+1) × (ARR+1)]

例：TIM2_CLK = 72MHz，想每 1ms 采样一次
PSC = 71, ARR = 999
Fs = 72M / (72 × 1000) = 1kHz = 每秒 1000 次 ✓
```

---

## CubeMX 配置步骤

### TIM 侧

1. Timers → TIM2 → Clock Source: Internal Clock
2. PSC = 71, ARR = 999（1kHz 触发，即 1ms 一次）
3. **Trigger Output (TRGO) → Update Event**（这是关键！）
4. NVIC → 不需要开（ADC DMA 收完才进中断，TIM 本身不进中断）

### ADC 侧

1. ADC1 → 启用需要的 Channel
2. **External Trigger Conversion Source → Timer 2 Trigger Out**
3. **External Trigger Conversion Edge → Rising edge**
4. Continuous Conversion Mode = **Disable**（由 TIM 触发，不需要连续转换）
5. Scan Conversion Mode = Enable（多通道时）
6. DMA Settings → Add → ADC1 → Mode: Circular
7. DMA Continuous Requests = Enable

### NVIC

- ADC1 global interrupt → Enable（DMA 搬完一轮进回调）
- TIM2 global interrupt → **Disable**（不需要 TIM 中断）

---

## HAL 库代码模板

```c
// 全局变量
#define ADC_BUF_SIZE  1000   // 缓存大小（根据需求）
#define ADC_CH_NUM    3      // 通道数（根据实际配置）
uint32_t adcBuf[ADC_BUF_SIZE * ADC_CH_NUM];  // 数据缓存
volatile uint8_t adcComplete = 0;            // 采集完成标志

// main() 初始化
HAL_ADCEx_Calibration_Start(&hadc1);                         // 先校准
HAL_ADC_Start_DMA(&hadc1, adcBuf, ADC_BUF_SIZE * ADC_CH_NUM); // 启动 ADC+DMA

// 等 ADC 准备好，再启动 TIM（顺序不能反！）
HAL_TIM_Base_Start(&htim2);  // 注意：不要用 _IT，TIM 不需要中断

// ADC DMA 采集完成回调
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    if (hadc->Instance == ADC1)
    {
        adcComplete = 1;  // 通知 main 循环数据准备好了
    }
}

// while(1) 中处理数据
while (1)
{
    if (adcComplete)
    {
        adcComplete = 0;

        // 数据处理：adcBuf[0]=第一轮 CH1, adcBuf[1]=第一轮 CH2...
        // adcBuf[3]=第二轮 CH1, adcBuf[4]=第二轮 CH2...
        for (int i = 0; i < ADC_BUF_SIZE; i++)
        {
            float voltage = (float)adcBuf[i * ADC_CH_NUM] / 4095.0f * 3.3f;
            // ...
        }
    }
}
```

### 多通道时的数据排布

```
假设 3 通道，DMA 循环模式：

adcBuf[0] = 第一轮 Rank1 的值
adcBuf[1] = 第一轮 Rank2 的值
adcBuf[2] = 第一轮 Rank3 的值

adcBuf[3] = 第二轮 Rank1 的值
adcBuf[4] = 第二轮 Rank2 的值
adcBuf[5] = 第二轮 Rank3 的值
...
```

---

## 易错点

| 坑 | 原因 | 解决 |
|----|------|------|
| ADC 不触发 | 没配 TRGO | TIM → Trigger Output 选 Update Event |
| ADC 不触发 | External Trigger 没选对 | ADC → Ext Trigger → Timer 2 Trigger Out |
| ADC 不触发 | TIM 时钟没开 | 检查 CubeMX 时钟配置 |
| ADC 触发了一次就停 | Continuous Conversion 误开了 | TIM 触发模式下关掉 Continuous Conversion |
| 采样频率不对 | TIM 时钟算错 | 确认 TIMx_CLK（APB1 ×2 规则！） |
| ADC 数据错位 | 多通道时 buf 索引算错 | 数据按 Rank 顺序交织排列，仔细算偏移 |
| 启动顺序反了 | 先启动了 TIM | **必须先启动 ADC+DMA，再启动 TIM** |

---

*课程：[24]*
