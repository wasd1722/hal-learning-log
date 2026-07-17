# D4: PWM 呼吸灯

## 目标
通过 TIM1 高级定时器 PWM 模式，实现 RGB 三色交替呼吸灯效果。理解占空比、频率、人眼非线性感知。

## 硬件
- 芯片：STM32F103C8T6
- RGB 灯：PA9(B)、PA10(G)、PA11(R)，共阴极，高电平点亮
- TIM1 高级定时器，Channel2/3/4 分别对应 B/G/R

## 环境
- STM32CubeMX + Keil MDK + ST-Link
- 时钟：HSE 8MHz → PLL 9倍频 → SYSCLK 72MHz

---

## CubeMX 配置

### TIM1 高级定时器
| 参数 | 值 | 说明 |
|------|-----|------|
| Clock Source | Internal Clock | 内部时钟 |
| Prescaler (PSC) | 71 | 72MHz / 72 = 1MHz |
| Counter Period (ARR) | 999 | 1MHz / 1000 = 1kHz PWM 频率 |
| Auto-reload preload | Disable | |
| Channel2 | PWM Generation CH2 | PA9 - B |
| Channel3 | PWM Generation CH3 | PA10 - G |
| Channel4 | PWM Generation CH4 | PA11 - R |

### PWM 通道参数
| 参数 | 值 | 说明 |
|------|-----|------|
| Mode | PWM Mode 1 | 向上计数，小于 Pulse 输出高 |
| Pulse | 0 | 初始占空比 0% |
| Polarity | High | 共阴极，高电平点亮 |
| Output compare preload | Enable | 双缓冲，防止毛刺 |
| Fast Mode | Disable | |
| CH Idle State | Reset | 空闲低电平 |

### Break And Dead Time
| 参数 | 值 | 说明 |
|------|-----|------|
| Automatic Output | Disable | 刹车恢复不自动使能 MOE |
| Off State Run Mode | Disable | |
| Off State IDLE Mode | Disable | |
| Break State | Disable | 不用刹车 |

---

## 关键代码

### 全局变量（USER CODE BEGIN PV）
```c
uint16_t pwm_val = 0;      // 当前 PWM 占空比值
uint8_t breathe_dir = 1;   // 1=增加，0=减少
uint8_t color_state = 0;   // 0=红，1=绿，2=蓝
```

### 启动 PWM（USER CODE BEGIN 2）
```c
HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_2);  // PA9 - B
HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_3);  // PA10 - G
HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_4);  // PA11 - R
```

### 主循环呼吸灯（USER CODE BEGIN 3）
```c
while (1)
{
    // 呼吸逻辑：0 -> 999 -> 0
    if (breathe_dir)
    {
        pwm_val++;
        if (pwm_val >= 999) breathe_dir = 0;
    }
    else
    {
        pwm_val--;
        if (pwm_val == 0)
        {
            breathe_dir = 1;
            color_state++;  // 切换颜色
            if (color_state > 2) color_state = 0;
        }
    }

    // 根据颜色状态设置对应通道
    switch (color_state)
    {
        case 0:  // 红色呼吸
            __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_4, pwm_val);  // R
            __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_3, 0);          // G 灭
            __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_2, 0);          // B 灭
            break;

        case 1:  // 绿色呼吸
            __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_4, 0);          // R 灭
            __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_3, pwm_val);    // G
            __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_2, 0);          // B 灭
            break;

        case 2:  // 蓝色呼吸
            __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_4, 0);          // R 灭
            __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_3, 0);          // G 灭
            __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_2, pwm_val);    // B
            break;
    }

    HAL_Delay(2);  // 延时 2ms，控制呼吸速度
}
```

---

## 踩坑记录：MOE 主输出使能

### 现象
- 配置完 PWM，灯不亮
- 怀疑 MOE 没使能，手动加 `__HAL_TIM_MOE_ENABLE(&htim1)`

### 排查过程
1. 查手册：复位后 MOE 默认 0，高级定时器必须使能 MOE 才能输出
2. 但代码里没有 `__HAL_TIM_MOE_ENABLE`，灯却亮了
3. 查 HAL 库源码 `HAL_TIM_PWM_Start`：发现内部自动判断高级定时器并调用 `__HAL_TIM_MOE_ENABLE`

### 根因
HAL 库新版本在 `HAL_TIM_PWM_Start` 内部自动使能 MOE，无需手动调用。

### 代码证据
```c
if (IS_TIM_BREAK_INSTANCE(htim->Instance) != RESET)
{
    /* Enable the main output */
    __HAL_TIM_MOE_ENABLE(htim);
}
```

### 教训
- 不要死记硬背旧版本 HAL 的行为
- 遇到"应该这样但结果不同"，查源码最可靠
- `Automatic Output` 配置只在刹车恢复时生效，不是首次启动

---

## 学到的

### PWM 原理
- 频率 >100Hz，人眼看不到闪烁
- 占空比 = 高电平时间 / 周期，决定亮度
- `__HAL_TIM_SET_COMPARE` 修改占空比，值范围 0~ARR

### 高级定时器
- TIM1/TIM8 有 MOE 主输出使能，普通定时器没有
- `HAL_TIM_PWM_Start` 内部自动处理 MOE（新版本 HAL）
- 刹车、死区、互补输出是电机控制的安全机制

### 人眼感知
- 占空比线性变化（0→999→0），但人眼感知非线性
- 对暗部敏感，对亮部迟钝
- 需要做伽马校正（Gamma Correction）才能得到平滑的呼吸效果
- STM32F1 无硬件浮点，可用查表法实现

### 工程规范
- 三色交替时，非当前颜色通道占空比置 0，避免混色
- 状态机（color_state）管理颜色切换，比多个 if-else 清晰

---

## 验证现象

| 阶段 | 颜色 | 现象 |
|------|------|------|
| 1 | 红色呼吸 | 暗→亮→暗，约 2 秒 |
| 2 | 绿色呼吸 | 暗→亮→暗，约 2 秒 |
| 3 | 蓝色呼吸 | 暗→亮→暗，约 2 秒 |
| 循环 | 红→绿→蓝→红... | 持续循环 |

---

## 下一步

D5: ADC 采集光敏传感器，根据环境亮度自动调节 PWM 占空比，实现"天黑自动亮灯"。