# 02 GPIO 与传感器

> 覆盖课程：[9] [10] [11] [12] [13] [14] [15] [16]
>
> 目标：掌握 GPIO 输入/输出、基础延时、按键消抖、蜂鸣器驱动和数字传感器读取。

---

## 概念

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

```c
void HAL_GPIO_WritePin(GPIO_TypeDef *GPIOx,
                       uint16_t GPIO_Pin,
                       GPIO_PinState PinState);
```

作用：写指定引脚输出电平。内部主要通过 `BSRR` 寄存器完成置位/复位，避免普通读-改-写带来的竞争问题。

```c
HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);
HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET);
```

### `HAL_GPIO_TogglePin`

```c
void HAL_GPIO_TogglePin(GPIO_TypeDef *GPIOx, uint16_t GPIO_Pin);
```

作用：翻转输出电平。

```c
HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);
```

注意：引脚必须配置为输出模式。输入模式下调用它不会改变外部电路电平。

### `HAL_GPIO_ReadPin`

```c
GPIO_PinState HAL_GPIO_ReadPin(GPIO_TypeDef *GPIOx, uint16_t GPIO_Pin);
```

作用：读取引脚数字电平，返回 `GPIO_PIN_SET` 或 `GPIO_PIN_RESET`。

```c
if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)
{
    // 上拉输入下，低电平通常表示按键按下
}
```

---

## 示例代码

### LED 以 1 Hz 闪烁

```c
while (1)
{
    HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);
    HAL_Delay(500);
}
```

周期 = 500 ms 亮 + 500 ms 灭 = 1 s。

### 流水灯

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

```c
while (1)
{
    Key_Task();
    // 其他任务也可以继续执行
}
```

### 按键控制蜂鸣器

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

*课程：[9] [10] [11] [12] [13] [14] [15] [16]*
