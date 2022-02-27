---
title:      STM32从原理入门(一)-点灯(上篇)
date:       2020-07-26 15:07:00
subtitle:   嵌入式软件的Hello World
author:     Letter
cover:      
catalog:    true
categories: STM32
tags:       嵌入式
            STM32
---

## 前言

有了基本概念之后，下一步要做的，就是搭建好适合自己的开发环境。STM32可以选择的开发环境很多，这个我在上一篇文章里面也提到过，对于初学者，这里就直接选择keil吧，同时搭配STM32Cube MX做工程配置。当然，一开始的工程肯定不能使用STM32Cube MX生成，我们需要手动创建工程，只有这样才能对工程的结构有深入的了解

## 准备工作

1. 安装开发环境

    在编写程序之前，请确认已经安装好keil软件，并安装芯片对应的包，这部分网上教程很多，请自行查找

2. 准备好开发板

    嵌入式开发离不开硬件，请确保至少要有一块开发板或者自己制作PCB，本篇文章的内容，至少需要用到一个LED资源

3. 准备好文档

    芯片手册和开发板原理图是必须要的，本篇文章所使用的芯片是STM32F407系列的，所以我准备了STM32F4xx参考手册，具体手册可以到ST官网下载(学会查找资料也是一个重要的技能）

## 建立工程

打开keil软件，通过project选项卡新建工程，选择工程目录和工程名，选择对应的芯片，创建工程，此时，keil可能会弹出一下的窗口

![Manager RT Env](https://s1.ax1x.com/2020/07/26/a9f9Mt.png)

这是keil本身支持的组件管理工具，我们可以在这里直接添加一些包，比如说文件管理之类的，由于我们是第一次使用，目的不是为了快速开发，而是学习，所以我们直接退出这个窗口，手动添加所有文件

在左侧的project窗口中，展开Target1，右键单击Source Group1，选择添加新文件

![add file](https://s1.ax1x.com/2020/07/26/a9fjmV.png)

这里我们首先添加一个.s文件，命名为start.s，文件放置在工程目录的src子文件夹中

![start.s](https://s1.ax1x.com/2020/07/26/a9hMpd.png)

然后编辑这个文件，内容如下：

```asm
;******************** (C) COPYRIGHT 2017 STMicroelectronics ********************
;* File Name          : startup_stm32f407xx.s
;* Author             : MCD Application Team
;* Description        : STM32F407xx devices vector table for MDK-ARM toolchain.
;*                      This module performs:
;*                      - Set the initial SP
;*                      - Set the initial PC == Reset_Handler
;*                      - Set the vector table entries with the exceptions ISR address
;*                      - Branches to __main in the C library (which eventually
;*                        calls main()).
;*                      After Reset the CortexM4 processor is in Thread mode,
;*                      priority is Privileged, and the Stack is set to Main.
;* <<< Use Configuration Wizard in Context Menu >>>
;*******************************************************************************
;
;* Redistribution and use in source and binary forms, with or without modification,
;* are permitted provided that the following conditions are met:
;*   1. Redistributions of source code must retain the above copyright notice,
;*      this list of conditions and the following disclaimer.
;*   2. Redistributions in binary form must reproduce the above copyright notice,
;*      this list of conditions and the following disclaimer in the documentation
;*      and/or other materials provided with the distribution.
;*   3. Neither the name of STMicroelectronics nor the names of its contributors
;*      may be used to endorse or promote products derived from this software
;*      without specific prior written permission.
;*
;* THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
;* AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
;* IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
;* DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
;* FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
;* DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
;* SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
;* CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
;* OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
;* OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
;
;*******************************************************************************

; Amount of memory (in bytes) allocated for Stack
; Tailor this value to your application needs
; <h> Stack Configuration
;   <o> Stack Size (in Bytes) <0x0-0xFFFFFFFF:8>
; </h>

Stack_Size      EQU     0x00000400

                AREA    STACK, NOINIT, READWRITE, ALIGN=3
Stack_Mem       SPACE   Stack_Size
__initial_sp


; <h> Heap Configuration
;   <o>  Heap Size (in Bytes) <0x0-0xFFFFFFFF:8>
; </h>

Heap_Size       EQU     0x00000200

                AREA    HEAP, NOINIT, READWRITE, ALIGN=3
__heap_base
Heap_Mem        SPACE   Heap_Size
__heap_limit

                PRESERVE8
                THUMB


; Vector Table Mapped to Address 0 at Reset
                AREA    RESET, DATA, READONLY
                EXPORT  __Vectors
                EXPORT  __Vectors_End
                EXPORT  __Vectors_Size

__Vectors       DCD     __initial_sp               ; Top of Stack
                DCD     Reset_Handler              ; Reset Handler
                DCD     NMI_Handler                ; NMI Handler
                DCD     HardFault_Handler          ; Hard Fault Handler
                DCD     MemManage_Handler          ; MPU Fault Handler
                DCD     BusFault_Handler           ; Bus Fault Handler
                DCD     UsageFault_Handler         ; Usage Fault Handler
                DCD     0                          ; Reserved
                DCD     0                          ; Reserved
                DCD     0                          ; Reserved
                DCD     0                          ; Reserved
                DCD     SVC_Handler                ; SVCall Handler
                DCD     DebugMon_Handler           ; Debug Monitor Handler
                DCD     0                          ; Reserved
                DCD     PendSV_Handler             ; PendSV Handler
                DCD     SysTick_Handler            ; SysTick Handler

                ; External Interrupts
                DCD     WWDG_IRQHandler                   ; Window WatchDog
                DCD     PVD_IRQHandler                    ; PVD through EXTI Line detection
                DCD     TAMP_STAMP_IRQHandler             ; Tamper and TimeStamps through the EXTI line
                DCD     RTC_WKUP_IRQHandler               ; RTC Wakeup through the EXTI line
                DCD     FLASH_IRQHandler                  ; FLASH
                DCD     RCC_IRQHandler                    ; RCC
                DCD     EXTI0_IRQHandler                  ; EXTI Line0
                DCD     EXTI1_IRQHandler                  ; EXTI Line1
                DCD     EXTI2_IRQHandler                  ; EXTI Line2
                DCD     EXTI3_IRQHandler                  ; EXTI Line3
                DCD     EXTI4_IRQHandler                  ; EXTI Line4
                DCD     DMA1_Stream0_IRQHandler           ; DMA1 Stream 0
                DCD     DMA1_Stream1_IRQHandler           ; DMA1 Stream 1
                DCD     DMA1_Stream2_IRQHandler           ; DMA1 Stream 2
                DCD     DMA1_Stream3_IRQHandler           ; DMA1 Stream 3
                DCD     DMA1_Stream4_IRQHandler           ; DMA1 Stream 4
                DCD     DMA1_Stream5_IRQHandler           ; DMA1 Stream 5
                DCD     DMA1_Stream6_IRQHandler           ; DMA1 Stream 6
                DCD     ADC_IRQHandler                    ; ADC1, ADC2 and ADC3s
                DCD     CAN1_TX_IRQHandler                ; CAN1 TX
                DCD     CAN1_RX0_IRQHandler               ; CAN1 RX0
                DCD     CAN1_RX1_IRQHandler               ; CAN1 RX1
                DCD     CAN1_SCE_IRQHandler               ; CAN1 SCE
                DCD     EXTI9_5_IRQHandler                ; External Line[9:5]s
                DCD     TIM1_BRK_TIM9_IRQHandler          ; TIM1 Break and TIM9
                DCD     TIM1_UP_TIM10_IRQHandler          ; TIM1 Update and TIM10
                DCD     TIM1_TRG_COM_TIM11_IRQHandler     ; TIM1 Trigger and Commutation and TIM11
                DCD     TIM1_CC_IRQHandler                ; TIM1 Capture Compare
                DCD     TIM2_IRQHandler                   ; TIM2
                DCD     TIM3_IRQHandler                   ; TIM3
                DCD     TIM4_IRQHandler                   ; TIM4
                DCD     I2C1_EV_IRQHandler                ; I2C1 Event
                DCD     I2C1_ER_IRQHandler                ; I2C1 Error
                DCD     I2C2_EV_IRQHandler                ; I2C2 Event
                DCD     I2C2_ER_IRQHandler                ; I2C2 Error
                DCD     SPI1_IRQHandler                   ; SPI1
                DCD     SPI2_IRQHandler                   ; SPI2
                DCD     USART1_IRQHandler                 ; USART1
                DCD     USART2_IRQHandler                 ; USART2
                DCD     USART3_IRQHandler                 ; USART3
                DCD     EXTI15_10_IRQHandler              ; External Line[15:10]s
                DCD     RTC_Alarm_IRQHandler              ; RTC Alarm (A and B) through EXTI Line
                DCD     OTG_FS_WKUP_IRQHandler            ; USB OTG FS Wakeup through EXTI line
                DCD     TIM8_BRK_TIM12_IRQHandler         ; TIM8 Break and TIM12
                DCD     TIM8_UP_TIM13_IRQHandler          ; TIM8 Update and TIM13
                DCD     TIM8_TRG_COM_TIM14_IRQHandler     ; TIM8 Trigger and Commutation and TIM14
                DCD     TIM8_CC_IRQHandler                ; TIM8 Capture Compare
                DCD     DMA1_Stream7_IRQHandler           ; DMA1 Stream7
                DCD     FMC_IRQHandler                    ; FMC
                DCD     SDIO_IRQHandler                   ; SDIO
                DCD     TIM5_IRQHandler                   ; TIM5
                DCD     SPI3_IRQHandler                   ; SPI3
                DCD     UART4_IRQHandler                  ; UART4
                DCD     UART5_IRQHandler                  ; UART5
                DCD     TIM6_DAC_IRQHandler               ; TIM6 and DAC1&2 underrun errors
                DCD     TIM7_IRQHandler                   ; TIM7
                DCD     DMA2_Stream0_IRQHandler           ; DMA2 Stream 0
                DCD     DMA2_Stream1_IRQHandler           ; DMA2 Stream 1
                DCD     DMA2_Stream2_IRQHandler           ; DMA2 Stream 2
                DCD     DMA2_Stream3_IRQHandler           ; DMA2 Stream 3
                DCD     DMA2_Stream4_IRQHandler           ; DMA2 Stream 4
                DCD     ETH_IRQHandler                    ; Ethernet
                DCD     ETH_WKUP_IRQHandler               ; Ethernet Wakeup through EXTI line
                DCD     CAN2_TX_IRQHandler                ; CAN2 TX
                DCD     CAN2_RX0_IRQHandler               ; CAN2 RX0
                DCD     CAN2_RX1_IRQHandler               ; CAN2 RX1
                DCD     CAN2_SCE_IRQHandler               ; CAN2 SCE
                DCD     OTG_FS_IRQHandler                 ; USB OTG FS
                DCD     DMA2_Stream5_IRQHandler           ; DMA2 Stream 5
                DCD     DMA2_Stream6_IRQHandler           ; DMA2 Stream 6
                DCD     DMA2_Stream7_IRQHandler           ; DMA2 Stream 7
                DCD     USART6_IRQHandler                 ; USART6
                DCD     I2C3_EV_IRQHandler                ; I2C3 event
                DCD     I2C3_ER_IRQHandler                ; I2C3 error
                DCD     OTG_HS_EP1_OUT_IRQHandler         ; USB OTG HS End Point 1 Out
                DCD     OTG_HS_EP1_IN_IRQHandler          ; USB OTG HS End Point 1 In
                DCD     OTG_HS_WKUP_IRQHandler            ; USB OTG HS Wakeup through EXTI
                DCD     OTG_HS_IRQHandler                 ; USB OTG HS
                DCD     DCMI_IRQHandler                   ; DCMI  
                DCD     0                                 ; Reserved
                DCD     HASH_RNG_IRQHandler               ; Hash and Rng
                DCD     FPU_IRQHandler                    ; FPU


__Vectors_End

__Vectors_Size  EQU  __Vectors_End - __Vectors

                AREA    |.text|, CODE, READONLY

; Reset handler
Reset_Handler    PROC
                 EXPORT  Reset_Handler             [WEAK]
;        IMPORT  SystemInit
;        IMPORT  __main
        IMPORT  main

;                 LDR     R0, =SystemInit
;                 BLX     R0
;                 LDR     R0, =__main
;                 BX      R0
                 LDR     R0, =main
                 BX      R0
                 ENDP

; Dummy Exception Handlers (infinite loops which can be modified)

NMI_Handler     PROC
                EXPORT  NMI_Handler                [WEAK]
                B       .
                ENDP
HardFault_Handler\
                PROC
                EXPORT  HardFault_Handler          [WEAK]
                B       .
                ENDP
MemManage_Handler\
                PROC
                EXPORT  MemManage_Handler          [WEAK]
                B       .
                ENDP
BusFault_Handler\
                PROC
                EXPORT  BusFault_Handler           [WEAK]
                B       .
                ENDP
UsageFault_Handler\
                PROC
                EXPORT  UsageFault_Handler         [WEAK]
                B       .
                ENDP
SVC_Handler     PROC
                EXPORT  SVC_Handler                [WEAK]
                B       .
                ENDP
DebugMon_Handler\
                PROC
                EXPORT  DebugMon_Handler           [WEAK]
                B       .
                ENDP
PendSV_Handler  PROC
                EXPORT  PendSV_Handler             [WEAK]
                B       .
                ENDP
SysTick_Handler PROC
                EXPORT  SysTick_Handler            [WEAK]
                B       .
                ENDP

Default_Handler PROC

                EXPORT  WWDG_IRQHandler                   [WEAK]
                EXPORT  PVD_IRQHandler                    [WEAK]
                EXPORT  TAMP_STAMP_IRQHandler             [WEAK]
                EXPORT  RTC_WKUP_IRQHandler               [WEAK]
                EXPORT  FLASH_IRQHandler                  [WEAK]
                EXPORT  RCC_IRQHandler                    [WEAK]
                EXPORT  EXTI0_IRQHandler                  [WEAK]
                EXPORT  EXTI1_IRQHandler                  [WEAK]
                EXPORT  EXTI2_IRQHandler                  [WEAK]
                EXPORT  EXTI3_IRQHandler                  [WEAK]
                EXPORT  EXTI4_IRQHandler                  [WEAK]
                EXPORT  DMA1_Stream0_IRQHandler           [WEAK]
                EXPORT  DMA1_Stream1_IRQHandler           [WEAK]
                EXPORT  DMA1_Stream2_IRQHandler           [WEAK]
                EXPORT  DMA1_Stream3_IRQHandler           [WEAK]
                EXPORT  DMA1_Stream4_IRQHandler           [WEAK]
                EXPORT  DMA1_Stream5_IRQHandler           [WEAK]
                EXPORT  DMA1_Stream6_IRQHandler           [WEAK]
                EXPORT  ADC_IRQHandler                    [WEAK]
                EXPORT  CAN1_TX_IRQHandler                [WEAK]
                EXPORT  CAN1_RX0_IRQHandler               [WEAK]
                EXPORT  CAN1_RX1_IRQHandler               [WEAK]
                EXPORT  CAN1_SCE_IRQHandler               [WEAK]
                EXPORT  EXTI9_5_IRQHandler                [WEAK]
                EXPORT  TIM1_BRK_TIM9_IRQHandler          [WEAK]
                EXPORT  TIM1_UP_TIM10_IRQHandler          [WEAK]
                EXPORT  TIM1_TRG_COM_TIM11_IRQHandler     [WEAK]
                EXPORT  TIM1_CC_IRQHandler                [WEAK]
                EXPORT  TIM2_IRQHandler                   [WEAK]
                EXPORT  TIM3_IRQHandler                   [WEAK]
                EXPORT  TIM4_IRQHandler                   [WEAK]
                EXPORT  I2C1_EV_IRQHandler                [WEAK]
                EXPORT  I2C1_ER_IRQHandler                [WEAK]
                EXPORT  I2C2_EV_IRQHandler                [WEAK]
                EXPORT  I2C2_ER_IRQHandler                [WEAK]
                EXPORT  SPI1_IRQHandler                   [WEAK]
                EXPORT  SPI2_IRQHandler                   [WEAK]
                EXPORT  USART1_IRQHandler                 [WEAK]
                EXPORT  USART2_IRQHandler                 [WEAK]
                EXPORT  USART3_IRQHandler                 [WEAK]
                EXPORT  EXTI15_10_IRQHandler              [WEAK]
                EXPORT  RTC_Alarm_IRQHandler              [WEAK]
                EXPORT  OTG_FS_WKUP_IRQHandler            [WEAK]
                EXPORT  TIM8_BRK_TIM12_IRQHandler         [WEAK]
                EXPORT  TIM8_UP_TIM13_IRQHandler          [WEAK]
                EXPORT  TIM8_TRG_COM_TIM14_IRQHandler     [WEAK]
                EXPORT  TIM8_CC_IRQHandler                [WEAK]
                EXPORT  DMA1_Stream7_IRQHandler           [WEAK]
                EXPORT  FMC_IRQHandler                    [WEAK]
                EXPORT  SDIO_IRQHandler                   [WEAK]
                EXPORT  TIM5_IRQHandler                   [WEAK]
                EXPORT  SPI3_IRQHandler                   [WEAK]
                EXPORT  UART4_IRQHandler                  [WEAK]
                EXPORT  UART5_IRQHandler                  [WEAK]
                EXPORT  TIM6_DAC_IRQHandler               [WEAK]
                EXPORT  TIM7_IRQHandler                   [WEAK]
                EXPORT  DMA2_Stream0_IRQHandler           [WEAK]
                EXPORT  DMA2_Stream1_IRQHandler           [WEAK]
                EXPORT  DMA2_Stream2_IRQHandler           [WEAK]
                EXPORT  DMA2_Stream3_IRQHandler           [WEAK]
                EXPORT  DMA2_Stream4_IRQHandler           [WEAK]
                EXPORT  ETH_IRQHandler                    [WEAK]
                EXPORT  ETH_WKUP_IRQHandler               [WEAK]
                EXPORT  CAN2_TX_IRQHandler                [WEAK]
                EXPORT  CAN2_RX0_IRQHandler               [WEAK]
                EXPORT  CAN2_RX1_IRQHandler               [WEAK]
                EXPORT  CAN2_SCE_IRQHandler               [WEAK]
                EXPORT  OTG_FS_IRQHandler                 [WEAK]
                EXPORT  DMA2_Stream5_IRQHandler           [WEAK]
                EXPORT  DMA2_Stream6_IRQHandler           [WEAK]
                EXPORT  DMA2_Stream7_IRQHandler           [WEAK]
                EXPORT  USART6_IRQHandler                 [WEAK]
                EXPORT  I2C3_EV_IRQHandler                [WEAK]
                EXPORT  I2C3_ER_IRQHandler                [WEAK]
                EXPORT  OTG_HS_EP1_OUT_IRQHandler         [WEAK]
                EXPORT  OTG_HS_EP1_IN_IRQHandler          [WEAK]
                EXPORT  OTG_HS_WKUP_IRQHandler            [WEAK]
                EXPORT  OTG_HS_IRQHandler                 [WEAK]
                EXPORT  DCMI_IRQHandler                   [WEAK]
                EXPORT  HASH_RNG_IRQHandler               [WEAK]
                EXPORT  FPU_IRQHandler                    [WEAK]

WWDG_IRQHandler
PVD_IRQHandler
TAMP_STAMP_IRQHandler
RTC_WKUP_IRQHandler
FLASH_IRQHandler
RCC_IRQHandler
EXTI0_IRQHandler
EXTI1_IRQHandler
EXTI2_IRQHandler
EXTI3_IRQHandler
EXTI4_IRQHandler
DMA1_Stream0_IRQHandler
DMA1_Stream1_IRQHandler
DMA1_Stream2_IRQHandler
DMA1_Stream3_IRQHandler
DMA1_Stream4_IRQHandler
DMA1_Stream5_IRQHandler
DMA1_Stream6_IRQHandler
ADC_IRQHandler
CAN1_TX_IRQHandler
CAN1_RX0_IRQHandler
CAN1_RX1_IRQHandler
CAN1_SCE_IRQHandler
EXTI9_5_IRQHandler
TIM1_BRK_TIM9_IRQHandler
TIM1_UP_TIM10_IRQHandler
TIM1_TRG_COM_TIM11_IRQHandler  
TIM1_CC_IRQHandler
TIM2_IRQHandler
TIM3_IRQHandler
TIM4_IRQHandler
I2C1_EV_IRQHandler
I2C1_ER_IRQHandler
I2C2_EV_IRQHandler
I2C2_ER_IRQHandler
SPI1_IRQHandler
SPI2_IRQHandler
USART1_IRQHandler
USART2_IRQHandler
USART3_IRQHandler
EXTI15_10_IRQHandler
RTC_Alarm_IRQHandler
OTG_FS_WKUP_IRQHandler
TIM8_BRK_TIM12_IRQHandler
TIM8_UP_TIM13_IRQHandler
TIM8_TRG_COM_TIM14_IRQHandler  
TIM8_CC_IRQHandler
DMA1_Stream7_IRQHandler
FMC_IRQHandler
SDIO_IRQHandler
TIM5_IRQHandler
SPI3_IRQHandler
UART4_IRQHandler
UART5_IRQHandler
TIM6_DAC_IRQHandler
TIM7_IRQHandler
DMA2_Stream0_IRQHandler
DMA2_Stream1_IRQHandler
DMA2_Stream2_IRQHandler
DMA2_Stream3_IRQHandler
DMA2_Stream4_IRQHandler
ETH_IRQHandler
ETH_WKUP_IRQHandler
CAN2_TX_IRQHandler
CAN2_RX0_IRQHandler
CAN2_RX1_IRQHandler
CAN2_SCE_IRQHandler
OTG_FS_IRQHandler
DMA2_Stream5_IRQHandler
DMA2_Stream6_IRQHandler
DMA2_Stream7_IRQHandler
USART6_IRQHandler
I2C3_EV_IRQHandler
I2C3_ER_IRQHandler
OTG_HS_EP1_OUT_IRQHandler
OTG_HS_EP1_IN_IRQHandler
OTG_HS_WKUP_IRQHandler
OTG_HS_IRQHandler
DCMI_IRQHandler
HASH_RNG_IRQHandler
FPU_IRQHandler  

                B       .

                ENDP

                ALIGN

;*******************************************************************************
; User Stack and Heap initialization
;*******************************************************************************
                 IF      :DEF:__MICROLIB

                 EXPORT  __initial_sp
                 EXPORT  __heap_base
                 EXPORT  __heap_limit

                 ELSE

                 IMPORT  __use_two_region_memory
                 EXPORT  __user_initial_stackheap

__user_initial_stackheap

                 LDR     R0, =  Heap_Mem
                 LDR     R1, =(Stack_Mem + Stack_Size)
                 LDR     R2, = (Heap_Mem +  Heap_Size)
                 LDR     R3, = Stack_Mem
                 BX      LR

                 ALIGN

                 ENDIF

                 END

;************************ (C) COPYRIGHT STMicroelectronics *****END OF FILE*****
```

这个文件是ST的固件包直接提供的，不过为了各位初学者理解他的作用和工程结构，我对其做了修改，以适用于我们此次所建立的最简单STM32工程，大家先不用管这里面的内容，只需要复制到你的工程，后面会对这个文件进行分析

之后，同样的方式，我们再建立一个main.c文件，这个文件，需要根据你具体使用的硬件资源做修改，我这里的内容如下：

```C
int main(void)
{
    volatile unsigned int *rcc = (unsigned int *)0x40023830U;
    volatile unsigned int *mode = (unsigned int *)0x40021400U;
    volatile unsigned int *bssr = (unsigned int *)0x40021418U;
    *rcc = (*rcc | (1 << 5));
    *mode = *mode & 0xFFF3FFFF | 0x00040000;
    while (1)
    {
        *bssr |= 1 << 9;
        for (int i = 0; i < 1000000; i++) ;
        *bssr |= 1 << 25;
        for (int i = 0; i < 1000000; i++) ;
    }
    return 0;
}
```

这就是一个最简单的STM32工程了

看到这里，你可能已经懵了，这都是啥啊，这里面奇奇怪怪的数字都是怎么来的，不要着急，下面我们慢慢分析

## 程序启动

首先我们需要知道的是，STM32的程序是怎么启动的，一般来说，芯片上电后，会从芯片上固定的地址开始运行程序，对于STM32，一般是内部Flash的起始地址，也就是0x08000000这个地址，当然，程序也可以从RAM启动，这取决于两个BOOT引脚的配置。程序的第一个字是堆栈地址，从第二个字开始，便是中断向量表，中断向量表第一个数据，也就是程序的第二个字(从内部Flash启动时，这个字地址为0x08000004)存放的是复位中断的服务函数地址，程序便是从这里开始执行的

也就是说，芯片一上电，就会先执行此一复位操作，一切我们编写的程序逻辑，都是从复位中断的服务函数中开始的

所以，我们再看回start.s文件的内容，在68行左右开始，如下：

```asm
; Vector Table Mapped to Address 0 at Reset
                AREA    RESET, DATA, READONLY
                EXPORT  __Vectors
                EXPORT  __Vectors_End
                EXPORT  __Vectors_Size

__Vectors       DCD     __initial_sp               ; Top of Stack
                DCD     Reset_Handler              ; Reset Handler
                DCD     NMI_Handler                ; NMI Handler
```

首先用AREA指令声明了一块区域，这块区域标记为RESET，也就是存放至程序开始的地方，对应到内部Flash 0x08000000的地址，后面调EXPORT是声明符号，紧接着，使用DCD指令定义了`__initial_sp`，也就是堆栈的栈顶地址，意思就是把`__initial_sp`的值，放置在这个地址(0x08000000)上，同样的，下一条DCD指令，把`Reset_Handler`的值，放置在了0x08000004这个地址上，前面说到，程序会从这里执行，而`Reset_Handler`声明的是一段汇编代码，如下：

```asm
; Reset handler
Reset_Handler    PROC
                 EXPORT  Reset_Handler             [WEAK]
;        IMPORT  SystemInit
;        IMPORT  __main
        IMPORT   main

;                 LDR     R0, =SystemInit
;                 BLX     R0
;                 LDR     R0, =__main
;                 BX      R0
                 LDR     R0, =main
                 BX      R0
                 ENDP
```

以`;`开头的为注释，我们先忽略，再看这一段代码，首先，使用IMPORT声明了`main`这个符号，我们知道，`main`即为我们再main.c这个文件中声明的主函数，之后，使用LDR指令，将`main`函数的地址加载到R0寄存器中，然后使用BX指令，跳转到R0寄存器值指向的地址，也就是跳转到main函数执行

到这里，程序就已经进入到了C语言世界了，我们可以再main函数中，实现自己的逻辑，调用其他函数等

## 一切都是寄存器和内存操作

start.s的关键代码解释清楚了，那main.c文件中那一堆莫名其妙的数字是什么意思呢，再具体解释main函数之前，我们首先需要明确一个概念，那就是，在嵌入式软件开发中，一切的操作，最终都是对寄存器和对内存(储存)的操作

什么意思呢，举个例子，比如说你现在想要控制一个GPIO口输入一个高电平，你可以通过调用库的方式，可能一个`HAL_GPIO_WritePin`就行了，但是，你有去了解，这个函数又做了哪些操作吗。事实上，这个函数只做了一个简单的事情，那就是往一个对应的地址，写入了一个对应的数值

对于32位arm cortex M系列的芯片，它所有的外设，RAM，ROM都是对应映射到一个4GB的地址空间的，这一块Cortex M3(M4)权威指南有详细的介绍，也就是说，你操作外设，实际上是通过操作外设对应于4GB地址空间中的某一个地址的内存实现的，所以说，一切都是寄存器和内存操作

举例来说，对于STM32F4系列芯片的GPIOF端口引脚，控制其模式的寄存器地址为`0x40021400U`，那我们想要控制GPIOF的某一个引脚为输出模式，直需要往这个地址对应的数据位写入对应的数据即可

## 点灯

我们再来分析main.c文件，首先，说明三个寄存器

```C
    volatile unsigned int *rcc = (unsigned int *)0x40023830U;
    volatile unsigned int *mode = (unsigned int *)0x40021400U;
    volatile unsigned int *bssr = (unsigned int *)0x40021418U;
```

`rcc`, `mode`, `bssr`分别是时钟源控制，GPIOF引脚模式和GPIOF端口复位/置位寄存器的对应地址，正如前面说到的，我们通过直接往对应地址写数据的方式，控制外设

首先，开启GPIOF的时钟源

```C
*rcc = (*rcc | (1 << 5));
```

然后设置引脚输入模式，我这里LED使用的引脚是GPIOF的9号引脚，对照手册，我们知道，需要设置引脚模式寄存器的第19，20位为二进制01

```C
*mode = *mode & 0xFFF3FFFF | 0x00040000;
```

然后交替控制引脚复位和置位，达到LED闪烁的效果

```C
    while (1)
    {
        *bssr |= 1 << 9;
        for (int i = 0; i < 1000000; i++) ;
        *bssr |= 1 << 25;
        for (int i = 0; i < 1000000; i++) ;
    }
```

## 结语

本篇基本介绍了一个最简的STM32工程，当然，我们实际项目中，是不会使用这种方式去操作外设，但是，这篇文章的主要目的是让大家了解外设操作的一些细节，下一篇文章，我将会说明，如何根据你所以使用的开发板资源，修改这个工程，让它能在你的板子上跑起来，以及一些重要的，编译器相关的知识
