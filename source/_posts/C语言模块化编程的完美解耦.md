---
title:      C语言模块化编程的完美解耦
subtitle:   cevent应用
date:       2020-12-20 13:57:02
author:     Letter
catalog:    true
categories: C
tags:
    - 嵌入式
    - STM32
    - C
    - shell
---

## 前言

对于模块化编程来说，如何实现各模块间的解耦一直是一个比较令人头疼的问题，特别是对于嵌入式编程，由于控制逻辑复杂，并且对程序体积有控制，经常容易写出各独立模块之间相互调用的问题。由此，[cpost](https://github.com/NevermindZZT/cpost)中的`cevent组件`，通过模仿Android系统中的广播机制，提供了一种非常简单的模块间解耦实现

## 原理

`cevent`借鉴的是Android系统的广播机制，一方面，各模块在工作的时候，都会有多个具体的事件点，在高耦合的编程中，可能会在这些地方调用其他模块的功能，比如说，在通信模块接收到指令的时候，需要闪烁一下指示灯

使用`cevent`，我们可以在这些地方抛出一个事件，当前模块不需要关心在这各地方需要执行哪些其他模块的逻辑，由其他模块，或者用户定义一个事件监听，当具体的事件发生时，执行相应的动作

## 使用

`cevent`使用注册的方式监听事件，会依赖于编译环境，目前支持keil，iar，和gcc，对于gcc，需要修改链接文件(.ld)，在只读数据区添加：

```ld
_cevent_start = .;
KEEP (*(cEvent))
_cevent_end = .;
```

1. 初始化cevent

    系统初始化时，调用`ceventInit`

    ```c
    ceventInit();
    ```

2. 注册cevent事件监听

    在c文件中，调用`CEVENT_EXPORT`导出事件监听

    ```c
    CEVENT_EXPORT(0, handler, (void *)param);
    ```

3. 发送cevent事件

    在事件发生的地方，调用`ceventPost`抛出事件

    ```c
    ceventPost(0);
    ```

## 使用`cevent`解耦模块初始化

嵌入式编程中，我们习惯会在程序启动的时候，调用各个模块的初始化函数，其实这也是一种耦合，会造成`main`函数中出现很长的初始化代码，借助`cevent`，我们可以对初始化进行优化解耦

1. 定义初始化事件

    定义初始化事件的值，对于初始化，有些模块可能会依赖于其他模块的初始化，会有一个先后顺序要求，所以这里我们可以把初始化分成两个阶段，定义两个事件，当然，如果有更复杂的要求，可以再多分几个阶段，只需要多定义几个事件就行

    ```c
    #define     EVENT_INIT_STAGE1       0
    #define     EVENT_INIT_STAGE2       1
    ```

2. 初始化`cevent`，抛出事件

    在`main`函数中初始化`cevent`，并抛出初始化事件

    ```c
    int main(void)
    {
        ...
        ceventInit();

        ceventPost(EVENT_INIT_STAGE1);
        ceventPost(EVENT_INIT_STAGE2);
        ...
        return 0;
    }
    ```

3. 注册事件监听

    对所有需要初始化的函数注册事件监听，这里我以对`letter-shell`注册事件监听为例，分为两个部分，初始化串口和初始化shell

    在`serial`模块中，将串口初始化注册到初始化第一阶段，`cevent`支持将不大于7个的参数直接传递到注册的监听函数中，下面的注册方式，相当于在`EVENT_INIT_STAGE1`事件发生的地方，也就是`main`函数中对应的位置，调用`serialInit(&debugSerial)`

    ```c
    CEVENT_EXPORT(EVENT_INIT_STAGE1, serialInit, (void *)(&debugSerial));
    ```

    然后再`shell`模块中，将shell初始化函数注册到初始化第二阶段

    ```c
    CEVENT_EXPORT(EVENT_INIT_STAGE1, shellInit);
    ```

## 使用`cevent`解耦`mainloop`

再无操作系统的嵌入式编程中，我们如果同时希望运行多个模块的逻辑，通常是在`mainloop`中循环调用，这种将函数写入`mainloop`的做法，也会增加耦合

```c
int main(void)
{
    ...

    while (1)
    {
        // 写在mainloop中的模块逻辑
        shellTask(&shell);
        LedProcess();
        ...
    }
    return 0;
}
```

通过使用`cevent`，也可以很方便的消除这种耦合

1. 定义mainloop事件

    定义mainloop事件的值

    ```c
    #define     EVENT_MAIN_LOOP         3
    ```

2. 在mainloop中抛出事件

    去掉mainloop中对其他模块的调用，改为排除mainloop事件

    ```c
    int main(void)
    {
        ...

        while (1)
        {
            ceventPost(EVENT_MAIN_LOOP);
        }
        return 0;
    }
    ```

3. 在各模块中注册事件监听

    分别在各个模块中，注册对mainloop事件的监听

    ```c
    CEVENT_EXPORT(EVENT_MAIN_LOOP, shellTask, (void *)(&shell));
    ```

    ```c
    CEVENT_EXPORT(EVENT_MAIN_LOOP, LedProcess);
    ```

## 结语

`cevent`是一个非常小的模块，本身代码及其简单，但是，通过模仿广播机制，让`cevent`可以发挥很强大的功能，通过，还可以结合`cpost`，实现延迟事件等功能
