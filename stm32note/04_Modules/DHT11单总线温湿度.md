---
type: module
status: refactor
course: stm32-f103-hal
source:
  - local-notes
  - module-datasheet
tags:
  - stm32
  - dht11
  - one-wire
---
# DHT11 单总线温湿度

> 模块定位：学习严格 GPIO 时序、方向切换、超时和校验。
>
> 原始来源：[[10_DWT与单总线传感器]]。
---
## 本节目标

- 能画出 DHT11 起始、响应、40 bit 数据和校验流程。
- 能用 GPIO 方向切换与微秒计时实现带超时的读取框架。
- 能区分通信失败、位判定失败和校验失败。

## 核心概念

### 知识地图

| 名词 | 工程含义 |
|---|---|
| 起始信号 | MCU 请求传感器开始发送 |
| 响应信号 | DHT11 对请求的时序回应 |
| 40 bit 帧 | 湿度/温度四字节 + 校验和 |
| 位判决 | 按高电平持续时间区分 0/1 |
| 超时 | 线路异常时退出等待 |

### 模块作用

通过一根双向数据线读取 40 bit 温湿度帧，练习微秒时序、GPIO 方向切换、超时和校验。

### 通信流程

```text
MCU 输出起始信号
-> 释放数据线并切输入
-> 等待传感器响应
-> 读取 40 bit
-> 按脉宽判断 0/1
-> 校验前 4 字节与校验和
-> 输出温湿度
```

### 工程重点

- 数据线需要稳定上拉。
- GPIO 在输出起始信号后要切回输入。
- 所有等待电平变化的循环都必须有超时。
- 先验证 5 个原始字节和校验，再处理显示。

> 视频待核对：数据引脚、上拉方式、起始/响应精确时序、读位阈值、采样间隔和课程驱动封装。

不能凭空补完整实验。

## 本质理解

DHT11 通信的核心是 GPIO 方向切换和脉宽判位，不是调用某个标准总线外设。一次读取只有在响应、40 bit 数据和校验和都成立时才有效，任何等待电平的环节都必须能超时退出。

## 硬件连接关系

- 数据线接可切换输入/输出的 GPIO
- 按模块要求配置上拉
- 供电和逻辑电平与 STM32 兼容
- 线长和干扰会影响边沿

## 通信 / 控制链路

```text
检查空闲高电平
-> 发送起始信号
-> 等待响应并超时
-> 循环读取 40 bit
-> 验证校验和
-> 再换算和显示
```

## CubeMX 配置要点

1. GPIO 初始保持释放/高电平状态
2. 不配置硬件 UART/I2C
3. DWT 不需 CubeMX，但需正确 SystemCoreClock
4. 串口/OLED仅作为结果观察

## 本模块依赖的 HAL 层

> DHT11 不是 STM32 的标准 HAL 外设。HAL 只负责 GPIO 配置和电平读写；微秒计时通常使用 TIM 或 CMSIS/DWT。`DWT->CYCCNT` 不是 HAL API，而是 Cortex-M 内核调试组件寄存器。

| 模块动作 | 依赖接口 | 说明 |
|---|---|---|
| 切换单总线输出/输入状态 | `HAL_GPIO_Init()` | 修改 `GPIO_InitTypeDef.Mode` 后重新初始化同一引脚；GPIO 时钟必须已使能 |
| 输出起始电平 | `HAL_GPIO_WritePin()` | 只负责输出 SET/RESET，具体持续时间必须由模块手册和实测确认 |
| 读取响应和数据位 | `HAL_GPIO_ReadPin()` | 返回当前输入状态；位宽判断仍由驱动状态机完成 |
| 整帧超时与读取间隔 | `HAL_GetTick()` | 适合毫秒级保护，不适合判定单个位的微秒脉宽 |
| 微秒计时 | TIM 或 `DWT->CYCCNT` | 这部分不是标准 DHT11 HAL API；DWT 属于 CMSIS/内核组件 |
| 打印原始 5 字节/校验结果 | `HAL_UART_Transmit()` 或 `printf` | 仅用于调试，避免在严格时序窗口内打印 |

HAL API 的返回值和 GPIO 模式切换要检查，但 40 bit 帧格式、位判定阈值、校验和与最小采样间隔仍以模块手册和视频实测为准。

### 典型调用顺序

```text
GPIO 配置为输出
-> HAL_GPIO_WritePin() 产生起始状态
-> GPIO 改为输入
-> 用 TIM/DWT 带超时测量响应和 40 bit 脉宽
-> HAL_GPIO_ReadPin() 读取电平
-> 校验 5 字节
-> 主循环输出结果
```

### 什么时候用哪个接口？

| 场景 | 推荐接口 |
|---|---|
| GPIO 方向切换 | `HAL_GPIO_Init()` |
| 普通电平输出/读取 | `HAL_GPIO_WritePin()` / `HAL_GPIO_ReadPin()` |
| 毫秒级整帧超时 | `HAL_GetTick()` |
| 微秒脉宽判定 | TIM 输入捕获/计数，或经验证的 CMSIS/DWT 实现 |

## Bring-up 顺序

1. 确认供电、共地、数据线上拉和空闲电平。
2. 只发送起始信号并观察传感器是否产生响应脉冲。
3. 验证 GPIO 输出到输入的切换位置，以及每个等待循环的超时。
4. 逐位记录 40 bit 原始结果，暂不解释温湿度。
5. 拼成 5 个字节并验证校验和。
6. 数据连续稳定后再接入显示和周期采集任务。

## 最小驱动框架

> 代码性质：示例框架，用于理解调用顺序，不能保证直接编译。
> 验证状态：未上板验证，需按 CubeMX 变量名、开发板原理图和课程源码核对。

```c
static void DHT11_PinOutput(void)
{
    GPIO_InitTypeDef init = {0};
    init.Pin = DHT11_Pin;
    init.Mode = GPIO_MODE_OUTPUT_PP;
    init.Speed = GPIO_SPEED_FREQ_HIGH;
    HAL_GPIO_Init(DHT11_GPIO_Port, &init);
}

static void DHT11_PinInput(void)
{
    GPIO_InitTypeDef init = {0};
    init.Pin = DHT11_Pin;
    init.Mode = GPIO_MODE_INPUT;
    init.Pull = DHT11_INPUT_PULL;
    HAL_GPIO_Init(DHT11_GPIO_Port, &init);
}

static uint8_t DHT11_WaitLevel(GPIO_PinState level, uint32_t timeout_cycles)
{
    uint32_t start = DWT->CYCCNT;
    while (HAL_GPIO_ReadPin(DHT11_GPIO_Port, DHT11_Pin) != level)
    {
        if ((uint32_t)(DWT->CYCCNT - start) >= timeout_cycles) return 0U;
    }
    return 1U;
}

static uint8_t DHT11_StartSignal(void)
{
    DHT11_PinOutput();
    HAL_GPIO_WritePin(DHT11_GPIO_Port, DHT11_Pin, GPIO_PIN_RESET);
    DWT_Delay_us(DHT11_START_LOW_US);
    HAL_GPIO_WritePin(DHT11_GPIO_Port, DHT11_Pin, GPIO_PIN_SET);
    DHT11_PinInput();
    return DHT11_WaitLevel(GPIO_PIN_RESET, DHT11_RESPONSE_TIMEOUT_CYCLES);
}

static uint8_t DHT11_ReadBit(uint8_t *bit)
{
    if (!DHT11_WaitLevel(GPIO_PIN_SET, DHT11_BIT_TIMEOUT_CYCLES)) return 0U;
    uint32_t high_start = DWT->CYCCNT;
    if (!DHT11_WaitLevel(GPIO_PIN_RESET, DHT11_BIT_TIMEOUT_CYCLES)) return 0U;
    uint32_t high_cycles = (uint32_t)(DWT->CYCCNT - high_start);
    *bit = high_cycles > DHT11_ONE_THRESHOLD_CYCLES;
    return 1U;
}

uint8_t DHT11_Read(DHT11_Data_t *out)
{
    uint8_t raw[5] = {0};

    if ((out == NULL) || !DHT11_StartSignal()) return 0U;
    for (uint32_t i = 0U; i < DHT11_FRAME_BITS; ++i)
    {
        uint8_t bit;
        if (!DHT11_ReadBit(&bit)) return 0U;
        raw[i / 8U] = (uint8_t)((raw[i / 8U] << 1U) | bit);
    }

    uint8_t checksum = (uint8_t)(raw[0] + raw[1] + raw[2] + raw[3]);
    if (checksum != raw[4]) return 0U;
    return DHT11_Decode(raw, out);
}
```

`DHT11_START_LOW_US`、响应/bit 超时和 `DHT11_ONE_THRESHOLD_CYCLES` 必须按模块手册、系统时钟、逻辑分析仪和视频实测确定。DWT/CYCCNT 是 CMSIS/内核组件，不是 HAL API。

## 调试方法

```text
确认上拉与空闲高电平
-> 示波器观察起始和响应
-> 检查 GPIO 输入/输出切换
-> 定位等待响应或读 bit 超时
-> 打印 5 个原始字节
-> 检查校验和
-> 最后检查采样间隔和显示
```

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| 无响应 | 起始时序/方向/上拉错误 | 看数据线波形 |
| 程序卡死 | 等待循环无超时 | 每阶段加入超时 |
| 校验失败 | 位阈值或中断干扰 | 记录 5 字节和脉宽 |
| 偶发错误 | 读取间隔过短/线长干扰 | 按手册限制采样频率 |
| 温湿度乱码 | 未先验证 raw | 先打印原始字节 |

## 待核对项

- [ ] 起始、响应和 bit 精确时序
- [ ] 读位阈值、上拉和采样间隔
- [ ] 课程 GPIO 封装与 DWT 延时实现

以实际开发板原理图、CubeMX 配置、视频课程源码和模块手册为准。

## 与其它章节的关系

- STM32 侧依赖：[[GPIO与AFIO]]、[[DWT与调试计时]]、[[OLED显示模块]]。
- 通用 bring-up 和数据显示见 [[传感器采集与显示]]。
- 跨外设排错见 [[通用调试方法]]。

## 复习检查清单

- [ ] 能画出主机起始、传感器响应、40 bit 数据和校验和流程。
- [ ] 能说明数据线为什么要在输出起始信号后切换为输入。
- [ ] 能给所有等待电平变化的循环设置超时退出条件。
- [ ] 能根据高电平宽度判定位值，但不写死未核对阈值。
- [ ] 能保存 5 个原始字节并在使用温湿度前验证校验和。
- [ ] 能区分无响应、位判定错误和校验失败三类问题。
- [ ] 能指出上拉、精确时序、读位阈值和采样间隔必须按视频/手册实测。
