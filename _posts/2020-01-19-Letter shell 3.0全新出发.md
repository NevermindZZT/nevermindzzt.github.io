---
layout:     post
title:      Letter shell 3.0 全新出发
subtitle:   letter shell 3.0不完全食用指南
date:       2020-01-19
author:     Letter
header-img:
catalog:    true
tags: 
    - 嵌入式
    - STM32
    - C
    - shell
---

## 前言

从我一开始写[letter shell](https://github.com/NevermindZZT/letter-shell)已经一年多了，从1.x版本shell只能做命令解析，到2.x版本加入快捷键等功能，letter shell功能慢慢变多，但是体积也越来越大，似乎违背了我一开始只是想做一个超小型调试工具的想法，因此，我重新实现了letter shell 3.0，作为一个功能更加强大的版本，原先的2.x版本保留原有功能，继续维护。

## 3.0改变了什么

letter shell 3.0保留了2.x版本的所有功能，此次新增加了用户管理，权限管理，加入了对main函数形式和普通C函数命令的同时支持，加强了快捷键功能，此外，后续还会增加文件系统支持等其他功能

## 移植

letter shell 3.0在一直这一块基本保留了2.x的结构，原先使用2.x版本的如果想迁移，只需要简单的修改即可

1. 定义shell

    首先定义shell对象，3.0版本修改了结构体命名，从2.x版本迁移的话需要注意

    新建一个shell_port.c文件，定义shell

    ```C
    Shell shell;
    ```

2. 申请内存

    letter shell需要申请一片内存作为数据缓冲，申请内存大小取决于你希望命令行输入的最大长度和历史命令记录的数量，具体计算方式为`(历史命令最大数量 + 1) * 命令行输入最大长度`

    ```C
    char buffer[512];
    ```

3. 实现字符接收函数

    若使用`shellTask`，需要实现字符接收函数函数原型为`typedef signed char (*shellRead)(char *);`，该函数有一个输出参数，表示接收到的字符，返回值为0表示接收到了数据，-1表示没有接收到数据

    ```C
    signed char shellRead(char *data)
    {
        return serialReceive(&debugSerial, (uint8_t *)data, 1, 0) == 1 ? 0 : -1;
    }
    ```

    若不使用`shellTask`，可不实现读函数

4. 实现写字符发送函数

    需要实现的字符发送函数原型为`typedef void (*shellWrite)(const char);`，该函数有一个输入参数，表示需要发送的字符，无返回值

    ```C
    void shellWrite(const char data)
    {
        serialTransmit(&debugSerial, (uint8_t *)&data, 1, 0xFF);
    }
    ```

5. 初始化shell

    对shell进行初始化

    ```C
    shell.read = shellRead;
    shell.write = shellWrite;
    shellInit(&shell, buffer, 512);
    ```

6. 建立shell任务

    对于使用操作系统的情况，需要建立`shellTask`任务，任务参数为定义的shell对象，任务栈大小可以根据需要分配，请以shell可能运行的函数所需要的最大栈为准，注意，请确认shell_cfg.h文件中的宏`SHELL_TASK_WHILE`打开

    ```C
    OSTaskCreate(shellTask, &shell, ...);
    ```

    对于不使用操作系统的情况，需要在主循环中调用`shellTask`，注意，此时shell_cfg.h文件中的宏`SHELL_TASK_WHILE`应该是关闭状态

    ```C
    while (1)
    {
        shellTask(&shell);
        // 其他任务
    }
    ```

    其他情况，也可以不使用`shellTask`，但是需要在接收到字符时，主动调用`shellHandler`

## 命令声明

letter shell支持main函数命令，普通C函数命令，用户声明，变量声明，快捷键声明，通过宏`SHELL_EXPORT_XXX`导出

### main形式函数声明

main函数形式的函数原型为`int func(int argc, char *argv[])`，函数的所有参数都以字符串数组的形式传入`argv`参数

```C
int func_main(int argc, char *agrv[])
{
    printf("%dparameter(s)\r\n", argc);
    for (char i = 1; i < argc; i++)
    {
        printf("%s\r\n", argv[i]);
    }
}
SHELL_EXPORT_CMD(SHELL_CMD_PERMISSION(0)|SHELL_CMD_TYPE(SHELL_TYPE_CMD_MAIN), func, func_main, this is a main like function);
```

其中，`SHELL_CMD_PERMISSION(0)`表示这个命令的权限为0，即无权限需要，所有用户都可以调用，`SHELL_CMD_TYPE(SHELL_TYPE_CMD_MAIN)`表示这个是一个main函数形式的命令，`func`表示命令名，即在终端调用的名字，`func_main`即函数名，最后一个参数是对这个命令的描述，可通过`help`明林查看

在终端中调用这个命令

```sh
letter:/$ func "hello world"
2 parameter(s)
hello world
```

### 普通C函数命令声明

普通C函数形式的命令是通过letter shell对参数进行解析，自动转换参数，然后执行函数，支持整数，浮点数，字符和字符串参数，支持变量作为参数

```C
int func(int i, char ch, char *str)
{
    printf("input int: %d, char: %c, string: %s\r\n", i, ch, str);
}
SHELL_EXPORT_CMD(SHELL_CMD_PERMISSION(0)|SHELL_CMD_TYPE(SHELL_TYPE_CMD_FUNC), func, func, this is a c like function);
```

其中，`SHELL_CMD_TYPE(SHELL_TYPE_CMD_FUNC)`表示这是一个普通C函数形式的命令

在终端中调用这个命令

```sh
letter:/$ func 666 'A' "hello world"
input int: 666, char: A, string: hello world
```

### 变量声明

letter shell支持支持定义整数，浮点数，指针形式的变量，定义的变量可以通过`setVar`命令进行修改，可以以`${var}`的形式作为参数传进命令中

```C
int testVar = 256;
SHELL_CMD_VAR(SHELL_CMD_PERMISSION(0)|SHELL_CMD_TYPE(SHELL_TYPE_VAR_INT), testVar, &testVar, var for test);
```

其中，`SHELL_CMD_TYPE(SHELL_TYPE_VAR_INT)`表示这是一个`int`类型的变量，变量不是指针类型时，需要对变量取引用`&testVar`，相应的，支持的变量类型和类型定义宏如下：

| 类型  | 宏                   |
| ----- | -------------------- |
| char  | SHELL_TYPE_VAR_CHAR  |
| short | SHELL_TYPE_VAR_SHORT |
| int   | SHELL_TYPE_VAR_INT   |
| float | SHELL_TYPE_VAR_FLOAT |
| 常量  | SHELL_TYPE_VAL       |

在终端中查看变量

```sh
letter:/$ testVar
testVar = 256, 0x00000100
```

修改变量

```sh
letter:/$ setVar testVar 65536
testVar = 65536, 0x00010000
```

变量作为命令参数

```sh
letter:/$ func &testVar 'A' "hello world"
input int: 256, char: A, string: hello world
```

### 用户声明

letter shell 3.0版本新加入了用户管理，支持像定义命令一样定义用户

```C
SHELL_EXPORT_USER(SHELL_CMD_PERMISSION(0xFF), root, admin, root user);
```

其中`SHELL_CMD_PERMISSION(0xFF)`表示这个用户的权限，同命令的权限对应，当命令权限为0，或者用户权限同命令权限按位与不为0时，表明这个用户拥有该命令的权限，可以查看，执行，`root`表示用户名称，`admin`表示用户密码，留空时表示这个用户不需要设置密码，最后一个参数时对用户的描述

在终端切换用户

```C
letter:/$ root

Please input password:admin

...
```

对于设置了密码的用户，也可以通过执行`{user} {password}`直接切换用户并校验密码

### 按键声明

letter shell 3.0增强了2.x版本的快捷键功能，最多支持四个字节键的快捷键

```C
SHELL_EXPORT_KEY(SHELL_CMD_PERMISSION(0)|SHELL_CMD_ENABLE_UNCHECKED,
0x1B5B337E, shellDelete, delete);
```

其中，`0x1B5B337E`表示这个按键的键值，即按下按键之后，终端会依次发送0x1B, 0x5B, 0x33, 0x7E四个字节，如果按键键值不足四个字节，以大端模式表示，低字节补0，比如tab键，键值为`0x0B000000`
