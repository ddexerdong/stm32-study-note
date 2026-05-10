# 二、GPIO 与传感器基础篇

> 覆盖课程：[9] [10] [11] [12] [13] [14] [15] [16]
>
> 目标：掌握 GPIO 输入/输出的所有基本操作，能驱动 LED、蜂鸣器、读取各类数字传感器。

---

## [9] GPIO 输出与点灯

### 知识点
- GPIO 是什么：通用输入输出引脚，STM32 最基础的外设
- 引脚命名：`PB3` = Port B 第 3 脚
- 8 种 GPIO 工作模式
- GPIO 输出速度（Low / Medium / High）

### 关键原理

#### 8 种模式速查

| 模式 | 说明 | 典型场景 |
|------|------|---------|
| 推挽输出 | 能主动输出高/低电平 | LED、蜂鸣器、继电器 |
| 开漏输出 | 只能拉低，高电平靠外部上拉 | I2C 总线 |
| 上拉输入 | 内部上拉到 VCC，默认高电平 | 按键（按下接 GND）⭐ |
| 下拉输入 | 内部下拉到 GND，默认低电平 | 按键（按下接 VCC） |
| 浮空输入 | 无上下拉，电平不确定 | 慎用 |
| 模拟输入 | 关闭数字电路，直连 ADC | 传感器模拟量采集 |
| 复用推挽 | GPIO 交给外设，推挽输出 | UART TX、SPI、PWM |
| 复用开漏 | GPIO 交给外设，开漏输出 | I2C |

#### 输出速度

速度控制的是信号翻转的斜率（slew rate），不是数据速率：

| 速度 | 适用 |
|------|------|
| Low | LED、继电器（省电、干扰小） |
| Medium | UART、I2C |
| High | SPI、PWM 高速信号 |

#### 限流电阻计算

LED 必须串电阻，否则烧引脚：

```
R = (VCC - V_LED) / I_LED
  = (3.3V - 2.0V) / 0.01A
  = 130Ω → 取 220Ω（留余量）
```

### CubeMX 配置步骤

1. Pinout → 点目标引脚（如 PB0）→ 选 GPIO_Output
2. System Core → GPIO → 选中该引脚 → 可改 User Label（取个名字比如 LED1）
3. 生成代码

### HAL 库代码模板

```c
// 点灯（while(1) 中）
HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);   // 亮
HAL_Delay(500);
HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET);  // 灭
HAL_Delay(500);

// 翻转简写
HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);
HAL_Delay(500);
```

### 易错点
- LED 不串限流电阻，电流过大烧引脚或 LED
- GPIO 模式选错（比如该用推挽输出却选了开漏）→ 灯不亮
- `HAL_Delay()` 是阻塞延时，延时期间 CPU 死等，后续项目尽量用定时器替代

---

## [10] 基础延时与三个流水灯

### 知识点
- `HAL_Delay()` 的工作原理（依赖 SysTick）
- `HAL_GetTick()` 获取系统开机毫秒数
- 流水灯循环控制

### 关键原理

CubeMX 自动配置 SysTick 定时器每 1ms 中断一次，`HAL_Delay()` 和 `HAL_GetTick()` 都依赖它：

```
SysTick 中断（1ms 一次）→ uwTick++ → HAL_GetTick() 返回 uwTick
                                     → HAL_Delay(n) 循环等 uwTick 走到目标值
```

### HAL 库代码模板

```c
// 流水灯
uint16_t leds[] = {GPIO_PIN_0, GPIO_PIN_1, GPIO_PIN_2};

for (int i = 0; i < 3; i++)
{
    HAL_GPIO_WritePin(GPIOB, leds[i], GPIO_PIN_SET);
    HAL_Delay(200);
    HAL_GPIO_WritePin(GPIOB, leds[i], GPIO_PIN_RESET);
}
```

### 易错点
- 流水灯用 `HAL_Delay()` 会导致 CPU 不能做别的事，只适合教学演示
- `HAL_GetTick()` 返回值约 49 天后溢出归零（32 位毫秒），长时间运行的项目要注意

---

## [11] GPIO 输入与按键开关灯

### 知识点
- 上拉输入 / 下拉输入的选择
- 按键抖动原理与消抖方法
- 延时消抖 vs 状态机消抖

### 关键原理

#### 按键为什么需要消抖

机械按键按下时，金属弹片会反复弹跳 5~20ms，产生一连串毛刺信号。不消抖 = 按一次触发好几次。

#### 上拉 vs 下拉的选择

```
上拉输入 + 按键接 GND（推荐）：
  没按 = 读到 1（上拉电阻保持高电平）
  按下 = 读到 0（GND 把引脚拉低）

下拉输入 + 按键接 VCC：
  没按 = 读到 0
  按下 = 读到 1
```

> 推荐用上拉 + 接 GND，因为 GND 满板都是，布线方便。

### CubeMX 配置步骤

1. Pinout → 点目标引脚（如 PA0）→ 选 GPIO_Input
2. System Core → GPIO → 该引脚 → GPIO Pull-up/Pull-down → Pull-up
3. 生成代码

### HAL 库代码模板

**方法一：延时消抖（简单，会阻塞）**

```c
if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)  // 按下
{
    HAL_Delay(20);  // 等 20ms 跳过抖动
    if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)  // 确认
    {
        HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);  // 翻转 LED
        while (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET);  // 等松开
    }
}
```

**方法二：状态机消抖（不阻塞，推荐）**

```c
typedef enum { KEY_IDLE, KEY_DEBOUNCE, KEY_PRESSED } KeyState;
KeyState keyState = KEY_IDLE;
uint32_t keyTimer = 0;

// while(1) 中
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
        if (HAL_GetTick() - keyTimer > 20)  // 等 20ms
            keyState = KEY_PRESSED;
        break;

    case KEY_PRESSED:
        if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_SET)  // 松开时触发
        {
            HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);
            keyState = KEY_IDLE;
        }
        break;
}
```

### 易错点
- 没做消抖 → 按一次触发多次
- 上拉/下拉选反了 → 没按的时候读到的是 0，逻辑全错
- `while(等待松开)` 卡死 CPU → 如果需要同时做别的事，用状态机

---

## [12] 按键触发蜂鸣器

### 知识点
- 有源蜂鸣器 vs 无源蜂鸣器
- 三极管开关电路
- GPIO 驱动能力限制

### 关键原理

#### 有源 vs 无源

| | 有源 | 无源 |
|--|------|------|
| 内部 | 自带振荡电路 | 纯线圈 |
| 驱动 | GPIO 高/低电平即可 | 必须 PWM |
| 音调 | 固定 | 可变 |

#### 为什么必须用三极管？

```
STM32 GPIO 最大输出电流 ≈ 20mA
蜂鸣器工作电流 ≈ 30~80mA

→ GPIO 直驱可能永久损坏引脚！

三极管开关电路：
  STM32 GPIO → 1kΩ 电阻 → 三极管 B 极（基极）
                            三极管 C 极 → 蜂鸣器 → VCC
                            三极管 E 极 → GND

  GPIO 高电平 = 三极管导通 = 蜂鸣器响
  GPIO 低电平 = 三极管截止 = 蜂鸣器停
```

### HAL 库代码模板

```c
// 按键触发蜂鸣器（在 while(1) 中）
if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)
{
    HAL_Delay(20);  // 消抖
    if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)
    {
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_8, GPIO_PIN_SET);  // 蜂鸣器响
        while (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET);
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_8, GPIO_PIN_RESET);  // 蜂鸣器停
    }
}
```

### 易错点
- 蜂鸣器直接接 GPIO → 烧引脚
- 三极管基极没串 1kΩ 限流电阻 → 烧三极管
- 有源蜂鸣器接了 PWM → 音调不会变（它自带振荡电路），纯属浪费定时器资源

---

## [13] 光敏传感器触发 LED 灯

### 知识点
- 光敏传感器原理：光照越强，电阻越小
- LM393 比较器：把模拟信号转成数字开关信号
- DO 数字输出读取

### 关键原理

```
传感器模拟信号（光敏电阻分压）
        ↓
  LM393 比较器 ← 电位器设定参考电压（阈值）
        ↓
  DO 数字输出（高/低电平）
```

- 光线暗到阈值以下 → DO 拉低
- 光线亮 → DO 保持高电平

### CubeMX 配置步骤

DO 脚接任意 GPIO 输入脚（上拉输入模式），其余同 [11]。

### HAL 库代码模板

```c
// 光敏传感器触发 LED
if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_1) == GPIO_PIN_RESET)  // 光线暗，DO 拉低
{
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);  // 开灯
}
else
{
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET);  // 关灯
}
```

### 易错点
- 传感器和 STM32 没共地 → DO 信号无参考，读到随机值
- 电位器阈值没调 → 一直触发或一直不触发（用小螺丝刀拧模块上的蓝色电位器）
- 不同模块 DO 有效电平可能相反 → 先拿万用表量，确认逻辑再写代码

---

## [14] 反射传感器触发 LED 灯

### 知识点
- 反射传感器原理：红外发射 + 红外接收，检测反射光
- 和光敏传感器的区别：光敏是被动检测环境光，反射是主动发射红外 + 检测反射

### 关键原理

```
红外 LED 发射红外光
      ↓ 遇到障碍物反射
红外接收管收到反射光 → 输出变化
```

本质和 [13] 一样，都是 LM393 比较器输出 DO 信号，代码通用。

### 易错点
- 检测距离有限（通常几毫米到几厘米），远距离检测用超声波或激光测距
- 环境红外干扰（阳光）可能导致误触发

---

## [15] 热敏传感器触发蜂鸣器

### 知识点
- 热敏传感器（NTC）原理：温度越高，电阻越小
- DO 数字输出和 AO 模拟输出两种用法
- DO 用于温度阈值报警（本章），AO 用于精确测温（见 [21] ADC 章）

### 关键原理

```
DO 用法（本章）：
  温度超过阈值 → DO 拉低 → 蜂鸣器报警
  和 [13] 光敏一样，只是传感器不同

AO 用法（见 ADC 章）：
  输出电压随温度连续变化 → ADC 读取 → 算出具体温度值
```

### HAL 库代码模板

```c
// 热敏传感器温度报警
if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_2) == GPIO_PIN_RESET)  // 超温
{
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_8, GPIO_PIN_SET);  // 蜂鸣器响
}
else
{
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_8, GPIO_PIN_RESET);  // 蜂鸣器停
}
```

### 易错点
- DO 阈值和 AO 温度值是两码事：DO 只能告诉你"过线了没"，想知道几度必须用 AO + ADC
- 电位器调阈值时用热水/冷水辅助标定

---

## [16] 火焰传感器触发蜂鸣器

### 知识点
- 火焰传感器原理：检测火焰发出的特定波长红外光
- 应用场景：火灾报警

### 关键原理

火焰传感器检测特定波长的红外线（火焰特征波段），同样是 LM393 比较器输出 DO。代码和前面几节完全通用。

### 通用接线（数字传感器统一接法）

```
传感器模块            STM32
  VCC  ────────────  3.3V
  GND  ────────────  GND（必须共地！）
  DO   ────────────  任意 GPIO 输入脚
```

### 易错点
- 火焰传感器对打火机火焰响应快，但对酒精灯（蓝色火焰）可能不敏感
- 室内强光源（白炽灯、太阳光含红外）可能误触发
- 所有数字传感器排查顺序：供电 → 共地 → DO 电平 → 电位器阈值 → 代码逻辑

---

*课程：[9] [10] [11] [12] [13] [14] [15] [16]*
