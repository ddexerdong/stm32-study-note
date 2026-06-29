---
type: module
status: refactor
course: stm32-f103-hal
source:
  - local-notes
  - module-datasheet
tags:
  - stm32
  - esp8266
  - uart
  - wifi
---
# ESP8266 AT 指令

> 模块定位：把 WiFi 模块视为“UART 控制的状态机”，而不是透明网络黑盒。
>
> 原始来源：[[14_RGB灯与ESP8266]]。
---
## 本节目标

- 能把 ESP8266 当作 UART 控制的异步状态机。
- 能分阶段验证 AT、联网、连接、透传和业务协议。
- 能设计响应解析、超时和重连，而不阻塞主循环。

## 核心概念

### 知识地图

| 名词 | 工程含义 |
|---|---|
| AT 命令 | 文本命令和响应协议 |
| STA/AP | 连接热点/创建热点 |
| TCP/UDP | 面向连接/无连接传输 |
| 透传 | UART 字节直接转发到网络连接 |
| 异步消息 | 模块主动上报连接或数据事件 |
| 状态机 | 按响应和超时推进步骤 |

### 模块作用

通过 UART 控制 WiFi 模块，建立 AT 命令状态机、网络连接和业务数据收发。

### 分层

```text
UART 字节收发
-> AT 命令发送与响应匹配
-> 模式状态机（STA/AP）
-> 建立网络连接
-> 普通收发或透传
-> 业务协议
```

### 工程重点

- 每条命令要有期望响应和超时。
- 模块启动、复位、入网和连接都需要状态判断。
- 透传模式和 AT 命令模式要能明确切换。
- 网络层成功不等于业务协议正确。

> 视频待核对：固件版本、默认波特率、指令结尾、STA/AP 指令顺序、透传进入/退出、响应格式和课程网络工具。

不要记录真实 WiFi 名称、密码、服务器 IP 或端口。

## 本质理解

ESP8266 AT 通信是异步的命令、响应和主动上报过程，不是“发送字符串后立刻得到结果”的同步函数。可靠控制必须把命令状态、期望响应、超时、重试和网络数据入口分开处理。

## 硬件连接关系

- 稳定供电和峰值电流能力
- UART TX/RX 交叉并共地
- EN/RST/BOOT 脚按模块设计
- 逻辑电平兼容
- 调试串口与模块串口分开更清晰

## 通信 / 控制链路

```text
上电等待 ready
-> 发送基础 AT 并匹配 OK
-> 查询版本
-> 设置 STA/AP
-> 联网
-> 建立 TCP/UDP
-> 普通收发/透传
-> 异常退出和重连
```

## CubeMX 配置要点

1. 启用一个 USART 连接 ESP8266
2. 配置 RX DMA + IDLE 或足够缓冲
3. 另一个 USART 可做日志
4. 不在代码中保存真实 WiFi/IP/端口示例

## 本模块依赖的 HAL 层

| 模块动作 | 依赖 HAL | 说明 |
|---|---|---|
| 发送短 AT 命令 | `HAL_UART_Transmit()` | 阻塞发送；命令文本和行结束格式以当前 AT 固件文档为准 |
| 接收固定长度测试数据 | `HAL_UART_Receive_IT()` | 适合最小验证，不适合复杂异步 AT 响应 |
| 接收不定长响应/网络数据 | `HAL_UARTEx_ReceiveToIdle_DMA()` | 把 `Size` 个字节送入 ring buffer，再由解析状态机识别行和提示符 |
| 处理接收事件 | `HAL_UARTEx_RxEventCallback()` | 回调只搬运/发布数据，不在其中等待 `OK` 或执行网络业务 |
| 命令等待和重试超时 | `HAL_GetTick()` | 每个状态有截止时间，禁止无限等待 |
| 控制 RST/EN | `HAL_GPIO_WritePin()` | 可选；有效电平和上电时序以模块原理图/官方文档为准 |
| 错误恢复 | `HAL_UART_ErrorCallback()` / `HAL_UART_DMAStop()` | 记录错误、停止旧接收链、清状态后重新启动 |

HAL 只负责 UART、DMA、GPIO 和时间基准。AT 命令集合、固件差异、STA/AP、TCP/UDP、透传进入/退出和业务协议属于 ESP-AT/课程层，不能由 UART HAL 自动完成。

### 典型调用顺序

```text
MX_DMA_Init()
-> MX_USARTx_UART_Init()
-> 可选 RST/EN 上电控制
-> HAL_UARTEx_ReceiveToIdle_DMA() 先启动接收
-> HAL_UART_Transmit() 发送一条 AT 命令
-> RxEventCallback 将响应送入 ring buffer
-> 主循环状态机匹配 OK/ERROR/提示符并处理超时
```

### 什么时候用哪个函数？

| 场景 | 推荐函数 |
|---|---|
| 最小 AT 连通测试 | `HAL_UART_Transmit()` + 接收原始响应 |
| 持续接收异步 AT/网络数据 | `HAL_UARTEx_ReceiveToIdle_DMA()` |
| 命令超时/重试 | `HAL_GetTick()` 状态机 |
| 模块硬复位 | 经原理图确认后的 `HAL_GPIO_WritePin()` |

## Bring-up 顺序

1. 确认模块供电能力、使能/复位状态、共地和 UART 接线。
2. 用串口助手或最小 UART 程序验证基础 AT 命令及行结束。
3. 记录启动信息、固件响应和实际波特率，不先进入透传。
4. 分别验证 STA 或 AP 状态，再建立单一网络连接。
5. 验证发送数据与异步接收入口，保留原始字节日志。
6. 最后加入超时、重试、重连和业务命令解析。

## 最小驱动框架

> 代码性质：示例框架，用于理解调用顺序，不能保证直接编译。

```c
static uint8_t ESP8266_SendLine(const char *command)
{
    if (command == NULL) return 0U;
    return HAL_UART_Transmit(&huart2, (const uint8_t *)command,
                              (uint16_t)strlen(command),
                              ESP8266_UART_TIMEOUT_MS) == HAL_OK;
}

static uint8_t ESP8266_WaitResponse(const char *expected,
                                    uint32_t timeout_ms)
{
    uint32_t start = HAL_GetTick();
    while ((uint32_t)(HAL_GetTick() - start) < timeout_ms)
    {
        ESP8266_PollRx();
        if (ESP8266_ResponseContains(expected)) return 1U;
        if (ESP8266_ResponseHasError()) return 0U;
    }
    return 0U;
}

uint8_t ESP8266_Command(const char *cmd, const char *expect, uint32_t timeout)
{
    ESP8266_ClearRx();
    if (!ESP8266_SendLine(cmd)) return 0U;
    return ESP8266_WaitResponse(expect, timeout);
}

static uint8_t ESP8266_BasicTest(void)
{
    return ESP8266_Command(ESP8266_CMD_AT,
                           ESP8266_RESPONSE_OK,
                           ESP8266_COMMAND_TIMEOUT_MS);
}

static uint8_t ESP8266_ConnectWifi(const char *ssid, const char *password)
{
    char command[ESP8266_COMMAND_BUFFER_SIZE];
    if (!ESP8266_FormatJoinCommand(command, sizeof(command), ssid, password))
        return 0U;
    return ESP8266_Command(command, ESP8266_RESPONSE_OK,
                           ESP8266_JOIN_TIMEOUT_MS);
}

static uint8_t ESP8266_SendPayload(const uint8_t *data, uint16_t length)
{
    if (!ESP8266_PrepareSend(length)) return 0U;
    return HAL_UART_Transmit(&huart2, data, length,
                              ESP8266_DATA_TIMEOUT_MS) == HAL_OK;
}

static void ESP8266_OnReceivedCommand(const uint8_t *data, uint16_t length)
{
    Protocol_PushBytes(data, length);
}

if (!ESP8266_BasicTest() ||
    !ESP8266_ConnectWifi(WIFI_SSID_PLACEHOLDER, WIFI_PASSWORD_PLACEHOLDER))
{
    Debug_ReportEsp8266State();
}
```

AT 命令文本、行结束、响应、STA/AP、连接和透传步骤由当前 ESP-AT 固件决定。SSID、密码、服务器和端口必须由安全配置提供，不写入公开笔记或源码示例。

## 调试方法

```text
确认供电峰值与 UART 电平
-> 保存上电原始输出
-> 验证基础 AT 命令和行结束
-> 检查每条命令的期望响应/超时
-> 分阶段验证入网与连接
-> 检查透传进入/退出状态
-> 最后调业务协议和重连
```

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| 无响应 | 供电/波特率/换行错误 | 先用串口助手测基础 AT |
| 偶发复位 | 供电峰值不足 | 改善电源和去耦 |
| 命令顺序失控 | 未等响应就发下一条 | 使用状态机和超时 |
| 透传退不出 | 退出条件/静默时间不符 | 按固件手册执行 |
| 粘包半包 | 按一次 DMA 当完整响应 | 流式匹配和 ring buffer |
| 联网但业务失败 | 应用协议不一致 | 打印原始网络数据 |

## 待核对项

- [ ] AT 固件版本和默认波特率
- [ ] 命令结尾、STA/AP/TCP/UDP 指令顺序
- [ ] 透传进入/退出和异步响应格式

以实际开发板原理图、CubeMX 配置、视频课程源码和模块手册为准。

## 与其它章节的关系

- STM32 侧依赖：[[USART与串口协议]]、[[DMA与缓冲区]]、[[联网控制与总线通信]]。
- 响应和下行数据解析见 [[串口协议解析]]。
- 通用 bring-up 和数据显示见 [[传感器采集与显示]]。
- 跨外设排错见 [[通用调试方法]]。

## 复习检查清单

- [ ] 能先验证供电峰值能力、共地、UART 接线和基础 AT 响应。
- [ ] 能把 AT 命令、期望响应和主动上报作为异步字节流解析。
- [ ] 能为每个命令状态设置超时、失败和重试路径。
- [ ] 能区分 STA、AP、连接和透传阶段，不把它们混成一次函数调用。
- [ ] 能处理 DMA 接收中的半包、粘包和响应跨回调问题。
- [ ] 能保证真实 WiFi 名称、密码和服务器信息不写入公开笔记示例。
- [ ] 能指出固件版本、默认波特率、AT 顺序和透传退出流程必须实测确认。
