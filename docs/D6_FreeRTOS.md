# D6: FreeRTOS 多任务调度

## 目标
将 D5 裸机超循环改造为 FreeRTOS 多任务架构。ADCTask 采集光敏数据并通过队列发送，PWMTask 接收并设置 PWM，两个任务由抢占式调度器自动切换。理解任务状态机、队列通信、栈溢出风险、优先级饥饿。

## 硬件
- 芯片：STM32F103C8T6
- RGB 灯：PA9(B)、PA10(G)、PA11(R)，TIM1 PWM
- 光敏模块：AO→PA0(ADC1_IN0)
- FreeRTOS：CMSIS-RTOS v2, SysTick 1kHz 心跳，TIM4 替代 HAL 时基

## 环境
- STM32CubeMX + Keil MDK + ST-Link
- 时钟：HSE 8MHz → PLL 9倍频 → SYSCLK 72MHz

---

## CubeMX 配置

### 新增 FreeRTOS
| 参数 | 值 | 说明 |
|------|-----|------|
| FREERTOS → Interface | CMSIS_V2 | 新版 CMSIS-RTOS 封装 |
| TICK_RATE_HZ | 1000 (默认) | 系统心跳 1kHz |

### 改 HAL 时基
| 参数 | 值 | 说明 |
|------|-----|------|
| SYS → Timebase Source | TIM4 | FreeRTOS 抢走 SysTick，HAL 改用 TIM4 |

FreeRTOS 启动 SysTick 做心跳（每 1ms 触发调度器检查），HAL 的 `HAL_Delay`/`HAL_GetTick` 换到 TIM4。

### 任务
| 任务 | 入口函数 | 优先级 | 栈大小 |
|------|---------|--------|--------|
| defaultTask | StartDefaultTask | Normal | 128 words |
| ADCTask | adcTaskFunc | **Normal** | 128 words |
| PWMTask | pwmTaskFunc | **Normal** | 128 words |

三个任务同级轮转。priority **必须 Normal**——defaultTask 是 Normal，ADCTask/PWMTask 若设 Low 会被 defaultTask 永久饿死。

### 队列
| 名称 | 类型 | 条目数 | 说明 |
|------|------|--------|------|
| adcQueue | uint32_t | 4 | ADCTask → PWMTask 传递 PWM 值 |

### 保留外设（D4/D5 配置）
- TIM1: PSC=71, ARR=999, 1kHz PWM, CH2(B)/CH3(G)/CH4(R)
- ADC1: PA0(IN0), /6=12MHz, 239.5Cycles, Continuous
- USART2: 38400, PA2/PA3（未在本 D6 使用，保留）

---

## 关键代码

### ADCTask（USER CODE BEGIN adcTaskFunc）
```c
uint32_t adc_value = 0;
uint32_t pwm_duty = 0;

/* 自动标定：持续追踪 ADC 的最小和最大值 */
static uint32_t adc_min = 4095;
static uint32_t adc_max = 0;

for(;;)
{
    adc_value = HAL_ADC_GetValue(&hadc1);
    if (adc_value < adc_min) adc_min = adc_value;
    if (adc_value > adc_max) adc_max = adc_value;
    if (adc_max > adc_min)
        pwm_duty = (uint32_t)((adc_value - adc_min) * 999 / (adc_max - adc_min));
    else
        pwm_duty = 0;
    osMessageQueuePut(adcQueueHandle, &pwm_duty, 0, 0);
    osDelay(20);
}
```

### PWMTask（USER CODE BEGIN pwmTaskFunc）
```c
uint32_t pwm = 0;

for(;;)
{
    osMessageQueueGet(adcQueueHandle, &pwm, NULL, osWaitForever);
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_4, pwm);  // R
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_3, pwm);  // G
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_2, pwm);  // B
}
```

### while(1)（清空）
D5 的 ADC 读值、标定、PWM 设置全部移入任务函数。while(1) 为空——调度器在 `osKernelStart()` 接管后永不返回此处。

---

## 踩坑记录

### 坑1：核心代码写在 for(;;) 外面
- 现象：ADC 只采一次，之后灯不随光照变化
- 原因：ADC 读值、队列 Put 写在 `for(;;)` 上面（函数体内但循环外），只在上电后跑一次就进空循环 `osDelay(1)`
- 解决：所有核心逻辑搬进 `for(;;)` 内部

### 坑2：队列类型不匹配导致栈越界
- 现象：编译无警告，运行时偶尔数据异常
- 原因：队列创建用 `sizeof(uint32_t)=4`，但任务内用 `uint16_t` 变量 Put/Get。`osMessageQueuePut` 从 `uint16_t*` 地址读 4 字节（多读 2 字节栈垃圾），`osMessageQueueGet` 往 `uint16_t*` 地址写 4 字节（覆盖相邻栈内存）
- 解决：任务内变量全部改为 `uint32_t`

### 坑3：优先级饥饿
- 现象：ADCTask 和 PWMTask 完全不运行
- 原因：defaultTask 是 Normal，两个任务设了 Low。Low 优先级任务在 Normal 就绪期间永远拿不到 CPU——defaultTask 每 1ms 醒一次反复抢占
- 解决：改为 Normal，三个同级轮转

### 坑4：代码写在了 USER CODE 保护外
- 现象：无即时影响
- 风险：`for(;;)` 和 `osMessageQueueGet` 若写在 `/* USER CODE BEGIN */` 上面，CubeMX 重新生成代码时会被丢弃
- 解决：全部任务逻辑下沉到 USER CODE BEGIN/END 之间

---

## 学到的

### FreeRTOS 运行机制
- **SysTick 心跳**：1kHz 中断触发调度器，检查延时到期、唤醒任务
- **任务状态机**：Ready → Running（调度器选中）→ Blocked（osDelay / 队列等待）→ Ready（延时到 / 数据到）
- **PendSV 上下文切换**：ARM 最低优先级异常，保存当前任务寄存器到私栈 → 切换栈指针 → 恢复下一任务寄存器。耗时 1~2μs
- **同时刻只有一个任务 Running**，同级轮转，高优先级永久抢占低优先级

### 队列 vs 全局变量
- 队列是线程安全的数据管道，**不共享全局变量**
- `osMessageQueuePut/Get` 的 `timeout` 参数控制阻塞行为：`0` 无阻塞，`osWaitForever` 永久等
- PWMTask 不需要 `osDelay`——`osMessageQueueGet(osWaitForever)` 本身就是阻塞点
- 队列元素大小必须与变量类型完全一致，否则栈越界（编译器不警告）

### osDelay vs HAL_Delay
- `HAL_Delay(20)`：20ms 空循环，CPU 闲烧
- `osDelay(20)`：任务挂 Blocked 20 tick，CPU 切给其他任务

### USER CODE 保护线
- CubeMX 只保留 `/* USER CODE BEGIN X */ ... /* USER CODE END X */` 之间的代码
- 写在线外的内容在下一次生成时全部丢失
- 任务函数的所有逻辑（包括 `for(;;)` 和变量声明）必须放在 BEGIN/END 之内

---

## 验证现象

| 动作 | 灯状态 | 说明 |
|------|--------|------|
| 上电 | 点亮 | ADCTask 采一次→发队列→PWMTask 收→设 PWM |
| 遮住光敏模块 | 灯变亮 | 暗→高 ADC→高 PWM |
| 手电筒照模块 | 灯变暗 | 亮→低 ADC→低 PWM |
| 持续运行 | 自动标定收敛 | adc_min/max 稳定，变化幅度明显 |

---

## 下一步

D7: DMA + ADC 连续采集，或信号量/互斥锁多任务同步。
