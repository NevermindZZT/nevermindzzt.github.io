---
layout:     post
title:      C语言复合文字的使用
subtitle:
date:       2019-04-03
author:     Letter
header-img:
catalog:    true
tags: 
    - C
---

# 前言

在一些高级语言中，有一种定义函数的方法，如Python中，可以定义`def func(arg1, arg2=2)`，这样可以定义函数形参的默认值，对于一些参数多，但是大多数参数基本不会变动的函数来说极其有用，那么，C语言是否也可以实现这样的功能呢

<!-- more -->

在C99的标准中，增加了一个新的特性，复合文字(composite literal)，很奇怪，这个特性是如此的好用，可是在我的身边几乎没有人知道这个东西

# 复合文字应用

## 数组操作

假设一个场景，有一个函数`void func(char *p, short size)`，你希望传递一个有数据的数组进去，那么可以这样

```C
char array[4] = {1, 2, 3, 4};
func(array, 4);
```

你需要先创建数据，然后给数组里面的数据复制，最后将数据传递给函数，但是在应用了复合文字之后，你可以这样做

```C
func((char []){1, 2, 3, 4}, 4);
```

如何，数组像是一个基本数据类型的常量一样，被直接传递给了函数，当然，你也可以用这个机制，直接将一个数组作为函数的返回值返回

## 结构体操作

使用复合文字特性操作数组，虽然看起来确实简化了一些操作，但是实际一看，好像对程序并没有多大提升，但是，当复合文字碰到结构体的时候，真正的作用就体现出来了

正如我在前言里说到的，使用复合文字结合结构体，我们可以实现参数默认值功能，特别是对于一些初始化操作，可以大大简化函数的使用。

以嵌入式中常用的串口为例，我们一般初始化串口时，要设置波特率，字长，起始位，校验，流控等，因而，我们可以要写一个参数表极长的函数，或者定义一个结构体，一个一个设置结构体成员的值，然而，一般情况下，上面说到的这些参数都是有一个默认值的，所以如果能定义一个函数，只需要给需要设置的参数，剩下的为默认值就好了，这就到复合文字大显身手的时候了

定义结构体(STM32 HAL库)

```C
typedef struct
{
    uint32_t BaudRate;
    uint32_t WordLength;
    uint32_t StopBits;
    uint32_t Parity;
    uint32_t Mode;
    uint32_t HwFlowCtl;
    uint32_t OverSampling;
}UART_InitTypeDef;
```

定义初始化函数

```C
void _SerialInit(char uart, UART_InitTypeDef init)
{
    /** 初始化操作 */
}
```

一般来说，利用复合文字，我们可以使用这种形式调用函数

```C
_SerialInit(1, (UART_InitTypeDef){BaudRate=115200, .WordLength=UART_WORDLENGTH_8B, /** 其他参数 */});
```

好像还是没有事项参数默认值啊，不急，我们只差一个宏定义了

```C
#define serialInit(uart, ...) \
        _SerialInit( \
            uart, \
            (UART_InitTypeDef){ \
                .BaudRate = 115200, \
                .WordLength = UART_WORDLENGTH_8B, \
                .StopBits = UART_STOPBITS_1, \
                .Parity = UART_PARITY_NONE, \
                .Mode = UART_MODE_TX_RX, \
                .HwFlowCtl = UART_HWCONTROL_NONE, \
                .OverSampling = UART_OVERSAMPLING_16, \
                ##__VA_ARGS__, \
            })
```

好了，我们在这个宏中设置所有参数的默认值，我们调用的时候，就可以只设置我们需要设置的参数了

```C
serialInit(1);                              /** 所有参数使用默认值 */
serialInit(1, .BaudRate=19200);             /** 重设置波特率 */
serialInit(1, .BaueRate=19200, .StopBits = UART_STOPBITS_2) /** 重设波特率和停止位 */
```

# 总结

C语言其实也一直在成长，每一次更新都会引入一些新的特性，这些特性无疑是为了更好的使用这门语言，所以即使了解一些新的特性，可以让我们写出更高效的代码

PS：突然萌生想法，可不可以利用这个特性实现类似C++, JAVA等语言中的函数重载