# Uart串口
> 串口（UART），全称 Universal Asynchronous Receiver/Transmitter，中文叫“通用异步收发器”，当然，我们更习惯称之为串口。

## 1. 简单介绍
它的本职工作很简单：
- 把 MCU 内部的并行数据转成串行形式发出去；
- 或者把接收到的串行数据还原成并行数据送进系统。

## 2. 数据传输
UART 通信说白了就是两端 UART 对着通信：一个发，一个收。
### 2.1 线路
Uart通信通常使用三条线路进行数据传输：
- TX（Transmit）：发送数据线，用于将数据从发送端传输到接收端。
- RX（Receive）：接收数据线，用于将数据从接收端传输到发送端。
- GND（Ground）：地线，用于提供共同的参考电位

**注意：UART 串口的TX和RX是交叉连接的，即发送端的TX连接到接收端的RX，反之亦然。**
### 2.2 波特率
由于 UART 是异步通信方式，没有时钟同步信号，靠的是通信双方提前约定好的波特率来保持节奏一致。波特率就是每秒传多少个 bit，比如 9600bps、115200bps 都是常见值。**uart 通信的双方必须使用相同的波特率**，否则数据就会错乱。
### 2.3 数据帧格式
UART 通信的数据是以数据帧的形式传输的。一个典型的数据帧包括以下几个部分：
![Uart数据帧格式](https://youke1.picui.cn/s1/2025/11/19/691d3d15374fe.png)
1. 起始位
在空闲状态下，TX 引脚是高电平（逻辑 1），当有数据要发时，先拉低一位（逻辑 0），告诉接收端“准备好了”，这就是起始位。
2. 数据位
真正的数据部分，一般是 8 位，也有的 UART 支持 5~9 位可选，最常见的是 8 位无校验。
3. 校验位（可选）
为了检测传输过程中的错误，UART 可以加个奇偶校验位（Parity Bit）。
比如“偶校验”就要求总共有偶数个 1，这样接收方可以根据收到的位数判断有没有出错。
注意：这不是真正的纠错，**只能报错，不能“纠错”**，跟 CRC、FEC 那种比还是简单得多。
4. 停止位
表示数据包结束。通常是 1 位或 2 位逻辑高电平，给接收方缓口气，也防止包和包之间连在一起。
### 2.4 一次完整的数据传输
1. MCU 把要发的数据送到 UART（并行形式）
2. UART 把数据封装成帧，加上起始位、奇偶校验位、停止位
3. UART 把这个包按位发出去（通过 TX 引脚）
4. 接收端 UART 从 RX 引脚接收数据
5. 还原数据，去掉起始位、校验位、停止位，送回系统

## 3. 几种数据收发方式
### 3.1 阻塞方式（Polling/Blocking）

原理：CPU 通过不断轮询（循环检查）UART 的状态寄存器（如 TXE 发送空标志、RXNE 接收非空标志）来判断是否可以发送或接收数据。如果状态未就绪，CPU 就一直等待（阻塞），直到操作完成。
示例流程：发送数据时，循环检查 TXE 标志位为 1，然后写入数据寄存器；接收时，检查 RXNE 为 1，然后读取数据。

优缺点：
优点：实现简单，不需要中断或 DMA 配置，代码易懂，适合初学者。
缺点：CPU 被完全占用，无法处理其他任务，效率低下，尤其在低波特率或大数据量时会浪费大量 CPU 时间。可能导致系统响应迟钝。

适用场景：简单调试、数据量小、低实时性要求的场合，如初级实验或非多任务系统。这是最基础的方式，但不推荐生产环境使用。

### 3.2 中断方式（Interrupt）

原理：UART 硬件在特定事件发生时（如数据接收完成、发送缓冲空闲）触发中断信号。CPU 响应中断，进入中断服务函数（ISR）处理数据（如读取/写入数据寄存器）。处理完后，返回主程序。
示例流程：使能 UART 中断（如 NVIC 配置），在 ISR 中处理数据，并可能清除中断标志。可以使用 FIFO 缓冲区来缓存数据，避免丢失。

优缺点：
优点：比阻塞高效，CPU 只在需要时介入，可以并发处理其他任务。实时性较好，适合中等数据率。
缺点：中断开销（上下文切换）较大，如果中断频繁（如高波特率），可能导致 CPU 负载高，甚至中断风暴。代码复杂，需要处理优先级和嵌套。

适用场景：大多数嵌入式应用，如传感器数据采集、串口调试工具。

### 3.3 DMA 方式（Direct Memory Access）

原理：DMA 控制器直接在内存和 UART 外设之间传输数据，而不需 CPU 干预。CPU 只需初始化 DMA（如设置源/目的地址、传输长度），然后 DMA 自动搬运数据。传输完成后，通过中断或标志通知 CPU。
示例流程：配置 DMA 通道与 UART 关联，使能 DMA 模式（如 UART 的 DMAT/DMAR 位），启动传输。适合连续大数据块。

优缺点：
优点：最高效，CPU 几乎不参与传输，可以专注于其他计算任务。支持高吞吐量，减少延迟，适合高速通信。
缺点：配置复杂，需要硬件支持（不是所有 MCU 都有 DMA），调试难度高，可能有总线竞争问题。内存管理需小心，避免覆盖数据。

适用场景：高性能应用，如高速数据采集、音频/视频传输、大文件下载。

#### 3.3.1 DMA 是什么？
DMA 全称是 Direct Memory Access，翻译成“直接内存访问”。简单来说，它是一种硬件技术，让你的嵌入式设备（如单片机）里的外设（比如 UART 串口、传感器）可以直接和内存“聊天”交换数据，而不需要 CPU（中央处理器）来帮忙当“中间人”。这样，CPU 就可以解放出来，去干其他重要的事，比如计算或处理别的任务，提高整个系统的效率。
想象一下：没有 DMA 时，CPU 像个勤快的搬运工，得亲自把数据从一个地方搬到另一个地方（比如从内存到 UART 发送出去）。但用 DMA 后，CPU 只需说一声“开始吧”，DMA 就自己干活了。
#### 3.3.2 DMA 如何跳过 CPU 的原理？
DMA 的核心是它有一个独立的“DMA 控制器”（一种硬件模块），这个控制器可以暂时“接管”系统的总线（数据高速公路），直接在内存和外设之间搬运数据。原理步骤简单如下：
    1. **初始化（CPU 参与一下）**：CPU 先设置好 DMA 的参数，比如“从内存的这个地址开始，传输多少字节数据，到 UART 外设去”。这就像给 DMA 下个命令单。
    2. **跳过 CPU 的传输过程**：DMA 控制器收到命令后，自己控制总线，直接从内存读取数据，然后写入外设（或反过来）。在这个过程中，CPU 不需要插手，它可以去执行其他代码。为什么能跳过？因为 DMA 控制器有能力“借用”总线，而不需要 CPU 每次都手动读写寄存器。
    3. **完成通知**：传输完了，DMA 会通过中断（或标志位）告诉 CPU：“我干完了！” CPU 再来检查结果或处理后续。
#### 3.3.3 两个模式
DMA 主要有两种工作模式：
|  | 普通模式 | 循环模式 |
|---|---|---|
| 适用场景 | 一次性数据传输，比如发送一段数据 | 持续不断的数据流，比如接收传感器数据 |
| 工作原理 | 传输完成后停止，需要重新配置 | 传输完成后自动重新开始，无需重新配置 |
| CPU 参与度 | 需要每次配置传输参数 | 只需初始配置一次，后续自动进行 |
| 地址更新 | 需要手动更新地址和长度 | 自动更新地址，适合连续数据 |
| 优点 | 简单高效，适合偶尔传输 | 省去重复配置的麻烦，适合连续数据 |
| 缺点 | 需要每次都重新设置 | 可能会覆盖旧数据，需谨慎管理 |
#### 3.3.4 总结
DMA优点是高效，尤其大数据量时（比如传输一堆传感器数据），CPU 不被占用，不会卡住系统。但缺点是配置有点复杂，需要硬件支持。

## 4. 代码示例
### 4.1 阻塞方式
#### 4.1.1 配置cubemx
其余常规配置别忘了
打开uart，`Mode`配置为`Asynchronous`异步通信，然后记住此时的波特率，数据位，校验位，停止位等参数，后续调试要用
![阻塞式cubemx](https://youke1.picui.cn/s1/2025/11/19/691d4b05e897c.png)
#### 4.1.2 代码编写
添加如下代码到`main.c`

**重点函数**：
- `HAL_UART_Transmit(&huart1, tx_buff, sizeof(tx_buff), HAL_MAX_DELAY);`：发送数据函数，参数分别是UART句柄、数据缓冲区、数据长度和超时时间。
- `HAL_UART_Receive(&huart1, rx_buff, sizeof(rx_buff), HAL_MAX_DELAY);`：接收数据函数，参数同上。
```c
int main(void)
{

  /* USER CODE BEGIN 1 */

  /* USER CODE END 1 */

  /* MCU Configuration--------------------------------------------------------*/

  /* Reset of all peripherals, Initializes the Flash interface and the Systick. */
  HAL_Init();

  /* USER CODE BEGIN Init */

  /* USER CODE END Init */

  /* Configure the system clock */
  SystemClock_Config();

  /* USER CODE BEGIN SysInit */

  /* USER CODE END SysInit */

  /* Initialize all configured peripherals */
  MX_GPIO_Init();
  MX_USART1_UART_Init();
  /* USER CODE BEGIN 2 */
  uint8_t tx_buff[] = "Hello, STM32!\r\n";
  HAL_UART_Transmit(&huart1, tx_buff, sizeof(tx_buff), HAL_MAX_DELAY);
  uint8_t rx_buff[20] = {0};
  /* USER CODE END 2 */

  /* Infinite loop */
  /* USER CODE BEGIN WHILE */
  while (1)
  {
    HAL_UART_Receive(&huart1, rx_buff, sizeof(rx_buff), HAL_MAX_DELAY);
    HAL_Delay(100);
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
  }
  /* USER CODE END 3 */
}
```
#### 4.1.3 Debug和上位机接收数据
把stm32连接到电脑，打开串口调试助手，**选择对应的串口号和波特率**，点击打开串口，下载代码到stm32。
运行后可以在串口助手看到发送的"Hello, STM32!"消息。然后在keil5的调试窗口观察rx_buff数组，可以看到接收到的数据"Hello World!"。
![阻塞式调试](https://youke1.picui.cn/s1/2025/11/19/691d554cba644.png)

### 4.2 中断方式
#### 4.2.1 **配置cubemx**
与阻塞方式类似，但要启用中断。在`NVIC Settings`中使能对应的`USARTx global interrupt`。
![中断式cubemx](https://youke1.picui.cn/s1/2025/11/19/691d56ea41bdd.png)
#### 4.2.2 **代码编写**

**重点函数**
- `HAL_UART_Receive_IT(&huart1, rx_buff, sizeof(rx_buff));`：启动接收中断函数，参数是UART句柄、数据缓冲区和数据长度。
- `HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart)`：发送完成回调函数，当数据发送完成时被调用。
- `HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)`：接收完成回调函数，当数据接收完成时被调用。
```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
  if (huart->Instance == USART1)
  {
    // 接收完成，发送一个确认信息
    uint8_t tx_buff[] = "Get It!\r\n";
    HAL_UART_Transmit_IT(&huart1, tx_buff, sizeof(tx_buff));
    
    // 再次启动中断接收，准备下一次数据接收
    HAL_UART_Receive_IT(&huart1, rx_buff, sizeof(rx_buff));
  }
}
/* USER CODE END 0 */

/**
  * @brief  The application entry point.
  * @retval int
  */
int main(void)
{

  /* USER CODE BEGIN 1 */

  /* USER CODE END 1 */

  /* MCU Configuration--------------------------------------------------------*/

  /* Reset of all peripherals, Initializes the Flash interface and the Systick. */
  HAL_Init();

  /* USER CODE BEGIN Init */

  /* USER CODE END Init */

  /* Configure the system clock */
  SystemClock_Config();

  /* USER CODE BEGIN SysInit */

  /* USER CODE END SysInit */

  /* Initialize all configured peripherals */
  MX_GPIO_Init();
  MX_USART1_UART_Init();
  /* USER CODE BEGIN 2 */

  /* USER CODE END 2 */

  /* Infinite loop */
  /* USER CODE BEGIN WHILE */
  while (1)
  {
    HAL_UART_Receive_IT(&huart1, rx_buff, sizeof(rx_buff));
    HAL_Delay(100);
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
  }
  /* USER CODE END 3 */
}
```
#### 4.2.3 Debug和上位机接收数据
调试方式与阻塞方式类似。下载代码到stm32，打开串口调试助手，选择对应的串口号和波特率，点击打开串口。运行后可以在串口助手看到发送的"Get It!"消息。然后在keil5的调试窗口观察rx_buff数组，可以看到接收到的数据"Hello World?"。
![中断式调试](https://youke1.picui.cn/s1/2025/11/19/691d59214031f.png)

### 4.3 DMA方式
#### 4.3.1 **配置cubemx**
与中断方式一致，但要启用DMA。在`DMA Settings`中添加对应的`USARTx_TX`和`USARTx_RX`通道。然后我们把USARTx_RX的DMA的模式改为`Circular`循环模式，这样可以实现连续接收数据而不需要每次都重新配置DMA。
![DMA式cubemx](https://youke1.picui.cn/s1/2025/11/19/691d5a467d163.png)
#### 4.3.2 **代码编写**
**重点函数**
- `HAL_UART_Receive_DMA(&huart1, rx_buff, sizeof(rx_buff));`：启动DMA接收函数，参数是UART句柄、数据缓冲区和数据长度。**这个函数要注意是哪一个模式，如果是循环模式只用调用一边即可。**
- `HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)`：接收完成回调函数，当数据接收完成时被调用。

代码部分与中断方式类似，但要换为DMA函数皆可：
```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
  if (huart->Instance == USART1)
  {
    // 接收完成，发送一个确认信息
    uint8_t tx_buff[] = "Get It!\r\n";
    HAL_UART_Transmit_DMA(&huart1, tx_buff, sizeof(tx_buff));
  }
}
/* USER CODE END 0 */

/**
  * @brief  The application entry point.
  * @retval int
  */
int main(void)
{

  /* USER CODE BEGIN 1 */

  /* USER CODE END 1 */

  /* MCU Configuration--------------------------------------------------------*/

  /* Reset of all peripherals, Initializes the Flash interface and the Systick. */
  HAL_Init();

  /* USER CODE BEGIN Init */

  /* USER CODE END Init */

  /* Configure the system clock */
  SystemClock_Config();

  /* USER CODE BEGIN SysInit */

  /* USER CODE END SysInit */

  /* Initialize all configured peripherals */
  MX_GPIO_Init();
  MX_DMA_Init();
  MX_USART1_UART_Init();
  /* USER CODE BEGIN 2 */
  HAL_UART_Receive_DMA(&huart1, rx_buff, sizeof(rx_buff));
  /* USER CODE END 2 */

  /* Infinite loop */
  /* USER CODE BEGIN WHILE */
  while (1)
  {
    HAL_Delay(100);
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
  }
  /* USER CODE END 3 */
}
```
注意循环模式下不需要在回调函数中再次启动接收，因为DMA会自动重新开始接收。
#### 4.3.3 Debug和上位机接收数据
现象完全一致不在赘述。

> Copyright (c) 2025 PHOENIX
> 作者： 何文轩 <wenx_public@163.com>
> 说明：本文件版权由杭州电子科技大学PHOENIX实验室持有