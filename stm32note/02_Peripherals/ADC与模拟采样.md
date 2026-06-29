---
type: peripheral
status: refactor
course: stm32-f103-hal
source:
  - local-notes
  - st-reference-manual
  - st-hal-documentation
tags:
  - stm32
  - adc
  - analog
---
# ADC 与模拟采样

> 能力定位：从模拟电压可靠地得到可解释、可校准的数字数据。
>
> 原始来源：[[05_ADC模数转换]]、[[07_TIM触发ADC采集]]。
---
## 本节目标

- 能完成单通道轮询、多通道扫描和 ADC DMA
- 能解释 VREF、采样时间、源阻抗和滤波
- 能判断原始值、电压和物理量的可信度

## 核心概念

### 知识地图

| 名词 | 工程含义 |
|---|---|
| SAR ADC | 逐次逼近模数转换器 |
| LSB | 一个数字量级对应的电压 |
| 采样时间 | 采样电容连接输入的周期数 |
| 规则组/注入组 | 常规序列/高优先级序列 |
| Rank | 规则组转换顺序 |
| 模拟前端 | ADC 前的分压、缓冲和滤波电路 |

### 基础概念

| 名词 | 工程含义 |
|---|---|
| 分辨率 | 12 位 ADC 常输出 0 到 4095 |
| VREF | 决定数字值与电压的换算基准 |
| 采样时间 | 输入给采样电容充电的时间 |
| 校准 | 减小 ADC 内部偏差 |
| Rank | 多通道规则组的转换顺序 |
| 规则组 / 注入组 | 常规序列 / 可插队的高优先级序列 |

```text
voltage = raw / 4095.0 * Vref
```

### 采集方式

- 单通道轮询：最适合最小验证。
- 多通道扫描：按 Rank 依次转换。
- 中断：转换完成后回调处理。
- DMA：连续搬运到缓冲区。
- TIM 触发：获得稳定采样节拍。

### 模拟前端思维

- 输入阻抗高时，适当增加采样时间。
- VREF 和供电噪声会直接影响结果。
- 输入电压不得超过 ADC 允许范围，必要时分压或限幅。
- 滑动平均用于降低随机噪声；限幅滤波用于拒绝突发异常。
- RC 滤波可抑制高频噪声，但会改变响应速度。

不写死 RC、分压和滤波参数，应根据传感器输出、电气限制和采样频率设计。

## 本质理解

ADC 只测电压，不认识温度、湿度或浓度。可信的数据链必须依次验证模拟输入、参考电压、采样过程、数字原始值和传感器模型；跳过中间层直接套公式，结果通常不可解释。

## STM32F103 / HAL 中的实现方式

- F103 常用 12 位 ADC，原始范围 0–4095；电压换算依赖实际 VREF。
- 高源阻抗信号需要更长采样时间，让内部采样电容充分充电。
- 规则组按 Rank 扫描，DMA 数组下标跟随转换顺序；注入组适合高优先级采样。
- 上电后按 HAL 流程进行 ADC 校准。

## CubeMX 配置要点

1. ADC 引脚设 Analog。
2. 设置 ADC 时钟分频和采样时间。
3. 单通道轮询关闭 Scan/DMA。
4. 多通道配置每个 Channel 的 Rank。
5. 连续采样配置 DMA Circular 和合适的数据宽度。

## 常用 HAL API

### API 速查

| HAL 函数 / 宏 | 作用 | 常见位置 |
|---|---|---|
| `HAL_ADCEx_Calibration_Start()` | 执行 STM32F1 ADC 校准 | 初始化后、首次采样前 |
| `HAL_ADC_Start()` | 启动轮询或外部触发转换流程 | 采样前 |
| `HAL_ADC_PollForConversion()` | 阻塞等待转换完成 | 单通道最小实验 |
| `HAL_ADC_GetValue()` | 读取 ADC 数据寄存器结果 | Poll 成功后 |
| `HAL_ADC_Start_IT()` | 启动 ADC 转换完成中断 | 低频事件采样 |
| `HAL_ADC_Start_DMA()` | 启动 ADC 并把结果搬入数组 | 连续/多通道采样 |
| `HAL_ADC_Stop_DMA()` | 停止 ADC DMA 采集 | 模式切换、停机 |
| `HAL_ADC_ConvHalfCpltCallback()` | 通知 DMA 前半区可处理 | 双半区/流式处理 |
| `HAL_ADC_ConvCpltCallback()` | 通知整缓冲区或后半区完成 | DMA/IT 完成事件 |

### 使用注意

| HAL 函数 / 宏 | 前置条件 | 参数重点 | 返回值 / 阻塞与常见坑 |
|---|---|---|---|
| `HAL_ADCEx_Calibration_Start()` | ADC 初始化完成且空闲 | ADC 句柄 | 返回 `HAL_StatusTypeDef`；同步等待；忘记检查返回值；采样过程中调用 |
| `HAL_ADC_Start()` | 通道、Rank、触发方式已配置 | ADC 句柄 | 返回状态；启动非阻塞；外部触发模式下调用后仍需等待触发事件 |
| `HAL_ADC_PollForConversion()` | 已 `HAL_ADC_Start()` | ADC 句柄、超时 | 返回状态；阻塞；超时无限大导致程序卡住；多通道扫描时误解每次 Poll 的含义 |
| `HAL_ADC_GetValue()` | 转换完成 | ADC 句柄 | 返回 `uint32_t`；非阻塞；未完成就读旧值；直接把 raw 当物理量 |
| `HAL_ADC_Start_IT()` | ADC NVIC 已启用 | ADC 句柄 | 返回状态；非阻塞；扫描多通道时中断粒度需结合 F1 HAL 行为确认 |
| `HAL_ADC_Start_DMA()` | DMA 已配置绑定；buffer 长期有效 | 句柄、`uint32_t *pData`、元素数 | 返回状态；非阻塞；DMA 长度与 Rank 数不一致；buffer 类型/对齐错误 |
| `HAL_ADC_Stop_DMA()` | ADC DMA 已运行 | ADC 句柄 | 返回状态；同步停止；停止后直接使用正在更新的旧状态，未清业务标志 |
| `HAL_ADC_ConvHalfCpltCallback()` | Circular DMA 与中断已启用 | ADC 句柄 | `void`；中断上下文；前半区尚在处理时被下一轮覆盖 |
| `HAL_ADC_ConvCpltCallback()` | 中断链完整 | ADC 句柄 | `void`；中断上下文；回调中滤波/打印过重；多个 ADC 不判断实例 |

### 典型调用顺序

```text
轮询：MX_ADCx_Init()
-> HAL_ADCEx_Calibration_Start()
-> HAL_ADC_Start()
-> HAL_ADC_PollForConversion()
-> HAL_ADC_GetValue()

DMA：MX_DMA_Init()
-> MX_ADCx_Init()
-> 定义全局/静态 DMA buffer
-> HAL_ADCEx_Calibration_Start()
-> HAL_ADC_Start_DMA()
-> Half/Cplt 回调置处理标志
```

### 什么时候用哪个函数？

| 场景 | 推荐函数 |
|---|---|
| 单次读取一个通道 | `Start` + `PollForConversion` + `GetValue` |
| 低频转换完成事件 | `HAL_ADC_Start_IT()` |
| 连续多通道采样 | `HAL_ADC_Start_DMA()` |
| 固定时间间隔采样 | TIM TRGO + ADC 外部触发 + DMA |
| 停止连续采样再改配置 | `HAL_ADC_Stop_DMA()` 后确认状态 |

## 最小实验 / 最小框架

> 代码性质：示例框架，用于理解调用顺序，不能保证直接编译。

```c
static uint32_t adc_dma_buffer[ADC_CHANNEL_COUNT];
static volatile uint8_t adc_dma_ready;

static uint8_t ADC_ReadOnce(uint32_t *raw)
{
    if ((raw == NULL) || (HAL_ADC_Start(&hadc1) != HAL_OK))
    {
        return 0U;
    }
    if (HAL_ADC_PollForConversion(&hadc1, ADC_POLL_TIMEOUT_MS) != HAL_OK)
    {
        return 0U;
    }
    *raw = HAL_ADC_GetValue(&hadc1);
    return 1U;
}

static float ADC_RawToVoltage(uint32_t raw, float vref)
{
    return ((float)raw * vref) / ADC_FULL_SCALE_COUNTS;
}

static uint32_t MovingAverage_Push(uint32_t sample)
{
    static uint32_t history[FILTER_WINDOW_SIZE];
    static uint32_t sum;
    static uint32_t index;

    sum -= history[index];
    history[index] = sample;
    sum += sample;
    index = (index + 1U) % FILTER_WINDOW_SIZE;
    return sum / FILTER_WINDOW_SIZE;
}

void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    if (hadc->Instance == ADC1)
    {
        adc_dma_ready = 1U;
    }
}

if (HAL_ADCEx_Calibration_Start(&hadc1) != HAL_OK)
{
    Error_Handler();
}

uint32_t single_raw;
if (ADC_ReadOnce(&single_raw))
{
    float voltage = ADC_RawToVoltage(single_raw, measured_vref);
    App_ReportVoltage(voltage);
}

if (HAL_ADC_Start_DMA(&hadc1, adc_dma_buffer, ADC_CHANNEL_COUNT) != HAL_OK)
{
    Error_Handler();
}

while (1)
{
    if (adc_dma_ready)
    {
        adc_dma_ready = 0U;
        uint32_t filtered = MovingAverage_Push(adc_dma_buffer[ADC_RANK1_INDEX]);
        App_ProcessAdcChannels(adc_dma_buffer, ADC_CHANNEL_COUNT, filtered);
    }
}
```

`adc_dma_buffer[0]` 对应 Rank 1，后续下标按 CubeMX Rank 顺序排列。`ADC_FULL_SCALE_COUNTS`、VREF、滤波窗口和通道数量以实际分辨率、电路与采样目标为准。

## 调试方法

```text
万用表量 AO 电压
-> 单通道读原始值
-> 核对 VREF 假设
-> 延长采样时间观察变化
-> 多通道逐个改变输入确认 Rank
-> 再启用 DMA 和滤波
-> 最后套物理量公式
```

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| 读数为 0 | 引脚非 Analog/未启动 | 检查 GPIO 和 Start |
| 固定偏差 | 未校准或 VREF 假设错 | 校准并测参考电压 |
| 多通道错位 | Rank 与数组不一致 | 建立明确映射表 |
| DMA 长度错 | 元素数与序列不符 | 按转换序列设置长度 |
| 读数漂移 | 源阻抗高/采样短/噪声 | 延长采样并改善前端 |
| 输入损伤 | 电压超范围 | 分压、限幅或运放缓冲 |

## 与其它章节的关系

- [[DMA与缓冲区]] 管理连续结果。
- [[TIM触发ADC]] 提供稳定节拍。
- [[传感器采集与显示]] 完成换算、滤波和展示。

## 复习检查清单

- [ ] 能说明 ADC 原始值表示输入电压相对 VREF 的量化结果。
- [ ] 能在 F103 启动采样前调用并检查 `HAL_ADCEx_Calibration_Start()`。
- [ ] 能说明 `HAL_ADC_PollForConversion()` 是带超时的阻塞等待。
- [ ] 能把多通道 Rank 顺序与 DMA 数组下标逐一对应。
- [ ] 能保证 DMA buffer 生命周期覆盖整个传输过程。
- [ ] 能根据源阻抗和信号稳定性选择采样时间并观察噪声。
- [ ] 能区分传感器 DO 阈值输出与 AO 模拟量，并在换算前检查原始值。
- [ ] 能说明滤波只能抑制波动，不能修复错误的 VREF、接线或通道配置。
