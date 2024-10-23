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
gcc -S hello.i -o hello.s
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

静态链接的过程如下：
![](https://cdn.jsdelivr.net/gh/sineagle/pic_image@master/20241021152506.png)

使用链接器时，会根据所引用其他模块的函数和全局变量无需知道它们的地址，链接器在链接时会根据引用的符号自动去相应的模块查找真正的地址。

创建a.c和b.c，分别进行`gcc -c`生成`a.o`和`b.o`文件，

```c
// a.c
#include <stdio.h>

int main () {
        int a = 2, b = 3;
        int c = sum (a, b);
        printf("%d\n", c);
        return 0;
}
```

```c
// b.c
int sum (int l, int r) {
        return l + r;
}
```

```bash
gcc -c a.c -o a.o -g
gcc -c b.c -o b.o -g
objdump -dSCl a.o
gcc a.o b.o -o ab
objdump -dSCl ab
```

反编译`a.o`和`ab.o`进行对比
![](https://cdn.jsdelivr.net/gh/sineagle/pic_image@master/20241021180011.png)
![](https://cdn.jsdelivr.net/gh/sineagle/pic_image@master/20241021180045.png)

在单独编译时，无法知道sum函数的地址，所以将目标地址设置成0，当链接器链接时进行了**目标地址的修正**， 这个地址修正的过程叫做**重定位**（Relocation），每个要被修正的地方叫一个**重定位入口**（Relocation Entry）。

## 目标文件

目前可执行文件格式Windows下的PE（Portable Executable Linkable Format）和Linux下的ELF（Executable Linkable Format）都是COFF（Common file format）格式的变种。目标文件就是源代码编译但未进行链接的哪些中间文件（.obj或.o文件）。
动态链接库（.dll、.so）、静态链接库（.lib、.a）文件也都按照可执行文件格式存储
ELF标准中把系统中采用的ELF文件格式归为以下4类


| ELF文件类型      | 说明                                                                                                                                              | 实例                             |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------- |
| 可重定位目标文件 | 包含了代码和数据、静态链接库                                                                                                                      | Linux下.o、Windows下的.obj       |
| 可执行文件       | 可以直接执行的程序                                                                                                                                | /bin/bash下的文件、Windows的.exe |
| 共享目标文件     | 包含代码和数据，1.链接器使用这种文件跟其他可重定位文件和目标文件进行链接 2.动态连接器将这几种共享目标文件和可执行文件结合、进程映像的一部分来运行 | Linux下的.so Windows下的DLL      |
| 核心转储·文件   | 当进程意外终止时，系统可以将改进程的地址空间的内容及终止时的一些其他信息转储到核心转储文件                                                        | Linux下的 core dump              |

```bash
@sinserver:~/study# file perf_event_open.o
perf_event_open.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
root@sinserver:~/study# file perf_event_open
perf_event_open: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=364416fe87f07e054089c81de8852d4ee865d22e, for GNU/Linux 3.2.0, with debug_info, not stripped
```

通过`file`命令查看对应的文件格式，通过`readelf`命令查看程序头

```bash

root@sinserver:~/study# readelf -h perf_event_open.o
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          0 (bytes into file)
  Start of section headers:          1496 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           0 (bytes)
  Number of program headers:         0
  Size of section headers:           64 (bytes)
  Number of section headers:         14
  Section header string table index: 13

root@sinserver:~/study# readelf -h perf_event_open
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Position-Independent Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x1140
  Start of program headers:          64 (bytes into file)
  Start of section headers:          18808 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         13
  Size of section headers:           64 (bytes)
  Number of section headers:         37
  Section header string table index: 36
```

### 目标文件的格式

ELF文件格式分为：

- ELF文件头 (ELF Header)
- 程序头表（Program header table）
- 段表/节头部表（Section Header Table）
- 重定位表（Relcation Table）
- 字符串表
- 符号表
- 各个段
  ------

### ELF文件头 (ELF Header)

包含整个文件的基本属性，文件头中定义了ELF魔数、文件机器字节长度、数据存储方式、版本、运行平台、ABI版本、ELF重定位类型、硬件平台、硬件平台版本、入口地址、程序头入口和长度、段表的位置及段的数量。

```bash
root@sinserver:~/study# readelf -h perf_event_open
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Position-Independent Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x1140
  Start of program headers:          64 (bytes into file)
  Start of section headers:          18808 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         13
  Size of section headers:           64 (bytes)
  Number of section headers:         37
  Section header string table index: 36
```

ELF魔数：最前面的 Magic 的十六个字节表示

- 第0个字节对应ASCII字符里面的DEL控制符，第1~3个字节刚好是ELF三个字母的ASCII值，几乎所有的可执行文件开始的几个字节都是这个魔数，这种魔数用于确认文件的类型
- 第4个字节标识文件的位数，0x01表示32位，0x02表示64位
- 第5个字节表示文件的字典序，0x01表示小端序、0x02表示大端序
- 第6个字节表示ELF文件的主版本号，一般是1
- 后面的9个字节ELF标准没有定义，作为扩展使用


### 段表/节头部表（Section Header Table）

描述了ELF各个段的信息，比如每个段的段名、段的长度、在文件中的偏移、读写权限及段的其他属性。ELF的段结构就是由段表决定的。

```bash
root@sinserver:~/study# readelf -W -S perf_event_open
There are 37 section headers, starting at offset 0x4978:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .interp           PROGBITS        0000000000000318 000318 00001c 00   A  0   0  1
  [ 2] .note.gnu.property NOTE            0000000000000338 000338 000030 00   A  0   0  8
  [ 3] .note.gnu.build-id NOTE            0000000000000368 000368 000024 00   A  0   0  4
  [ 4] .note.ABI-tag     NOTE            000000000000038c 00038c 000020 00   A  0   0  4
  [ 5] .gnu.hash         GNU_HASH        00000000000003b0 0003b0 000024 00   A  6   0  8
  [ 6] .dynsym           DYNSYM          00000000000003d8 0003d8 000150 18   A  7   1  8
  [ 7] .dynstr           STRTAB          0000000000000528 000528 0000d1 00   A  0   0  1
  [ 8] .gnu.version      VERSYM          00000000000005fa 0005fa 00001c 02   A  6   0  2
  [ 9] .gnu.version_r    VERNEED         0000000000000618 000618 000040 00   A  7   1  8
  [10] .rela.dyn         RELA            0000000000000658 000658 0000c0 18   A  6   0  8
  [11] .rela.plt         RELA            0000000000000718 000718 0000c0 18  AI  6  24  8
  [12] .init             PROGBITS        0000000000001000 001000 00001b 00  AX  0   0  4
  [13] .plt              PROGBITS        0000000000001020 001020 000090 10  AX  0   0 16
  [14] .plt.got          PROGBITS        00000000000010b0 0010b0 000010 10  AX  0   0 16
  [15] .plt.sec          PROGBITS        00000000000010c0 0010c0 000080 10  AX  0   0 16
  [16] .text             PROGBITS        0000000000001140 001140 000253 00  AX  0   0 16
  [17] .fini             PROGBITS        0000000000001394 001394 00000d 00  AX  0   0  4
  [18] .rodata           PROGBITS        0000000000002000 002000 00002b 00   A  0   0  4
  [19] .eh_frame_hdr     PROGBITS        000000000000202c 00202c 00003c 00   A  0   0  4
  [20] .eh_frame         PROGBITS        0000000000002068 002068 0000cc 00   A  0   0  8
  [21] .init_array       INIT_ARRAY      0000000000003d80 002d80 000008 08  WA  0   0  8
  [22] .fini_array       FINI_ARRAY      0000000000003d88 002d88 000008 08  WA  0   0  8
  [23] .dynamic          DYNAMIC         0000000000003d90 002d90 0001f0 10  WA  7   0  8
  [24] .got              PROGBITS        0000000000003f80 002f80 000080 08  WA  0   0  8
  [25] .data             PROGBITS        0000000000004000 003000 000010 00  WA  0   0  8
  [26] .bss              NOBITS          0000000000004010 003010 000008 00  WA  0   0  1
  [27] .comment          PROGBITS        0000000000000000 003010 000026 01  MS  0   0  1
  [28] .debug_aranges    PROGBITS        0000000000000000 003036 000030 00      0   0  1
  [29] .debug_info       PROGBITS        0000000000000000 003066 00071c 00      0   0  1
  [30] .debug_abbrev     PROGBITS        0000000000000000 003782 0001d5 00      0   0  1
  [31] .debug_line       PROGBITS        0000000000000000 003957 0000c7 00      0   0  1
  [32] .debug_str        PROGBITS        0000000000000000 003a1e 000651 01  MS  0   0  1
  [33] .debug_line_str   PROGBITS        0000000000000000 00406f 0000f5 01  MS  0   0  1
  [34] .symtab           SYMTAB          0000000000000000 004168 000420 18     35  18  8
  [35] .strtab           STRTAB          0000000000000000 004588 000281 00      0   0  1
  [36] .shstrtab         STRTAB          0000000000000000 004809 00016a 00      0   0  1

```

readelf -S 输出的结果就是段表的内容


### 重定位表

链接器在处理目标文件时，需要对目标文件一些部位进行重定位，修正地址引用的位置，对于每个需要重定位的代码段或数据段，都需要一个相应的重定位表。比如`.rel.txt`就是`.text`的重定位表

### BSS段

未初始化的全局变量和局部静态变量，只是预留一个**未定义的全局变量符号**，等最终链接为可执行文件时再分配空间

### 代码段

### .data段

保存已经初始化的全局静态变量和局部静态变量，前4个字节涉及CPU字节序

### .rodata

存放只读数据

END
