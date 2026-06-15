---
title: "自瞄步战车系统方案"
date: 2025-12-20
draft: false
description: "基于H750XBH6 + ESP32-S3 CAM的自瞄+全向移动步战车"
cover: "/Bzone-blog/images/combat-vehicle-cover.png"
tags: ["STM32", "H750", "FreeRTOS", "ESP32", "PID", "麦轮", "自瞄", "I2C", "W5500"]
categories: ["嵌入式"]
---

本项目是一套基于 **H750XBH6 + ESP S3 CAM 的自瞄 + 全向移动的步战车** 开发的竞赛级车控方案。本文分为"遥控 + 自瞄"两个主要部分讲解。

硬件部分：底盘 + 遥控部分使用的是成品方案，主控和舵机云台部分是自主设计的，当时刚好在学 H750+LWIP，想着就地取材节省经费，但如果你不善折腾，那么强烈推荐你买塔克的 F407 主控的全套方案（H7 太难学了！！！）

![实物图1](/Bzone-blog/images/file-20260602151713773.png)
![实物图2](/Bzone-blog/images/file-20260607044520054.png)

代码方面：车控部分直接用了麦轮运动学解算，PID 部分使用江协科技的规范重写成增量式 PI（一是为了通用和兼容，二是减少认知成本），同时加入了衰减刹车逻辑、5 点滑动窗口编码器滤波、I2C 退避策略、舵机轨迹步进等工程化细节，整体采用 BSP 分层管理。
![代码架构](/Bzone-blog/images/file-20260614210358892.png)

## 项目链接

- GitHub： https://github.com/BiaoZai1214/AutoAimCombatBot
- 百度云： https://pan.baidu.com/s/1B6TsMMYSHaVEvOWRM5zCJg?pwd=xti5

个人能力有限，代码可能并不完美，欢迎各位朋友的学习和指正。

---

## 一、系统架构

本系统采用 **两层架构**：底层实时控制由 STM32H750 负责，视觉识别由 ESP32-S3 CAM 负责。两片 MCU 通过 I2C 总线通信，各司其职。

```
┌─────────────────────────────────────────────────┐
│              应用层 (App/)                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│  │ ui.c     │ │robot.c   │ │ ps2_control.c    │ │
│  │主任务调度│ │运动学+PID│ │ 遥控器映射       │ │
│  └────┬─────┘ └────┬─────┘ └────────┬─────────┘ │
│       │            │                │           │
│  ┌────┴─────┐ ┌────┴─────┐ ┌───────┴─────────┐ │
│  │tracking.c│ │ pid.c/h  │ │                  │ │
│  │视觉追踪  │ │ 增量PID  │ │                  │ │
│  └────┬─────┘ └────┬─────┘ └────────┬─────────┘ │
├───────┼────────────┼────────────────┼───────────┤
│              驱动层 (BSP/)                        │
│  ┌────┴─────┐ ┌────┴─────┐ ┌───────┴─────────┐ │
│  │esp_s3_i2c│ │tb6612    │ │ ps2             │ │
│  │servo     │ │encoder   │ │ lcd             │ │
│  └──────────┘ └──────────┘ └─────────────────┘ │
├─────────────────────────────────────────────────┤
│              硬件层 (Core/)                       │
│  STM32H750XBH6 + FreeRTOS + HAL库               │
└─────────────────────────────────────────────────┘
```

**数据流：**
- **遥控模式**：PS2手柄 → SPI2 → ps2_control.c → robot.c（运动学+增量式PID）→ tb6612.c → 电机
- **自瞄模式**：ESP32 视觉识别+P控制 → I2C(0x01, 5字节角度) → tracking.c → servo.c → 舵机
- **反馈回路**：编码器 → TIM2-5正交编码 → encoder.c → robot.c（速度闭环）

核心设计理念：**算力分离** — ESP32 跑视觉+PID，只把最终的角度指令通过 I2C 发过来；H750 只管实时控制，不做任何视觉运算。这样 ESP32 追踪卡顿不影响底盘安全，H750 的控制循环也不会被图像处理打断。

### main() 启动链

```
MPU_Config → 使能Cache → HAL_Init → 400MHz时钟 → GPIO/SPI/TIM/I2C/ADC初始化
  → 用户初始化(电机GPIO + LCD + 舵机 + 底盘参数 + I2C + 追踪)
  → freertos_start() → 创建3个任务 → 启动调度器，不再返回
```

### FreeRTOS 任务划分

| 任务 | 栈 | 优先级 | 周期 | 职责 |
|------|------|--------|------|------|
| Control_Task | 512 words | idle+2 | 20ms | 自检→PS2扫描→运动学PID→追踪更新→LCD刷新 |
| W5500_Task | 512 words | idle+2 | 按需 | 以太网初始化 → HTTP Server 主循环 |
| LED_Task | 128 words | idle+1 | 动态 | 心跳灯 + 电池电压每分钟打印 |

**为什么只用一个控制任务？** 底盘、PS2、舵机、LCD 共享同一个 20ms 节奏。拆成多个任务需要互斥锁管理 SPI/I2C 总线竞争，反而增加复杂度和延迟。实测单次循环 < 5ms，在 20ms 窗口内有余量。LED 灯还兼职了调试指示——快闪=ESP32 离线，慢闪=I2C 通信正常，看一眼就知道自瞄状态。

---

## 二、底盘部分

### 2.1 主控：STM32H750XBH6

选用 H750 而非常见的 F407，主要原因是手头刚好有这块板子。H7 系列相比 F4 系列有几个关键差异：

| 特性 | STM32F407 | STM32H750 |
|------|-----------|-----------|
| 主频 | 168MHz | 400MHz |
| 内核 | Cortex-M4 | Cortex-M7 |
| MPU | 无 | 有（4区隔离） |
| D-Cache | 无 | 有（需配合MPU管理） |
| 定时器时钟 | 84MHz | 200MHz |

**MPU 配置**是 H7 开发的核心难点。H750 开启 D-Cache 后，外设寄存器访问必须通过 MPU 配置为不可缓存区域，否则读写外设会返回陈旧数据。本项目配置了 4 个 MPU 区域：Region0 覆盖 D2 SRAM（DMA 缓冲区），Region1-3 覆盖 TIM1 寄存器、GPIO 寄存器、APB 外设区。

**最典型的踩坑**：编码器定时器（TIM2-5）在 APB1 总线上，如果没被 MPU Region3 覆盖，D-Cache 会一直返回上一帧的旧计数值，PID 拿着假数据算，车就乱跑了。定位方法很简单——在 `Encoder_GetCounter()` 中连续两次读到一样的值（车明明在动），基本就是 Cache 没穿透。

**时钟链**：HSE 25MHz → PLL(/5, ×160, /2) → 400MHz SYSCLK → APB1/2 各 /2 → 200MHz 外设时钟。HAL 时基用 TIM6 而非 SysTick（SysTick 被 FreeRTOS 占了）。

### 2.2 电机驱动：TB6612

TB6612 本身是控制双路电机的，但是这块板子是级联两块 TB6612 的思路电机板。

| 电机 | 位置 | PWM通道 | IN1 | IN2 |
|------|------|---------|-----|-----|
| M1 | 左前 | TIM1_CH1 | PD3 | PI0 |
| M2 | 右前 | TIM1_CH2 | PG12 | PG13 |
| M3 | 左后 | TIM1_CH3 | PB11 | PB10 |
| M4 | 右后 | TIM1_CH4 | PH14 | PC11 |

TB6612 真值表：
- IN1=HIGH, IN2=LOW → 正转
- IN1=LOW, IN2=HIGH → 反转
- IN1=LOW, IN2=LOW → 制动

TIM1 产生 4 路 PWM（PSC=4, ARR=4199），频率约 9.5kHz，分辨率 4200 级。**STBY 引脚**（PC7）两块 TB6612 共用，初始化时必须先拉高 STBY 再启动 PWM，否则电机不响应。

### 2.3 麦轮运动学

采用 **O 型麦克纳姆轮**布局，四轮对称安装。运动学核心参数：

```
轮径: 80mm        轮距(WHEEL_BASE): 182mm
轴距(ACLE_BASE): 124mm    有效半径(MEC_RADIUS): 153mm
编码器分辨率: 1560 (13线 × 30减速比 × 4倍频)
```

**逆运动学**（底盘速度 → 四轮转速）：
```
M1(左前) = vx - vy - vw × MEC_RADIUS
M2(右前) = vx + vy + vw × MEC_RADIUS
M3(左后) = vx + vy - vw × MEC_RADIUS
M4(右后) = vx - vy + vw × MEC_RADIUS
```

**正运动学**（四轮转速 → 底盘速度，用于反馈）：
```
vx = (M1 + M2 + M3 + M4) / 4
vy = (-M1 + M2 + M3 - M4) / 4
vw = (-M1 + M2 - M3 + M4) / (4 × MEC_RADIUS)
```

**动手验证**：纯前进 → 四轮全是 +vx；纯右移 → A=-vy, B=+vy, C=+vy, D=-vy（对角配合）；纯旋转 → A/C=-R·vw, B/D=+R·vw（左侧反右侧正）。

### 2.4 速度环 PID 控制

将 PID 抽成了独立通用模块 `App/pid.c/h`，沿用了江协科技的规范——统一接口、减少认知成本。底盘和追踪层可以共用同一套 PID 代码。

采用**增量式 PID**，每 20ms 执行一次完整的控制循环：

```
读编码器 → 滑动窗口滤波 → 正运动学 → 速度限幅 → 逆运动学 → PID计算 → 刹车判断 → 写电机
```

**为什么用增量式？** 输出在上一次基础上累加，天然平滑无跳变；切模式时 `PID_Reset()` 清零就好，不会有积分饱和问题。增量式 PID 的核心公式：

```c
// App/pid.c — 增量式PID
pid->out += kp * (err - err_last)      // 比例项：误差变化量
          + ki * err                     // 积分项：当前误差
          + kd * (err - 2*err_last + err_prev);  // 微分项：误差二阶差分
// 输出限幅 ±4199（对应 TIM1 ARR）
```

**参数选择思路**（KP=1000, KI=800, KD=0）：

| 参数 | 值 | 作用 | 选择理由 |
|------|-----|------|----------|
| KP | 1000 | 比例响应 | 1m/s 误差产生约 50 PWM 增量，满速 PWM≈3500，比例合理 |
| KI | 800 | 消除稳态误差 | 补掉纯 PD 的稳态误差，低速跟踪更准 |
| KD | 0 | 阻尼（未启用） | 增量式本身有平滑效果 + 衰减刹车覆盖急停需求，加 D 项反而让加速变肉 |

> 注意：虽然代码中变量名为 `PID_t`，但 KD=0 实际上是 **增量式 PI** 控制器。保留 KD 参数是为了方便后续调试——如果发现急停过冲严重，可以尝试加 KD=200~500。

**衰减刹车**（`SpeedLoop_Update` 中实现）：

```c
// App/robot.c — 当 TG=0 时跳过 PID，多级暴力减速
if (R_Wheel[i].TG == 0.0f) {
    if      (pwm_out[i] >  400) pwm_out[i] -= 300;   // 大输出：每帧减 300
    else if (pwm_out[i] < -400) pwm_out[i] += 300;
    else if (pwm_out[i] >  100) pwm_out[i] -= 80;    // 中输出：每帧减 80
    else if (pwm_out[i] < -100) pwm_out[i] += 80;
    else                        pwm_out[i]  = 0;      // 小输出：直接归零
    PID_Reset(&speed_pid[i]);  // 清零误差历史，防止积分累积
}
```

**为什么需要衰减刹车？** PID 自己收敛需要 10+ 帧（200ms+），而衰减刹车 2-3 帧（40-60ms）就能停住。多级衰减的设计逻辑：大输出时快速减速（避免惯性冲过头），小输出时精细减速（避免反向抖动），最终归零。

**编码器滤波**（5 点滑动窗口）：

```c
// App/robot.c — ReadEncoders() 中的滑动平均
static int16_t enc_buf[4][5] = {{0}};  // 4 路编码器 × 5 点窗口
static uint8_t enc_idx = 0;

for (int i = 0; i < 4; i++) {
    enc_buf[i][enc_idx] = enc[i];
    int32_t sum = 0;
    for (int k = 0; k < 5; k++) sum += enc_buf[i][k];
    R_Wheel[i].RT = (float)sum / 5.0f * WHEEL_SCALE * PID_RATE_HZ;
}
enc_idx = (enc_idx + 1) % 5;
```

每 20ms 读取计数器并清零（差分测速），通过 5 点滑动窗口平滑。低速时每帧只几个脉冲，不滤波的话噪声会让 PID 输出抖动。窗口 5 是实测折中——太小噪声大，太大响应慢。`WHEEL_SCALE = π × 0.080 / 1560 ≈ 0.000161 m/pulse`，乘以 `PID_RATE_HZ(50)` 换算成 m/s。

### 2.5 遥控器：PS2 无线手柄

PS2 手柄通过 SPI2 接口通信（PI1/PI2/PI3 + PD4，Mode3 LSB，约 780kHz）。数据校验做了一层保护：全 0 或全 0xFF 判定为异常，直接刹车。

```c
// App/ps2_control.c — 数据有效性校验
static int8_t PS2_IsValid(const PS2_Joystick_t *js) {
    uint8_t all0  = (js->RJoy_LR == 0 && js->RJoy_UD == 0 && ...);
    uint8_t allFF = (js->RJoy_LR == 0xFF && js->RJoy_UD == 0xFF && ...);
    return (js->mode == 0x73) && !all0 && !allFF;  // 模式 0x73 = 按键+摇杆模式
}
```

**操作映射：**

| 操作 | 功能 | 映射方式 |
|------|------|----------|
| 左摇杆上下 | 前后行走 | 摇杆值 × speed倍率 → vx (mm/s) |
| 左摇杆左右 | 左右横移 | 摇杆值 × speed倍率 × 1.3补偿 → vy |
| 右摇杆左右 | 原地旋转 | 摇杆值 × speed倍率 × 4 → vw |
| DPAD | 固定速度移动 | 固定值 128 × speed倍率 |
| L1/R1 | 舵机 TILT 上/下 | 每按 12°，3帧消抖 |
| L2/R2 | 舵机 PAN 左/右 | 每按 12°，3帧消抖 |
| SELECT | 切换手动/自瞄 | 下降沿触发，300ms冷却防连发 |

摇杆死区 ±6（代码中 `DEADZONE=6`），防止漂移。横向补偿 1.3x 是因为麦轮侧移效率比直行低约 30%。SELECT 用下降沿触发而非电平触发，配合 300ms 冷却，彻底根除了"按住不放反复跳模式"的问题。

**摇杆映射的数学**：

```c
// 左摇杆: 原始值 0x00(前/左) ~ 0x80(中) ~ 0xFF(后/右)
int16_t rx = (int16_t)0x80 - js->LJoy_UD;   // 前推为正，范围 -128~+127
int16_t ry = (int16_t)js->LJoy_LR - 0x80;   // 右推为正，范围 -128~+127
// 死区过滤
if (rx > -DEADZONE && rx < DEADZONE) rx = 0;
if (ry > -DEADZONE && ry < DEADZONE) ry = 0;
// 最终目标速度
Robot_SetTarget(g_speed * rx,           // vx: ±12×127 ≈ ±1524 mm/s
                g_speed * -ry * 13/10,  // vy: 横向 1.3x 补偿
                4 * g_speed * -rw);     // vw: 旋转 4x 倍率
```

**舵机手动控制的消抖机制**：

```c
// App/ps2_control.c — 4 按键独立消抖
typedef struct { uint8_t count, state, lock; } Debounce_t;
static Debounce_t db[4] = {{0}};
#define DB_THRESH 3  // 3 帧消抖 = 60ms

// 状态机: count 达到阈值才翻转 state，lock 防止按住连续触发
// 每次触发: pan += ±12° 或 tilt += ±12°
// 范围限幅: ±900 (即 ±90°)
```

### 2.6 W5500 以太网与 HTTP Server

项目集成了 W5500 以太网模块，通过 SPI4 连接，运行一个轻量级 HTTP Server 用于远程监控。

**网络配置**：

| 参数 | 值 |
|------|-----|
| IP | 192.168.155.100 |
| 子网掩码 | 255.255.255.0 |
| 网关 | 192.168.155.1 |
| MAC | 6E:78:82:8C:96:A0 |
| 端口 | 80 |

**W5500 初始化流程**（`BSP/w5500/eth.c`）：

```c
hardSPI_Init();          // SPI4 硬件初始化
user_register_function(); // 注册 SPI 读写回调
ETH_Reset();              // 硬件复位 W5500
setMR(MR_RST);            // 软复位
ver = getVERSIONR();      // 读版本号，应为 0x04
wizchip_init(txsize, rxsize);  // 初始化 Socket 缓冲区（每 Socket 2KB TX/RX）
setSHAR(mac); setSIPR(ip); setSUBR(submask); setGAR(gateway);
wizphy_setphyconf(&phyconf);   // 100M 全双工 自协商
```

**HTTP Server 工作模式**（`BSP/w5500/http_server.c`）：

HTTP Server 运行在独立的 FreeRTOS 任务中，采用"单次连接"模式——处理完一个 HTTP 请求后关闭 Socket，下次循环重新 listen。返回一个 HTML 页面，显示网络状态和系统信息（IP、MAC、PHY 链路状态、芯片型号、主频、RTOS 版本）。

```c
// W5500_Task 主循环
for (;;) {
    HTTP_Server_Run();  // 内部: socket → listen → accept → recv → send → close
}
```

**为什么用 W5500 而不是 lwIP？** H750 虽然自带 ETH MAC，但本项目的编码器用了 TIM2-5，与 ETH 的 DMA 缓冲区在 D2 SRAM 上有地址冲突。W5500 自带 MAC+PHY+Socket 缓冲区，SPI 接口简单可靠，不占用 D2 SRAM。这就是"速度换空间"的取舍——SPI 带宽不如 ETH MAC，但对于状态监控足够了。

### 2.7 LCD 显示系统

项目使用 ST7789 240×320 RGB565 LCD，通过 SPI5 连接，采用**帧缓冲**架构。

**帧缓冲**：在 RAM 中维护一整帧 `uint16_t LCD_FrameBuf[240×320]`，所有绘图操作先写缓冲区，最后通过 `LCD_Update()` 一次性 DMA 刷屏。好处是避免逐像素 SPI 传输的撕裂感，坏处是占用 150KB RAM（240×320×2 字节）。

**LCD 刷新频率**：Control_Task 中每 200ms 刷新一次（而非每 20ms），避免 SPI5 总线占用过高影响其他外设。

```c
// Core/Src/freertos.c — LCD 刷新节流
if (HAL_GetTick() - lcd_last >= 200) {
    lcd_last = HAL_GetTick();
    UI_ShowTarget();  // 刷新追踪界面
}
```

**显示内容**：追踪模式下显示 I2C 连接状态（OK/FAIL）、目标锁定状态（LOCKED/SEARCHING）、PAN/TILT 角度、追踪开关状态。颜色编码：绿色=正常，红色=异常，黄色=搜索中，青色=角度值。

### 2.8 LED 状态指示与电池监测

LED 任务除了心跳指示外，还兼任了**调试指示器**和**电池监测**：

```c
// Core/Src/freertos.c — LED 状态指示
if (!g_tracking.enabled) {
    // 追踪关闭 → LED 常灭
    HAL_GPIO_WritePin(LED_B_GPIO_Port, LED_B_Pin, GPIO_PIN_SET);
} else {
    HAL_GPIO_TogglePin(LED_B_GPIO_Port, LED_B_Pin);
    if      (g_tracking.i2c_ok == 2) dly = 2000;  // I2C 正常 → 慢闪 2s
    else if (g_tracking.i2c_ok == 0) dly = 200;   // I2C 失败 → 快闪 200ms
}
```

**电池电压检测**：每 60 秒读取 ADC1_IN7（PA7），通过 18:1 电阻分压换算实际电压。

```c
uint16_t adc = ADC_ReadBattery();
float volt = (float)adc * 3.3f / 4095.0f * 18.0f;  // ADC值 → 实际电压
printf("Battery: %d mV\r\n", (int)(volt * 1000));
```

7.4V 锂电池充满 8.4V，低压报警阈值约 6.5V（单节 3.25V）。通过 printf 输出到串口，可以用 Vofa+ 监控电池电压曲线。

---

## 三、自瞄部分

### 3.1 视觉识别：ESP32-S3 CAM

自瞄系统采用 **ESP32-S3 CAM** 作为视觉处理单元，独立于主控 STM32H750。

**为什么不用 H750 直接跑视觉？**
- H750 虽然没有硬件 GPU，跑 OpenMV 帧率低
- H750 的 RAM 有限（D2 SRAM 256KB），图像缓冲区占用大
- ESP32-S3 有专用的 AI 加速器，跑轻量级目标检测更合适
- 两片 MCU 分工明确：H750 管实时控制，ESP32 管视觉识别
- 还有，我这块板子的摄像头排线最高只支持 15cm 以下的长度，超出则无法正常输出

ESP32-S3 的任务：
1. 采集摄像头图像
2. HSV 颜色识别，检测目标色块
3. 位置式 P 控制 + EMA 低通滤波，计算舵机目标角度
4. 通过 I2C 将角度指令发送给 H750

**核心设计理念**：计算全部放在 ESP32 上做。H750 只负责"收角度 → 写舵机"，不做任何视觉计算。这就是典型的**算力分离**——ESP32 追踪抖动不影响安全，H750 的底盘控制循环必须 20ms 内完成不能被打断。

HSV 基础：工具包有个 HSV 的检测工具，拖动滑块来调整 HSV 的阈值，让目标颜色能够显示白色，这样我们就能通过二值化来确定目标色块的位置了。

![HSV检测工具](/Bzone-blog/images/file-20260614054711347.png)

### 3.2 I2C 通信协议

H750（Master）与 ESP32（Slave 0x52）通过 I2C1 通信（PB8/PB9，100kHz，开漏+上拉）：

```
H750 发送: [0x01] (1字节命令)
    ↓ 延时约 500us（ESP32 准备 TX FIFO，空跑 5000 次 NOP）
ESP32 返回: [pan_lo, pan_hi, tilt_lo, tilt_hi, id] (5字节)
```

角度格式：int16 **小端序**，单位 **0.1°**，范围 ±600（即 ±60°）。id=2 表示锁定目标，id=0 表示目标丢失。失败重试 3 次，每次超时 20ms。

```c
// BSP/esp_s3/esp_s3_i2c.c — I2C 读取（带重试）
for (int retry = 0; retry < 3; retry++) {
    HAL_I2C_Master_Transmit(&hi2c1, addr, &reg, 1, 20);   // 发 0x01
    for (volatile int d = 0; d < 5000; d++) { __asm volatile("nop"); }  // 500us 延时
    if (HAL_I2C_Master_Receive(&hi2c1, addr, buf, 5, 20) == HAL_OK) {
        angle->pan  = (int16_t)(buf[0] | (buf[1] << 8));  // 小端序
        angle->tilt = (int16_t)(buf[2] | (buf[3] << 8));
        angle->id   = buf[4];  // 2=锁定, 0=丢失
        return HAL_OK;
    }
}
```

**I2C 退避策略**（`tracking.c` 中实现）：

```c
// 连续失败时逐步拉长轮询间隔
static uint8_t fail_cnt = 0;
uint32_t interval = (fail_cnt < 3)  ? 20    // 前 3 次: 正常 20ms
                  : (fail_cnt < 10) ? 200   // 4-10 次: 降速到 200ms
                  : 500;                     // 10+ 次: 最低 500ms

if (HAL_GetTick() - last_try < interval) return;  // 未到间隔，跳过
last_try = HAL_GetTick();

if (ESP_S3_ReadAngle(&angle) != HAL_OK) {
    fail_cnt++;
    if (fail_cnt > 150) fail_cnt = 0;  // 防溢出
    return;
}
fail_cnt = 0;  // 成功则重置
```

**为什么需要退避？** HAL 的 I2C 收发在从机不响应时会死等 timeout（20ms）。ESP32 离线时每帧被拖到 60ms+（3 次重试 × 20ms），严重影响底盘控制循环。退避策略让 I2C 失败后逐步降低轮询频率，保证主循环的 20ms 节奏不受影响。

**为什么用 0.1° 精度？** ESP32 用浮点做 PID 计算，结果乘 10 变成整数传给 H7，避免浮点传输的精度丢失。与之前 F103 云台项目共用同一套 ESP32 固件，协议完全兼容。

### 3.3 舵机云台

采用 2 自由度舵机云台，搭载 DS3218 数码舵机：

| 舵机 | 功能 | 引脚 | TIM通道 |
|------|------|------|---------|
| PAN | 水平旋转（偏航） | PA2 | TIM15_CH1 |
| TILT | 垂直旋转（俯仰） | PA3 | TIM15_CH2 |

PWM 频率 50Hz（TIM15, PSC=199, ARR=19999），脉宽 500~2500μs 对应 ±90°。脉宽换算：`us = 1500 + 角度(0.1°) × 10/9`。

#### 轨迹步进：逐步逼近而非一步到位

这是舵机控制里一个容易被忽视的设计。`Servo_SetAngle()` 只写入目标角度，真正的运动由 TIM7 中断（每 20ms）中的 `Servo_Tick()` 完成——每次最多走 3°（150°/s），逐步逼近目标。

**为什么需要步进？** 手动模式下长按 L2，角度目标瞬间跳 12°，没有步进插值的话舵机以最大速度猛冲——电流冲击大，机械振动大。追踪时 ESP32 偶尔给出跳跃角度，步进能平滑这种跳变。还有一个好处：舵机内部的 PID 是为小角度调节优化的，大跳变会导致内部 PID 饱和发热。

**软件死区 3μs（约 0.27°）**：角度变化小于 3μs 不写 CCR，有两个目的：过滤步进后的舍入抖动，减少 TIM15 CCR 写入频率从而降低 EMI。DS3218 内部的开关电源是 EMI 主要来源，少写寄存器 = 少干扰 I2C 通信。

### 3.4 自瞄控制流程

```
ESP32 视觉检测 → P控制+EMA滤波 → 输出角度 → I2C 0x01 发送
                                                     ↓
H750 读取5字节 → id判断 → ±60°硬限幅 → Servo_SetAngle → 轨迹步进 → 舵机
     ↑                                                    ↓
     └──────── 摄像头跟随云台转动 ← 视觉闭环 ←───────────┘
```

**H750 端的处理极简**（`tracking.c`）：
1. 读取 ESP32 角度数据（带重试 + 退避）
2. id=0 → 目标丢失，舵机停在原位不动
3. id=2 → 硬限幅 ±60°，直驱舵机

没有死区滤波、没有低通滤波、没有像素→角度转换——这些全部由 ESP32 完成。H7 只做三件事：**读 I2C、限幅、写目标角度**。舵机层的软死区和轨迹步进自动处理掉微小抖动。

**手动/自动切换**：SELECT 下降沿触发，300ms 冷却期防连发。切手动时舵机自动归中——安全设计，防止舵机停在偏转位置时操作者不知情导致云台撞障碍物。

---

## 四、开发调试与踩坑

### 4.1 CubeMX 与手写代码边界

本项目基于 CubeMX 生成外设初始化代码，所有自定义代码在 `/* USER CODE BEGIN/END */` 块内。CubeMX 重新生成不会覆盖，但仍需检查：MPU 4 区是否保留、TIM1 CH2-4 是不是 PWM 模式（CubeMX 可能重置为 OC Timing）、TIM2-5 编码器模式是否保留、I2C1 引脚是否还在、FreeRTOS 堆大小是否还是 65536。

### 4.2 编译与调试

Keil MDK-ARM V5.27 + ARMCC V5.06。命令行编译：`UV4.exe -b MDK-ARM/Project.uvprojx -o MDK-ARM/build.log -j0`。典型体积：Code≈49KB, RO≈6.5KB, RW≈292B, ZI≈223KB。

USART1 已重定向 `fputc`，可以直接 `printf()` 调试。调试 PID 参数最有效的方法不是猜，而是取消 `Control_Task` 中的 CSV 输出注释，用 **Vofa+** 或 Serial Plotter 看目标速度 vs 实际速度的实时曲线——一条曲线胜过一百次盲调。

### 4.3 踩坑记录

**D-Cache 编码器卡住**：读 TIM2-5 计数器一直不变。原因是 MPU Region3 没覆盖 APB 外设区，D-Cache 返回了旧值。解法：`MPU_Config()` 必须在 `SCB_EnableDCache()` 之前，Region3 覆盖 0x40000000。

**舵机 EMI 干扰 I2C**：舵机转动时 ESP32 通信偶发失败。原因是 DS3218 内部开关电源的 EMI。解法：软件死区减少 CCR 写入（少产生 PWM 沿跳变）+ I2C 重试 3 次 + 退避策略做兜底。

**SPI5 初始化空的**：CubeMX 的 SPI5 MspInit 函数体是空的，GPIO 复用没配。解法：在 SPI4 的 MspInit USER CODE 块里调用手写的 `SPI5_MspInit()` 曲线救国。

**SELECT 反复跳模式**：电平触发 + 无冷却。解法：下降沿触发 + 300ms 冷却 + 3 帧消抖。

**FreeRTOS SysTick 冲突**：`HAL_Delay()` 时间全乱。原因是 FreeRTOS 用了 SysTick，HAL 库也默认用 SysTick。解法：CubeMX 中把 HAL 时基选为 TIM6。

---

## 五、硬件引脚速查

| 模块 | 外设 | 引脚 | 说明 |
|------|------|------|------|
| 电机 PWM | TIM1 | PE9/11/13/14 | CH1-4, ~9.5kHz |
| 电机方向 | GPIO | PD3/PI0, PG12/13, PB11/10, PH14/PC11 | IN1/IN2 |
| 电机使能 | GPIO | PC7 | STBY，高电平有效 |
| 编码器 | TIM2-5 | PA15/PB3, PB4/5, PD12/13, PA0/PH11 | 正交编码 |
| 舵机 | TIM15 | PA2/PA3 | 50Hz, 500~2500μs |
| PS2 手柄 | SPI2 | PI1/2/3 + PD4 | Mode3 LSB, ~780kHz |
| ESP-S3 | I2C1 | PB8/PB9 | 100kHz, Slave 0x52 |
| LCD | SPI5 | PK0/PJ10/PJ11/PH5/PH6 | ST7789, 240×320 |
| W5500 | SPI4 | PE2/5/6 + PG14/PE3/PE4 | 以太网 |
| 电池检测 | ADC1 | PA7 | ADC1_IN7, 18:1分压 |

---

## 六、总结

本项目的核心价值在于 **H750 + FreeRTOS 的全栈嵌入式开发**，涵盖了麦轮运动学、增量式 PID、I2C 主从协议、FreeRTOS 多任务调度、MPU + D-Cache 底层调试。

对于想复现或学习的同学：控制算法（运动学、PID、滤波）是通用的，在 F407 上一样能学。选 H750 是因为刚好手头有，但它带来的 Cache/MPU 调试时间确实不少。如果你刚开始，F407 足够；如果你想深入 MCU 底层，H750 是最佳教材——它逼你理解 ARM 架构里那些"高级芯片才有的东西"。

> 项目持续更新中，欢迎交流、指正、贡献代码。
