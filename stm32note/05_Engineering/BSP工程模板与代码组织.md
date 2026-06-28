---
type: engineering
status: refactor
course: stm32-f103-hal
source:
  - local-notes
tags:
  - stm32
  - bsp
  - architecture
---
# BSP 工程模板与代码组织

> 能力定位：让 CubeMX 生成代码、板级驱动和业务逻辑各自有清晰边界。
>
> 原始来源：[[09_电机与工程模板]]。

## 推荐分层

| 位置 | 职责 |
|---|---|
| `Core/Src/main.c` | 初始化顺序、主循环和高层调度 |
| `Core/Src/*.c` | CubeMX 生成的外设初始化 |
| `BSP/xxx.c` | 模块驱动实现 |
| `BSP/xxx.h` | 对外 API、结构体和必要宏 |
| `App/` | 业务状态机和项目逻辑，可按项目扩展 |

## 工程操作

- 手动文件加入 MDK Group。
- 头文件目录加入 Include Path。
- 用户代码放 USER CODE 区域或独立文件。
- 重新生成前先看 Git diff；重新生成后检查手动分组和路径。
- `main.c` 保持为流程入口，不堆积模块细节。

课程最终模板目录需要视频核对；本文件提供的是可维护性原则，不是唯一目录标准。

## 关联知识

- [[工具链与开发环境]]
- [[HAL_API速查表]]
- [[通用调试方法]]

---

## 本节目标

- 能解释为什么 `main.c` 不应承载全部驱动和业务逻辑。
- 能按 Driver / BSP / App 划分职责和依赖方向。
- 能安全使用 CubeMX 重新生成代码，不丢用户实现。

## 知识地图

| 层 | 职责 | 不应承担 |
|---|---|---|
| HAL/Driver | 芯片外设寄存器和 HAL 句柄 | 产品业务规则 |
| BSP | 板级引脚、模块驱动和统一接口 | 页面/协议状态机 |
| App | 采集、控制、通信等业务流程 | 直接散落寄存器操作 |
| main | 初始化顺序、调度入口 | 大量模块细节 |

## 本质理解

分层的目的不是增加文件，而是控制变化范围：换引脚只改 BSP，换传感器驱动不改业务，CubeMX 重新生成不覆盖独立模块。依赖应从 App 指向 BSP，再指向 HAL，避免底层反向调用业务。

## 示例目录树

```text
Core/Inc, Core/Src      CubeMX 与系统入口
Drivers/                CMSIS 和 HAL
BSP/Inc, BSP/Src        板级模块驱动
App/Inc, App/Src        业务状态机
Middlewares/            可选协议栈/算法库
```

## `.c/.h` 分工与命名

- `.h` 公开最小 API、类型和必要配置，不暴露内部静态状态。
- `.c` 保存私有函数、缓冲区和具体 HAL 调用。
- 接口使用模块前缀，如 `Motor_Init`、`OLED_Clear`。
- 初始化、周期任务、回调入口分开，如 `X_Init`、`X_Task`、`X_OnRx`。

## MDK / CubeMX 配置要点

1. 把 BSP/App 文件加入对应 MDK Group。
2. 把 `BSP/Inc`、`App/Inc` 加入 Include Path。
3. `main.c` 中只在 USER CODE 区调用自定义初始化和任务。
4. 生成前提交或查看 Git diff；生成后检查文件分组和路径。
5. 不修改 HAL 库文件来实现业务功能。

## 常用 HAL / 工程接口

### API 速查

| 接口 | 类型 | 作用 | 推荐位置 |
|---|---|---|---|
| `MX_xxx_Init()` | CubeMX 生成函数，非 HAL API | 应用 `.ioc` 外设配置 | `main()` 初始化阶段 |
| `HAL_TIM_Base_Start()`、`HAL_UART_Receive_IT()` 等 | HAL API | 让已配置外设进入运行态 | BSP/主初始化 |
| `Module_Init()` | 项目自定义 BSP 接口 | 设备 ID、最少寄存器和内部状态初始化 | BSP |
| `Module_Read/Write()` | 项目自定义 BSP 接口 | 提供原始值/工程值或控制动作 | BSP 对外 API |
| `App_Init()/App_Task()` | 项目自定义 App 接口 | 业务状态机和周期调度 | App |
| `HAL_UART_RxCpltCallback()` 等具体弱回调 | HAL 弱回调 | 接收中断/DMA事件并转交 | 回调适配层 |

### 使用注意

| 接口 | 前置条件 | 返回/阻塞 | 常见坑 |
|---|---|---|---|
| `MX_xxx_Init()` | 时钟和生成代码完整 | 通常 `void`；同步配置 | 误以为 Init 后外设会自动 Start |
| `HAL_TIM_Base_Start()`、`HAL_UART_Receive_IT()` 等 | `MX_xxx_Init()` 已完成 | 多数返回 `HAL_StatusTypeDef` | 忽略返回值和启动顺序 |
| `Module_Init()` | 底层 HAL 外设可用 | 建议返回模块状态 | 直接暴露 HAL 句柄和模块细节给 App |
| `Module_Read/Write()` | 模块已初始化 | 明确阻塞性和错误码 | 函数名不表达单位、长度和失败方式 |
| `App_Init()/App_Task()` | BSP 已准备 | 应尽量非阻塞 | App 直接散落 HAL 调用，失去分层 |
| `HAL_UART_RxCpltCallback()` 等具体弱回调 | IRQ/DMA 调用链完整 | 中断上下文 | 回调直接做显示、协议和长延时 |

### 典型调用顺序

```text
HAL_Init() / SystemClock_Config()
-> MX_GPIO/DMA/外设_Init()
-> BSP_Init() 调 HAL Start/Receive 和模块最少配置
-> App_Init()
-> while (1) 中 App_Task()
-> HAL Callback 只向 BSP/App 投递事件
```

### 什么时候用哪一层？

| 需求 | 推荐位置 |
|---|---|
| 芯片外设启动/传输 | BSP 内调用 HAL API |
| 模块寄存器和原始数据 | BSP Driver |
| 数据换算/业务规则 | App 或独立 Service |
| 中断事件入口 | HAL Callback -> 轻量适配 -> 队列/标志 |

## 最小框架

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_USART1_UART_Init();

    BSP_Init();
    App_Init();

    while (1)
    {
        App_Task();
    }
}
```

## 调试方法

```text
编译文件是否加入工程
-> Include Path
-> 头文件声明/实现是否一致
-> 初始化顺序
-> HAL 句柄依赖
-> BSP 单测
-> App 状态机
```

## 常见坑

| 现象 | 原因 | 处理 |
|---|---|---|
| 找不到头文件 | Include Path 缺失 | 加入 Inc 目录 |
| Undefined symbol | `.c` 未加入 Group | 添加源文件并重新 Build |
| CubeMX 后代码丢失 | 写在保护区外 | 移到 USER CODE/BSP |
| 循环依赖 | BSP 反向 include App | 重新定义依赖方向 |
| main 越来越大 | 未抽模块接口 | 用 Init/Task/Callback 分层 |

## 与其它章节的关系

- 常用外设接口见 [[HAL_API速查表]]。
- 调试流程见 [[通用调试方法]]。
- 综合应用参考 [[传感器采集与显示]]、[[串口协议解析]]。

## 复习检查清单

- [ ] 能画出 Driver、BSP、App、main 的依赖方向。
- [ ] 能创建一个 `.c/.h` 模块并加入 MDK。
- [ ] 能说明 USER CODE 区和独立文件的保留策略。
- [ ] 能把模块初始化、业务任务和回调分开。
- [ ] 能用 Git diff 检查 CubeMX 重新生成影响。
- [ ] 能避免循环 include 和全局变量滥用。
