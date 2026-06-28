---
type: concept
status: refactor
course: stm32-f103-hal
source:
  - local-notes
  - st-reference-manual
  - st-datasheet
  - st-hal-documentation
tags:
  - stm32
  - dwt
  - debug
---
# DWT 与调试计时

> 能力定位：用 CPU 周期计数测量程序耗时，并提供微秒级计时基础。
>
> 原始来源：[[10_DWT与单总线传感器]]。

---

## 本节目标

- 能初始化 CYCCNT 并换算微秒
- 能测量函数运行时间
- 能判断 DWT 忙等是否适合当前场景

## 核心概念

### 周期计时

DWT 是 Cortex-M 调试组件，`CYCCNT` 可记录 CPU 周期数。已知 `SystemCoreClock` 后，可以把周期差换算成时间。

```text
time_us = cycles * 1000000 / SystemCoreClock
```

### 工程用途

- 比较两种算法的执行时间。
- 定位中断或临界代码耗时。
- 构建短时间微秒延时。
- 观察优化等级对运行时间的影响。

与 `HAL_Delay()` 的区别：`HAL_Delay()` 依赖毫秒 tick 并阻塞等待，适合普通毫秒级流程；DWT 直接读取 CPU 周期，适合短时间测量和微秒级等待，但同样会占用 CPU。

### 使用边界

- DWT 依赖内核和调试相关寄存器配置。
- 微秒忙等会占用 CPU，不适合长时间等待。
- 系统时钟变化后必须更新换算依据。
- 单步调试会干扰被测时间，应使用断点或连续运行测量。

> 视频待核对：课程的 DWT 初始化封装、Debug 观察步骤和不同编译环境差异。

### 知识地图

| 名词 | 工程含义 |
|---|---|
| DWT | Cortex-M 调试与跟踪组件 |
| CYCCNT | CPU 周期计数器 |
| SystemCoreClock | 周期到时间换算基准 |
| 忙等 | CPU 持续轮询直到时间到 |
| 测量扰动 | 断点/单步对运行时间的影响 |

## 本质理解

DWT 给出的是 CPU 走过多少周期，而不是抽象的“延时”。这使它适合短时间测量和微秒级等待，但它仍占用 CPU，不能替代定时器、DMA 或任务调度。

为什么这样做：先建立硬件和数据流模型，再记 HAL API，才能在 API 变化或板卡变化时仍然定位问题。

## STM32F103 / HAL 中的实现方式

- Cortex-M3 的 DWT 需开启跟踪和 CYCCNT 计数；不同内核、低功耗和调试配置支持情况需核对。
- 时间差应使用无符号减法，允许计数器回绕。
- DHT11 等微秒时序可使用 DWT；WS2813E 的纳秒窗口通常更适合 TIM/DMA 或经过严格验证的实现。

## CubeMX 配置要点

1. DWT 不需要 CubeMX 外设实例。
2. 先确认 `SystemCoreClock` 与真实主频一致。
3. Debug 时在 start/stop/delta 变量上观察，不要把单步停顿算入测量。

## 计时接口：HAL Tick 与 CMSIS / DWT

> 这部分不是 HAL API，而是 CMSIS/内核调试组件相关操作。`CoreDebug`、`DWT` 和 `SystemCoreClock` 不能包装成并不存在的 HAL 函数名；是否可用还取决于 Cortex-M 内核、调试配置和低功耗状态。

### API 速查

| 接口 / 对象 | 类型 | 解决什么问题 | 常见调用位置 |
|---|---|---|---|
| `CoreDebug->DEMCR` | CMSIS 内核寄存器映射 | 允许跟踪/调试组件工作 | DWT 初始化 |
| `DWT->CTRL` | CMSIS 内核寄存器映射 | 使能 `CYCCNT` 周期计数 | DWT 初始化 |
| `DWT->CYCCNT` | CMSIS 内核寄存器映射 | 读取/清零 CPU 周期数 | 测量起止点、短延时 |
| `SystemCoreClock` | CMSIS 系统时钟变量 | 把周期数换算成时间 | 时间换算 |
| `HAL_GetTick()` | HAL API | 获取默认毫秒时基 | 长超时、普通节拍 |
| `HAL_Delay()` | HAL API | 毫秒级阻塞等待 | 简单初始化/实验 |

### 使用注意

| 接口 / 对象 | 前置条件 | 返回值 / 阻塞 | 常见坑 |
|---|---|---|---|
| `CoreDebug->DEMCR` | 当前内核实现 DWT | 直接寄存器访问；非阻塞 | 把它误认为 HAL API；覆盖其它控制位 |
| `DWT->CTRL` | 跟踪功能已打开 | 直接寄存器访问；非阻塞 | 未使能计数就读取，数值一直不变 |
| `DWT->CYCCNT` | `CYCCNTENA` 已置位 | 返回/写入 32 位计数；非阻塞 | 忽略回绕；单步和中断会改变测量结果 |
| `SystemCoreClock` | 变量与真实 HCLK 同步 | 读取变量；非阻塞 | 改时钟后未更新，微秒换算整体偏差 |
| `HAL_GetTick()` | HAL tick 正常运行 | 返回 `uint32_t`；非阻塞 | 精度只有 tick 级，不适合微秒时序 |
| `HAL_Delay()` | HAL tick 正常运行 | `void`；阻塞 | 不能替代 DWT 的微秒测量；中断中调用风险高 |

### 典型调用顺序

```text
确认 Cortex-M3 支持 DWT
-> 使能 CoreDebug 跟踪位
-> 清零并使能 DWT->CYCCNT
-> 读取 start
-> 连续运行被测代码
-> 读取 end，以无符号差值换算时间
```

### 什么时候用哪个接口？

| 场景 | 推荐接口 |
|---|---|
| 普通毫秒超时/状态机 | `HAL_GetTick()` |
| 允许阻塞的毫秒等待 | `HAL_Delay()` |
| 测函数 CPU 周期 | `DWT->CYCCNT` |
| 短微秒等待 | 经验证的 DWT 忙等或 TIM |
| 纳秒级严格输出时序 | 优先 TIM/DMA 等硬件方案，不依赖普通 HAL GPIO 调用延时 |

## 最小实验 / 最小框架

> 代码性质：示例框架，用于理解调用顺序，不能保证直接编译。

```c
/* CMSIS / Cortex-M 内核调试组件操作，不是 HAL API。 */
static void DWT_Delay_Init(void)
{
    CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;
    DWT->CYCCNT = 0U;
    DWT->CTRL |= DWT_CTRL_CYCCNTENA_Msk;
}

static uint32_t DWT_GetCycles(void)
{
    return DWT->CYCCNT;
}

static void DWT_Delay_us(uint32_t us)
{
    uint32_t cycles_per_us = SystemCoreClock / 1000000U;
    uint32_t wait_cycles = cycles_per_us * us;
    uint32_t start = DWT_GetCycles();

    while ((uint32_t)(DWT_GetCycles() - start) < wait_cycles)
    {
        /* 短时间忙等；不要用于长延时。 */
    }
}

static uint32_t Measure_TargetCycles(void)
{
    uint32_t start = DWT_GetCycles();
    Target_Function();
    return (uint32_t)(DWT_GetCycles() - start);
}

/* main() 中 SystemClock_Config() 完成后调用。 */
DWT_Delay_Init();
uint32_t cycles = Measure_TargetCycles();
float time_us = (float)cycles * 1000000.0f / (float)SystemCoreClock;
```

`SystemCoreClock` 必须与真实 HCLK 一致。DWT 支持情况、低功耗行为和中断对测量的影响以实际 Cortex-M 内核与调试配置为准。

## 调试方法

```text
确认 DWT 计数器在递增
-> 确认 SystemCoreClock
-> 连续运行到断点而非逐行单步
-> 重复测量并比较波动
-> 关闭不相关中断验证干扰来源
```

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| CYCCNT 一直为 0 | 跟踪或计数未使能 | 检查 DEMCR/CTRL |
| 微秒换算错误 | 主频变量错误 | 更新 SystemCoreClock |
| 测量时间巨大 | 断点停顿被计入 | 改为连续运行 |
| 系统响应变差 | 长时间忙等 | 改用 TIM/中断 |
| 低功耗后异常 | 计数支持或时钟变化 | 按内核/低功耗状态核对 |

## 与其它章节的关系

- [[RCC时钟树]] 提供主频依据。
- [[DHT11单总线温湿度]] 使用微秒级时序。
- [[通用调试方法]] 说明 Debug 与波形工具选择。

## 复习检查清单

- [ ] 能解释 `CYCCNT` 记录的是 CPU 周期而不是直接的微秒数。
- [ ] 能说明 `SystemCoreClock` 错误会怎样影响时间换算。
- [ ] 能写出 DWT 初始化、读取周期数和无符号差值的调用顺序。
- [ ] 能用连续运行和断点测量函数耗时，避免把单步停顿计入结果。
- [ ] 能判断普通毫秒超时、短微秒等待和严格输出时序分别应选什么方案。
- [ ] 能说明 DWT 忙等、计数器回绕、中断干扰和低功耗状态的风险。
