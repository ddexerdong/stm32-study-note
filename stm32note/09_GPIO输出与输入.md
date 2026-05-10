# 09 GPIO 输出与输入

## GPIO 8种模式

| 模式 | 说明 | 典型使用场景 |
|------|------|------------|
| 推挽输出 | 能输出高/低电平，驱动能力强 | LED、蜂鸣器 |
| 开漏输出 | 只能拉低，高电平靠外部上拉 | I2C总线 |
| 上拉输入 | 默认高电平 | 按键（按下接GND） |
| 下拉输入 | 默认低电平 | 少用 |
| 浮空输入 | 不确定，易受干扰 | 一般不用 |
| 模拟输入 | ADC采集用 | 传感器模拟量 |
| 复用推挽 | GPIO复用为外设功能（推挽） | UART TX、SPI |
| 复用开漏 | GPIO复用为外设功能（开漏） | I2C |


## 上拉，下拉按键例子

按键一端接 GND，另一端接输入脚，输入脚开上拉，按下读到低电平。
按键一端接 VCC，另一端接输入脚，输入脚开下拉，按下读到高电平。


## 关键 HAL 函数

```c
// 写引脚
HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);    // 拉高
HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET);  // 拉低

// 读引脚
GPIO_PinState state = HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0);

// 翻转
HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);
```

## LED 点灯

```c
// 在 while(1) 中
HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);   // LED亮
HAL_Delay(500);                                         // 延时500ms
HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET);  // LED灭
HAL_Delay(500);

// 或者简写
HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);
HAL_Delay(500);
```

## 限流电阻计算

```
R = (VCC - V_LED) / I_LED
  = (3.3V - 2.0V) / 0.01A
  = 130Ω  → 取 220Ω（安全）
```

## 按键读取（[11]）

### 方法一：延时消抖

```c
if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET) {
    HAL_Delay(20);  // 消抖
    if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET) {
        // 确认按下，执行操作
        HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);
        while(HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET); // 等待松开
    }
}
```

### 方法二：状态机消抖（更优）

```c
typedef enum { KEY_IDLE, KEY_DEBOUNCE, KEY_PRESSED } KeyState;
KeyState keyState = KEY_IDLE;
uint32_t keyTimer = 0;

// 在 while(1) 中
switch(keyState) {
    case KEY_IDLE:
        if(HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)
            keyState = KEY_DEBOUNCE, keyTimer = HAL_GetTick();
        break;
    case KEY_DEBOUNCE:
        if(HAL_GetTick() - keyTimer > 20) keyState = KEY_PRESSED;
        break;
    case KEY_PRESSED:
        if(HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_SET) {
            // 松开时触发
            HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);
            keyState = KEY_IDLE;
        }
        break;
}
```

## 流水灯（[10]）

```c
uint16_t leds[] = {GPIO_PIN_0, GPIO_PIN_1, GPIO_PIN_2};
for(int i = 0; i < 3; i++) {
    HAL_GPIO_WritePin(GPIOB, leds[i], GPIO_PIN_SET);
    HAL_Delay(200);
    HAL_GPIO_WritePin(GPIOB, leds[i], GPIO_PIN_RESET);
}
```

## 踩坑记录

<!-- 实验中遇到的问题 -->
- [ ] 

---
*课程：[9] [10] [11]*
