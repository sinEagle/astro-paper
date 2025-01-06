---
title: Kbuild编译系统
author: sineagle
pubDatetime: 2024-12-03T14:02:42
modDatetime: 
featured: false
draft: false
tags:
  - Linux
  - debug
description:
---
## Table of contents

## Kbuild工作流程

```bash
make ARCH=xxx xx_defconfig
make menuconfig
make ARCH=xxx CROSS_COMPILE=xxx xImage
```

编译三步骤

- 配置阶段：编译平台、目标、配置文件
- 编译阶段：解析Makefile、建立目标依赖关系、按照依赖关系依次生成各个目标及目标依赖
- 安装阶段：内核镜像/头文件/模块安装

## Kbuild编译系统构成

Kbuild可扩展、可配置的Makefile框架

• Makefile：顶层目录下的Makefile

• .config：内核的配置文件

• arch/$(ARCH)/Makefile：跟平台架构相关的Makefile

• scripts/Makefile.*：通用编译规则

• Kbuild Makefile：分布在各个子目录下

• Kconfig：配置菜单，定义每个config symbol的属性（类型、描述、依赖等）

## Kconfig作用

• 用来生成配置菜单，配置各种config symbol

• 生成对应的配置变量：CONFIG_XXX

• 每个目录下都有一个Kconfig文件

• 各个Kconfig文件通过source命令构建多级菜单

• 解析工具：scripts/kconfig/*conf

config symbol

• bool：y n

• tristate： y m n

• int：数值

• hex：数值

• string：字符串

## config文件

make defconfig -> .config

make menuconfig

.config –> syncconfig –> Makefile

- include/config/auto.conf：用来配置Makefile
- include/generated/autoconf.h：供C程序引用
- include/config/*.h：空头文件，用于构建依赖关系

```makefile
include/generated/autoconf.h：
#define CONFIG_USB_MON 1

include/linux/usb.h:
#if defined(CONFIG_USB_MON) || defined(CONFIG_USB_MON_MODULE)
  struct mon_bus *mon_bus; /* non-null when associated */
  int monitored; /* non-zero when monitored */
#endif
```

## Kbuild Makefile工作流程

• 顶层Makefile：主要用来调用相应规则的Makefile

• .config：用户配置的各种选项

• arch/$(ARCH)/Makefile：跟平台相关的Makefile

• 各个目录下的Makefile：负责编译各个模块

• scripts/Makefile.*：定义各种通用规则

Kbuild Makefile预定义目标和变量

- obj-m：将当前文件编译为独立的模块
- obj-y：将当前文件编译进内核
- xxx-objs：一个模块依赖的多个源文件
- bzImage：
- menuconfig：
- CONFIG_xxx：

  - include include/config/auto.conf
  - include/config/auto.conf.cmd

工作流程如下：

- 根据ARCH变量，首先include arch/$(ARCH)/Makefile
- 读取 .config文件：读取用户的各种配置变量
- 解析预定义目标、目标，构建依赖关系
- 编译各个模块或组件（使用scripts/Makefile.*）

  - 将每个目录下的源文件编译为对应的.o目标文件
  - 将.o目标文件归档为built-in.a
- 将所有对象链接成 vmlinux
- 编译模块


## vmlinux编译过程分析


vmlinux -> zImage -> uImage
