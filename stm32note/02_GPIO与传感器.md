# 02 GPIO 与传感器

> 覆盖课程：[9] [10] [11] [12] [13] [14] [15] [16]
>
> 目标：掌握 GPIO 输入/输出全部操作，驱动 LED、蜂鸣器，读取数字传感器。

---

## 知识地图

| 新名词 | 解释 | 哪里出现 |
|--------|------|---------|
| GPIO | General Purpose Input/Output，通用输入输出引脚 | [9] |
| 推挽输出 | 内部用一对 MOS 管轮流导通，主动输出高或低电平 | [9] |
| 开漏输出 | 内部只有一个下拉 MOS 管，只能拉低，高电平靠外部上拉电阻 | [9] |
| 上拉电阻 | 引脚到 VCC 之间的电阻（内部约 40kΩ），引脚悬空时保持高电平 | [11] |
| 下拉电阻 | 引脚到 GND 之间的电阻，引脚悬空时保持低电平 | [11] |
| 施密特触发器 | 输入端的波形整形电路，把毛刺信号变成干净的方波 | [11] |
| 按键抖动 | 机械触点闭合/断开时的弹跳，持续 5~20ms，产生毛刺信号 | [11] |
| 消抖 | 用延时或状态机过滤抖动，确保一次按下只触发一次 | [11] |
| SysTick | Cortex-M3 内核自带的 24 位定时器（非 STM32 外设），HAL 默认用它产生 1ms 时基 | [10] |
| LM393 | 双路电压比较器芯片，把模拟电压和参考电压比较后输出高低电平 | [13] |
| NPN 三极管 | 用小电流控制大电流的开关器件。B 极高电平 → C/E 导通 | [12] |

---

## GPIO HAL 函数速查表

本节涉及的所有 GPIO 相关 HAL 函数，先总览，后面用到时不再重复解释：

| 函数 | 参数 | 返回值 | 作用 |
|------|------|--------|------|
| `HAL_GPIO_WritePin(GPIOx, Pin, State)` | GPIOx=端口, Pin=引脚掩码, State=SET/RESET | void | 写引脚输出电平 |
| `HAL_GPIO_TogglePin(GPIOx, Pin)` | GPIOx=端口, Pin=引脚掩码 | void | 翻转引脚电平（高变低、低变高） |
| `HAL_GPIO_ReadPin(GPIOx, Pin)` | GPIOx=端口, Pin=引脚掩码 | GPIO_PIN_SET 或 GPIO_PIN_RESET | 读引脚输入电平 |
| `HAL_GPIO_Write(GPIOx, Value)` | GPIOx=端口, Value=16位值（整个端口一起写） | void | 一次写整个端口（16 个引脚） |
| `HAL_GPIO_Read(GPIOx)` | GPIOx=端口 | 16 位值 | 一次读整个端口 |
| `HAL_GPIO_LockPin(GPIOx, Pin)` | GPIOx=端口, Pin=引脚掩码 | HAL_StatusTypeDef | 锁定引脚配置（下次复位前不能改） |

参数类型说明：
- `GPIOx`：`GPIO_TypeDef*`，取值为 `GPIOA`、`GPIOB`、`GPIOC`...（CubeMX 生成的全局变量）
- `Pin`：`uint16_t`，位掩码宏：`GPIO_PIN_0`(0x0001) ~ `GPIO_PIN_15`(0x8000)，可用 `|` 组合多个
- `State`：`GPIO_PinState` 枚举，取值为 `GPIO_PIN_SET`(=1, 高电平) 或 `GPIO_PIN_RESET`(=0, 低电平)

---

## [9] GPIO 输出与点灯

### 关键原理

#### GPIO 8 种模式的物理含义

每个 GPIO 引脚内部有一套可配置的电路，通过设置不同寄存器位，让引脚工作在不同模式：

| 模式 | 内部电路行为 | 应用场景 |
|------|------------|---------|
| 推挽输出 | 两个 MOS 管轮流导通：输出高时 P-MOS 导通连 VCC，输出低时 N-MOS 导通连 GND。驱动电流可达 25mA。 | LED、蜂鸣器、继电器 |
| 开漏输出 | 只有 N-MOS 管，导通时拉到 GND，截止时引脚悬空（高阻态）。高电平必须靠外部上拉电阻提供。 | I2C 总线 |
| 上拉输入 | 内部约 40kΩ 电阻连到 VCC → 引脚悬空时读到高电平 | 按键接 GND |
| 下拉输入 | 内部约 40kΩ 电阻连到 GND → 引脚悬空时读到低电平 | 按键接 VCC |
| 浮空输入 | 无上拉/下拉，输入电平完全由外部电路决定。引脚悬空时电平不确定 | 仅在有外部上下拉的场合使用 |
| 模拟输入 | 关闭施密特触发器、关闭内部上下拉，引脚直连 ADC 模拟通道 | ADC 采集 |
| 复用推挽 | 输出控制权交给外设（UART/TIM/SPI），输出级是推挽 | UART TX、PWM、SPI SCK/MOSI |
| 复用开漏 | 输出控制权交给外设，输出级是开漏 | I2C SDA/SCL |

> 施密特触发器：输入通路上的迟滞比较器。输入电压从低往高走时，要超过 VIH（约 2.0V）才判为高；从高往低走时，要低于 VIL（约 0.8V）才判为低。中间有一段"不回跳"区间，能过滤输入信号上的小幅毛刺。浮空输入时，如果引脚悬空，电平恰好在 VIH 和 VIL 之间 → 读到 0 还是 1 不确定。

#### 输出速度（GPIO Speed）的物理含义

速度设置控制的是输出 MOS 管的栅极驱动电流，决定了信号翻转时电压变化的速率（slew rate）：

| 速度 | 翻转时间（近似） | 电磁干扰(EMI) | 功耗 | 适用 |
|------|---------------|-------------|------|------|
| Low | ~100ns | 小 | 低 | LED、继电器 |
| Medium | ~25ns | 中 | 中 | UART、I2C |
| High | ~10ns | 大 | 高 | SPI、PWM |

速度越高，翻转越快，但信号中的高频分量也越大，容易对外辐射干扰。能用低速解决的问题不要用高速。

#### LED 限流电阻计算

LED 是非线性器件，电压超过正向导通电压（Vf）后电流急剧增大，不限制就会烧毁。电阻的作用就是把电流限制在安全值：

```
R = (VCC - Vf) / If
  = (3.3V - 2.0V) / 0.01A    （红色 LED：Vf≈2.0V, 电流取 10mA）
  = 130Ω → 取标准值 220Ω（留安全余量）

如果 LED 亮度过高：增大电阻（如 330Ω、470Ω、1kΩ）
如果 LED 太暗：减小电阻（但不要低于计算值）
```

10mA 是保守值，普通 LED 通常 5~20mA 都能正常工作。

### HAL 函数详解

#### `HAL_GPIO_WritePin`

```c
void HAL_GPIO_WritePin(GPIO_TypeDef *GPIOx, uint16_t GPIO_Pin, GPIO_PinState PinState);
```

三个参数：
- `GPIOx`：端口选择。CubeMX 每启用一个 GPIO 端口，就生成一个对应的全局变量（`GPIOA`、`GPIOB` 等）。这些变量的类型是 `GPIO_TypeDef*`，内部包含该端口所有寄存器的地址。
- `GPIO_Pin`：引脚选择。使用宏 `GPIO_PIN_0` ~ `GPIO_PIN_15`。这些是 16 位掩码（0x0001 向左移位）。一次可以 `|` 多个引脚同时操作，比如 `GPIO_PIN_0|GPIO_PIN_3` 同时写两个。
- `PinState`：目标电平。`GPIO_PIN_SET` = 高电平（3.3V），`GPIO_PIN_RESET` = 低电平（0V）。

函数内部操作的是 GPIO 的 BSRR（Bit Set/Reset Register）寄存器，是原子操作——写高和写低在同一个寄存器完成，不会出现「读-改-写」的竞争问题。

#### `HAL_GPIO_TogglePin`

```c
void HAL_GPIO_TogglePin(GPIO_TypeDef *GPIOx, uint16_t GPIO_Pin);
```

两个参数（没有 State），因为翻转就是「当前高 → 变低，当前低 → 变高」。内部先读引脚当前电平，再反向写入。

> 调用TogglePin的引脚必须配置为输出模式（推挽输出或开漏输出）。输入模式下调用此函数无意义（改不了外部电路的电平）。

### CubeMX 配置

1. Pinout → 点目标引脚（如 PB0）→ 选 GPIO_Output
2. System Core → GPIO → 选中 PB0 → GPIO Output Level 可设初值（High/Low）

### 代码

```c
// LED 以 1Hz 闪烁
while (1)
{
    HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);  // 翻转
    HAL_Delay(500);                           // 延时 500ms
}
// 周期 = 500ms亮 + 500ms灭 = 1s，频率 = 1Hz
```

### 易错点
- LED 无串联电阻 → 电流过大会烧毁 GPIO 引脚（永久损坏，不可逆）
- 模式选错（如选成开漏输出但没有外部上拉电阻）→ 灯不亮
- `HAL_Delay()` 是阻塞延时：延时期间 CPU 原地等待，不能响应其他事件。这只是点灯的教学写法，实际工程中应尽量用定时器做定时

---

## [10] 基础延时与流水灯

### SysTick 与 HAL 延时的机制

SysTick 是 Cortex-M3 内核自带的 24 位递减定时器（不是 STM32 外设定时器），CubeMX 默认配置为 1ms 中断一次：

```
SysTick 时钟 = HCLK / 8 = 72MHz / 8 = 9MHz
重载值 = 9000 → 9000 / 9MHz = 1ms
每 1ms → SysTick 中断 → HAL_IncTick() → uwTick++

HAL_GetTick()
    └→ return uwTick;         // 返回自启动以来的毫秒数（uint32_t, 溢出周期≈49.7天)

HAL_Delay(uint32_t Delay)
    └→ 内部 while(HAL_GetTick() - start < Delay);   // 循环等待目标时刻
```

`HAL_Delay` 的参数单位是**毫秒**（不是微秒）。`HAL_Delay(500)` = 延时 500ms。

### 流水灯

```c
uint16_t leds[] = {GPIO_PIN_0, GPIO_PIN_1, GPIO_PIN_2};

for (int i = 0; i < 3; i++)
{
    HAL_GPIO_WritePin(GPIOB, leds[i], GPIO_PIN_SET);   // 亮
    HAL_Delay(200);
    HAL_GPIO_WritePin(GPIOB, leds[i], GPIO_PIN_RESET);  // 灭
}
```

### 易错点
- `HAL_GetTick()` 约 49.7 天后溢出归零（uint32_t 毫秒计满）。如果项目需要长时间运行，计时比较要用 `HAL_GetTick() - start < delay` 这种形式而不能用 `HAL_GetTick() > target`（因为溢出时会出错）

---

## [11] GPIO 输入与按键消抖

### 关键原理

#### 按键为什么需要消抖

机械按键由金属弹片组成。按下时弹片接触，但接触瞬间不是一次性稳定接触，而是在 5~20ms 内反复弹跳多次，每次弹跳都产生一次电平翻转。如果直接用这个信号触发操作，会误触发多次。

```
理想按键：    ────────┐         └────────
                      └─────────┘
实际波形：    ────────┐ ┌─┐ ┌───┐└────────
                      └─┘ └─┘   └─┘
                      ← 抖动区 →
                      (5~20ms)
```

消抖的逻辑是：第一次检测到电平变化后，等一段超过最大抖动时间（通常 20ms）的延时，再读一次确认。如果两次读到的电平一致，才认为是有效动作。

#### 上拉输入为什么推荐接 GND

上拉输入：引脚内部有一个约 40kΩ 的电阻连到 VCC。按键另一端接 GND。

- 按键**未按**：引脚通过内部电阻上拉到 VCC → 读到高电平（1）
- 按键**按下**：引脚被按键直接连到 GND → 虽然有上拉电阻但 GND 是低阻抗 → 引脚被拉到低电平（0）

推荐接 GND 的原因：板上 GND 引脚到处都是，布线方便。如果用下拉输入 + 接 VCC，VCC 引脚没有 GND 多。

### HAL 读取函数详解

#### `HAL_GPIO_ReadPin`

```c
GPIO_PinState HAL_GPIO_ReadPin(GPIO_TypeDef *GPIOx, uint16_t GPIO_Pin);
```

- 参数同 WritePin（少了 State）
- 返回值：`GPIO_PIN_SET`（读到高电平）或 `GPIO_PIN_RESET`（读到低电平）
- 内部读的是 IDR（Input Data Register）寄存器，反映的是引脚经过施密特触发器整形后的数字电平
- 可以在输入模式和输出模式下调用（输出模式读回自己写的值）

### CubeMX 配置

1. Pinout → 点 PA0 → 选 GPIO_Input
2. System Core → GPIO → PA0 → GPIO Pull-up/Pull-down → **Pull-up**

### 代码：延时消抖（最简单，会阻塞）

```c
if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)  // ↓ 检测到按下
{
    HAL_Delay(20);                                            // 等 20ms 跳过错动区
    if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET) // 再确认一次
    {
        HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);               // 执行操作

        while (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET); // 等松开（阻塞！）
    }
}
```

这个写法有两个阻塞点：`HAL_Delay(20)` 和松开等待的 `while`。如果 while(1) 里只有这一个任务，问题不大。如果还有其他任务要处理，用状态机。

### 代码：状态机消抖（不阻塞，标准写法）

```c
typedef enum { KEY_IDLE, KEY_DEBOUNCE, KEY_PRESSED } KeyState;
KeyState keyState = KEY_IDLE;
uint32_t keyTimer = 0;

// 放在 while(1) 中，每次循环跑一遍
switch (keyState)
{
    case KEY_IDLE:  // 空闲：等待按下
        if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)
        {
            keyState = KEY_DEBOUNCE;
            keyTimer = HAL_GetTick();
        }
        break;

    case KEY_DEBOUNCE:  // 消抖中：等 20ms 再确认
        if (HAL_GetTick() - keyTimer > 20)  // ⚠️ 中断里也能用 HAL_GetTick
            keyState = KEY_PRESSED;
        break;

    case KEY_PRESSED:  // 已按下：等松开
        if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_SET)  // 松开了
        {
            // ← 在松开的瞬间触发操作（防抖效果更好）
            HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);
            keyState = KEY_IDLE;
        }
        break;
}
```

状态机不阻塞：每次循环只跑一个 case，跑完立刻交还 CPU。while(1) 循环可以同时处理其他任务。

### 易错点
- 没做消抖 → 按一次触发 N 次
- 上拉/下拉和按键接线不匹配（上拉输入但按键接了 VCC → 没按时读到 0）
- `while(等待松开)` 会卡死 CPU → 多任务场景必须用状态机

---

## [12] 按键触发蜂鸣器与三极管驱动

### 关键原理

#### 有源蜂鸣器 vs 无源蜂鸣器

| | 有源蜂鸣器 | 无源蜂鸣器 |
|--|----------|----------|
| 内部结构 | 振荡电路 + 蜂鸣片 | 仅蜂鸣片（线圈+磁铁） |
| 驱动信号 | 直流高低电平即可响 | 必须 PWM（方波）才能响 |
| 频率/音调 | 固定（约 2~3kHz） | 由 PWM 频率决定，可变 |
| 如何辨别 | 通电就响（一个频率） | 通电不响，给方波才响 |
| 价格 | 略贵 | 便宜 |

有源蜂鸣器底部的封胶通常比无源的厚，因为里面有振荡芯片。另一个鉴别方法：用万用表电阻档测，有源的有极性（正反向电阻差别大），无源的电阻较小（几十 Ω）。

#### 为什么必须用三极管驱动？

STM32 GPIO 引脚输出高电平时，最大拉电流（source current）约为 20~25mA，总电流（所有引脚合计）约 150mA。蜂鸣器工作电流通常 30~80mA，远超单个 GPIO 的承受能力。芯片内部的输出 MOS 管不是用来驱动大功率负载的。

**直接驱动 = 过流烧毁芯片引脚（不可逆）。**

三极管的作用：用小电流（GPIO → 基极）控制大电流（VCC → 蜂鸣器）。NPN 三极管导通条件：基极电压 > 发射极电压 + 0.7V。

```
          VCC
           │
        蜂鸣器
           │
        C 集电极
STM32 ──1kΩ── B 基极  NPN三极管 (如 S8050)
              E 发射极
           │
          GND

计算：GPIO 输出 3.3V，基极串 1kΩ
  Ib = (3.3V - 0.7V) / 1kΩ = 2.6mA   ← GPIO 完全承受得住
  如果三极管 β=100，则 Ic_max = 260mA  ← 蜂鸣器 50mA 绰绰有余
```

基极电阻的作用：限制基极电流。不接电阻 = Vbe 被钳在 0.7V → 基极电流只受 GPIO 输出能力限制 → 可能烧 GPIO 或三极管。

### 代码

```c
if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)  // 按键按下
{
    HAL_Delay(20);                                            // 消抖
    if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)
    {
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_8, GPIO_PIN_SET);  // GPIO高 → 三极管导通 → 蜂鸣器响
        while (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET); // 按着一直响
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_8, GPIO_PIN_RESET); // 松开 → 三极管截止 → 停
    }
}
```

### 易错点
- 蜂鸣器直接接 GPIO → GPIO 引脚永久烧毁
- 三极管基极不串电阻 → 烧 GPIO 或烧三极管
- 有源蜂鸣器接 PWM → 不会变音调（它自带振荡频率），浪费定时器资源

---

## [13]~[16] 数字传感器（光敏/反射/热敏/火焰）

### 统一原理：LM393 比较器

这四类传感器模块结构完全一样，唯一的区别在传感元件：

```
传感元件（光敏电阻/热敏电阻/红外接收管/火焰传感器）
    ↓ 分压电压（模拟量，随物理量连续变化）
LM393 比较器 ← 电位器设定参考电压（阈值）
    ↓
DO = 0（触发） 或 1（未触发）
```

- LM393 是比较器芯片：同相输入 > 反相输入 → 输出 HIGH；反之输出 LOW
- 模块上的蓝色电位器调节的是参考电压，即触发阈值
- DO 脚大部分模块默认上拉到 VCC（未触发 = HIGH，触发 = LOW）。少数模块相反，实际使用前用万用表实测确认。

### 调节阈值的方法

1. 模块上电（VCC 和 GND 接好）
2. 万用表黑表笔接 GND，红表笔接 DO 脚
3. 人为制造触发条件（如遮光、靠近火源）
4. 用小螺丝刀拧蓝色电位器，直到电压刚好在触发/不触发边界翻转
5. 略偏移一点作为阈值余量

### 通用接线

```
传感器模块                STM32
  VCC  ───────────────  3.3V (或 5V，看模块标注)
  GND  ───────────────  GND     ← 必须共地
  DO   ───────────────  任意 GPIO 输入脚（上拉输入）
```

### 通用代码

```c
// 所有数字传感器通用模板
if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_1) == GPIO_PIN_RESET)  // DO=0 表示触发
{
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);      // 开LED/响蜂鸣器
}
else
{
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET);
}
```

### 传感器不触发时的排查顺序（按可能性从高到低）

1. 万用表量 VCC-GND → 供电是否正常（3.3V/5V）
2. 万用表量 DO-GND → 人为改变环境，电平是否变化（不变化 = 传感器或模块故障）
3. 确认 STM32 GND 和传感器 GND 连接（共地是最常见的遗漏）
4. 调节电位器阈值（可能是阈值太极限，始终在触发/不触发一侧）
5. 换杜邦线（接触不良外表无法判断）
6. 检查代码：引脚号、GPIO 模式

### 各传感器差异

| 传感器 | 传感原理 | 特有问题 |
|--------|---------|---------|
| 光敏 | 光敏电阻：光越强，电阻越小 | 室内外光照差异大，需在不同环境下调阈值 |
| 反射 | 红外对管：发射红外，检测反射 | 检测距离短（几 cm）；阳光下红外干扰 |
| 热敏(NTC) | 热敏电阻：温度越高，电阻越小 | DO 只能做超温报警；精确测温需 AO+ADC（见 05 章） |
| 火焰 | 检测特定波长红外光 | 白炽灯/太阳光含红外可能误触发；蓝色酒精灯不敏感 |

---

*课程：[9] [10] [11] [12] [13] [14] [15] [16]*
