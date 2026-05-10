# 22-23 定时器 PWM 输出

## PWM 基本概念

```
PWM = Pulse Width Modulation（脉冲宽度调制）

一个周期内：
┌──┐    ┌──┐    ┌──
│  │    │  │    │
┘  └────┘  └────┘
|<-- T -->|
|<CCR>|
    |<-- ARR+1 -->|

频率 = 1/T
占空比 = CCR / (ARR+1) × 100%
```

### 公式 ⭐

```
PWM频率  = TIMx_CLK / [(PSC+1) × (ARR+1)]
PWM占空比 = CCR / (ARR+1) × 100%

例：1kHz PWM，72MHz时钟
PSC = 71，ARR = 999
频率 = 72M / (72 × 1000) = 1kHz
占空比50% → CCR = 500
```

## HAL PWM 函数

```c
// 启动PWM（main函数中）
HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);

// 动态修改占空比（在while或回调中）
__HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, 500);  // 50%

// 停止PWM
HAL_TIM_PWM_Stop(&htim3, TIM_CHANNEL_1);
```

## 呼吸灯实现

```c
// 在while(1)中
// 渐亮
for(uint16_t duty = 0; duty <= 1000; duty++) {
    __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, duty);
    HAL_Delay(1);
}
// 渐灭
for(uint16_t duty = 1000; duty > 0; duty--) {
    __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, duty);
    HAL_Delay(1);
}
```

## 通用定时器 vs 高级定时器

| | 通用定时器（TIM2-5） | 高级定时器（TIM1/8） |
|--|---------------------|---------------------|
| PWM通道 | 4个 | 4个+互补输出 |
| 互补输出 | ❌ | ✅（用于电机驱动） |
| 死区时间 | ❌ | ✅（防止上下管同时导通）|
| 刹车输入 | ❌ | ✅ |
| 适用 | LED、舵机、一般PWM | 无刷电机、逆变器 |

## CubeMX 配置步骤

1. Timers → TIM3 → Channel1: PWM Generation CH1
2. 填写 PSC 和 ARR（决定频率）
3. Pulse = 初始CCR值（决定初始占空比）
4. 不需要开NVIC（普通PWM不需要中断）

## 舵机控制（扩展）

```
舵机PWM：50Hz（周期20ms）
  0° → 高电平0.5ms → 占空比2.5%
90° → 高电平1.5ms → 占空比7.5%
180°→ 高电平2.5ms → 占空比12.5%

ARR = 19999，PSC = 71（72MHz下）
0°  → CCR = 500
90° → CCR = 1500
180°→ CCR = 2500
```

## 踩坑记录

- [ ] PWM引脚要选复用推挽输出，在CubeMX里会自动配置
- [ ] TIM3的CH1对应PA6，CH2对应PA7，注意引脚对应关系
- [ ] 

---
*课程：[22-1] [22-2] [23-1]*
