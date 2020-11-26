---
title: CPU | 指令 - 编译器优化
date: 2020-11-19
categories:
- CPU
tags:
- CPU
---

一段代码想要最终被计算机执行，首先需要被翻译成机器可识别和执行的指令。在这个过程中，有两个重要的角色，编译器 和 指令集。

代码编译的过程往往包含几个步骤：

```
代码 -> 词法语法分析 -> 语义分析 -> 中间代码生成 -> 目标代码生成
```

在这个过程中，
（1）、（2）依赖于上层的编程语言设计
（3）会将分析结果编译成中间代码。在这个阶段，编译器会尝试对中间代码进行优化，通过减少无效或冗余的代码、计算强度优化等手段，以助于减少最终生成的指令数，或使用更高效的指令
（4）基于中间代码生成机器可执行的目标代码，这个过程和操作系统、指令集、内存等相关。其中，不同的指令集也会带来不同的效率。

## 编译器优化

以 gcc 为例，我们来简单了解一下编译器优化。

gcc 编译器会在编译速度、生成代码大小和生成代码执行速度三个方面尝试进行优化。在默认情况下，gcc 编译器不开启编译优化，因为编译器的目标是减少编译时间、保证编译结果能够按照期望进行测试。

```
Without any optimization option, the compiler's goal is to reduce the cost of compilation and to make debugging produce the expected results.  

Statements are independent: if you stop the program with a breakpoint between statements, you can then assign a new value to any variable or change the program counter to any other statement in the function and get exactly the results you expect from the source code.
```


#### 编译优化

我们以一个简单的例子，来简单了解一下编译优化，看看编译器可能会做哪些事情。

```
#include <stdlib.h>

void main() {
        int loop = 1000000000;
        long sum = 0;
        int index = 1;

        printf("%d", index + 6);
}
```

这是一段 C 语言代码，代码做了变量声明和打印，先来看看不开启编译优化的情况下，编译出来的汇编语言结果：

```
.LC0:
        .string "%d"
main:
        pushq   %rbp    #
        movq    %rsp, %rbp      #,
        subq    $16, %rsp       #,
        movl    $1000000000, -16(%rbp)  #, loop
        movq    $0, -8(%rbp)    #, sum
        movl    $1, -12(%rbp)   #, index
        movl    -12(%rbp), %eax # index, tmp60
        addl    $6, %eax        #, D.2418
        movl    %eax, %esi      # D.2418,
        movl    $.LC0, %edi     #,
        movl    $0, %eax        #,
        call    printf  #
        ret
```

通过 gcc -S -fverbose-asm，我们可以获得添加了变量备注的汇编结果。从中可以粗略看到各个变量的创建，通过 `addl` 进行了 `index + 6` 的操作，并最终通过 `call`调用 `printf` 方法。

如果我们开启编译优化，则得到的结果会有很大不同：

```
.LC0:
        .string "%d"
main:
        movl    $7, %esi        #,
        movl    $.LC0, %edi     #,
        xorl    %eax, %eax      #
        jmp     printf  #
```

最直观的感受是指令少了更多，更加精简了，细看之下，会发现有一些改变：

（1）*消除了冗余的声明*  
里面一些声明了，但是没有使用到的变量，最终被移除了。
比如`sum`、`loop`、`index`

（2）*编译期间能够进行的计算提前做*  
在代码中，`index`变量是参与运算，但是在最终的指令中也被移除了，而这个过程被提前进行了计算，`index = 6`，那么 `index + 1`可以提前预知判断为`7`，则在编译阶段直接进行了替换。在指令上，实际是将：
```
 movl    $1, -12(%rbp)   #, index
 movl    -12(%rbp), %eax # index, tmp60
 addl    $6, %eax        #, D.2418
 movl    %eax, %esi      # D.2418,
```
变为了：
```
 movl    $7, %esi
```
精简了编译出的指令结果。

（3）*指令优化*  
在编译优化前的指令里，函数执行的最后可以看到这样一段指令：
```
 movl    $0, %eax        
```
它的目的只将累加器归0，而在开启编译后话后，变成了这样：
```
 xorl    %eax, %eax      
```

为什么会有这样的变化呢？我们可以简单对比一下这两个命令。
我们使用GNU 汇编器`as`，分别对这两段指令进行编译，并通过 `objdump -D` 来看看两个命令编译后的结果：

```
 0:	b8 00 00 00 00       	mov    $0x0,%eax

 0:	31 c0                	xor    %eax,%eax
```

可以发现，`mov` 指令占用大小是 `xor`的 2.5 倍，从大小上来看，`xor`更胜一筹。可以看出，编译优化会对指令进行优化，选择成本代价更低的指令，生成代码的大小也是编译优化中考虑的一部分。

#### 编译优化项

从上面的例子可以看到，编译器会从不同的角度进行指令优化，在 gcc 的编译优化选项中，我们选取其中的一部分来看看：

```
-fmove-loop-invariants      
Move loop invariant computations out of loops

-funroll-loops              
Perform loop unrolling when iteration count is known

-finline-functions          
Integrate functions not declared "inline" into their callers when profitable

-fshort-double              
Use the same size for double as for float

-fjump-tables               
Use jump tables for sufficiently large switch statements

-fno-threadsafe-statics     
Do not generate thread-safe code for initializing local statics
```

以`-finline-functions`为例，该优化项会尝试将非`inline`函数进行内联，对下面的代码进行优化：

```
#include <stdlib.h>
#include <time.h>

int add(int a, int b) {
        return a + b;
}

void main(int argc, char* argv[]) {
        printf("%d \n", add(atoi(argv[1]), atoi(argv[2])));
}
```

代码读取main函数参数的两个参数，转化为整数猴进行想加，来看看它优化前的中间代码：

```
add:
        pushq   %rbp    #
        movq    %rsp, %rbp      #
        movl    %edi, -4(%rbp)  # a, a
        movl    %esi, -8(%rbp)  # b, b
        movl    -8(%rbp), %eax  # b, tmp61
        movl    -4(%rbp), %edx  # a, tmp62
        addl    %edx, %eax      # tmp62, D.2430
        popq    %rbp    #
        ret
main:
		  ......
        movl    %ebx, %esi      # b -> esi
        movl    %eax, %edi      # a -> edi
        call    add     #
        movl    %eax, %esi      # D.2433,
        movl    $.LC0, %edi     #,
        movl    $0, %eax        #,
        call    printf  #
        ......
```

从`main`函数中可以看到，先调用了`call add`进行加法，再调用`call printf`输出。

当我们开启编译优化，并开启函数内联优化，再来看看编译结果：

```
add:
        leal    (%rdi,%rsi), %eax        
        ret
main:
		  ......
        leal    0(%rbp,%rax), %esi     # a,b值已放入rbp、rax
        movl    $.LC0, %edi     
        call    printf  #
        ......
```

通过`gcc -O -finline-functions`开启函数内联优化进行编译，结果发生了变化：
（1）`add`函数的指令得到精简和优化，原因是参数`-O`实际上是开启了`-O1`级优化，该级别优化不包含`-finline-functions`，所以手动开启了函数内联优化
（2）`main`函数中已经没有了`call add`，而是将`add`函数进行内联处理，将`leal`指令放在了`main`函数中

#### 编译优化等级

在上面的优化中，我们有说到一个概念，编译优化等级。gcc 将编译优化项进行了分类，划分了出多个优化等级用于不同的场景。

| 优化等级 | 说明 |
| ------- | ------- |
| -O0 | 默认优化等级，即不开启编译优化，只尝试减少编译时间|
| -O/-O1 | 尝试减少代码大小，缩短代码执行时间，不会执行需要消耗大量编译时间的优化。对于大函数的编译优化会占用更多的时间和内存。|
| -O2| 与-O1类似，会提升编译时间和代码性能，几乎开启除了时空权衡优化外的所有优化项|
| -O3| 在-O2级别的基础上，开启了更多的优化项，最高优化等级，以编译时间、代码大小、内存为代价获取更高的性能。在部分情况下可能会降低性能。|
|-Os|优化代码大小，会开启-O2中不会增加代码大小的优化项|
|-Ofast| 开启-O3所有优化项，该优化等级不会严格遵循语言标准|

#### 对代码性能的影响

编译优化对最终的代码性能有什么影响呢，来看一个例子：

```
#include <stdlib.h>
#include <time.h>

void main() {
        int loop = 1000000000;
        long sum = 0;
        int start_time = clock();

        int index = 0;
        for (index = 0; index < loop; index ++) {
                sum += index;
        }
        int end_time = clock();

        printf("Sum : %ld, Time Cost : %lf \n", sum, (end_time - start_time) * 1.0 / CLOCKS_PER_SEC);
}
```

依旧是一个循环加的例子，并记录加法运算时间。在不开启编译优化的情况下，执行时间如下：

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/cpu/2/test_1.jpg){:height="500" width="500"}  
开启`-O3`级优化后，执行时间如下：

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/cpu/2/test_2.jpg){:height="500" width="500"}  

优化后，执行时间只有原来的10%左右，效果明显。
（但是这个例子算是一个特殊的例子，计算过程可以用上向量化计算指令，实现并行计算，才能达到这么明显的效果，其他场景下不一定）

#### 优化有风险

编译器优化这么厉害，当然也有“副作用”，来看看下面的例子：

```
#include <stdlib.h>
#include <stdio.h>

int f() {
        int i;
        int j = 0;
        for ( i = 1; i > 0; i += i) {
                ++j;
        }
        return j;
}

void main() {
        int ret = f();
        printf("%d\n", ret);
}
```

函数中的`for`里，依赖于`i`来判断是否结束。正常判断下，`i`循环加溢出后变成负数，就会终止循环，结果如下：

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/cpu/2/test_3.jpg){:height="500" width="500"}  

但是当我们开启`-O2`优化后，会进入死循环：

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/cpu/2/test_4.jpg){:height="500" width="500"}  

我们来看看优化前后的汇编指令：

```
优化前 main：
		  ......
        movl    $1, -8(%rbp)    #, i
        jmp     .L2     #
.L3:
        addl    $1, -4(%rbp)    #, j
        movl    -8(%rbp), %eax  # i, tmp64
        addl    %eax, %eax      # tmp64, tmp63
        movl    %eax, -8(%rbp)  # tmp63, i
.L2:
        cmpl    $0, -8(%rbp)    #, i
        jg      .L3     #,
		  ......

优化后 main：
     	  ......
.L2:
        jmp     .L2     
		  ......

```

在编译优化后，变成了死循环。这是一个特殊的现象，在这个过程中，有符号证书越界（Signed Integer Overflow）在 C99 标准中是一个未定义行为，该行为对应的实现可能是隐式转换或者异常等。在 gcc 中，编译器会 overflow 不会发生，将这段代码编译为死循环。

（可能还会有类似的情况，具体可以参考[CERT - Dangerous Optimizations and the Loss of Causality](https://my.eng.utah.edu/~cs5785/slides-f10/Dangerous+Optimizations.pdf)）

#### 选择性编译优化

既然危险系数这么高，就放弃不用吗，当然不行，我们可以选择对部分代码进行编译优化：

```
#pragma GCC optimize ("O3")
```

该语句会对其之后的代码开启对应级别的编译变化，语句之前的代码不影响，举个栗子：

```
#include <stdlib.h>
#include <time.h>

int f();

void main() {
        int loop = 1000000000;
        long sum = 0;
        int index = 1;
        int ret = f();

        printf("%d \n", index + 6);
        printf("%d \n", ret);
}

# pragma GCC optimize("O3")
int f() {
        int i;
        int j = 0;
        for ( i = 1; i > 0; i += i) {
                ++j;
        }
        return j;
}
```

在`gcc`编译时不开启编译优化，得到编译后的汇编指令：

```
main:
        ......
        movl    $1000000000, -20(%rbp)  #, loop
        movq    $0, -8(%rbp)    #, sum
        movl    $1, -16(%rbp)   #, index
        movl    $0, %eax        #,
        call    f       #
        movl    %eax, -12(%rbp) # tmp60, ret
        movl    -16(%rbp), %eax # index, tmp61
        addl    $6, %eax        #, D.2431
        movl    %eax, %esi      # D.2431,
        movl    $.LC0, %edi     #,
        movl    $0, %eax        #,
        call    printf  #
        movl    -12(%rbp), %eax # ret, tmp62
        movl    %eax, %esi      # tmp62,
        movl    $.LC0, %edi     #,
        movl    $0, %eax        #,
        call    printf  #
        ret

f:
        pushq   %rbp    #
        movq    %rsp, %rbp      #
.L3:
        jmp     .L3     #
```

此时在`main`函数阶段是正常编译的，在`#pragma GCC optimize`之后的`f`函数将会开启编译优化，编译成死循环
