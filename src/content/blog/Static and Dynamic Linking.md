---
title: Static and Dynamic Linking
author: sineagle
pubDatetime: 2024-10-17T17:36:22
modDatetime: 
featured: false
draft: false
tags:
  - Linux
  - debug
description: Static and Dynamic Linking Note
---
## Table of contents

## 概述

在我们进行GCC编译用户程序时，分为四个步骤：预处理（Prepressing)、编译（Compilation)、汇编(Assembly)、链接(Linking)

### 预编译

gcc 加上-E参数只进行预编译，或者使用cpp命令,生成.i文件

```bash
gcc -E hello.c -o hello.i
cpp hello.c > hello.i
```

展开所有的宏定义、处理条件预编译指令、删除注释、添加调试的行号、保留#pramgma编译器指令


### 编译

词法分析、语法分析、语义分析，优化生产相应的汇编代码文件。

```bash
gcc -S hello.i -o hello.o
cc1 hello.c
```

gcc实际上是后台程序封装，根据不同参数要求去调用：cc1、as、ld



### 汇编

通过汇编器将汇编代码转换成机器指令，生成目标文件（Object File）

~~~bash
as hello.s -o hello.o
gcc -c hello.s -o hello.o
~~~


### 链接

分为动态链接和静态链接


## 静态链接

链接的过程包括地址和空间分配（Address and Storage Allocation）、符号决议（Symbol Resolution）和重定位（Relocation）这几步
