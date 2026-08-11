# 前言
memfault有这么一篇文章：
> From Zero to main(): Bare metal C

这个文章的措辞本身就很有意思，bare metal，裸露金属，表示原生的C。
文章开头还用了莎翁式的古英语：
- The entry point of thy program shall be named “main”.
> 你的程序的入口点应命名为“main”。
- Thou shalt initialize thy static variables, else The Machine shall set them to zero.
> 你必须自行初始化静态变量，否则系统会将其设置为零。
- Thou shalt implement The Interrupts. HardFault_Handler chief among them, but also SysTick_Handler.
> 你必须实现各种中断处理程序。其中，HardFault_Handler 是最重要的，SysTick_Handler 也同样重要

（这里的thou =you，shelt=shell，是圣经口吻的古英语）


于是它很快就引起了我的兴趣。是从什么时候开始，我们把标准库，startup当成真理一样崇拜与遵循？我们是不是缺失了一堂课，一堂告诉我们真正的嵌入式是什么的课？
所以我在这里写这篇文章，记录，学习到底什么是真正有趣的嵌入式。


# 祂说：要有电
显然MCU是从上电开始运行的。但是上电之后发生了什么？原作是用Atmel 公司的 SAMD21G18 处理器(Cortex-M0+内核,和rp2040一样)做例子的，参考的是ARMv6-M 架构参考手册,我这里用ARMv7-M来替代，在B1.5.5 
 Reset behavior这一节里记录着：
 ```c
 // TakeReset()
// ============
TakeReset()
   CurrentMode = Mode_Thread;
   PRIMASK<0> = '0';             
   FAULTMASK<0> = '0';           
  /* priority mask cleared at reset */
  /* fault mask cleared at reset */
   BASEPRI<7:0> = Zeros(8);        /* base priority disabled at reset */
    if HaveFPExt() then             /* initialize the Floating Point Extn */
        CONTROL<2:0> = '000';       /* FP inactive, stack is Main, thread is privileged */
        CPACR.cp10 = '00';
        CPACR.cp11 = '00';
        FPDSCR.AHP = '0';
        FPDSCR.DN = '0';
        FPDSCR.FZ = '0';
        FPDSCR.RMode = '00';
        FPCCR.ASPEN = '1';
        FPCCR.LSPEN = '1';
        FPCCR.LSPACT = '0';
        FPCAR = bits(32) UNKNOWN;
        FPFSR = bits(32) UNKNOWN;
        for i = 0 to 31
            S[i] = bits(32) UNKNOWN;
    else
        CONTROL<1:0> = '00';        /* current stack is Main, thread is privileged */
    for i = 0 to 511                /* all exceptions Inactive */
        ExceptionActive[i] = '0';
    ResetSCSRegs();                 /* catch-all function for System Control Space reset */
    ClearExclusiveLocal(ProcessorID()); /* Synchronization (LDREX* / STREX*) monitor support */
    ClearEventRegister();           /* see WFE instruction for more details */
    for i = 0 to 12
        R[i] = bits(32) UNKNOWN;
    bits(32) vectortable = VTOR<31:7>:'0000000';
    SP_main = MemA_with_priv[vectortable, 4, AccType_VECTABLE] AND 0xFFFFFFFC<31:0>;
    SP_process = ((bits(30) UNKNOWN):'00');
    LR = 0xFFFFFFFF<31:0>;          /* preset to an illegal exception return value */
    tmp = MemA_with_priv[vectortable+4, 4, AccType_VECTABLE];
    tbit = tmp<0>;
    APSR = bits(32) UNKNOWN;        /* flags UNPREDICTABLE from reset */
    IPSR<8:0> = Zeros(9);           /* Exception Number cleared */
    EPSR.T = tbit;                  /* T bit set from vector */
    EPSR.IT<7:0> = Zeros(8);        /* IT/ICI bits cleared */
    BranchTo(tmp AND 0xFFFFFFFE<31:0>);  /* address of reset service routine */
 ```
 手册里指出，触发复位的时候进程会在不保存寄存器的情况下，按照TakeReset里的行为执行，而这个TakeRest的定义就是以上的伪代码。简单阅览就能发现，开头重置了中断屏蔽MASK，切换为线程模式（这里之前Cortex-M相关文章有讲），接着` if HaveFPExt() `之后的一大段就是重新给FPU相关寄存器赋予一个默认关闭的默认值。而`ExceptionActivel`相关是用于清空各种异常状态的，原文中如此描述：

 > ExceptionActive[*]是一个概念性的数组，包含所有异常的活动标志位，这意味着它具有针对固定优先级系统异常、可配置优先级系统异常和外部中断的活动标志。固定优先级异常的活动标志仅为概念性标志，并不要求存在于系统寄存器中。

 之后，模拟程序（实际上是硬件行为）继续清空寄存器，对于通用寄存器一般不处理（会被覆盖）接着从向量表寄存器VTOR取了0字节指针`vectortable`，再把栈指针SP指向其前4个字节（向量表一个项4字节），也就是第一项。接着从这个地址末尾读取4个字节，得到Reset_Handler的地址，作为PC指针跳过去。
 之后的内容就是为了防止返回等非法操作，处理一下thumb指令集，进行收尾。


 **总而言之**，就是:
 - 将向量表重置为 [0x00000000]
 - SP指向 [0x00000000]（VTOR第一个表项）
 - PC指向 [0x00000004] (第二个表项)

 # 我从哪来？
 那么我们知道了会跑到0X00000004去执行代码，但是这是我们的main吗？为什么一定要取名main呢？我们必须看看这个地址到底放了什么东西，因此我们要看看编译产生的文件。GCC产生的通常是elf格式的，keil产生的只有hex和axf，二者略有差异，因此我们先搞清楚这些文件的区别，才能通过分析工具看到具体地址里放了什么。
 ## .axf —— ARM Executable File（Keil 调试文件）
 .axf内部实际上是 ELF 格式（包含文件头、段表、代码、数据、调试信息），其储存了原始代码，调试信息，还有二进制代码，是最大的部分。通常来说这个不是直接烧录的，而是用于调试器使用的，比如ozone就依赖这个文件寻找到对应的代码和汇编关系。
 ## .elf —— 可执行可链接格式文件
 .elf是链接器的产物，大部分 GCC 工具链（如 ARM GCC）直接输出 .elf,内容与axf几乎完全一致，基本上二者可以互相改后缀名。
 ## .hex —— Intel HEX 格式烧录文件
 .hex是keil等实际烧录的文件，是一种文本文件，每一行以 : 开始，包含数据长度、地址、记录类型、数据、校验和。它比.bin略多一些文件头之类，因此不能直接执行（比如qemu）,但是可以直接烧录，其提供了执行地址，并且保存了具体的内存分段等。
## .bin —— 二进制烧录文件
 .bin仅包含原始字节，没有地址、没有校验、没有分段信息,烧录的时候需要指定地址之类，一般是配合bootloader使用的。

 # 在windows上分析
 由于我们手头没有bin，最多只有hex和axf，因此我们首先得对hex做处理。这里以我的P-RTLog的hex文件做处理（因为它有一个储存日志哈希的段，所以文件会比较大）
 ```PowerShell
 arm-none-eabi-objcopy -I ihex -O binary CubeMX_Config.hex CubeMX_Config.bin
 ```
 这行命令的意思是输入HEX输出二进制bin。
 但是bin文件还是不能直接看的，我们不是linux可以xxd，windows里面文件后缀名要可以作为文本查看得有一小段描述文件本身的文件头，bin则不是这样的。因此我们做个转换：
 ```PowerShell
 Format-Hex -Path CubeMX_Config.bin -Count 256
 ```
 然后它输出
 ```
 0x00000000  EE 8D A9 BA 8F 0E 89 C9 01 00 00 00 59 00 00 00  ............Y...
00000010  00 E2 96 A0 6D 73 67 E2 99 A6 49 20 61 6D 20 74  ....msg...I am t
00000020  68 65 20 73 74 6F 72 6D 20 74 68 61 74 20 69 73  he storm that is
00000030  20 61 70 70 72 6F 61 63 68 69 6E 67 EF BC 81 E2   approaching....
00000040  96 A0 6D 6F 64 75 6C 65 E2 99 A6 64 65 66 61 75  ..module...defau
00000050  6C 74 E2 96 A0 66 69 6C 65 E2 99 A6 2E 2E 2F 53  lt...file...../S
00000060  72 63 2F 6D 61 69 6E 2E 63 00 EE 8D A9 BA 01 28  rc/main.c......(
00000070  0F B3 01 00 00 00 51 00 00 00 00 E2 96 A0 6D 73  ......Q.......ms
00000080  67 E2 99 A6 48 6F 77 20 6D 61 6E 79 20 74 69 6D  g...How many tim
00000090  65 73 20 68 61 76 65 20 77 65 20 66 69 67 68 74  es have we fight
000000A0  3F E2 96 A0 6D 6F 64 75 6C 65 E2 99 A6 64 65 66  ?...module...def
 ```
 OPS!我们看到了什么玩意？！
 这是为什么呢？
 ~~嵌入式笑传之fakerfakerbin~~
我们看到我们的链接脚本
```
; *************************************************************
; *** Stable Scatter-Loading Description for P-RTLog + AC6  ***
; *************************************************************
;
; This file lives outside the generated output directory so it will not be
; overwritten by rebuilds or project regeneration.
;
; Tokenizer metadata: keep in AXF for host-side decode, do not place in flash image.
LR_PW_TOKENIZER 0x00000000 {
  ER_PW_TOKENIZER 0x00000000 {
   * (.pw_tokenizer.entries)
  }
}

; Main image regions
LR_IROM1 0x08000000 0x00100000  {
  ER_IROM1 0x08000000 0x00100000  {
   *.o (RESET, +First)
   *(InRoot$$Sections)
   .ANY (+RO)
   .ANY (+XO)
  }
  RW_IRAM1 0x20000000 0x00020000  {
   .ANY (+RW +ZI)
  }
  RW_IRAM2 0x24000000 0x00050000  {
   .ANY (+RW +ZI)
  }
}

```
从sct可以看出来，实际上keil常把代码烧录在0x0x08000000
而前面这一段我用于储存了测试用的rtlog:
```
LR_PW_TOKENIZER 0x00000000 {
  ER_PW_TOKENIZER 0x00000000 {
   * (.pw_tokenizer.entries)
  }
}
```
但是我们前面不是说代码跳转到0x0000000执行吗？这样岂不是有问题？
其实没问题，这是描述不同的原因：
- ARMv7-M 手册（B1.5.5）描述的是内核行为：复位后 VTOR = 0，从地址 0x00000000 读 MSP、从 0x00000004 读 Reset_Handler。这对架构是对的——内核只认这个地址。
- 但芯片厂商 ST 在设计时，把 0x00000000 ~ 0x1FFFFFFF 这段 Cortex-M 的 code 区做成了可重映射区。STM32 参考手册的 Memory map 图里明确写着：
```
0x00000000 ─┐  ┌─> 主 Flash  (0x08000000)   ← BOOT0=0（默认，绝大多数产品）
            ├──┤─> 系统存储器 (0x1FFF0000)   ← BOOT0=1, BOOT1=0（内置 bootloader）
            ┘  └─> SRAM      (0x20000000)   ← BOOT0=1, BOOT1=1
```
所以我们实际的代码应该是在0x08000000,之所以原文可以直接看bin前几段，是因为bin不包含地址，它并不关心你是从0x08000000开始还是从0x00000000开始,因此如果前面没有数据，你就可以直接看到代码。
呃，但是为什么要这么做？如果你看过linux什么的代码的话会发现，他们也喜欢在内存的开始段留些空，因为那些一般是留给内存向量表什么的用的，我这里岂不是给它覆盖了？
答案很简单，我烧进去了吗？没呀！  
Keil 的 Flash 下载只针对 flash 算法覆盖的地址范围（0x08000000），LR_PW_TOKENIZER 这个加载域在下载时会被跳过——它只存在于 AXF/HEX 文件里，供主机端工具（host-side decoder）提取 token 表。SCT 注释里那句 "keep in AXF for host-side decode, do not place in flash image" 就是这个意图。
______________________________________________________________________________
好，现在我们回来专注于bin本身。  
```
=== 0x08000000 Flash 固件区 (前 256 字节) ===
0x08000000  C8 0B 00 24 99 03 00 08 99 24 00 08 89 24 00 08  ...$.....$...$..
08000010  91 24 00 08 89 06 00 08 B9 35 00 08 00 00 00 00  .$.......5......
08000020  00 00 00 00 00 00 00 00 00 00 00 00 C1 29 00 08  .............)..
08000030  91 06 00 08 00 00 00 00 A1 24 00 08 C9 29 00 08  .........$...)..
08000040  B7 03 00 08 B7 03 00 08 B7 03 00 08 B7 03 00 08  ................
08000050  B7 03 00 08 B7 03 00 08 B7 03 00 08 B7 03 00 08  ................
08000060  B7 03 00 08 B7 03 00 08 B7 03 00 08 B7 03 00 08  ................
08000070  B7 03 00 08 B7 03 00 08 B7 03 00 08 B7 03 00 08  ................
08000080  B7 03 00 08 B7 03 00 08 B7 03 00 08 B7 03 00 08  ................
08000090  B7 03 00 08 B7 03 00 08 B7 03 00 08 B7 03 00 08  ................
```
如果按照我们前面认为的，第一个4个字节就应该是初始SP和初始地址指针：
栈顶指针SP=0x24000BC8(记得按小端顺序拼喔)
起始地址指针=0x08000399  
那么这个起始地址指针应该指向main，好打完收工，我们教程结束。 (*^▽^*)  
真是这样吗？   
我们去符号表里找找这俩玩意。  
```PowerShell
PS D:\CODES\Robomaster\Codes_Update\P-RTLog\example\MDK-ARM\CubeMX_Config> arm-none-eabi-objdump -t "D:\CODES\Robomaster\Codes_Update\P-RTLog\example\MDK-ARM\CubeMX_Config\CubeMX_Config.axf" | Sort-Object

00000000 g       *ABS*  00000000 .hidden BuildAttributes$$THM_ISAv4$E$P$D$K$B$S$7EM$VFPv5_D16$PE$PLD8$A:L22UL41UL21$X:L11$S22US41US21$IEEE1$IW$~IW$USESV6$~STKCKD$USESV7$~SHL$OTIME$ROPI$EBA8$UX$STANDARDLIB$REQ8$PRES8$EABIv2
00000000 l       ER_PW_TOKENIZER        00000000 __tagsym$$used.0
00000000 l     O ER_PW_TOKENIZER        0000006a main._pw_tokenizer_string_entry_109_0
00000000 l    df *ABS*  00000000 ../clib/angel/boardlib.s
00000000 l    df *ABS*  00000000 ../clib/angel/boardlib.s
00000000 l    df *ABS*  00000000 ../clib/angel/boardlib.s
00000000 l    df *ABS*  00000000 ../clib/angel/boardlib.s
```
太长了，我们搜一下：

```
08000398  w    F ER_IROM1       0000000c .hidden Reset_Handler
08000398 l       ER_IROM1       00000000 $v0
```
很奇怪，我们搜不到399.但是在这附近找到了个类似的内容，而且也不是_main。
Copliot这么解释的：

>符号表里只有 0x08000398：objdump -t 看到的是 08000398 w F ER_IROM1 0000000c .hidden Reset_Handler，bit0 = 0。

>向量表里写的是 0x08000399：Cortex-M 要求向量表存的是带 Thumb 位的地址（bit0 = 1 表示 Thumb 指令）。所以 bin 里看到 99 03 00 08，等于 Reset_Handler + 1。

>这是 CPU 硬件约定：取向量时硬件把 bit0 剥掉后跳转到 0x08000398 执行，所以永远不会有一个符号叫 0x08000399——它只是"Reset_Handler 的地址 | 1"的编码值。

因此这段就是我们的初始地址指针，指向的是Reset_Handler。不是main，对吧？
这是为什么呢？大致地讲，这是为了处理前面硬件摆烂不管的寄存器，以及初始化很多库之类的，使得进入main
之后你可以放心大胆地执行代码。
但是为了真正地理解，我们还是看看代码.......  
哦不，

> Unfortunately, the Reset_Handler is often an inscrutable mess of Assembly code.

看起来我们如果不想被arm汇编肘击的话（这个下一期再讲），我们最好是自己写一个功能一致的简单Reset_Handler。
> 魂五：接下来，ARM 的技术参考手册会很有用

接下来我们要做的内容可以用一张表涵盖：


| 行动 | 描述 |
|---|---|
| 初始化变量 | 所有的全局/静态变量都必须进行初始化处理。这包括将 BSS 区域的变量初始化为 0，同时将那些非常量变量的初始值从 ROM 复制到 RAM 中。 |
| 设置堆栈 | 如果使用多个堆栈，那么其他被暂存的 SP 值必须被重新初始化。当前 SP 值也可以从"Main"模式更改为"Process"模式。 |
| 初始化所有运行时相关内容 | 可以选择调用 C/C++ 运行时初始化代码，从而启用堆内存、浮点运算等功能。通常，这一操作是由 C/C++ 库中的 `__main` 来完成的。 |


这就是所谓的CRT(C Runtime)。
> All objects with static storage duration shall be initialized (set to their initial values) before program startup. The manner and timing of such initialization are otherwise unspecified.
>
>所有具有静态存储特性的对象，都必须在程序启动之前被初始化（即被设置为它们的初始值）。至于具体的初始化方式及时间点，则没有明确的规定。

但是正如原文所说的，我们不可能逐个去初始化每个变量。相反，我们必须依靠编译器（更准确地说，是链接器）来把所有变量放到同一个位置，这样我们才能一次性将它们全部初始化。因此编译器提供了一些编译拓展：
对于那些必须被置为零的静态变量，链接器会提供 _sbss 和 _ebss 作为这些变量的起始地址和结束地址。因此，我们可以这样处理：
```c
/* Clear the zero segment */
for (uint32_t *bss_ptr = &_sbss; bss_ptr < &_ebss;) {
    *bss_ptr++ = 0;
}
```
对于那些具有初始化值的静态变量，链接器会进行如下处理：

1. _etext ：作为存储初始化值的地址。

2. _sdata ：即静态变量所存储的地址。

3. _edata 作为静态变量内存的末尾位置
所以我们可以写个循环来赋值
```c
uint32_t *init_values_ptr = &_etext;
uint32_t *data_ptr = &_sdata;
if (init_values_ptr != data_ptr) {
    for (; data_ptr < &_edata;) {
        *data_ptr++ = *init_values_ptr++;
    }
}
```
值得一提的是，这里的 `*data_ptr++ = *init_values_ptr++;`是个值得推敲的语法，虽然++会优先于*起作用，但是`init_values_ptr`指向的值依然会更新为data_ptr的旧值，而不是被data_ptr++后的值。
简单来说就是这样：
```
init_values_ptr  ──▶  [ 初始值 ][ 初始值 ][ 初始值 ] ...
                       ▲用了这个                  ▲随后指针挪到这

data_ptr         ──▶  [ 变量A  ][ 变量B  ][ 变量C  ] ...
                       ▲写进这里                  ▲随后指针挪到这
```
之所以这么写，是因为这条 C 语句和 Cortex-M 的后变址寻址（post-indexed addressing） 是一一对应的。
例如：
```c
LDR  r3, [r1], #4    ; 加载 [r1]，然后 r1 += 4
STR  r3, [r0], #4    ; 存储到 [r0]，然后 r0 += 4
```
如果维护一个索引index，使用循环进行赋值，在 CPU 上每轮要( i++) + 计算 data + i*4 两次加法，而指针版每轮只有两次后变址访问。
顺便一提如果你这么玩：
```c
*p++ = *p++;   // ❌ 未定义行为：同一对象 p 被两次未排序的 ++ 修改
```
绝对会被你玩坏的。  

## 最终的Reset_Handel
原文提供的是：
```C
void Reset_Handler(void)
{
    /* Copy init values from text to data */
    uint32_t *init_values_ptr = &_etext;
    uint32_t *data_ptr = &_sdata;

    if (init_values_ptr != data_ptr) {
        for (; data_ptr < &_edata;) {
            *data_ptr++ = *init_values_ptr++;
        }
    }

    /* Clear the zero segment */
    for (uint32_t *bss_ptr = &_sbss; bss_ptr < &_ebss;) {
        *bss_ptr++ = 0;
    }

    /* Overwriting the default value of the NVMCTRL.CTRLB.MANW bit (errata reference 13134) */
    NVMCTRL->CTRLB.bit.MANW = 1;

    /* Branch to main function */
    main();

    /* Infinite loop */
    while (1);
}
```
我们注意到，这里不仅最终指向了main，并且还留了个while(1)避免程序意外返回后跑飞。
但是，我们也注意到了，除了我们上述说好的变量初始化之外，还有
```C
 /* Overwriting the default value of the NVMCTRL.CTRLB.MANW bit (errata reference 13134) */
    NVMCTRL->CTRLB.bit.MANW = 1;
```

这一行,看起来很令人困惑。copilot是这么解释的：


> 这一行是 SAMD21 芯片特有的**勘误补丁**。NVMCTRL（Non-Volatile Memory Controller）是 Atmel 系芯片管理 Flash 擦写的外设。它默认处于**自动写模式**（MANW=0）：往 Flash 地址写数据，控制器自动触发擦写，一步到位。但勘误表 13134 号条目指出，自动模式在写操作撞上复位、调试器介入等时机时可能写失败甚至损坏数据。
>
> 所以 Reset_Handler 里第一时间把 `MANW` 置 1，强制切到**手动写模式**：必须先写 `DATA` 寄存器、再发 Write Page 命令才执行擦写，把写时序的控制权收回到软件手里。
>
> 为什么放在这里？因为复位后寄存器回到默认值（MANW=0），而 Reset_Handler 是复位后第一段代码——必须在**任何可能写 Flash 的代码之前**修正这个默认值。它和前面的 `.data` 拷贝、`.bss` 清零是同一性质的工作：**在进入 main 之前，把硬件摆烂不管的默认状态修正成可用状态**。只不过前两个修的是 RAM，这个修的是 Flash 控制器的脾气。
>
> 这也是真实世界 Reset_Handler 的常态：**90% 是通用的 CRT 搬砖，10% 是芯片特有的"擦屁股"**——所以没人愿意手写它，都直接抄厂商的 startup 文件。

---

实际上原文也解释了，对于芯片中的缺陷，最好在我们的程序开始执行之前就加以处理。有时候，这些缺陷会被封装在名为 SystemInit 的函数中，而该函数则由 Reset_Handler 在 main 之前调用。Nordic 公司就是采用这种处理方式。
至于其他寄存器，我们通常来说不清理，因为其会被自动覆盖，除非我们有意地使用其值。在keil提供的startup中，其更加保险：
```
LDR  r0, =__initial_sp
MSR  MSP, r0          ; ① 再设一次 SP —— 有些工具链/调试场景不信任硬件加载
BL   SystemInit        ; ② ST 系：配时钟 PLL、开 FPU（SAMD21 不需要，所以原文没有）
BL   __main            ; ③ C 库：scatterload + 堆栈/堆 + C 运行时初始化
```
至此，我们简单地实现了从零开始的嵌入式。但是其中很多概念，包括thumb指令集，还有连接脚本，我们都了解的不够透彻，在下次的文章中我会学习并且谈到这些。
