---
title: CSAPP深入理解计算机系统：缓存、内存和虚拟内存（下）
date: 2024-07-19
description: CMU-213学习笔记
cover: /img/cmu213/p3_cover.jpg
tags: 
    - 计算机系统
    - C
    - 内存
    - CMU-213
categories:
    - 笔记
---

## 虚拟内存（Virtual Memory）

### 内存分页

早期的计算机内存被管理为一个数据的array，程序使用内存的物理地址来index数据，然而这样的设计过于简单，无法构建复杂的系统。现在计算机中的内存管理使用虚拟内存（Virtual Memory）只有在一些简单的嵌入式系统中还保留物理地址的方式。

> 我们可以考虑以下因素：现代cpu都是多核的，能够并行地执行程序，每个进程都需要使用一部分内存，计算机系统需要一种能够管理所有进程的内存的方法；每个计算机的进程数也是不固定的，因此需要一种能够随意扩展的内存管理方式；并且，每个进程都应该有相同的内存访问方式，目的是将访问内存的方式抽象化，这样更上一层的软件就不用关心内存的管理过程。

于是现代计算机系统使用虚拟内存技术来实现这一目的。每个进程都有一个虚拟内存空间，下图是一个Linux进程的虚拟内存：

![Virtual Memory of a Linux System](/img/cmu213/vm_vm.png)

其中有我们熟知的Stack和Heap，Code区域为程序的指令，Initialized data区域为全局常数或字符串，Memory-Mapped Region用于管理共享的库，虚拟内存中高位的区域被Kernel的数据占用。实际上Virtual Memory中大部分的空间都是未分配的。

当一个进程访问一个地址时，这个地址其实是对应虚拟内存的虚拟地址，需要通过一定方式转化为物理地址后才能发给内存。

而在上一章中提到过，内存是磁盘的cache，当需要访问的数据不在内存中时（Miss），需要从磁盘中retrieve数据。当内存中已经没有空位时，需要发生内存交换，将一部分内存中的数据flush到磁盘中，再将需要的数据块复制到内存中，内存和磁盘交换数据的最小单位block被称为页（Page），内存和磁盘中的数据被分为很多的Page，又因为它们实际存储着数据，因此被称为物理页（Physical Page）。同样，虚拟内也被分页，他的内存页是虚拟页（Virtual Page），因为Virtual Memory是由于需要内存管理而被想象出来的内存空间，并不存储于某个地方。

因此，只需要一张表，将每个进程中的虚拟页map到物理页，就能实现：通过虚拟地址，访问物理存储器。虚拟页有三种情况：

- 被使用（Allocated），存储在磁盘上
- 被使用（Allocated），存储在内存中和磁盘上（Cached）
- 未被使用（Unallocated）

这张表的每行（Page Table Entry）包含：

- 一个 `Valid` bit，表示是否Cashed
- 物理页地址（如果是Cashed），或磁盘地址，或 `NULL`（如果是Unallocated）

![Page Table](/img/cmu213/vm_page_table.png)

每个进程都有一个Page Table，不同的虚拟页可能对应同一个物理页：

![Page Table](/img/cmu213/vm_separate_page_table.png)

### 虚拟内存的优点

除了上述提到的这些优点，虚拟内存还有以下优点：

- 同一个物理页可以共享被多个进程使用，比如Kernel中的一些共享的数据，还有共享的库
- 虚拟内存中连续的Page在物理内存中可以不连续
- 可以单独给每个Page设置读写权限

### 地址转换

计算机中由MMU负责虚拟地址到物理地址的转换。

![Address Translation](/img/cmu213/vm_addr_trans.png)

如图，在转换过程中，page offset不变，virtual page number用于查表。

Page Table也需要被存在物理存储器中，MMU需要获取Page Table来进行转换，如果Page Table不在缓存中，还需一级一级向下查找。为了提高效率，cpu核心中有一个专门用于cache Page Table的缓存：TLB