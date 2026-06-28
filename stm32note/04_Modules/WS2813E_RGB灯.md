---
type: module
status: refactor
course: stm32-f103-hal
source:
  - local-notes
  - module-datasheet
tags:
  - stm32
  - ws2813e
  - rgb
  - tim
---
# WS2813E RGB 灯

> 模块定位：学习严格单线时序、颜色缓冲区和批量刷新。
>
> 原始来源：[[14_RGB灯与ESP8266]]。

## 数据路径

```text
业务颜色
-> 像素缓冲区
-> 按颜色顺序展开 bit
-> 编码成高低电平时序
-> GPIO/TIM/DMA 输出
-> reset/latch
```

## 工程重点

- 先验证单颗灯和固定颜色。
- 供电能力、共地和逻辑电平要先于软件排查。
- 时序输出可能受中断和编译优化影响。
- 不同器件的颜色顺序和时序窗口不能猜。

> 视频待核对：灯珠数量、颜色顺序、纳秒级时序、reset/latch 时间、数据引脚、驱动方式和供电方案。

## 关联知识

- [[TIM_PWM输出]]
- [[DMA与缓冲区]]
- [[联网控制与总线通信]]

---

## 本节目标

- 能解释颜色缓冲区、bit 时序和 reset/latch 的关系。
- 能比较 GPIO、TIM PWM 和 DMA 三种驱动路径。
- 能从供电、波形、颜色顺序和刷新逻辑逐层排错。

## 模块作用

把像素颜色编码成严格单线脉宽序列，并通过 GPIO、TIM 或 DMA 稳定刷新灯珠链。

## 知识地图

| 名词 | 工程含义 |
|---|---|
| 像素缓冲区 | 每颗灯的颜色状态 |
| 颜色顺序 | RGB/GRB 等字节排列 |
| bit 时序 | 高低脉宽编码 0/1 |
| reset/latch | 帧结束后的锁存间隔 |
| 级联 | 前级消费自身数据并转发后续数据 |

## 本质理解

模块驱动不是“抄一份初始化代码”，而是把硬件连接、总线事务、状态、原始数据和业务结果逐层验证。任何上层结果都必须建立在下层证据可靠的基础上。

## 硬件连接关系

- 灯珠电源电流按数量和亮度预算
- 控制器与灯带共地
- 数据方向必须从 DIN 输入
- 逻辑高电平兼容需查手册
- 长线考虑串阻/电平转换/供电注入

## 通信 / 控制链路

```text
建立颜色缓冲
-> 按颜色顺序展开 bit
-> 编码 0/1 脉宽
-> 输出一帧
-> 保持 reset/latch
-> 验证第一颗
-> 扩展灯珠数量
```

## CubeMX 配置要点

1. GPIO bit-bang：普通输出，易受中断影响
2. TIM PWM：配置固定 bit 周期和两个 CCR 值
3. TIM + DMA：把编码缓冲自动送入 CCR
4. 先单颗低亮度验证

## 本模块依赖的 HAL 层

| 驱动动作 | 依赖 HAL / 接口 | 说明 |
|---|---|---|
| GPIO bit-bang 原型 | `HAL_GPIO_WritePin()` | HAL 调用开销和中断抖动通常不适合未经验证的纳秒级量产时序，仅可用于概念/低风险试验 |
| 启动固定 PWM 通道 | `HAL_TIM_PWM_Start()` | 适合观察基本波形，不会自动发送整帧颜色编码 |
| DMA 推送 CCR 序列 | `HAL_TIM_PWM_Start_DMA()` | buffer 在发送完成前必须有效；每个元素代表一个 bit 的比较值 |
| 帧发送完成通知 | `HAL_TIM_PWM_PulseFinishedCallback()` | 仅在相应 PWM IT/DMA 流程下进入；回调中停止 DMA/拉低输出的细节需按驱动验证 |
| 生成位宽序列 | 普通 C 编码函数 | 不是 HAL API；颜色顺序、bit 时序和 reset/latch 必须来自器件手册 |

### 典型调用顺序

```text
按手册确认颜色顺序和高低电平窗口
-> CubeMX 配置 TIM PWM + DMA
-> 把颜色 buffer 编码成 CCR 序列
-> HAL_TIM_PWM_Start_DMA()
-> PulseFinishedCallback 标记帧完成/停止输出
-> 保持满足手册的 reset/latch 低电平时间
```

### 什么时候用哪个函数？

| 场景 | 推荐函数 |
|---|---|
| 观察单个 PWM bit 基础波形 | `HAL_TIM_PWM_Start()` |
| 稳定发送多灯颜色帧 | `HAL_TIM_PWM_Start_DMA()` + 经验证的编码 buffer |
| 纯 GPIO bit-bang | 仅在时序经过示波器验证时使用，不把普通 HAL GPIO 调用当确定方案 |

## Bring-up 顺序

1. 只验证供电、共地和静态电平。
2. 只验证 GPIO/总线和 HAL 返回值。
3. 读取 ID、DO、原始字节或原始 ADC 值。
4. 加入最少配置，保留每一步返回值。
5. 原始数据稳定后再换算、滤波和显示。
6. 最后加入业务状态机和异常恢复。

## 最小驱动框架

> 代码性质：示例框架，用于理解调用顺序，不能保证直接编译。

```c
typedef struct
{
    uint8_t red;
    uint8_t green;
    uint8_t blue;
} RGB_Color_t;

static RGB_Color_t pixel_buffer[RGB_PIXEL_COUNT];
static uint16_t pwm_buffer[RGB_PWM_BUFFER_LENGTH];
static TIM_HandleTypeDef *rgb_htim;
static volatile uint8_t rgb_dma_done = 1U;

void WS2813E_Init(TIM_HandleTypeDef *htim)
{
    rgb_htim = htim;
}

static uint8_t WS2813E_SetPixel(uint16_t index, RGB_Color_t color)
{
    if (index >= RGB_PIXEL_COUNT) return 0U;
    pixel_buffer[index] = color;
    return 1U;
}

void WS2813E_Refresh(void)
{
    if (rgb_htim == NULL) return;

    uint16_t length = WS2813E_EncodePixelsAndReset(
        pixel_buffer, RGB_PIXEL_COUNT,
        pwm_buffer, RGB_PWM_BUFFER_LENGTH);

    rgb_dma_done = 0U;
    if (HAL_TIM_PWM_Start_DMA(rgb_htim, RGB_TIM_CHANNEL,
                              (uint32_t *)pwm_buffer, length) != HAL_OK)
    {
        rgb_dma_done = 1U;
        Debug_ReportRgbDmaError();
    }
}

void HAL_TIM_PWM_PulseFinishedCallback(TIM_HandleTypeDef *htim)
{
    if (htim == rgb_htim)
    {
        (void)HAL_TIM_PWM_Stop_DMA(htim, RGB_TIM_CHANNEL);
        rgb_dma_done = 1U;
    }
}

RGB_Color_t color = {
    .red = App_GetRed(),
    .green = App_GetGreen(),
    .blue = App_GetBlue(),
};
if (WS2813E_SetPixel(App_GetPixelIndex(), color) && rgb_dma_done)
{
    WS2813E_Refresh();
}
```

`WS2813E_EncodePixelsAndReset()` 负责颜色顺序、bit 对应 CCR 和 reset/latch 低电平槽。颜色顺序、灯珠数量、PWM 计数和纳秒级时序必须依据具体器件手册并用示波器实测。

## 调试方法

```text
供电/共地
-> 接线和电平
-> CubeMX 与 HAL 返回值
-> 原始波形/字节/数值
-> 模块状态
-> 换算或算法
-> 显示与业务
```

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| 全不亮 | 供电/共地/方向错 | 先检查 DIN 和电源 |
| 颜色错 | RGB/GRB 顺序错 | 固定红绿蓝逐一验证 |
| 随机闪烁 | 时序被中断/电平不兼容 | 用 TIM/DMA 并测波形 |
| 后面灯不亮 | 编码长度/级联数据不足 | 核对像素和 buffer 长度 |
| MCU 复位 | 灯带电流冲击 | 独立供电、去耦、限亮度 |

## 待核对项

- [ ] WS2813E 具体版本、颜色顺序和灯珠数量
- [ ] 0/1 纳秒级窗口及 reset/latch 时间
- [ ] 课程采用 GPIO、TIM PWM 还是 DMA

以实际开发板原理图、CubeMX 配置、视频课程源码和模块手册为准。

## 与其它章节的关系

- STM32 侧依赖：[[TIM_PWM输出]]、[[DMA与缓冲区]]、[[通用调试方法]]。
- 通用 bring-up 和数据显示见 [[传感器采集与显示]]。
- 跨外设排错见 [[通用调试方法]]。

## 复习检查清单

- [ ] 能说明模块解决什么问题，以及 STM32 侧依赖哪个外设。
- [ ] 能画出供电、信号线、总线和业务数据流。
- [ ] 能完成 CubeMX 最小配置并检查 HAL 返回值。
- [ ] 能按 bring-up 顺序得到原始、可观察的数据。
- [ ] 能区分通信问题、驱动问题、换算/算法问题和显示问题。
- [ ] 能指出哪些地址、寄存器、时序、公式或库必须查手册/视频。
