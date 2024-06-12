# chap0 前言

参考《一个64位操作系统的设计与实现》，这本书主要是从0实现一个操作系统内核，操作系统主要参考`linux`的一些实现方式。

简单地看了下目录，认为能学到的东西非常丰富，包括不限于

1.intel白皮书硬件架构认识 √

2.操作系统的理解和设计 √

3.linux系统的使用，linux内核的理解 √

4.makefile使用 ×

5.文件系统理解 √

**环境**

> 1.ubuntu20.04
>
> 2.GNU 11.4.0
>
> 3..bochs 2.7
>
> 4.nasm 2.15.05
>
> 5.windows11 23h2

**主要记录**

> 1.学习心得
>
> 2.环境搭建和实现时候的坑

**尽量做到先自己写出来代码,写不出来看随书源码,实在不行最后在抄上去（最后发现基本没有写过一行代码...幸亏随书源码一点不少，不然以这本书的质量，估计随时弃书：(**

## 基础概念

主要是这本书的前两章，下面快速过一下OS的基础概念.

### 操作系统

**操作系统**根本目的是隔绝硬件层，是方便用户和硬件设备交互的**软件**，从功能上来说，它对下兼容不同的硬件架构体系，对上为用户提供相关系统api接口。

### 操作系统构成

现代OS的构成主要是

![image-20240303164413633](%E7%AC%94%E8%AE%B0.assets/image-20240303164413633.png)

下面就不同的模块进行介绍

- **bootloader**

其实是`boot`+`loader`,众所周知，如果主板是bios启动的话,**最终会以实模式跑到`Ip:0x7c00`,`cs:0`的地方去执行**,这个地方的512字节是选择引导的物理介质(比如U盘,硬盘,软盘)的第一个磁道第一个扇区。

bios程序会设置好bios的中断处理程序以及相关的内存映射，保证正常执行。

boot就是那一个扇区,512byte(**B**),因此其实boot很小，能做到的功能有限，boot基本上会马上调用loader，loader检查完无误，传递参数，控制权交给内核。

- **内存管理**

OS的一个必须的重要组成模块，**通过各种算法+数据结构记录维护已使用未使用的内存，有效利用物理内存/虚拟内存**，防止内存碎片化。

- **异常/中断处理**

产生错误时候或者外围设备中断的时候，os必须响应，这个执行速度影响OS的整体执行速度，因为会大量触发。

**中断分为前/后部分,前面快速响应，后部打开中断，比如数据解析，驱动修改状态，在这个地方完成。**

事实上,linux中断后半部分专门放在一个进程去做，windows似乎DPC就是这个概念。

- **进程管理**

分为进程调用+进程通信，都很重要。

- **设备驱动**

一个外围设备一般会有很多驱动，因为只有生产厂商才知道这个外围设备的状态寄存器的调整等等，win/linux都提供了稳定的设备驱动模型，让程序员(厂商)写驱动，方便响应不同厂商的外围设备，**即插即用，适时加载到内核**。(**现代而言，驱动已经不是直接使用in/out来操作物理设备，而是通过驱动程序与USB设备的通信通常采用USB请求块（URB），这是一种特定于USB的数据结构，用于封装发送到USB设备的命令或从USB设备接收的数据。**)

- **文件系统**

把硬盘组成一个方便管理的结构化单元，比如FAT32,还是linux的EXT,都是为了方便操作系统使用。

## 环境搭建

环境搭建方便,这本书的坑还挺多的。

### bochs编译

推荐不编译,当然编译也行，编译bochs会有很多坑

因为bochs官网的源码包缺符号，编译好了根本就不能用..

[解决方法](https://blog.csdn.net/Asdfffy/article/details/84035543?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-84035543-blog-115451130.235%5Ev43%5Epc_blog_bottom_relevance_base3&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-84035543-blog-115451130.235%5Ev43%5Epc_blog_bottom_relevance_base3)

而这里推荐直接

```bash
sudo apt-get install bochs bochs-x
```

bochsx是bochs依赖的,因为bochs有窗口，这玩意必须。

还得安装一堆依赖项

```bash
sudo apt-get install build-essential 
sudo apt-get install xorg-dev 
sudo apt-get install bison 
sudo apt-get install g++ 
```

然后

```bash
sudo startx
```

把windows-x启动

就可以正常使用bochs了;

### nasm

```bash
sudo apt-get install nasm
```

直接安装,然后写一个`.asm`,bios实模式环境下的在窗口上显示东西的代码

这个代码就在随书源码3.1的`boot.asm`

使用命令

```bash
nasm boot.asm -o boot.bin
```

### boot.img

编译好,bochs环境搭建完成后,使用bximage创建一个软盘,直接shell输入`bximage`按照他的提示创建一个`1.44MB`的软盘即可

然后使用命令把`boot.bin`写入这个创建的软盘的第一个扇区的第一个磁道;

```bash
dd if=boot.bin of=boot.img bs=512 count=1 conv=notrunc
```

接着直接复制随书附带的`bochsrc`这个配置文件,这个文件内容入下

![image-20240303171954828](%E7%AC%94%E8%AE%B0.assets/image-20240303171954828.png)

其实就是配置如何启动,启动文件在哪,cpu的属性，开启的功能。

一切就绪,直接弹出bochs的窗口,成功

```bash
bochs -f ./bochsrc
```

![image-20240303172110458](%E7%AC%94%E8%AE%B0.assets/image-20240303172110458.png)

## 前置知识

### GNU的内嵌汇编

下面快速地介绍一下GNU的内联汇编,这玩意非常复杂,比如MSVC复杂多了，考虑的东西也更多，因此可以兼容x64.

GNU的内联汇编有四个部分

> 必要前缀 指令 : 输出 : 输入 : 破坏

必要前缀不算,前面必须写`__asm__ __volatile__`

- 指令 就是汇编指令,但是书写格式是`AT&T`,比如 `movl $0X10,%%eax`,可以多条指令,但是每条指令必须加`\n\t`或者`;`比如`__asm__ __volatile__ ("nop \n\t mov $0x10,%%eax \n\t")`

- 输出 就是可以把一些信息输出,通常是`"输出操作约束"(输出表达式)`,比如`"eax"(tmp)`,代表内联结束后把eax的值输出到tmp这个变量
- 输入 就是输入到寄存器,也是和输出同样的构成,比如`"eax"(tmp)`,就是把tmp输入到eax寄存器;
- 损坏 就是指令执行的时候,损坏的寄存器,内存,标志寄存器,格式是"损坏描述",..."损坏描述",比如下面这条内联汇编`__asm__ __volatile__ "movl %0,%%ecx"::"a"(tmp):"ecx"`代表ecx被改变了

剩下的还有一些其他的小注意事项,可以使用缩写,a==rax/eax/ax/al,会自动根据数据宽度匹配

`__asm__ __volatile__ "movl %0,%%ecx"::"a"(tmp):"ecx"`的`%0`代表占位符,他表示第一个输入的数,这里其实就是`eax`,因为eax被tmp输入了;

总之这玩意比较复杂...后续在深入学习吧

# chap1 BootLoader

前面学到了,BootLoader=`Boot`+`Loader`;

作用有些许不同,简而言之Boot其实最多不会超过`512B`,Loader可以比较大，一般是检查硬件自检+传递检查的参数给内核，跳转到内核去执行。

这一章主要是两个知识点,`Loader`怎么写和文件系统`FAT12`的学习。

## 文件系统

书中用到的文件系统是`FAT12`,FAT家族的文件系统都是比较简单的。windows用得多的是`NTFS`,linux则是`EXT`,他们更加现代更加快速。

**文件系统**

> 文件系统（File System）是计算机中用于存储、组织、访问和管理文件的一种数据结构。它定义了文件、目录和元数据等的组织方式，使得操作系统能够高效地管理存储设备上的数据。文件系统可以应用于硬盘、闪存、光盘等各种存储设备上。

简而言之,文件系统很容易会被误解是操作系统的一部分。实际上，他是独立于操作系统的，反而是操作系统识别出来了一个**存储介质设备**的文件系统，根据这个文件系统的组织，来进行识别不同的文件夹文件，然后按照这个文件系统去写入不同的磁道，磁盘，扇面等等...比如，windows的文件系统驱动就是专门写入数据的,**理解就是ntfs.sys知道磁盘怎么组织的，但是磁盘驱动只能接受irp，irp中有特殊的结构组成，来去写哪个磁道，所以要靠ntfs.sys，而disk.sys则真正的去给主板上的磁盘/软盘控制器发送SATA ATA SCSI等命令去读写磁盘**，他最终根据这个文件系统的类型，找到要写入的扇区，调用对应磁盘驱动来写入内容。

我们在使用`格式化`这个功能的时候，其实就已经把文件系统写入到了这个存储介质中了;

### FAT12的构成

FATXX家族全程`File Allocation Table`,也就是文件分配表的文件系统，后面的12 16 32其实就是`FAT`表项的位数。

![image-20240307192754353](%E7%AC%94%E8%AE%B0.assets/image-20240307192754353.png)

下面就是一个FAT12的文件系统的构成。

我们要知道,FAT文件系统是以**簇**为基本读写单位的，就是扇面的整数倍。

文章的簇用的就是一个扇面，这样更加简单。

### FAT

`FAT`就是这个文件系统的核心部分，**他的组成是一个12位的数组**(FATXX就是XX位的数组),**我们知道，文件肯定有可能跨扇面或者是跨簇的**，一个文件开始的簇定位了，那么如何定位接下来的簇或者扇面在那个地方呢?

**FAT就可以定位，FAT采取了类似单链表的形式，比如簇 3,FAT[3]的值是16,那么也就是当前这个文件的下一个簇内容位于簇16,然后如此反复的找，直到找到`0xFFF`这样的无效簇，代表这次寻找文件结束**

==其实不难发现,FATXX,如果XX越大，一个文件就可以更大==	

![image-20240307193135674](%E7%AC%94%E8%AE%B0.assets/image-20240307193135674.png)

**值得一提是的,FAT[0]和FAT[1]不作为数据区的索引使用，也就是簇2是第一个开始的有效簇。**

### 根目录区

本质上,根目录区也算是数据区的一部分，只不过**根目录区**存的只有文件信息。

根目录是由**目录项**的**一棵树**构成的，想想也是，毕竟树查找的更快。

![image-20240307193851564](%E7%AC%94%E8%AE%B0.assets/image-20240307193851564.png)

文章中的`boot.asm`就实现了如下功能

1. 根据簇索引获取下一个簇
2. 封装DOS的int10,更方便的读取磁盘,只穿入第几个**逻辑扇区**就行
3. 搜索`loader.bin`这个文件名(在磁盘使用解析FAT12的形式进行搜索)
4. 搜索找到之后,把loader.bin加载到内存为`0x10000`的地址,然后控制权转移交给loader。

## Loader

### 虚拟软盘挂载

先实现一个简单的`Loader`程序,只打印`Start Loader...`

这个时候我们的软盘一旦使用`dd`指令把`boot.bin`写入`boot.img`,实际上本来就是`FAT12`的文件系统形式。(可以理解为格式化成了FAT12)

因此这个时候软盘有了文件系统，就可以直接使用`mount`进行挂载到linux的文件系统中。然后直接使用cp指令,把编译好的`loader.bin`拷贝过去就行。

因为我们的`boot.asm`解析了`FAT12`文件系统,并且专门去查找了叫做`loader.bin`位于哪里，放在内存中，jmp到`loader`。

具体指令比较简单,我们直接把这个软盘挂载到`/media/`这个目录下，然后直接拷贝就行了。

```shell
ls
bin  boot  cdrom  dev  etc  home  lib  lib32  lib64  libx32  lost+found  media  mnt  opt  proc  root  run  sbin  snap  srv  swapfile  sys  tmp  usr  var

sudo mount boot.img /media/ -t vfat -o loop
cp loader.bin /media/
```

这个时候loader.bin放到了以`FAT12`文件系统组织的软盘。直接启动bochs，可以观察到成功。

打印出来了`Start Loader`,同时`IP`==`0x1xxxxx`,不在是`7e00`这个地址也证明从`Boot`逃离出来了。

![image-20240307195022197](%E7%AC%94%E8%AE%B0.assets/image-20240307195022197.png)

### loader原理

前文提到,loader的原理是检测硬件信息，模式切换，向内核传递数据。事实上，只是老的bios启动才会这样，如果是uefi的话，bootloader可以直接是一个PE格式的文件，而且CR3,GDT啥的都是给你设置好的，启动就是x64。

比如微软的bootmgr.efi，专门放在一个专属分区，有着专门的文件系统，当然，这玩意是有签名的，所以别想着控制os引导，`blacklotus`就是利用漏洞找到一个没有吊销的签名实现的`bootkit`。

![image-20240314090038636](%E7%AC%94%E8%AE%B0.assets/image-20240314090038636.png)



而loader一般就进入ia32e模式了，文中对于这个部分主要是讲述了一下16(实模式)->32(保护模式),32(保护模式)->64(IA-32E)如何进行转变。

具体文件在文中的`3-4的loader.asm`，文章中的loader.asm很大，有600行汇编，涉及大量的bios和IA32E和保护模式的基础，说实话作者并不是特别厚道，这本书没有相关基础完全读不懂。下面主要是分析一下loader.asm的源码

- **调用int15查看物理内存的情况**

这个di就是保存的地方，

```assembly
Label_Get_Mem_Struct:

	mov	eax,	0x0E820
	mov	ecx,	20
	mov	edx,	0x534D4150
	int	15h
	jc	Label_Get_Mem_Fail
	add	di,	20

	cmp	ebx,	0
	jne	Label_Get_Mem_Struct
	jmp	Label_Get_Mem_OK
```

- **设置SVGA芯片，设置显示模式**

```assembly
;=======	set the SVGA mode(VESA VBE)

	mov	ax,	4F02h
	mov	bx,	4180h	;========================mode : 0x180 or 0x143
	int 	10h
```

可以简单地理解为设置分辨率，总之都是bios中断程序自带的。

- **实模式->保护模式**

实模式->保护模式一般是固定的，这里不再涉及分页了，因为不开启分页机制。这样当然是可以的，控制分页的是Cr0::PG是控制开启的，可以开启Cr0::PE但是不开启Cr0::PG(这里不开启是为了后面无缝切换IA32E准备)。

```assembly
;=======	init IDT GDT goto protect mode 

	cli			;======close interrupt

	db	0x66
	lgdt	[GdtPtr]

;	db	0x66
;	lidt	[IDT_POINTER]

	mov	eax,	cr0
	or	eax,	1
	mov	cr0,	eax	

	jmp	dword SelectorCode32:GO_TO_TMP_Protect
```

关中断(因为没考虑IDT,所以保证不要触发异常和中断)，然后`lgdt`加载全局描述符表，紧接着写cr0::PE,然后jmp cs:xx,这个跨段跳是为了进入保护模式，也是很重要的一步，这是硬件规定的，这是为了刷新流水线。这里的GDT是提前构造好的，同样的，代码段的段选择子也是。

- **保护模式->IA32E模式**

这里可以参考loader.asm的如下代码

首先要确保cpu能够支持long mode

```assembly
support_long_mode:

	mov	eax,	0x80000000
	cpuid
	cmp	eax,	0x80000001
	setnb	al	
	jb	support_long_mode_done
	mov	eax,	0x80000001
	cpuid
	bt	edx,	29
	setc	al
support_long_mode_done:
	
	movzx	eax,	al
	ret
```

`setnb`这类`setcc`可以讲一下，就是`nb(not below)`的把al置位。cpuid id==`80000001`的时候,29位是1,就可以支持IA32E模式。

检测完成之后

```assembly
;=======	init temporary page table 0x90000

	mov	dword	[0x90000],	0x91007
	mov	dword	[0x90800],	0x91007		

	mov	dword	[0x91000],	0x92007

	mov	dword	[0x92000],	0x000083

	mov	dword	[0x92008],	0x200083

	mov	dword	[0x92010],	0x400083

	mov	dword	[0x92018],	0x600083

	mov	dword	[0x92020],	0x800083

	mov	dword	[0x92028],	0xa00083

;=======	load GDTR

	db	0x66
	lgdt	[GdtPtr64]
	mov	ax,	0x10
	mov	ds,	ax
	mov	es,	ax
	mov	fs,	ax
	mov	gs,	ax
	mov	ss,	ax

	mov	esp,	7E00h

;=======	open PAE

	mov	eax,	cr4
	bts	eax,	5
	mov	cr4,	eax

;=======	load	cr3

	mov	eax,	0x90000
	mov	cr3,	eax

;=======	enable long-mode

	mov	ecx,	0C0000080h		;IA32_EFER
	rdmsr

	bts	eax,	8
	wrmsr

;=======	open PE and paging

	mov	eax,	cr0
	bts	eax,	0
	bts	eax,	31
	mov	cr0,	eax

	jmp	SelectorCode64:OffsetOfKernelFile
```

设置页表,然后构造x64的GDT和IDT,进行加载，这里需要打开PAE,来支持IA32E模式的4级分页，同时加载cr3，因为这个时候没开启PG,cr3随便修改。

然后修改寄存器的`IA32_EFER`的LM位，紧接着打开PE位和PG位。然后jmp跨段跳转进入IA32E模式，成功。

# chap2 内核层

## 内核头

文章头使用了内核执行头来作为内核的最前面的小程序，简而言之就是一段特殊的汇编代码。它的作用就是给即将执行的内核**设置好GDT IDT Cr3**,最重要的是设置好cr3;

### 内核头

文章中的内核执行头如下

```assembly
//=======	GDT_Table

.section .data

.globl GDT_Table

GDT_Table:
	.quad	0x0000000000000000			/*0	NULL descriptor		       	00*/
	.quad	0x0020980000000000			/*1	KERNEL	Code	64-bit	Segment	08*/
	.quad	0x0000920000000000			/*2	KERNEL	Data	64-bit	Segment	10*/
	.quad	0x0020f80000000000			/*3	USER	Code	64-bit	Segment	18*/
	.quad	0x0000f20000000000			/*4	USER	Data	64-bit	Segment	20*/
	.quad	0x00cf9a000000ffff			/*5	KERNEL	Code	32-bit	Segment	28*/
	.quad	0x00cf92000000ffff			/*6	KERNEL	Data	32-bit	Segment	30*/
	.fill	10,8,0					/*8 ~ 9	TSS (jmp one segment <7>) in long-mode 128-bit 40*/
GDT_END:

GDT_POINTER:
GDT_LIMIT:	.word	GDT_END - GDT_Table - 1
GDT_BASE:	.quad	GDT_Table

//=======	IDT_Table

.globl IDT_Table

IDT_Table:
	.fill  512,8,0
IDT_END:

IDT_POINTER:
IDT_LIMIT:	.word	IDT_END - IDT_Table - 1
IDT_BASE:	.quad	IDT_Table

//=======	TSS64_Table

.globl	TSS64_Table

TSS64_Table:
	.fill  13,8,0
TSS64_END:

TSS64_POINTER:
TSS64_LIMIT:	.word	TSS64_END - TSS64_Table - 1
TSS64_BASE:	.quad	TSS64_Table

```

上述代码是内核执行头的全局变量，主要是IDT,GDT,TSS的数据结构内容等

然后开始设置GDT和IDT

```assembly
_start:

	mov	$0x10,	%ax
	mov	%ax,	%ds
	mov	%ax,	%es
	mov	%ax,	%fs
	mov	%ax,	%ss
	mov	$0x7E00,	%esp

//=======	load GDTR

	lgdt	GDT_POINTER(%rip)

//=======	load	IDTR

	lidt	IDT_POINTER(%rip)

	mov	$0x10,	%ax
	mov	%ax,	%ds
	mov	%ax,	%es
	mov	%ax,	%fs
	mov	%ax,	%gs
	mov	%ax,	%ss

	movq	$0x7E00,	%rsp
```

这里后面的(%rip)是x64汇编独特的RIP+Disp寻址。

然后开始load cr3

```assembly
__PML4E:

	.quad	0x102007
	.fill	255,8,0
	.quad	0x102007
	.fill	255,8,0

.org	0x2000

__PDPTE:
	
	.quad	0x103003
	.fill	511,8,0

.org	0x3000

__PDE:

	.quad	0x000083	
	.quad	0x200083
	.quad	0x400083
	.quad	0x600083
	.quad	0x800083
	.quad	0xe0000083		/*0x a00000*/
	.quad	0xe0200083
	.quad	0xe0400083
	.quad	0xe0600083		/*0x1000000*/
	.quad	0xe0800083
	.quad	0xe0a00083
	.quad	0xe0c00083
	.quad	0xe0e00083
	.fill	499,8,0

//=======	GDT_Table

.section .data
```

上面是页表结构

不难发现，线性地址0x0-0x100000对应物理地址0x0-0x100000，是1:1映射的。

而从256个PML4E,从0xffff8000'00000000+0xa00000开始,对应物理地址e0000000(**这其实就是映射的显存**);

这样是为了方便页表使用，因为现在bootloader转移到内核，虽然分页了，但是本质上还是1:1使用的物理地址(**详细看前面的bootloader的分页**)。这样至少有了内核的线性地址。

```assembly
//=======	load	cr3

	movq	$0x101000,	%rax
	movq	%rax,		%cr3
	movq	switch_seg(%rip),	%rax
	pushq	$0x08
	pushq	%rax
	lretq

//=======	64-bit mode code

switch_seg:
	.quad	entry64

entry64:
	movq	$0x10,	%rax
	movq	%rax,	%ds
	movq	%rax,	%es
	movq	%rax,	%gs
	movq	%rax,	%ss
	movq	$0xffff800000007E00,	%rsp		/* rsp address */

	movq	go_to_kernel(%rip),	%rax		/* movq address */
	pushq	$0x08
	pushq	%rax
	lretq

go_to_kernel:
	.quad	Start_Kernel

//=======	init page
.align 8
```

然后加载cr3，注意，下面两句，执行完就从底线性地址跨越到高线性地址了。

```assembly
	pushq	%rax
	lretq
```

`Start_Kernel`则是内核的`main.c`里面的函数,是一个死循环。

### makefile

对于这玩意，因为他是汇编，所以编译器不能是gcc，但是必须先预处理一下，使用-E指令,然后使用`as`编译器编译成.o

```assembly
head.o:	head.S
	gcc -E  head.S > head.s
	as --64 -o head.o head.s
```

## 屏幕显示

显存，顾名思义就是显示器的cache，每一个内存就代表屏幕上可以显示色彩的点位。

就具体显存大小，是根据分辨率来的。一个显示点是32位组成的，其中前三个8位分别代表蓝绿红，最后八位保留。

### 显存写入

在bootloader，我们已经将显存设置成了1440*900的分辨率，而显存的位置位于0xe0000000(**详情参考SVGA芯片设置，这个显存都是在这**),之前已经映射到了0xffff80000a00000了;

```C
int *addr = (int *)0xffff800000a00000;
	int i;

	Pos.XResolution = 1440;
	Pos.YResolution = 900;

	Pos.XPosition = 0;
	Pos.YPosition = 0;

	Pos.XCharSize = 8;
	Pos.YCharSize = 16;

	Pos.FB_addr = (int *)0xffff800000a00000;
	Pos.FB_length = (Pos.XResolution * Pos.YResolution * 4);

	for(i = 0 ;i<1440*20;i++)
	{
		*((char *)addr+0)=(char)0x00;
		*((char *)addr+1)=(char)0x00;
		*((char *)addr+2)=(char)0xff;
		*((char *)addr+3)=(char)0x00;	
		addr +=1;	
	}
	for(i = 0 ;i<1440*20;i++)
	{
		*((char *)addr+0)=(char)0x00;
		*((char *)addr+1)=(char)0xff;
		*((char *)addr+2)=(char)0x00;
		*((char *)addr+3)=(char)0x00;	
		addr +=1;	
	}
	for(i = 0 ;i<1440*20;i++)
	{
		*((char *)addr+0)=(char)0xff;
		*((char *)addr+1)=(char)0x00;
		*((char *)addr+2)=(char)0x00;
		*((char *)addr+3)=(char)0x00;	
		addr +=1;	
	}
	for(i = 0 ;i<1440*20;i++)
	{
		*((char *)addr+0)=(char)0xff;
		*((char *)addr+1)=(char)0xff;
		*((char *)addr+2)=(char)0xff;
		*((char *)addr+3)=(char)0x00;	
		addr +=1;	
	}
```

如上述，写入，放到bochs中，最终可以达到如下效果

![image-20240315093215206](%E7%AC%94%E8%AE%B0.assets/image-20240315093215206.png)

### ASCII像素位图

接下来就是自己实现printk(专门在内核使用的printf)。

文章中使用ASCII像素位图来显示字符，具体而言是任何一个ASCII字符可以变成

8*16的像素矩阵，比如

![image-20240315093723990](%E7%AC%94%E8%AE%B0.assets/image-20240315093723990.png)

这个矩阵的数据结构可以抽象成一个16长度的char数组，char本身就有八位，因此就是一个8*16的矩阵

最后，一共256个ascii字符串，可以抽象成

```C
unsigned char font_ascii[256][16]=
{

	/*前面省略一堆*/
	/*	0030	*/
	{0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00},
	{0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00},
	{0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00},
	{0x00,0x10,0x10,0x10,0x10,0x10,0x10,0x10,0x10,0x10,0x00,0x00,0x10,0x10,0x00,0x00},	//33	'!'
	{0x28,0x28,0x28,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00},	//	'"'
	{0x00,0x44,0x44,0x44,0xfe,0x44,0x44,0x44,0x44,0x44,0xfe,0x44,0x44,0x44,0x00,0x00},	//	'#'
	{0x10,0x3a,0x56,0x92,0x92,0x90,0x50,0x38,0x14,0x12,0x92,0x92,0xd4,0xb8,0x10,0x10},	//	'$'
	{0x62,0x92,0x94,0x94,0x68,0x08,0x10,0x10,0x20,0x2c,0x52,0x52,0x92,0x8c,0x00,0x00},	//	'%'
	{0x00,0x70,0x88,0x88,0x88,0x90,0x60,0x47,0xa2,0x92,0x8a,0x84,0x46,0x39,0x00,0x00},	//	'&'
	{0x04,0x08,0x10,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00},	//	'''
```

font_ascii[ascii值]=对应ASCII字符的像素位图。font_ascii[ascii值] [行数]=value,而value如果是1，那么就代表哪一列是字体颜色，如果是0，就代表是背景颜色。

### color_printk

有了这个，就可以实现一个printk了。原文中实现printk非常非常复杂，用了600+行代码，因为要考虑非常复杂的转义字符等...其实主要是`vsprintf`这个函数比较难以实现。

```C
int color_printk(unsigned int FRcolor,unsigned int BKcolor,const char * fmt,...)
{
	int i = 0;
	int count = 0;
	int line = 0;
	va_list args;
	va_start(args, fmt);

	i = vsprintf(buf,fmt, args);

	va_end(args);

	for(count = 0;count < i || line;count++)
	{
		////	add \n \b \t
		if(line > 0)
		{
			count--;
			goto Label_tab;
		}
		if((unsigned char)*(buf + count) == '\n')
		{
			Pos.YPosition++;
			Pos.XPosition = 0;
		}
		else if((unsigned char)*(buf + count) == '\b')
		{
			Pos.XPosition--;
			if(Pos.XPosition < 0)
			{
				Pos.XPosition = (Pos.XResolution / Pos.XCharSize - 1) * Pos.XCharSize;
				Pos.YPosition--;
				if(Pos.YPosition < 0)
					Pos.YPosition = (Pos.YResolution / Pos.YCharSize - 1) * Pos.YCharSize;
			}	
			putchar(Pos.FB_addr , Pos.XResolution , Pos.XPosition * Pos.XCharSize , Pos.YPosition * Pos.YCharSize , FRcolor , BKcolor , ' ');	
		}
		else if((unsigned char)*(buf + count) == '\t')
		{
			line = ((Pos.XPosition + 8) & ~(8 - 1)) - Pos.XPosition;

Label_tab:
			line--;
			putchar(Pos.FB_addr , Pos.XResolution , Pos.XPosition * Pos.XCharSize , Pos.YPosition * Pos.YCharSize , FRcolor , BKcolor , ' ');	
			Pos.XPosition++;
		}
		else
		{
			putchar(Pos.FB_addr , Pos.XResolution , Pos.XPosition * Pos.XCharSize , Pos.YPosition * Pos.YCharSize , FRcolor , BKcolor , (unsigned char)*(buf + count));
			Pos.XPosition++;
		}


		if(Pos.XPosition >= (Pos.XResolution / Pos.XCharSize))
		{
			Pos.YPosition++;
			Pos.XPosition = 0;
		}
		if(Pos.YPosition >= (Pos.YResolution / Pos.YCharSize))
		{
			Pos.YPosition = 0;
		}

	}
	return i;
}

```

简而言之,vsprintf根据arg,把参数转换成字符串,结合`fmt`的结构，插入到buf中。最后是一个有格式的字符串，但是只有`\t \b \n`没有处理，因此循环buf字符串，查找这些字符串，对于该进位的进位，该缩进的缩进。

`putchar`就是打印一个字符，根据ascii码，代码如下

可以看到,有两个循环，16*8,`Pos.Resolution`是分辨率的意思，而`Pos.Position`就是光标所在位置。

`putchar(Pos.FB_addr , Pos.XResolution , Pos.XPosition * Pos.XCharSize , Pos.YPosition * Pos.YCharSize , FRcolor , BKcolor , (unsigned char)*(buf + count));`比如这个调用，其中因为我们的ASCII位图是8*16,所以x的大小就是8,ysize就是16，这是固定的。

然后两个循环，`testval = 0x100;`转换成二进制是`1 0000 0000`,每次循环向右移动一位，当作测试，如果是，那么就写入字体颜色，否则背景颜色。

```C
void putchar(unsigned int * fb,int Xsize,int x,int y,unsigned int FRcolor,unsigned int BKcolor,unsigned char font)
{
	int i = 0,j = 0;
	unsigned int * addr = NULL;
	unsigned char * fontp = NULL;
	int testval = 0;
	fontp = font_ascii[font];

	for(i = 0; i< 16;i++)
	{
		addr = fb + Xsize * ( y + i ) + x;
		testval = 0x100;
		for(j = 0;j < 8;j ++)		
		{
			testval = testval >> 1;
			if(*fontp & testval)
				*addr = FRcolor;
			else
				*addr = BKcolor;
			addr++;
		}
		fontp++;		
	}
}
```

最终的makefile如下

```makefile
all: system
	objcopy -I elf64-x86-64 -S -R ".eh_frame" -R ".comment" -O binary system kernel.bin

system:	head.o main.o printk.o
	ld -b elf64-x86-64 -z muldefs -o system head.o main.o printk.o -T Kernel.lds 

head.o:	head.S
	gcc -E  head.S > head.s
	as --64 -o head.o head.s
	
main.o:	main.c
	gcc  -fno-stack-protector -mcmodel=large -fno-builtin -m64 -c main.c
	
printk.o: printk.c
	gcc   -fno-stack-protector -mcmodel=large -fno-builtin -m64 -c printk.c

clean:
	rm -rf *.o *.s~ *.s *.S~ *.c~ *.h~ system  Makefile~ Kernel.lds~ kernel.bin 
```

这里简单解释一下，首先all依赖system,system依赖那几个o

o依赖c和S，

`ld -b elf64-x86-64 -z muldefs -o system head.o main.o printk.o -T Kernel.lds `这一句是链接器,`kernelk.lds`是连接脚本这个链接脚本可以执行链接时候的不同节点的位置,这样就可以保证内核头位于最前面了,然后这些会生成一个`system`文件,但是后面all的时候，使用`objcopy`就会删除`.eh_frame`和`.comment`两个不必要的节段。最终kernek.bin链接成功，放到boot.img即可运行。

```makefile
OUTPUT_FORMAT("elf64-x86-64","elf64-x86-64","elf64-x86-64")
OUTPUT_ARCH(i386:x86-64)
ENTRY(_start)
SECTIONS
{

	. = 0xffff800000000000 + 0x100000;
	.text :
	{
		_text = .;
		*(.text)

		_etext = .;
	}
	. = ALIGN(8);
	.data :
	{
		_data = .;
		*(.data)
		
		_edata = .;
	}
	.bss :
	{
		_bss = .;
		*(.bss)
		_ebss = .;
	}

	_end = .;
}

```

总之，上面的makefile自定义地生成了kernel.bin,虽然用现代编译器，但是却可以用作操作系统内核文件的生成，离不开上面的ld,objcopy等各种参数。

## 异常

异常一般分为

- fault
- trap
- abort

![image-20240321160350333](%E7%AC%94%E8%AE%B0.assets/image-20240321160350333.png)

intel对于IDT的一些向量是固定的，如上表，32-255才开始是OS自定义的。

我们只需要知道发生异常时候栈空间的变化就行了，去哪里处理其实本质上就是根据IDT来找的

IDT的每一个表项内容是

![image-20240321161157598](%E7%AC%94%E8%AE%B0.assets/image-20240321161157598.png)

**文章中主要就是对这个GDT进行初始化**，具体代码不展示了。

而我们知道异常/中断发生的时候栈会被填进去,也就是比如cs,ss,rip,rsp,rflags,error_code(**有的异常可能有，这个是处理器填写的，为了更详细地表述中断，比如发生#PF，就需要解析这个errcode**)

![image-20240321160939934](%E7%AC%94%E8%AE%B0.assets/image-20240321160939934.png)

而我们知道，栈是有rsp0/rsp3的这个概念的，如果栈的特权级没发生变化/发生变化压入的是有区别的。

当然，如果手动int,是不会填写上面所述的东西的。

之前逆向windows的时候，我们知道，windows发生异常和中断其实都会在线程内核栈的固定位置(**其实切换栈的本质是TSS.Stack0,处理器会根据当前加载的TSS任务描述符来切换栈,当然syscall不会自动切换栈，这是后话了**)维护一个`TrapFrame`,其实这个结构本质上是为了从**中断/异常**恢复出来，文章中也有相同的概念。

不过文章的这个结构实在难绷，用的偏移，实在是太不优雅。

因为发生异常涉及到栈切换,因此必须填写TSS，TSS必须先初始化

```C++
load_TR(8);

set_tss64(0xffff800000007c00, 0xffff800000007c00, 0xffff800000007c00, 0xffff800000007c00, 0xffff800000007c00, 0xffff800000007c00, 0xffff800000007c00, 0xffff800000007c00, 0xffff800000007c00, 0xffff800000007c00);

/*

#define load_TR(n) 							\
do{									\
	__asm__ __volatile__(	"ltr	%%ax"				\
				:					\
				:"a"(n << 3)				\
				:"memory");				\
}while(0)

*/

```

然后设置IDT

```C++
void sys_vector_init()
{
	set_trap_gate(0,1,divide_error);
	set_trap_gate(1,1,debug);
	set_intr_gate(2,1,nmi);
	set_system_gate(3,1,int3);
	set_system_gate(4,1,overflow);
	set_system_gate(5,1,bounds);
	set_trap_gate(6,1,undefined_opcode);
	set_trap_gate(7,1,dev_not_available);
	set_trap_gate(8,1,double_fault);
	set_trap_gate(9,1,coprocessor_segment_overrun);
	set_trap_gate(10,1,invalid_TSS);
	set_trap_gate(11,1,segment_not_present);
	set_trap_gate(12,1,stack_segment_fault);
	set_trap_gate(13,1,general_protection);
	set_trap_gate(14,1,page_fault);
	//15 Intel reserved. Do not use.
	set_trap_gate(16,1,x87_FPU_error);
	set_trap_gate(17,1,alignment_check);
	set_trap_gate(18,1,machine_check);
	set_trap_gate(19,1,SIMD_exception);
	set_trap_gate(20,1,virtualization_exception);

	//set_system_gate(SYSTEM_CALL_VECTOR,7,system_call);

}
```

可以看到，基本上设置的都是陷阱门，除了`nmi`。**但是windows的IDT都是中断门，也就是进入异常自动关闭中断。可能win为了统一，windows会用个KiDispatcherException来进行统一处理**

```assembly
#define _set_gate(gate_selector_addr,attr,ist,code_addr)	\
do								\
{	unsigned long __d0,__d1;				\
	__asm__ __volatile__	(	"movw	%%dx,	%%ax	\n\t"	\
					"andq	$0x7,	%%rcx	\n\t"	\
					"addq	%4,	%%rcx	\n\t"	\
					"shlq	$32,	%%rcx	\n\t"	\
					"addq	%%rcx,	%%rax	\n\t"	\
					"xorq	%%rcx,	%%rcx	\n\t"	\
					"movl	%%edx,	%%ecx	\n\t"	\
					"shrq	$16,	%%rcx	\n\t"	\
					"shlq	$48,	%%rcx	\n\t"	\
					"addq	%%rcx,	%%rax	\n\t"	\
					"movq	%%rax,	%0	\n\t"	\
					"shrq	$32,	%%rdx	\n\t"	\
					"movq	%%rdx,	%1	\n\t"	\
					:"=m"(*((unsigned long *)(gate_selector_addr)))	,					\
					 "=m"(*(1 + (unsigned long *)(gate_selector_addr))),"=&a"(__d0),"=&d"(__d1)		\
					:"i"(attr << 8),									\
					 "3"((unsigned long *)(code_addr)),"2"(0x8 << 16),"c"(ist)				\
					:"memory"		\
				);				\
}while(0)
```

```assembly
					"movq	%%rax,	%0	\n\t"	\
					"shrq	$32,	%%rdx	\n\t"	\
					"movq	%%rdx,	%1	\n\t"	\
```

这几句就把设置好的IDT表项写入IDT了

### errcode

有些异常是处理器会压入errocde的，这里举个简单的例子来了解一下`errcode`。比如`#PF`的errocde

![image-20240321163055607](%E7%AC%94%E8%AE%B0.assets/image-20240321163055607.png)

P:页表的P位是否存在的错误,1页表存在,说明是涉及到页表保护，0页表本身不存在触发的异常

W/R:1写错误,0读取错误

U/S:1 R0异常,0 R3异常

RSVD:1 页表项保留位置位触发异常(**IA32E 开启分页，页表项保留位默认不可置为**)

I/D:1 执行触发的不可执行的异常

## 初级内存管理

在boot的时候，调用了bios的int，来获取当前物理地址的信息。

本质上就是一个内存结构体数组，这个结构体

```C
struct Bios_Phy_Struct{
    unsigned long start;
    unsigned long length;
    unsigned int type;
}
```

书中对于物理内存的管理使用的是`Global+Pgae+Zone`的方便,Global就是全局变量，Page,Zone前者是表述把物理内存分为2MB的pages，使用情况，而Zone则把物理地址划分了区域

### 内核相关结构初始化

- **Global**

```C++
for(i = 0;i < 32;i++)
	{
		color_printk(ORANGE,BLACK,"Address:%#018lx\tLength:%#018lx\tType:%#010x\n",p->address,p->length,p->type);
		unsigned long tmp = 0;
		if(p->type == 1)
			TotalMem +=  p->length;

		memory_management_struct.e820[i].address += p->address;

		memory_management_struct.e820[i].length	 += p->length;

		memory_management_struct.e820[i].type	 = p->type;
		
		memory_management_struct.e820_length = i;

		p++;
		if(p->type > 4 || p->length == 0 || p->type < 1)
			break;		
	}
```

遍历之前boot时候存在`0x7E00`的物理地址的(**现在被映射到0xffff800000007e00**)

成功转移到全局变量之后，开始根据当前内核的结束地址，设置`bitmap`,`pages`,`zones`的起始地址和结束地址，同时初始化值。

```C++
//bits map construction init

	memory_management_struct.bits_map = (unsigned long *)((memory_management_struct.end_brk + PAGE_4K_SIZE - 1) & PAGE_4K_MASK);

	memory_management_struct.bits_size = TotalMem >> PAGE_2M_SHIFT;

	memory_management_struct.bits_length = (((unsigned long)(TotalMem >> PAGE_2M_SHIFT) + sizeof(long) * 8 - 1) / 8) & ( ~ (sizeof(long) - 1));

	memset(memory_management_struct.bits_map,0xff,memory_management_struct.bits_length);		//init bits map memory

	//pages construction init

	memory_management_struct.pages_struct = (struct Page *)(((unsigned long)memory_management_struct.bits_map + memory_management_struct.bits_length + PAGE_4K_SIZE - 1) & PAGE_4K_MASK);

	memory_management_struct.pages_size = TotalMem >> PAGE_2M_SHIFT;

	memory_management_struct.pages_length = ((TotalMem >> PAGE_2M_SHIFT) * sizeof(struct Page) + sizeof(long) - 1) & ( ~ (sizeof(long) - 1));

	memset(memory_management_struct.pages_struct,0x00,memory_management_struct.pages_length);	//init pages memory

	//zones construction init

	memory_management_struct.zones_struct = (struct Zone *)(((unsigned long)memory_management_struct.pages_struct + memory_management_struct.pages_length + PAGE_4K_SIZE - 1) & PAGE_4K_MASK);

	memory_management_struct.zones_size   = 0;

	memory_management_struct.zones_length = (5 * sizeof(struct Zone) + sizeof(long) - 1) & (~(sizeof(long) - 1));

	memset(memory_management_struct.zones_struct,0x00,memory_management_struct.zones_length);
```

- **zone**

```C++
for(i = 0;i <= memory_management_struct.e820_length;i++)
	{
		unsigned long start,end;
		struct Zone * z;
		struct Page * p;
		unsigned long * b;

		if(memory_management_struct.e820[i].type != 1)
			continue;
		start = PAGE_2M_ALIGN(memory_management_struct.e820[i].address);
		end   = ((memory_management_struct.e820[i].address + memory_management_struct.e820[i].length) >> PAGE_2M_SHIFT) << PAGE_2M_SHIFT;
		if(end <= start)
			continue;
		
		//zone init

		z = memory_management_struct.zones_struct + memory_management_struct.zones_size;
		memory_management_struct.zones_size++;

		z->zone_start_address = start;
		z->zone_end_address = end;
		z->zone_length = end - start;

		z->page_using_count = 0;
		z->page_free_count = (end - start) >> PAGE_2M_SHIFT;

		z->total_pages_link = 0;

		z->attribute = 0;
		z->GMD_struct = &memory_management_struct;

		z->pages_length = (end - start) >> PAGE_2M_SHIFT;
		z->pages_group =  (struct Page *)(memory_management_struct.pages_struct + (start >> PAGE_2M_SHIFT)); //这一句的意思是page结构+当前zone.start的>>PAGE_2M_SHIFT就是当前zone的group
```

zone的初始化就是继续遍历一遍结构，对比`type==1`,也就是normal内存，填入zone，初始化zone这个结构体的相关内容。因为是2MB管理的，所以zone的`page_free_count=(end - start) >> PAGE_2M_SHIFT;`

- **pages**

```C++
//page init
		p = z->pages_group;//这是计算好了的
		for(j = 0;j < z->pages_length; j++ , p++)
		{
			p->zone_struct = z;
			p->PHY_address = start + PAGE_2M_SIZE * j;
			p->attribute = 0;

			p->reference_count = 0;

			p->age = 0;

			//这是将对应的page转换到bitmap，xor一下都是0了，因为本来就是1
			//其实挺容易理解的 他写的比较难理解
			//一个unsign long是64位，也就是可以表示64个2MB物理页面，>>6=2的6次方==64
			//所以前面((p->PHY_address >> PAGE_2M_SHIFT) >> 6))就是找位于第几个unsigned long  1UL << (p->PHY_address >> PAGE_2M_SHIFT) % 64后面的则是位于这个unsigned long的第几位
			//公式就是 (p+(cur_phy_address >> PHY_SIZE_SHIFT) / 8*sizeof(bit_map_type)) ^=1 << (cur_phy_address >> PHY_SIZE_SHIFT) % 8*sizeof(bit_map_type)

			*(memory_management_struct.bits_map + ((p->PHY_address >> PAGE_2M_SHIFT) >> 6)) ^=/*亦或 表示没有使用*/ 1UL << (p->PHY_address >> PAGE_2M_SHIFT) % 64;

		}
		
	}
```

如代码清单表示，pages的起始地址可以计算，根据`j`物理地址页表索引来进行初始化，同时把`bitmap`给复位。注释写的比较详细，个人觉得这本书的很多代码写的都有点难懂。

- 其他

初始化上面哪些，基本上是完成了物理空间的结构收集，还有一些收尾的工作。比如我们为了方便，之前在页表映射的时候低0x100000地址的物理地址线性地址是1:1映射的。现在因为有这个结构了，就得解除映射了。
**同时我们要根据内核所占据的虚拟空间的范围来进行设置`pages`**这个很重要，因为毫无疑问我们的内核也是使用了物理地址空间的。

```C++
//因为内核是从物理地址0开始的? 所以这个i就是内核+内存管理单元所占据的全部2MB内存索引
	i = Virt_To_Phy(memory_management_struct.end_of_struct) >> PAGE_2M_SHIFT;

	//把他初始化成如下
	for(j = 0;j <= i;j++)
	{
		page_init(memory_management_struct.pages_struct + j,PG_PTable_Maped | PG_Kernel_Init | PG_Active | PG_Kernel);
	}


	Global_CR3 = Get_gdt();

	color_printk(INDIGO,BLACK,"Global_CR3\t:%#018lx\n",Global_CR3);
	color_printk(INDIGO,BLACK,"*Global_CR3\t:%#018lx\n",*Phy_To_Virt(Global_CR3) & (~0xff));
	color_printk(PURPLE,BLACK,"**Global_CR3\t:%#018lx\n",*Phy_To_Virt(*Phy_To_Virt(Global_CR3) & (~0xff)) & (~0xff));


	//消除页表一致性，把前10个pml4e内容给清楚，只留下高地址(内核地址)的映射
	for(i = 0;i < 10;i++)
		*(Phy_To_Virt(Global_CR3)  + i) = 0UL;
	
	flush_tlb();
```

`flush_tlb`很简单，他用的`mov cr3,%0`来回mov一下，整个刷新。

### alloc memory

```C++
struct Page * alloc_pages(int zone_select,int number,unsigned long page_flags)
{
	int i;
	unsigned long page = 0;

	int zone_start = 0;
	int zone_end = 0;

	switch(zone_select)
	{
		case ZONE_DMA:
				zone_start = 0;
				zone_end = ZONE_DMA_INDEX;

			break;

		case ZONE_NORMAL:
				zone_start = ZONE_DMA_INDEX;
				zone_end = ZONE_NORMAL_INDEX;

			break;

		case ZONE_UNMAPED:
				zone_start = ZONE_UNMAPED_INDEX;
				zone_end = memory_management_struct.zones_size - 1;

			break;

		default:
			color_printk(RED,BLACK,"alloc_pages error zone_select index\n");
			return NULL;
			break;
	}

	for(i = zone_start;i <= zone_end; i++)
	{
		struct Zone * z;
		unsigned long j;
		unsigned long start,end,length;
		unsigned long tmp;

		if((memory_management_struct.zones_struct + i)->page_free_count < number)
			continue;

		z = memory_management_struct.zones_struct + i;
		start = z->zone_start_address >> PAGE_2M_SHIFT;
		end = z->zone_end_address >> PAGE_2M_SHIFT;
		length = z->zone_length >> PAGE_2M_SHIFT;

		//余剩的位数？
		tmp = 64 - start % 64;

		for(j = start;j <= end;j += j % 64 ? tmp : 64/*j是对齐的?*/)
		{
			//位图的第几个unsigned long 
			unsigned long * p = memory_management_struct.bits_map + (j >> 6);

			//左移的位数
			unsigned long shift = j % 64;


			unsigned long k;


			for(k = shift;k < 64 - shift;k++)
			{
				if( !(  ( (*p >> k) | (*(p + 1) << (64 - k)) ) & (number == 64 ? 0xffffffffffffffffUL : ((1UL << number) - 1))  ) )
				{
					unsigned long	l;
					page = j + k - 1;
					for(l = 0;l < number;l++)
					{
						struct Page * x = memory_management_struct.pages_struct + page + l;
						page_init(x,page_flags);
					}
					goto find_free_pages;
				}
			}
		
		}
	}

	return NULL;

find_free_pages:

	return (struct Page *)(memory_management_struct.pages_struct + page);
}
```

他这个分配比较简单，核心算法就是通过比较`bitmap`，这个比较bitmap的比较难理解。

他最多可以申请64个pages，`((1UL << number) - 1))`这个就是申请的掩码，因为用的`unsigned long`数组当作bitmap，需要考虑到跨`unsigned long`的问题。

大体算法是计算出当前区域的start,然后算出位于bitmap的起始地址和unsigend long的起始位数，`tmp`就是余剩的位数，`j`是位图的位数索引，如果j不是对齐的，那么就先加`tmp`,让他对齐，如果这次没对齐的还没找到合适的页面，那么下次一定是按照对齐了。

```C++
for(j = start;j <= end;j += j % 64 ? tmp : 64/*j是对齐的?*/)
{}
```

对于可能跨`unsigned long`的处理

```C++
for(k = shift;k < 64 - shift;k++)
			{
				if( !(  ( (*p >> k) | (*(p + 1) << (64 - k)) ) & (number == 64 ? 0xffffffffffffffffUL : ((1UL << number) - 1))  ) )
```

因为最多就申请64pages,所以巧妙地` (*p >> k) | (*(p + 1) << (64 - k))` `|`将可能跨的结合起来

所以感觉这个算法还是不错的，这种位扫描位图判断，涉及到跨页的情况，用这种上一个`unsiged long`多少个bit,下一个`unsigned long`用`64-上一个扫描的`,然后拼接成一个完整的`unsigned long`,比较位图就可以了。

前提是必须有限制。

最终可以有如下的结果

![image-20240321165257654](%E7%AC%94%E8%AE%B0.assets/image-20240321165257654.png)

## 中断处理

在单核处理器时期，CPU只有一个中断引脚(`INTR`)，最常用的是`8259A PIC`，后续的多核处理器更加复杂之后引入了`APIC`框架。

### 8259a pic

![image-20240404085708165](%E7%AC%94%E8%AE%B0.assets/image-20240404085708165.png)

一般是两个8259级联，因为一个这个中断控制芯片引脚太少了，所以级联让他们中断接收引脚(`INT`)多一点，从而可以多链接一点外围设备。通常情况下，8259A PIC会按照下表与外围设备进行链接。

![image-20240404085911833](%E7%AC%94%E8%AE%B0.assets/image-20240404085911833.png)

可以看到，传统的8259A 其实也就最多能链接8+8-1个外围设备。

8259A PIC是可以编程的中断控制器，因此他有一套可以配置的寄存器用于可编程自定义。一个8259包含两组寄存器，分别是`ICW(Initializeaion COmmand word,初始化命令字)`和`OCW(Operational Control Word,操作控制字)`，首先设置ICW以初始化8259，而ocw则用于操作中断控制器。

我们知道，**PC机对于外围设备的操作一般是I/O映射设备地址的方式**，也就是8259的寄存器被映射到了IO端口地址空间，需要通过IN OUT指令来进行访问，主 8259A PIC I/O端口是20H和21H,另一个是A0H和A1H。通过上述端口来设置ICW和OCW寄存器，**进而配置8259的`IRR(int reuqest register)`,`PR`,`ISR`,`IMR`等寄存器。**

**其实通过后面的架构不难发现，是ocw 和 icw影响上述几个状态寄存器**

`IRP`这个寄存器是用来保存IR0~IR7引脚上接受的中断请求，IMR(int mask register)，顾名思义，就是用于记录屏蔽的外部引脚，这两个寄存器都是8bit的，用于代表每一位IRN;

PR(Priority Resolver)优先级解析器从IRP寄存器接受中断优先级最高的，然后发送到`ISR(In-Service Register)正在服务寄存器`，ISR记录正在处理的中断请求，然后8259a通过INT引脚向CPU发送一个中断信号，CPU每执行一条指令就会检测是否收到相关信号，一旦发现有中断信号，那么CPU不执行接下来的一条指令，转而去给8259A发送一个INTA(`INT Ack`)表示处理次中断信号，8259a收到，会把中断请求保存在ISR(置位相关位)中。复位IRR对应的中断请求信号位。

整体的工作流程如下

![image-20240404091537112](%E7%AC%94%E8%AE%B0.assets/image-20240404091537112.png)

注意，到目前，CPU还没有进行任何的中断处理，放到接着，CPU发送第二次INTA信号，第二次是通知PIC发送int vector。因为第一次发送INTA的时候已经记录下来当前正在处理的中断信息了。然后PIC通过数据总线发送到CPU,CPU内部读取，跳转到相应的IDT执行中断处理程序。

具体的执行逻辑要看8259相关的控制寄存器，如果采取`AEOI(Auto End of Interrupt)`，那么就会在第二个INTA的时候复位ISR对应位，则代表当前没有正在投递的中断。如果采取`EOI(End Of Interrupt)`，那么必须等到CPU在处理完中断的时候发送EOI(这个发送要手动改写0x20端口)信号来复位ISR。在此之后，PIC 继续判断下一个最高优先级中断，重复如上过程。

接下来介绍ICW寄存器，这个寄存器有四个，**按照ICW1-ICW4顺序初始化**，主PIC IO端口ICW1映射20h，其余21h，从PIC io端口 ICW1映射到A0h,其他A1h(这里可能会对同一个端口映射不同的寄存器感到困扰，其实这几个ICW，OCW是有顺序的，写入IO端口也是有顺序的)。**一般来说先配置主PIC 的ICW然后配置从芯片的ICW。**

- **ICW1**

![image-20240404092948612](%E7%AC%94%E8%AE%B0.assets/image-20240404092948612.png)

主从都是固定为00010001B(11h)，Linux中都初始化成11h；

- **ICW2**

![image-20240404093102688](%E7%AC%94%E8%AE%B0.assets/image-20240404093102688.png)

icw2是8bit寄存器，中断向量号是一个`Base`,也就是IR0==BASE,如果触发IR3,那么就是BASE+3的向量号，投递给CPU.

linux初始化为主ICW2 `20h(20H-27H)`,从`28H(28H~2FH)`

- **ICW3**

主PIC

![image-20240404093243507](%E7%AC%94%E8%AE%B0.assets/image-20240404093243507.png)

这个主要是主PIC进行设置。一般是IR2进行级联另一个芯片。

从PIC

![image-20240404093342483](%E7%AC%94%E8%AE%B0.assets/image-20240404093342483.png)

- **ICW4**

![image-20240404093400518](%E7%AC%94%E8%AE%B0.assets/image-20240404093400518.png)

AEOI:在CPU发送第二个INTA时候，自动复位ISR

EOI:CPU手动发送EOI指令，来复位ISR

FNM(Fully Nested Mode):中断优先级 从高到低依次为IR0~IR7,高处理的时候屏蔽低优先级

SFNM(Special Fully Nested Mode)：与FNM基本相同，唯一区别是主PIC不会频闭从PIC,主可以接受从PIC优先级更高的IR

**通常icw4都是设置成01h**，使用简单的逻辑。

接下来是OCW寄存器，PIC有三个OCW,用于控制调整工作期间的PIC。无顺序之分，主OCW1->I/O端口21h,OCW2、OCW3->I/O端口20h，以此类推从PIC。

- **OCW1**

![image-20240404095452473](%E7%AC%94%E8%AE%B0.assets/image-20240404095452473.png)

- OCW2

![image-20240404095547657](%E7%AC%94%E8%AE%B0.assets/image-20240404095547657.png)

D5-D7位多种模式组成如下

![image-20240404095622558](%E7%AC%94%E8%AE%B0.assets/image-20240404095622558.png)

bit7=1,代表8259使用8位循环队列保存各个引脚中断请求，一个中断结束，该优先级自动降低到最低。依次类推，也就是队列尾部优先级最低。

bit7=1,bit6=1,如果这样,那么在循环优先级的基础上,D0~D2指定优先级设为最低优先级。然后按照循环降低。

- **OCW3**

![image-20240404100015861](%E7%AC%94%E8%AE%B0.assets/image-20240404100015861.png)

### 触发中断

文章里面写的实在是太难懂...各种宏。不过还是比较能够看出来8259A PIC的初始化过程的，可以直观地了解到这些外设的初始化过程。

简而言之，首先弄好所有的IRQN处理函数(也就是中断处理函数)

```C++
void (* interrupt[24])(void)=
{
	IRQ0x20_interrupt,
	IRQ0x21_interrupt,
	IRQ0x22_interrupt,
	IRQ0x23_interrupt,
	IRQ0x24_interrupt,
	IRQ0x25_interrupt,
	IRQ0x26_interrupt,
	IRQ0x27_interrupt,
	IRQ0x28_interrupt,
	IRQ0x29_interrupt,
	IRQ0x2a_interrupt,
	IRQ0x2b_interrupt,
	IRQ0x2c_interrupt,
	IRQ0x2d_interrupt,
	IRQ0x2e_interrupt,
	IRQ0x2f_interrupt,
	IRQ0x30_interrupt,
	IRQ0x31_interrupt,
	IRQ0x32_interrupt,
	IRQ0x33_interrupt,
	IRQ0x34_interrupt,
	IRQ0x35_interrupt,
	IRQ0x36_interrupt,
	IRQ0x37_interrupt,
};
```

弄到一个函数指针数组里面，然后`init_interrupt`里面进行初始化

```C++
void init_interrupt()
{
	int i;
	//设置IDT里面的描述符
	for(i = 32;i < 56;i++)
	{
		set_intr_gate(i , 2 , interrupt[i - 32]);
	}

	color_printk(RED,BLACK,"8259A init \n");

	//重复向一个端口写入值，如果是同一个接口 但是这个顺序固定
	//8259A-master	ICW1-4
	io_out8(0x20,0x11);
	io_out8(0x21,0x20);
	io_out8(0x21,0x04);
	io_out8(0x21,0x01);

	//8259A-slave	ICW1-4
	io_out8(0xa0,0x11);
	io_out8(0xa1,0x28);
	io_out8(0xa1,0x02);
	io_out8(0xa1,0x01);

	//8259A-M/S	OCW1
	io_out8(0x21,0x00);
	io_out8(0xa1,0x00);

	sti();
}
```

这里设置了`ICW2`的vector base分别主从是`0x20`和`0x28`

从上面看到，初始化过程都是同时写一个`端口`,因此这个顺序是固定的也不难理解了。

这里的IRQN函数省略(**和异常处理基本上是一样的，errcode无，汇编代码修改rdi(arg1),rsi(arg2)然后jmp到do_IRQ这个地方**)

```C++
void do_IRQ(unsigned long regs,unsigned long nr)	//regs:rsp,nr
{
	color_printk(RED,BLACK,"do_IRQ:%#08x\t",nr);
	io_out8(0x20,0x20);
}
```

`io_out8(0x20,0x20);`这个就是往主PIC写`EOI`信号，代表中断处理结束。

最后运行效果如下

![image-20240404145635717](%E7%AC%94%E8%AE%B0.assets/image-20240404145635717.png)

全都变成了IRQ20(也就是pic引脚位IR0)链接的，也就是时钟中断触发了,其实这个时候按一下键盘可以发现，会触发一条IRQ1,但是继续按再也不能触发了，这是因为IRQ1是连接到的是8042控制器，而这个控制器会把按键的`scan code`放到端口，必须读取，才会清空这个缓冲区。8042控制器才会继续写入这个端口。

### 简单键盘驱动编写

因为书中是早期的PC机架构，并且没有考虑APIC，USB键盘啥的。

书中的键盘是键盘控制器与PS/2接口键盘进行链接的。当然在现在的PC机架构统一都集成到芯片组里面去了

![image-20240404155122407](%E7%AC%94%E8%AE%B0.assets/image-20240404155122407.png)

可以看到，8042不仅控制键盘，其实还控制A20,鼠标等等外围设备连接，通过INT连接8259的IRQ1引脚，触发中断。(**其实8042就是南桥的一部分!只不过现在计算机南桥北桥和cpu+各种控制器集成在一块变成chipset了**)

当然键盘里面本身就有一个名叫8048的控制器，它输出不同格式的`Scan code`，有三种。一般而言都是采用`AT`扫描码。

键盘控制器的端口是60h和64h端口，60h是读写缓冲区，64h地址处的寄存器用于读取寄存器状态或者发送控制命令。

```C++
void do_IRQ(unsigned long regs,unsigned long nr)	//regs:rsp,nr
{
	unsigned char x;
	color_printk(RED,BLACK,"do_IRQ:%#08x\t",nr);
	x = io_in8(0x60);
	color_printk(RED,BLACK,"key code:%#08x\n",x);
	io_out8(0x20,0x20);
}
```

如图就是简单地处理IRQ21h,也就是键盘。

当然目前我们没有对8042控制器初始化或者设置啥的，但是无所谓，这已经算是一个"简单的"键盘驱动了。实际上对8042控制器初始化是很复杂的。

## 进程管理基础概念

后续还有一章单独的进程管理，这一章只讲解基础概念，至于一些太基础的概念，比如进程、线程是什么，就暂且不解释了。

这一大节的主要内容就是展示`PCB`的作用和如何存放以及如何快速查找，总的来说因为逆向过windows的内核，对于这方面还是比较容易接受。同时展示了第一个系统进程的创建。

### PCB

- **PCB的定义**

就是进程控制块，对于windows,在EPROCESS对象结构的Pcb里面，其实就是`KPROCESS`,保存着最基础的 信息。比如页表、内存使用信息，线程...

文中的操作系统参考的是linux，也差不多，里面的结构如下

```C++

struct task_struct
{
	struct List list;
	volatile long state;
	unsigned long flags;

	struct mm_struct *mm;
	struct thread_struct *thread;

	unsigned long addr_limit;	/*0x0000,0000,0000,0000 - 0x0000,7fff,ffff,ffff user*/
					/*0xffff,8000,0000,0000 - 0xffff,ffff,ffff,ffff kernel*/

	long pid;

	long counter;

	long signal;

	long priority;
};

```

其实不难发现，对比windows的KPROCESS结构，里面很多设计思想都是一样的 。比如链表，比如内存使用信息`mm,vad`...

`mm_struct`和`thread_struct`结构分别如下

```C++
struct mm_struct
{
	pml4t_t *pgd;	//page table point
	
	unsigned long start_code,end_code;
	unsigned long start_data,end_data;
	unsigned long start_rodata,end_rodata;
	unsigned long start_brk,end_brk;
	unsigned long start_stack;	
};

/*

*/

struct thread_struct
{
	unsigned long rsp0;	//in tss

	unsigned long rip;
	unsigned long rsp;	

	unsigned long fs;
	unsigned long gs;

	unsigned long cr2;
	unsigned long trap_nr;
	unsigned long error_code;
};
```

分别描述当前进程的内存空间使用情况+线程对象。

- **PCB的存放位置**

文章把PCB巧妙地存在线程内核栈的`栈顶`

```C++
union task_union
{
	struct task_struct task;
	unsigned long stack[STACK_SIZE / sizeof(unsigned long)];
}__attribute__((aligned (8)));	//8Bytes align
```

因为他的栈分配地址是按照`STACK_SIZE`对齐的,所以如果想要找到PCB,只需要按照这个对齐即可。**文中是32KB对齐的。**

比如可以看文中的

```C++
inline	struct task_struct * get_current()
{
	struct task_struct * current = NULL;
	__asm__ __volatile__ ("andq %%rsp,%0	\n\t":"=r"(current):"0"(~32767UL));
	return current;
}
```

都是通过and运算进行的。

而`32767`是`7FFF`,然取反之后与运算就可以了。其实就是按照0x8000对齐,因为0x1000是4kb,32kb是8倍,0x8000-1=0x7fff。

### 进程的创建

在了解进程创建之前，我们先了解一下进程的切换。和windows差不多，进程的切换肯定是要在内核的公共区域进行，这块区域前后进程必须是共享的。

![image-20240408160259629](%E7%AC%94%E8%AE%B0.assets/image-20240408160259629.png)

基本上步骤都是

分配器找到pre和next的进程/线程->进入进程切换函数->切换cr3/tss/fs/gs/rsp->分配器公共代码->根据rsp 返回之前正在执行的代码。

文中中的切换进程的代码如下:

```C++
#define switch_to(prev,next)			\
do{							\
	__asm__ __volatile__ (	"pushq	%%rbp	\n\t"	\
				"pushq	%%rax	\n\t"	\
				"movq	%%rsp,	%0	\n\t"	\
				"movq	%2,	%%rsp	\n\t"	\
				"leaq	1f(%%rip),	%%rax	\n\t"	\
				"movq	%%rax,	%1	\n\t"	\
				"pushq	%3		\n\t"	\
				"jmp	__switch_to	\n\t"	\
				"1:	\n\t"	\
				"popq	%%rax	\n\t"	\
				"popq	%%rbp	\n\t"	\
				:"=m"(prev->thread->rsp),"=m"(prev->thread->rip)		\
				:"m"(next->thread->rsp),"m"(next->thread->rip),"D"(prev),"S"(next)	\
				:"memory"		\
				);			\
}while(0)

inline void __switch_to(struct task_struct *prev,struct task_struct *next)
{

	init_tss[0].rsp0 = next->thread->rsp0;

	set_tss64(init_tss[0].rsp0, init_tss[0].rsp1, init_tss[0].rsp2, init_tss[0].ist1, init_tss[0].ist2, init_tss[0].ist3, init_tss[0].ist4, init_tss[0].ist5, init_tss[0].ist6, init_tss[0].ist7);

	__asm__ __volatile__("movq	%%fs,	%0 \n\t":"=a"(prev->thread->fs));
	__asm__ __volatile__("movq	%%gs,	%0 \n\t":"=a"(prev->thread->gs));

	__asm__ __volatile__("movq	%0,	%%fs \n\t"::"a"(next->thread->fs));
	__asm__ __volatile__("movq	%0,	%%gs \n\t"::"a"(next->thread->gs));

	color_printk(WHITE,BLACK,"prev->thread->rsp0:%#018lx\n",prev->thread->rsp0);
	color_printk(WHITE,BLACK,"next->thread->rsp0:%#018lx\n",next->thread->rsp0);
}
```

可以发现，就像我们说的那样，切rsp/fs/gs/tss等这些数据。

接下来我们看第一个进程的创建

```C++
void task_init()
{
	struct task_struct *p = NULL;

	init_mm.pgd = (pml4t_t *)Global_CR3;

	init_mm.start_code = memory_management_struct.start_code;
	init_mm.end_code = memory_management_struct.end_code;

	init_mm.start_data = (unsigned long)&_data;
	init_mm.end_data = memory_management_struct.end_data;

	init_mm.start_rodata = (unsigned long)&_rodata; 
	init_mm.end_rodata = (unsigned long)&_erodata;

	init_mm.start_brk = 0;
	init_mm.end_brk = memory_management_struct.end_brk;
              
	init_mm.start_stack = _stack_start;

//	init_thread,init_tss
	set_tss64(init_thread.rsp0, init_tss[0].rsp1, init_tss[0].rsp2, init_tss[0].ist1, init_tss[0].ist2, init_tss[0].ist3, init_tss[0].ist4, init_tss[0].ist5, init_tss[0].ist6, init_tss[0].ist7);

	init_tss[0].rsp0 = init_thread.rsp0;

	list_init(&init_task_union.task.list);

	//创建一个新的进程 叫做init
	kernel_thread(init,10,CLONE_FS | CLONE_FILES | CLONE_SIGNAL);

	init_task_union.task.state = TASK_RUNNING;

	//等效于windows的CONTAING_RECORD
	p = container_of(list_next(&current->list),struct task_struct,list);

	switch_to(current,p);
}
```

第一个进程的创建其实很简单，因为第一个进程的PCB是固定在内核`kernel.bin`的节区的，也就是全局变量中。

当然这个节区也是按照8KB对齐。

```C++
	//创建一个新的进程 叫做init
	kernel_thread(init,10,CLONE_FS | CLONE_FILES | CLONE_SIGNAL);
```

这个就是创建了第二个进程，叫做init。

继续看里面的代码,来细看一个进程到底是如何创建的

```C++
int kernel_thread(unsigned long (* fn)(unsigned long), unsigned long arg, unsigned long flags)
{
	struct pt_regs regs;
	memset(&regs,0,sizeof(regs));

	regs.rbx = (unsigned long)fn;
	regs.rdx = (unsigned long)arg;

	regs.ds = KERNEL_DS;
	regs.es = KERNEL_DS;
	regs.cs = KERNEL_CS;
	regs.ss = KERNEL_DS;
	regs.rflags = (1 << 9);
	regs.rip = (unsigned long)kernel_thread_func;

	return do_fork(&regs,flags,0,0);
}
```

可以看到,构造了一个regs结构,`regs.rbx = (unsigned long)fn;`是这个线程要去执行函数，arg就是参数了

而实际上,我们看到`regs.rip = (unsigned long)kernel_thread_func;`其实第一个真正去执行的函数，可以理解位中转函数，就是他的中转可以达到我们调用这个函数时候设置的线程上下文。

而do_fork则是创建一个进程，换句话说就是创建一个新的`PCB`,这个也是参考了linux，细看：

```C++
unsigned long do_fork(struct pt_regs * regs, unsigned long clone_flags, unsigned long stack_start, unsigned long stack_size)
{
	struct task_struct *tsk = NULL;
	struct thread_struct *thd = NULL;
	struct Page *p = NULL;
	
	color_printk(WHITE,BLACK,"alloc_pages,bitmap:%#018lx\n",*memory_management_struct.bits_map);

	p = alloc_pages(ZONE_NORMAL,1,PG_PTable_Maped | PG_Active | PG_Kernel);

	color_printk(WHITE,BLACK,"alloc_pages,bitmap:%#018lx\n",*memory_management_struct.bits_map);

	tsk = (struct task_struct *)Phy_To_Virt(p->PHY_address);
	color_printk(WHITE,BLACK,"struct task_struct address:%#018lx\n",(unsigned long)tsk);

	memset(tsk,0,sizeof(*tsk));
	*tsk = *current;//直接复制

	//添加到链表中
	list_init(&tsk->list);
	list_add_to_before(&init_task_union.task.list,&tsk->list);	
	tsk->pid++;	
	tsk->state = TASK_UNINTERRUPTIBLE;

	thd = (struct thread_struct *)(tsk + 1);
	tsk->thread = thd;	

	memcpy(regs,(void *)((unsigned long)tsk + STACK_SIZE - sizeof(struct pt_regs)),sizeof(struct pt_regs));

	thd->rsp0 = (unsigned long)tsk + STACK_SIZE;
	thd->rip = regs->rip;
	thd->rsp = (unsigned long)tsk + STACK_SIZE - sizeof(struct pt_regs);

	if(!(tsk->flags & PF_KTHREAD))
		thd->rip = regs->rip = (unsigned long)ret_from_intr;

	tsk->state = TASK_RUNNING;

	return 0;
}
```

其实就是复制，然后设置自己的一些结构，和linux和fork还是有点像的。

大概过程就是通过kalloc分配一块自己的内存，然后用作pcb，同时fork的时候传入了一个regs，这个是新的进程的执行环境，复制fork父进程的pcb之后，**直接根据regs跑到要求的地方即可。**

# chap3 应用层

**非常遗憾的是，由于本人不小心点了恢复快照，相关linux环境始终无法按照原先那样正常配置，bochs启动后无法正常跑到0x7c00，无法解决，以后无法查看运行效果了，对于积极性来言无疑是一个重大打击。**

前两章中，已经实现了os的`R0`，接下来书中将会简单地完善R3。R3和R0最大的区别就是R3可以随心所欲执行，出了任何异常都是R0兜底，同时不会直接操作外部的硬件设备，必须借助内核提供的相关接口，哪怕print一个`hello world`；因为他们也不具备配置处理器的功能，所以**需要在R0先配置完成，然后转到应用层。**

## 返回应用层

返回应用层需要跨权限级，所可以选择的是`iret`和`ret`，但是一般这两个指令涉及到大量的栈操作，一旦涉及到换栈，压栈，读写内存。那么这个CPU时钟周期就不是很快，因此为了解决这个问题，intel推出了新的处理器指令

`Syscall/sysret(x86，现代)`，`sysenter/sysexit(x86，早期)`；

上述两类跨级别调用的指令显著性的优点就是不会数据压栈，因此非常快速。

文中用的处理器比较老，所以是没有`syscall/sysret`这个指令的，但是他们和`sysenter/syseixt`其实没有啥差别。唯一区别是在进行CS和RIP切换的时候，访问的MSR寄存器不一样。**这一点可以理解，基本上处理器所更新的新的内容，都是放在新的MSR寄存器中。**

`sysexit`只能在R0执行，执行sysexit之前，一些数据需要被保存在以下MSR和GR(通用寄存器，下同)中：

- IA32_SYSENTER_CS(IA32E模式中，为IA32_SYSENTER_CS[15:0]+32,否则是[15:0]+16,栈选择子是+8)
- RDX,保存的地址被装载到RIP之中。
- RCX,返回的时候装到RSP之中,**这一点在`syscall`中二者是存储的是相反的。具体而言需要看intel硬件开发参考手册**

文中对于进程返回到R3的步骤如下：

`main`中调用`task_init();`

```C++
void task_init()
{
	struct task_struct *p = NULL;

	init_mm.pgd = (pml4t_t *)Global_CR3;

	init_mm.start_code = memory_management_struct.start_code;
	init_mm.end_code = memory_management_struct.end_code;

	init_mm.start_data = (unsigned long)&_data;
	init_mm.end_data = memory_management_struct.end_data;

	init_mm.start_rodata = (unsigned long)&_rodata; 
	init_mm.end_rodata = (unsigned long)&_erodata;

	init_mm.start_brk = 0;
	init_mm.end_brk = memory_management_struct.end_brk;

	init_mm.start_stack = _stack_start;
	
	wrmsr(0x174,KERNEL_CS);

//	init_thread,init_tss
	set_tss64(init_thread.rsp0, init_tss[0].rsp1, init_tss[0].rsp2, init_tss[0].ist1, init_tss[0].ist2, init_tss[0].ist3, init_tss[0].ist4, init_tss[0].ist5, init_tss[0].ist6, init_tss[0].ist7);

	init_tss[0].rsp0 = init_thread.rsp0;

	list_init(&init_task_union.task.list);

	kernel_thread(init,10,CLONE_FS | CLONE_FILES | CLONE_SIGNAL);

	init_task_union.task.state = TASK_RUNNING;

	p = container_of(list_next(&current->list),struct task_struct,list);

	switch_to(current,p);
}
```

其他的上一节进程创建的时候大概都知道是什么意思，而多加的代码部分在`kernel_thread`,flags=10;

我们知道，这个里面有一个`do_fork`,这样就会复制一个新的PCB，在进程链表新增一个。

```C++
p = container_of(list_next(&current->list),struct task_struct,list);

switch_to(current,p);
```

这两句话就是切换线程了，到我们`kernel_thread`创建的新的线程。第一个执行的函数毫无疑问是`init`

```C++
unsigned long init(unsigned long arg)
{
	struct pt_regs *regs;

	color_printk(RED,BLACK,"init task is running,arg:%#018lx\n",arg);

	current->thread->rip = (unsigned long)ret_system_call;
	current->thread->rsp = (unsigned long)current + STACK_SIZE - sizeof(struct pt_regs);
	regs = (struct pt_regs *)current->thread->rsp;

	__asm__	__volatile__	(	"movq	%1,	%%rsp	\n\t"
					"pushq	%2		\n\t"
					"jmp	do_execve	\n\t"
					::"D"(regs),"m"(current->thread->rsp),"m"(current->thread->rip):"memory");

	return 1;
}	
```

这里面初始化了进程的rsp，rip=`ret_system_call`，栈是最高(StackBase)，`push %2`就是push的ret_system_call。同时设置了`rdx`的值是`&regs`,也就是线程的context；

然后调用`do_execve`

```C++
unsigned long do_execve(struct pt_regs * regs)
{
	regs->rdx = 0x800000;	//RIP
	regs->rcx = 0xa00000;	//RSP
	regs->rax = 1;	
	regs->ds = 0;
	regs->es = 0;
	color_printk(RED,BLACK,"do_execve task is running\n");

	memcpy(user_level_function,(void *)0x800000,1024);

	return 0;
}
```

可以发现，do_execve有一个参数，linux下gun的编译器第一个参数是`rdi`，在`init`已经设置；

这里memcpy把`user_level_function`拷贝到了`0x800000`地址，然后修改regs(线程的context)，直接return;

这个时候return会跑到`ret_system_call`

```assembly
ENTRY(ret_system_call)						
	movq	%rax,	0x80(%rsp)		 
	popq	%r15				 
	popq	%r14				 	
	popq	%r13				 	
	popq	%r12				 	
	popq	%r11				 	
	popq	%r10				 	
	popq	%r9				 	
	popq	%r8				 	
	popq	%rbx				 	
	popq	%rcx				 	
	popq	%rdx				 	
	popq	%rsi				 	
	popq	%rdi				 	
	popq	%rbp				 	
	popq	%rax				 	
	movq	%rax,	%ds			 
	popq	%rax				 
	movq	%rax,	%es			 
	popq	%rax				 
	addq	$0x38,	%rsp	
	.byte	0x48		 
	sysexit			
```

`movq	%rax,	0x80(%rsp)`这样可以刚好让堆栈指向regs的栈顶，然后依次pop，可以还原寄存器，从而跑到指定的`user_level_funcion`。从而完成r0->r3的返回

## 系统调用

这里系统调用文中使用的是`sysenter`，总体来说，其实和windows的思想差不多，都是使用一个`系统调用号+系统调用表`；

```C++
void user_level_function()
{
	long ret = 0;
//	color_printk(RED,BLACK,"user_level_function task is running\n");
	char string[]="Hello World!\n";

	__asm__	__volatile__	(	"leaq	sysexit_return_address(%%rip),	%%rdx	\n\t"
					"movq	%%rsp,	%%rcx		\n\t"
					"sysenter			\n\t"
					"sysexit_return_address:	\n\t"
					:"=a"(ret):"0"(1),"D"(string):"memory");	

//	color_printk(RED,BLACK,"user_level_function task called sysenter,ret:%ld\n",ret);

	while(1);
}
```

`user_level_function`中，使用`sysenter`进行系统调用，而具体处理系统调用的地方

在初始化中如下设置

```C++
	wrmsr(0x174,KERNEL_CS);
	wrmsr(0x175,current->thread->rsp0);
	wrmsr(0x176,(unsigned long)system_call);
```

也就是一旦`sysenter`进入的就是`system_call`,这个其实和`windows`的`KiSystemCall64`是一样的。

这个函数是汇编写的，这里不在复制，总之最终设置好参数会跑到如下函数。

```C++
unsigned long  system_call_function(struct pt_regs * regs)
{
	return system_call_table[regs->rax](regs);
}
```

可以看到，是通过rax作为syscall_table的索引。

这便是系统调用的全部分了。比较简单，因为是intel后续增加的硬件功能，所以需要通过mSR寄存器来设置，难的是汇编+C语言写的函数直接的参数兼容。一些设计思想，linux和windows是通用的。

# chap4 intel硬件平台知识

简单地看了一下，发现没有什么值得去记录的，详细地参考，因为都学过

具体参考以前学过的IA32E模式+白皮书。

对于IA32E模式，想必已然了熟于心，无非就是一些段页保护手段，GDT、IDT、各种CR寄存器等。都是一些非常基础的东西。

而其他的硬件平台知识，比如APIC、性能计数....这些东西，都是后面新的intel处理器加的，都是放在了一些`MSR`寄存器进行启动的，后面学的APIC正是如此。

简而言之，现代硬件平台很多都是往安全方向发展,比如:

- IN/OUT->MMIO
- 8259A PIC ->APIC
- bios引导启动->uefi...

cpu的新功能基本上都是加载了MSR寄存器中，而外围设备硬件的新功能，比如USB接口，PCIe...基本上都是通过MMIO,来进行写相关内存，写协议，更安全。

而其他的硬件新技术，比如uefi，需要去看相关的uefi白皮书，来看她是如何启动引导的。

# chap5 高级内存管理单元

所谓高级内存管理，本书用到了和linux类似的虚拟地址管理--`slab`内存池管理，同时又对物理内存的分配进行了一次升级。

观察本书、linux、windows，其对于内存管理的大概思路惊人的相似，既物理内存的管理是通过一个全局变量数组，每一个数组项代表了物理地址N的使用情况，这个数组结构里面可能包含了这块物理地址的使用情况，是否映射到页表...

对于内核虚拟内存的管理，内核通常都是采取内存池的管理手段。本书介绍了`SLAB`内存池管理方法。

## slab内存池

其实原理非常简单，首先来看下面两个结构体

```C++
struct Slab
{
	struct List list;
	struct Page * page;

	unsigned long using_count;
	unsigned long free_count;

	void * Vaddress;

	unsigned long color_length;
	unsigned long color_count;

	unsigned long * color_map;
};

struct Slab_cache
{
	unsigned long	size;
	unsigned long	total_using;
	unsigned long	total_free;
	struct Slab *	cache_pool;
	struct Slab *	cache_dma_pool;
	void *(* constructor)(void * Vaddress,unsigned long arg);
	void *(* destructor)(void * Vaddress,unsigned long arg);
};
```

这就是slab内存池管理的最重要的两个结构体，其中`Slab_cache`可以理解为这个内存池的总管，而`slab`从属于`slab_cache`，**它是提前申请好的一块内存池**。

一个os中，有很多个`slab_cache`，他们的区别是**size**不同，比如可能是8B,32B...1MB，这个size决定了你在调用内存分配函数时候，具体去找哪一个`slab_cache`。

而`slab`中，有位图`slab.color_map`，这个位图的大小可以在下面看到是如何确定的。其实就是`2MB/size`就是当前`slab`可以分配的多少个内存对象，然后`(num>>6)<<3`就是color_map的大小了(**以字节为单位**)。

而`slab_cache`还有`constructor`和`destructor`，这是后面有`slab_malloc`和`slab_free`，会调用这两个函数，通知内存对象被销毁和创建。

书中有如下不同`size`的`slab_cache`：

```C++
struct Slab_cache kmalloc_cache_size[16] = 
{
	{32	,0	,0	,NULL	,NULL	,NULL	,NULL},
	{64	,0	,0	,NULL	,NULL	,NULL	,NULL},
	{128	,0	,0	,NULL	,NULL	,NULL	,NULL},
	{256	,0	,0	,NULL	,NULL	,NULL	,NULL},
	{512	,0	,0	,NULL	,NULL	,NULL	,NULL},
	{1024	,0	,0	,NULL	,NULL	,NULL	,NULL},			//1KB
	{2048	,0	,0	,NULL	,NULL	,NULL	,NULL},
	{4096	,0	,0	,NULL	,NULL	,NULL	,NULL},			//4KB
	{8192	,0	,0	,NULL	,NULL	,NULL	,NULL},
	{16384	,0	,0	,NULL	,NULL	,NULL	,NULL},
	{32768	,0	,0	,NULL	,NULL	,NULL	,NULL},
	{65536	,0	,0	,NULL	,NULL	,NULL	,NULL},			//64KB
	{131072	,0	,0	,NULL	,NULL	,NULL	,NULL},			//128KB
	{262144	,0	,0	,NULL	,NULL	,NULL	,NULL},
	{524288	,0	,0	,NULL	,NULL	,NULL	,NULL},
	{1048576,0	,0	,NULL	,NULL	,NULL	,NULL},			//1MB
};
```

既然有这些结构了，那么我们来看一下这些`slab_cache`具体是怎么初始化的吧。（上面的`salb_cache`结构只初始化了size）；

## `slab_cache`初始化

```C++
struct Slab_cache * slab_create(unsigned long size,void *(* constructor)(void * Vaddress,unsigned long arg),void *(* destructor)(void * Vaddress,unsigned long arg),unsigned long arg)
{
	struct Slab_cache * slab_cache = NULL;

	//返回的是虚拟地址,
	slab_cache = (struct Slab_cache *)kmalloc(sizeof(struct Slab_cache),0);
	
	if(slab_cache == NULL)
	{
		color_printk(RED,BLACK,"slab_create()->kmalloc()=>slab_cache == NULL\n");
		return NULL;
	}
	
	memset(slab_cache,0,sizeof(struct Slab_cache));

	slab_cache->size = SIZEOF_LONG_ALIGN(size);
	slab_cache->total_using = 0;
	slab_cache->total_free = 0;
	slab_cache->cache_pool = (struct Slab *)kmalloc(sizeof(struct Slab),0);
	
	if(slab_cache->cache_pool == NULL)
	{
		color_printk(RED,BLACK,"slab_create()->kmalloc()=>slab_cache->cache_pool == NULL\n");
		kfree(slab_cache);
		return NULL;
	}
	
	memset(slab_cache->cache_pool,0,sizeof(struct Slab));

	slab_cache->cache_dma_pool = NULL;
	slab_cache->constructor = constructor;
	slab_cache->destructor = destructor;

	list_init(&slab_cache->cache_pool->list);

	//分配物理页 也就是pages 2MB
	slab_cache->cache_pool->page = alloc_pages(ZONE_NORMAL,1,0);
	

	if(slab_cache->cache_pool->page == NULL)
	{
		color_printk(RED,BLACK,"slab_create()->alloc_pages()=>slab_cache->cache_pool->page == NULL\n");
		kfree(slab_cache->cache_pool);
		kfree(slab_cache);
		return NULL;
	}
	
	//这个物理页面设置相关属性
	page_init(slab_cache->cache_pool->page,PG_Kernel);

	//size是内存对象的大小 也就是这个内存池目前可以申请多少个如此大小的内存对象
	slab_cache->cache_pool->using_count = PAGE_2M_SIZE/slab_cache->size;

	slab_cache->cache_pool->free_count = slab_cache->cache_pool->using_count;

	slab_cache->total_free = slab_cache->cache_pool->free_count;

	//申请的pages物理地址 转换成虚拟地址
	slab_cache->cache_pool->Vaddress = Phy_To_Virt(slab_cache->cache_pool->page->PHY_address);

	slab_cache->cache_pool->color_count = slab_cache->cache_pool->free_count;

	//>>6<<3 意思是位图！，一个64bit 要>>6 然后 64bit占8字节 所以 <<3 这样可以算出color位图占多少个字节 方便下面申请
	slab_cache->cache_pool->color_length = ((slab_cache->cache_pool->color_count + sizeof(unsigned long) * 8 - 1) >> 6) << 3;

	slab_cache->cache_pool->color_map = (unsigned long *)kmalloc(slab_cache->cache_pool->color_length,0);

	if(slab_cache->cache_pool->color_map == NULL)
	{
		color_printk(RED,BLACK,"slab_create()->kmalloc()=>slab_cache->cache_pool->color_map == NULL\n");
		
		free_pages(slab_cache->cache_pool->page,1);
		kfree(slab_cache->cache_pool);
		kfree(slab_cache);
		return NULL;
	}
	
	memset(slab_cache->cache_pool->color_map,0,slab_cache->cache_pool->color_length);

	return slab_cache;
}
```

上述是一个`slab_cache`的创建过程，这里面有一个函数是`kmalloc`，在内核分配内存。

这个`kmalloc`用到了全局变量`kmalloc_cache_size`，这里你可能会疑惑，为什么`kmalloc`依赖`slab_cache`但是`slab_create`还用到这个函数了呢。

其实原因是有个函数`kmalloc_create`，这个里面初始化了`kmalloc_cache_size`全局变量中的每一个`slab_cache`。**而`slab_create`可以理解为我现在想创建其他大小的内存对象的内存池**，自然是在`slab_init`之后进行的了，所以不冲突。

这里主要是看一下`slab_create`的逻辑，他创建`slab_cache`，必然会创建一个新的`slab`，这个`slab`通过`alloc_page`申请一个`2MB`的内存，然后转换成线性地址，存在`slab::vaddress`。

接着计算位图，位图的大小计算完成用`kmalloc`申请，`color_length`是以字节为单位的，如何计算前文已经提到了。

```C++
//申请的pages物理地址 转换成虚拟地址
	slab_cache->cache_pool->Vaddress = Phy_To_Virt(slab_cache->cache_pool->page->PHY_address);

	slab_cache->cache_pool->color_count = slab_cache->cache_pool->free_count;

	//>>6<<3 意思是位图！，一个64bit 要>>6 然后 64bit占8字节 所以 <<3 这样可以算出color位图占多少个字节 方便下面申请
	slab_cache->cache_pool->color_length = ((slab_cache->cache_pool->color_count + sizeof(unsigned long) * 8 - 1) >> 6) << 3;

	slab_cache->cache_pool->color_map = (unsigned long *)kmalloc(slab_cache->cache_pool->color_length,0);
```

而这个地方，也显示了一个`size`大小的内存对象，在`2MB`内存池下，有多少个`count`的计算。

```C++
	//size是内存对象的大小 也就是这个内存池目前可以申请多少个如此大小的内存对象
	slab_cache->cache_pool->using_count = PAGE_2M_SIZE/slab_cache->size;

	slab_cache->cache_pool->free_count = slab_cache->cache_pool->using_count;

	slab_cache->total_free = slab_cache->cache_pool->free_count;
```

## `slab_cache`销毁

有相关创建逻辑，就有相关销毁逻辑，所谓的销毁是指销毁整个`slab_cache`

```C++
unsigned long slab_destroy(struct Slab_cache * slab_cache)
{
	struct Slab * slab_p = slab_cache->cache_pool;
	struct Slab * tmp_slab = NULL;

	//必须总的使用==0才可以销毁内存池
	if(slab_cache->total_using != 0)
	{
		color_printk(RED,BLACK,"slab_cache->total_using != 0\n");
		return 0;
	}

	//销毁slab_cache的每一个slab
	while(!list_is_empty(&slab_p->list))
	{
		tmp_slab = slab_p;
		slab_p = container_of(list_next(&slab_p->list),struct Slab,list);

		list_del(&tmp_slab->list);
		kfree(tmp_slab->color_map);

		page_clean(tmp_slab->page);
		free_pages(tmp_slab->page,1);
		kfree(tmp_slab);
	}

	kfree(slab_p->color_map);

	page_clean(slab_p->page);
	free_pages(slab_p->page,1);
	kfree(slab_p);
		
	kfree(slab_cache);

	return 1;
}	
```

整体逻辑比较简单，和`slab_create`对应。

## `slab_cache`内存对象的alloc/free

有内存池就要分配内存，对应的分配函数如下：

```C++
void * slab_malloc(struct Slab_cache * slab_cache,unsigned long arg)
{
	struct Slab * slab_p = slab_cache->cache_pool;
	struct Slab * tmp_slab = NULL;
	int j = 0;

	if(slab_cache->total_free == 0)
	{
		tmp_slab = (struct Slab *)kmalloc(sizeof(struct Slab),0);
	
		if(tmp_slab == NULL)
		{
			color_printk(RED,BLACK,"slab_malloc()->kmalloc()=>tmp_slab == NULL\n");
			return NULL;
		}
	
		memset(tmp_slab,0,sizeof(struct Slab));

		list_init(&tmp_slab->list);

		tmp_slab->page = alloc_pages(ZONE_NORMAL,1,0);
	
		if(tmp_slab->page == NULL)
		{
			color_printk(RED,BLACK,"slab_malloc()->alloc_pages()=>tmp_slab->page == NULL\n");
			kfree(tmp_slab);
			return NULL;
		}
	
		page_init(tmp_slab->page,PG_Kernel);

		tmp_slab->using_count = PAGE_2M_SIZE/slab_cache->size;
		tmp_slab->free_count = tmp_slab->using_count;
		tmp_slab->Vaddress = Phy_To_Virt(tmp_slab->page->PHY_address);

		tmp_slab->color_count = tmp_slab->free_count;
		tmp_slab->color_length = ((tmp_slab->color_count + sizeof(unsigned long) * 8 - 1) >> 6) << 3;
		tmp_slab->color_map = (unsigned long *)kmalloc(tmp_slab->color_length,0);

		if(tmp_slab->color_map == NULL)
		{
			color_printk(RED,BLACK,"slab_malloc()->kmalloc()=>tmp_slab->color_map == NULL\n");
			free_pages(tmp_slab->page,1);
			kfree(tmp_slab);
			return NULL;
		}
	
		memset(tmp_slab->color_map,0,tmp_slab->color_length);	

		list_add_to_behind(&slab_cache->cache_pool->list,&tmp_slab->list);

		slab_cache->total_free  += tmp_slab->color_count;	

		for(j = 0;j < tmp_slab->color_count;j++)
		{
			//位图匹配 j是索引 j>>6 == color_map的idx，然后 & 运算 j%64 位于第几个位
			if( (*(tmp_slab->color_map + (j >> 6)) & (1UL << (j % 64))) == 0 )
			{
				*(tmp_slab->color_map + (j >> 6)) |= 1UL << (j % 64);
			
				tmp_slab->using_count++;
				tmp_slab->free_count--;

				slab_cache->total_using++;
				slab_cache->total_free--;
				
				if(slab_cache->constructor != NULL)
				{
					return slab_cache->constructor((char *)tmp_slab->Vaddress + slab_cache->size * j,arg);
				}
				else
				{			
					return (void *)((char *)tmp_slab->Vaddress + slab_cache->size * j);
				}		
			}
		}
	}
	else
	{
		do
		{
			if(slab_p->free_count == 0)
			{
				slab_p = container_of(list_next(&slab_p->list),struct Slab,list);
				continue;
			}
		
			for(j = 0;j < slab_p->color_count;j++)
			{
				
				if(*(slab_p->color_map + (j >> 6)) == 0xffffffffffffffffUL)
				{
					j += 63;
					continue;
				}	
				
				if( (*(slab_p->color_map + (j >> 6)) & (1UL << (j % 64))) == 0 )
				{
					*(slab_p->color_map + (j >> 6)) |= 1UL << (j % 64);
				
					slab_p->using_count++;
					slab_p->free_count--;

					slab_cache->total_using++;
					slab_cache->total_free--;
					
					if(slab_cache->constructor != NULL)
					{
						return slab_cache->constructor((char *)slab_p->Vaddress + slab_cache->size * j,arg);
					}
					else
					{
						return (void *)((char *)slab_p->Vaddress + slab_cache->size * j);
					}
				}
			}
		}while(slab_p != slab_cache->cache_pool);		
	}

	color_printk(RED,BLACK,"slab_malloc() ERROR: can`t alloc\n");
	if(tmp_slab != NULL)
	{
		list_del(&tmp_slab->list);
		kfree(tmp_slab->color_map);
		page_clean(tmp_slab->page);
		free_pages(tmp_slab->page,1);
		kfree(tmp_slab);
	}

	return NULL;
}
```

函数整体来说还是比较大的，简而言之逻辑比较简单，就是先看够不够内存，如果有足够的`SLAB`,那么就用这个`slab`分配内存，而具体如何找到分配的地址，是通过`slab`的`bitmap`来进行找到的。

具体而言逻辑比较简单，需要注意的是调用了`return slab_cache->constructor((char *)slab_p->Vaddress + slab_cache->size * j,arg);`构造函数。

与之对应的`slab_free`比较简单：

```C++
unsigned long slab_free(struct Slab_cache * slab_cache,void * address,unsigned long arg)
{
	
	struct Slab * slab_p = slab_cache->cache_pool;
	int index = 0;

	do
	{
		if(slab_p->Vaddress <= address && address < slab_p->Vaddress + PAGE_2M_SIZE)
		{
			//find
			index = (address - slab_p->Vaddress) / slab_cache->size;
			*(slab_p->color_map + (index >> 6)) ^= 1UL << index % 64;
			slab_p->free_count++;
			slab_p->using_count--;

			slab_cache->total_using--;
			slab_cache->total_free++;
			
			if(slab_cache->destructor != NULL)
			{
				slab_cache->destructor((char *)slab_p->Vaddress + slab_cache->size * index,arg);
			}

			if((slab_p->using_count == 0) && (slab_cache->total_free >= slab_p->color_count * 3 / 2) /*这里可以理解为其他的slab内存池有充足的内存 大于1.5倍的一个slab内存池最大可以有的内存对象*/)
			{
				//那么就释放掉这个slab
				list_del(&slab_p->list);
				slab_cache->total_free -= slab_p->color_count;

				kfree(slab_p->color_map);
				
				page_clean(slab_p->page);
				free_pages(slab_p->page,1);
				kfree(slab_p);				
			}

			return 1;
		}
		else
		{
			slab_p = container_of(list_next(&slab_p->list),struct Slab,list);
			continue;	
		}		

	}while(slab_p != slab_cache->cache_pool);

	color_printk(RED,BLACK,"slab_free() ERROR: address not in slab\n");

	return 0;
}

```

销毁逻辑很简单，就是根据传入的`address`，找到位于`slab::VAddress`范围内存的，然后找到之后销毁即可。值得一提的是这里`if((slab_p->using_count == 0) && (slab_cache->total_free >= slab_p->color_count * 3 / 2) /*这里可以理解为其他的slab内存池有充足的内存 大于1.5倍的一个slab内存池最大可以有的内存对象*/)`这一句可以做到动态调整内存。

## 基于`slab`内存池的内核内存分配实现

前面一直看到`kmalloc`和`kfree`函数，其实底层也是借助了`slab`实现的。只不过它使用的是那16个提前申请好的全局变量。

### `slab_init`

```C++
unsigned long slab_init()
{
	struct Page * page = NULL;
	unsigned long * virtual = NULL; // get a free page and set to empty page table and return the virtual address
	unsigned long i,j;

	unsigned long tmp_address = memory_management_struct.end_of_struct;

	for(i = 0;i < 16;i++) 
	{
		kmalloc_cache_size[i].cache_pool = (struct Slab *)memory_management_struct.end_of_struct;
		memory_management_struct.end_of_struct = memory_management_struct.end_of_struct + sizeof(struct Slab) + sizeof(long) * 10;

		list_init(&kmalloc_cache_size[i].cache_pool->list);	

	//////////// init sizeof struct Slab of cache size

		kmalloc_cache_size[i].cache_pool->using_count = 0;
		kmalloc_cache_size[i].cache_pool->free_count  = PAGE_2M_SIZE / kmalloc_cache_size[i].size;

		kmalloc_cache_size[i].cache_pool->color_length =((PAGE_2M_SIZE / kmalloc_cache_size[i].size + sizeof(unsigned long) * 8 - 1) >> 6) << 3;

		kmalloc_cache_size[i].cache_pool->color_count = kmalloc_cache_size[i].cache_pool->free_count;

		kmalloc_cache_size[i].cache_pool->color_map = (unsigned long *)memory_management_struct.end_of_struct;
	
		memory_management_struct.end_of_struct = (unsigned long)(memory_management_struct.end_of_struct + kmalloc_cache_size[i].cache_pool->color_length + sizeof(long) * 10) & ( ~ (sizeof(long) - 1));

		memset(kmalloc_cache_size[i].cache_pool->color_map,0xff,kmalloc_cache_size[i].cache_pool->color_length);

		//清空位图
		for(j = 0;j < kmalloc_cache_size[i].cache_pool->color_count;j++)
			*(kmalloc_cache_size[i].cache_pool->color_map + (j >> 6)) ^= 1UL << j % 64;

		kmalloc_cache_size[i].total_free = kmalloc_cache_size[i].cache_pool->color_count;
		kmalloc_cache_size[i].total_using = 0;

	}
	
	////////////	init page for kernel code and memory management struct


	//这部分物理地址 设置被占用
	i = Virt_To_Phy(memory_management_struct.end_of_struct) >> PAGE_2M_SHIFT;

	for(j = PAGE_2M_ALIGN(Virt_To_Phy(tmp_address)) >> PAGE_2M_SHIFT;j <= i;j++)
	{
		page =  memory_management_struct.pages_struct + j;
		*(memory_management_struct.bits_map + ((page->PHY_address >> PAGE_2M_SHIFT) >> 6)) |= 1UL << (page->PHY_address >> PAGE_2M_SHIFT) % 64;
		page->zone_struct->page_using_count++;
		page->zone_struct->page_free_count--;
		page_init(page,PG_PTable_Maped | PG_Kernel_Init | PG_Kernel);
	}

	color_printk(ORANGE,BLACK,"2.memory_management_struct.bits_map:%#018lx\tzone_struct->page_using_count:%d\tzone_struct->page_free_count:%d\n",*memory_management_struct.bits_map,memory_management_struct.zones_struct->page_using_count,memory_management_struct.zones_struct->page_free_count);

	for(i = 0;i < 16;i++)
	{
		//slab内存池指向的相关内存池 这个时候也是在mms.end_of_struct里面分配
		virtual = (unsigned long *)((memory_management_struct.end_of_struct + PAGE_2M_SIZE * i + PAGE_2M_SIZE - 1) & PAGE_2M_MASK);
		page = Virt_To_2M_Page(virtual);

		*(memory_management_struct.bits_map + ((page->PHY_address >> PAGE_2M_SHIFT) >> 6)) |= 1UL << (page->PHY_address >> PAGE_2M_SHIFT) % 64;
		page->zone_struct->page_using_count++;
		page->zone_struct->page_free_count--;
		//标记
		page_init(page,PG_PTable_Maped | PG_Kernel_Init | PG_Kernel);
		//分配
		kmalloc_cache_size[i].cache_pool->page = page;
		kmalloc_cache_size[i].cache_pool->Vaddress = virtual;
	}

	color_printk(ORANGE,BLACK,"3.memory_management_struct.bits_map:%#018lx\tzone_struct->page_using_count:%d\tzone_struct->page_free_count:%d\n",*memory_management_struct.bits_map,memory_management_struct.zones_struct->page_using_count,memory_management_struct.zones_struct->page_free_count);

	color_printk(ORANGE,BLACK,"start_code:%#018lx,end_code:%#018lx,end_data:%#018lx,end_brk:%#018lx,end_of_struct:%#018lx\n",memory_management_struct.start_code,memory_management_struct.end_code,memory_management_struct.end_data,memory_management_struct.end_brk, memory_management_struct.end_of_struct);

	return 1;
}

```

逻辑比较简单，因为这个时候是为了初始化全局变量`kmalloc_cache_size`，所以现在无法使用`kmalloc`，所有的申请函数都必须是使用`alloc_page`来进行的。

其实和`slab_create`很相似，唯一区别就是这个时候没有`kmalloc`，所有内存必须先申请物理内存然后转换成虚拟地址。

### `kmalloc`

`init`之后，`kmalloc`就可以使用了

```C++
void * kmalloc(unsigned long size,unsigned long gfp_flages)
{
	int i,j;
	struct Slab * slab = NULL;
	if(size > 1048576)
	{
		color_printk(RED,BLACK,"kmalloc() ERROR: kmalloc size too long:%08d\n",size);
		return NULL;
	}
	for(i = 0;i < 16;i++)
		if(kmalloc_cache_size[i].size >= size)
			break;
	slab = kmalloc_cache_size[i].cache_pool;

	//这个if else负责找到
	if(kmalloc_cache_size[i].total_free != 0)
	{
		//
		do
		{
			if(slab->free_count == 0)
				slab = container_of(list_next(&slab->list),struct Slab,list);
			else
				break;
		}while(slab != kmalloc_cache_size[i].cache_pool);	
	}
	else
	{
		//没有空间进行分配 创建一个
		slab = kmalloc_create(kmalloc_cache_size[i].size);
		
		if(slab == NULL)
		{
			color_printk(BLUE,BLACK,"kmalloc()->kmalloc_create()=>slab == NULL\n");
			return NULL;
		}
		
		kmalloc_cache_size[i].total_free += slab->color_count;

		color_printk(BLUE,BLACK,"kmalloc()->kmalloc_create()<=size:%#010x\n",kmalloc_cache_size[i].size);///////
		
		list_add_to_before(&kmalloc_cache_size[i].cache_pool->list,&slab->list);
	}

	//具体分配逻辑和之前差不多
	for(j = 0;j < slab->color_count;j++)
	{
		if(*(slab->color_map + (j >> 6)) == 0xffffffffffffffffUL)
		{
			j += 63;
			continue;
		}
			
		if( (*(slab->color_map + (j >> 6)) & (1UL << (j % 64))) == 0 )
		{
			*(slab->color_map + (j >> 6)) |= 1UL << (j % 64);
			slab->using_count++;
			slab->free_count--;

			kmalloc_cache_size[i].total_free--;
			kmalloc_cache_size[i].total_using++;

			return (void *)((char *)slab->Vaddress + kmalloc_cache_size[i].size * j);
		}
	}

	color_printk(BLUE,BLACK,"kmalloc() ERROR: no memory can alloc\n");
	return NULL;
}
```

可以发现，其实判断用哪个`slab_cache`逻辑非常简单

```C++
	for(i = 0;i < 16;i++)
		if(kmalloc_cache_size[i].size >= size)
			break;
	slab = kmalloc_cache_size[i].cache_pool;
```

`slab`有链表，遍历即可，找到有空闲的`slab`之后，**进行分配，和`slab_malloc`逻辑非常相似。**

里面有个`kmalloc_create`专门用于创建`slab_cache`的，这里可以看一下他的算法：

```C++
struct Slab * kmalloc_create(unsigned long size)
{
	int i;
	struct Slab * slab = NULL;
	struct Page * page = NULL;
	unsigned long * vaddresss = NULL;
	long structsize = 0;

	page = alloc_pages(ZONE_NORMAL,1, 0);
	
	if(page == NULL)
	{
		color_printk(RED,BLACK,"kmalloc_create()->alloc_pages()=>page == NULL\n");
		return NULL;
	}
	
	page_init(page,PG_Kernel);

	switch(size)
	{
		////////////////////slab + map in 2M page

		case 32:
		case 64:
		case 128:
		case 256:
		case 512:

			vaddresss = Phy_To_Virt(page->PHY_address);
			structsize = sizeof(struct Slab) + PAGE_2M_SIZE / size / 8;

			//slab和内存池在同一个页面
			slab = (struct Slab *)((unsigned char *)vaddresss + PAGE_2M_SIZE - structsize);
			slab->color_map = (unsigned long *)((unsigned char *)slab + sizeof(struct Slab));

			//这样难免会损失一部分内存池的大小
			slab->free_count = (PAGE_2M_SIZE - (PAGE_2M_SIZE / size / 8) - sizeof(struct Slab)) / size;
			slab->using_count = 0;
			slab->color_count = slab->free_count;
			slab->Vaddress = vaddresss;
			slab->page = page;
			list_init(&slab->list);

			slab->color_length = ((slab->color_count + sizeof(unsigned long) * 8 - 1) >> 6) << 3;
			memset(slab->color_map,0xff,slab->color_length);

			for(i = 0;i < slab->color_count;i++)
				*(slab->color_map + (i >> 6)) ^= 1UL << i % 64;

			break;

		///////////////////kmalloc slab and map,not in 2M page anymore

		case 1024:		//1KB
		case 2048:
		case 4096:		//4KB
		case 8192:
		case 16384:

		//////////////////color_map is a very short buffer.

		case 32768:
		case 65536:
		case 131072:		//128KB
		case 262144:
		case 524288:
		case 1048576:		//1MB

			//通过kmalloc分配
			slab = (struct Slab *)kmalloc(sizeof(struct Slab),0);

			slab->free_count = PAGE_2M_SIZE / size;
			slab->using_count = 0;
			slab->color_count = slab->free_count;

			slab->color_length = ((slab->color_count + sizeof(unsigned long) * 8 - 1) >> 6) << 3;

			slab->color_map = (unsigned long *)kmalloc(slab->color_length,0);
			memset(slab->color_map,0xff,slab->color_length);

			slab->Vaddress = Phy_To_Virt(page->PHY_address);
			slab->page = page;
			list_init(&slab->list);

			for(i = 0;i < slab->color_count;i++)
				*(slab->color_map + (i >> 6)) ^= 1UL << i % 64;

			break;

		default:

			color_printk(RED,BLACK,"kmalloc_create() ERROR: wrong size:%08d\n",size);
			free_pages(page,1);
			
			return NULL;
	}	
	
	return slab;
}

```

对于小内存内存对象，位图和内存池放在同一个`2MB`之中，而对于大内存，那么则另申请内存存放位图。本质上是考虑空间利用率的问题。

### `kfree`

```C++
unsigned long kfree(void * address)
{
	int i;
	int index;
	struct Slab * slab = NULL;
	void * page_base_address = (void *)((unsigned long)address & PAGE_2M_MASK);

	for(i = 0;i < 16;i++)
	{
		slab = kmalloc_cache_size[i].cache_pool;
		do
		{
			if(slab->Vaddress == page_base_address)
			{
				//find slab
				index = (address - slab->Vaddress) / kmalloc_cache_size[i].size;

				*(slab->color_map + (index >> 6)) ^= 1UL << index % 64;

				slab->free_count++;
				slab->using_count--;

				kmalloc_cache_size[i].total_free++;
				kmalloc_cache_size[i].total_using--;

				if((slab->using_count == 0) && (kmalloc_cache_size[i].total_free >= slab->color_count * 3 / 2) && (kmalloc_cache_size[i].cache_pool != slab))
				{
					switch(kmalloc_cache_size[i].size)
					{
						////////////////////slab + map in 2M page
				
						case 32:
						case 64:
						case 128:
						case 256:	
						case 512:
							list_del(&slab->list);
							kmalloc_cache_size[i].total_free -= slab->color_count;

							page_clean(slab->page);
							free_pages(slab->page,1);
							break;
				
						default:
							list_del(&slab->list);
							kmalloc_cache_size[i].total_free -= slab->color_count;

							kfree(slab->color_map);//color map通样是kmalloc
							page_clean(slab->page);
							free_pages(slab->page,1);
							kfree(slab); //大内存 slab是用kmalloc分配的 所以需要调用kfree
							break;
					}
 
				}

				return 1;
			}
			else
				slab = container_of(list_next(&slab->list),struct Slab,list);				

		}while(slab != kmalloc_cache_size[i].cache_pool);
	
	}
	
	color_printk(RED,BLACK,"kfree() ERROR: can`t free memory\n");
	
	return 0;
}
```

kfree也比较简单，主要问题是对于不同大小的内存对象的`slab`，因为位图存放的地方不一样，所以有的需要`kfree`，有的只需要`free_pages`。

## 物理内存分配的调整

调整的地方比较少，完善了一下物理页面的属性，大体逻辑，以及Pages这个对象返回值啥的是没有改变的。

因为跑到物理机执行，所以新增加了很多物理页面的属性，符合实际。

```C++
unsigned long page_init(struct Page * page,unsigned long flags)
{
	page->attribute |= flags;
	
	if(!page->reference_count || (page->attribute & PG_Shared))
	{
		page->reference_count++;
		page->zone_struct->total_pages_link++;		
	}	
	
	return 1;
}


unsigned long page_clean(struct Page * page)
{
	page->reference_count--;
	page->zone_struct->total_pages_link--;

	if(!page->reference_count)
	{
		page->attribute &= PG_PTable_Maped;
	}
	
	return 1;
}

unsigned long get_page_attribute(struct Page * page)
{
	if(page == NULL)
	{
		color_printk(RED,BLACK,"get_page_attribute() ERROR: page == NULL\n");
		return 0;
	}
	else
		return page->attribute;
}

unsigned long set_page_attribute(struct Page * page,unsigned long flags)
{
	if(page == NULL)
	{
		color_printk(RED,BLACK,"set_page_attribute() ERROR: page == NULL\n");
		return 0;
	}
	else
	{
		page->attribute = flags;
		return 1;
	}
}
```

# chap6 APIC

## APIC概述

前面在学习了`8259A`中断控制器，而APIC可以理解为`8259A`控制器的进化版本，这是因为前者发生中断的时候每次只能投递给一个处理器核心，一旦涉及到多核环境，势必会影响中断的请求。

APIC出现改善这种情况，当然，APIC的架构更加复杂，更现代：

![image-20240511101755879](%E7%AC%94%E8%AE%B0.assets/image-20240511101755879.png)

下图表示了LAPIC/IOAPIC的结构关系示意图。可以发现，APIC是分为`I/O APIC`和`Local APIC`的，在不同的`APIC`版本中，Local APIC和`CPU`之间连接的总线不同，较新版本的APIC的`local APIC`部分之间是以系统总线进行连接的，可以说`Local APIC`则更像一个CPU的组件，与CPU连接更密切。

事实也是如此，`APIC`中`LAPIC`和`IO APIC`作用大相径庭，**而且二者的关系其实没有这么密切**，比如，你可以在不开启IO APIC的条件下单独开启`LAPIC`,外部中断使用`8259A`，完全没有问题。

下面是二者的对比：

- `Local APIC`。它可以接受来自CPU 内部的中断请求(主要是来自CPU的INTR0/1引脚连接的，比如NMI...投递到CPU的中断先经过Local APIC,然后Local APIC 根据设置，找到向量号，根据总线发送向量号到CPU)，除此之外，它还可以接受内部中断和外部IO APIC中断，如果是内部中断，Local APic根据配置转发中断向量号到CPU。外部中断，比如来自IO APIC的中断，Local APIC只负责转发
- `IO APIC`，位于主板芯片组，他有很多中断引脚，负责连接外围IO设备

其实不难发现，`LAPIC`和`IO APIC`各司其职，前者配置内部的CPU中断，包括IPIs,后者可以通过一些方法（后面会讲到）配置外围设备的`int vec`，中断触发方式等等...

## Local APIC

Local APIC更接近CPU,结构更复杂，初始化比`8259A`复杂得多。

下面来介绍LAPIC相关寄存器

### LAPIC寄存器表

LAPIC有不同的版本，这一点可以在其版本寄存器中找到，而如何找`LocalA PIC`的寄存器呢，一般是通过`MSR`寄存器或者`MMIO`映射的物理地址的。

下表就是`Local APIC`寄存器地址映射表：

![image-20240511105311869](%E7%AC%94%E8%AE%B0.assets/image-20240511105311869.png)

![image-20240511105337089](%E7%AC%94%E8%AE%B0.assets/image-20240511105337089.png)

有了上述表，开始讲述各个LAPIC寄存器的含义

### LAPIC 寄存器

- **版本寄存器**

![image-20240511110911160](%E7%AC%94%E8%AE%B0.assets/image-20240511110911160.png)

顾名思义，是Local APIC的版本。现代而言，一般都是xApic版本，这个可以不太关注。

LVT就是配置CPU内部中断的向量号啥的，LVT表项数就是表示这个Local Apic可以支持多少个，一般都是6-7个？

**这个24位，仅用来判断是否支持APIC的EOI广播，实际上设置开启在SVR寄存器中。**

- **IA32_APIC_BASE**

这个只有`xApic`版本才有，一般都有。他控制Local APIC的开启，同时配置Local APIC的物理基地址(有点像BAR)。

![image-20240511111256815](%E7%AC%94%E8%AE%B0.assets/image-20240511111256815.png)

![image-20240511111616064](%E7%AC%94%E8%AE%B0.assets/image-20240511111616064.png)

一般来说，上电之后，APIC的BAR就是`0xFEE00000`默认值。

BSP：代表是引导时候的CPU，EN就是开启Local APIC的使能位。

至于如何判断是否支持APIC,那么毫无疑问肯定是通过`cpuid`了,通过CPUID.01.ECX[21]可以判断是否支持x2Apic.

- Local APIC ID寄存器

这个还是很重要的，APIC ID==CPU 核心ID，CPU上电之后，这个就不变了，作为CPU核心的ID号使用。

![image-20240511111723430](%E7%AC%94%E8%AE%B0.assets/image-20240511111723430.png)

除了这些重要寄存器，Local APIC还有很多其他的重要寄存器：

![image-20240511112419310](%E7%AC%94%E8%AE%B0.assets/image-20240511112419310.png)

### LVT

![image-20240511153724381](%E7%AC%94%E8%AE%B0.assets/image-20240511153724381.png)

相关功能位如下：

![image-20240511153743693](%E7%AC%94%E8%AE%B0.assets/image-20240511153743693.png)

LVT主要是用于Local APIC配置CPU内部中断的表，可以说Local APIC可以管的中断可编程除了优先级就是这个了，总的来说还是很重要的。

当然还有其他位域，比如投递模式，这个也很重要，值得一提的是`NMI`投递模式，这个一旦设置，自动忽略向量号，向量号固定是2（NMI中断）。

![image-20240511155125859](%E7%AC%94%E8%AE%B0.assets/image-20240511155125859.png)

### 中断优先级

Local APIC以中断向量号作为优先级，简而言之就是向量号越大，中断优先级越大，优先级越大，可以把优先级低的打断。

书上描写的非常复杂，分为**任务优先权**和**处理器优先权**两个寄存器

![image-20240511154055449](%E7%AC%94%E8%AE%B0.assets/image-20240511154055449.png)

![image-20240511154102099](%E7%AC%94%E8%AE%B0.assets/image-20240511154102099.png)

这两个寄存器都是8位，1B，其实就是一句话：

**任务优先权就是中断的阈值，这个里面的值代表了当前cpu所能被APIC投递的最低中断级别。其中，任务优先权这个寄存器的[8:4]被映射到了cr8，也就是直接通过写cr8就可以写任务优先级。而处理器优先权则是当前处理的中断的中断向量号，肯定是>=任务优先权这个阈值的**

![image-20240511154346477](%E7%AC%94%E8%AE%B0.assets/image-20240511154346477.png)

而windows把中断优先权抽象为irql，取值为0~15，这正是cr8的值，其实就是任务优先权等级，**也就是TPR[7:4]**，并且在这个基础上，不同的irql对应不同的场景。

### LAPIC中断的投递

- **IRR、ISR寄存器**

在设置完LVT之后，正常的`Fixed`模式投递情况下，中断请求一般是放在`IRR(Interupt Reuqest Register)`和`ISR`中，其他的投递模式不会经过`IRR 和 ISR`。这两个寄存器结构如下

![image-20240511155335154](%E7%AC%94%E8%AE%B0.assets/image-20240511155335154.png)

IRR,ISR这两个概念在`8259A`就已经出现过了，简而言之就是中断触发先放到IRR,如果此时可以投递中断(优先级够)，那么就会复位IRR,置位ISR。处理完毕（一般是等待cpu发送EOI），复位ISR,等待下一个中断请求。

这里还有个TMR寄存器，这个没啥用，主要是和触发模式（电平、边沿触发）有关。

- **EOI寄存器**

这个寄存器LAPIC有，IO APIC也有，除了`NMI SMI INTX ExiINT`触发模式，中断后必须明确写入`EOI`，通常是`IRET`这个指令执行之前完成的。写完这个，LAPIC才会复位`ISR`，才会进行下一个中断的投递。

![image-20240511155922379](%E7%AC%94%E8%AE%B0.assets/image-20240511155922379.png)

一般而言，如果在`IA32_APIC_BASE`没有关闭EOI广播，那么使用电平触发的中断，Local APIC接受到EOI,会广播到IO APIC。

而软件（os）则偏向直接将EOI投递到IO APIC,而不是借助广播。

- **SVR寄存器**

![image-20240511160233034](%E7%AC%94%E8%AE%B0.assets/image-20240511160233034.png)

它主要决定了电平触发的中断是否广播EOI，默认是0，是广播的。

下面是各位的功能介绍：

![image-20240511160500727](%E7%AC%94%E8%AE%B0.assets/image-20240511160500727.png)

## IO APIC

介绍完了Local APIC，接下来赶来的是IO APIC。正如前面所说，Local APIC和IO APIC虽然都属于APIC结构，但是二者联系却没有想象的那么大，比如你可以只开启Local APIC，而IO中断使用8259A(前提是主板兼容)，甚至是不开启IO APIC。

而IO APIC主要是收集IO 设备的中断请求，然后封装成终端消息到目标处理器的`Local APIC`。

我们可以配置的是外围设备的中断向量号（通过一个表，和LVT很相像）。

下面开始介绍IO APIC：

### IO APIC间接访问寄存器

从前面的架构可以知道，IO APIC位于主板上，有也就导致了它不像Local APIC那样，而是通过一个索引寄存器进行间接访问`IO APIC`的各个寄存器。

> ps：这种访问方法要多留意，因为很多主板上面的访问都是通过这种间接方式来访问

下面是IO APIC的间接访问寄存器：

![image-20240511161639788](%E7%AC%94%E8%AE%B0.assets/image-20240511161639788.png)

这个`OIC`寄存器是专门配置`MMIO`的，有点像`BAR（Base Address Reigser）`的功能，这里先不管，后面代码部分会介绍                                                                                                                                                   IC的间接访问寄存器，`OIC`默认是0x0.

下面来介绍这种间接索引寄存器的方法：

首先你需要往``写一个APIC寄存器的索引，表如下：

![image-20240511162049112](%E7%AC%94%E8%AE%B0.assets/image-20240511162049112.png)

写入之后，你就可以在`I/O Windows`读写这个值了，比如我想获取IO APIC的ID寄存器，那么我需要往这个里面写入0，然后等着获取就行了。

```C++
*IOREGSEL=0;
auto id=*IOWINDOW;
```

而这个window寄存器是32位的，读写64位的IO APIC寄存器，有时候还得读写两次。

### IO APIC寄存器介绍

下面介绍IO APIC的各个寄存器

- ID寄存器

![image-20240511162828035](%E7%AC%94%E8%AE%B0.assets/image-20240511162828035.png)

- **版本寄存器**

![image-20240511162857131](%E7%AC%94%E8%AE%B0.assets/image-20240511162857131.png)

RTE（Redirection Table）就是前面一直提到的，和LVT差不多的玩意，这个是用来设置外部IO设备连接到IO APIC中断引脚的部分的中断向量号，投递方式...

RET数+1=实际上可以用的RET数量。

- **RTE**

![image-20240511163016686](%E7%AC%94%E8%AE%B0.assets/image-20240511163016686.png)

这里简单地介绍一下各个位

- 目标模式

![image-20240511163142006](%E7%AC%94%E8%AE%B0.assets/image-20240511163142006.png)

- 投递目标

![image-20240511163326763](%E7%AC%94%E8%AE%B0.assets/image-20240511163326763.png)

不同的目标模式，可以让IO APIC投递到不同的Local APIC，这一点先按下不表，因为后面多核才涉及到这个，目前我们一律按照单核考虑。

至此，我们知道，IOAPIC其实就是不停低将外部I/O设备发送的中断请求封装成中断消息，然后发给Local APIC，Local APIC投递给CPU,CPU此时接受到中断向量号，根据IDT来进行中断。下面这张图是APIC的整体结构：

![image-20240511163535401](%E7%AC%94%E8%AE%B0.assets/image-20240511163535401.png)

而IO APIC中断源各引脚如下：

![image-20240511163944101](%E7%AC%94%E8%AE%B0.assets/image-20240511163944101.png)

## APIC初始化

这里参考原文和linux(github最新版：2024/5/14版本)的代码来进行查看，linux/mineos到底是如何初始化`local apic`和`io apic`的。

这里`APIC`我们一般选择`Symmetirc IO`模式，具体处理模式这里不详细解释了。

![image-20240514110349139](%E7%AC%94%E8%AE%B0.assets/image-20240514110349139.png)

### Local APIC Setup

首先是`Local APIC`的初始化，这个比较简单，只需要通过cpuid判断是否支持`APIC`还是`xAPIC/x2APIC`就好，然后通过设置`IA32_APIC_BASE`开启`Local APIC`

```C++
	//check APIC & x2APIC support
	get_cpuid(1,0,&a,&b,&c,&d);
	//void get_cpuid(unsigned int Mop,unsigned int Sop,unsigned int * a,unsigned int * b,unsigned int * c,unsigned int * d)
	color_printk(WHITE,BLACK,"CPUID\t01,eax:%#010x,ebx:%#010x,ecx:%#010x,edx:%#010x\n",a,b,c,d);

	if((1<<9) & d)
		color_printk(WHITE,BLACK,"HW support APIC&xAPIC\t");
	else
		color_printk(WHITE,BLACK,"HW NO support APIC&xAPIC\t");
	
	if((1<<21) & c)
		color_printk(WHITE,BLACK,"HW support x2APIC\n");
	else
		color_printk(WHITE,BLACK,"HW NO support x2APIC\n");
//enable xAPIC & x2APIC
	__asm__ __volatile__(	"movq 	$0x1b,	%%rcx	\n\t"
				"rdmsr	\n\t"
				"bts	$10,	%%rax	\n\t"
				"bts	$11,	%%rax	\n\t"
				"wrmsr	\n\t"
				"movq 	$0x1b,	%%rcx	\n\t"
				"rdmsr	\n\t"
				:"=a"(x),"=d"(y)
				:
				:"memory");
```

初步开启了`local APIC`，我们需要判断一下`APIC`的版本，是集成的，还是和cpu分离的。

```C++
//get local APIC version
	__asm__ __volatile__(	"movq $0x803,	%%rcx	\n\t"
				"rdmsr	\n\t"
				:"=a"(x),"=d"(y)
				:
				:"memory");

	color_printk(WHITE,BLACK,"local APIC Version:%#010x,Max LVT Entry:%#010x,SVR(Suppress EOI Broadcast):%#04x\t",x & 0xff,(x >> 16 & 0xff) + 1,x >> 24 & 0x1);

	if((x & 0xff) < 0x10)
		color_printk(WHITE,BLACK,"82489DX discrete APIC\n");

	else if( ((x & 0xff) >= 0x10) && ((x & 0xff) <= 0x15) )
		color_printk(WHITE,BLACK,"Integrated APIC\n");
```

然后就是初始化LVT,设置中断向量号，这也是`Local APIC`最重要的功能之一——配置CPU内部中断

```C++
	//mask all LVT	
	__asm__ __volatile__(	"movq 	$0x82f,	%%rcx	\n\t"	//CMCI
				"wrmsr	\n\t"
				"movq 	$0x832,	%%rcx	\n\t"	//Timer
				"wrmsr	\n\t"
				"movq 	$0x833,	%%rcx	\n\t"	//Thermal Monitor
				"wrmsr	\n\t"
				"movq 	$0x834,	%%rcx	\n\t"	//Performance Counter
				"wrmsr	\n\t"
				"movq 	$0x835,	%%rcx	\n\t"	//LINT0
				"wrmsr	\n\t"
				"movq 	$0x836,	%%rcx	\n\t"	//LINT1
				"wrmsr	\n\t"
				"movq 	$0x837,	%%rcx	\n\t"	//Error
				"wrmsr	\n\t"
				:
				:"a"(0x10000),"d"(0x00)
				:"memory");

	color_printk(GREEN,BLACK,"Mask ALL LVT\n");

	//TPR
	__asm__ __volatile__(	"movq 	$0x808,	%%rcx	\n\t"
				"rdmsr	\n\t"
				:"=a"(x),"=d"(y)
				:
				:"memory");

	color_printk(GREEN,BLACK,"Set LVT TPR:%#010x\t",x);

	//PPR
	__asm__ __volatile__(	"movq 	$0x80a,	%%rcx	\n\t"
				"rdmsr	\n\t"
				:"=a"(x),"=d"(y)
				:
				:"memory");

	color_printk(GREEN,BLACK,"Set LVT PPR:%#010x\n",x);
```

可以发现，设置成了`10000`，中断屏蔽。然后设置TPR和PPR，开启中断，其实目前是让我们只接受来自`IO APIC`的中断，那么我们来看`IO APIC`的初始化

### IO APIC setup

首先是要设置了`IDT`,屏蔽了8259A，然后设置IDT,至于为什么是从32开始，这个我们可以设置的，只需要设置`RTE`就可以了。

```C++
void APIC_IOAPIC_init()
{
	//	init trap abort fault
	int i ;

	for(i = 32;i < 56;i++)
	{
		set_intr_gate(i , 2 , interrupt[i - 32]);
	}

	//mask 8259A
	color_printk(GREEN,BLACK,"MASK 8259A\n");
	io_out8(0x21,0xff);
	io_out8(0xa1,0xff);

	//init local apic
	Local_APIC_init();

	//enable IF eflages
	sti();
}
```

前面我们知道，IO APIC的读取是通过`window`寄存器，然后通过偏移来进行读取的。

而具体去哪里（哪个物理地址读），其实是通过`OIC`寄存器进行设置的。

当然你也可以不找`OIC`寄存器，默认是0，所以`Window`的默认地址是`0xFFEC0000`，当然最好先找一下，这个找需要从`chipset data sheet`里面找。

具体而言：

> 通过`RCBA`寄存器（位于PCI总线上某个设备的寄存器，这个通过主板白皮书找）找到芯片组配置寄存器，而OIC位于芯片组配置寄存器的`31FE-31EE`偏移（这个似乎是约定成俗的，OIC寄存器就是控制IO APIC的，但是还是需要通过主板确定一下偏移）。

上述两个寄存器：

![image-20240514113223825](%E7%AC%94%E8%AE%B0.assets/image-20240514113223825.png)

![image-20240514113236558](%E7%AC%94%E8%AE%B0.assets/image-20240514113236558.png)

具体而言，文中首先需要映射一下最主要的地址，所以逻辑相关代码如下：

```C++
void IOAPIC_pagetable_remap()
{
	//似乎默认OIC寄存器是0?
	unsigned long * tmp;
	unsigned char * IOAPIC_addr = (unsigned char *)Phy_To_Virt(0xfec00000);


	ioapic_map.physical_address = 0xfec00000;
	ioapic_map.virtual_index_address  = IOAPIC_addr;
	ioapic_map.virtual_data_address   = (unsigned int *)(IOAPIC_addr + 0x10);
	ioapic_map.virtual_EOI_address    = (unsigned int *)(IOAPIC_addr + 0x40);
	
	Global_CR3 = Get_gdt();

	//获取对应的PML4E项目
	tmp = Phy_To_Virt(Global_CR3 + (((unsigned long)IOAPIC_addr >> PAGE_GDT_SHIFT) & 0x1ff));
	if (*tmp == 0)
	{
		//设置PDPT
		unsigned long * virtual = kmalloc(PAGE_4K_SIZE,0);
		set_mpl4t(tmp,mk_mpl4t(Virt_To_Phy(virtual),PAGE_KERNEL_GDT));
	}

	color_printk(YELLOW,BLACK,"1:%#018lx\t%#018lx\n",(unsigned long)tmp,(unsigned long)*tmp);

	//获取PDPTE
	tmp = Phy_To_Virt((unsigned long *)(*tmp & (~ 0xfffUL)) + (((unsigned long)IOAPIC_addr >> PAGE_1G_SHIFT) & 0x1ff));
	if(*tmp == 0)
	{
		//设置PDT
		unsigned long * virtual = kmalloc(PAGE_4K_SIZE,0);
		set_pdpt(tmp,mk_pdpt(Virt_To_Phy(virtual),PAGE_KERNEL_Dir));
	}

	color_printk(YELLOW,BLACK,"2:%#018lx\t%#018lx\n",(unsigned long)tmp,(unsigned long)*tmp);
	
	//大页面分页 获取PDT
	tmp = Phy_To_Virt((unsigned long *)(*tmp & (~ 0xfffUL)) + (((unsigned long)IOAPIC_addr >> PAGE_2M_SHIFT) & 0x1ff));
	
	//设置物理地址 remap
	set_pdt(tmp,mk_pdt(ioapic_map.physical_address,PAGE_KERNEL_Page | PAGE_PWT | PAGE_PCD));

	color_printk(BLUE,BLACK,"3:%#018lx\t%#018lx\n",(unsigned long)tmp,(unsigned long)*tmp);

	color_printk(BLUE,BLACK,"ioapic_map.physical_address:%#010x\t\t\n",ioapic_map.physical_address);
	color_printk(BLUE,BLACK,"ioapic_map.virtual_address:%#018lx\t\t\n",(unsigned long)ioapic_map.virtual_index_address);

	flush_tlb();
}
```

主要就是申请内存，然后把要映射的最重要的`IOAPIC`物理地址，比如`index`，`data`,`eoi`映射，映射的代码比较简单，就是移位找到相关的PML4...的索引，然后插入物理页即可。

然后文中又实现了`ioapic_rte_read/write`，这一点和linux的`ioapic_write/read_entry`本质上是一样的，都是更方便。

```C++
unsigned long ioapic_rte_read(unsigned char index)
{
	unsigned long ret;

	*ioapic_map.virtual_index_address = index + 1;
	io_mfence();
	ret = *ioapic_map.virtual_data_address;
	ret <<= 32;
	io_mfence();

	*ioapic_map.virtual_index_address = index;		
	io_mfence();
	ret |= *ioapic_map.virtual_data_address;
	io_mfence();

	return ret;
}

/*

*/

void ioapic_rte_write(unsigned char index,unsigned long value)
{
	*ioapic_map.virtual_index_address = index;
	io_mfence();
	*ioapic_map.virtual_data_address = value & 0xffffffff;
	value >>= 32;
	io_mfence();
	
	*ioapic_map.virtual_index_address = index + 1;
	io_mfence();
	*ioapic_map.virtual_data_address = value & 0xffffffff;
	io_mfence();
}
```

`io_mfence`可以保证处理器不会乱序执行，在这种严格要求谁先谁后的代码比较有用。

而参考linux内核提到，必须先写`rte`的高位，因为先写低位会直接触发中断

```C++
static void __ioapic_write_entry(int apic, int pin, struct IO_APIC_route_entry e)
{
	io_apic_write(apic, 0x11 + 2*pin, e.w2);//先写RTE高位
	io_apic_write(apic, 0x10 + 2*pin, e.w1);
}

static void ioapic_write_entry(int apic, int pin, struct IO_APIC_route_entry e)
{
	unsigned long flags;

	raw_spin_lock_irqsave(&ioapic_lock, flags);
	__ioapic_write_entry(apic, pin, e);
	raw_spin_unlock_irqrestore(&ioapic_lock, flags);
}
```

`io_apic_write`定义如下

```C++
static void io_apic_write(unsigned int apic, unsigned int reg,
			  unsigned int value)
{
	struct io_apic __iomem *io_apic = io_apic_base(apic);

	writel(reg, &io_apic->index);
	writel(value, &io_apic->data);
}
```

可以看到，其实`linux`也是如法炮制(其实是书中炒的linux)，把io_apic的一些玩意映射到了一个结构，而且是每个cpu一个（linux初始化考虑多核），而linux为了解决读写IO地址的先后顺序严格性，专门写了一个`writel`函数，设置了内存屏障：

```C++
extern inline void writel(u32 b, volatile void __iomem *addr)
{
	mb();
	__raw_writel(b, addr);
}
```

这样就确保了

```C++
	writel(reg, &io_apic->index);
	writel(value, &io_apic->data);
```

上面先执行，下面后执行，也就是先读后写。

然后就是设置`RTE`，开启中断，代码如下：

```C++
void IOAPIC_init()
{
	int i ;
	//	I/O APIC
	//	I/O APIC	ID	
	*ioapic_map.virtual_index_address = 0x00;
	io_mfence();
	*ioapic_map.virtual_data_address = 0x0f000000;
	io_mfence();
	color_printk(GREEN,BLACK,"Get IOAPIC ID REG:%#010x,ID:%#010x\n",*ioapic_map.virtual_data_address, *ioapic_map.virtual_data_address >> 24 & 0xf);
	io_mfence();

	//	I/O APIC	Version
	*ioapic_map.virtual_index_address = 0x01;
	io_mfence();
	color_printk(GREEN,BLACK,"Get IOAPIC Version REG:%#010x,MAX redirection enties:%#08d\n",*ioapic_map.virtual_data_address ,((*ioapic_map.virtual_data_address >> 16) & 0xff) + 1);

	//RTE	
	for(i = 0x10;i < 0x40;i += 2)
		ioapic_rte_write(i,0x10020 + ((i - 0x10) >> 1)); //设置RTE 中断向量号从0x21开始增加

	ioapic_rte_write(0x12,0x21);
	color_printk(GREEN,BLACK,"I/O APIC Redirection Table Entries Set Finished.\n");	
}

/*

*/

void APIC_IOAPIC_init()
{
	//	init trap abort fault
	int i ;
	unsigned int x;
	unsigned int * p;

	IOAPIC_pagetable_remap();

	for(i = 32;i < 56;i++)
	{
		set_intr_gate(i , 2 , interrupt[i - 32]);
	}

	//mask 8259A
	color_printk(GREEN,BLACK,"MASK 8259A\n");
	io_out8(0x21,0xff);
	io_out8(0xa1,0xff);

	//enable IMCR
	io_out8(0x22,0x70);
	io_out8(0x23,0x01);	
	
	//init local apic
	Local_APIC_init();

	//init ioapic
	IOAPIC_init();

	//get RCBA address
	io_out32(0xcf8,0x8000f8f0);
	x = io_in32(0xcfc);
	color_printk(RED,BLACK,"Get RCBA Address:%#010x\n",x);	
	x = x & 0xffffc000;
	color_printk(RED,BLACK,"Get RCBA Address:%#010x\n",x);	

	//get OIC address
	if(x > 0xfec00000 && x < 0xfee00000)
	{
		p = (unsigned int *)Phy_To_Virt(x + 0x31feUL);
	}

	//enable IOAPIC
	x = (*p & 0xffffff00) | 0x100;
	io_mfence();
	*p = x;
	io_mfence();

	//enable IF eflages
	sti();
}

/*

*/

void do_IRQ(struct pt_regs * regs,unsigned long nr)	//regs:rsp,nr
{
	unsigned char x;

	x = io_in8(0x60);	
	color_printk(BLUE,WHITE,"(IRQ:%#04x)\tkey code:%#04x\n",nr,x);

	__asm__ __volatile__(	"movq	$0x00,	%%rdx	\n\t"
				"movq	$0x00,	%%rax	\n\t"
				"movq 	$0x80b,	%%rcx	\n\t"
				"wrmsr	\n\t"
				:::"memory");
}
```

这里有一个知识点是，`RCBA`寄存器位于`PCI`总线第`31`号设备的`F0`功能/偏移，计算方法是：

```C++
#include <linux/io.h>  // for outl and inl

#define PCI_CONFIG_ADDRESS 0xCF8
#define PCI_CONFIG_DATA    0xCFC

u32 pci_config_read_dword(unsigned int bus, unsigned int devfn, int where)
{
    u32 address;
    u32 value;

    // 构建配置地址
    address = (1 << 31) | (bus << 16) | (devfn << 8) | (where & 0xfc);

    // 写入配置地址寄存器
    outl(address, PCI_CONFIG_ADDRESS);

    // 从配置数据寄存器读取数据
    value = inl(PCI_CONFIG_DATA);
    
    return value;
}

```

这个方式读写比较老，是通过io端口读写的，现在linux似乎是通过内存映射间接读写的(好像都有)，我在linux内核找到了如下代码进行读写

```C++
/*
 * Functions for accessing PCI base (first 256 bytes) and extended
 * (4096 bytes per PCI function) configuration space with type 1
 * accesses.
 */

#define PCI_CONF1_ADDRESS(bus, devfn, reg) \
	(0x80000000 | ((reg & 0xF00) << 16) | (bus << 16) \
	| (devfn << 8) | (reg & 0xFC))

static int pci_conf1_read(unsigned int seg, unsigned int bus,
			  unsigned int devfn, int reg, int len, u32 *value)
{
	unsigned long flags;

	if (seg || (bus > 255) || (devfn > 255) || (reg > 4095)) {
		*value = -1;
		return -EINVAL;
	}

	raw_spin_lock_irqsave(&pci_config_lock, flags);

	outl(PCI_CONF1_ADDRESS(bus, devfn, reg), 0xCF8);

	switch (len) {
	case 1:
		*value = inb(0xCFC + (reg & 3));
		break;
	case 2:
		*value = inw(0xCFC + (reg & 2));
		break;
	case 4:
		*value = inl(0xCFC);
		break;
	}

	raw_spin_unlock_irqrestore(&pci_config_lock, flags);

	return 0;
}
```

可以看到，对于`PCI`总线上某个设备的某个设备号某个功能，linux是这样定位的

```C++
#define PCI_CONF1_ADDRESS(bus, devfn, reg) \
	(0x80000000 | ((reg & 0xF00) << 16) | (bus << 16) \
	| (devfn << 8) | (reg & 0xFC))
```

下面视角继续转到这本书，看一下`mineos`是如何找到`OIC`来进行开启`IO APIC enable`标志位的。找到`OIC`：

```C++
	//get RCBA address
	io_out32(0xcf8,0x8000f8f0);
	x = io_in32(0xcfc);
	color_printk(RED,BLACK,"Get RCBA Address:%#010x\n",x);	
	x = x & 0xffffc000;
	color_printk(RED,BLACK,"Get RCBA Address:%#010x\n",x);	

	//get OIC address
	if(x > 0xfec00000 && x < 0xfee00000)
	{
		p = (unsigned int *)Phy_To_Virt(x + 0x31feUL); //物理地址转换成虚拟地址 31EF是OIC在RCBA的偏移
	}

	//enable IOAPIC
	x = (*p & 0xffffff00) | 0x100;
	io_mfence();
	*p = x;
	io_mfence();
```

来开启IO APIC。

至此，`IO APIC`和`Local APIC`皆以初始化完成，我们对于`APIC`理解可能还不够全面，但是大体知道了工作原理、初始化。后续我们将会根据这个来写一个键盘鼠标驱动了。

## 高级中断处理

所谓的高级中断处理，对于`mineos`来说，目前只是增加了一个可以注册中断的接口：

```C++
unsigned long IOAPIC_install(unsigned long irq,void * arg)
{
	struct IO_APIC_RET_entry *entry = (struct IO_APIC_RET_entry *)arg;
	ioapic_rte_write((irq - 32) * 2 + 0x10,*(unsigned long *)entry);

	return 1;
}
```

```C++
int register_irq(unsigned long irq,
		void * arg,
		void (*handler)(unsigned long nr, unsigned long parameter, struct pt_regs * regs),
		unsigned long parameter,
		hw_int_controller * controller,
		char * irq_name)
{	
	irq_desc_T * p = &interrupt_desc[irq - 32];
	
	p->controller = controller;
	p->irq_name = irq_name;
	p->parameter = parameter;
	p->flags = 0;
	p->handler = handler;

	p->controller->install(irq,arg);
	p->controller->enable(irq);
	
	return 1;
}
int unregister_irq(unsigned long irq)
{
	irq_desc_T * p = &interrupt_desc[irq - 32];

	p->controller->disable(irq);
	p->controller->uninstall(irq);

	p->controller = NULL;
	p->irq_name = NULL;
	p->parameter = NULL;
	p->flags = 0;
	p->handler = NULL;

	return 1; 
}
```

而既然注册了中断，那么`do_irq`也需要改：

```C++
void do_IRQ(struct pt_regs * regs,unsigned long nr)	//regs:rsp,nr
{
	unsigned char x;
	irq_desc_T * irq = &interrupt_desc[nr - 32];

	x = io_in8(0x60);	
	color_printk(BLUE,WHITE,"(IRQ:%#04x)\tkey code:%#04x\n",nr,x);

	if(irq->handler != NULL)
		irq->handler(nr,irq->parameter,regs);

	if(irq->controller != NULL && irq->controller->ack != NULL)
		irq->controller->ack(nr);

	__asm__ __volatile__(	"movq	$0x00,	%%rdx	\n\t"
				"movq	$0x00,	%%rax	\n\t"
				"movq 	$0x80b,	%%rcx	\n\t"
				"wrmsr	\n\t"
				:::"memory");
}
```

```C++
	__asm__ __volatile__(	"movq	$0x00,	%%rdx	\n\t"
				"movq	$0x00,	%%rax	\n\t"
				"movq 	$0x80b,	%%rcx	\n\t"
				"wrmsr	\n\t"
				:::"memory");
```

最后这个部分，把`80B` `wrmsr`写入0，代表处理了中断，发送`EOI`消息。

这里可能会疑惑，为啥只往`Local APIC`的`EOI`写0，其实不用疑惑，我们设置了`EOI`广播，`Local APIC`的`EOI`会广播到主板的`I/O APIC`。

> **简而言之，过去的PIC是直接连接到CPU的INTR中断引脚的。而更现代架构的PIC，也就是APIC，细分成了I/O APIC和LAPIC，他们各司其职，LAPIC可以通过LVT配置本地的中断，连接到CPU的INTR。而同时I/O APIC可以通过配置RTE配置外部连接I/O APIC引脚的中断，同时通过总线连接LAPIC（通过ICC BUS，专门给LAPIC传输中断信号的），然后LAPIC 通过INTR像CPU触发中断。**

# chap7 驱动

驱动是操作系统一个重要的概念。一般的os都会提供成熟的模型，驱动开发者按照对应模型去开发驱动，然后按照OS约定去加载驱动到内核一部分。可以理解为驱动是内核可扩展的一部分，其主要作用就是为了管理不同的真实、抽象的外设，可以理解驱动就是最了解外围物理设备的协议、使用的内核的一部分。

## 键/鼠驱动

这里把键鼠放在一块，是因为老式的键鼠接口比如PS/2接口的他们都会用到配置8042，相关配置很像。

### 8042键盘控制器

8042是古早的南桥时期的产物，当然现在北桥->cpu，南桥->芯片组PCH中。

虽然叫做键盘控制器，但是8042既可以初始化键盘，还连接着鼠标，甚至还掌管着A20地址线的开启（**至于为什么，下文会介绍**）。

其结构示意图如下：

![image-20240526085554080](%E7%AC%94%E8%AE%B0.assets/image-20240526085554080.png)

可以看到，8042是连接到PIC的`IRQ1`和`IRQ12`的。

- **8042端口**

8042键盘控制器又叫做8042端口设备，其实暗含了他是一个通过IO端口来进行控制、操作的设备。相关操作端口如下：

![image-20240526085730163](%E7%AC%94%E8%AE%B0.assets/image-20240526085730163.png)

8042最常用的是`60h`和`64h`端口，这两个端口即可读也可写，简单总结一下：

| Port | W                                | R                                     |
| ---- | -------------------------------- | ------------------------------------- |
| 60h  | 向8042发送控制指令附带的传递参数 | 8042控制器返回的参数或者键盘scan code |
| 64h  | 向8042发送控制指令               | 8042的状态信息                        |

相关的控制指令如下：

![image-20240526090457993](%E7%AC%94%E8%AE%B0.assets/image-20240526090457993.png)

比如我想使能键盘IRQ1的中断（其实这个是使能8042的INT引脚），我就可以

```assembly
out 64h,60h
out 60h,1h;使能键盘中断
```

主要是`20h`和`60h`比较重要，同时`c0`、`D0`、`D1`是三个端口，主要是是8042相关引脚，这里首先来介绍一下不同的引脚

> 1. **电源引脚（Power Pins）**：
>    - **Vcc/Vdd**：为芯片提供工作电压的正极引脚。
>    - **Vss/GND**：为芯片提供工作电压的负极引脚或接地引脚。
> 2. **输入引脚（Input Pins）**：
>    - 接收外部信号或数据的引脚。这些引脚用于将外部的模拟或数字信号输入到芯片内部进行处理。
> 3. **输出引脚（Output Pins）**：
>    - 输出信号或数据的引脚。这些引脚用于将芯片内部处理后的信号或数据发送到外部电路。
> 4. **输入/输出引脚（I/O Pins）**：
>    - 可以配置为输入或输出模式的引脚。这类引脚通常用于通信接口或可编程逻辑器件。
> 5. **控制引脚（Control Pins）**：
>    - 用于控制芯片工作模式的引脚，包括但不限于复位引脚（Reset Pin）、中断引脚（Interrupt Pin）、片选引脚（Chip Select Pin）等。
> 6. **模拟引脚（Analog Pins）**：
>    - 处理模拟信号的引脚。这些引脚用于模拟信号的输入和输出，如模拟传感器信号、模拟输出等。
> 7. **时钟引脚（Clock Pins）**：
>    - 提供时钟信号的引脚。这些引脚用于给芯片提供时钟信号，以同步内部电路的工作。
> 8. **编程引脚（Programming Pins）**：
>    - 用于芯片编程或调试的引脚，如JTAG引脚、ISP引脚等。

我们比较感兴趣的是P2里面的几个控制引脚

![image-20240526091054426](%E7%AC%94%E8%AE%B0.assets/image-20240526091054426.png)

比如我们可以通过`D1h`来手动开启A20地址线

```C++
out(0x64,0xD1);
out(0x60,0xDF/*开启A20地址线,最低一位是1*/);
```

而其实正是因为当时在设计的时候，**`A20`地址线的开启和8042有关系是因为恰好`8042`有一个空闲的控制引脚。**

### 键盘驱动

通过PS/2接口连接的键盘比较简单，现代一般都是USB键盘， 一般是USB键盘控制器发送URB给USB控制器，USB控制器连接IO APIC->LAPIC这个触发过程。

windows有通用的USB键盘驱动（一般的OS肯定都是有通用的USB键盘驱动）

![image-20240526100434081](%E7%AC%94%E8%AE%B0.assets/image-20240526100434081.png)

![image-20240526100454766](%E7%AC%94%E8%AE%B0.assets/image-20240526100454766.png)

负责解析，而本书中`mineos`，只考虑了ps/2接口的键盘。

这种实在过于落后，因此我打算自己写个USB键鼠驱动，学一下USB的HID协议和发送URB。

因此这里简单地过一下：

首先是初始化键盘中断函数。

```C++
void keyboard_init()
{
	struct IO_APIC_RET_entry entry;
	unsigned long i,j;

	p_kb = (struct keyboard_inputbuffer *)kmalloc(sizeof(struct keyboard_inputbuffer),0);
	
	p_kb->p_head = p_kb->buf;
	p_kb->p_tail = p_kb->buf;
	p_kb->count  = 0;
	memset(p_kb->buf,0,KB_BUF_SIZE);

	entry.vector = 0x21;
	entry.deliver_mode = APIC_ICR_IOAPIC_Fixed ;
	entry.dest_mode = ICR_IOAPIC_DELV_PHYSICAL;
	entry.deliver_status = APIC_ICR_IOAPIC_Idle;
	entry.polarity = APIC_IOAPIC_POLARITY_HIGH;
	entry.irr = APIC_IOAPIC_IRR_RESET;
	entry.trigger = APIC_ICR_IOAPIC_Edge;
	entry.mask = APIC_ICR_IOAPIC_Masked;
	entry.reserved = 0;

	entry.destination.physical.reserved1 = 0;
	entry.destination.physical.phy_dest = 0;
	entry.destination.physical.reserved2 = 0;

	//等输出缓冲区完毕 因为外围设备太慢 60是配置8042的
	wait_KB_write();
	io_out8(PORT_KB_CMD,KBCMD_WRITE_CMD);
	wait_KB_write();
	//配置8042 0x47
	io_out8(PORT_KB_DATA,KB_INIT_MODE);

	//键盘控制器完成 因为比较慢
	for(i = 0;i<1000;i++)
		for(j = 0;j<1000;j++)
			nop();
	
	shift_l = 0;
	shift_r = 0;
	ctrl_l  = 0;
	ctrl_r  = 0;
	alt_l   = 0;
	alt_r   = 0;

	//去注册irq
	register_irq(0x21, &entry , &keyboard_handler, (unsigned long)p_kb, &keyboard_int_controller, "ps/2 keyboard");
}
```

里面的有些细节需要注意：

1. ```C++
   //去注册irq
   	register_irq(0x21, &entry , &keyboard_handler, (unsigned long)p_kb, &keyboard_int_controller, "ps/2 keyboard");
   //注册的ISR为keyboard_handler
   ```

2. ```C++
   //等输出缓冲区完毕 因为外围设备太慢 60是配置8042的
   	wait_KB_write();
   	io_out8(PORT_KB_CMD,KBCMD_WRITE_CMD); //64h 第二个是60h
   	wait_KB_write();
   	//配置8042 0x47
   	io_out8(PORT_KB_DATA,KB_INIT_MODE);//使能键盘 开启
   ```

这样就完成了初步初始化，使能键盘+中断。

同样为了实现驱动的动态加载，exit函数如下：

```C++
void keyboard_exit()
{
	unregister_irq(0x21);
	kfree((unsigned long *)p_kb);
}
```

而产生中断，调用do_IRQ之后，就会调用`keyboard_handler`;

```C++
//这里是do_IRQ跑到这来的，这样不用修改IDT啥的结构，同时ack也不用自己写EOI寄存器，do_IRQ干了
void keyboard_handler(unsigned long nr, unsigned long parameter, struct pt_regs * regs)
{
	unsigned char x;
	x = io_in8(0x60);
	color_printk(WHITE,BLACK,"(K:%02x)",x);

	if(p_kb->p_head == p_kb->buf + KB_BUF_SIZE)
		p_kb->p_head = p_kb->buf;

	*p_kb->p_head = x;
	p_kb->count++;
	p_kb->p_head ++;	
}
```

**本文中`handler`会不断地读取`60h`端口，然后把数据放到一个环形循环数组里面。**

接下来就是解析这个scan_code了：

```C++
unsigned char get_scancode()
{
	unsigned char ret  = 0;

	if(p_kb->count == 0)
		while(!p_kb->count)
			nop();//等待发送数据 while死循环
	
	if(p_kb->p_tail == p_kb->buf + KB_BUF_SIZE)	
		p_kb->p_tail = p_kb->buf;

	ret = *p_kb->p_tail;
	p_kb->count--;
	p_kb->p_tail++;

	return ret;
}

/*

*/

//分析键盘scanp_code，变成按键
void analysis_keycode()
{
	unsigned char x = 0;
	int i;	
	int key = 0;	
	int make = 0;

	//读取环形队列
	x = get_scancode();
	
	//第一类 如果开头是0xE1	 说明是pausebreak
	if(x == 0xE1)	//pause break;
	{
		key = PAUSEBREAK;
		for(i = 1;i<6;i++)
			if(get_scancode() != pausebreak_scode[i])
			{
				key = 0;
				break;
			}
	}	
	else if(x == 0xE0) //print screen 第二类键盘扫描码 功能键盘
	{
		x = get_scancode();

		switch(x)
		{
			case 0x2A: // press printscreen
		
				if(get_scancode() == 0xE0)
					if(get_scancode() == 0x37)
					{
						key = PRINTSCREEN;
						make = 1;
					}
				break;

			case 0xB7: // UNpress printscreen
		
				if(get_scancode() == 0xE0)
					if(get_scancode() == 0xAA)
					{
						key = PRINTSCREEN;
						make = 0;
					}
				break;

			case 0x1d: // press right ctrl

				//匹配到这些 使用全局变量进行记录
				ctrl_r = 1;
				key = OTHERKEY;
				break;

			case 0x9d: // UNpress right ctrl
		
				ctrl_r = 0;
				key = OTHERKEY;
				break;
			
			case 0x38: // press right alt
		
				alt_r = 1;
				key = OTHERKEY;
				break;

			case 0xb8: // UNpress right alt
		
				alt_r = 0;
				key = OTHERKEY;
				break;		

			default:
				key = OTHERKEY;
				break;
		}
		
	}
	
	if(key == 0)
	{
		//key==0 是普通按键 只有一个Byte
		unsigned int * keyrow = NULL;
		int column = 0;

		//看看是按下还是抬起 最高位1 是make 0 是抬起
		make = (x & FLAG_BREAK ? 0:1);

		keyrow = &keycode_map_normal[(x & 0x7F) * MAP_COLS];

		if(shift_l || shift_r)
			column = 1;

		//这个还挺巧妙 其实就是二维数组 根据是否按下shift来进行大小写
		key = keyrow[column];
		
		switch(x & 0x7F)
		{
			case 0x2a:	//SHIFT_L:
				shift_l = make;
				key = 0;
				break;

			case 0x36:	//SHIFT_R:
				shift_r = make;
				key = 0;
				break;

			case 0x1d:	//CTRL_L:
				ctrl_l = make;
				key = 0;
				break;

			case 0x38:	//ALT_L:
				alt_l = make;
				key = 0;
				break;

			default:
				if(!make)
					key = 0;
				break;
		}			

		if(key)
			color_printk(RED,BLACK,"(K:%c)\t",key);
	}
}
```

总体来说比较简单，然后开个死循环，不断地解析

```C++
while(1)
		analysis_keycode();
```

### 鼠标驱动

同样，本书的鼠标驱动也是只能是`PS/2`接口的鼠标，相比键盘这么简单的解析，直接读取`60h`，鼠标因为其自身原因读取的数据包其实是比较复杂的。

首先键盘的数据包格式如下：

![image-20240526110126558](%E7%AC%94%E8%AE%B0.assets/image-20240526110126558.png)

常用的鼠标控制参数如下：

![image-20240526110148560](%E7%AC%94%E8%AE%B0.assets/image-20240526110148560.png)

文中同样是定义了一个环形循环数组作为鼠标的数据包容器，那么首先看一下初始化

```C++
void mouse_init()
{
	struct IO_APIC_RET_entry entry;
	unsigned long i,j;

	p_mouse = (struct keyboard_inputbuffer *)kmalloc(sizeof(struct keyboard_inputbuffer),0);
	
	p_mouse->p_head = p_mouse->buf;
	p_mouse->p_tail = p_mouse->buf;
	p_mouse->count  = 0;
	memset(p_mouse->buf,0,KB_BUF_SIZE);

	entry.vector = 0x2c;
	entry.deliver_mode = APIC_ICR_IOAPIC_Fixed ;
	entry.dest_mode = ICR_IOAPIC_DELV_PHYSICAL;
	entry.deliver_status = APIC_ICR_IOAPIC_Idle;
	entry.polarity = APIC_IOAPIC_POLARITY_HIGH;
	entry.irr = APIC_IOAPIC_IRR_RESET;
	entry.trigger = APIC_ICR_IOAPIC_Edge;
	entry.mask = APIC_ICR_IOAPIC_Masked;
	entry.reserved = 0;

	entry.destination.physical.reserved1 = 0;
	entry.destination.physical.phy_dest = 0;
	entry.destination.physical.reserved2 = 0;

	mouse_count = 0;

	register_irq(0x2c, &entry , &mouse_handler, (unsigned long)p_mouse, &mouse_int_controller, "ps/2 mouse");

	wait_KB_write();
	io_out8(PORT_KB_CMD,KBCMD_EN_MOUSE_INTFACE);

	for(i = 0;i<1000;i++)
		for(j = 0;j<1000;j++)
			nop();

	wait_KB_write();
	io_out8(PORT_KB_CMD,KBCMD_SENDTO_MOUSE);
	wait_KB_write();
	io_out8(PORT_KB_DATA,MOUSE_ENABLE);

	for(i = 0;i<1000;i++)
		for(j = 0;j<1000;j++)
			nop();

	wait_KB_write();
	io_out8(PORT_KB_CMD,KBCMD_WRITE_CMD);
	wait_KB_write();
	io_out8(PORT_KB_DATA,KB_INIT_MODE);
}
	
```

同样是使能鼠标的IRQ，然后注册中断，基本上和`mouse_init`别无二致。

然后`handler`将数据包记录到环形数组中。

```C++
void mouse_handler(unsigned long nr, unsigned long parameter, struct pt_regs * regs)
{
	unsigned char x;
	x = io_in8(PORT_KB_DATA);
	color_printk(GREEN,WHITE,"(M:%02x)",x);

	if(p_mouse->p_head == p_mouse->buf + KB_BUF_SIZE)
		p_mouse->p_head = p_mouse->buf;

	*p_mouse->p_head = x;
	p_mouse->count++;
	p_mouse->p_head ++;
}
```

然后是解析鼠标移动

```C++
unsigned char get_mousecode()
{
	unsigned char ret  = 0;

	if(p_mouse->count == 0)
		while(!p_mouse->count)
			nop();
	
	if(p_mouse->p_tail == p_mouse->buf + KB_BUF_SIZE)	
		p_mouse->p_tail = p_mouse->buf;

	ret = *p_mouse->p_tail;
	p_mouse->count--;
	p_mouse->p_tail++;

	return ret;
}

/*
其他的都和解析键盘很相似

*/

void analysis_mousecode()
{
	unsigned char x = get_mousecode();

	//3个字节的时候解析
	switch(mouse_count)
	{
		case 0:
			mouse_count++;
			break;

		case 1:
			mouse.Byte0 = x;
			mouse_count++;
			break;
		
		case 2:
			mouse.Byte1 = (char)x;
			mouse_count++;
			break;

		case 3:
			mouse.Byte2 = (char)x;
			mouse_count = 1;
			color_printk(RED,GREEN,"(M:%02x,X:%3d,Y:%3d)\n",mouse.Byte0,mouse.Byte1,mouse.Byte2);
			break;

		default:			
			break;
	}
}
```

鼠标是三个字节才是一个完整的数据包，因此这里只有`case 3:`才进行解析。

## 磁盘驱动

在之前本书对硬盘的操作是通过bios启动的时候的借助BIOS的`INTN`来进行的，但是现在很明显我们已经位于`IA32-E`模式了，无法借助BIOS了，因此我们需要自己来根据磁盘的要求，来发送命令，操作磁盘。既，写一个通用磁盘驱动。

当然，毫无疑问本书中使用的是`传统的SATA接口硬盘`，虽然有一些M.2接口的固态用的是SATA协议，但是绝大部分现代的固态用的基本上都是`NVMe`。

**而磁盘设备和之前所接触到的设备都不太一样，它属于块设备，是因为访问它是一块块（一个扇区）为单位的，而非字节。而之前所接触到的鼠标、键盘都是字符设备。**

### 硬盘控制器

在南北桥时期，磁盘控制器（SATA控制器）是位于南桥里面的，内部通过一些PCI总线进行连接，然后连接到北桥。当然，这是早期的，目前南桥变成PCH了，而北桥移动到CPU,他们一块变成了`chipset`，并且通过更快速的总线进行连接。

![image-20240527085252569](%E7%AC%94%E8%AE%B0.assets/image-20240527085252569.png)

### ATA标准

**ATA**是一个广泛的标准框架，定义了存储设备连接和通信的接口和协议。

目前ATA协议有非常多的版本：

![image-20240527094658468](%E7%AC%94%E8%AE%B0.assets/image-20240527094658468.png)

- **SATA和PATA**

ATA协议最早期的实现是`IDE`，这个非常非常古老了，而后延伸出来了两种标准，既SATA和PATA。PATA因为是并行的接口，也不太优秀。但是他是ATA标准的最初实现方式。**所以SATA和PATA都是ATA标准/协议**，在这个标准框架内，有不同的接口实现方式，包括PATA（Parallel ATA）和SATA（Serial ATA）。

> - ATA标准不仅定义了物理接口（如连接器类型和电气特性），还定义了通信协议（如数据传输模式、命令集等）。
> - PATA和SATA虽然在物理接口和某些协议层面有所不同，但它们都遵循ATA标准的总体框架和目标，即提供高效、可靠的存储设备连接方式。

- **AHCI**

而**AHCI（Advanced Host Controller Interface）**是Intel提出的一个技术标准，用于定义主机和SATA设备之间的接口。

它和SATA的驱动就是`接口`和`标准`的区别，**AHCI**是一个由Intel开发并标准化的**接口规范**，用于定义主机控制器与SATA设备之间的通信方式。而SATA就是定义了设备和主机之间的数据传输协议和物理接口。

如果像网络协议那些理解的话，**SATA关注数据链路层和物理层**，****，比如相关的状态管理啥的。

![image-20240604142903046](%E7%AC%94%E8%AE%B0.assets/image-20240604142903046.png)

- **ATA控制命令**

ATA的控制命令比较多，主要有如下命令表：

![image-20240604154836585](%E7%AC%94%E8%AE%B0.assets/image-20240604154836585.png)

- **PIO模式**

一般来说，外围设备的操作分为`MMIO`和`PIO`,后者比较古老，也就是通过IO端口形式进行访问，本书就是通过这种方式进行访问的。而ATA磁盘控制器的IO端口主要有下面几个端口：

![image-20240604155000035](%E7%AC%94%E8%AE%B0.assets/image-20240604155000035.png)

- **磁盘识别**

如何进行操作呢，其实只需要记录几个常用的端口就行了，比如1F2~1F5，如果写入这个，回头读写的时候，其实就是读写你写入的柱面，磁头，扇区...

而如果你选择写入1F7和177端口，那么只需要写入表11-12那几个命令参数，即可完成查询磁盘，读写磁盘这些功能，比如下述代码进行操作识别磁盘信息：

```C++
void disk_init()
{
	struct IO_APIC_RET_entry entry;

	entry.vector = 0x2f;
	entry.deliver_mode = APIC_ICR_IOAPIC_Fixed ;
	entry.dest_mode = ICR_IOAPIC_DELV_PHYSICAL;
	entry.deliver_status = APIC_ICR_IOAPIC_Idle;
	entry.polarity = APIC_IOAPIC_POLARITY_HIGH;
	entry.irr = APIC_IOAPIC_IRR_RESET;
	entry.trigger = APIC_ICR_IOAPIC_Edge;
	entry.mask = APIC_ICR_IOAPIC_Masked;
	entry.reserved = 0;

	entry.destination.physical.reserved1 = 0;
	entry.destination.physical.phy_dest = 0;
	entry.destination.physical.reserved2 = 0;

	register_irq(0x2f, &entry , &disk_handler, 0, &disk_int_controller, "disk1");

	io_out8(PORT_DISK1_ALT_STA_CTL,0);	

	io_out8(PORT_DISK1_ERR_FEATURE,0);
	io_out8(PORT_DISK1_SECTOR_CNT,0);
	io_out8(PORT_DISK1_SECTOR_LOW,0);
	io_out8(PORT_DISK1_SECTOR_MID,0);
	io_out8(PORT_DISK1_SECTOR_HIGH,0);
	io_out8(PORT_DISK1_DEVICE,0xe0);
	io_out8(PORT_DISK1_STATUS_CMD,0xec);	//identify
}void disk_init()
{
	struct IO_APIC_RET_entry entry;

	entry.vector = 0x2f;
	entry.deliver_mode = APIC_ICR_IOAPIC_Fixed ;
	entry.dest_mode = ICR_IOAPIC_DELV_PHYSICAL;
	entry.deliver_status = APIC_ICR_IOAPIC_Idle;
	entry.polarity = APIC_IOAPIC_POLARITY_HIGH;
	entry.irr = APIC_IOAPIC_IRR_RESET;
	entry.trigger = APIC_ICR_IOAPIC_Edge;
	entry.mask = APIC_ICR_IOAPIC_Masked;
	entry.reserved = 0;

	entry.destination.physical.reserved1 = 0;
	entry.destination.physical.phy_dest = 0;
	entry.destination.physical.reserved2 = 0;

	register_irq(0x2f, &entry , &disk_handler, 0, &disk_int_controller, "disk1");

	io_out8(PORT_DISK1_ALT_STA_CTL,0);	

	io_out8(PORT_DISK1_ERR_FEATURE,0);
	io_out8(PORT_DISK1_SECTOR_CNT,0);
	io_out8(PORT_DISK1_SECTOR_LOW,0);
	io_out8(PORT_DISK1_SECTOR_MID,0);
	io_out8(PORT_DISK1_SECTOR_HIGH,0);
	io_out8(PORT_DISK1_DEVICE,0xe0);
	io_out8(PORT_DISK1_STATUS_CMD,0xec);	//identify
}
```

首先先注册中断，然后向`0x177`端口发送`0xEC`命令，也就是磁盘识别信息，一旦磁盘准备好了，那么就会触发中断。

我们按照下述磁盘识别信息的结构，即可识别磁盘的一些信息。

![image-20240604155435040](%E7%AC%94%E8%AE%B0.assets/image-20240604155435040.png)

```C++
void disk_handler(unsigned long nr, unsigned long parameter, struct pt_regs * regs)
{

	int i = 0;
	struct Disk_Identify_Info a;
	unsigned short *p = NULL;
	port_insw(PORT_DISK1_DATA,&a,256);
	
	color_printk(ORANGE,WHITE,"\nSerial Number:");
	for(i = 0;i<10;i++)
		color_printk(ORANGE,WHITE,"%c%c",(a.Serial_Number[i] >> 8) & 0xff,a.Serial_Number[i] & 0xff);
	
	color_printk(ORANGE,WHITE,"\nFirmware revision:");
	for(i = 0;i<4;i++)
		color_printk(ORANGE,WHITE,"%c%c",(a.Firmware_Version[i] >> 8 ) & 0xff,a.Firmware_Version[i] & 0xff);
	
	color_printk(ORANGE,WHITE,"\nModel number:");
	for(i = 0;i<20;i++)
		color_printk(ORANGE,WHITE,"%c%c",(a.Model_Number[i] >> 8) & 0xff,a.Model_Number[i] & 0xff);
	color_printk(ORANGE,WHITE,"\n");

	p = (unsigned short *)&a;
	for(i = 0;i<256;i++)
		color_printk(ORANGE,WHITE,"%04x ",*(p+i));
}
```

### 磁盘读写

前面学了磁盘的识别，其实磁盘读写是一个道理，都是向命令端口发送相关命令，比如读就是`020h`，写就是`030h`，然后一旦读写完成，就会通过`ATA控制器->IO APIC->LAPIC->INTR(LAPIC连接CPU的中断引脚)->CPU`触发中断。

而设置具体要读哪个磁盘、扇区都是要写入这几个端口来确定的。

![image-20240604155934127](%E7%AC%94%E8%AE%B0.assets/image-20240604155934127.png)

总体而言，磁盘驱动是比较简单的，而一般而言，一个理性的OS的磁盘架构绝非这么简单，一个常见的驱动架构如下：

![image-20240604160039366](%E7%AC%94%E8%AE%B0.assets/image-20240604160039366.png)

这一点`Linux`和`Windows`都有惊人的相似，都是应用程序如果要读写某个文件，通过系统调用->文件系统->文件系统接受到IRP(IO Request Packet)->文件系统解析，重新封装IRP->磁盘驱动->真正处理，读写磁盘，配置磁盘。

而这里面表面上只涉及到了一个驱动，其实对于一个现代的OS，里面设计到的驱动可谓是非常的多。

**比如实际上对于windows驱动调用链：**

xxx.exe->ntoskrnl.exe->fs(ntfs.sys)->xx filter.sys->class xx.sys(**windows 类驱动，各个品牌的鼠标、键盘、磁盘统一，kbdclass.sys,disk.sys...，其实就是把同一种类型的设备抽象成一个类驱动**)->PnpClass.sys(**就正如C++的类抽象一样，所有的Pnp设备有一个类驱动**)->**storport.sys** 完成任务。

# chap8 多核处理器

事实上，这一章之后还有好几章，但是这本书的后几章是一些用户态shell以及系统调用+文件系统(FAT32)的实现，个人不太感兴趣，因此就不继续看了。

## 超线程与多核

- **超线程**

超线程这个概念还是很常见的，是intel推出的一项新功能，**简而言之超线程技术可以把一个cpu的核心变成两个cpu的逻辑处理核心。**

平常所说的**逻辑处理单元**就是指超线程技术之后的cpu核心数量，其实实现起来很简单，可以理解为现在在**硬件层面**cpu实现了一个**线程调度器**，一个核心分成两个逻辑核心，每个逻辑核心独享大部分寄存器，除了少部分MSR和MTRR寄存器。

而硬件负责调度这两个逻辑核心，乍看起来似乎不能提高效率，因为他们的**执行引擎**是共享的，只能并发执行。但是实际上，假如cpu0触发了缺页，`Os`执行换页的时候，那么执行引擎等待缓存插入啥的会非常慢，这个时候会切cpu，减少等待时间。

- **多核**

现代硬件平台都是多核的，在处理器上电之后，硬件选择一个处理器逻辑核心作为`BSP（BootStrap Processor，引导处理器）`，而其他处理器逻辑核心都是`AP（Application Processor，应用处理器）`。区别当前CPU是否是BSP，需要看IA32_APIC_BASE_BSP[8]是否是1。

**当然，每个cpu核心，都有一个唯一的`APIC ID`，apic和xapic为8位，通过cpuid.01:Ebx[31:24]可以获得。而x2apic通过cpuid.0b:Edx获得。**

事实上，Apic ID是一个拓扑结构：

![image-20240612091401788](%E7%AC%94%E8%AE%B0.assets/image-20240612091401788.png)

通过cpuid可以查询APIC ID的不同拓扑层级值，这里不赘述。

## 多核启动

之前`mineos`一直都是单核启动，不用考虑多核的情况，现在处于多核平台，不得不考虑多核之间的同步启动。再次之前，我们需要了解两个前置知识(对称OS/非对称OS,ICR)，来完成多核启动。

### ICR发送IPI

首先是如何发送IPI进行同步，这个其实需要借助`Local APIC`来进行发送。好玩的是`IPI`的类型有很多种（**这个在初始化配置AP的时候非常重要**），在发送的时候可以自己手动选择，比如INIT,StartUp，和正常的Fixed。

**总之和`LVT`以及`RTE`结构有惊人的相似。不一样的是那两个是通过写入IO地址进行配置，而这个就是写入ICR一个类似`LVT、RTE`结构，配置结构信息，就可以发送IPI了。**

首先来看一下相关寄存器：

![image-20240612092935527](%E7%AC%94%E8%AE%B0.assets/image-20240612092935527.png)

基本上都不用解释，就是往`ICR`(MSR 830h，MMIO FEE00300h)写上述结构，这个结构是64位的。

需要额外解释的是`Start Up`，因为这个是用来配置其他`AP`的，但是这个时候其他AP都没执行过，因此肯定IDT啥的也都没初始化，所以此时Vec区域的`0xVV`代表`0x000VV000h`，一旦发送这个，其他处理器自动跑到这个物理地址执行（**这里疑问：如果AP初始化进入保护模式，这个地址是否还是物理地址呢？**）。

同时投递目标可以选全部发送，也可以设置投递目标，来根据`APIC ID`投递单个寄存器。

### SMP/ASMP

SMP(symmetric Mutil-Processing，对称多处理器)和ASMP（Asymmetric Mutil-Processing，非对称多处理器）是操作系统重要的系统结构。

- SMP

SMP是指多核处理器对称的，顾名思义，每个处理器都相同地享用整个硬件的`IO`,`ROM`...

![image-20240612093929625](%E7%AC%94%E8%AE%B0.assets/image-20240612093929625.png)

因此这种操作系统需要注重`资源竞争`，一般的有`信号量`，`自旋锁`...技术

- ASMP

与SMP相反，他们处理器不是对称的，每个处理器独立运行，负责掌握不同的资源：

![image-20240612094135163](%E7%AC%94%E8%AE%B0.assets/image-20240612094135163.png)

win/linux都是SMP架构的操作系统，同样mineos也是如此。

在初始化时候，二者区别如下：

![image-20240612094250976](%E7%AC%94%E8%AE%B0.assets/image-20240612094250976.png)

![image-20240612094315862](%E7%AC%94%E8%AE%B0.assets/image-20240612094315862.png)

主要区别是在AP配置之后，因为都是独立运行的，所以发送`Start UP`后，BSP无需等待AP初始化完成，AP也不需要等待BSP指派任务去哪里执行。

### 配置AP

配置AP比较简单，其实就是发送`IPI`的时候，让AP去执行000VV000地址，然后从实模式->保护模式->IA32E即可。

书中是写了一个`APU_Boot.s`，代码截取部分：

```assembly
#include "linkage.h"

.balign	 0x1000

.text
.code16

ENTRY(_APU_boot_start)

_APU_boot_base = .

	cli
	wbinvd

	mov	%cs,	%ax
	mov	%ax,	%ds
	mov	%ax,	%es
	mov	%ax,	%ss
	mov	%ax,	%fs
	mov	%ax,	%gs

#	set sp

	movl	$(_APU_boot_tmp_Stack_end - _APU_boot_base),	%esp

#	get base address

	mov	%cs,	%ax
	movzx	%ax,	%esi
	shll	$4,	%esi

#	set gdt and 32&64 code address

	leal	(_APU_Code32 - _APU_boot_base)(%esi),	%eax
	movl	%eax,	_APU_Code32_vector - _APU_boot_base

	leal	(_APU_Code64 - _APU_boot_base)(%esi),	%eax
	movl	%eax,	_APU_Code64_vector - _APU_boot_base

	leal	(_APU_tmp_GDT - _APU_boot_base)(%esi),	%eax
	movl	%eax,	(_APU_tmp_GDT + 2 - _APU_boot_base)
	
#	load idt gdt
	
	lidtl	_APU_tmp_IDT - _APU_boot_base
	lgdtl	_APU_tmp_GDT - _APU_boot_base

#	enable protected mode

	smsw	%ax
	bts	$0	,%ax
	lmsw	%ax
```

需要提一嘴的就是，代码中涉及到大量的重定位，想想也是，因为APU只能去`000VV000`去执行，但是我们编译的时候地址确定了(lds脚本确定了.text节的初始地址为`. = 0xffff800000000000 + 0x100000;`)

```assembly

#	get base address

	mov	%cs,	%ax
	movzx	%ax,	%esi
	shll	$4,	%esi
```

计算出来了`BaseAddress`，然后后面就是繁杂的重定位计算，计算出GDT、IDT的地址，进入IA32E。

IPI执行执行已完成，接下来就是看`main`中如何初始化APU的：

```C++
void SMP_init()
{
	int i;
	unsigned int a,b,c,d;

	//get local APIC ID
	for(i = 0;;i++)
	{
		get_cpuid(0xb,i,&a,&b,&c,&d);
		if((c >> 8 & 0xff) == 0)
			break;
		color_printk(WHITE,BLACK,"local APIC ID Package_../Core_2/SMT_1,type(%x) Width:%#010x,num of logical processor(%x)\n",c >> 8 & 0xff,a & 0x1f,b & 0xff);
	}
	
	color_printk(WHITE,BLACK,"x2APIC ID level:%#010x\tx2APIC ID the current logical processor:%#010x\n",c & 0xff,d);
	
	color_printk(WHITE,BLACK,"SMP copy byte:%#010x\n",(unsigned long)&_APU_boot_end - (unsigned long)&_APU_boot_start);
	memcpy(_APU_boot_start,(unsigned char *)0xffff800000020000,(unsigned long)&_APU_boot_end - (unsigned long)&_APU_boot_start);
}
```

`	memcpy(_APU_boot_start,(unsigned char *)0xffff800000020000,(unsigned long)&_APU_boot_end - (unsigned long)&_APU_boot_start);`这句话，就是把我们的汇编写的APU第一句执行的代码复制到`0x20000`物理地址。

然后发送StartUp消息(发送两次是intel推荐的)：

```C++
	icr_entry.vector = 0x00;
	icr_entry.deliver_mode =  APIC_ICR_IOAPIC_INIT;
	icr_entry.dest_mode = ICR_IOAPIC_DELV_PHYSICAL;
	icr_entry.deliver_status = APIC_ICR_IOAPIC_Idle;
	icr_entry.res_1 = 0;
	icr_entry.level = ICR_LEVEL_DE_ASSERT;
	icr_entry.trigger = APIC_ICR_IOAPIC_Edge;
	icr_entry.res_2 = 0;
	icr_entry.dest_shorthand = ICR_ALL_EXCLUDE_Self;
	icr_entry.res_3 = 0;
	icr_entry.destination.x2apic_destination = 0x00;
	
	wrmsr(0x830,*(unsigned long *)&icr_entry);	//INIT IPI

	icr_entry.vector = 0x20;
	icr_entry.deliver_mode = ICR_Start_up;
	
	wrmsr(0x830,*(unsigned long *)&icr_entry);	//Start-up IPI
	wrmsr(0x830,*(unsigned long *)&icr_entry);	//Start-up IPI
```

至此，多核处理器完成初始化，注意，这里有一个坑，就是多核心OS每个核心的GDT都是不一样的，因为每个核心都得有自己的`Tss`描述符！

因为Tss描述符的Rsp0/Rsp3涉及到换栈，假如多核时候，同时进入R0/R3，导致换栈，那么就会栈爆炸了，mineos实现是所有cpu核心的`gdtr`一样，但是gdt限长大，塞了一堆TSS描述符。

而windows的处理方案是每个核心的GDTR不一样，GDT地址除了TSS之外其他内容都是一样的：

![image-20240612100030281](%E7%AC%94%E8%AE%B0.assets/image-20240612100030281.png)

### 自旋锁同步

多核之后，难免引发数据、资源竞争和同步需求，自旋锁是一个实现起来很简单的技术，原理是用到了硬件指令前缀`lock`，汇编指令加了这个，访问内存地址时候会锁住数据总线，其他cpu无法访问，达到同步。

具体实现：

```C++
typedef struct
{
	__volatile__ unsigned long lock;		//1:unlock,0:lock
}spinlock_T;

/*

*/

inline void spin_init(spinlock_T * lock)
{
	lock->lock = 1;
}

/*

*/

inline void spin_lock(spinlock_T * lock)
{
	__asm__	__volatile__	(	"1:	\n\t"
					"lock	decq	%0	\n\t"
					"jns	3f	\n\t"
					"2:	\n\t"
					"pause	\n\t"
					"cmpq	$0,	%0	\n\t"
					"jle	2b	\n\t"
					"jmp	1b	\n\t"
					"3:	\n\t"
					:"=m"(lock->lock)
					:
					:"memory"
				);
}

/*

*/

inline void spin_unlock(spinlock_T * lock)
{
	__asm__	__volatile__	(	"movq	$1,	%0	\n\t"
					:"=m"(lock->lock)
					:
					:"memory"
				);
}
```

初始化spin_lock，值是1，**每次`spin_lock`的时候lock -1**，如果不是负数（说明之前是1），就放行，否则就`pause`一下然后继续判断是否是0，小于等于0就继续判断，否则代表已经解锁，`jmp 1b`来进行重新尝试加锁，

**这里2:这个地方可能会有两个核心同时为1的情况，但是后面又尝试加锁，lock了，所以不会有问题，保证只有一个核心拿到锁，设计还是很巧妙的。**

# chap9 番外

## USB键鼠驱动编写



## PCI总线驱动



## 磁盘驱动编写

