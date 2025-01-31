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

## 编译和链接

通过我们直接使用gcc编译运行源文件时，实际分为四个过程，整体过程如下所示

* 预处理（Prepressing)
* 编译（Compilation)
* 汇编(Assembly)
* 链接(Linking)

---

![](assets/ad327.png)

### 预编译

gcc 加上-E参数只进行预编译，或者使用cpp命令,生成.i文件

```bash
gcc -E hello.c -o hello.i
cpp hello.c > hello.i
```

* 展开所有的宏定义
* 处理条件预编译指令
* 删除注释、添加调试的行号、
* 保留#pramgma编译器指令

### 编译

词法分析、语法分析、语义分析，优化生产相应的汇编代码文件。

```bash
gcc -S hello.i -o hello.s
cc1 hello.c
```

gcc实际上是后台程序封装，根据不同参数要求去调用：cc1、as、ld

由于早期直接使用机器语言和汇编语言编写指令十分费事，而且总是依赖于特定的机器，无法令人接受，所以人们期望采用自然语言的形式来描述程序，后来诞生出了各种编程语言

编译过程可以分为六步：

- 词法分析
- 语法分析
- 语义分析
- 中间语言生成
- 目标代码优化

#### 词法分析

将源代码输入到扫描器，进行词法分析，将源代码的字符序列分割成一系列的记号，

词法分析产生的记号一般可以分为如下几类：关键字、标识符、字面量（包括数字、字符串等）和特殊符号（如加号、等号）。在识别记号的同时，并将标识符存放到符号表，将数字、字符串常量存放到文字表

* `int` -> 关键字
* `a` -> 标识符
* `=` -> 运算符
* `5` -> 常量
* `;` -> 分号

#### 语法分析

语法分析器将对由扫描器产生的记号进行语法分析，生成语法树表达式，检查语法的正确性（比如表达式不正确、括号不匹配、缺少操作符等）

```cpp
int a = b + c;
    =
   / \
  a   +
     / \
    b   c

```

#### 语义分析

语法分析是在语句上检查是否有意义

编译期分析的语意是**静态语意**，检查不同类型之间进行运算、不同类型数值之前的转换是否合法

运行期间进行分析的语意是**动态语义**，比如除零不合法

语义分析器还对符号表里的符号类型也做了更新。

#### 中间语言的生成

将语法树转化为与硬件无关的中间语言

中间代码通常采用三地址码、P代码形式

比如a = b + c * d; 生成的三地址码如下

```cpp
t1 = c * d
t2 = b + t1
a = t2
```

编译期前端将产生机器无关的中间代码，编译器后端针对不同的机器平台生成目标机器代码

#### 目标代码生成与优化

编译器后端包括**代码生成器**和**目标代码优化器**

代码生成器将中间代码转换成目标机器代码

目标代码优化器对目标代码进行优化，用位移代替乘法：1<<32 等等价于 2^32，删除多余的指令等

### 汇编

通过汇编器将汇编代码转换成机器指令，生成目标文件（Object File）

~~~bash
as hello.s -o hello.o
gcc -c hello.s -o hello.o
~~~

通过汇编器将汇编代码转换成机器指令，生成目标文件（Object File）

在这个过程中会生重定位表、符号表，便于后续链接过程

### 链接

分为动态链接和静态链接

分为动态链接和静态链接

* 链接的过程包括地址和空间分配（Address and Storage Allocation）、符号决议（Symbol Resolution）和重定位（Relocation）这几步
* 使用链接器时，会根据所引用其他模块的函数和全局变量无需知道它们的地址，链接器在链接时会根据引用的符号自动去相应的模块查找真正的地址。

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

目标文件的结构因平台和格式而异，但通常遵循某种文件格式，如 ELF（Executable and Linkable Format）在 Linux 中广泛使用。

目前可执行文件格式Windows下的PE（Portable Executable Linkable Format）和Linux下的ELF（Executable Linkable Format）都是COFF（Common file format）格式的变种。目标文件就是源代码编译但未进行链接的哪些中间文件（.obj或.o文件）。
动态链接库（.dll、.so）、静态链接库（.lib、.a）文件也都按照可执行文件格式存储
ELF标准中把系统中采用的ELF文件格式归为以下4类


| ELF文件类型      | 说明                                                                                                                                              | 实例                             |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------- |
| 可重定位目标文件 | 包含了代码和数据、进行链接后会生成可执行文件、共享目标文件                                                                                        | Linux下.o、Windows下的.obj       |
| 可执行文件       | 可以直接执行的程序                                                                                                                                | /bin/bash下的文件、Windows的.exe |
| 共享目标文件     | 包含代码和数据，1.链接器使用这种文件跟其他可重定位文件和目标文件进行链接 2.动态连接器将这几种共享目标文件和可执行文件结合、进程映像的一部分来运行 | Linux下的.so Windows下的DLL      |
| 核心转储·文件   | 当进程意外终止时，系统可以将改进程的地址空间的内容及终止时的一些其他信息转储到核心转储文件                                                        | Linux下的 core dump              |

```bash
root@wangzy115-bp3eh ~/s/Link (link_example_1)# file a.o
a.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), with debug_info, not stripped

root@wangzy115-bp3eh ~/s/Link (link_example_1)# file ab
ab: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, with debug_info, not stripped

```

通过`file`命令查看对应的文件格式，通过`readelf`命令查看程序头

```bash
root@wangzy115-bp3eh ~/s/Link (link_example_1)# readelf -h ab
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
  Entry point address:               0x1060
  Start of program headers:          64 (bytes into file)
  Start of section headers:          15176 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         12
  Size of section headers:           64 (bytes)
  Number of section headers:         34
  Section header string table index: 33


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

![](assets/20241229_232939_image.png)

### ELF文件头 (ELF Header)

包含整个文件的基本属性，文件头中定义了ELF魔数、文件机器字节长度、数据存储方式、版本、运行平台、ABI版本、ELF重定位类型、硬件平台、硬件平台版本、入口地址、程序头入口和长度、段表的位置及段的数量。

ELF魔数：最前面的 Magic 的十六个字节表示

- 第0个字节对应ASCII字符里面的DEL控制符，第1~3个字节刚好是ELF三个字母的ASCII值，几乎所有的可执行文件开始的几个字节都是这个魔数，这种魔数用于确认文件的类型
- 第4个字节标识文件的位数，0x01表示32位，0x02表示64位
- 第5个字节表示文件的字典序，0x01表示小端序、0x02表示大端序
- 第6个字节表示ELF文件的主版本号，一般是1
- 后面的9个字节ELF标准没有定义，作为扩展使用

当前内核是开启地址随机化的

```bash
root@wangzy115-bp3eh ~/s/Link (link_example_1)# cat /proc/sys/kernel/randomize_va_space

2

```

![](assets/20250106_190917_image.png)

![](assets/20250106_190952_image.png)

Type: DYN：动态共享对象文件或位置无关可执行文件

Position-Independent Executable (PIE)：

* `DYN` 类型文件通常是位置无关可执行文件（PIE) 或共享库（Shared Library）
* 这意味着代码可以被加载到内存中的任何地址，而不需要固定的加载基址。
* 使用位置无关代码（Position-Independent Code, PIC），使得代码中的绝对地址依赖被最小化。

动态加载与链接:

* DYN 文件通常需要一个动态链接器来加载和解析符号。
* 动态链接器负责将 DYN 文件和其依赖的库加载到内存，并处理符号重定位。

vvar (Virtual Variable Page)

* 是内核映射到用户态的一个只读虚拟内存区域。
* 用于为用户态程序提供某些内核状态信息，例如高效访问时间信息（如`CLOCK_MONOTONIC` 或`CLOCK_REALTIME`）
* 避免用户态调用系统调用获取这些数据，从而减少用户态与内核态的上下文切换，提高性能。

vdso (Virtual Dynamically Shared Object)

* 是内核为用户态程序提供的一个特殊的共享库（.so 文件）

在编译时加入`-no-pie`参数后，观察生成的可执行文件，Type变成了EXEC（Executable file），下面可以看到Entry point address的地址变化

![](assets/20250106_224031_image.png)

然后我们关闭地址随机化

![](assets/20250106_231218_image.png)

使用默认编译（pie）

![](assets/20250106_231449_image.png)

加入`-no-pie`参数后

![](assets/20250106_231159_image.png)


| 编译选项 / 状态 | ASLR 打开                            | ASLR 关闭                              |
| ----------------- | -------------------------------------- | ---------------------------------------- |
| **PIE 打开**    | 程序和其他内存段地址随机化，安全性高 | 程序和其他内存段地址固定，便于调试     |
| **PIE 关闭**    | 程序地址固定，堆、栈、共享库随机化   | 所有地址固定，最方便调试，但安全性最低 |

pie编译 + 是否打开随机化

![](assets/20250107_105100_image.png)

nopie编译 + 是否打开随机化

![](assets/20250107_104910_image.png)

### 节头部表（Section Header Table）

节头部表描述了ELF各个段的信息，比如每个段的段名、段的长度、在文件中的偏移、读写权限及段的其他属性。

ELF的段结构就是由段表决定的。通过 `readelf -S` 命令可以进行查看

![](assets/20241231_145630_1735628181271.jpg)

* **Name** - **节区名称（Section Name）**
  * 显示节区的名称。例如`.text`、`.data`、`.bss` 等。若没有名称，可能显示为`NULL` 或空字符串。
* **Type** - **节区类型（Section Type）**
  * 显示节区的类型，决定该节区的用途。例如：
    * `PROGBITS`：包含程序数据或代码。
    * `STRTAB`：字符串表，存储字符串。
    * `DYNSYM`：动态符号表，包含动态链接时使用的符号。
    * `RELA`：重定位表，包含程序的重定位信息。
    * `GNU_HASH`：GNU 格式的符号哈希表等。
* **Address** - **节区加载地址（Section Address）**
  * 显示该节区在内存中的加载地址。该地址指示节区在程序运行时加载到内存中的位置。若节区不需要加载到内存（如`.bss`），此字段可能为`0` 或`0x0`。
* **Off** - **节区偏移（Section Offset）**
  * 表示节区在文件中的偏移量，也就是该节区从文件开头开始的字节位置。它决定了节区在 ELF 文件中的起始位置。
* **Size** - **节区大小（Section Size）**
  * 显示节区的字节大小，表示该节区在文件或内存中的占用空间大小。如果节区没有实际数据（例如`.bss`），它可能为 0 或一个虚拟大小。
* **ES** - **节区条目大小（Entry Size）**
  * 对于某些类型的节区（如符号表、重定位表等），显示节区条目的大小。即每个元素所占用的字节数。
    * 例如，符号表`.dynsym` 的条目大小是 18 字节。
* **Flg** - **节区标志（Flags）**
  * 显示节区的标志，描述节区的特性，使用字符表示：
    * `A`：节区可分配（Allocated），表示它会被加载到内存。
    * `X`：节区可执行（Executable），表示它包含可执行代码。
    * `W`：节区可写（Writable），表示它可以被修改。
    * `I`：信息（Info），通常表示该节区包含信息，比如重定位信息。
    * `L`：包含本地符号（Local），该节区只包含局部符号。
    * `M`：合并（Merge），表示节区支持合并。
    * `S`：支持分段（String），通常用于字符串表。
* **Lk** - **节区链接（Section Link）**
  * 表示节区之间的链接。通常指向与该节区相关的节区或段。比如对于符号表，它会链接到字符串表。
* **Inf** - **节区信息（Section Info）**
  * 对于某些类型的节区，表示附加的节区信息。例如，对于符号表，它可能是符号的数量；对于重定位表，它可能是重定位条目的数量。
* **Al** - **节区对齐（Section Alignment）**
  * 表示节区在内存中的对齐要求。节区的起始位置必须满足特定的对齐约束。
  * 例如，`Al` 的值可能为 4，表示节区必须在 4 字节对齐的位置开始；如果为 16，则意味着节区必须在 16 字节对齐的位置开始。

### 程序头表（Program header table）

描述ELF的Segment，用于加载程序时的内存映射

通过`readelf -l`命令查看可执行文件中的段

`LOAD`类型的segment需要被映射，根据Flg不同的标志位映射到不同的`VMA`，只有可执行文件和共享库文件才需要有程序头表，目标文件是没有的，面向运行时内存映射

`readelf -l ab -W`

![](assets/20241231_155558_image.png)

`LOAD`类型的segment需要被映射

根据Flg不同的标志位映射到不同的`VMA`

只有可执行文件和共享库文件才需要有程序头表，目标文件是没有的

### 重定位表

链接器怎么知道哪些指令是需要被调整的呢？

* 在ELF文件中有重定位表用来描述如何修改相应段里的内容
* 通过 `objdump -r` 或 `readelf -r` 命令可以查看对应的目标文件需要重定位的地方

链接器在处理目标文件时，需要对目标文件一些部位进行重定位，修正地址引用的位置，对于每个需要重定位的代码段或数据段，都需要一个相应的重定位表。比如`.rel.text`就是`.text`的重定位表

![](assets/20241231_153451_1735630488864.jpg)

![](assets/20250107_143308_1736231577252.jpg)

* Offset: 表示需要纠正的符号引用的起始位置在目标段的偏移
* Info: 包含符号索引和类型信息。
  * 前 8 位表示符号索引，指向目标文件符号表 (.symtab)
  * 后 8 位表示重定位类型（如 0002 表示 R_X86_64_PC32，0004 表示 R_X86_64_PLT32）。
* Type: 重定位类型，表示如何修正地址。
  * `R_X86_64_PC32`：相对地址重定位。
  * `R_X86_64_32`：绝对地址重定位。
  * `R_X86_64_JUMP_SLOT`：函数调用入口的重定位。
* Sym.Value: 符号的当前值（在未链接时为 0）。
* Sym.Name + Addend: 符号名称偏移量

![](assets/20241231_154533_1735631125954.jpg)

![](assets/20241231_154317_1735630993510.jpg)

### BSS段

BSS段未初始化的全局变量和静态变量，只是预留一个**未定义的全局变量符号**，等最终链接为可执行文件时再分配空间

初始化为零的变量也会被归入 BSS 段，因为它们无需在目标文件中存储实际值。

当程序加载到内存时，BSS 段中的所有变量会自动初始化为零。

### 代码段

代码段也称为 `.text` 段，是目标文件或可执行文件中存储程序**机器指令** 的一部分。它包含程序的实际执行代码

### .data段

保存已经初始化的全局静态变量和局部静态变量，前4个字节涉及CPU字节序

数据段是目标文件或可执行文件中专门用于存储**已初始化的全局变量和静态变量** 的内存区域。这些变量在程序执行期间保留其值，直到程序结束。

### .rodata

存放只读数据

### .symtab表

每个目标文件都有一个符号表，记录了所有到的所有的符号，每个符号都有一个对应的值，叫做符号值，对于函数或变量来说，符号值就是他们的地址，当然还存在其他的符号，分类如下：

* 定义在本目标文件中的全局符号
* 所引用的全局符号，即外部符号
* 段名，由编译器产生，值是该段的其实地址
* 局部符号，编译单元内部可见
* 行号信息，目标文件中的指令和源代码中的对应关系

![](assets/20250107_144456_image.png)

![](assets/20241231_153038_1735630222548.jpg)

### 强符号和弱符号

强符号：函数名、初始化的全局变量

* 不允许多次定义
* 强符号可以覆盖弱符号

弱符号：未初始化的全局变量

* 允许多个弱符号
* 编译期间由于不知道弱符号的大小，通过COMMON符号标记
* 链接期间，当存在多个弱符号，选择占用空间最大的

强引用和弱引用

- 强引用：最终链接为可执行文件时，如果没有符号的定义，链接器会报符号未定义错误，这种被称为强引用（Strong Reference)
- 弱引用：处理弱引用时，如果该符号有定义，则链接器将该符号的引用决议；如果该符号未被定义，则链接器对于该引用不报错

## 静态链接

链接的过程分为两步：

1. 空间与地址分配： 扫描所有的输入目标文件，并且获取它们的各个段的长度、属性和位置，并且将输入符号表中所有的符号定义和符号引用都收集起来，统一放到一个全局符号表。链接器将各个段合并，建立映射关系
2. 符号解析和重定位：读入文件中段的数据、重定位信息，进行符号解析与重定位、调整代码中的地址。

### 空间和地址分配

当链接后，使用`objdump -h`命令可以查看到对应段的VMA和LMA都已经分配好了地址。

之后开始计算各个符号的虚拟地址，需要给每个符号添加一个偏移量，调整到正确的虚拟地址。

### 符号解析和重定位

由于编译的时候，并不知道定义在其他目标文件中的符号的地址，所以先用一个临时的假地址，就可以根据符号对需要重定位的地址进行绝对修正或相对修正到正确的位置。

### 静态库链接

静态库实际上是一组目标文件的集合

可以通过ar压缩程序将目标文件压缩到一起，比如`libc.a`这个静态库文件

```bash
ar -t /usr/lib/x86_64-linux-gnu/libc.a # 列出归档文件包含的目标文件
```

### 链接器的控制

编写一个不依赖库函数的 "最小程序"

```cpp
char* str = "Hello World!\n";

void print () {
        asm ( "movl $13, %%edx \n\t"
             "movl %0, %%ecx \n\t"
             "movl $0, %%ebx \n\t"
             "movl $4, %%eax \n\t"
             "int $0x80 \n\t"
             ::"r"(str):"edx","ecx","ebx");
}
void exit() {
        asm ("movl $42, %ebx \n\t"
             "movl $1, %eax \n\t"
             "int $0x80 \n\t" );
}
void nomain () {
        print();
        exit();
}
```

```bash
gcc -c -fno-builtin -nostdlib -static -m64 -fno-asynchronous-unwind-tables \
   -fno-pic -fno-pie TinyHelloWorld.c -o TinyHelloWorld.o
ld -static -e nomain -o TinyHelloWorld \
   TinyHelloWorld.o -m elf_x86_64

```

默认情况下，编译器可能会生成位置无关代码，这会导致创建 `.got.plt` 段, 所以加上`-fno-pic -fno-pie` 参数去除

使用 `-fno-asynchronous-unwind-tables` 禁用栈展开表`.eh_frame`的生成

观察到这样生成的可执行文件的段如下：

```bash
root@sinserver:~/study# readelf -S TinyHelloWorld
There are 8 section headers, starting at offset 0x2144:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        08049000 001000 000044 00  AX  0   0  1
  [ 2] .rodata           PROGBITS        0804a000 002000 00000e 00   A  0   0  1
  [ 3] .data             PROGBITS        0804b010 002010 000004 00  WA  0   0  4
  [ 4] .comment          PROGBITS        00000000 002014 000026 01  MS  0   0  1
  [ 5] .symtab           SYMTAB          00000000 00203c 000090 10      6   2  4
  [ 6] .strtab           STRTAB          00000000 0020cc 000040 00      0   0  1
  [ 7] .shstrtab         STRTAB          00000000 00210c 000038 00      0   0  1

```

.text存放的是程序的指令

.rodata存放的是字符串"Hello World!"

.data存放的是str全局变量

.comment存放的是编译器和系统版本信息

编写链接脚本`TinyHelloWorld.lds`，将上面四个公用的段合并成一个

```bash
ENTRY(nomain)

SECTIONS
{
        . = 0x08048000 + SIZEOF_HEADERS;
        tinytext : { *{.text} *{.data} *{.rodata} }
        /DISCARD/ : { {*.comment} }
}

```

```bash
gcc -c -fno-builtin -nostdlib -static -m32 -fno-asynchronous-unwind-tables -fno-pic -fno-pie TinyHelloWorld.c -o TinyHelloWorld.o

ld -static -T TinyHelloWorld.lds -o TinyHelloWorld TinyHelloWorld.o -m elf_i386

```

```bash
root@sinserver:~/study# readelf -S TinyHelloWorld
There are 5 section headers, starting at offset 0x198:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] tinytext          PROGBITS        08048094 000094 000056 00 WAX  0   0  4
  [ 2] .symtab           SYMTAB          00000000 0000ec 000060 10      3   2  4
  [ 3] .strtab           STRTAB          00000000 00014c 000028 00      0   0  1
  [ 4] .shstrtab         STRTAB          00000000 000174 000024 00      0   0  1

```

观察到已经合并成功，但是还有三个段

- .symtab : 符号表
- .strtab : 字符表
- .shstrtab : 段名称字符表

ld命令加上 `-s` 去除字符表和符号表，.shstrtab是保存段名是不少的

```bash
root@sinserver:~/study# readelf -S TinyHelloWorld
There are 3 section headers, starting at offset 0x100:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] tinytext          PROGBITS        08048094 000094 000056 00 WAX  0   0  4
  [ 2] .shstrtab         STRTAB          00000000 0000ea 000014 00      0   0  1


root@sinserver:~/study# ./TinyHelloWorld
Hello World!

```

这样就生成了一个最小的ELF可执行文件啦

## 可执行文件的装载

### 从操作系统的角度看可执行文件的装载

首先要进行进程的创建

- 创建虚拟地址空间
  - 虚拟地址空间由一组页映射函数将虚拟地址空间的各个页映射到相应的物理地址空间，创建虚拟地址空间就是创建映射函数所需要的相应的数据结构，在i386的Linux中，创建虚拟地址空间实际上是分配一个页目录，不需要建立页的映射关系，这些映射关系等到后面发生页错误的时候再进行设置，完成虚拟空间和物理内存的映射
- 读取可执行文件头，建立虚拟地址空间与可执行文件的映射关系。
  - 当缺页异常发生时，程序需要知道所需要的页在可执行文件中的哪个位置，这也是"装载"的重要过程

![](assets/20241025_113432_image.png)

这种映射关系只是保存在操作系统内部的一个数据结构，Linux中将进程虚拟空间中的一个段叫做虚拟内存区域（VMA），OS在创建进程后，在进程相应数据结构设置相应的.text段的VMA，将CPU指令寄存器设置为可执行文件的入口，启动运行，将控制权转交给进程，跳转到可执行文件的入口地址。

### 进程虚拟地址分布

在可执行文件里，包含的各种段数量增多时，会产生空间浪费的问题，在OS进行装在时，可以不关系每个段包含的实际内容，最主要的是**权限问题（可读、可写、可执行）**。ELF文件中，段的权限往往只有为数不多的几种组合

- 可读可执行：代码段
- 可读可写：数据段、BSS段等
- 只读：其他

所以将相同权限的段，可以合并到一个段中进行映射

Segment包含一个或多个属性相似的"Section"，装载的时候可以将他们看作一个整体一起映射，对应一个VMA,可以减少页面内部碎片，节省内存空间

通过`readelf -l`查看ELF的 segment，上面的程序头表有介绍

```bash
root@sinserver:~/study# readelf -l SectionMapping.elf -W

Elf file type is EXEC (Executable file)
Entry point 0x401720
There are 10 program headers, starting at offset 64

Program Headers:
  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
  LOAD           0x000000 0x0000000000400000 0x0000000000400000 0x0004f8 0x0004f8 R   0x1000
  LOAD           0x001000 0x0000000000401000 0x0000000000401000 0x07d6b1 0x07d6b1 R E 0x1000
  LOAD           0x07f000 0x000000000047f000 0x000000000047f000 0x0259ec 0x0259ec R   0x1000
  LOAD           0x0a4f50 0x00000000004a5f50 0x00000000004a5f50 0x005b78 0x00b2f8 RW  0x1000
  NOTE           0x000270 0x0000000000400270 0x0000000000400270 0x000030 0x000030 R   0x8
  NOTE           0x0002a0 0x00000000004002a0 0x00000000004002a0 0x000044 0x000044 R   0x4
  TLS            0x0a4f50 0x00000000004a5f50 0x00000000004a5f50 0x000018 0x000058 R   0x8
  GNU_PROPERTY   0x000270 0x0000000000400270 0x0000000000400270 0x000030 0x000030 R   0x8
  GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW  0x10
  GNU_RELRO      0x0a4f50 0x00000000004a5f50 0x00000000004a5f50 0x0040b0 0x0040b0 R   0x1

 Section to Segment mapping:
  Segment Sections...
   00     .note.gnu.property .note.gnu.build-id .note.ABI-tag .rela.plt
   01     .init .plt .text .fini
   02     .rodata .stapsdt.base rodata.cst32 .eh_frame .gcc_except_table
   03     .tdata .init_array .fini_array .data.rel.ro .got .got.plt .data .bss
   04     .note.gnu.property
   05     .note.gnu.build-id .note.ABI-tag
   06     .tdata .tbss
   07     .note.gnu.property
   08
   09     .tdata .init_array .fini_array .data.rel.ro .got


```

在OS中，VMA除了可以映射可执行文件的各个Segment以外，还可以对进程地址空间进行控制，比如我们需要的栈和堆在进程虚拟地址空间也是以VMA的形式存在的。

```bash
root@sinserver:~/study# ./SectionMapping.elf &
[1] 15664
root@sinserver:~/study# cat /proc/15664/maps
00400000-00401000 r--p 00000000 fc:00 3969119                            /root/study/SectionMapping.elf
00401000-0047f000 r-xp 00001000 fc:00 3969119                            /root/study/SectionMapping.elf
0047f000-004a5000 r--p 0007f000 fc:00 3969119                            /root/study/SectionMapping.elf
004a5000-004aa000 r--p 000a4000 fc:00 3969119                            /root/study/SectionMapping.elf
004aa000-004ac000 rw-p 000a9000 fc:00 3969119                            /root/study/SectionMapping.elf
004ac000-004b2000 rw-p 00000000 00:00 0
01b63000-01b85000 rw-p 00000000 00:00 0                                  [heap]
7fff5fa62000-7fff5fa83000 rw-p 00000000 00:00 0                          [stack]
7fff5fbbc000-7fff5fbc0000 r--p 00000000 00:00 0                          [vvar]
7fff5fbc0000-7fff5fbc2000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0                  [vsyscall]
root@sinserver:~/study#

```

可以看到，只有前面几个是映射到可执行文件的Segment，后面几个主设备号和此设备号都是0，表示没有映射到为你教案中，这种VMA叫做**匿名虚拟内存区域（Anonymous Virtual Memory Area）**，每个进程都有自己的堆栈

### Linux下ELF文件的装载

其实do_execve系统调用的主要功能，步骤如下：

- 检查ELF可执行文件的有效性（魔数、程序头表中Segment数量）
- 寻找动态链接 `.interp`段，设置动态链接路径
- 根据ELF可执行文件的程序头表的描述，对ELF文件进行应映射（代码、数据段...)
- 初始化ELF进程环境，进程启动时的EDX地址应该是DT_FINI的地址
- 将系统调用的返回地址修改为ELF可执行文件的入口点，对于静态链接，是`e_entry`所指向的地址；对于动态链接的可执行文件，程序入口点是动态链接器
- 最后返回到用户态，EIP寄存器已经跳到ELF程序的入口，从新程序开始执行，装载完成

## 动态链接

动态链接不需要对那些组成程序的目标文件进行链接，而是等到程序要运行才进行链接。把这个过程推迟到运行时再进行，这就是动态链接（Dynamic Linking）的基本思想。

比如`pro1`和`pro2`程序，首先运行`proc1`，前面链接过程和静态链接类似，将依赖的目标文件加载到内存后，`proc2`运行时，对于依赖相同的目标文件不需要重新加载到内存中，而是直接共享。

在Linux下，ELF的动态链接文件称为**动态共享对象**, 以`.so`为扩展名

### 动态链接例子

```cpp
// Lib.c
#include <stdio.h>

void foobar (int i) {
        printf ("Printing from Lib.so %d\n", i);
}

// Lib.h
#ifndef LIB_H
#define LIB_H

void foobar (int i);

#endif
```

```bash
gcc -fPIC -shared -o Lib.so Lib.c

gcc -o Program1 Program1.c ./Lib.so
```

![](assets/20241025_152229_image.png)

当Program1.o被链接为可执行文件时，Lib.o没有被链接起来

回到动态链接机制，如果所引用的函数或变量时在动态共享对象中，那么链接器会将这个符号的引用标记为一个动态链接的符号，不对它进行地址重定位，把这个过程留到装载时再进行。

链接器如何知道符号的引用是静态链接还是动态链接？

- 之前的Lib.so中保存了完整的符号信息，将它作为链接的文件之一，链接器在解析时就可以知道，xxx就是定义在Lib.so的动态符号，这样链接器就可以对这个引用做特殊处理，成为一个动态符号的引用

编写Program1.c

```cpp
#include "Lib.h"

int main () {
        foobar(5);

        return 0;
}

```

```bash
root@sinserver:~/study# ./Program1 &
[2] 16094
Printing from Lib.so 5
root@sinserver:~/study# cat /proc/16094/maps
593d06eca000-593d06ecb000 r--p 00000000 fc:00 3969133                    /root/study/Program1
593d06ecb000-593d06ecc000 r-xp 00001000 fc:00 3969133                    /root/study/Program1
593d06ecc000-593d06ecd000 r--p 00002000 fc:00 3969133                    /root/study/Program1
593d06ecd000-593d06ece000 r--p 00002000 fc:00 3969133                    /root/study/Program1
593d06ece000-593d06ecf000 rw-p 00003000 fc:00 3969133                    /root/study/Program1
593d084ae000-593d084cf000 rw-p 00000000 00:00 0                          [heap]
7da9fb400000-7da9fb428000 r--p 00000000 fc:00 302893                     /usr/lib/x86_64-linux-gnu/libc.so.6
7da9fb428000-7da9fb5b0000 r-xp 00028000 fc:00 302893                     /usr/lib/x86_64-linux-gnu/libc.so.6
7da9fb5b0000-7da9fb5ff000 r--p 001b0000 fc:00 302893                     /usr/lib/x86_64-linux-gnu/libc.so.6
7da9fb5ff000-7da9fb603000 r--p 001fe000 fc:00 302893                     /usr/lib/x86_64-linux-gnu/libc.so.6
7da9fb603000-7da9fb605000 rw-p 00202000 fc:00 302893                     /usr/lib/x86_64-linux-gnu/libc.so.6
7da9fb605000-7da9fb612000 rw-p 00000000 00:00 0
7da9fb77c000-7da9fb77f000 rw-p 00000000 00:00 0
7da9fb786000-7da9fb787000 r--p 00000000 fc:00 3969129                    /root/study/Lib.so
7da9fb787000-7da9fb788000 r-xp 00001000 fc:00 3969129                    /root/study/Lib.so
7da9fb788000-7da9fb789000 r--p 00002000 fc:00 3969129                    /root/study/Lib.so
7da9fb789000-7da9fb78a000 r--p 00002000 fc:00 3969129                    /root/study/Lib.so
7da9fb78a000-7da9fb78b000 rw-p 00003000 fc:00 3969129                    /root/study/Lib.so
7da9fb78b000-7da9fb78d000 rw-p 00000000 00:00 0
7da9fb78d000-7da9fb78e000 r--p 00000000 fc:00 302890                     /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
7da9fb78e000-7da9fb7b9000 r-xp 00001000 fc:00 302890                     /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
7da9fb7b9000-7da9fb7c3000 r--p 0002c000 fc:00 302890                     /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
7da9fb7c3000-7da9fb7c5000 r--p 00036000 fc:00 302890                     /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
7da9fb7c5000-7da9fb7c7000 rw-p 00038000 fc:00 302890                     /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
7ffe349d4000-7ffe349f5000 rw-p 00000000 00:00 0                          [stack]
7ffe349f5000-7ffe349f9000 r--p 00000000 00:00 0                          [vvar]
7ffe349f9000-7ffe349fb000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0                  [vsyscall]

```

可以看到，在进程虚拟地址空间中，多了几个文件的映射。Libc.so和Program1一样，都被OS用同样的方式映射到了虚拟地址空间，只是占据的虚拟地址和长度不同。

通过 `readelf -l` 查看Lib.so的装载属性

```bash

root@sinserver:~/study# readelf -l Lib.so -W

Elf file type is DYN (Shared object file)
Entry point 0x0
There are 11 program headers, starting at offset 64

Program Headers:
  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
  LOAD           0x000000 0x0000000000000000 0x0000000000000000 0x000560 0x000560 R   0x1000
  LOAD           0x001000 0x0000000000001000 0x0000000000001000 0x000181 0x000181 R E 0x1000
  LOAD           0x002000 0x0000000000002000 0x0000000000002000 0x0000dc 0x0000dc R   0x1000
  LOAD           0x002df8 0x0000000000003df8 0x0000000000003df8 0x000220 0x000228 RW  0x1000
  DYNAMIC        0x002e08 0x0000000000003e08 0x0000000000003e08 0x0001c0 0x0001c0 RW  0x8
  NOTE           0x0002a8 0x00000000000002a8 0x00000000000002a8 0x000020 0x000020 R   0x8
  NOTE           0x0002c8 0x00000000000002c8 0x00000000000002c8 0x000024 0x000024 R   0x4
  GNU_PROPERTY   0x0002a8 0x00000000000002a8 0x00000000000002a8 0x000020 0x000020 R   0x8
  GNU_EH_FRAME   0x00201c 0x000000000000201c 0x000000000000201c 0x00002c 0x00002c R   0x4
  GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW  0x10
  GNU_RELRO      0x002df8 0x0000000000003df8 0x0000000000003df8 0x000208 0x000208 R   0x1

 Section to Segment mapping:
  Segment Sections...
   00     .note.gnu.property .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt
   01     .init .plt .plt.got .plt.sec .text .fini
   02     .rodata .eh_frame_hdr .eh_frame
   03     .init_array .fini_array .dynamic .got .got.plt .data .bss
   04     .dynamic
   05     .note.gnu.property
   06     .note.gnu.build-id
   07     .note.gnu.property
   08     .eh_frame_hdr
   09
   10     .init_array .fini_array .dynamic .got

```

除了为你类型，其他与普通程序基本一样，装载地址是从0x00000000开始的，从这里可以知道，**共享对象的最终装载地址在编译时是不确定的，而是在装载时，装载器根据当前地址空间的空闲情况，动态分配一块足够大小的虚拟地址空间给相应的共享对象**

Linux和GCC支持装载时重定位，需要加上 `-shared`和`fPIC`参数，如果只是用-shared，那么是输出的对象就是装置时重定位的方法。

### 地址无关技术

装载时重定位的缺点是指令部分无法在多个进程间进行共享，这样就失去了动态链接节省内存的一大优势。我们还需要一种更好的方法解决共享对象指令中的绝对地址重定向问题，如何解决呢？

通过地址无关技术，将指令中那些需要被修改的部分剥离出来，和数据部分放到一起，这样指令部分就可以保持不变，而数据部分可以在每个进程中拥有一个副本。

共享对象模块中的地址引用分为两种，模块内部引用和模块外部引用；按照不同的引用方式可以分为指令引用和数据引用

4种情况分别是：

- 函数模块内部调用、跳转
- 模块内部的数据访问（模块中定义的全局变量、局部变量）
- 模块外部的函数调用、跳转等
- 模块外部的数据访问，比如其他模块中定义的全局变量

模块当之间数据访问基本思想就是把地址相关的部分放到数据段里，这些其他模块的全局变量地址是和模块装载地址有关，ELF的做法是在数据段里面建立一个指向这些变量的指针数组，也被称为**全局偏移表（Global Offset Table）**，当代码需要引入该全局变量时，可以通过GOT中相对应的项间接引用。

当指令中访问哪些外部变量时，首先找到GOT，然后根据GOT中的变量所对应的项找到变量的目标地址，链接器在装载时会查找每个变量所在的地址，然后填充到GOT中的各个项，确保每个指针指向的地址正确，由于GOT本身是放在数据段的，所以它可以在模块装载时被修改，并且每个进程都可以有独立的副本，相互不受影响。

-fpic和fPIC区别：

`-fpic`产生的代码较少，而且更快，但是在某些平台上会有一些限制，方便期间，还是使用`fPIC`产生地址无关代码

### 延迟绑定（PLT）

动态链接以牺牲时间为代价比静态链接要慢一些，对全局和静态访问都要通过GOT定位，间接进行寻址；模块间的调用也是先重定位GOT，再间接进行跳转，而且动态链接是程序运行时完成，每当程序开始执行时，都会进行一次链接工作，大大影响程序的启动速度。

延迟绑定的基本思想就是当函数第一次被用到才开始绑定（符号查找、重定位），如果没有用到则不进行绑定。

ELF使用PLT（Procedure Linkage Table）来实现，当我们调用外部模块的函数时，按照通常做法是通过GOT中的项进行间接跳转。PLT为了实现延迟绑定，在这过程中间又增加了一层间接跳转。调用函数并不通过GOT跳转，而是通过`PLT`项的结构来进行跳转。

重定位过程：第一次调用时通过PLT表跳转到GOT，再跳到动态链接器进行链接共享库、重定位、修改GOT表真实符号地址。第二次调用，直接从GOT表中跳转到符号真实地址，执行函数

![](https://cdn.jsdelivr.net/gh/sineagle/pic_image@master/20241118143402.png)

编写main.c sub.c

```c
#include<stdio.h>
extern int num;
extern int val;
int add(int a, int b);
int sub(int a, int b);
int mul(int a, int b);

int main()
{
	add(1,2);
//	printf("&sub = %p ...\n",sub);
	sub(3,4);
//	printf("&sub = %p ...\n",sub);
	mul(5,6);
	printf("hello world!\n");
	printf("num=%d\n",num);
	printf("val=%d\n",val);
	return 0;
}
```

```c
int num = 10;
int val = 20;

int add(int a, int b)
{
	return a+b;
}

int sub(int a, int b)
{
	return a-b;
}

int mul(int a, int b)
{
	return a*b;
}

int div(int a, int b)
{
	return a/b;
}

```

```bash
gcc -fPIC -c sub.c -o sub.o
gcc -shared -o libsub.so sub.o

gcc -fPIC -shared -o libsub.so sub.c
gcc main.c -L. -lsub -Wl,-rpath=. -o main -Wl,-z,lazy

objdump -dSCl main > main.s

```

![](https://cdn.jsdelivr.net/gh/sineagle/pic_image@master/20241118154855.png)

![](https://cdn.jsdelivr.net/gh/sineagle/pic_image@master/20241118155003.png)

要调用add函数，首先进入plt表中，在进行计算：0x1094 (当前 RIP) + 0x2f16 (偏移) = 0x3fb0，跳转到GOT表相应的位置查看GOT表

```bash
root@ub2401-sin ~/s/link_study# readelf -x .got main

Hex dump of section '.got':
 NOTE: This section has relocations against it, but these have NOT been applied to this dump.
  0x00003f98 983d0000 00000000 00000000 00000000 .=..............
  0x00003fa8 00000000 00000000 30100000 00000000 ........0.......
  0x00003fb8 40100000 00000000 50100000 00000000 @.......P.......
  0x00003fc8 60100000 00000000 70100000 00000000 `.......p.......
  0x00003fd8 00000000 00000000 00000000 00000000 ................
  0x00003fe8 00000000 00000000 00000000 00000000 ................
  0x00003ff8 00000000 00000000                   ........ 
```

前面8个字节存放了.dynamic的地址

找到3fb0地址处的值为1030,其他的1040、1050类似，这些地址处的值为`endbr64`，

![](https://cdn.jsdelivr.net/gh/sineagle/pic_image@master/20241118224203.png)

### 动态链接相关结构

在动态链接的情况下，操作系统映射完可执行文件，会先启动一个**动态链接器**，而不是直接运行程序

OS在加载完动态链接器后，将控制权交给动态链接器入口地址，然后进行一系列初始化，开始对可执行文件进行动态链接。

### 动态链接相关符号表

- 动态链接器（.interp)
- 链接所需要的信息：dynamic段
- 动态符号表（.dynsyms)
- 动态链接重定位表
- 过程链接表（PLT）
- 全局偏移表（GOT)

#### 过程链接表（PLT）

使用.plt后缀，内容是一个跳转指令，跳到GOT对应的项

过程链接表无法单独工作，是和GOT相关联

#### 全局偏移表

got表

- 每个引用外部模块定义的符号在GOT表中都有相应的条目
- .got： 编译器将对外引用（绝对地址）的符号全部分离出来放到该表中

#### .interp段

指明动态链接器的路径

通过`objdump -s`可以看到`.interp`的内容,里面保存一个字符串，就是可执行文件需要的链接器的路径

```bash
root@sinserver:~/study/link# objdump -s ab | grep -i -A 5 .interp
Contents of section .interp:
 0318 2f6c6962 36342f6c 642d6c69 6e75782d  /lib64/ld-linux-
 0328 7838362d 36342e73 6f2e3200           x86-64.so.2.   
```

通过下面命令也可以找到对应动态库的路径

```bash
root@sinserver:~/study/link# readelf -l ab | grep interpreter
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

#### .dynamic段

这个段里保存了动态链接器所需要的基本信息，比如依赖哪些共享对象、动态链接符号表的位置、动态链接重定位表的位置、共享对象初始化代码的地址等。。

通过`readelf -d`可以查看到`.dynamic`段的内容

显示的内容可以看成动态链接下的ELF文件的文件头

```bash
root@sinserver:~/study/link# readelf -d ab 

Dynamic section at offset 0x2dc8 contains 27 entries:
  Tag        Type                         Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 0x000000000000000c (INIT)               0x1000
 0x000000000000000d (FINI)               0x11b4
 0x0000000000000019 (INIT_ARRAY)         0x3db8
 0x000000000000001b (INIT_ARRAYSZ)       8 (bytes)
 0x000000000000001a (FINI_ARRAY)         0x3dc0
 0x000000000000001c (FINI_ARRAYSZ)       8 (bytes)
 0x000000006ffffef5 (GNU_HASH)           0x3b0
 0x0000000000000005 (STRTAB)             0x480
 0x0000000000000006 (SYMTAB)             0x3d8
 0x000000000000000a (STRSZ)              143 (bytes)
 0x000000000000000b (SYMENT)             24 (bytes)
 0x0000000000000015 (DEBUG)              0x0
 0x0000000000000003 (PLTGOT)             0x3fb8
 0x0000000000000002 (PLTRELSZ)           24 (bytes)
 0x0000000000000014 (PLTREL)             RELA
 0x0000000000000017 (JMPREL)             0x610
 0x0000000000000007 (RELA)               0x550
 0x0000000000000008 (RELASZ)             192 (bytes)
 0x0000000000000009 (RELAENT)            24 (bytes)
 0x000000000000001e (FLAGS)              BIND_NOW
 0x000000006ffffffb (FLAGS_1)            Flags: NOW PIE
 0x000000006ffffffe (VERNEED)            0x520
 0x000000006fffffff (VERNEEDNUM)         1
 0x000000006ffffff0 (VERSYM)             0x510
 0x000000006ffffff9 (RELACOUNT)          3
 0x0000000000000000 (NULL)               0x0
```

通过`ldd`命令也可以查看一个程序主模块依赖哪些共享库

```bash
root@sinserver:~/study/link# ldd ab 
        linux-vdso.so.1 (0x00007ffcafe13000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007dfbc1800000)
        /lib64/ld-linux-x86-64.so.2 (0x00007dfbc1b28000)
```

#### 动态符号表

在静态链接中，有一个专门的表叫`.symtab`（Symbol Table），保存了所有关于该目标文件的符号的定义和引用。动态链接的符号也类似，为了表示动态链接这些模块之间的符号导入导出关系，有一个段名叫做`.dynsym`，但它只保存了与动态链接相关的符号，对于那些模块内部的符号，比如模块私有变量则不保存。

与`.symtab`类似，动态符号表也需要一些辅助的表，比如保存符号名的，静态链接使用`.strtab`,动态链接符号表是`.dynstr`；由于在动态链接下，我们需要在程序运行时查找符号，为了加快符号查找过程，还需要辅助的符号哈希表(`.hash`）。通过`readelf -sD` 查看对应的动态符号表及哈希表

```bash
root@sinserver:~/study/link# readelf -sD ab 

Symbol table for image contains 7 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND _[...]@GLIBC_2.34 (2)
     2: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterT[...]
     3: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND [...]@GLIBC_2.2.5 (3)
     4: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     5: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMC[...]
     6: 0000000000000000     0 FUNC    WEAK   DEFAULT  UND [...]@GLIBC_2.2.5 (3)
```

#### 动态链接重定位表

动态链接文件中，重定位表分别叫做 `rel.dyn`：对数据引用的修正，`rel.plt`对函数引用的修正，修正的位置位于`.got.plt`，通过`readelf -r`可以查看一个动态链接文件的重定位表

```bash
root@sinserver:~/study/link# readelf -r ab 

Relocation section '.rela.dyn' at offset 0x550 contains 8 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000003db8  000000000008 R_X86_64_RELATIVE                    1140
000000003dc0  000000000008 R_X86_64_RELATIVE                    1100
000000004008  000000000008 R_X86_64_RELATIVE                    4008
000000003fd8  000100000006 R_X86_64_GLOB_DAT 0000000000000000 __libc_start_main@GLIBC_2.34 + 0
000000003fe0  000200000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_deregisterTM[...] + 0
000000003fe8  000400000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0
000000003ff0  000500000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_registerTMCl[...] + 0
000000003ff8  000600000006 R_X86_64_GLOB_DAT 0000000000000000 __cxa_finalize@GLIBC_2.2.5 + 0

Relocation section '.rela.plt' at offset 0x610 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000003fd0  000300000007 R_X86_64_JUMP_SLO 0000000000000000 printf@GLIBC_2.2.5 + 0
```

### 动态链接的步骤和实现

首先会先启动动态链接器，然后装载所有需要的共享对象，最后进行重定位和初始化

#### 动态链接器自举

动态链接器的入口地址就是自举代码的入口，OS将控制权交给动态链接器时，自举代码开始执行，会首先找到自己的GOT，而GOT第一个入口保存着.dynamic段的偏移地址，通过这里的信息，可以获得动态链接器本身的重定位表和符号表，从而将动态链接器本身的重定位入口进行重定位，这时候开始可以使用自己的全局变量和静态变量

#### 装载共享对象

动态链接器将可执行文件和链接器本身的符号表合并到**全局符号表**中，然后将可执行文件中依赖的共享对象的名字放入到一个装载集合中，从集合中取一个所需要共享对象的名字，将相应的文件打开后读取相应的ELF文件头和.dynamic段，然后将相应的代码段和数据段映射到进程空间中，如果ELF共享对象还依赖其他的共享对象，再这样递归进行装载。

#### 重定位和初始化

链接器开始重新遍历可执行文件和每个共享对象的重定位表，将它们的GOT/PLT中的每个需要重定位的位置进行修正。

## Linux下的共享库

共享库的路径
三个主要的路径：
/lib：存放系统最关键和基础的共享库
/usr/lib: 非系统运行所需要的关键性共享库，开发时可能会用到的静态库、目标文件
/usr/local/lib: 主要存放第三方应用程序所需要的共享库

### end
