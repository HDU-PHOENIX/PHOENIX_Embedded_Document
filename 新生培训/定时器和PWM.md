# 定时器 (Timer) 与 PWM 快速指南

> 针对 STM32 (HAL 库) 的定时器基础、PWM 输出、中断配置及常用函数整理。

## 1. 定时器基本知识

### 1.1 核心组成模块

通用定时器（General-purpose Timer）主要由以下几个部分构成：

-   **计数器 (CNT - Counter)**:
    -   核心部件，是一个向上或向下计数的寄存器。
    -   它的值在每个时钟周期（经过预分频后）更新。

-   **预分频器 (PSC - Prescaler)**:
    -   对输入的时钟源进行分频，从而“减慢”计数器的计数速度。
    -   `计数器时钟频率 = 输入时钟 / (PSC + 1)`

-   **自动重装载寄存器 (ARR - Auto-Reload Register)**:
    -   设置计数器的计数周期或“上限”。
    -   当 CNT 计数到 ARR 的值时，会产生一个“更新事件 (Update Event)”，并将 CNT 清零（或重置为 ARR 的值，在向下计数模式中），然后重新开始计数。
    -   `更新事件频率 (中断频率) = 计数器时钟频率 / (ARR + 1)`

-   **捕获/比较寄存器 (CCR - Capture/Compare Register)**:
    -   通常有 1 到 4 个 (CCR1 ~ CCR4)。
    -   **比较模式 (PWM)**: CNT 的值与 CCR 的值进行比较。当 CNT 达到 CCR 时，可以触发一个事件，如翻转输出引脚的电平。这是实现 PWM 的基础。
    -   **捕获模式**: 捕获输入引脚上的电平变化，并将当时的 CNT 值锁存到 CCR 中，用于测量周期信号的频率或脉宽。

### 1.2 关键公式
-   **定时器时钟 (Timer Clock)**:
$$
T_{out} = \frac{((PSC + 1) * (ARR + 1))}{F_{clk}}
$$
-   **PWM占空比(Duty Cycle)**:
   $$
Duty\ Cycle\ (\%) = \frac{CCR}{ARR + 1} * 100\%
$$

### 1.3 精度考量 (PSC 小, ARR 大)

-   **场景**: 假设你需要一个 1kHz 的 PWM。你可以设置 `PSC=71, ARR=999` 或者 `PSC=719, ARR=99` (假设 `F_clk=72MHz`)。
-   **高精度**: 选择 `PSC=71, ARR=999`。
    -   占空比调节的“步进”是 `1 / (999 + 1) = 0.1%`。你可以非常精细地控制占空比。
-   **低精度**: 选择 `PSC=719, ARR=99`。
    -   占空比调节的“步进”是 `1 / (99 + 1) = 1%`。调节粒度较粗。
-   **结论**: 在满足频率要求的前提下，**优先选择较小的 PSC 和较大的 ARR**，可以获得更高的 PWM 分辨率（精度）。
-   **寄1器位数**: STM32F103 的通用定时器是 16 位的，所以 PSC 和 ARR 的最大值都是 65535。

## 2. 常见工作模式与其中 `

主要掌握 **基本定时器中`断** 和 **PWM 输出** 两种模式。

### 2.1 定时器中断模式
此模式用于周期性地执行某段代码，例如每隔 1ms 翻转一次 LED。

**配置步骤 (以 TIM1 为例, 1ms 中断):**

1.  **在 CubeMX 中配置**:
    -   选择一个定时器 (如 TIM1)，设置 `Clock Source` 为 `Internal Clock`。
    -   在 `Parameter Settings` 中，假设 APB1 Timer Clocks 为 72MHz。
    -   设置 `Prescaler (PSC)` 为 `71`  (分频后得到 12MHz / (71+1) = 1MHz)。
    -   设置 `ARR` 为 `999` (1MHz / (999+1) = 1kHz，即 1ms)。    

2.  **在代码中启动**:
    ```c
    // main.c
    // 在 main 函数的 while(1) 循环前启动定时器中断,但是要在初始化之后
    HAL_TIM_Base_Start_IT(&htim1);
    ```

3.  **HAL库会调用一个弱定义的回调函数。我们需要在用户代码区重新实现它。**
    ```c
    // stm32f1xx_it.c 或 main.c 的用户代码区
    void HAL_TIM_PeriodElapsedCallback(TIM_ndleTypeDef *htim)
    {
        // 判断是哪时器触发了中断
        if (htim->Instance == TIM3)
        {
            // 在这里编写需要周期执行的代码
            HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin); // 例如，每 1ms 翻转一次 LED
        }
    }
    ```

### 2.2 PWM 输出模式

此模式用于从引脚输出脉宽调制信号，常用于控制电机转速、舵机角度、LED 亮度等。

**配置步骤 (以 TIM2 Channel 1 为例, 1kHz PWM):**

1.  **在 CubeMX 中配置**:
    -   选择一个定时器 (如 TIM2)，设置 `Clock Source` 为 `Internal Clock`
    -   设置 `Channel 1` 为 `PWM Generation CH1`
    -   在 `Parameter Settings` 中，配置频率（同上，PSC=71, ARR=999）
    -   设置 `Pulse (CCR)` 的初始值，例如 `500`，表示初始占空比为 `500 / 1000 = 50%`。
    ![](https://youke1.picui.cn/s1/2025/11/08/690f3f2615bc8.png)

2.  **在代码中启动**:
    ```c
    // main.c
    // 在 main 函数的 while(1) 循环前启动 1WM 通道
    HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_1);
    ```

3.  **动态修改占空比**:
    在程序的任何地方，你都可以通过修改 CCR 的值来改变 PWM 的占空比。
    ```c
    // 在 while(1) 循环中，实现一个呼吸灯效果
    for (uint16_t i = 0; i < 1000; i++)
    {
        __HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, i);
        HAL_    -   功能: 以中断模式启动一个基本定时器。
    -   参数: `htim` - 定时器句柄，如 `&htim3`。_HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, i);
        HAL_Delay(1);
    }
    ```

## 3. 需要掌关键:

-   `HAL_StatusTypeDef HATIBase_Start_IT(TIM_HandleTypeDef *htim);`
    -   **功能**: 以中断模式启动一个基本定时器。
    -   **参数**: `htim` - 定时器句柄，如 `&htim3`。

-   `HAL_StatusTypeDef HAL_TIM_PWM_Start(TIM_HandTyDef *htim, uint32_t Channel);`
    -   **功能**: 启动一个定时器的特定 PWM 通道。
    -   *数*
        -   `htim`: 定时器句柄，如 `&htim2`。
        -   `Channel`: 要启动的通道，如 `TIM_CHANNEL_1`。

-   `__HAL_TIM_SET_COMPARE(__HANDLE__, __CHANNEL__, __COMPARE__);`
    -   **功能**: 这是一个宏，用于 **动态设置捕获/比较寄存器 (CCR) 的值**，从而改变 PWM 的占空比。这是用的WM 控制方法。
    -   **参数**:
        -   `__HANDLE__ 定句柄，如 `&htim2`。
        -   `__CHANNEL__`: 要设置的通道，如 `TIM_CHANNEL_1`。
        -   `__COMPARE__`: 新的 CCR 值 (0 ~ ARR)。

-   `void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim);`
    -   **功能**: 定时器更新事件（计数溢出）的回调函数。在定时器中断模式下，代码逻辑写在这里。
    -   **注意**: 这是一个弱函数 (`__weak`)，你需要自己在一个源文件中重新实现它。多个定时器中断都会进入此函数，需要通过 `htim->Instance` 来区分。
    
> Copyright (c) 2025 PHOENIX
> 作者： 何文轩 <wenx_public@163.com>
> 说明：本文件版权由杭州电子科技大学PHOENIX实验室持有
