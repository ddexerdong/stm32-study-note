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

## 控制器与收发器

STM32 内部 CAN 控制器负责帧、仲裁、邮箱、FIFO 和过滤器；外部 CAN 收发器负责逻辑电平与 CANH/CANL 差分信号转换。

## 核心概念

| 名词 | 工程含义 |
|---|---|
| 标准帧 / 扩展帧 | 使用不同长度的标识符 |
| ID 仲裁 | 多节点同时发送时，按总线规则决定优先级 |
| 发送邮箱 | 待发送帧进入的硬件槽位 |
| 接收 FIFO | 通过过滤器的帧进入的接收队列 |
| 过滤器 | 决定哪些 ID 被接收及进入哪个 FIFO |

## HAL 主线

```text
配置位时序与过滤器
-> HAL_CAN_Start
-> 激活接收通知
-> HAL_CAN_AddTxMessage
-> FIFO 回调中 HAL_CAN_GetRxMessage
```

## Loopback 调试

内部 Loopback 可先验证控制器配置、发送邮箱、接收 FIFO 和过滤器逻辑；它不能替代真实收发器、终端电阻和双节点总线测试。

> 视频待核对：收发器型号、波特率、采样点、过滤器、课程消息 ID、双板实验和分析仪设置。

## 关联知识

- [[NVIC_EXTI_SysTick]]
- [[联网控制与总线通信]]
- [[通用调试方法]]

---

## 本节目标

- 能区分 CAN 控制器和收发器
- 能解释仲裁、过滤器、邮箱和 FIFO
- 能用 Loopback 和真实总线分层调试

## 知识地图

| 名词 | 工程含义 |
|---|---|
| 显性/隐性位 | 有线与仲裁的两种总线状态 |
| 标准/扩展帧 | 不同长度的标识符 |
| 数据帧/远程帧 | 携带数据/请求数据 |
| ID 仲裁 | ID 位级竞争决定优先级 |
| 过滤器 | 决定哪些帧进入 FIFO |
| Bus-Off | 错误累计后的总线隔离状态 |

## 本质理解

CAN 是以“消息 ID”而不是点对点地址为中心的多节点总线。控制器负责帧、仲裁和错误处理，收发器负责 CANH/CANL 电气信号；两者缺一不可。

为什么这样做：先建立硬件和数据流模型，再记 HAL API，才能在 API 变化或板卡变化时仍然定位问题。

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

| API / 对象 | 作用 | 常见位置 |
|---|---|---|
| `HAL_CAN_ConfigFilter` | 配置过滤器 | 启动前 |
| `HAL_CAN_Start` | 启动 CAN 控制器 | 初始化后 |
| `HAL_CAN_ActivateNotification` | 开启 FIFO/错误通知 | 启动后 |
| `HAL_CAN_AddTxMessage` | 把帧放入发送邮箱 | 发送 |
| `HAL_CAN_GetRxMessage` | 从 FIFO 取帧 | 接收回调 |
| `HAL_CAN_GetError` | 读取错误状态 | 故障诊断 |

## 最小实验 / 最小框架

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
HAL_CAN_ConfigFilter(&hcan, &filter);
HAL_CAN_Start(&hcan);
HAL_CAN_ActivateNotification(&hcan, CAN_IT_RX_FIFO0_MSG_PENDING);

/* 发送时填充 header，再调用 HAL_CAN_AddTxMessage。 */
```

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

## 复习检查清单

- [ ] 能否不用看代码说清本章数据流和硬件链路？
- [ ] 能否在 CubeMX 中完成最小配置并说明每个关键选项？
- [ ] 能否写出初始化、启动和回调/轮询的最小框架？
- [ ] 能否用原始电平、波形、返回值或寄存器证明外设在工作？
- [ ] 能否按“硬件 -> 时钟/配置 -> Start -> 回调 -> 业务”排查故障？
- [ ] 能否指出本章哪些参数必须按原理图、手册或实测确认？
