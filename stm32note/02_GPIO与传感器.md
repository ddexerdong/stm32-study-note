# 02 GPIO 与传感器

> 覆盖课程：[9] [10] [11] [12] [13] [14] [15] [16]
>
> 目标：掌握 GPIO 输入/输出、基础延时、按键消抖、蜂鸣器驱动和数字传感器读取。
>
> 关联章节：[[03_中断与定时器|中断]] · [[05_ADC模数转换|ADC]] · [[踩坑日记|踩坑日记]]

---

## 本节目标

完成本节后，应能做到：

- 使用 GPIO 输出控制 LED、蜂鸣器等简单外设。
- 使用 GPIO 输入读取按键和数字传感器模块。
- 理解推挽、开漏、上拉、下拉、浮空输入和模拟输入的区别。
- 掌握按键消抖和数字传感器读取的基本工程写法。

---

## 核心概念

### 知识地图

| 名词 | 工程含义 | 常见位置 |
|------|----------|----------|
| GPIO | General Purpose Input/Output，通用输入输出引脚 | [9] |
| 推挽输出 | 内部上下两个 MOS 管交替导通，可主动输出高电平和低电平 | [9] |
| 开漏输出 | 只有下拉 MOS 管，只能主动拉低，高电平依赖上拉电阻 | [9] |
| 上拉电阻 | 引脚到 VCC 的电阻，引脚悬空时保持高电平 | [11] |
| 下拉电阻 | 引脚到 GND 的电阻，引脚悬空时保持低电平 | [11] |
| 施密特触发器 | 输入端整形电路，可过滤小幅毛刺，把模拟边沿变成干净数字电平 | [11] |
| 按键抖动 | 机械触点闭合/断开时 5~20 ms 内的多次弹跳 | [11] |
| 消抖 | 用延时、时间戳或状态机过滤抖动，确保一次动作只触发一次 | [11] |
| SysTick | Cortex-M3 内核自带 24 位定时器，HAL 默认用它产生 1 ms 时基 | [10] |
| LM393 | 双路电压比较器，把模拟电压和阈值比较后输出高低电平 | [13] |
| NPN 三极管 | 用 GPIO 小电流控制蜂鸣器等较大电流负载的开关器件 | [12] |

### GPIO 常用 HAL API

| 函数 | 返回值 | 作用 |
|------|--------|------|
| `HAL_GPIO_WritePin(GPIOx, Pin, State)` | `void` | 写引脚输出电平 |
| `HAL_GPIO_TogglePin(GPIOx, Pin)` | `void` | 翻转引脚输出电平 |
| `HAL_GPIO_ReadPin(GPIOx, Pin)` | `GPIO_PinState` | 读取引脚输入电平 |
| `HAL_GPIO_LockPin(GPIOx, Pin)` | `HAL_StatusTypeDef` | 锁定引脚配置，直到下次复位 |
| `HAL_GPIO_EXTI_IRQHandler(Pin)` | `void` | HAL 内部 EXTI 中断处理入口 |
| `HAL_GPIO_EXTI_Callback(Pin)` | `void` | 用户实现的 EXTI 回调 |

参数说明：

- `GPIOx`：端口基地址指针，例如 `GPIOA`、`GPIOB`、`GPIOC`。
- `Pin`：16 位位掩码，例如 `GPIO_PIN_0`、`GPIO_PIN_13`，可用 `|` 组合。
- `State`：`GPIO_PIN_SET` 或 `GPIO_PIN_RESET`。

注意：STM32 HAL 没有通用的 `HAL_GPIO_Write(GPIOx, Value)` / `HAL_GPIO_Read(GPIOx)` API。若要整口读写，应直接理解并谨慎操作 `ODR`、`IDR`、`BSRR` 等寄存器，入门阶段优先用 `WritePin/ReadPin`。

---

## 本质理解

GPIO 的本质是：芯片把一个引脚连接到一套可配置的输入/输出电路。配置不同，电路连接方式不同，引脚表现也不同。

| 模式 | 内部行为 | 典型应用 |
|------|----------|----------|
| 推挽输出 | 上管拉高、下管拉低，可主动输出 0/1 | LED、片选、普通控制脚 |
| 开漏输出 | 只能拉低，释放时为高阻态 | I2C、线与逻辑 |
| 上拉输入 | 悬空时被内部电阻拉到高电平 | 按键接 GND |
| 下拉输入 | 悬空时被内部电阻拉到低电平 | 按键接 VCC |
| 浮空输入 | 无内部上下拉，电平由外部决定 | 外部电路已有明确上下拉 |
| 模拟输入 | 关闭数字输入逻辑，连接 ADC 模拟通道 | ADC 采样 |
| 复用推挽 | 引脚控制权交给外设，输出级为推挽 | UART TX、PWM、SPI |
| 复用开漏 | 引脚控制权交给外设，输出级为开漏 | I2C SDA/SCL |

工程判断原则：

- 引脚要**输出控制信号**：优先推挽输出。
- 引脚要**读取外部状态**：用输入模式，并保证电平不悬空。
- 引脚交给**外设**：用复用功能，不要再当普通 GPIO 手动写。
- 引脚接 **ADC**：必须用模拟输入。

---

## 工作流程 / 原理

### GPIO 输出与 LED

LED 是非线性器件，必须串联限流电阻。

```text
R = (VCC - Vf) / If
  = (3.3 V - 2.0 V) / 0.01 A
  = 130 ohm

工程取值：220 ohm、330 ohm、470 ohm、1 k ohm 都常见。
```

普通 LED 取 5~10 mA 更保守。STM32 GPIO 不是电源驱动器，单个引脚即使规格允许较大电流，也不建议长期贴近极限使用。

### GPIO 输出速度

GPIO Speed 控制的是输出边沿快慢，不是业务数据速度。STM32F1 常见配置是 2 MHz / 10 MHz / 50 MHz。

| 速度 | 特点 | 适用 |
|------|------|------|
| 低速 | 边沿慢，EMI 小 | LED、继电器、蜂鸣器控制 |
| 中速 | 折中 | 普通控制信号 |
| 高速 | 边沿快，EMI 大 | SPI、较高频 PWM |

能用低速解决的 GPIO，不要习惯性全配高速。

### SysTick 与 HAL 延时

SysTick 是 Cortex-M3 内核定时器，CubeMX 默认让它每 1 ms 中断一次：

```text
SysTick 中断
-> HAL_IncTick()
-> uwTick++
-> HAL_GetTick() 返回系统运行毫秒数
-> HAL_Delay(ms) 等待时间差达到目标值
```

`HAL_Delay()` 是**阻塞延时**。点灯教学可以用，工程里大量使用会让 CPU 在等待期间无法处理其他任务。

### 按键为什么需要消抖

机械按键按下/松开时触点会弹跳：

```text
理想波形：    ────────┐         └────────
                      └─────────┘

实际波形：    ────────┐ ┌─┐ ┌───┐└────────
                      └─┘ └─┘   └─┘
                      <- 5~20 ms ->
```

如果不消抖，一次按下可能被程序识别为多次触发。

### 上拉输入为什么常配按键接 GND

上拉输入时，按键一端接 GPIO，另一端接 GND：

- 未按下：内部上拉电阻把引脚拉到高电平。
- 按下：按键把引脚直接接到 GND，读到低电平。

这种接法布线简单，GND 引脚多，也更符合常见核心板实验习惯。

### 蜂鸣器为什么要三极管驱动

有源蜂鸣器和无源蜂鸣器的区别：

| 对比项 | 有源蜂鸣器 | 无源蜂鸣器 |
|--------|------------|------------|
| 内部结构 | 自带振荡电路 | 只有蜂鸣片/线圈 |
| 驱动方式 | 给直流电平即可响 | 需要 PWM 方波 |
| 音调 | 基本固定 | 由 PWM 频率决定 |
| 快速判断 | 通电就响 | 直流通电通常不响 |

蜂鸣器工作电流通常超过 GPIO 合理驱动范围，应用 NPN 三极管做低端开关：

```text
          VCC
           |
        蜂鸣器
           |
           C
STM32 --1k--B   NPN（如 S8050）
           E
           |
          GND
```

基极电阻必须保留：

```text
Ib = (3.3 V - 0.7 V) / 1 k ohm = 2.6 mA
```

这样 GPIO 只负责提供很小的基极电流，蜂鸣器电流由三极管承载。

### 数字传感器模块的统一结构

光敏、反射、热敏、火焰等数字传感器模块常见结构相同：

```text
传感元件（光敏电阻 / 热敏电阻 / 红外接收管 / 火焰传感器）
    -> 分压电压（模拟量）
LM393 比较器 <- 电位器设定阈值
    -> DO 输出高/低电平
```

- `DO` 是数字输出，只能表示“是否超过阈值”。
- 蓝色电位器调的是比较阈值。
- 不同模块的有效电平可能相反，必须实测。
- 需要连续物理量时，应接 `AO` 到 ADC，而不是只看 `DO`。

---

## CubeMX 配置

### GPIO 输出

1. `Pinout` 点击目标引脚，例如 `PB0`。
2. 选择 `GPIO_Output`。
3. `System Core -> GPIO` 中设置：
   - `GPIO output level`：初始电平。
   - `GPIO mode`：推挽输出或开漏输出。
   - `GPIO Pull-up/Pull-down`：一般输出模式不需要上下拉。
   - `Maximum output speed`：LED/蜂鸣器控制用低速即可。

### GPIO 输入按键

1. `Pinout` 点击目标引脚，例如 `PA0`。
2. 选择 `GPIO_Input`。
3. `GPIO Pull-up/Pull-down` 选择 `Pull-up`。
4. 按键另一端接 GND。

### 数字传感器输入

1. 传感器 `VCC` 接 3.3 V 或模块标注允许的电源。
2. 传感器 `GND` 接 STM32 `GND`。
3. 传感器 `DO` 接任意 GPIO 输入脚。
4. CubeMX 将该引脚配置为 `GPIO_Input`，通常可开内部上拉。

---

## HAL 函数 / API

### `HAL_GPIO_WritePin`

函数：

```c
void HAL_GPIO_WritePin(GPIO_TypeDef *GPIOx,
                       uint16_t GPIO_Pin,
                       GPIO_PinState PinState);
```

作用：

- 调用后，指定引脚输出 `SET` 或 `RESET`。
- 内部主要通过 `BSRR` 寄存器完成置位/复位，避免普通读-改-写带来的竞争问题。
- 不调用时，输出引脚不会按程序要求改变电平。
- 适合 LED、蜂鸣器、继电器驱动、模块使能脚等明确高低电平控制。

参数说明：

| 参数 | 说明 |
|------|------|
| `GPIOx` | GPIO 端口，例如 `GPIOA`、`GPIOB` |
| `GPIO_Pin` | GPIO 引脚位掩码，例如 `GPIO_PIN_0` |
| `PinState` | `GPIO_PIN_SET` 或 `GPIO_PIN_RESET` |

```c
HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);
HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET);
```

### `HAL_GPIO_TogglePin`

函数：

```c
void HAL_GPIO_TogglePin(GPIO_TypeDef *GPIOx, uint16_t GPIO_Pin);
```

作用：

- 调用后，指定输出引脚电平翻转。
- 不调用时，引脚保持原状态。
- 适合 LED 闪烁和状态指示。

```c
HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);
```

注意：引脚必须配置为输出模式。输入模式下调用它不会改变外部电路电平。

### `HAL_GPIO_ReadPin`

函数：

```c
GPIO_PinState HAL_GPIO_ReadPin(GPIO_TypeDef *GPIOx, uint16_t GPIO_Pin);
```

作用：

- 调用后，读取输入引脚当前数字电平。
- 返回 `GPIO_PIN_SET` 或 `GPIO_PIN_RESET`。
- 不调用时，程序无法知道按键或数字传感器当前状态。
- 适合按键、避障模块、循迹模块、比较器模块的 `DO` 输出读取。

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)
{
    // 上拉输入下，低电平通常表示按键按下
}
```

---

## 示例代码

### LED 以 1 Hz 闪烁

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
while (1)
{
    HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);
    HAL_Delay(500);
}
```

周期 = 500 ms 亮 + 500 ms 灭 = 1 s。

### 流水灯

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
uint16_t leds[] = {GPIO_PIN_0, GPIO_PIN_1, GPIO_PIN_2};

for (int i = 0; i < 3; i++)
{
    HAL_GPIO_WritePin(GPIOB, leds[i], GPIO_PIN_SET);
    HAL_Delay(200);
    HAL_GPIO_WritePin(GPIOB, leds[i], GPIO_PIN_RESET);
}
```

### 延时消抖

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)
{
    HAL_Delay(20);
    if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)
    {
        HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);

        while (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)
        {
            // 等待松开，阻塞写法
        }
    }
}
```

这个写法简单，但 `HAL_Delay(20)` 和等待松开的 `while` 都会阻塞 CPU。只有单任务实验时适合。

### 状态机消抖

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
typedef enum
{
    KEY_IDLE,
    KEY_DEBOUNCE,
    KEY_PRESSED
} KeyState;

KeyState keyState = KEY_IDLE;
uint32_t keyTimer = 0;

void Key_Task(void)
{
    switch (keyState)
    {
        case KEY_IDLE:
            if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)
            {
                keyState = KEY_DEBOUNCE;
                keyTimer = HAL_GetTick();
            }
            break;

        case KEY_DEBOUNCE:
            if (HAL_GetTick() - keyTimer > 20)
            {
                if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)
                {
                    keyState = KEY_PRESSED;
                }
                else
                {
                    keyState = KEY_IDLE;
                }
            }
            break;

        case KEY_PRESSED:
            if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_SET)
            {
                HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);
                keyState = KEY_IDLE;
            }
            break;

        default:
            keyState = KEY_IDLE;
            break;
    }
}
```

在 `while (1)` 中周期性调用：

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
while (1)
{
    Key_Task();
    // 其他任务也可以继续执行
}
```

### 按键控制蜂鸣器

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)
{
    HAL_Delay(20);
    if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)
    {
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_8, GPIO_PIN_SET);

        while (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)
        {
            // 按着一直响
        }

        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_8, GPIO_PIN_RESET);
    }
}
```

### 数字传感器通用读取模板

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_1) == GPIO_PIN_RESET)
{
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);
}
else
{
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET);
}
```

这里假设 `DO = 0` 表示触发。实际模块必须用万用表或串口打印确认有效电平。

---

## 代码执行流程

```text
CubeMX 配置 GPIO 模式
↓
生成 MX_GPIO_Init()
↓
main() 中完成 HAL 和 GPIO 初始化
↓
输出类外设：HAL_GPIO_WritePin() / TogglePin()
↓
输入类外设：HAL_GPIO_ReadPin()
↓
根据电平状态控制 LED、蜂鸣器或业务逻辑
```

---

## 常见错误

| 问题 | 常见原因 | 解决方法 |
|------|----------|----------|
| LED 不亮 | 没串限流电阻、极性接反、GPIO 模式错误 | 检查硬件接线和 CubeMX 模式 |
| LED 烧坏或 GPIO 损坏 | 直接接 LED 无限流 | 串 220 ohm~1 k ohm 电阻 |
| 按键按一次触发多次 | 没有消抖 | 延时消抖或状态机消抖 |
| 按键状态一直不变 | 上拉/下拉和接线不匹配 | 上拉输入配按键接 GND |
| `while` 等待松开卡死 | 阻塞式按键处理 | 多任务场景改为状态机 |
| 蜂鸣器不响 | 有源/无源类型搞错，或驱动电流不够 | 有源用电平，无源用 PWM，并加三极管驱动 |
| GPIO 烧坏 | 蜂鸣器/继电器直接接 GPIO | 使用三极管、MOS 管或驱动芯片 |
| 传感器不触发 | 没共地、阈值没调、有效电平判断反了 | 先量 DO 电压，再调电位器 |
| 模块输出不稳定 | 杜邦线接触不良或电源噪声 | 换线、共地、检查供电 |

---

## 调试方法

### GPIO 输出调试

1. 先用万用表量 GPIO 引脚电压是否随代码变化。
2. 再确认 LED、电阻、极性是否正确。
3. 如果 GPIO 有变化但负载无响应，问题大概率在外部电路。
4. 如果 GPIO 无变化，回查 CubeMX 模式、初始化函数和引脚号。

### 按键输入调试

1. 未按下时量 GPIO 电压，确认是高还是低。
2. 按下时再量一次，确认电平是否翻转。
3. 用串口打印 `HAL_GPIO_ReadPin()` 的结果确认代码判断。
4. 出现多次触发时，优先加消抖而不是怀疑 HAL。

### 数字传感器调试

1. 量 `VCC-GND`，确认供电。
2. 量 `DO-GND`，人为改变环境，确认电平是否变化。
3. 确认 STM32 和模块共地。
4. 调蓝色电位器，观察 DO 翻转。
5. 换杜邦线，排除接触不良。
6. 核对 GPIO 引脚号和有效电平。

---

## 工程意义

GPIO 是嵌入式工程里最基本的硬件接口。它看起来只是读 0/1、写 0/1，但背后已经包含了很多工程习惯：

- 任何外设通信前，先保证**供电和共地**。
- 任何输入脚都不能长期悬空。
- 任何负载都要先估算电流，GPIO 不是电源。
- 教学实验可以阻塞，实际项目要逐步改成状态机或中断/定时任务。
- 数字传感器的 `DO` 只是阈值判断，精确测量要走 ADC 或数字通信协议。

---

## 易混淆点

| 容易混淆 | 正确理解 |
|----------|----------|
| GPIO Speed 和程序运行速度 | GPIO Speed 控制边沿快慢，不是 CPU 或代码速度 |
| `GPIO_PIN_0` 和数字 0 | `GPIO_PIN_0` 是位掩码 `0x0001` |
| 高电平和“触发” | 触发电平由模块电路决定，不一定是高电平 |
| 有源蜂鸣器和无源蜂鸣器 | 有源给电平就响，无源需要 PWM |
| 上拉输入和按键逻辑 | 上拉输入常见逻辑是未按=1，按下=0 |
| 延时消抖和状态机消抖 | 延时消抖简单但阻塞，状态机适合多任务 |
| `DO` 和 `AO` | `DO` 是数字阈值，`AO` 是连续模拟量 |

---

## 我的理解

GPIO 这一章真正要练的是“硬件状态”和“代码判断”之间的对应关系。

我现在的理解是：

- 输出脚不是随便接负载，先看电流。
- 输入脚不是随便悬空，必须有明确的上拉或下拉。
- 按键不是理想开关，必须考虑抖动。
- 数字传感器模块的核心就是“传感器分压 + LM393 比较器 + 电位器阈值”。
- 代码里看到 `GPIO_PIN_SET/RESET` 时，要立刻联想到真实电压，再联想到模块的有效电平。

以后做任何 GPIO 实验，我先问三件事：

```text
这个引脚现在是输入还是输出？
这个电平在硬件上代表什么？
这个外设和 STM32 有没有共地、有没有超电流？
```

---

## [9] GPIO输出与点灯

> 来源：PDF 强对应 + 现有笔记整理

### 实验目标

用一个 GPIO 输出高低电平，控制 LED 亮灭，确认“CubeMX 配引脚 -> HAL 写电平 -> 硬件响应”这条最基础链路。

### 输入 / 输出关系

| 项目 | 含义 |
|------|------|
| 输入 | 程序里的开灯/关灯逻辑 |
| 输出 | GPIO 引脚输出高电平或低电平 |
| 观察 | LED 亮灭是否符合预期 |

LED 可能是高电平点亮，也可能是低电平点亮，以实际开发板原理图和 CubeMX 配置为准。

### CubeMX 配置要点

1. 选择 LED 所在 GPIO 引脚。
2. 配置为 `GPIO_Output`。
3. 设置默认输出电平，避免上电瞬间误亮或误灭。
4. 设置合适的 GPIO Speed，普通 LED 使用低速或中速即可。
5. 生成工程后，业务代码写在 `Core/Src/main.c` 的 USER CODE 区域。

### 最小业务逻辑

常见位置：`Core/Src/main.c`

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
while (1)
{
    HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_SET);
    HAL_Delay(500);
    HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_RESET);
    HAL_Delay(500);
}
```

### 调试观察点

- 先看 LED 是否接到你配置的那个 GPIO。
- 再看 LED 是高电平亮还是低电平亮。
- 用万用表量 GPIO 输出电压，比只看 LED 更可靠。
- 如果程序能下载但 LED 不动，先确认 `MX_GPIO_Init()` 是否被调用。

### 常见坑

- 把 LED 有效电平理解反。
- CubeMX 配了 A 引脚，杜邦线接到 B 引脚。
- USER CODE 写到会被 CubeMX 覆盖的位置。

## [10] 基础延时与三个流水灯

> 来源：PDF 强对应 + 现有笔记整理

### 实验目标

理解 `HAL_Delay()` 依赖 SysTick 毫秒节拍，用三个 GPIO 做顺序亮灭，形成最简单的流水灯。

### 输入 / 输出关系

| 项目 | 含义 |
|------|------|
| 输入 | 主循环中的顺序执行逻辑和延时时间 |
| 输出 | 三个 GPIO 依次翻转 |
| 观察 | 三个 LED 是否按固定节奏流动 |

### CubeMX 配置要点

1. 配置三个 LED 对应 GPIO 为输出模式。
2. 保持 HAL 默认 SysTick 时基，不要随意关闭全局中断。
3. 确认系统时钟配置正常，否则延时节奏会异常。
4. 生成工程后，在主循环 USER CODE 区域写流水灯逻辑。

### 最小业务逻辑

常见位置：`Core/Src/main.c`

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
while (1)
{
    HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, GPIO_PIN_SET);
    HAL_GPIO_WritePin(LED2_GPIO_Port, LED2_Pin, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(LED3_GPIO_Port, LED3_Pin, GPIO_PIN_RESET);
    HAL_Delay(200);

    HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(LED2_GPIO_Port, LED2_Pin, GPIO_PIN_SET);
    HAL_GPIO_WritePin(LED3_GPIO_Port, LED3_Pin, GPIO_PIN_RESET);
    HAL_Delay(200);

    HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(LED2_GPIO_Port, LED2_Pin, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(LED3_GPIO_Port, LED3_Pin, GPIO_PIN_SET);
    HAL_Delay(200);
}
```

### 调试观察点

- 单独点亮每个 LED，确认三路接线没交叉。
- 调大延时到 1000 ms，先用肉眼确认顺序。
- 如果所有 LED 都不亮，回到 `[9]` 单灯实验排查。

### 常见坑

- 三个灯的宏名和实际接线顺序对不上。
- `HAL_Delay()` 放在中断回调里导致系统卡顿。
- LED 有效电平不同，导致看起来顺序反了。

## [11] GPIO输入与按键开关灯

> 来源：PDF 强对应 + 现有笔记整理

### 实验目标

读取按键输入电平，并在确认一次有效按下后切换 LED 状态。

### 输入 / 输出关系

| 项目 | 含义 |
|------|------|
| 输入 | 按键引脚电平变化 |
| 输出 | LED 状态切换 |
| 观察 | 按一次按键，LED 只改变一次 |

按键是上拉输入还是下拉输入，以实际开发板原理图和 CubeMX 配置为准。

### CubeMX 配置要点

1. LED 引脚配置为 GPIO 输出。
2. 按键引脚配置为 GPIO 输入。
3. 根据硬件选择 Pull-up、Pull-down 或 No pull。
4. 生成工程后，在主循环里做轮询和消抖。

### 最小业务逻辑

常见位置：`Core/Src/main.c`

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
uint8_t led_state = 0;

while (1)
{
    if (HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin) == GPIO_PIN_RESET)
    {
        HAL_Delay(20);
        if (HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin) == GPIO_PIN_RESET)
        {
            led_state = !led_state;
            HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin,
                              led_state ? GPIO_PIN_SET : GPIO_PIN_RESET);

            while (HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin) == GPIO_PIN_RESET)
            {
                HAL_Delay(5);
            }
        }
    }
}
```

### 调试观察点

- 先打印或断点观察按键未按、按下时的电平。
- 如果按一次变多次，优先检查消抖和松手等待。
- 如果完全无反应，确认 GPIO 输入上下拉和实际接线。

### 常见坑

- 有效电平写反。
- 没有消抖，按一次触发多次。
- 按键悬空，输入电平随机跳动。

## [13] 光敏传感器触发LED灯

> 来源：PDF 弱对应 + 现有笔记整理

### 实验目标

把光敏模块的数字输出接到 STM32 GPIO，读取 DO 电平后控制 LED。

### 输入 / 输出关系

| 项目 | 含义 |
|------|------|
| 输入 | 光敏模块 DO 高低电平 |
| 输出 | LED 亮灭 |
| 观察 | 遮光或照光时 LED 状态变化 |

如果模块是 LM393 比较器数字输出，本质是模块内部把模拟光照变化和阈值比较后输出 DO。DO 的有效电平不写死，以实际开发板原理图和 CubeMX 配置为准。

### CubeMX 配置要点

1. 传感器 DO 引脚配置为 GPIO 输入。
2. LED 引脚配置为 GPIO 输出。
3. 根据模块输出方式选择是否开启内部上拉。
4. 如果模块还有 AO，本节只使用 DO；AO 应归到 ADC 章节。

### 最小业务逻辑

常见位置：`Core/Src/main.c`

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
while (1)
{
    GPIO_PinState sensor_state = HAL_GPIO_ReadPin(LIGHT_DO_GPIO_Port, LIGHT_DO_Pin);

    if (sensor_state == GPIO_PIN_SET)
    {
        HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_SET);
    }
    else
    {
        HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_RESET);
    }
}
```

### 调试观察点

- 先看模块电源指示灯和比较器输出指示灯。
- 调节模块电位器，让 DO 在遮光/照光时有明显翻转。
- 用串口打印 DO 状态，确认不是 LED 有效电平造成误判。

### 常见坑

- 把 AO 当 DO 接到普通 GPIO。
- LM393 模块有效电平判断反。
- 阈值电位器没有调到合适位置。

## [14] 反射传感器触发LED灯

> 来源：PDF 弱对应 + 现有笔记整理

### 实验目标

读取反射传感器数字输出，用 LED 显示是否检测到反射目标。

### 输入 / 输出关系

| 项目 | 含义 |
|------|------|
| 输入 | 反射传感器 DO 电平 |
| 输出 | LED 亮灭或翻转 |
| 观察 | 放入/移开反射物时输出变化 |

反射传感器受距离、表面颜色、环境光影响明显，有效电平以实际模块和 CubeMX 配置为准。

### CubeMX 配置要点

1. DO 接入 GPIO 输入。
2. LED 接入 GPIO 输出。
3. 若模块输出抖动明显，业务层增加连续多次确认。
4. 不确定模块输出形式时，先用万用表或串口打印确认电平。

### 最小业务逻辑

常见位置：`Core/Src/main.c`

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
while (1)
{
    GPIO_PinState reflect_state = HAL_GPIO_ReadPin(REFLECT_DO_GPIO_Port, REFLECT_DO_Pin);
    HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, reflect_state);
    HAL_Delay(10);
}
```

### 调试观察点

- 改变目标距离，观察 DO 是否翻转。
- 换黑色/白色目标，确认反射强弱对结果的影响。
- 如果输出频繁跳变，降低灵敏度或增加软件滤波。

### 常见坑

- 模块太靠近或太远导致一直同一电平。
- 环境光干扰导致误触发。
- 忘记模块和 STM32 共地。

## [15] 热敏传感器触发蜂鸣器

> 来源：PDF 弱对应 + 现有笔记整理

### 实验目标

读取热敏模块数字输出，当温度条件达到模块阈值时驱动蜂鸣器。

### 输入 / 输出关系

| 项目 | 含义 |
|------|------|
| 输入 | 热敏模块 DO 电平 |
| 输出 | 蜂鸣器打开或关闭 |
| 观察 | 温度变化或阈值调节后蜂鸣器状态变化 |

热敏 DO 只表示“超过/低于模块比较阈值”，不等于真实温度值。要读真实温度趋势，应使用 AO + ADC 或专用数字温度传感器。

### CubeMX 配置要点

1. 热敏 DO 配置为 GPIO 输入。
2. 蜂鸣器控制引脚配置为 GPIO 输出。
3. 蜂鸣器如果通过三极管或驱动模块控制，有效电平以实际开发板原理图和 CubeMX 配置为准。
4. 业务层可加短延时滤波，避免阈值附近抖动。

### 最小业务逻辑

常见位置：`Core/Src/main.c`

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
while (1)
{
    GPIO_PinState temp_alarm = HAL_GPIO_ReadPin(TEMP_DO_GPIO_Port, TEMP_DO_Pin);
    HAL_GPIO_WritePin(BEEP_GPIO_Port, BEEP_Pin, temp_alarm);
    HAL_Delay(20);
}
```

### 调试观察点

- 先单独控制蜂鸣器，确认输出有效电平。
- 再打印热敏 DO，确认阈值变化。
- 如果蜂鸣器一直响，先反转逻辑验证是否有效电平写反。

### 常见坑

- 把热敏 DO 当作温度数值。
- 蜂鸣器电流过大，直接由 GPIO 驱动不可靠。
- 温度变化慢，误以为程序没运行。

## [16] 火焰传感器触发蜂鸣器

> 来源：PDF 弱对应 + 现有笔记整理

### 实验目标

读取火焰传感器数字输出，在检测条件满足时驱动蜂鸣器报警。

### 输入 / 输出关系

| 项目 | 含义 |
|------|------|
| 输入 | 火焰传感器 DO 电平 |
| 输出 | 蜂鸣器报警 |
| 观察 | 安全测试条件下 DO 和蜂鸣器是否联动 |

安全提醒：本实验只做低风险教学测试，避免真实明火靠近易燃物。可以优先用安全光源或课程推荐方式验证，以实际开发板原理图、模块资料和视频演示为准。

### CubeMX 配置要点

1. 火焰传感器 DO 配置为 GPIO 输入。
2. 蜂鸣器引脚配置为 GPIO 输出。
3. 先确认模块供电、电平兼容和共地。
4. 不确定有效电平时，先串口打印 DO，不要直接让蜂鸣器长时间报警。

### 最小业务逻辑

常见位置：`Core/Src/main.c`

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
while (1)
{
    GPIO_PinState flame_state = HAL_GPIO_ReadPin(FLAME_DO_GPIO_Port, FLAME_DO_Pin);
    HAL_GPIO_WritePin(BEEP_GPIO_Port, BEEP_Pin, flame_state);
    HAL_Delay(20);
}
```

### 调试观察点

- 先确认蜂鸣器可以被 GPIO 单独控制。
- 再确认火焰模块 DO 是否会随测试条件变化。
- 如果室内光源导致误触发，调整阈值并记录环境条件。

### 常见坑

- 用危险明火做实验。
- 火焰模块有效电平判断反。
- 只看蜂鸣器，不先确认传感器 DO 的真实电平。

---

*课程：[9] [10] [11] [12] [13] [14] [15] [16]*

---

## 复习检查清单

- [ ] 能区分 GPIO 输出、输入、上拉、下拉、浮空和模拟模式。
- [ ] 能说明推挽输出和开漏输出的工程区别。
- [ ] 能解释 PC13 LED、普通 LED、按键和蜂鸣器的有效电平。
- [ ] 能写出点灯、流水灯、按键控制 LED 的最小逻辑。
- [ ] 能说明按键为什么需要消抖，以及延时消抖的局限。
- [ ] 能区分有源蜂鸣器和无源蜂鸣器的驱动方式。
- [ ] 能解释光敏、反射、热敏、火焰模块中 DO 输出和阈值电位器的关系。
- [ ] 能按供电、共地、GPIO 模式、有效电平、业务判断排查传感器问题。
