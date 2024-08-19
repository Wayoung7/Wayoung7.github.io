---
title: CSAPP深入理解计算机系统：程序的机器级表示（上）
date: 2024-07-28
description: CMU-213学习笔记
cover: /img/cmu213/p2_cover.png
tags: 
    - 计算机系统
    - C
    - ASM
    - CMU-213
categories:
    - 笔记
---

> 本章是此书中最重要的一章，也是CMU-213这门课的精华所在。本章不会教你如何写汇编代码，而是带领你从汇编的角度理解程序在计算机中是如何表示和运行的。

## 程序的编码

当你使用gcc编译C程序代码时，会经历以下步骤：

![从C代码到可执行程序](/img/cmu213/compile_c.png)

- 你所写的C程序（`main.c`）会先由编译器（一般为gcc/clang）进行优化，生成汇编代码（ASM）`main.s`
- 然后由Assembler（gcc或as）翻译为二进制文件，也就是Object File `main.o`
- 最后由链接器Linker（gcc或ld）将你的程序和你所用到的其他人的代码（`#include ...`）（也就是静态库Static Library）组合，生成最后的二进制可执行程序

使用`gcc -o main main.c`编译器会隐藏中间所有的过程，直接输出可执行程序。如果想看到中间的结果，可以使用：
- `gcc -Og -S` 生成汇编代码
- `gcc -Og -c` 生成Object File
- 使用`gdb`时，在`gdb shell`中输入`disassemble`后接函数的名称，可以利用**反汇编**从可执行文件提炼出汇编代码
- 在*Linux*中还可以使用`objdump -d`后接Object File或Executable的名称进行反汇编

> `-Og`的作用是让编译器对代码进行**小幅度**的优化，类似的`compiler flag`还有`-O1` `-O2`，这两个优化方式会对代码进行大幅度优化，甚至改变整个程序的结构。而`-Og`既能优化掉程序中的冗余，又不至于将汇编变成我们完全无法理解的样子，便是我们需要的。

```c
int move(int a, int *b) {
    *b = a;
    return a;
}
```

将以上的函数反汇编，我们能够得到：

```asm
0000000000000000 <move>:
   0:   f3 0f 1e fa             endbr64
   4:   89 f8                   mov    %edi,%eax
   6:   89 3e                   mov    %edi,(%rsi)
   8:   c3                      ret
```

每一列冒号前的16进制数为当前指令的 $offset$ ，每一行最后的英文代码是**汇编**（Assembly Language），中间冒号后的16进制数便是这一行的汇编代码在计算机中的表示形式。不同的指令有不同的长度，从`1`到`15`bytes不等，越是常用的指令的长度越短。

*需要注意的是，不同的cpu架构有着完全不同的指令集，目前主流的架构是$Intel$的`x86-64`（64位）。*

简单解释一下上面的**汇编代码**：
```asm
endbr64       # 做一些函数运行前的准备，这部分我们可以忽略

mov    %edi,%eax       # 将寄存器edi中的值复制到寄存器eax中，eax = edi;
mov    %edi,(%rsi)     # 将寄存器edi中的值复制到寄存器rsi中的地址所指向的一块内存中，*rsi = edi;
ret                    # 函数返回
```

**汇编(Assembly Code)**也被称为Machine Code（汇编代码的二进制形式），用于告诉cpu“*每一步*”该怎么做，可以看出其与C语言这样的*高级语言*最明显的区别是：
1. Machine Language中不再有任何我们定义的**变量名称**
2. Machine Language中不再区分**数据类型**

## 数据长度

由于历史原因，Intel最开始设计的cpu架构为16位，因此就将`16-bits`定义为一个“字”（`word`）。使用`gcc`生成汇编代码时，一般都会在Instruction后加一个表示操纵的数据的长度的字母。比如`movq`的意思是移动（实际上是复制）一个64位的数据。

![汇编中的数据长度](/img/cmu213/asm_data_len.png)

## 寄存器（Registers）

**寄存器**是计算机中最高一级的高速**缓存**（后续的章节会详细讲述），位于cpu核心中，其大小是所有缓存中**最小**的，但速度是**最快**的，寄存器中存储程序运行中需要频繁访问的数据。**`x86-64`架构中有`16`个`64`位的通用寄存器**，每一个都有自己的名称（以`%r`开头），每一个寄存器中的后`32` `16` `8`位都可以单独访问，并且都有自己的名称。这16个寄存器用来存放**整数**和**指针**（浮点数有另外的寄存器）。

![寄存器](/img/cmu213/regs.png)

前面提到这16个寄存器是**通用寄存器**，其中只有`%rsp`寄存器是特殊的，它只存放**栈指针** $Stack Pointer$（指向目前内存栈区最低字节）。其余的寄存器有些有特殊的作用，比如`%rax`存放函数返回值、`%rdi`存放函数的第一个参数等等，但它们也都可以被用作*通用目的*。

### movq

我们以`movq`**移动指令**为例理解程序是怎么操纵寄存器中的数据的。其格式为 `movq <Source>, <Destination>`，意思是将寄存器`<Source>`中的数据**复制**到`<Destination>`中（对你没看错，虽然"mov"代表了"move"，意为移动，但其实是复制，`<Source>`中的数据在移动后依然可用）。**操作数**（Operand）的类型可以有三种：

- Immediate：常数
  - 和C语言相似，但以`$`开头，如`$0x2a` `$-213`
- Register：来自于寄存器中的数
  - 比如`%rax`
- Memory：来自于寄存器中的地址所指向的内存`64-bits`空间的数
  - 比如`(%rax)`的意思是`%rax`中存放了一个内存地址，我们要取出这个内存地址所指向的内存空间里的值

下面这张图很好的展示了`movq`在不同Operands下的例子：    

![movq example](/img/cmu213/movq_example.png)

<span style="color: orange;">*注：使用一个指令无法做Memory-Memory的mov操作*</span>

> **地址计算**：用括号括起来的寄存器表示去寄存器中存放的内存地址寻找，
> $$(R)  \space\rightarrow\space  Mem[Reg[R]]$$
> **实际上，内存地址的完整表示方式为** $D(Rb,Ri,S)$ 
> $$D(Rb,Ri,S)  \space\rightarrow\space  Mem[Reg[Rb] + S * Reg[Ri] + D]$$
> - D：Displacement地址的**常数**偏移量
> - Rb：基准地址，比如`%rbi`
> - Ri：索引Index Integer
> - S：Scale，值可以是`1` `2` `4` `8`
> - D和S可省略
> 比如：`movq 8(%rbi, %rbp, 4), %rax`  

举一些例子：假设我们有**寄存器**：

|Register|Value   |
|------|--------|
| %rdx | 0xf000 |
| %rcx | 0x0100 |

| Expression | Address Computation | Address |
|------------|---------------------|---------|
|0x8(%rdx)|%rdx + 0x8|0xf008|
|(%rdx,%rcx) |%rdx + 1 * %rcx|0xf100|
|(%rdx,%rcx,4)|%rdx + 4 * %rcx|0xf400|
|0x80(,%rdx,2) |2 * %rdx + 0x80|0x1e080|

另外，我们可以更改mov的**后缀**，*来只改变寄存器中的一部分bit*。下面的例子展示了`movb` `movw` `movl`

```asm
movabsq $0x0011223344556677, %rax   # %rax = 0011223344556677  movabsq意为"move absolute quad-value"，只能用于将64位Immediate Value 复制到64位寄存器中
movb    $-1, %al                    # %rax = 00112233445566FF  修改最后一字节的数据，具体寄存器的名称请参考表格
movw    $-1, %ax                    # %rax = 001122334455FFFF
movl    $-1, %eax                   # %rax = 00000000FFFFFFFF  mov一个寄存器的后4字节会将前4字节全设为0，称为"Zero Out"
movq    $-1, %rax                   # %rax = FFFFFFFFFFFFFFFF
```

> 设置寄存器的后4字节为什么要"Zero Out"前4字节？
> - 使用后4字节的寄存器一般是为了计算32位数据，或兼容32位系统，高位上的数是不需要的垃圾数据或是旧数据
> - "Zero Out"的操作非常高效，基本不会有额外的开销

mov指令还有其他各种**变体**，比如`movz`系列（Zero-extending Data Movement）和`movs`系列（Sign-extending Data Movement），这里就不展开细讲了。

*再举一个使用mov的例子：*

```c
long exchange(long *xp, long y) {
    long x = *xp;
    *xp = y;
    return x;
}
```

```asm
exchange:
    movq    (%rdi), %rax    # 第一个参数存在%rdi中，是一个指针，x = *xp;
    movq    %rsi, (%rdi)    # 第二个参数在%rsi中，将此值存到%rdi指向的内存中，*xp = y;
    ret                     # 默认存在%rax中的数即是返回值，return x;
```

## 数学与逻辑运算

通过指令我们等进行各种整数的数学和逻辑运算：

![Integer Arithmetic Operations](/img/cmu213/asm_arithmetic.png)

其中最重要的是`leaq`(load effective address of quadword)，`leaq <Src>, <Dst>`意为将`<Src>`计算出的地址（使用上文中讲过的**地址计算**形式）值赋给`<Dst>`。<span style="color: orange;">*`leaq`虽然有地址计算，但并不会访问内存*</span>。常被用于指针的计算，比如`p = &arr[i];`，同时也可以进行**整数运算**，表达式**必须**是 $x + k \times y(k=1,2,4.8)$ 的形式，比如：

```c
long m12(long x) {
    return x * 12;
}
```

此段代码在被编译器优化后会转化为：

```asm
leaq (%rdi, %rdi, 2), %rax      # t = x + 2 * x
salq $2, %rax                   # return t << 2
```

**相比于使用普通的乘法运算，大大提高了效率。**

下面举了一个Arithmetic Operation的例子，你可以先对着C代码思考一下，*假设你是编译器，你会怎么用最少的指令处理这个程序*：

```c
long arith(long x, long y, long z) {
    long t1 = x + y;
    long t2 = z + t1;
    long t3 = x + 4;
    long t4 = y * 48;
    long t5 = t3 + t4;
    long rval = t2 * t5;
    return rval;
}
```

```asm
arith:
    leaq    (%rdi, %rsi), %rax     # t1
    addq    %rdx, %rax             # t2
    leaq    (%rsi, %rsi, 2), %rdx
    salq    $4, %rdx               # t4
    leaq    4(%rdi, %rdx), %rcx    # t5
    imulq   %rcx, %rax             # rval
    ret
```

> 为什么`64-bit`的整数运算要用`leaq`而不是`addq`和`imulq`？
> - 对于既有加法又有乘法的运算，`leaq`可以减少指令数
> - `addq`是由cpu上的$ALU$计算的，而`leaq`是由cpu上的$AGU$（Address Generation Unit**地址生成单元**）计算的，使用`leaq`不会设置**条件码**（下个部分会提到），提高效率

## 控制

asm中的控制语句全部由跳转（jump）来完成，类似于C中的`goto`。下面是一个简单的$Control Flow$例子：

```c
extern void func1();
extern void func2();

void cond(long c) {
    if (c > 10) {
        func1();
    else
        func2();
}
```

```asm
0000000000000000 <cond>:
   0:   48 83 ec 08             sub    $0x8,%rsp
   4:   48 83 ff 0a             cmp    $0xa,%rdi
   8:   7e 0f                   jle    19 <cond+0x19>
   a:   b8 00 00 00 00          mov    $0x0,%eax
   f:   e8 00 00 00 00          call   14 <cond+0x14>
  14:   48 83 c4 08             add    $0x8,%rsp
  18:   c3                      ret
  19:   b8 00 00 00 00          mov    $0x0,%eax
  1e:   e8 00 00 00 00          call   23 <cond+0x23>
  23:   eb ef                   jmp    14 <cond+0x14>
```

其中`jle` `jmp`就是跳转指令。

### 条件码（Condition Codes）

除了上面提到的16个整数寄存器，cpu中还有一系列`single-bit`的**条件码寄存器**（Condition Code Registers），用来保存最近一次做各种数学和逻辑运算所获得的结果的属性。*有四个最常用的Condition Code：*

- `CF`$(Carry flag)$：最近一次计算若最高位有进位，则设置成`1`，否则设置成`0`，用于检测无符号整数的运算是否有溢出。
- `ZF`$(Zero flag)$：最近一次计算结果为$0$。
- `SF`$(Sign flag)$：最近一次计算结果为负数。
- `OF`$(Overflow flag)$：最近一次计算造成了**补码**表示方式下的溢出（包括正、负溢出）。*（补码计算的溢出请自行搜索）*

举例：假设我们有一条指令：`addq %rbi, %rax`，将rbi中的数值加到rax上(`t = a + b`)，下面是四个flag的决定方式：
- `CF`: `(unsigned)t < (unsigned)a`，$Unsigned overflow$
- `ZF`: `t == 0`
- `SF`: `t < 0`
- `OF`: `(a < 0 == b < 0) && (t < 0 != a < 0)` (**补码**溢出的定义)

**设置条件码的一些特殊情况：**
- `leaq`不会设置条件码，其他所有运算都会设置条件码
- 对于**逻辑运算**，`CF` `OF`被设成`0`
- 对于**移位运算**，`CF`为最后一个被移出的位，`OF`为`0`
- 对于 $INC$ 和 $DEC$， 不会改变`CF`

**还有一类运算，只改变条件码，不改变16个寄存器中的值：**

![CMP和TEST](/img/cmu213/control_comp.png)

- $CMP$计算$S_2-S_1$（类似于sub），但不改变$S_2$的值
  - 通常被用于条件语句`if (a < b) {...}`
- $TEST$计算$S_1$ & $S_2$（类似于and），同样不改变$S_2$的值
  - `test a, a`检测`a`是否为`0`

### 访问条件码

设置好条件码之后，我们如何访问并使用它们呢？

1. 可以根据条件码，把寄存器或内存中的一个字节设成`0`或`1`
2. 可以根据条件码，跳转到其他指令位置
3. 可以根据条件码，传输数据

先来看第一种用途：**根据条件码，把寄存器或内存中的一个字节设成`0`或`1`**

![SET指令](/img/cmu213/asm_set.png)

> 使用英文很容易就能理解这些指令的含义：
> 指令的名称和其作用的英文是对应的，比如`setl`是指`set something if less than`，大多数指令还有*别名*（Synonym），比如`setl`的别名是`setnge`是指`set something if not greater than or equal to`，和名称指的是同一个意思。
> 注：整数的比较要区分有符号和无符号

假如我们有以下代码：

```asm
# 假设 a 存在 %rdi 中，b 存在 %rsi 中
comp:
    cmpq    %rsi, %rdi      # 比较 a 和 b （实际上就是做了 a - b 计算），注意指令中的顺序和实际比较的顺序是反的
    setl    %al             # 如果 a < b，即 (SF ^ OF) == 1，把 %al 设成 1 ，反之设成 0
    movzbl  %al, %eax       # 把 %eax 中其余的位都设成 0，%rax中的前 8 字节也都设成了 0 ，由于 "Zero Out"
    ret
```

上面是一个典型的比较两个`64-bit`有符号整数的汇编代码，对应了C代码：

```c
int lt (long a, long b) {
    return a < b;
}

```

接下来看访问条件码的第二种用途：**跳转指令**

![JUMP指令](/img/cmu213/asm_jump.png)

和 $SET$ 指令一样可以用英文来理解每个指令的含义。其中，`jmp *`被称为间接跳转（Indirect Jump）后接寄存器或内存地址，比如`jmp *(%rax)`。*你可以把跳转理解成C语言中的`goto`语句*，下面这个课件上的例子清楚地展示了跳转的过程，**相同颜色的代码是对应的**：

![JUMP示例](/img/cmu213/asm_jump_example.png)

### 条件传送（Conditional Moves）

<span style="color: orange;">上述的方式能够完全实现分支的功能，但现代cpu一般不会使用上述方法来实现分支。</span>假如我们有如下C程序：

```c
long absdiff(long x, long y) {
    long result;
    if (x < y)
        result = y - x;
    else
        result = x - y;
    return result;
}
```

经过GCC编译后得到的汇编代码：

```asm
absdiff:
    movq    %rsi, %rax
    subq    %rdi, %rax      # rval = y - x
    movq    %rdi, %rdx
    subq    %rsi, %rdx      # eval = x - y
    cmpq    %rsi, %rdi      # Compare x:y
    cmovge  %rdx, %rax      # If >=, rval = eval
    ret
```

在上面的程序中，汇编代码先是**分别计算了两个分支的结果**，然后再根据条件语句，**判断选用哪个结果**，复制到返回值中。这与我们平时认为的条件语句先判断条件，然后再执行对应区域的代码不同。那么现代cpu为什么要怎么做呢？

> **分支预测**
> 在现代cpu中，提高cpu效率的一个很重要的手段被称为*流水线*（Pipelining），每条指令在流水线中按序执行，完成一小部分任务（比如：先取指令，解析指令，然后从内存读取数据，做一系列运算，最后再向内存写入数据）。cpu提高运行效率的方法是将这些流水线的执行重叠，当前一套指令在ALU做计算时，后一套指令可能在取指令或解析指令，这样保证了流水线上的每一个*机器*几乎都在**满功率运行**，大大提升了cpu的效率。
> Pipelining要求cpu提前知道整一套指令，然而当遇到分支时，cpu无法提前知道该往哪个方向运行。于是现代cpu采用**分支预测**技术，来猜测每个跳转是否会执行。对于一些比较容易预测的条件语句，现代cpu的分支预测正确率能达到$90\%$，而像比较两个数的大小这样不容易预测的条件语句，正确率在$50\%$左右。正确的预测能提升cpu的效率，而对于一个失败的预测，cpu往往要花更多的时间来弥补错误（一般为15-30时钟周期）。

最后使用`cmovge`（conditional move when greater than or equal to）来判断是否要将`eval`复制给`rval`。下面是一系列条件传送的指令，其中，开头的`cm`指$Conditional Move$，`mov`后的英文后缀和前一个表格对应。

![条件传送](/img/cmu213/asm_conditional_move.png)

我们可以用一下代码概括条件传送：

```
v = then-expr;
ve = else-expr;
t = test-expr;
if (!t) v = ve;
```

> 并不是所有的分支都会使用条件传送，*编译器足够聪明*，当分支中的代码有**副作用**（Side Effect）、比较**复杂**、或**不安全**（解引用一个指针）时，就不会使用条件传送。

## 循环

C中的循环有`do-while` `while` `for`，分别来看它们是怎么运行的。

### Do-While

Do-While的C代码一般结构是这样的：

```c
do
    Body
    while (Test);
```

很容易将其变为`goto`格式：

```c
loop:
    Body
    if (Test)
        goto loop;
```

在汇编中使用**跳转**指令可以轻松实现。举一个例子：

```c
// 计算x中1的个数
long pcount_do(unsigned long x) {
    long result = 0;
    do {
        result += x & 0x1;    // 取x的最低位，加到result上
        x >>= 1;              // 移除x的最低位
    } while (x);
    return result;
}
```

```asm
0000000000000000 <pcount_do>:
   0:   ba 00 00 00 00      mov    $0x0, %edx   # result = 0
   5:   48 89 f8            mov    %rdi, %rax   # return = x
   8:   83 e0 01            and    $0x1, %eax   # return = return & 0x1
   b:   48 01 d0            add    %rdx, %rax   # return += result
   e:   48 89 c2            mov    %rax, %rdx   # result = return
  11:   48 d1 ef            shr    $1, %rdi     # x >>= 1
  14:   75 ef               jne    5 <pcount_do+0x5>    # while (x != 0)
  16:   c3                  ret                 # return return
```

### While

While的一般形式为：

```c
while (Test)
    Body
```

转换成`goto`：

```c
    goto test;
loop:
    Body
test:
    if (Test)
        goto loop;
done:
```

While还可以直接转化成Do-While，只需要在最开头多判断一次：

```c
    if (!Test) 
        goto done;
    do
        Body
        while(Test);
done:
```

### For

一般形式：

```c
for (Init; Test; Update )
    Body
```

任何`for`循环都可以完美转化为`while`或`do-while`循环，这里就不多赘述了。另外再某些情况下，`for`循环转化为`do-while`循环可以省去第一次判断，提高效率。下面展示一个`for`循环的例子：

```c
long pcount_for(unsigned long x) {
    unsigned long i;
    long result = 0;
    for (i = 0; i < 16; i++) {
        unsigned bit = (x >> i) & 0x1;
        result += bit;
    }
    return result;
}
```

```asm
0000000000000000 <pcount_for>:
   0:   ba 00 00 00 00      mov    $0x0,%edx
   5:   b9 00 00 00 00      mov    $0x0,%ecx
   a:   eb 10               jmp    1c <pcount_for+0x1c>
   c:   48 89 f8            mov    %rdi,%rax
   f:   48 d3 e8            shr    %cl,%rax
  12:   83 e0 01            and    $0x1,%eax
  15:   48 01 c2            add    %rax,%rdx
  18:   48 83 c1 01         add    $0x1,%rcx
  1c:   48 83 f9 0f         cmp    $0xf,%rcx
  20:   76 ea               jbe    c <pcount_for+0xc>
  22:   48 89 d0            mov    %rdx,%rax
  25:   c3                  ret
```

## Switch

### 跳转表（Jump Table）

编译器有多种处理`switch`语句的方式。最先能想到的，也是简单的就是把`switch`转变成一系列`if-else`语句，使用上述分支技术处理。然而这样的方式需要与**每一个**`case`做比较，有没有更高效的办法呢？一般情况下我们使用`switch`时，`case`中的值都是比较**相近**的，甚至就是**连续**的。因此在编译器可以建立一张表：**跳转表**（$Jump Table$），将`case`中的每个数值对应到相应代码块中，在运行时可以通过`switch`的数的值$index$到$Jump Table$中，获得跳转位置并跳转到相应的代码块，下图是$Jump Table$的结构：

![Jump Table的结构](/img/cmu213/asm_jump_table.png)

**如此一来，无论`case`有多少个，跳转到相应的位置只需$O(1)$常数时间。**

### 处理特殊case

- 多个`case`对应一个代码块：多个$Jump Table$的$entry$对应同一个代码块
- 一个`case`最后没有`break;`（Fall Through）：在代码块的最后再加一个Jump
- 在连续的`case`数值中缺少个别数值：这些数值直接跳转到`default`

### 编译器如何选择

> *编译器足够聪明。*

上面的技术只能处理*连续的*或是比较*相近*的`case`数值，如果`case`较为分散，编译器还可能用二分查找（$Binary Search$）来$index$。并不是每个`switch`语句编译器都会做如此的优化，当`case`数量较少，或每个`case`中的代码块非常复杂时，编译器会使用普通的`if-else`来处理。

## 总结

“程序的机器级表示”（$Machine Level Programming$）是整本书中的**核心**章节之一，内容非常多，目前这些只涵盖了本章*不到一半*的内容，更多的内容会放在下一篇文章中。