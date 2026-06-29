---
type: engineering
status: refactor
course: stm32-f103-hal
source:
  - local-notes
  - st-hal-documentation
tags:
  - stm32
  - hal
  - api
---
# HAL API 速查表

> 用法：先按目标动作找到 API，再回到对应外设笔记确认 CubeMX、硬件链路、调用顺序和回调条件。本表核对基准为 ST 官方 STM32F1 HAL Driver；具体工程仍以实际使用的 CubeF1/HAL 版本为准。

## 使用原则

1. CubeMX 生成 `MX_xxx_Init()` 只完成配置，不代表外设已经开始计数、发送、接收或采样。
2. 大部分启动/传输函数返回 `HAL_StatusTypeDef`，应检查 `HAL_OK`、`HAL_BUSY`、`HAL_TIMEOUT`、`HAL_ERROR`。
3. `_IT` 和 `_DMA` 通常快速返回，buffer 必须保持有效，完成事件在 HAL 回调中出现。
4. 回调运行在中断上下文：区分句柄/实例，只复制数据、置标志或投递事件。
5. HAL 只解决 MCU 外设状态机。模块寄存器、传感器公式、AT 命令、Flash 命令和应用协议不属于 HAL。
6. DWT/CYCCNT、`CoreDebug`、NVIC 内核对象等部分来自 CMSIS/内核组件，不应伪装成普通外设 HAL API。

## 本质理解

API 速查表不是函数大全，而是帮助判断当前场景应该选择阻塞、中断还是 DMA 入口。选错执行模型会直接影响 buffer 生命周期、回调时机和主循环响应，通常比单个函数名写错更难排查。

## GPIO

详见 [[GPIO与AFIO]]。

### API 速查

| API | 作用 | 常见调用位置 |
|---|---|---|
| `HAL_GPIO_Init()` | 配置引脚模式、上下拉和速度 | `MX_GPIO_Init()`/BSP Init |
| `HAL_GPIO_WritePin()` | 设置普通输出电平 | LED、CS、DE/RE、使能脚 |
| `HAL_GPIO_ReadPin()` | 读取当前输入状态 | 按键、DO、状态脚 |
| `HAL_GPIO_TogglePin()` | 翻转普通输出 | LED 心跳/低速标记 |

### 使用注意

| API | 前置条件 | 阻塞 / 非阻塞与易错点 |
|---|---|---|
| `HAL_GPIO_Init()` | GPIO 时钟已开 | 同步配置；复用、模拟、输入输出模式配错 |
| `HAL_GPIO_WritePin()` | 引脚为普通输出 | 非阻塞；不控制已交给外设复用的波形 |
| `HAL_GPIO_ReadPin()` | 输入模式和默认电平可靠 | 非阻塞；不是读取输出锁存器；悬空值无意义 |
| `HAL_GPIO_TogglePin()` | 普通输出 | 非阻塞；不适合精确时序或 PWM |

## EXTI / NVIC

详见 [[NVIC_EXTI_SysTick]]。

### API 速查

| API | 作用 | 常见调用位置 |
|---|---|---|
| `HAL_NVIC_SetPriority()` | 设置 IRQ 优先级 | MSP/初始化 |
| `HAL_NVIC_EnableIRQ()` | 使能 IRQ | 初始化 |
| `HAL_NVIC_DisableIRQ()` | 屏蔽 IRQ | 临界维护/停机 |
| `HAL_GPIO_EXTI_IRQHandler()` | 清 EXTI 挂起并分发回调 | `stm32f1xx_it.c` |
| `HAL_GPIO_EXTI_Callback()` | 用户 EXTI 弱回调 | 用户代码 |

### 使用注意

| API | 前置条件 | 阻塞 / 非阻塞与易错点 |
|---|---|---|
| `HAL_NVIC_SetPriority()` | 优先级分组明确 | 同步配置；数值越小优先级越高 |
| `HAL_NVIC_EnableIRQ()` | IRQn 与外设匹配 | 同步配置；只开外设中断、没开 NVIC |
| `HAL_NVIC_DisableIRQ()` | 有恢复策略 | 同步配置；不会自动清外设中断标志 |
| `HAL_GPIO_EXTI_IRQHandler()` | EXTI/NVIC 已配置 | 中断上下文；IRQHandler 传错 Pin |
| `HAL_GPIO_EXTI_Callback()` | HAL IRQHandler 链完整 | 中断上下文；修改 HAL 源码；回调中阻塞 |

## SysTick / Tick

### API 速查

| API | 作用 | 常见调用位置 |
|---|---|---|
| `HAL_GetTick()` | 读取 HAL tick | 超时、消抖、状态机 |
| `HAL_Delay()` | 阻塞等待指定毫秒 tick | 简单初始化/实验 |
| `HAL_IncTick()` | 递增 HAL tick | 默认时基 ISR |

### 使用注意

| API | 前置条件 | 阻塞 / 非阻塞与易错点 |
|---|---|---|
| `HAL_GetTick()` | tick 正常递增 | 非阻塞；未用无符号差值处理回绕 |
| `HAL_Delay()` | tick 中断可运行 | 阻塞；中断中调用可能卡死或拖延其它 IRQ |
| `HAL_IncTick()` | HAL 时基配置正确 | 中断上下文；业务手动调用导致系统时间失真 |

## TIM Base

详见 [[TIM定时器体系]]。

### API 速查

| API | 作用 | 常见调用位置 |
|---|---|---|
| `HAL_TIM_Base_Start()` | 启动计数，不开更新中断 | 触发源/自由计数 |
| `HAL_TIM_Base_Start_IT()` | 启动计数和更新中断 | 周期任务 |
| `HAL_TIM_Base_Stop_IT()` | 停止计数和更新中断 | 暂停/停机 |
| `HAL_TIM_PeriodElapsedCallback()` | 更新事件用户回调 | 用户代码 |
| `__HAL_TIM_SET_COUNTER()` | 写 CNT | 回零/新测量 |
| `__HAL_TIM_GET_COUNTER()` | 读 CNT | 位置/时间采样 |

### 使用注意

| API | 前置条件 | 阻塞 / 非阻塞与易错点 |
|---|---|---|
| `HAL_TIM_Base_Start()` | TIM Init 完成 | 非阻塞；误以为会进入回调 |
| `HAL_TIM_Base_Start_IT()` | TIM NVIC 已开 | 非阻塞；只调用 Init，没有 Start_IT |
| `HAL_TIM_Base_Stop_IT()` | 已用 IT 启动 | 同步停止；停止后业务标志未清 |
| `HAL_TIM_PeriodElapsedCallback()` | IRQHandler 链完整 | 中断上下文；多 TIM 不判断 `htim->Instance` |
| `__HAL_TIM_SET_COUNTER()` | 句柄有效 | 非阻塞宏；运行中修改改变本周期 |
| `__HAL_TIM_GET_COUNTER()` | 计数频率已知 | 非阻塞宏；忽略溢出和单位 |

## TIM PWM

详见 [[TIM_PWM输出]]。

### API 速查

| API | 作用 | 常见调用位置 |
|---|---|---|
| `HAL_TIM_PWM_Start()` | 启动指定 PWM 通道 | 初始化后 |
| `HAL_TIM_PWM_Stop()` | 停止指定 PWM 通道 | 安全停机 |
| `__HAL_TIM_SET_COMPARE()` | 写 CCR 改占空比/脉宽 | 调光、舵机、电机 |
| `__HAL_TIM_GET_COMPARE()` | 读 CCR | 状态/调试 |
| `HAL_TIM_PWM_PulseFinishedCallback()` | PWM IT/DMA 完成回调 | 用户代码 |

### 使用注意

| API | 前置条件 | 阻塞 / 非阻塞与易错点 |
|---|---|---|
| `HAL_TIM_PWM_Start()` | 通道和复用引脚已配置 | 非阻塞；CubeMX 配好但未启动；通道错误 |
| `HAL_TIM_PWM_Stop()` | PWM 已启动 | 同步停止；与 CCR=0 的业务状态混淆 |
| `__HAL_TIM_SET_COMPARE()` | PWM 已启动 | 非阻塞宏；CCR 超 ARR；单位理解错 |
| `__HAL_TIM_GET_COMPARE()` | 通道有效 | 非阻塞宏；读到 CCR 不代表引脚有波形 |
| `HAL_TIM_PWM_PulseFinishedCallback()` | 使用 PWM IT/DMA 模式 | 中断上下文；普通 PWM Start 不会每周期调用它 |

## TIM Input Capture

详见 [[TIM输入捕获与编码器]]。

### API 速查

| API | 作用 | 常见调用位置 |
|---|---|---|
| `HAL_TIM_IC_Start()` | 启动输入捕获，不开捕获中断 | 轮询方案 |
| `HAL_TIM_IC_Start_IT()` | 启动输入捕获中断 | 脉宽/频率测量 |
| `HAL_TIM_ReadCapturedValue()` | 读取锁存的 CCR | 捕获回调 |
| `__HAL_TIM_SET_CAPTUREPOLARITY()` | 切换捕获边沿 | 单通道双边沿测量 |
| `HAL_TIM_IC_CaptureCallback()` | 输入捕获用户回调 | 用户代码 |

### 使用注意

| API | 前置条件 | 阻塞 / 非阻塞与易错点 |
|---|---|---|
| `HAL_TIM_IC_Start()` | IC 通道已配置 | 非阻塞；误以为会进入回调 |
| `HAL_TIM_IC_Start_IT()` | TIM NVIC 已开 | 非阻塞；边沿、通道或 NVIC 错 |
| `HAL_TIM_ReadCapturedValue()` | 已有捕获事件 | 非阻塞；把 CCR 当当前 CNT；忽略回绕 |
| `__HAL_TIM_SET_CAPTUREPOLARITY()` | IC 通道有效 | 非阻塞宏；切换边沿后阶段状态未同步 |
| `HAL_TIM_IC_CaptureCallback()` | HAL TIM IRQ 链完整 | 中断上下文；未检查活动通道；回调中重计算 |

## TIM Encoder

### API 速查

| API | 作用 | 常见调用位置 |
|---|---|---|
| `HAL_TIM_Encoder_Start()` | 启动编码器接口 | 初始化后 |
| `HAL_TIM_Encoder_Start_IT()` | 启动编码器及相关中断 | 特殊事件方案 |
| `__HAL_TIM_GET_COUNTER()` | 读取位置计数 | 固定周期任务 |
| `__HAL_TIM_SET_COUNTER()` | 设置计数起点 | 回零 |

### 使用注意

| API | 前置条件 | 阻塞 / 非阻塞与易错点 |
|---|---|---|
| `HAL_TIM_Encoder_Start()` | CH1/CH2 与 Encoder Mode 已配 | 非阻塞；A/B 相、极性、滤波不对 |
| `HAL_TIM_Encoder_Start_IT()` | NVIC/中断策略明确 | 非阻塞；普通测速滥用每边沿中断 |
| `__HAL_TIM_GET_COUNTER()` | 编码器已启动 | 非阻塞宏；差分未处理回绕和符号 |
| `__HAL_TIM_SET_COUNTER()` | 并发访问已处理 | 非阻塞宏；与 ISR 同时修改造成跳变 |

## ADC

详见 [[ADC与模拟采样]]、[[TIM触发ADC]]。

### API 速查

| API | 作用 | 常见调用位置 |
|---|---|---|
| `HAL_ADCEx_Calibration_Start()` | 校准 F1 ADC | 首次采样前 |
| `HAL_ADC_Start()` | 启动转换/等待外部触发 | 轮询或 TIM 触发 |
| `HAL_ADC_PollForConversion()` | 等待转换完成 | 最小轮询实验 |
| `HAL_ADC_GetValue()` | 读取转换结果 | Poll 成功后 |
| `HAL_ADC_Start_IT()` | 启动转换中断 | 低频事件采样 |
| `HAL_ADC_Start_DMA()` | 启动 ADC DMA | 连续多通道 |
| `HAL_ADC_Stop_DMA()` | 停止 ADC DMA | 模式切换/停机 |
| `HAL_ADC_ConvHalfCpltCallback()` | 前半 buffer 完成 | 双半区处理 |
| `HAL_ADC_ConvCpltCallback()` | 整轮/后半完成 | 用户代码 |

### 使用注意

| API | 前置条件 | 阻塞 / 非阻塞与易错点 |
|---|---|---|
| `HAL_ADCEx_Calibration_Start()` | ADC 初始化且空闲 | 同步等待；忘记检查返回值 |
| `HAL_ADC_Start()` | 通道/触发配置正确 | 非阻塞启动；外部触发未到却认为已完成 |
| `HAL_ADC_PollForConversion()` | ADC 已启动 | 阻塞；超时过长卡主循环 |
| `HAL_ADC_GetValue()` | 转换已完成 | 非阻塞；读旧值；把 raw 直接当物理量 |
| `HAL_ADC_Start_IT()` | ADC NVIC 已开 | 非阻塞；扫描模式的回调粒度理解错误 |
| `HAL_ADC_Start_DMA()` | DMA 绑定、buffer 长期有效 | 非阻塞；Rank/长度/数组顺序不一致 |
| `HAL_ADC_Stop_DMA()` | DMA 正在运行 | 同步停止；停止后旧业务状态未清 |
| `HAL_ADC_ConvHalfCpltCallback()` | Circular/IRQ 已开 | 中断上下文；消费不及下一轮覆盖 |
| `HAL_ADC_ConvCpltCallback()` | 中断链完整 | 中断上下文；回调中滤波/打印过重 |

## DMA

详见 [[DMA与缓冲区]]。

### API 速查

| API | 作用 | 常见调用位置 |
|---|---|---|
| `HAL_DMA_Init()` | 初始化 DMA 通道 | MSP/CubeMX |
| `HAL_DMA_Start()` | 启动裸 DMA，无完成中断 | 特殊搬运 |
| `HAL_DMA_Start_IT()` | 启动裸 DMA 并开中断 | 自定义 DMA 驱动 |
| `HAL_DMA_IRQHandler()` | 处理 DMA IRQ | `stm32f1xx_it.c` |
| `HAL_DMA_Abort()` | 中止 DMA | 错误恢复 |
| `HAL_DMA_GetState()` | 查询 DMA 状态 | 诊断/重启前 |
| `__HAL_DMA_GET_COUNTER()` | 读剩余传输数 NDTR | 部分长度计算 |

### 使用注意

| API | 前置条件 | 阻塞 / 非阻塞与易错点 |
|---|---|---|
| `HAL_DMA_Init()` | 时钟、通道、方向正确 | 同步配置；一般无需业务手动重复调用 |
| `HAL_DMA_Start()` | 源/目标地址和宽度明确 | 非阻塞；新手场景应优先外设封装 API |
| `HAL_DMA_Start_IT()` | NVIC/回调已配置 | 非阻塞；IRQHandler 未绑定正确句柄 |
| `HAL_DMA_IRQHandler()` | DMA IRQ 已启用 | 中断上下文；漏调用导致外设完成回调不进 |
| `HAL_DMA_Abort()` | 句柄有效 | 同步中止；未同步停止外设状态 |
| `HAL_DMA_GetState()` | DMA 初始化 | 非阻塞；只看 DMA，不看外设状态/错误 |
| `__HAL_DMA_GET_COUNTER()` | DMA 已配置 | 非阻塞宏；把剩余量当已接收量 |

多数 UART/ADC 场景不直接调用 `HAL_DMA_Start()`，而是调用 `HAL_UART_Receive_DMA()`、`HAL_UARTEx_ReceiveToIdle_DMA()` 或 `HAL_ADC_Start_DMA()`，由外设 HAL 管理 DMA 回调关系。

## USART

详见 [[USART与串口协议]]。

> 版本兼容提醒：`HAL_UARTEx_ReceiveToIdle_DMA()` 属于较新的 HAL 扩展接口，旧版 STM32CubeF1 / 本地 HAL 包可能不提供。实际使用前先在 `stm32f1xx_hal_uart.h` 中搜索该函数；如果没有，可改用 `HAL_UART_Receive_DMA()` + IDLE 中断手动处理，或使用中断接收状态机。

### API 速查

| API | 作用 | 常见调用位置 |
|---|---|---|
| `HAL_UART_Transmit()` | 轮询发送固定字节数 | 日志/短命令 |
| `HAL_UART_Receive()` | 轮询收满固定长度 | 最小验证 |
| `HAL_UART_Transmit_IT()` | 中断发送 | 短异步消息 |
| `HAL_UART_Receive_IT()` | 中断接收固定长度 | 单字节/短帧 |
| `HAL_UART_Transmit_DMA()` | DMA 发送 | 大块数据 |
| `HAL_UART_Receive_DMA()` | DMA 收固定长度 | 固定块接收 |
| `HAL_UARTEx_ReceiveToIdle_DMA()` | DMA 接收到 IDLE/满 | 不定长协议 |
| `HAL_UART_RxCpltCallback()` | 固定长度接收完成 | IT/DMA 固定长度接收 |
| `HAL_UARTEx_RxEventCallback()` | Receive-to-Idle 事件 | ReceiveToIdle 已启动 |
| `HAL_UART_ErrorCallback()` | UART 错误通知 | UART 中断链完整 |
| `HAL_UART_DMAStop()` | 停止 UART DMA | 错误/模式切换 |

### 使用注意

| API | 前置条件 | 阻塞 / 非阻塞与易错点 |
|---|---|---|
| `HAL_UART_Transmit()` | UART 已初始化 | 阻塞；Timeout 是等待上限；高频打印阻塞 |
| `HAL_UART_Receive()` | RX/帧格式正确 | 阻塞；不定长数据频繁超时；字符串未补 `\0` |
| `HAL_UART_Transmit_IT()` | UART NVIC 已开 | 非阻塞；buffer 完成前被修改 |
| `HAL_UART_Receive_IT()` | UART NVIC 已开 | 非阻塞；完成后忘记重启 |
| `HAL_UART_Transmit_DMA()` | TX DMA 已绑定 | 非阻塞；buffer 生命周期不足 |
| `HAL_UART_Receive_DMA()` | RX DMA 已绑定 | 非阻塞；未收满不回完成；Normal 后未重启 |
| `HAL_UARTEx_ReceiveToIdle_DMA()` | RX DMA/IDLE 链完整 | 非阻塞；IDLE 不等于完整帧 |
| `HAL_UART_RxCpltCallback()` | HAL IRQ/DMA 链完整 | 中断上下文；与 RxEvent 回调混淆 |
| `HAL_UARTEx_RxEventCallback()` | 使用回调参数 Size | 中断上下文；用 `strlen` 处理二进制；回调做解析重活 |
| `HAL_UART_ErrorCallback()` | 读取错误并设计恢复 | 中断上下文；只记录，不重建接收链 |
| `HAL_UART_DMAStop()` | UART 句柄有效 | 同步停止；未清协议和 buffer 状态就重启 |

## I2C

详见 [[I2C总线]]。

### API 速查

| API | 作用 | 常见调用位置 |
|---|---|---|
| `HAL_I2C_Master_Transmit()` | 发送命令/数据字节流 | OLED/自定义协议 |
| `HAL_I2C_Master_Receive()` | 接收字节流 | 简单协议 |
| `HAL_I2C_Mem_Write()` | 写寄存器 | 传感器配置 |
| `HAL_I2C_Mem_Read()` | 读寄存器 | ID/状态/原始值 |
| `HAL_I2C_IsDeviceReady()` | 探测 ACK | bring-up/地址扫描 |
| `HAL_I2C_GetState()` | 读 HAL 状态 | BUSY 诊断 |
| `HAL_I2C_GetError()` | 读错误位 | API 失败后 |

### 使用注意

| API | 前置条件 | 阻塞 / 非阻塞与易错点 |
|---|---|---|
| `HAL_I2C_Master_Transmit()` | 地址/上拉正确 | 阻塞；设备地址与寄存器地址混淆 |
| `HAL_I2C_Master_Receive()` | 设备已进入可读状态 | 阻塞；忽略前置命令和长度要求 |
| `HAL_I2C_Mem_Write()` | MemAddSize 正确 | 阻塞；地址宽度和字节序错误 |
| `HAL_I2C_Mem_Read()` | 寄存器模型已确认 | 阻塞；连续读长度/自动递增不符 |
| `HAL_I2C_IsDeviceReady()` | 总线电气正常 | 阻塞尝试；ACK 不代表初始化成功 |
| `HAL_I2C_GetState()` | 句柄有效 | 非阻塞；不量 SDA/SCL，只看软件状态 |
| `HAL_I2C_GetError()` | 保留失败上下文 | 非阻塞；忽略原 API 返回值 |

## SPI

详见 [[SPI总线与Flash]]。

### API 速查

| API | 作用 | 常见调用位置 |
|---|---|---|
| `HAL_SPI_Transmit()` | 发送命令/地址/数据 | SPI 事务 |
| `HAL_SPI_Receive()` | 接收数据 | 特定接收模式 |
| `HAL_SPI_TransmitReceive()` | 同步边发边收 | dummy 读/全双工 |
| `HAL_SPI_Transmit_DMA()` | DMA 发送 | 大块写入 |
| `HAL_SPI_Receive_DMA()` | DMA 接收 | 连续接收 |
| `HAL_SPI_GetState()` | 读 SPI HAL 状态 | 诊断/重启前 |
| `HAL_SPI_GetError()` | 读错误位 | API 失败后 |

### 使用注意

| API | 前置条件 | 阻塞 / 非阻塞与易错点 |
|---|---|---|
| `HAL_SPI_Transmit()` | CS 已拉低、模式正确 | 阻塞；CS 边界和 Size 单位错误 |
| `HAL_SPI_Receive()` | 主机能产生时钟 | 阻塞；忽略读操作仍需 SCK |
| `HAL_SPI_TransmitReceive()` | Tx/Rx 长度一致 | 阻塞；CPOL/CPHA、dummy 和 buffer 错 |
| `HAL_SPI_Transmit_DMA()` | TX DMA 已绑定 | 非阻塞；完成前拉高 CS/修改 buffer |
| `HAL_SPI_Receive_DMA()` | RX DMA 和时钟方案明确 | 非阻塞；只开 RX DMA 不等于协议完整 |
| `HAL_SPI_GetState()` | 句柄有效 | 非阻塞；HAL READY 不等于 Flash 非 BUSY |
| `HAL_SPI_GetError()` | 句柄有效 | 非阻塞；不检查 CS/波形只看错误码 |

W25Q64 读 ID、写使能、读状态、页写和擦除属于 Flash 命令序列，不是单个 HAL API。

## CAN

详见 [[CAN总线]]。

### API 速查

| API | 作用 | 常见调用位置 |
|---|---|---|
| `HAL_CAN_ConfigFilter()` | 配置 ID/Mask 与 FIFO | Start 前 |
| `HAL_CAN_Start()` | 启动 CAN 控制器 | 过滤器配置后 |
| `HAL_CAN_Stop()` | 停止 CAN | 重配置/停机 |
| `HAL_CAN_AddTxMessage()` | 放一帧进发送邮箱 | 业务发送 |
| `HAL_CAN_GetRxMessage()` | 从 FIFO 取一帧 | RX 回调 |
| `HAL_CAN_ActivateNotification()` | 激活 RX/错误通知 | Start 后 |
| `HAL_CAN_RxFifo0MsgPendingCallback()` | FIFO0 待取回调 | FIFO0 通知已开 |
| `HAL_CAN_GetError()` | 读取 CAN 错误位 | 发送/总线故障诊断 |

### 使用注意

| API | 前置条件 | 阻塞 / 非阻塞与易错点 |
|---|---|---|
| `HAL_CAN_ConfigFilter()` | Filter bank/模式明确 | 同步配置；过滤器过窄导致收不到 |
| `HAL_CAN_Start()` | 位时序/收发器就绪 | 非阻塞；只 Init 未 Start |
| `HAL_CAN_Stop()` | CAN 已启动 | 同步停止；运行态直接改关键配置 |
| `HAL_CAN_AddTxMessage()` | CAN 已启动、有邮箱 | 非阻塞排队；`HAL_OK` 不代表已获总线 ACK |
| `HAL_CAN_GetRxMessage()` | FIFO 有消息 | 同步取出；回调不取帧导致 FIFO 堆积 |
| `HAL_CAN_ActivateNotification()` | NVIC 已配置 | 非阻塞；只开 NVIC，未开 HAL 通知 |
| `HAL_CAN_RxFifo0MsgPendingCallback()` | 回调内尽快取帧 | 中断上下文；长业务阻塞其它 CAN 事件 |
| `HAL_CAN_GetError()` | 句柄有效 | 非阻塞；不查 ACK/终端/位时序只重发 |

## 常见选择题

| 我想做什么 | 应该优先用什么 |
|---|---|
| 控制普通 LED/CS/使能脚 | `HAL_GPIO_WritePin()` |
| GPIO 边沿触发业务 | EXTI + `HAL_GPIO_EXTI_Callback()`，回调只置标志 |
| 周期任务 | `HAL_TIM_Base_Start_IT()` + `HAL_TIM_PeriodElapsedCallback()` |
| 改 PWM 占空比 | `__HAL_TIM_SET_COMPARE()` |
| 测输入脉宽 | `HAL_TIM_IC_Start_IT()` + `HAL_TIM_ReadCapturedValue()` |
| 读取编码器位置/速度 | `HAL_TIM_Encoder_Start()` + 周期读取 CNT |
| ADC 单通道最小读取 | `Start` + `PollForConversion` + `GetValue` |
| ADC 连续采多个通道 | `HAL_ADC_Start_DMA()` |
| 串口打印调试 | `HAL_UART_Transmit()` 或 `printf` 重定向 |
| 串口接收固定短数据 | `HAL_UART_Receive_IT()` |
| 串口接收不定长命令 | `HAL_UARTEx_ReceiveToIdle_DMA()` + ring buffer |
| 读 I2C 传感器寄存器 | `HAL_I2C_Mem_Read()` |
| SPI Flash 读 ID | 手动 CS + 发送命令 + `HAL_SPI_TransmitReceive()` |
| CAN 中断接收 | 配过滤器、Start、ActivateNotification、FIFO 回调中 GetRxMessage |

## 常见误用

| 误用 | 后果 | 正确做法 |
|---|---|---|
| 以为 CubeMX 配了就自动运行 | TIM/PWM/ADC/CAN/接收链没有启动 | 找到对应 `Start/Receive/ActivateNotification` |
| 忘记调用 Start 类函数 | 无波形、无回调、无采样 | 检查初始化后的启动顺序和返回值 |
| 中断接收后忘记重新启动 | UART 只收一次 | 在合适位置再次调用 `HAL_UART_Receive_IT()` |
| DMA buffer 使用局部变量 | 函数返回后 DMA 写入无效内存 | 使用全局、静态或有明确生命周期的 buffer |
| I2C 地址没按 HAL 约定处理 | 一直 NACK | 在驱动中统一 7 位地址与传参表达，不到处临时移位 |
| SPI 读数据没产生 dummy/时钟 | 读回全 0/全 1/错位 | 用明确事务和 `TransmitReceive()` 产生 SCK |
| CAN 过滤器没配或过窄 | 总线有数据但 FIFO 无消息 | Start 前配置过滤器，调试时先放宽 |
| 回调里做大量打印、协议解析、延时 | 丢中断、抖动、死锁 | 回调只搬数据/置标志，业务放主循环/任务 |
| 不检查 `HAL_StatusTypeDef` | HAL_BUSY/TIMEOUT 被吞掉 | 每一步记录返回值、State 和 Error |
| 把模块协议当 HAL 功能 | 误以为一个 API 能完成 Flash 擦写或传感器换算 | HAL 只搬字节，模块命令和算法按手册实现 |

## 典型通用调用顺序

```text
CubeMX 生成 MX_xxx_Init()
-> main() 中按依赖顺序调用 Init
-> 检查 HAL Start/Receive/Config 返回值
-> IRQHandler/DMA Handler 进入 HAL 驱动
-> HAL Callback 复制数据或设置事件
-> 主循环/App 完成协议、换算和业务
-> 错误时 Stop/Abort，清状态后按顺序重启
```

## 与其它章节的关系

- 硬件模式和调用细节回到对应的 Core、Peripherals、Communication 文件。
- DMA buffer、回调和错误恢复见 [[DMA与缓冲区]]、[[通用调试方法]]。
- 第三方模块的寄存器和算法不在本表，见 Modules 及其官方资料。

## 复习检查清单

- [ ] 能区分 Init、Start、阻塞操作、IT/DMA 启动和 Callback。
- [ ] 能说明 `_IT`、`_DMA` 对 NVIC、buffer 生命周期和回调的要求。
- [ ] 能为 GPIO、TIM、ADC、UART、I2C、SPI、CAN 写出典型启动顺序。
- [ ] 能检查 `HAL_StatusTypeDef`、State 和 Error，而不是只看业务结果。
- [ ] 能区分 HAL API、CMSIS/内核对象和第三方模块协议。
- [ ] 能在回调中区分外设实例并把重业务移到主循环/任务。
