# 六、定时器（TIM）应用篇

> 覆盖课程：[22-1] [22-2] [23-1] [23-2]
>
> 目标：掌握通用/高级定时器的中断应用，理解 PWM 原理，实现呼吸灯和舵机控制。

---

## [22-1] TIM 通用定时器基础中断应用

### 知识点
- 通用定时器（TIM2/3/4）中断
- 定时时间计算
- `HAL_TIM_Base_Start_IT()` 启动

### 关键原理

通用定时器 16 位（PSC、ARR 最大 65535），除了定时中断还能做 PWM、输入捕获、输出比较。

#### 重温定时器时钟坑

```
TIM2 所在 APB1 = 36MHz，分频 = /2（≠1）
→ TIM2_CLK = 36MHz × 2 = 72MHz  ← 不是 36MHz！
```

#### 计算实例

```
目标：1ms 定时，TIM2_CLK = 72MHz
PSC = 71       → 72M / 72 = 1MHz (1μs 一次)
ARR = 999      → 1000 次 = 1ms
T = 72 × 1000 / 72M = 1ms ✓
```

### CubeMX 配置步骤

1. Timers → TIM2 → Clock Source: Internal Clock
2. 填 PSC 和 ARR（决定定时周期）
3. NVIC → TIM2 global interrupt → Enable

### HAL 库代码模板

```c
// main() 初始化
HAL_TIM_Base_Start_IT(&htim2);  // 带 _IT！

// 回调
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM2)
    {
        // 定时触发
    }
}
```

### 易错点
- `_Start` 和 `_Start_IT` 搞混 → 中断不触发
- 定时时间算不准 → 没考虑 APB1 ×2 规则

---

## [22-2] TIM 高级定时器基础中断应用

### 知识点
- 高级定时器 TIM1/8 的特点
- 与通用定时器的区别

### 关键原理

| | 通用 TIM2~5 | 高级 TIM1/8 |
|--|-----------|-----------|
| 位数 | 16 位 | 16 位 |
| 定时中断 | ✅ | ✅ |
| PWM | 4 通道 | 4 通道 + 3 互补 |
| 死区插入 | ❌ | ✅ |
| 刹车输入 | ❌ | ✅ |
| 适用 | LED、舵机 | 电机半桥/全桥 |

> 只做定时中断的话，高级定时器和通用定时器用法完全一样。高级定时器多的功能（互补输出、死区、刹车）主要用在电机驱动场景。

### 易错点
- TIM1 在 APB2 上（TIM1_CLK = 72MHz），TIM2 在 APB1 上但也是 72MHz——确认时钟来源再算时间

---

## [23-1] TIM 通道输出应用之 PWM 控制舵机

### 知识点
- PWM 本质和工作原理
- 四个核心寄存器（PSC / ARR / CNT / CCR）
- 舵机控制协议

### 关键原理

#### 一句话理解 PWM

> PWM 不是改变电压，是用固定频率方波的**高电平占比**来控制平均输出效果。

#### 寄存器速记

| 寄存器 | 一句话 |
|--------|--------|
| PSC | 预分频，降速用 |
| ARR | 计数上限，决定周期 |
| CNT | 当前计数值，不停循环 |
| CCR | 比较值，决定高电平时长 |

#### 三个公式

```
PWM 频率   = TIMx_CLK / [(PSC + 1) × (ARR + 1)]
PWM 占空比  = CCR / (ARR + 1) × 100%
高电平时间  = CCR × (PSC + 1) / TIMx_CLK
```

#### PWM 模式

| 模式 | CNT < CCR 时 | 默认 |
|------|------------|------|
| Mode 1 | 输出高电平 | ⭐ 常用 |
| Mode 2 | 输出低电平 | 少用 |

#### 舵机控制

```
舵机用 50Hz PWM（周期 20ms），高电平持续时间决定角度：

TIMx_CLK = 72MHz, PSC = 71, ARR = 19999 → 50Hz

角度    脉宽       CCR
 0° →  0.5ms  →   500
90° →  1.5ms  →  1500
180°→  2.5ms  →  2500
```

> 🔴 不同舵机的极限角度不同。先用 1000~2000 范围测试，确认能正常工作再慢慢扩大。

### CubeMX 配置步骤

1. Timers → TIM3 → Channel1: PWM Generation CH1
2. 填写 PSC 和 ARR，Pulse = 初始 CCR
3. GPIO 会自动切到复用推挽，核对引脚：
   - TIM3_CH1 常见 PA6，CH2→PA7（具体看芯片数据手册）

### HAL 库代码模板

```c
// 启动 PWM
HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);

// 修改占空比（CCR 必须 ≤ ARR）
__HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, 1500);  // 舵机到 90°

// 停止
HAL_TIM_PWM_Stop(&htim3, TIM_CHANNEL_1);
```

### 易错点
- PWM 引脚没输出 → 检查通道和引脚是否匹配、是否调了 `_Start`
- 舵机抖动或不转 → **供电不足**（舵机不能从 ST-Link 3.3V 取电，需外接 5V~6V 电源并共地）
- 舵机烧了/堵转 → 高电平脉宽超出舵机承受范围
- 占空比 ≠ 角度线性关系，舵机关注的是**脉宽**而非百分比

---

## [23-2] TIM 通道输出应用之 PWM 实现呼吸灯

### 知识点
- LED 调光原理（PWM + 人眼视觉暂留）
- 动态修改 CCR 实现渐亮渐灭

### 关键原理

```
人眼对 50Hz 以上的闪烁感知为连续光。
亮度 = PWM 占空比 → 0% 全灭，100% 全亮

呼吸灯 → CCR 从 0 平滑变到 ARR（渐亮）再变回 0（渐灭）
```

#### 频率选择

| 场景 | 推荐频率 | 原因 |
|------|---------|------|
| LED 调光 | 1kHz+ | 高于人眼闪烁感知 |
| 舵机 | 50Hz | 舵机协议标准 |
| 电机调速 | 10~20kHz | 避开人耳听觉范围 |
| 无源蜂鸣器 | 1~5kHz | 人耳敏感范围 |

### HAL 库代码模板

```c
// 呼吸灯（while(1) 中——教学写法，会阻塞 CPU 约 2 秒）
while (1)
{
    // 渐亮
    for (uint16_t duty = 0; duty <= 1000; duty++)
    {
        __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, duty);
        HAL_Delay(1);
    }
    // 渐灭
    for (uint16_t duty = 1000; duty > 0; duty--)
    {
        __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, duty);
        HAL_Delay(1);
    }
}
```

> 💡 这是教学写法。更好的是把呼吸灯状态机放在定时器中断里，CPU 可以同时干别的事。

### 易错点
- `__HAL_TIM_SET_COMPARE` 改占空比没生效 → 检查通道号是否正确、定时器是否已启动
- CCR 值超过 ARR → 占空比异常
- 呼吸灯闪烁（不流畅） → PWM 频率太低，提高到 1kHz 以上

---

*课程：[22-1] [22-2] [23-1] [23-2]*
