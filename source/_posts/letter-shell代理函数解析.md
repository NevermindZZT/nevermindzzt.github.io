---
title:      letter-shell代理函数解析
date:       2020-04-17 08:32:45
subtitle:   letter shell实现任意参数的解析
author:     Letter
catalog:    true
categories: letter shell
tags:       嵌入式
            shell
            C
---

## 前言

[letter shell](https://github.com/NevermindZZT/letter-shell)默认支持整形，字符，字符串参数的自动解析，我一直以为，浮点型的参数也是可以支持的，结果前几天发现，浮点型参数只在某些特定情况下可以使用(仅当浮点型参数为函数的最后一个参数时)，为此，我尝试了一种新的方式，从而引出了代理函数和代理参数解析的概念，可以实现任意类型参数的解析

## 原理

如果你需要导出一个命令到shell，但是函数又有shell原生不支持的数据类型，比如说`void test(int a, float b, int c, float d)`，那么要怎么办呢

最简单的，你可能会重新定义一个函数`void testWarpper(int a, int b, int c, int d)`，在这个函数里面对参数进行转换，调用`test`，然后导出`testWarpper`作为命令

这就是所谓代理函数的概念，letter shell的代理函数就是基于此，只不过通过宏，简化了代理函数的定义，代理函数宏定义如下:

```C

/**
 * @brief shell 代理函数名
 */
#define     SHELL_AGENCY_FUNC_NAME(_func)   agency##_func

/**
 * @brief shell代理函数定义
 *
 * @param _func 被代理的函数
 * @param ... 代理参数
 */
#define     SHELL_AGENCY_FUNC(_func, ...) \
            void SHELL_AGENCY_FUNC_NAME(_func)(int p1, int p2, int p3, int p4, int p5, int p6, int p7) \
            { _func(__VA_ARGS__); }
```

定义了代理函数，我们需要在代理函数里对参数进行处理，我称之为代理参数解析，参考letter shell的代理函数宏定义，shell会将终端输入的参数，解析成shell支持的基本参数数据，按顺序以`p1~p7`的参数传递进来，使用者需要定义代理参数解析器，可以为一个函数，或者宏，或者只是简单的数据处理，通过代理参数解析器，将`p1~p7`中对应的参数，转换成原函数需要的数据类型

letter shell默认实现了浮点参数的代理参数解析器，如下：

```C
/**
 * @brief shell float型参数转换
 */
#define     SHELL_PARAM_FLOAT(x)            (*(float *)(&x))
```

有了代理函数和对应的代理参数解析器，就可以将代理函数导出到命令，从而可以实现任意参数类型的解析，letter shell提供了一个宏，可以一步定义代理函数和导出命令，定义如下：

```C
/**
 * @brief shell 代理命令定义
 *
 * @param _attr 命令属性
 * @param _name 命令名
 * @param _func 命令函数
 * @param _desc 命令描述
 * @param ... 代理参数
 */
#define SHELL_EXPORT_CMD_AGENCY(_attr, _name, _func, _desc, ...) \
        SHELL_AGENCY_FUNC(_func, ##__VA_ARGS__) \
        SHELL_EXPORT_CMD(_attr, _name, SHELL_AGENCY_FUNC_NAME(_func), _desc)
```

代理函数命令导出宏前4个参数和常规形式的命令导出一致，之后的参数即传递至目标函数的参数，对于shell直接支持的参数类型，直接写对应的`px(x为1~7)`即可，代理参数类型则需要使用代理参数解析器，如`SHELL_PARAM_FLOAT(p2)`

## 使用实例

下面通过几个例子详细说明letter shell中代理函数的使用

### 浮点参数

一个包含多个浮点参数的函数定义如下：

```C
void test(int a, float b, int c, float d)
{
    printf("a = %d, b = %f, c = %d, d = %f \r\n", a, b, c, d);
}
```

使用letter shell默认实现的浮点参数解析器定义代理函数以及导出命令：

```C
SHELL_EXPORT_CMD_AGENCY(SHELL_CMD_TYPE(SHELL_TYPE_CMD_FUNC),
test, test, test float param,
p1, SHELL_PARAM_FLOAT(p2), p3, SHELL_PARAM_FLOAT(p4));
```

参数中，第一个参数和第三个参数时letter shell原生支持的参数类型，不需要代理解析，所以直接写`p1`和`p3`即可，而第二个和第四个参数是浮点型的数据，这里使用letter shell默认实现的浮点参数代理解析器对参数进行代理解析，写入`SHELL_PARAM_FLOAT(p2)`和`SHELL_PARAM_FLOAT(p4))`

在命令行调用结果如下：

```sh
letter:/$ test 12 12.5 854 7895.4
a = 12, b = 12.500000, c = 854, d = 7895.399902
Return: 34, 0x00000022
```

### 结构体参数

一个包含结构体参数的函数定义如下：

```C
typedef struct {
    int a;
    char *b;
} Test;

void test(char *name, Test *test)
{
    printf("name: %s, a: %d, b: %s\r\n", name, test->a, test->b);
}
```

这里我们直接使用C99复合文字的特性，作为代理参数解析器，可以导出命令如下：

```C
SHELL_EXPORT_CMD_AGENCY(SHELL_CMD_TYPE(SHELL_TYPE_CMD_FUNC),
test, test, test struct,
(char *)p1, &(Test){p2, (char *)p3});
```

导出的命令需要三个参数，第一个参数传递给`test`函数的形参`name`，第二个参数和第三个参数作为结构体`Test`的成员，通过复合文字特性的代理参数解析，生成结构体传递给`test`函数的形参`test`

在命令行调用结果如下：

```sh
letter:/$ test hello 123 test
name: hello, a: 123, b: test
Return: 30, 0x0000001e
```

对于稍微复杂的结构体参数，还可以结合[cson](https://github.com/NevermindZZT/cson)，以字符串的形式将json数据传入，然后使用cson将字符串代理解析成结构体参数

## 结语

原理上来说，letter shell的代理函数并不复杂，本质上，就只是重新定义了一个函数，实现参数的转化，letter shell结合宏定义，简化了真个代理函数的定义流程，将函数定义和命令导出结合到一条宏，并和原先的宏定义保持类似的结构

总而言之，借助代理函数和代理参数解析，你可以使用letter shell实现任意参数的传递，将任意形式的函数作为shell命令导出
