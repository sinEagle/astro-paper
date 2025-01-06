---
title: Linux Kernel Interrupt
author: sineagle
pubDatetime: 2024-11-26T13:57:45
modDatetime: 
featured: false
draft: false
tags:
  - Linux
  - kernel
description:
---
## Table of contents

## 中断介绍

中断分类：

按中断产生源分为：

- 同步（synchronous）：又称为异常，由于CPU自身执行指令产生
  - faults
  - traps
  - aborts
  - int n
- 异步（asynchronous）：由外部IO设备产生

根据是否可以临时禁用中断分为

- 可屏蔽中断（maskable）

  - 通过INT引脚
- 不可屏蔽中断（non-maskable）

  - 通过NMI引脚



## 中断子系统框架

## 中断控制器

负责处理中断优先级、屏蔽、使能

## Linux内核中断接口

- request_irq()
- free_irq()

## Linux中断处理流程

request_irq  -> request_threaded_irq ->

## 中断上半部和下半部

## end
