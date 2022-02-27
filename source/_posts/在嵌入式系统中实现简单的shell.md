---
layout:     post
title:      在嵌入式系统中实现简单的shell
subtitle:   在STM32上使用shell
date:       2018-07-15
author:     Letter
catalog:    true
categories: letter shell
tags:
    - 嵌入式
    - STM32
---

对于嵌入式系统而言，特别对于没有使用操作系统，裸机运行程序的嵌入式系统，如何高效便捷的进行系统调试往往是一个比较令人头疼的问题。不久前，我接触到一个国产嵌入式操作系统，Thread RTOS，其中，该系统集成的finsh shell工具让我有种眼前一亮的感觉，它将shell工具引入到嵌入式系统中，极大的方便了系统的调试。

然而，finsh shell运行在操作系统之上，体积也比较大，对于某些小型嵌入式设备，基本是与其无缘了，既然如此，我们为何不自己编写一个shell呢。

<!-- more -->

我们首先对shell的运行原理进行分析，通过在命令行输入命令，shell对命令进行解析，然后执行相应的操作，更通俗的，就是使用输入的字符串，匹配到对应的函数，然后执行。那么，我们需要建立一个命令-函数的一一对应的关系，定义结构体。

```C++
typedef struct
{
    uint8_t *name;                                              //shell命令名称
    shellFunction function;                                     //shell命令函数
    uint8_t *desc;                                              //shell命令描述
}SHELL_CommandTypeDef;                                          //shell命令定义
```

其中，shellFunction为函数指针类型，定义为

```C++
typedef void (*shellFunction)();
```

有了定义之后，我们建立一个表，将所有的命令以及对应的函数进行声明

```C++
/**
* shell 命令表，使用 {command, function, description} 的格式添加命令
* 其中
* command   为命令，字符串格式，长度不能超过 SHELL_PARAMETER_MAX_LENGTH
*           若不使用带参命令，则长度不超过SHELL_COMMAND_MAX_LENGTH
* function  为该命令调用的函数，支持(void *)(void)类型的无参函数以及与带参主函数
*           类似的(void *)(uint32_t argc, uint8_t *argv[])类型的带参函数，其中，
*           argc 为参数个数，argv 为参数，参数皆为字符串格式，需自行进行数据转换
* description 为对命令的描述，字符串格式
*/
const SHELL_CommandTypeDef shellCommandList[] =
{
    /*command               function                description*/
    {(uint8_t *)"letter",   shellLetter,            (uint8_t *)"letter shell"},
    {(uint8_t *)"reboot",   shellReboot,            (uint8_t *)"reboot system"},
    {(uint8_t *)"help",     shellShowCommandList,   (uint8_t *)"show command list"},
    {(uint8_t *)"clear",    shellClear,             (uint8_t *)"clear command line"},
    {(uint8_t *)"iap",      iapMain,                (uint8_t *)"iap"},
    {(uint8_t *)"tftp",     iapTftp,                (uint8_t *)"start TFTP server"},
    {(uint8_t *)"userApp",  iapJumpToApplication,   (uint8_t *)"run user application"},
    {(uint8_t *)"erase",    (void (*)())iapErase,   (uint8_t *)"erase user application"},

#if SHELL_USE_PARAMETER == 1    /*带参函数命令*/
    {(uint8_t *)"paraTest", (void (*)())shellParaTest, (uint8_t *)"test parameter"},
#endif

};
```

然后，我们只需要获得输入的命令，并将其和命令表中的命令进行匹配，然后执行相应的函数即可，完整代码如下：

```C++
/*******************************************************************************
*@function  shellHandler
*@brief     shell处理函数
*@param     receiveData     接收到的数据
*@retval    None
*@author    Letter
*@note      此函数被shellMain函数调用，若使用shellMain阻塞式运行shell，直接调用
*           shellMain函数即可，但不建议这样做，建议在无操作系统情况下，在shell
*           输入触发的中断中调用此函数（通常为串口中断），此时无需调用shellMain，
*           shell也为非阻塞式，操作系统情况下，通常将此函数交给shell输入设备的
*           任务处理
*******************************************************************************/
void shellHandler(uint8_t receiveData)
{
    static uint8_t runFlag;
    static CONTROL_Status controlFlag = CONTROL_FREE;

    switch (receiveData)
    {
        case '\r':
        case '\n':
            if (shellCommandFlag >= SHELL_COMMAND_MAX_LENGTH - 1)
            {
                shellDisplay("\r\nError: Command is too long\r\n");
                shellCommandBuff[shellCommandFlag] = 0;
                shellCommandFlag = 0;
                shellDisplay(SHELL_COMMAND);
                break;
            }

            if (shellCommandFlag == 0)
            {
                shellDisplay(SHELL_COMMAND);
                break;
            }
            else
            {
                shellCommandBuff[shellCommandFlag++] = 0;
#if SHELL_USE_PARAMETER == 1
                commandCount = 0;

                uint8_t j = 0;
                for (int8_t i = 0; i < shellCommandFlag; i++)
                {
                    if (shellCommandBuff[i] != ' ' &&
                        shellCommandBuff[i] != '\t' &&
                        shellCommandBuff[i] != 0)
                    {
                        commandPara[commandCount][j++] = shellCommandBuff[i];
                    }
                    else
                    {
                        if (j != 0)
                        {
                            commandPara[commandCount][j] = 0;
                            commandCount ++;
                            j = 0;
                        }
                    }
                }
                shellCommandFlag = 0;

                if (commandCount == 0)                      //是否为无效指令
                {
                    shellDisplay(SHELL_COMMAND);
                    break;
                }

#if SHELL_USE_HISTORY ==1
                shellStringCopy(shellHistoryCommand[shellHistoryFlag++], shellCommandBuff);
                if (++shellHistoryCount > SHELL_HISTORY_MAX_NUMBER)
                {
                    shellHistoryCount = SHELL_HISTORY_MAX_NUMBER;
                }
                if (shellHistoryFlag >= SHELL_HISTORY_MAX_NUMBER)
                {
                    shellHistoryFlag = 0;
                }
                shellHistoryOffset = 0;
#endif

                shellDisplay("\r\n");
                runFlag = 0;

                for (int8_t i = sizeof(shellCommandList) / sizeof(SHELL_CommandTypeDef) - 1;
                     i >=  0; i--)
                {
                    if (strcmp((const char *)commandPara[0],
                        (const char *)shellCommandList[i].name) == 0)
                    {
                        runFlag = 1;
                        shellCommandList[i].function(commandCount, commandPointer);
                        break;
                    }
                }

#else /*SHELL_USE_PARAMETER == 1*/

                shellCommandBuff[shellCommandFlag] = 0;
                shellCommandFlag = 0;
                shellDisplay("\r\n");
                runFlag = 0;
                for (int8_t i = sizeof(shellCommandList) / sizeof(SHELL_CommandTypeDef) - 1;
                     i >=  0; i--)
                {
                    if (strcmp((const char *)shellCommandBuff,
                        (const char *)shellCommandList[i].name) == 0)
                    {
                        runFlag = 1;
                        shellCommandList[i].function();
                        break;
                    }
                }
#endif /*SHELL_USE_PARAMETER == 1*/

                if (runFlag == 0)
                {
                    shellDisplay("Command not found");
                }
            }
            shellDisplay(SHELL_COMMAND);
            break;

        case 0x08:                                          //退格
            if (shellCommandFlag != 0)
            {
                shellCommandFlag--;
                shellBackspace(1);
            }
            break;

        case '\t':                                          //制表符
        #if SHELL_USE_HISTORY == 1
            if (shellHistoryCount != 0)
            {
                shellBackspace(shellCommandFlag);
                shellCommandFlag = shellStringCopy(shellCommandBuff,
                                   shellHistoryCommand[(shellHistoryFlag + SHELL_HISTORY_MAX_NUMBER - 1)
                                                        % SHELL_HISTORY_MAX_NUMBER]);
                shellDisplay(shellCommandBuff);
            }
            else                                            //无历史命令，输入help
            {
                shellBackspace(shellCommandFlag);
                shellCommandFlag = 4;
                shellStringCopy(shellCommandBuff, (uint8_t *)"help");
                shellDisplay(shellCommandBuff);
            }
        #endif
            break;

        case 0x1B:                                          //控制键
            controlFlag = CONTROL_STEP_ONE;
            break;

        default:
            switch ((uint8_t)controlFlag)
            {
                case CONTROL_STEP_TWO:
                    if (receiveData == 0x41)                //方向上键
                    {
                    #if SHELL_USE_HISTORY == 1
                        shellBackspace(shellCommandFlag);
                        if (shellHistoryOffset--
                            <= -((shellHistoryCount > shellHistoryFlag)
                                ? shellHistoryCount : shellHistoryFlag))
                        {
                            shellHistoryOffset
                            = -((shellHistoryCount > shellHistoryFlag)
                                ? shellHistoryCount : shellHistoryFlag);
                        }
                        shellCommandFlag = shellStringCopy(shellCommandBuff,
                            shellHistoryCommand[(shellHistoryFlag + SHELL_HISTORY_MAX_NUMBER
                                                 + shellHistoryOffset) % SHELL_HISTORY_MAX_NUMBER]);
                        shellDisplay(shellCommandBuff);
                    #else
                        //shellDisplay("up\r\n");
                    #endif
                    }
                    else if (receiveData == 0x42)           //方向下键
                    {
                    #if SHELL_USE_HISTORY == 1
                        if (++shellHistoryOffset >= 0)
                        {
                            shellHistoryOffset = -1;
                            break;
                        }
                        shellBackspace(shellCommandFlag);
                        shellCommandFlag = shellStringCopy(shellCommandBuff,
                            shellHistoryCommand[(shellHistoryFlag + SHELL_HISTORY_MAX_NUMBER
                                                 + shellHistoryOffset) % SHELL_HISTORY_MAX_NUMBER]);
                        shellDisplay(shellCommandBuff);
                    #else
                        //shellDisplay("down\r\n");
                    #endif
                    }
                    else if (receiveData == 0x43)           //方向右键
                    {
                        //shellDisplay("right\r\n");
                    }
                    else if (receiveData == 0x44)           //方向左键
                    {
                        //shellDisplay("left\r\n");
                    }
                    else
                    {
                        controlFlag = CONTROL_FREE;
                        goto normal;
                    }
                    break;

                case CONTROL_STEP_ONE:
                    if (receiveData == 0x5B)
                    {
                        controlFlag = CONTROL_STEP_TWO;
                    }
                    else
                    {
                        controlFlag = CONTROL_FREE;
                        goto normal;
                    }
                    break;

                case CONTROL_FREE:                          //正常按键处理
normal:             if (shellCommandFlag < SHELL_COMMAND_MAX_LENGTH - 1)
                    {
                        shellCommandBuff[shellCommandFlag++] = receiveData;
                        shellDisplayByte(receiveData);
                    }
                    else
                    {
                        shellCommandFlag++;
                        shellDisplayByte(receiveData);
                    }
                    break;

            }
            break;
    }
}
```

我们使用串口进行命令的输入和输出，在输入命令并回车之后，程序解析命令，根据空格将输入分开为命令和参数，对命令进行匹配，匹配到命令之后，执行函数。

完整代码之后会同步到github。

end
