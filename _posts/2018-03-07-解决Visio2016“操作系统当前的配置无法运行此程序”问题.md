---
layout:     post
title:      解决Visio“操作系统当前的配置无法运行此程序”问题
subtitle:   office2016（Word等）与Visio共存安装
date:       2018-03-07
author:     Letter
header-img: img/post_Visio_20180307.png
catalog: true
tags:
    - office
    - Software
---


由于要写论文的原因，需要用到流程图，于是想装一个Visio用用。从下载Visio镜像到安装完成，一切都很顺利，但是安装完成之后打开，竟然提示“操作系统的当前配置无法运行此程序”，表示一脸懵逼...

在网上搜索找到很多说法，包括改注册表什么的，都没有成功。后面发现，office2016和Visio的镜像其实是一样的，只不过里面的setup.exe不同，于是替换Visio镜像的setup到解压后的office镜像中，在这里安装，然后就安装成功了。

个人猜测应该是两个镜像之间的微小差别导致的不兼容，有同样问题的可以试试这个方案。
