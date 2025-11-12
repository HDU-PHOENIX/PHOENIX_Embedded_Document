# GPIO 与外部中断快速指南
> 面向 STM32（HAL 库）等常见 MCU 的 GPIO 模式与外部中断速查文档，包含概念、模式对比、点灯示例、外部中断配置以及工程注意事项。

## 1. GPIO 基础理论

### 1.1 什么是 GPIO
GPIO（General Purpose Input/Output，通用输入输出端口）用于 MCU 与外部世界交换电压信号。电平“高/低”不一定等价于正/负，典型 TTL/CMOS 逻辑受不同接口标准影响（例如 RS232 的电压范围约 3~15V / -3~-15V，但与 MCU GPIO 逻辑电平判定不同）。

### 1.2 常见引脚模式总览

| 分类 | 模式名称 | HAL 宏示例 | 方向 | 说明 | 典型用途 |
|------|----------|------------|------|------|----------|
| 输入 | 浮空输入 | GPIO_MODE_INPUT + GPIO_NOPULL | IN | 无内部上/下拉，外部未驱动时电平不确定，易受干扰 | 很少直接使用，除非外部已经设计硬件电平钳制 |
| 输入 | 上拉输入 | GPIO_MODE_INPUT + GPIO_PULLUP | IN | 未输入时内部上拉到高；外部短接到地则为低 | 读取按键（默认高，按下低） |
| 输入 | 下拉输入 | GPIO_MODE_INPUT + GPIO_PULLDOWN | IN | 未输入时内部下拉到低；外部拉高则为高 | 读取跳线帽、接收脉冲信号 |
| 输入 | 模拟输入 | GPIO_MODE_ANALOG | IN | 关闭数字施密特触发器与上下拉，进入 ADC 通道 | 采集电压/传感器 |
| 输出 | 推挽输出 | GPIO_MODE_OUTPUT_PP | OUT | 内部同时含 P/N MOS，可主动驱动高/低，驱动能力强 | 直接驱动 LED（限流）、数字外设时钟线 |
| 输出 | 开漏输出 | GPIO_MODE_OUTPUT_OD | OUT | 内部只下拉（N MOS），上拉需外部或内部电阻；线与时“低”优先 | I2C、信号线复用、LED 共线 |
| 复用 | 复用推挽 | GPIO_MODE_AF_PP | OUT/AF | 由外设控制（如 UART TX、SPI SCK）驱动强 | 高速外设信号 |
| 复用 | 复用开漏 | GPIO_MODE_AF_OD | OUT/AF | 外设只下拉，需外部上拉；适合线与或容错 | I2C SDA/SCL |
| 输出 | 模拟输出 | DAC（外设）或 PWM（定时器） | OUT | DAC 输出连续模拟电压；PWM 通过占空比近似模拟 | 音频、亮度、电机控制 |

### 1.3 关键概念说明
- 上/下拉输入：在“没有外部驱动”时仍保持确定电平，避免漂浮；最常用。
- 浮空输入：非确定状态，容易受噪声影响，不等于“低电平”。
- 模拟输入：交由 ADC；需确保对应管脚未被配置为数字功能。
- 推挽输出：高低都能强驱动（典型可达 20mA，视芯片手册），若两个推挽输出互相短接成相反电平会烧毁端口。
- 开漏输出：低电平驱动强，高电平依靠外部上拉电阻；多个开漏端口可线与，任一拉低则总线为低。
- 复用输出：实际由片上外设（USART/SPI/I2C 等）控制，代码通过初始化外设而非直接写 pin。
- 模拟输出：DAC 为真正电压；PWM 需要低通滤波或外设理解占空比。
> 施密特触发器（Schmitt Trigger）：输入端带有两个阈值（上/下），形成迟滞，抑制噪声抖动；数字输入通常内部集成它。

### 1.4 开漏与上拉电阻选择
外接上拉电阻越小，上升沿越快（RC 时间常数减小），但功耗越大。常用取值：4.7kΩ ~ 10kΩ。高速总线或长线可用 2.2kΩ；需避免过小导致电流超限。

## 2. 示例
很多开发板上的板载 LED 连接在 `PC13`，且是“低电平点亮”。原因：LED 可能串接到 VCC，经电阻后接到 PC13，MCU 输出低即导通，输出高即熄灭，可减少高电平驱动需求。
### 2.1 CUBEMX配置
按图配置皆可
![](https://youke1.picui.cn/s1/2025/11/08/690f213726b8f.png)
### 2.2 点灯代码
```c
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_RESET); // 点亮 LED
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET);   // 熄灭 LED
    HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);                // 切换状态
```
## 3. 外部中断（EXTI）
外部中断可打断主循环执行，进入中断服务函数（ISR）。硬件自动保护现场（堆栈保存通用寄存器等）。不要在 ISR 里写过于复杂或耗时的逻辑。

### 3.1 中断与事件
- Interrupt：触发后进入 ISR，软件必须处理。
- Event：用于唤醒 MCU 或触发某些外设，不一定进入 ISR,这里不做了解。

### 3.2 优先级
优先级高的中断可以打断低优先级中断。配置 NVIC 时需设置抢占优先级（Preempt Priority）与响应优先级（Sub Priority）结构（不同芯片略有差异）。

### 3.3 基本配置示例
CUBEMX中按图配置皆可：
![](https://youke1.picui.cn/s1/2025/11/08/690f2265de15b.png)

### 3.4 中断入口与回调
`stm32f1xx_it.c` 中：
```c
void EXTI0_IRQHandler(void)
{
	HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_0); // HAL 框架调用，内部会最终触发回调
}
```

用户回调函数（放在 `stm32f1xx_hal_gpio.c` 所引用的用户区域或用户源文件中），建议把代码写在这里：
```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == GPIO_PIN_0) {
        // 避免长延时与阻塞
        HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13); // 切换 LED 状态
    }
}
```

### 3.5 按键去抖（软件示例）
按键可能在机械接触阶段抖动，加入简单定时过滤：
```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == GPIO_PIN_0) {
        uint16_t cnt = 2000;
        while(cnt--); // 简单延时去抖
        HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);
    }
}
```

### 3.6 ISR 中的禁止事项
- 不要调用 `HAL_Delay()`（依赖 SysTick 阻塞，会延长中断占用时间甚至失效）。
- 不要做大规模浮点运算或访问慢速外设（如耗时 I2C 轮询）。
- 不要长时间 `while` 轮询等待；改设标志位在主循环处理。

推荐模式：在回调中设置一个 `volatile` 标志：
```c
volatile uint8_t keyPressed = 0;

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == GPIO_PIN_0) {
        keyPressed = 1; // 轻量标记
    }
}

// 主循环中：
if (keyPressed) {
    keyPressed = 0;
    // 执行复杂逻辑或输出串口
}
```

## 4. 常见问题与调试建议
- LED 不亮：检查是否为“低电平点亮”、电阻是否过大、端口是否初始化成功。
- 中断不触发：确认 NVIC 使能、`IRQHandler` 是否调用 `HAL_GPIO_EXTI_IRQHandler`、触发沿设置是否与硬件动作匹配。
- 引脚读值乱跳：是否使用了浮空输入；考虑改为上拉/下拉或加入 RC 去抖。若外部频繁变化，检查是否需要硬件滤波。
- 推挽互连冲突：多个输出直接相连且电平不同，可能导致端口损坏，需改用开漏 + 上拉或三态控制。
- 开漏上升过慢：减小上拉电阻或降低总线电容；检查布线长度及并联器件。

> Copyright (c) 2025 PHOENIX
> 作者： 何文轩 <wenx_public@163.com>
> 说明：本文件版权由杭州电子科技大学PHOENIX实验室持有