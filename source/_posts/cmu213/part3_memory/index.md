---
title: CSAPP深入理解计算机系统：缓存、内存和虚拟内存（上）
date: 2024-08-19
description: CMU-213学习笔记
cover: /img/cmu213/p2_cover.png
tags: 
    - 计算机系统
    - C
    - 内存
    - CMU-213
categories:
    - 笔记
---

> 本章我们会探讨计算机为什么需要缓存，缓存运行的原理是什么，以及如何利用缓存来优化程序性能。本章大部分的内容在《计算机组成原理》中都有专业的阐述，CSAPP更多的是给读者一个大的框架，以及对于程序员来说，给出有用的程序优化技巧。

## 存储器层次结构

### 存储器

#### 内存

**易失性内存**主要分为两类：
- SRAM（Static Random Access Memory）
- DRAM（Dynamic Random Access Memory）

下面是SRAM一个cell的结构：

![SRAM](/img/cmu213/sram.png)

- 包含6个晶体管，结构较大
- 需要电力维持状态
- 信号变化迅速
- 成本高

下面是DRAM一个cell的结构：

![DRAM](/img/cmu213/dram.png)

- 相比SRAM，结构大幅简化，体积小
- 因为电容会漏电，每隔一段时间（一般为64ms）需要刷新一次（将数据读出再写入）
- 由于电容的特性，信号变化需要时间
- 成本低

#### 磁盘

分为**机械硬盘**（Disk）和**固态硬盘**（SSD），固态硬盘的读写速度更快，寿命更长，但价格更贵。

### CPU/Memory Gap（Motivation）

![Accessing Memory](/img/cmu213/mem_access.png)



这是一个最简单的总线示意图。cpu执行到访问内存的指令后，通过I/O Bridge上的北桥（North Bridge）上的内存控制器（Memory Controller），将读写地址发送到内存，内存将数据发送回cpu。cpu若想访问磁盘，则要通过I/O Bridge上的南桥（South Bridge），通过I/O总线访问磁盘设备。然而这样的设计有*很大的问题*。

> *近几十年，cpu飞速发展，然而内存速度由于成本原因，并没有质的飞跃。到了2015年，内存、cpu、ssd和disk之间都有了一个较大的访问时间gap。*

![CPU/Memory Gap](/img/cmu213/cpu_mem_gap.png)

可以想象，如果cpu发出指令后直接从内存或硬盘读取数据，将严重阴影cpu的效率，因为大部分时间都**浪费**在等待上了。

### Locality

通过观察我们可以发现，许多程序更倾向于**使用连续的空间**（Spatial Locality），或重**复使用同一空间多次**（Temporal Locality）。

比如下面这段代码：

```c
sum = 0;
for (i = 0; i < n; i++)
    sum += a[i];
```

- 对于数据（局部变量）来说：数组是连续的，数组的访问是连续的（Spatial Locality），访问了同一变量`sum`多次（Temporal Locality）
- 对于程序来说：程序都存在连续的内存快中，程序的执行是按序的（Spatial Locality），对于一个循环我们会多次执行同一段程序（Temporal Locality）

### 层次结构

我们想要更快的存储设备来弥补cpu和存储器之间的延迟，但更快的存储器成本更高，能耗也更高；程序的运行往往都遵循Locality原则，因此只有需要被频繁访问的数据才有被存在更快的存储器中的价值。当今计算机中解决cpu和存储器之间延迟的方法，就是在cpu和内存之间加入多级的**缓存**（Cache）。

![The Memory Hierarchy](/img/cmu213/mem_hierarchy.png)

计算机中的存储器呈现金字塔型结构，越往上存储器**越小**，速度**越快**，价格**越高**，存放**更常用**的数据。
- 其中最高一级是cpu核心中的寄存器，他们的速度最快，但数量最少（x86-64中一个核心只有16个`64-bit`的通用寄存器）。
- 其后是三极的缓存，使用SRAM，虽然价格高，但缓存的容量都不大。
- 然后是cpu外的内存，使用DRAM
- 最慢的是磁盘，和远程存储


在整个存储器体系结构中，**每一级都是下一级的缓存**，比如L3缓存中的数据是内存中的一部分（Subset），内存中的数据又是磁盘中的一部分...只有磁盘中的数据是持久的，也就是说寄存器、缓存和内存中的数据在计算机断电后全部失效。

当访问磁盘中的数据时，先将一块数据读入内存，再将其中的一小块读入L3、L2...最后读入寄存器。过程看似繁杂，但实际由于Locality的存在，使得整体的效率变得非常高。

#### 访问

![Cache Basic Idea](/img/cmu213/cache_visit.png)

数据以*块*为单位获取。访问数据时，逐级向下寻找数据块，如果cache中找到，则直接使用，称为"**Hit**"；如果cache中找不到，就从内存中把包含我们需要的数据的一块数据复制到cache中，称为"**Miss**"。

## 缓存（Cache）

*下面来详细讲一下cpu的缓存原理。*

### Hit & Miss

上面提到cache的"Hit"和"Miss"，有三种"Miss"：
- Cold Miss：第一次访问一个数据块，这个数据块肯定不在缓存中
- Capacity Miss：正在使用的数据块已经填满了cache，没有多余的空间
- Conflict Miss：从下一级读取数据块到上一级时，数据块的index会进行取模运算来获得一个在上一级cache里的index。即使cache不满，也可能发生多个数据块要求同一个cache中的位置

### 结构

![Cache](/img/cmu213/cache.png)

Cache中包含$S=2^s$个set，每个set包含$E=2^e$行（line），每行由三部分组成：
- Valid(1-bit)：表示这一行是否被使用
- Tag(t-bit)：每行的标签，用于查询
- Block($B=2^b$字节)：实际存储数据的地方

因此cache的**size**可被计算为$C=B\times E\times S$字节。

![Cache Address](/img/cmu213/cache_addr.png)

想要**访问一个地址**，将地址拆成长度为`t` `s` `b`的三部分。使用中间`s`位`S`作为index，选择一个set；然后遍历这个set中的每个line，查询是否有一个line的tag与地址前`t`位相同，如果相同，则为"Hit"；最后用地址后`b`位`B`作为index，在block中index到对应字节，找到数据。如果没有找到tag，则为"Miss"，在下一级获得block数据，选择一个空行写入。

> 为什么使用中间的`s`位来index set呢？如下图，假设使用最高的`s`位来index，那么同一个set中的cache地址是连续的，又由于程序往往都遵循**Locality**，一个连续的数组就会全部挤在一个set中，也就是说同一时刻最多只能有E个连续的block大小的数组，超出的则全为Miss，导致Hit Rate很低。而不使用最高的几位来作为set的index则没有这个问题。

![Middle Index](/img/cmu213/cache_mid_index.png)

每一个set中有`E`行称为"E-way Set Associative Cache"，"1-way"又被称为"**Direct-Mapped Cache**"。Direct-Mapped Cache在系统中不常用，因为其命中率（Hit Rate）很低。

## 优化程序性能


![The Memory Mountain](/img/cmu213/mem_mountain.png)

> **这是这本书封面上的图，直观地解释了为什么我们需要优化程序的性能。**
> 
> 测试场景是遍历一个大小为Size的数组，间隔为Stride（比如遍历`a[0]` `a[3]` `a[6]`...然后`a[1]` `a[4]` `a[7]`...直到每个元素都被遍历一次）。Stride越大，程序的Spatial Locality随之降低；Size越大，程序的Temporal Locality随之降低，因此效率降低。值得注意的是，如果沿着Size轴看，效率呈阶梯状下降，每个阶梯代表一级cache，在边缘处急速下降是因为数组的Size超过了这部分cache的大小，发生了Capacity Miss。若果沿着Stride方向看，同一个cache层级内，效率随着Stride的增大逐步下降，因为访问的元素之间跳跃越大，Spatial Locality逐步减小，当Stride增大到一定程度，使得每个元素的访问都在不同的block中时，效率达到最低，且不再改变。

<span style="color: orange;">因此，提升程序的性能，最本质的方法就是提升**Locality**。</span>

### 矩阵乘法

最经典的一个例子就是**矩阵乘法**。假设两个$n\times n$的矩阵相乘，最简单的写法是这样：

```c
/* ijk */
for (i = 0; i < n; i++)  {
    for (j = 0; j < n; j++) {
        // 对于矩阵中的每个位置，分别计算乘积后的值
        sum = 0.0;
        for (k = 0; k < n; k++) 
            sum += a[i][k] * b[k][j];
        c[i][j] = sum;
    }
} 
```

计算这个程序的Miss Rate，假设block size = `32 Bytes`，n远大于block size。我们知道，计算机中二维数组是以一行一行存的。对于遍历一行，每四个元素就有一个元素不在block中；对于遍历一列，每个元素必定都不在同一个block。

$$Miss\space Rate=0.25+1.0=1.25$$

假设交换一下`i`和`j`的遍历：

```c
/* jik */
for (j = 0; j < n; j++) {
    for (i = 0; i < n; i++)  {
        // 对于矩阵中的每个位置，分别计算乘积后的值
        sum = 0.0;
        for (k = 0; k < n; k++) 
            sum += a[i][k] * b[k][j];
        c[i][j] = sum;
    }
} 
```

Miss Rate依然是1.25

而把循环顺序换成kij：

```c
for (k = 0; k < n; k++) {
    for (i = 0; i < n; i++) {
        r = a[i][k];
        for (j = 0; j < n; j++)
            c[i][j] += r * b[k][j];
    }
}
```

`b`和`c`都以行为单位遍历，$Miss\space Rate= 0.25 + 0.25=0.5$

把循环换成jki：

```c
for (j = 0; j < n; j++){
    for (k = 0; k < n; k++) {
        r = b[k][j];
        for (i = 0; i < n; i++) {
            c[i][j] += a[i][k] * r;
        }
    }
}
```

由于`a`和`c`都以列为单位遍历，$Miss\space Rate= 1.0+1.0=2.0$

实际运行过程中，三者的运行效率的确有很大的差别。

![Matrix Multiplication Performances](/img/cmu213/iter_performance.png)

**实际上还有更高效的算法**，为了让Locality达到最大，每次做运算的行或列最好能完全fit进一个cache block中，因此我们可以将矩阵切成$B\times B$的小块，$B$为cache block的长度，以块为单位计算矩阵乘积。