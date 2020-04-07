---
layout:     post
title:      在C语言上优雅地使用json
subtitle:   CSON原理解析
date:       2019-08-29
author:     Letter
header-img:
catalog:    true
tags: 
    - 嵌入式
    - STM32
    - C
    - json
---

## 前言

json是目前最为流行的文本数据传输格式，特别是在网络通信上广泛应用，随着物联网的兴起，在嵌入式设备上，也需要开始使用json进行数据传输，那么，如何快速简洁地用C语言进行json的序列化和反序列化呢

当前，应用最广泛的C语言json解析库当属[cJSON](https://github.com/kbranigan/cJSON)，但是，使用cJSON读json进行序列化和反序列化，需要根据key一个一个进行处理，会导致代码冗余，逻辑性不强，哪有没有更好的方法呢

## 思路

在Android平台，一般会使用[gson](https://github.com/google/gson)等工具解析json，这些工具将json直接映射成对象，在C语言上使用对象的概念，我们需要借助结构体，然而，最大的问题在于，C语言没有高级语言具有的反射机制，直接从json映射到结构体对象几乎是不可能的

怎么解决呢，既然C语言没有反射机制，那么我们可以自己定义一套类似于反射的机制，这里我将其称之为结构体数据模型，在数据模型中，我们需要准确地描述结构体的特征，包括结构体各成员的名称，类型，在结构体中的偏移，有了这些，我们可以在解析josn的时候，将解析得到的数据直接写入到对应的内存里面去，或者是在序列化的时候，直接从对应的内存中读取数据，进行处理

## 实现

[CSON](https://github.com/NevermindZZT/cson)正是采用上面说到的思路，使用数据模型对结构体进行描述，然后基于cJSON，根据数据模型进行解析，将解析得到的数据直接写入到对应的内存区域，从而实现从json到结构体对象的映射

CSON最基本的数据模型定义如下：

```C
typedef struct cson_model
{
    CsonType type;                      /**< 数据类型 */
    char *key;                          /**< 元素键值 */
    short offset;                       /**< 元素偏移 */
} CsonModel;
```

通过`type`描述结构体成员的数据类型，`key`描述该成员在json中对应的字段，`offset`描述该结构体成员在结构体中的偏移，CSON在解析json的时候，根据`type`调用相应的cJSON API并传递`key`作为参数，得到解析出的数据，然后根据`offset`将数据写入到对应的内存空间

比如说这样一个结构体：

```C
struct project
{
    int id;
    char *name;
}
```

该结构体包含两个成员，对于成员`id`，我们使用数据模型对其进行描述`{.type=CSON_TYPE_CHAR, key="id", offset=0}`，对于结构体的每个成员，都进行数据模型的定义，就可以得到一个完整的结构体数据模型，CSON会根据这个模型，进行解析

因为是通过直接写内存的方式，所以在写不同类型的量到内存中时，会多次用到强制转型，导致CSON中赋值的代码都类似于`*(int *)((int)obj + model[i].offset) = (int)csonDecodeNumber(json, model[i].key);`

当然，上面说到的数据模型，只适用于基本数据类型的数据，对于子结构体，链表，数组等，需要对数据模型的定义进行扩充，有兴趣的朋友可以直接阅读CSON源码

## CSON使用实例

### 声明结构体

```C
/** 项目结构体 */
struct project
{
    int id;
    char *name;
};

/** 仓库结构体 */
struct hub
{
    int id;
    char *user;
    struct project *cson;
};
```

### 定义数据模型

对每一个需要使用cson的结构体，都需要定义相对应的数据模型

```C
/** 项目结构体数据模型 */
CsonModel projectModel[] =
{
    CSON_MODEL_OBJ(struct project),
    CSON_MODEL_INT(struct project, id),
    CSON_MODEL_STRING(struct project, name),
};

/** 仓库结构体数据模型 */
CsonModel hubModel[] =
{
    CSON_MODEL_OBJ(struct hub),
    CSON_MODEL_INT(struct hub, id),
    CSON_MODEL_STRING(struct hub, user),
    CSON_MODEL_STRUCT(struct hub, cson, projectModel, sizeof(projectModel)/sizeof(CsonModel))
};
```

### 使用CSON解析

只需要定义好数据模型，就可以使用CSON读json进行序列化和反序列化

```C
void csonDemo(void)
{
    char *jsonDemo = "{\"id\": 1, \"user\": \"Letter\", \"cson\": {\"id\": 2, \"name\": \"cson\"}}";

    /** 解析json */
    struct hub *pHub = csonDecode(jsonDemo, hubModel, sizeof(hubModel)/sizeof(CsonModel));
    printf("hub: id: %d, user: %s, project id: %d, project name: %s\r\n",
        pHub->id, pHub->user, pHub->cson->id, pHub->cson->name);

    /** 序列化对象 */
    char *formatJson = csonEncodeFormatted(pHub, hubModel, sizeof(hubModel)/sizeof(CsonModel));
    printf("format json: %s\r\n", formatJson);

    /** 释放结构体对象 */
    csonFree(pHub, hubModel, sizeof(hubModel)/sizeof(CsonModel));

    /** 释放序列化生成的json字符串 */
    csonFreeJson(formatJson);
}
```

运行结果：

```plain
hub: id: 1, user: Letter, project id: 2, project name: cson
format json: {
        "id":   1,
        "user": "Letter",
        "cson": {
                "id":   2,
                "name": "cson"
        }
}
```

可以看到，无论是解析json，还是序列化结构体到json，在使用CSON的情况下，都只需要一行代码就可以解决，同样的操作，在使用原生cJSON的情况下，你可能需要多次判断，解析元素

## 项目地址

CSON项目已经发布到Github，[点击查看](https://github.com/NevermindZZT/cson)
