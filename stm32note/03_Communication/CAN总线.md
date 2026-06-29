---
type: bus
status: refactor
course: stm32-f103-hal
source:
  - local-notes
  - st-reference-manual
  - st-hal-documentation
tags:
  - stm32
  - can
  - bus
---
# CAN 总线

> 能力定位：理解多节点消息总线的仲裁、过滤、发送和接收路径。
>
> 原始来源：[[15_RS485与CAN]]。

PDF 对课程 CAN 实验无直接专题，不能凭空补完整实验。
---
## 本节目标

- 能区分 CAN 控制器和收发器
- 能解释仲裁、过滤器、邮箱和 FIFO
- 能用 Loopback 和真实总线分层调试

## 核心概念

### 知识地图

| 名词 | 工程含义 |
|---|---|
| 显性/隐性位 | 有线与仲裁的两种总线状态 |
| 标准/扩展帧 | 不同长度的标识符 |
| 数据帧/远程帧 | 携带数据/请求数据 |
| ID 仲裁 | ID 位级竞争决定优先级 |
| 过滤器 | 决定哪些帧进入 FIFO |
| Bus-Off | 错误累计后的总线隔离状态 |

### 控制器与收发器

STM32 内部 CAN 控制器负责帧、仲裁、邮箱、FIFO 和过滤器；外部 CAN 收发器负责逻辑电平与 CANH/CANL 差分信号转换。

### 总线机制

| 名词 | 工程含义 |
|---|---|
| 标准帧 / 扩展帧 | 使用不同长度的标识符 |
| ID 仲裁 | 多节点同时发送时，按总线规则决定优先级 |
| 发送邮箱 | 待发送帧进入的硬件槽位 |
| 接收 FIFO | 通过过滤器的帧进入的接收队列 |
| 过滤器 | 决定哪些 ID 被接收及进入哪个 FIFO |

### HAL 主线

```text
配置位时序与过滤器
-> HAL_CAN_Start
-> 激活接收通知
-> HAL_CAN_AddTxMessage
-> FIFO 回调中 HAL_CAN_GetRxMessage
```

### Loopback 调试

内部 Loopback 可先验证控制器配置、发送邮箱、接收 FIFO 和过滤器逻辑；它不能替代真实收发器、终端电阻和双节点总线测试。

> 视频待核对：收发器型号、波特率、采样点、过滤器、课程消息 ID、双板实验和分析仪设置。

## 本质理解

CAN 是以“消息 ID”而不是点对点地址为中心的多节点总线。控制器负责帧、仲裁和错误处理，收发器负责 CANH/CANL 电气信号；两者缺一不可。

## STM32F103 / HAL 中的实现方式

- F103 CAN 控制器使用发送邮箱、接收 FIFO 和硬件过滤器。
- 位时序由预分频、SJW、BS1、BS2 组成，决定波特率和采样点；参数必须按 PCLK、线缆和节点统一计算。
- Loopback 可验证控制器和软件流程，但不能证明收发器、终端和真实总线正常。

## CubeMX 配置要点

1. 启用 CAN 并确认 RX/TX 复用引脚。
2. 按统一目标配置 Prescaler、SJW、BS1、BS2，不照抄未知参数。
3. 配置过滤器后启动 CAN。
4. 激活 FIFO 接收通知。
5. 真实总线连接收发器、CANH/CANL 和终端电阻。

## 常用 HAL API

### API 速查

| HAL 函数 / 宏 | 作用 | 常见位置 |
|---|---|---|
| `HAL_CAN_ConfigFilter()` | 配置哪些 ID 进入哪个 RX FIFO | CAN 启动前 |
| `HAL_CAN_Start()` | 让 CAN 控制器进入工作状态 | 过滤器配置后 |
| `HAL_CAN_Stop()` | 停止 CAN 控制器 | 重配置、停机 |
| `HAL_CAN_AddTxMessage()` | 把一帧放入可用发送邮箱 | 业务发送函数 |
| `HAL_CAN_GetRxMessage()` | 从指定 FIFO 取出一帧 | RX FIFO 回调 |
| `HAL_CAN_ActivateNotification()` | 开启 RX FIFO/错误等通知 | Start 后 |
| `HAL_CAN_RxFifo0MsgPendingCallback()` | FIFO0 有待取消息时进入用户回调 | 用户代码 |
| `HAL_CAN_GetError()` | 读取 CAN HAL/协议错误状态 | 发送失败、Bus-Off 诊断 |

### 使用注意

| HAL 函数 / 宏 | 前置条件 | 参数重点 | 返回值 / 阻塞与常见坑 |
|---|---|---|---|
| `HAL_CAN_ConfigFilter()` | 过滤器结构体与 bank 分配已设计 | ID/Mask、模式、FIFO、激活状态 | 返回 `HAL_StatusTypeDef`；同步配置；过滤器过窄导致“总线有帧但软件收不到” |
| `HAL_CAN_Start()` | 位时序、GPIO、收发器准备好 | CAN 句柄 | 返回状态；非阻塞；CubeMX 初始化后忘记 Start；位时序双方不一致 |
| `HAL_CAN_Stop()` | CAN 已启动 | CAN 句柄 | 返回状态；同步停止；运行中直接改过滤器/位时序而未按状态机停机 |
| `HAL_CAN_AddTxMessage()` | CAN 已 Start，总线可用 | Tx Header、最多 8 字节数据、邮箱输出指针 | 返回状态；非阻塞入邮箱；返回 `HAL_OK` 只表示成功排队，不保证已获 ACK 或发到总线 |
| `HAL_CAN_GetRxMessage()` | FIFO 中确有消息 | FIFO、Rx Header、数据数组 | 返回状态；同步取出；回调里不取帧导致 FIFO 堆积；FIFO 参数不匹配 |
| `HAL_CAN_ActivateNotification()` | CAN 与 NVIC 已配置 | 通知位掩码 | 返回状态；非阻塞；只开 NVIC 不激活 HAL 通知；通知类型与回调不匹配 |
| `HAL_CAN_RxFifo0MsgPendingCallback()` | 已激活 FIFO0 pending 通知 | CAN 句柄 | `void`；中断上下文；忘记调用 `HAL_CAN_GetRxMessage()`；回调中长时间处理 |
| `HAL_CAN_GetError()` | CAN 句柄有效 | CAN 句柄 | 返回错误位掩码；非阻塞；只重发不查 ACK、收发器、终端和错误计数 |

### 典型调用顺序

```text
MX_CAN_Init() 配置位时序
-> HAL_CAN_ConfigFilter()
-> HAL_CAN_Start()
-> HAL_CAN_ActivateNotification()
-> HAL_CAN_AddTxMessage() 放入发送邮箱
-> HAL_CAN_RxFifo0MsgPendingCallback()
-> HAL_CAN_GetRxMessage() 取出帧并发布事件
```

### 什么时候用哪个函数？

| 场景 | 推荐函数 |
|---|---|
| 启动接收前设置可见 ID | `HAL_CAN_ConfigFilter()`，调试初期可先放宽 |
| 发送一帧 | `HAL_CAN_AddTxMessage()`，并另外监控错误/邮箱状态 |
| 中断接收 FIFO0 | `ActivateNotification()` + `RxFifo0MsgPendingCallback()` + `GetRxMessage()` |
| 修改关键配置 | `HAL_CAN_Stop()` 后按 HAL 状态要求重配再 Start |
| 收不到/发不出 | 查过滤器、位时序、收发器、终端、ACK，再读 `HAL_CAN_GetError()` |

位时序、过滤器值和消息 ID 以实际 CubeMX 配置、节点协议和总线实测为准。

## 最小实验 / 最小框架

> 代码性质：示例框架，用于理解调用顺序，不能保证直接编译。
> 验证状态：未上板验证，需按 CubeMX 变量名、开发板原理图和课程源码核对。

```c
static volatile uint8_t can_rx_ready;
static CAN_RxHeaderTypeDef can_rx_header;
static uint8_t can_rx_data[CAN_CLASSIC_MAX_DATA_BYTES];

static uint8_t CAN_StartNode(void)
{
    CAN_FilterTypeDef filter = {0};
    CAN_LoadProjectFilter(&filter); /* ID/Mask/FIFO 由项目配置提供。 */

    if (HAL_CAN_ConfigFilter(&hcan, &filter) != HAL_OK) return 0U;
    if (HAL_CAN_Start(&hcan) != HAL_OK) return 0U;
    if (HAL_CAN_ActivateNotification(
            &hcan, CAN_IT_RX_FIFO0_MSG_PENDING | CAN_IT_ERROR) != HAL_OK)
    {
        (void)HAL_CAN_Stop(&hcan);
        return 0U;
    }
    return 1U;
}

static uint8_t CAN_SendFrame(const CAN_TxHeaderTypeDef *header,
                             const uint8_t *data)
{
    uint32_t mailbox;
    HAL_StatusTypeDef status = HAL_CAN_AddTxMessage(&hcan, header,
                                                    data, &mailbox);
    if (status != HAL_OK)
    {
        Debug_ReportCanError(HAL_CAN_GetError(&hcan));
        return 0U;
    }
    return 1U;
}

void HAL_CAN_RxFifo0MsgPendingCallback(CAN_HandleTypeDef *hcan)
{
    if (hcan->Instance != CAN1) return;

    if (HAL_CAN_GetRxMessage(hcan, CAN_RX_FIFO0,
                             &can_rx_header, can_rx_data) == HAL_OK)
    {
        can_rx_ready = 1U;
    }
    else
    {
        Debug_ReportCanError(HAL_CAN_GetError(hcan));
    }
}

if (!CAN_StartNode())
{
    Error_Handler();
}

while (1)
{
    if (can_rx_ready)
    {
        can_rx_ready = 0U;
        App_ProcessCanFrame(&can_rx_header, can_rx_data);
    }
}

```

`CAN_LoadProjectFilter()` 和 Tx Header 由项目协议填充；位时序、ID、过滤器和收发器参数不在示例中写死，以 CubeMX、节点协议和总线实测为准。

## 调试方法

```text
先用 Loopback 验证软件
-> 确认真实收发器供电/使能
-> 量 CANH/CANL 静态与波形
-> 确认两端终端和波特率
-> 过滤器先放宽
-> 检查 ACK/错误计数/Bus-Off
-> 再收紧过滤器和协议
```

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| 完全无波形 | 没有收发器或未启动 | 检查硬件并 HAL_CAN_Start |
| 发送失败 | 总线无 ACK/波特率不同 | 接入同参数节点 |
| 收不到 | 过滤器/FIFO/通知错误 | 放宽过滤器并开通知 |
| CANH/CANL 反 | 接线错误 | 按收发器定义连接 |
| 总线反射 | 终端缺失/拓扑错误 | 总线两端匹配终端 |
| Bus-Off | 持续错误累计 | 定位物理/位时序后恢复 |

## 与其它章节的关系

- [[RCC时钟树]] 提供 CAN PCLK。
- [[NVIC_EXTI_SysTick]] 解释接收通知。
- [[联网控制与总线通信]] 设计节点协议。
- [[通用调试方法]] 提供物理层到回调的分层排查方法。

## 复习检查清单

- [ ] 能区分 STM32 内部 CAN 控制器与外部 CAN 收发器的职责。
- [ ] 能说明节点必须在位时序、波特率和物理总线条件上兼容。
- [ ] 能按过滤器配置、`HAL_CAN_Start()`、通知使能的顺序启动 CAN。
- [ ] 能说明过滤器决定消息进入哪个 FIFO，而不是改变总线上的消息。
- [ ] 能检查发送邮箱返回值、总线 ACK 和错误计数。
- [ ] 能在 FIFO0 回调中调用 `HAL_CAN_GetRxMessage()` 取出报文。
- [ ] 能排查终端电阻、拓扑、CANH/CANL、过滤器和 Bus-Off 问题。
- [ ] 能指出课程位时序、消息 ID 和双板实验参数仍需视频实测确认。
