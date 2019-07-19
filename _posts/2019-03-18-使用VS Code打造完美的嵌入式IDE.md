---
layout:     post
title:      使用VS Code打造完美的嵌入式IDE
subtitle:
date:       2019-03-15
author:     Letter
header-img:
catalog:    true
tags: 
    - 嵌入式
    - STM32
    - VS Code
---

# 前言

在我刚开始接触STM32的时候，使用的keil作为IDE，由于在这之前，我使用过VS, 使用过eclipse, 因而在我使用keil之后，实在难以忍受keil编辑器简陋的功能，可以说是极其糟糕的写代码体验

之后，我尝试过各种IDE，使用eclipse + keil，结果发现eclipse对C语言的支持也是鸡肋，使用emBits + gcc，需要和其他人协同的话就比较麻烦，之后发现了platformIO，也是使用gcc作为编译器，不过只支持HAL库

最后，通过使用VS Code + keil的方式，完美解决了写代码的体验问题，以及工程协作问题

其实网上使用VS Code作为编辑器，keil作为编译器的教程很多，不过基本都是需要在VS Code中编辑，然后在keil中编译，下载，调试，本文就要实现编辑，编译，下载，调试，全部使用VS Code

# 环境

- VS Code
- keil
- python
- C/C++(VS Code 插件)
- Cortex-Debug(VS Code 插件)
- 其他VS Code插件(提升体验)

# 前提

正式写代码之前，首先需要建立好一个工程，这个需要使用keil完成，包括工程配置，文件添加...

# 编辑

在安装好VS Code插件之后，VS Code编写C代码本身体验就已经很好了，但是，因为我们使用的是keil环境，所以需要配置头文件包含，宏定义等

在工程路径的.vscode文件夹下打开c_cpp_properties.json文件，没有自己新建一个，内容配置如下：

```python
{
    "configurations": [
        {
            "name": "STM32",
            "includePath": [
                "C:/Program Files (x86)/keil/ARM/ARMCC/**",
                "${workspaceFolder}/**",
                ""
            ],
            "browse": {
                "limitSymbolsToIncludedHeaders": true,
                "databaseFilename": "${workspaceRoot}/.vscode/.browse.c_cpp.db",
                "path": [
                    "C:/Program Files (x86)/keil/ARM/ARMCC/**",
                    "${workspaceFolder}/**",
                    ""
                ]
            },
            "defines": [
                "_DEBUG",
                "UNICODE",
                "_UNICODE",
                "__CC_ARM"
            ],
            "intelliSenseMode": "msvc-x64"
        }
    ],
    "version": 4
}
```

其中，需要在`includePath`和`path`中添加头文件路径，`${workspaceFolder}/**`是工程路径，不用改动，额外需要添加的是keil的头文件路径

然后在`defines`中添加宏，也就是在keil的Options for Target的C++选项卡中配置的宏

然后就可以体验VS Code强大的代码提示，函数跳转等功能了(甩keil的编辑器一整个时代)

# 编译、烧录

编译和烧录通过VS Code的Task功能实现，通过Task，使用命令行的方式调用keil进行编译和烧录

keil本身就支持命令行调用，具体可以参考keil的手册，这里就不多说了，但是问题在于，使用命令行调用keil，不管是什么操作，他的输出都不会输出到控制台上!!!(要你这命令行支持有何用)

不过好在，keil支持输出到文件中，那我们就只能利用这个做点骚操作了————一边执行命令，一边读取文件内容并打印到控制台，从而就实现了输出在控制台上，我们就能直接在VS Code中看到编译过程了

为此，我编写了一个Python脚本，实现keil的命令行调用并同时读取文件输出到控制台

```python
#!/usr/bin/python
# -*- coding:UTF-8 -*-

import os
import threading
import sys

runing = True

def readfile(logfile):
    with open(logfile, 'w') as f:
        pass
    with open(logfile, 'r') as f:
        while runing:
            line = f.readline(1000)
            if line != '':
                line = line.replace('\\', '/')
                print(line, end = '')

if __name__ == '__main__':
    modulePath = os.path.abspath(os.curdir)
    logfile = modulePath + '/build.log'
    cmd = '\"C:/Program Files (x86)/keil/UV4/UV4.exe\" '
    for i in range(1, len(sys.argv)):
        cmd += sys.argv[i] + ' '
    cmd += '-j0 -o ' + logfile
    thread = threading.Thread(target=readfile, args=(logfile,))
    thread.start()
    code = os.system(cmd)
    runing = False
    thread.join()
    sys.exit(code)
```

此脚本需要结合VS Code的Task运行，通过配置Task，我们还需要匹配输出中的错误信息(编译错误)，实现在keil中，点击错误直接跳转到错误代码处，具体如何配置请参考VS Code的文档，这里给出我的Task

```python
{
    // See https://go.microsoft.com/fwlink/?LinkId=733558
    // for the documentation about the tasks.json format
    "version": "2.0.0",
    "tasks": [
        {
            "label": "build",
            "type": "shell",
            "command": "py",
            "args": [
                "-3",
                "E:/scripts/build.py",
                "-b",
                "${config:uvprojxPath}"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "problemMatcher": [
                {
                    "owner": "c",
                    "fileLocation": ["relative", "${workspaceFolder}/src/MDK-ARM"],
                    "pattern": {
                        "regexp": "^(.*)\\((\\d+)\\):\\s+(warning|error):\\s+(.*):\\s+(.*)$",
                        "file": 1,
                        "line": 2,
                        "severity": 3,
                        "code": 4,
                        "message": 5,
                    }
                }
            ]
        },
        {
            "label": "rebuild",
            "type": "shell",
            "command": "py",
            "args": [
                "-3",
                "E:/scripts/build.py",
                "-r",
                "${config:uvprojxPath}"
            ],
            "group": "build",
            "problemMatcher": [
                {
                    "owner": "c",
                    "fileLocation": ["relative", "${workspaceFolder}/src/MDK-ARM"],
                    "pattern": {
                        "regexp": "^(.*)\\((\\d+)\\):\\s+(warning|error):\\s+(.*):\\s+(.*)$",
                        "file": 1,
                        "line": 2,
                        "severity": 3,
                        "code": 4,
                        "message": 5,
                    }
                }
            ]
        },
        {
            "label": "download",
            "type": "shell",
            "command": "py",
            "args": [
                "-3",
                "E:/scripts/build.py",
                "-f",
                "${config:uvprojxPath}"
            ],
            "group": "test"
        },
        {
            "label": "open in keil",
            "type": "process",
            "command": "${config:uvPath}",
            "args": [
                "${config:uvprojxPath}"
            ],
            "group": "test"
        }
    ]
}
```

对于使用ARM Compiler 6编译的工程，build和rebuild中的problemMatcher应该配置为

```python
"problemMatcher": [
    {
        "owner": "c",
        "fileLocation": ["relative", "${workspaceFolder}/MDK-ARM"],
        "pattern": {
            "regexp": "^(.*)\\((\\d+)\\):\\s+(warning|error):\\s+(.*)$",
            "file": 1,
            "line": 2,
            "severity": 3,
            "message": 4,
        }
    }
]
```

文件中的`config:uvPath`和`config:uvprojxPath`分别为keil的UV4.exe文件路径和工程路径(.uvprojx)，可以直接修改为具体路径，或者在VS Code的setting.json中增加对应的项

至此，我们已经完美实现了在VS Code中编辑，编译，下载了

> 编译输出:
>
> ![img_vscode_keil_complie.png](https://nevermindzzt.github.io/img/img_vscode_keil_complie.png)
>
> 有错误时输出：
>
> ![img_vscode_keil_complie_error.png](https://nevermindzzt.github.io/img/img_vscode_keil_complie_error.png)
>
> 错误匹配：
>
> ![img_vscode_keil_error_match.png](https://nevermindzzt.github.io/img/img_vscode_keil_error_match.png)

# 调试

调试需要使用到Cortex-Debug插件，以及arm gcc工具链，这部分可以参考Cortex-Debug的文档，说的比较详细

首先安装Cortex-Debug插件和arm gcc工具链，然后配置好环境路径，如果使用Jlink调试，需要下载Jlink套件，安转好之后，找到`JLinkGDBServerCL.exe`这个程序，在VS Code的设置中添加`"cortex-debug.JLinkGDBServerPath": "C:/Program Files (x86)/SEGGER/JLink_V630f/JLinkGDBServerCL.exe"`，后面的路径是你自己的路径。

如果使用STLink调试，需要下载stutil工具，在GitHub上搜索即可找到，同样配置好路径即可。

以上步骤弄好之后，可以直接点击VS Code的调试按钮，此时会新建luanch.json文件，这个文件就是VS Code的调试配置文件，可参考我的文件进行配置

```json
{
    // 使用 IntelliSense 了解相关属性。 
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Cortex Debug(JLINK)",
            "cwd": "${workspaceRoot}",
            "executable": "src/MDK-ARM/Objects/stm32_boot.axf",
            "request": "attach",
            "type": "cortex-debug",
            "servertype": "jlink",
            "device": "STM32F407IG",
            "svdFile": "C:/Program Files (x86)/keil/ARM/PACK/Keil/STM32F4xx_DFP/2.11.0/CMSIS/SVD/STM32F40x.svd",
            "interface": "swd",
            "ipAddress": null,
            "serialNumber": null
        },
        {
            "name": "Cortex Debug(ST-LINK)",
            "cwd": "${workspaceRoot}",
            "executable": "src/MDK-ARM/Objects/stm32_boot.axf",
            "request": "attach",
            "type": "cortex-debug",
            "servertype": "stutil",
            "svdFile": "C:/Program Files (x86)/keil/ARM/PACK/Keil/STM32F4xx_DFP/2.11.0/CMSIS/SVD/STM32F40x.svd",
            "device": "STM32F407IG",
            "v1": false
        }
    ]
}
```

注意其中几个需要修改的地方，`executable`修改为你的工程生成的目标文件，也就是工程的`.axf`文件，`svdFile`用于对MCU外设的监控，该文件可以在keil的安装路径中找到，可以参考我的路径去找

配置完成后，再次点击调试按钮即可进行调试

> ![img_vscode_keil_debug.png](https://nevermindzzt.github.io/img/img_vscode_keil_debug.png)

相比keil自己的调试功能，VS Code还支持条件断点，可以设置命中条件，次数等，可以极大的方便调试

# 总结

通过以上的配置，我们基本上，除了建立工程和往工程中添加文件，其他完全不需要打开keil，所以也无妨说一句，再见，智障keil
