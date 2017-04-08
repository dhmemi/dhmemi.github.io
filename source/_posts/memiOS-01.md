---
title: 写一个操作系统（一）——预备知识：CUP
date: 2017-04-08 16:13:41
tags:
    - CPU
    - 汇编
categories:
    - 操作系统
---
# 引言
&emsp;&emsp;在正式开始写我们的操作系统之前，有必要了解一些预备知识，例如一些与硬件有关的知识、汇编、CUP等。这一篇博客主要讲一讲CPU。
# 80x86家族
&emsp;&emsp;在CPU领域，最具代表性的就是intel的80x86系列，这一系列CPU包含了从8位到32位的CPU,几乎见证了到目前为止整个计算机发展的历史，也是现在大多数系统、汇编教程的常用平台，在这里也不例外。所以我们有必要了解一下80x86系列CPU。
## 迭代与继承
### 8086
&emsp;&emsp;8086是intel进入16位CPU的标志，8086上提供了共14个寄存器，其名称与功能如下：

|寄存器|         全称                      |             通用                 |   专用     |
|:---:|:-------------------------------- |:--------------------------------:| :-------  |
| AX=AH+AL  | Accumulator<br>累加寄存器    |![](/images/memios_01_sml.png)    |_乘法指令(Mul)_:8x8->默认AL保存第一个乘数，AX保存结果<br>&emsp;&emsp;16x16->默认AX保存第一个乘数，AX保存结果的低16位<br>_除法指令(Div)_:16/8->AX存放被除数， AL保存商，AH保存余数<br>&emsp;&emsp;32/16-> AX存放被除数低16位， AX保存商|
| BX=AH+BL  | Base<br>基址寄存器           |![](/images/memios_01_sml.png)    |用作寻址时的偏移地址 |
| CX=CH+CL  | Counter<br>计数器寄存器      |![](/images/memios_01_sml.png)    |用作循环指令(loop)的计数器，每次自动减1，并比较是否大于0，若否，则跳出循环 |
| DX=DH+DL  | Data<br>数据寄存器           |![](/images/memios_01_sml.png)    |乘除法中用作32位数的高16位，用做保存余数|
| SI  | Source Index<br>源变址寄存器       |![](/images/memios_01_sml.png)    |源变址寄存器 |
| DI  | Destination Index<br>目的变址寄存器 |![](/images/memios_01_sml.png)   |目的变址寄存器|
| BP  | Base Pointer<br>基指针寄存器       |![](/images/memios_01_sml.png)    |单使用[BP]，则代表段地址为SS，偏移量为BP寄存器中的值的内存单元|
| SP  | Stack Pointer<br>堆栈指针寄存器    |![](/images/memios_01_sml.png)    |栈指针寄存器，指向调用栈当前位置|
| CS  | Code Segment<br>代码段寄存器       |![](/images/memios_01_cry.png)    |代码段寄存器，指向程序代码段的起始地址 |
| DS  | Data Segment<br>数据段寄存器       |![](/images/memios_01_cry.png)    |指向数据段起始地址|
| SS  | Stack Segment<br>堆栈段寄存器      |![](/images/memios_01_cry.png)    | |
| ES  | Extra Segment<br>附加段寄存器      |![](/images/memios_01_cry.png)    | |
| IP  | Instruction Pointer<br>指令指针寄存器 |![](/images/memios_01_cry.png)  |指向当前代码段中下一条指令的地址|
|FLAGS|flags<br>标志寄存器                 |![](/images/memios_01_cry.png)    |保存一些CPU状态的标志|
### 80x86
&emsp;&emsp;相较于8086，80186和80286在指令体系上并没有什么大的变化，只是增加了几条指令，拓宽了地址总线从而增加了寻址空间而已。
&emsp;&emsp;80386则有了较大的改变，因为从80386开始，intel进入了32位时代，除段寄存器外的寄存器都被扩展为32位，同时增加了两个16位段寄存器FS、GS，为了兼容16位，这些32位寄存器被分成了两部分，只看每个寄存器的低16位，则与8086完全相同，而高16位部分无法直接使用，只能和低16位一起当作32位使用。这些32位寄存器的名称也继承了16位时代的名称，不同的只是在前面加了个E(extended)，于是就有了这些32位寄存器：
> EAX, EBX, ECX, EDX, ESI, EDI, EBP, ESP, CS, DS, SS, ES, FS, GS, EIP, EFLAGS

这些寄存器关系如下图：
![图片挂掉了！！！](/images/memios_01_reg.png)
&emsp;&emsp;80486以及后面的奔腾单核系列相较于80386，基本体系结构也没有大的变化，只是速度有了提升和一些其他小的改进。总体而言，intel系列的CPU进化路线如下图：
![图片挂掉了！！！](/images/memios_01_cpu.png)

