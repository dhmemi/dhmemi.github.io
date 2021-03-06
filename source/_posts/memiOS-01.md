---
title: 写一个操作系统（一）——预备知识:CPU
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

&emsp;&emsp;需要注意的是，从80386开始，CPU还加入了四个于任务无关的全局状态寄存器（或称控制寄存器），他们是CR0、CR1、CR2、CR3。至于他们的作用，将会在后面讲到。
这些寄存器关系如下图：
![图片挂掉了！！！](/images/memios_01_reg.png)
&emsp;&emsp;80486以及后面的奔腾单核系列相较于80386，基本体系结构也没有大的变化，只是速度有了提升和一些其他小的改进。总体而言，intel系列的CPU进化路线如下图：
![图片挂掉了！！！](/images/memios_01_cpu.png)
## 实模式与保护模式
### 实模式
&emsp;&emsp;所谓实模式，就是指令寻址与物理地址一致。这就意味着指令可以接触到所有物理上可寻址的内存单元，也只能在物理可寻址范围寻址。在8086CPU上，由于地址总线位宽为20位，所以可寻址范围为1M。但由于寄存器只有16位，所以寻址时无法靠单个寄存器直接寻址。为了解决这一问题，intel在寻址时采用了两个16位寄存器进行分段寻址，第一个寄存器成为段选择器（Selector），保存地址的高16位，只能保存在段寄存器中；第二个寄存器称为偏移地址（Offset），保存地址相对于高16位对应整地址的偏移值，寻址时用 `Selecotr * 16 + Offset`得到物理地址，在汇编语言中通常以`[Selector:Offset]`的格式表示。例如：
``` math
[0x047C:0x0048] = 0x047C*16 + 0x0048
                = 0x047C0 + 0x0048
                = 0x04808
```
&emsp;&emsp;实模式的缺点是很明显的。首先，在一个固定的段选择器中，受限于偏移寄存器的位数，程序最大线性寻址空间只有64k，设使得我们难以编写超过64k的程序；其次，每个物理地址的寻址方式不独立，比如：
``` math
0x04808 = [0x047C:0x0048]
        = [0x047D:0x0038]
        = [0x047E:0x0028]
```
这也使得内存更加难以管理。
&emsp;&emsp;值得注意的是，在80286上，地址总线被拓宽为24位，这使得80286的寻址空间达到了16M，尽管现在看来，16M依然很小，但在当时的环境中，这样“巨大的”内存空间已经完全超出了单个程序所需要的内存空间，正是因此，[_多道程序_](https://zh.wikipedia.org/wiki/%E5%A4%9A%E4%BB%BB%E5%8A%A1%E5%A4%84%E7%90%86)开始得到真正的发展。保护模式也从此起步。
### 保护模式<small>——copy from </small>[<small>wikipedia</small>](https://zh.wikipedia.org/wiki/%E4%BF%9D%E8%AD%B7%E6%A8%A1%E5%BC%8F#.E4.BB.8E386.E5.BC.80.E5.A7.8B.E7.9A.84IA32.E4.BF.9D.E6.8A.A4.E6.A8.A1.E5.BC.8F.E5.AF.BB.E5.9D.80)
>&emsp;&emsp;在16位保护模式中，CPU仍然使用两个16位寄存器寻址，偏移地址（Offset）的含义也基本没有变化，不同的是，段选择器（Selecotr）的含义完全不同于实模式中的含义，在实模式中，段选择器的值对应了物理地址中的段起始位置；而在保护模式下，由于采用了[虚拟内存技术](https://zh.wikipedia.org/wiki/%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98)，段选择器保存的数据不再是内存物理地址，而是逻辑索引和一些权限信息。其中高13位指向描述符表（descriptor table）的条目；最低的两位数据定义了请求的权限，从0到3，0是最高权限，3是最低权限；剩下的一位表示是使用全局描述符表（GDT）还是局部描述符表（LDT）。描述符表的条目为8字节长，包括24位长的段起始物理地址、16位长的段长（因此段的长度范围从1 B到216 B，即不超过64 KiB）。每次内存操作所要访问的物理地址为描述符表相应条目给出的24位段起始物理地址再加上16位的偏移量。
![图片挂掉了！！！](/images/memios_01_pmodel.png)
&emsp;&emsp;由于偏移地址依旧只有16位，286保护模式下的应用程序能访问的内存线性地址空间仍为64 KB，依旧非常有限。所以程序员编写使用大内存的应用程序时还是必须使用远指针、近指针，相当繁琐。这影响了286保护模式的推广使用。

&emsp;&emsp;到了80386时代，寄存器变为32位，相应的，段选择器和偏移地址寄存器也变成了32位，于是线性可寻址空间变成了4GB，到此，32位机时代Linux和Windows才开始登上历史舞台。
&emsp;&emsp;在32位保护模式下，虚拟存储技术得到了进一步发展。由于在32位机中，线性寻址空间达到了惊人的4GB，但计算机的物理内存也只有4GB甚至不足4GB。为了更加有效的利用有限的物理内存，一个有效的方法就是只将尽可能小的程序片段放入物理内存中，这就要求内存分段的粒度更小一些，至少如果直接继承以前的分段方法，使用4GB的线性寻址空间肯定是太大了。所以人们提出了分页的方法。在分页方法中，以前的`Offset`寄存器被分为3段，其中高10位的值对应页目录索引，中间10位的值对应页表索引，低12位的值对应线性偏移地址。而页目录的的起始地址则保存在前面提到的CR3控制寄存器中，于是寻址方式变成了下图所示：
![图片挂掉了！！！](https://upload.wikimedia.org/wikipedia/commons/thumb/f/f2/080810-protected-386-paging.svg/1230px-080810-protected-386-paging.svg.png)
这样，内存的最小粒度就变成了4KiB。
### 控制寄存器
&emsp;&emsp;现在，我们回过头去讲讲前面提到的四个控制寄存器（CR0、CR1、CR2、CR3）。之所以现在才来讲这四个控制寄存器，是因为，除_CR1作为保留寄存器暂不被使用_外，其他三个控制寄存器都与分页机制有紧密的关系。而且在64位机中它们也和分页机制一起被继承了下来。四个控制寄存器如下图：
![图片挂掉了！！！](http://img.my.csdn.net/uploads/201212/02/1354429752_9982.png)
#### CR0中的保护控制位 
&emsp;&emsp;控制寄存器CR0中的位0用PE标记，位31用PG标记，这两个位控制分段和分页管理机制的操作，所以把它们称为保护控制位。PE控制分段管理机制。PE=0，处理器运行于实模式；PE=1，处理器运行于保护方式。PG控制分页管理机制。PG=0，禁用分页管理机制，此时分段管理机制产生的线性地址直接作为物理地址使用；PG=1，启用分页管理机制，此时线性地址经分页管理机制转换位物理地址。由此可知，如果要启用分页机制，那么PE和PG标志都要置位，如果PE=0而PG=1，将会产生严重异常。需要注意的是，PG位的改变将使系统启用或禁用分页机制，因而只有当所执行的程序的代码和至少有一部分数据在线性地址空间和物理地址空间具有相同的地址的情况下，才能改变PG位。
#### CR0协处理器控制位 
&emsp;&emsp;控制寄存器CR0中的位1—位4分别标记为MP(算术存在位)、EM(模拟位，用于选择与协处理器进行通信所使用的协议，即指明系统中是用的是80386还是80286协处理器)、TS(任务切换位) 和ET(扩展类型位)，它们控制浮点协处理器的操作。当处理器复位时，ET位被初始化，以指示系统中数字协处理器的类型。如果系统中存在 80387协处理器，那么ET位置1；如果系统中存在80287协处理器或者不存在协处理器，那么ET位清0。EM位控制浮点指令的执行是用软件模拟，还是由硬件执行。EM=0时，硬件控制浮点指令传送到协处理器；EM=1时，浮点指令由软件模拟。TS 位用于加快任务的切换，通过在必要时才进行协处理器切换的方法实现这一目的。每当进行任务切换时，处理器把TS置1。TS=1时，浮点指令将产生设备不可用(DNA)异常。 MP位控制WAIT指令在TS=1时，是否产生DNA异常。MP=1和TS=1时，WAIT产生异常；MP=0时，WAIT指令忽略 TS条件，不产生异常。
#### CR2和CR3 
&emsp;&emsp;控制寄存器CR2和CR3由分页管理机制使用。  
&emsp;&emsp;CR2用于发生页异常时报告出错信息。当发生页异常时，处理器把引起页异常的线性地址保存在CR2中。操作系统中的页异常处理程序可以检查CR2的内容，从而查出线性地址空间中的哪一页引起本次异常。
&emsp;&emsp;CR3用于保存页目录表页面的物理地址，因此被称为PDBR。由于目录是页对齐的，所以仅高20位有效，低12 位保留供更加高级的处理器使用。向CR3中装入一个新值时，低12位必须为0；但从 CR3中取值时，低12位被忽略。每当用MOV指令重置CR3的值时，会导致分页机制高速缓冲区的内容无效，用此方法，可以在启用分页机制之前，即把PG 位置1之前，预先刷新分页机制的高速缓存。CR3寄存器即使在CR0寄存器的PG位或PE位为0时也可装入，如在实模式下也可设置CR3，以便进行分页机制的初始化。在任务切换时，CR3要被改变，但是如果新任务中CR3的值与原任务中CR3的值相同，那么处理器不刷新分页高速缓存，以便当任务共享页表时有较快的执行速度。 



