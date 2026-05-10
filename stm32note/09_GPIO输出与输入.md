# 09 GPIO 输出与输入

> 本章学完你应该能：点灯、读按键、理解 GPIO 的 8 种模式分别在什么时候用。

---

## GPIO 是什么

STM32 的引脚大部分都可以配置为 **GPIO（General Purpose Input/Output，通用输入输出）**。每个引脚可以独立设置方向（输入/输出）、模式、速度。

引脚的命名规则：`PB3` = **P**ort **B** 的第 **3** 脚。查板子原理图/丝印可以看到物理引脚号和 GPIO 名称的对应关系。

---

## GPIO 8 种模式

| 模式 | 说明 | 典型场景 |
|------|------|---------|
| 推挽输出 | 能主动输出高/低电平，驱动能力强 | LED、蜂鸣器、继电器 |
| 开漏输出 | 只能拉低，高电平靠外部上拉电阻 | I2C 总线、电平转换 |
| 上拉输入 | 内部上拉电阻启用，默认读到高电平 | 按键（按下接 GND）⭐最常用 |
| 下拉输入 | 内部下拉电阻启用，默认读到低电平 | 按键（按下接 VCC） |
| 浮空输入 | 引脚悬空时电平不确定，易受干扰 | ADC 输入（设为模拟模式更好） |
| 模拟输入 | 关闭数字输入电路，用于 ADC 采集 | 传感器模拟量读取 |
| 复用推挽 | GPIO 交给外设（UART/SPI/TIM）控制，推挽输出 | UART TX、SPI SCK/MOSI、PWM |
| 复用开漏 | GPIO 交给外设控制，开漏输出 | I2C SDA/SCL |

### 输出速度（GPIO Speed）

CubeMX 里会让选 Low / Medium / High。这不是数据传输速率，是信号翻转的斜率：

| 速度 | 适用场景 |
|------|---------|
| Low | LED、继电器等低速信号（省电、干扰小） |
| Medium | 普通外设（UART、I2C） |
| High | 高速信号（SPI、PWM） |

---

## 上拉 / 下拉 —— 一图理解

```
上拉：按键一端接 GND，另一端接输入脚，输入脚开启内部上拉。
     没按下 = 读到高电平（上拉电阻拉到 VCC）
     按下   = 读到低电平（按键直接连到 GND）

下拉：按键一端接 VCC，另一端接输入脚，输入脚开启内部下拉。
     没按下 = 读到低电平（下拉电阻拉到 GND）
     按下   = 读到高电平（按键直接连到 VCC）
```

> 💡 绝大多数场景用**上拉 + 按键接 GND**，因为 GND 满板都是，布线方便。

---

## 关键 HAL 函数

```c
// 写引脚
HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);    // 输出高电平
HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET);  // 输出低电平

// 读引脚
GPIO_PinState state = HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0);

// 翻转（高变低、低变高）
HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);
```

---

## LED 点灯

### 限流电阻计算

LED 不能直接接 3.3V——会烧。必须串一个限流电阻。

```
R = (VCC - V_LED) / I_LED
  = (3.3V - 2.0V) / 0.01A
  = 130Ω  →  取 220Ω（留余量，亮度也够用）
```

### 点灯代码

```c
// 在 while(1) 中
while (1)
{
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);   // LED 亮
    HAL_Delay(500);                                         // 延时 500ms
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET);  // LED 灭
    HAL_Delay(500);

    // 或者用翻转简写
    // HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);
    // HAL_Delay(500);
}
```

> ⚠️ `HAL_Delay()` 是**阻塞延时**——延时期间 CPU 原地等待，什么都不能干。
> 这个阶段点灯没问题，但后续学到中断/多任务时要改用定时器来做定时，不要在所有项目里都用 `HAL_Delay`。

---

## 流水灯

```c
uint16_t leds[] = {GPIO_PIN_0, GPIO_PIN_1, GPIO_PIN_2};

for (int i = 0; i < 3; i++)
{
    HAL_GPIO_WritePin(GPIOB, leds[i], GPIO_PIN_SET);
    HAL_Delay(200);
    HAL_GPIO_WritePin(GPIOB, leds[i], GPIO_PIN_RESET);
}
```

---

## 按键读取 —— 消抖是关键

机械按键按下和释放时有**几毫秒到几十毫秒的电平抖动**，如果直接读，会误判成多次按下。必须消抖。

### SysTick 是什么（读之前先弄懂它）

CubeMX 自动配置了 SysTick 定时器，每 1ms 产生一次中断。两个函数都靠它：

| 函数 | 作用 |
|------|------|
| `HAL_GetTick()` | 返回系统启动以来的毫秒数 |
| `HAL_Delay(n)` | 死等 n 毫秒（内部循环调用 `HAL_GetTick()`） |

> 🔴 记住：`HAL_Delay` 不能在中断回调里用！中断里只能用 `HAL_GetTick()` 做时间判断。

### 方法一：延时消抖（简单，适合初学）

```c
if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)  // 检测按下
{
    HAL_Delay(20);  // 等 20ms 跳过抖动
    if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)  // 再次确认
    {
        HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);  // 执行操作
        while (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET);  // 等待松开
    }
}
```

> 缺点：`HAL_Delay(20)` 和 `while(等待松开)` 都会阻塞 CPU。

### 方法二：状态机消抖（推荐，不阻塞）

```c
typedef enum { KEY_IDLE, KEY_DEBOUNCE, KEY_PRESSED } KeyState;
KeyState keyState = KEY_IDLE;
uint32_t keyTimer = 0;

// 在 while(1) 中
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
        if (HAL_GetTick() - keyTimer > 20)  // 等待 20ms
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

> 💡 状态机方式的优势：while(1) 循环不会被卡住，可以同时处理多个按键和其他任务。

---

## 踩坑记录

- [ ] 忘了配 GPIO 模式（输出/输入），读按键读到的是随机值
- [ ] 按键没加消抖，按一下触发了好几次
- [ ] LED 没串限流电阻直接烧了
- [ ] 用 `HAL_Delay` 导致按键响应迟钝（阻塞太久）

---

*课程：[9] [10] [11]*
