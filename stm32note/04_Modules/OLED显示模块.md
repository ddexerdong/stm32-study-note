---
type: module
status: refactor
course: stm32-f103-hal
source:
  - local-notes
  - module-datasheet
tags:
  - stm32
  - oled
  - display
  - i2c
---
# OLED 显示模块

> 模块定位：在 [[I2C总线]] 之上理解命令、数据、显存、字库和位图。
>
> 原始来源：[[11_I2C与OLED]]。
---
## 本节目标

- 能区分 OLED 命令、数据、显存、字库和位图。
- 能按初始化、清屏、字符、字符串和位图逐层 bring-up。
- 能定位 I2C、控制器参数和取模格式三类问题。

## 核心概念

### 知识地图

| 名词 | 工程含义 |
|---|---|
| 控制器 | 解释命令并管理 GDDRAM |
| 控制字节 | 区分命令和显示数据 |
| 页地址 | 常见按若干像素高组织显存 |
| 字库 | 字符到点阵字节的映射 |
| Framebuffer | MCU 侧显示缓冲区 |

### 模块作用

通过 I2C 向显示控制器写命令和显存数据，实现清屏、字符、字符串和位图显示。

### 驱动分层

```text
I2C 字节发送
-> 写命令 / 写数据
-> 设置页和列
-> 清屏
-> 画点 / 字符 / 字符串
-> 位图和业务数据显示
```

### 显示机制

- 初始化命令配置控制器工作模式。
- 清屏是向显示 RAM 写入空像素。
- 字符显示是查字库后写点阵字节。
- 位图显示需要与页寻址和取模方向一致。

## 本质理解

OLED 显示不是把字符串直接发到 I2C：命令负责配置控制器，显示数据负责更新显存。字符和位图能否正确显示，还取决于字库、取模方向以及页列组织是否与控制器一致。

## 硬件连接关系

- VCC/GND 按模块额定电压
- SCL/SDA 接对应 I2C 并有上拉
- 地址由模块硬件和控制器决定
- 分辨率/控制器不能凭外观猜

## 通信 / 控制链路

```text
地址扫描
-> 实现 WriteCommand/Data
-> 发送初始化序列
-> 清屏
-> 显示单字符
-> 字符串
-> 小位图
-> 业务数据
```

## CubeMX 配置要点

1. 启用 I2C Master
2. 确认 SCL/SDA 开漏与上拉
3. 先使用阻塞发送验证
4. 配置串口用于打印 HAL 返回值

## 本模块依赖的 HAL 层

| 模块动作 | 依赖 HAL / C 接口 | 说明 |
|---|---|---|
| 探测 OLED 地址是否 ACK | `HAL_I2C_IsDeviceReady()` | 只能证明该地址有设备响应，不能证明控制器型号和初始化序列正确 |
| 发送控制字节、命令或显存数据 | `HAL_I2C_Master_Transmit()` | 常见驱动把控制字节与命令/数据组合成字节流；每次检查 `HAL_OK` |
| 使用寄存器式封装 | `HAL_I2C_Mem_Write()` | 仅当驱动明确把控制字节映射为 MemAddress 时使用，不能默认所有 OLED 都适合 |
| 查询 I2C 错误 | `HAL_I2C_GetState()` / `HAL_I2C_GetError()` | 用于区分 BUSY、NACK、超时等软件状态，仍需量 SCL/SDA |
| 格式化显示文本 | `snprintf()` | C 库函数，不是 HAL API；注意目标 buffer 容量 |

HAL 只负责 I2C 字节传输。OLED 地址、控制字节、初始化命令、页/列组织、字库和位图取模格式仍以实际控制器手册、模块板和课程源码为准。

### 典型调用顺序

```text
MX_I2Cx_Init()
-> HAL_I2C_IsDeviceReady()
-> 发送初始化命令序列
-> 清显示缓冲区
-> HAL_I2C_Master_Transmit() 刷新命令/数据
-> 检查返回值，失败时读取 I2C state/error
```

### 什么时候用哪个函数？

| 场景 | 推荐函数 |
|---|---|
| 先确认地址是否响应 | `HAL_I2C_IsDeviceReady()` |
| 发送命令/数据字节流 | `HAL_I2C_Master_Transmit()` |
| 已验证的寄存器式驱动 | `HAL_I2C_Mem_Write()` |
| 显示数字/字符串前格式化 | `snprintf()`，随后交给 OLED 字符绘制函数 |

## Bring-up 顺序

1. 确认供电、接口类型、I2C 上拉和地址 ACK。
2. 按已确认控制器资料发送初始化命令并检查 HAL 返回值。
3. 写入全空显存验证清屏。
4. 显示一个固定字符，核对字库尺寸和页列方向。
5. 扩展到字符串，再用测试位图核对取模格式。
6. 最后接入传感器数据和刷新策略。

## 最小驱动框架

> 代码性质：示例框架，用于理解调用顺序，不能保证直接编译。

```c
uint8_t OLED_WriteCommand(uint8_t cmd)
{
    uint8_t frame[2] = {OLED_CMD_CONTROL, cmd};
    return HAL_I2C_Master_Transmit(&hi2c1, OLED_ADDR_HAL, frame,
                                   sizeof(frame), OLED_I2C_TIMEOUT_MS) == HAL_OK;
}

static uint8_t OLED_WriteData(const uint8_t *data, uint16_t length)
{
    uint8_t frame[OLED_TX_CHUNK_SIZE + 1U];
    if ((data == NULL) || (length > OLED_TX_CHUNK_SIZE)) return 0U;

    frame[0] = OLED_DATA_CONTROL;
    memcpy(&frame[1], data, length);
    return HAL_I2C_Master_Transmit(&hi2c1, OLED_ADDR_HAL, frame,
                                   length + 1U, OLED_I2C_TIMEOUT_MS) == HAL_OK;
}

static void OLED_ClearBuffer(void)
{
    memset(oled_buffer, 0, sizeof(oled_buffer));
}

static uint8_t OLED_DrawChar(uint16_t x, uint16_t y, char ch)
{
    const uint8_t *glyph = Font_GetGlyph(ch);
    if ((glyph == NULL) || !OLED_CanDrawGlyph(x, y)) return 0U;
    OLED_CopyGlyphToBuffer(oled_buffer, x, y, glyph);
    return 1U;
}

static void OLED_DrawString(uint16_t x, uint16_t y, const char *text)
{
    while ((text != NULL) && (*text != '\0'))
    {
        if (!OLED_DrawChar(x, y, *text++)) break;
        x += FONT_GLYPH_WIDTH;
    }
}

static uint8_t OLED_Flush(void)
{
    for (uint16_t page = 0U; page < OLED_PAGE_COUNT; ++page)
    {
        if (!OLED_SetPageAndColumn(page, 0U)) return 0U;
        if (!OLED_WriteData(&oled_buffer[page][0], OLED_PAGE_WIDTH)) return 0U;
    }
    return 1U;
}

static uint8_t OLED_Init(const uint8_t *commands, uint16_t count)
{
    for (uint16_t i = 0U; i < count; ++i)
    {
        if (!OLED_WriteCommand(commands[i])) return 0U;
    }
    OLED_ClearBuffer();
    return OLED_Flush();
}
```

`OLED_ADDR_HAL`、控制字节、初始化命令、分辨率、页宽、字库和位图取模格式全部由实际控制器/模块资料提供。示例不包含未经核对的完整初始化序列。

## 调试方法

先做地址扫描，再验证写命令/写数据，最后逐级测试清屏、单字符和位图。I2C 有 ACK 不代表初始化序列正确。

> 视频待核对：控制器型号、I2C 地址、分辨率、控制字节、初始化命令、页寻址、字库和位图取模格式。

```text
确认接口、上拉和地址 ACK
-> 区分命令与显示数据事务
-> 核对初始化返回值和复位状态
-> 只测试清屏
-> 再测试固定字符和字符串
-> 用测试位图核对页列/bit 顺序
-> 最后检查刷新范围和闪烁
```

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| 地址 ACK 但黑屏 | 初始化命令/复位错误 | 逐步核对初始化序列 |
| 命令当数据 | 控制字节错误 | 分离 WriteCommand/Data |
| 字符乱码 | 字库尺寸/页方向不符 | 先显示固定小字符 |
| 位图颠倒 | 取模方向不一致 | 核对扫描和 bit 顺序 |
| 刷新闪烁 | 逐字节全屏更新 | 使用 framebuffer/局部刷新 |

## 待核对项

- [ ] 控制器型号、地址、分辨率和复位方式
- [ ] 初始化命令、控制字节、页/列组织
- [ ] 字库来源和位图取模格式

以实际开发板原理图、CubeMX 配置、视频课程源码和模块手册为准。

## 与其它章节的关系

- STM32 侧依赖：[[I2C总线]]、[[DMA与缓冲区]]、[[传感器采集与显示]]。
- [[DHT11单总线温湿度]] 可作为字符数据显示来源。
- 通用 bring-up 和数据显示见 [[传感器采集与显示]]。
- 跨外设排错见 [[通用调试方法]]。

## 复习检查清单

- [ ] 能确认控制器、接口、地址和分辨率，而不凭模块外观猜测。
- [ ] 能区分 OLED 命令、显示数据和 framebuffer。
- [ ] 能按初始化、清屏、固定字符、字符串和位图顺序 bring-up。
- [ ] 能说明字符显示依赖字库，位图显示依赖取模和页列组织。
- [ ] 能用 I2C ACK 与逻辑分析仪先排除总线问题。
- [ ] 能区分“地址有 ACK 但黑屏”与“完全无 ACK”的排查路径。
- [ ] 能指出初始化命令、控制字节、字库和位图格式必须按模块/视频核对。
