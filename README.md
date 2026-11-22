# STM32F407 双路步进电机驱动（TIM6 中断节拍）

四相步进电机（IN1~IN4 + ULN2003 类驱动板）的双路驱动，采用 **TIM6 更新中断统一产生步进节拍**，`Motor_Set()` 只登记步数与方向即返回，电机转动完全由中断推进，**不阻塞任务**。支持两路独立控制、动态调速与全局软件急停。

## 引脚

| 电机 | 引脚 | 说明 |
|---|---|---|
| 电机 1 | PC0 ~ PC3 | IN1 ~ IN4 推挽输出 |
| 电机 2 | PC4 ~ PC7 | IN1 ~ IN4 推挽输出 |

## 文件结构

```
src/   step_motor.c
inc/   step_motor.h
port/  system.h        # 位带宏 PCout()
```

## API

```c
void STEP_Motor_Init(void);                                  // 电机 1 的 IO 初始化
void STEP_Motor2_Init(void);                                 // 电机 2 的 IO 初始化
void Step_Motor_Run(u8 step, u8 dir, u8 speed, u16 angle, u8 sta);   // 阻塞式（保留，不推荐）

/* 非阻塞接口，推荐使用 */
void Motor_Set(uint8_t motor_id, uint8_t dir, uint16_t step_cnt);    // motor_id: 1/2，dir: 0 逆时针 / 1 顺时针
void Motor_EmergencyStop(void);                                       // 立即停止两路并清零剩余步数
void TIM6_SetParam(uint16_t prescaler, uint16_t period);              // 运行中动态改速度
extern uint8_t motor_emergency_stop;
```

在工程中的实际用法（`motor.c` 任务层）：

```c
STEP_Motor_Init();
STEP_Motor2_Init();
TIM6_Init(83, 999);            // 168MHz/(83+1)/(999+1) ≈ 2kHz 节拍
Motor_Set(1, 1, 512);          // 电机 1 顺时针转一圈（512 步/圈）
Motor_Set(2, 0, 256);          // 电机 2 逆时针转半圈
```

## 设计要点

- **8 拍相序表**：`motor_phase_table[8][4]` 用查表替代 switch-case，中断内只做数组索引与 IO 写入，执行时间确定；
- **两路状态独立**：每路各自维护 `remain_step`、`dir`、`phase_idx`，互不干扰，同一节拍内先后推进；
- **非阻塞**：应用层只写状态变量，转动由 `TIM6_DAC_IRQHandler` 驱动，任务不会被电机拖住；
- **调速**：`TIM6_SetParam` 用 `TIM_PSCReloadMode_Immediate` 立即生效预分频，实现运行中变速；
- **急停**：`Motor_EmergencyStop` 置标志并清零剩余步数、立即拉低 8 个 IO，中断检测到标志直接返回。

## 转速换算

节拍频率 `f = 168MHz / (prescaler+1) / (period+1)`，8 拍模式下 512 步/圈，则转速 `rpm = f * 60 / 512 ≈ f / 8.53`。例如 2kHz 节拍约 234 rpm。

## 依赖

- STM32F4xx 标准外设库（`stm32f4xx_tim.h` / `stm32f4xx_gpio.h`）
- `port/system.h`（位带 IO 宏）
- `delay_ms`（仅阻塞式 `Step_Motor_Run` 使用）

## 移植步骤

1. 修改 `inc/step_motor.h` 中的 `MOTOR_INx_PORT/PIN/RCC` 宏；
2. 若用 TIM7 或其他定时器，替换 `TIM6_Init` 与中断向量名（`TIM6_DAC_IRQn` → 目标通道）；
3. 注意 `TIM6_Init` 抢占优先级设为 2，与工程其他中断的相对优先级要整体规划，避免节拍被高优先级中断拖延而抖动；
4. 源文件为 **UTF-8**，Keil 需设置 Encoding 为 UTF-8。

## 已知限制与可改进点

- **无加减速曲线**：固定节拍直接启停，转速突变时易丢步/失步，工业场景应加梯形或 S 曲线加减速；
- 中断内直接操作位带 IO，多路电机同时动作时中断执行时间随路数线性增长，节拍上限约几千 Hz；
- **无位置反馈**：开环步数控制，丢步无法自知，建议加编码器或霍尔做闭环校验；
- 全局变量在中断与任务间共享，`remain_step` 等为非原子访问，多任务同时下发指令时需加临界区保护；
- 阻塞式 `Step_Motor_Run` 与中断接口并存，容易误用，建议后续删除。

## License

MIT
