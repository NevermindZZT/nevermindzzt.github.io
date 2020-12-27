---
title:      c语言上下文的快速切换
subtitle:   cpost应用
date:       2020-12-27 14:14:23
author:     Letter
catalog:    true
categories: C
tags:
    - 嵌入式
    - STM32
    - C
---

## 前言

我们通常认为，在中断中，不能执行耗时的操作，否则会影响系统的稳定性，尤其对于嵌入式编程。对于带操作系统的程序而言，可以通过操作系统的调度，将中断处理分成两个部分，耗时的操作可以放到线程中去执行，但是对于没有操作系统的情况，又应该如何处理呢

比较常见的，我们可能会定义一些全局变量，作为`flag`，然后在`mainloop`中不停的判断这些`flag`，再在中断中修改这些`flag`，最后在`mainloop`中执行具体的逻辑，但是这样，无疑会增加耦合，增加程序维护成本

## cpost

[cpost](https://github.com/NevermindZZT/cpost)正是应用在这种情况下的一个简单但又十分方便的工具，它可以特别方便的进行上下文的切换，减少模块耦合

`cpost`借鉴的Android的handler机制，通过在`mainloop`中跑一个任务，然后在其他地方，可以是中断，也可以是模块逻辑中，直接抛出需要执行的函数，使其脱离调用处的上下文，运行在`mainloop`中。`cpost`还支持延迟处理，可以指定函数在抛出后多久执行

## 使用

`cpost`的使用十分简单，这里以使用在嵌入式无操作系统中为例，主要用作中断延迟处理的情况

1. 配置系统tick

    配置`cpost.h`中的宏`CPOST_GET_TICK()`，配置成获取系统tick，以stm32 hal为例

    ```c
    #define     CPOST_GET_TICK()            HAL_GetTick()
    ```

2. 配置处理进程

    在`mainloop`调用`cpostProcess`函数

    ```c
    int main(void)
    {
        ...
        while (1)
        {
            cpostProcess();
        }
        return 0;
    }
    ```

3. 抛出任务

    在中断等需要进行上下文切换的地方调用`cpsot`接口，使其在`mainloop`中运行

    ```c
    cpost(intHandler);
    ```

## 原理解析

`cpost`的原理其实很简单，其代码量也十分少，总共加起来就只有几十行代码，`cpost`维护了一个而全局的数组

```c
CpostHandler cposhHandlers[CPOST_MAX_HANDLER_SIZE] = {0};
```

其中，数组的每一个元素表示包含了需要执行的函数和参数，当调用`cpost`接口时，被post的函数和参数会被保存在这个数组中，然后`mainloop`中运行的`cpostProcess`函数会遍历这个数组，当满足条件时，执行对应的函数，从而达到上下文切换的目的

```c
void cpostProcess(void)
{
    for (size_t i = 0; i < CPOST_MAX_HANDLER_SIZE; i++)
    {
        if (cposhHandlers[i].handler)
        {
            if (cposhHandlers[i].time == 0 || CPOST_GET_TICK() >= cposhHandlers[i].time)
            {
                cposhHandlers[i].handler(cposhHandlers[i].param);
                cposhHandlers[i].handler = NULL;
            }
        }
    }
}
```

其实，cpost的方式，和一开始提到的使用全局的flag进行上下文切换的方法很像，只不过，cpost通过一个数组的维护和直接post函数的方式，省去了维护flag的成本，也不需要将需要执行的函数耦合到mianloop中，从而变得简单易用
