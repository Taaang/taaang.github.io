---
title: Kotlin升级导致的when异常问题
date: 2019-02-10 16:15:36
categories:
- Kotlin
tags:
- Kotlin
---

送测结束，开心上线，结果线上突然报错，发现代码走到了一个理论上不可能走到的分支里，又遇到鬼故事了_(:з」∠)_ GG。

## 问题背景

一段神奇的代码：

两个枚举类，分别定义如下：
EnumA ↓

```
enum class EnumA(var value: Int) {

    A_1(1),
    A_2(2)

}
```

FakeEnumA ↓

```
enum class FakeEnumA(var value: Int) {

    A_1(1),
    A_2(2)

}
```

代码里进行了一段神奇的操作：

```
val enumA = EnumA.A_1

when (enumA) {
    FakeEnumA.A_1 -> println("I'm A_1")
    FakeEnumA.A_2 -> println("I'm A_2")
    else -> println("else")
}
```

代码正常编译，理论上来说，不同类型比较，应该进入else，而结果确实进入了else。

但是。。这个版本之前，代码执行，输出了“I'm A_1”。。

而在新版本发布后，代码执行， 输出了“else”。。

![kotlin_when](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/img_kotlin_when_1.jpeg)

可以确定的是这段代码相关内容没有任何改动，那么为什么会出现两种不同的结果呢？

## 异常现象

两个不同的类对象比较，理论上比较肯定会不同，但是原代码比较判定为相等

## 问题分析

咋办。。

![kotlin_when]https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/img_kotlin_when_2.jpg)

既然代码没有变过，那么项目有没有其他变更呢？

有的，我们把Kotlin升级了，从1.2升级到了1.3，因为1.3提供了协程。。贼开心。。

抱着算一把命的想法，把变更回滚，Kotlin改回1.2，发现果然复现了原来异常的情况，两个不同的类对象，比较判定为了相等，输出了"I'm A_1",那么基本可以确定是由于Kotlin升级导致的结果。

那么为什么旧版本Kotlin会产生这种现象呢？

Kotlin代码最终也是编译生成字节码跑在JVM上的，那么来看看字节码吧~

__Kotlin1.2的字节码实现__

先看看Kotlin 1.2的时候，这段代码的字节码是怎样的 ↓

```
Code:
   0: aload_0
   1: ldc           #9                  // String args
   3: invokestatic  #15                 // Method kotlin/jvm/internal/Intrinsics.checkParameterIsNotNull:(Ljava/lang/Object;Ljava/lang/String;)V
   6: getstatic     #21                 // Field com/seewo/share/example/when/EnumA.A_1:Lcom/seewo/share/example/when/EnumA;
   9: astore_1                          
  10: aload_1
  12: getstatic     #27                 // Field com/seewo/share/example/when/EnumTestMainKt$WhenMappings.$EnumSwitchMapping$0:[I
  14: swap
  15: invokevirtual #31                 // Method com/seewo/share/example/when/EnumA.ordinal:()I  //取enumA.A_1对应的oridinal()
  18: iaload
  19: tableswitch   { // 1 to 2         // 通过tableswitch，实现对应代码when(enumA)
                 1: 40                  
                 2: 53                  
           default: 66                  
      }
  40: ldc           #33                 // String I'm A_1
  42: astore_2
  43: getstatic     #39                 // Field java/lang/System.out:Ljava/io/PrintStream;
  46: aload_2
  47: invokevirtual #45                 // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
  50: goto          76
  53: ldc           #47                 // String I'm A_2
  55: astore_2
  56: getstatic     #39                 // Field java/lang/System.out:Ljava/io/PrintStream;
  59: aload_2
  60: invokevirtual #45                 // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
  63: goto          76
  66: ldc           #49                 // String else
  68: astore_2
  69: getstatic     #39                 // Field java/lang/System.out:Ljava/io/PrintStream;
  72: aload_2
  73: invokevirtual #45                 // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
  76: return
}

```
从上面的字节码看，好像并没有啥问题，生成映射、取值、获得映射结果，比较。。

比较。。。比。。。较。。。tableswitch。。oridinal。。。oridinal。。。

![kotlin_when](ihttps://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/img_kotlin_when_3.jpeg)

oridinal不是返回的枚举中类型序号吗。。。

所以这个比较只是在比较序号的吗。。。

![kotlin_when](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/img_kotlin_when_4.jpg)

EnumA和FakeEnumA中枚举类型声明的顺序确实是一样的，那如果我把FakeEnumA中的定义顺序换一下，不就正常了吗。。

然后我测试了一把，发现并没有用，结果依然是判定相等，那这是为什么呢。。。

__Kotlin 1.2 中，when的实现__

继续看字节码，发现通过iaload加载了EnumTestMainKt$WhenMappings.$EnumSwitchMapping$中index为0(A.A_1.oridnal)的元素，进入tableswitch进行跳转，那么这个EnumSwitchMapping又是什么呢？

继续看字节码:

```
public final class com.seewo.share.example.when.EnumTestMainKt$WhenMappings {
  public static final int[] $EnumSwitchMapping$0;

  static {};
    Code:
       0: invokestatic  #14                 // Method com/seewo/share/example/when/EnumA.values:()[Lcom/seewo/share/example/when/EnumA;
       3: arraylength
       4: newarray       int
       6: putstatic     #16                 // Field $EnumSwitchMapping$0:[I
       9: getstatic     #16                 // Field $EnumSwitchMapping$0:[I
      12: getstatic     #20                 // Field com/seewo/share/example/when/EnumA.A_1:Lcom/seewo/share/example/when/EnumA;
      15: invokevirtual #24                 // Method com/seewo/share/example/when/EnumA.ordinal:()I
      18: iconst_1
      19: iastore
      20: getstatic     #16                 // Field $EnumSwitchMapping$0:[I
      23: getstatic     #27                 // Field com/seewo/share/example/when/EnumA.A_2:Lcom/seewo/share/example/when/EnumA;
      26: invokevirtual #24                 // Method com/seewo/share/example/when/EnumA.ordinal:()I
      29: iconst_2
      30: iastore
      31: return
}

```

该Mapping中，基于EnumA中的元素个数，创建了一个数组，数组中的对应关系是怎样的呢，我们按照字节码一步步来看:

9         -> getstatic，获取到Mapping中的数组元素
12~15     -> EnumA.A_1.ordinal()，获取到0
18        -> iconst_1，得到整数1
19        -> iastore，存入数组

在iastore前，操作数栈中的元素如下:

>| 1 |
>| ordinal |
>| $EnumSwitchMapping$0 |

而iastore命令调用的栈描述如下：

>| value |
>| index |
>| arrayref |

可以得出其等同于语句$EnumSwitchMapping$0[ordinal]=index

则该Mapping中数组的对应关系为：mapping[0]=1, mapping[1]=2，可以看出，该Mapping实际保存了枚举类型ordinal到tableswitch的映射关系

那么再看回之前的iaload，通过EnumA.A_1的ordinal，从Mapping中加载出的值为1，对应到tableswitch，跳转到40，最终进入"I'm A_1"分支

所以总结下来，跳转过程如下：

![kotlin_when](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/img_kotlin_when_5.png)

那么整个过程看来，和FakeEnumA没有一点关系，那么真的是这样吗？

__EnumSwitchMapping__

在Kotlin 1.2编译生成的字节码中，EnumSwitchMapping主要保存了ordinal到tableswitch的映射关系，字节码如下：

```
public final class com.seewo.share.example.when.EnumTestMainKt$WhenMappings {
  public static final int[] $EnumSwitchMapping$0;

  static {};
    Code:
       0: invokestatic  #14                 // Method com/seewo/share/example/when/EnumA.values:()[Lcom/seewo/share/example/when/EnumA;
       3: arraylength
       4: newarray       int
       6: putstatic     #16                 // Field $EnumSwitchMapping$0:[I
       9: getstatic     #16                 // Field $EnumSwitchMapping$0:[I
      12: getstatic     #20                 // Field com/seewo/share/example/when/EnumA.A_1:Lcom/seewo/share/example/when/EnumA;
      15: invokevirtual #24                 // Method com/seewo/share/example/when/EnumA.ordinal:()I
      18: iconst_1
      19: iastore
      20: getstatic     #16                 // Field $EnumSwitchMapping$0:[I
      23: getstatic     #27                 // Field com/seewo/share/example/when/EnumA.A_2:Lcom/seewo/share/example/when/EnumA;
      26: invokevirtual #24                 // Method com/seewo/share/example/when/EnumA.ordinal:()I
      29: iconst_2
      30: iastore
      31: return
}
```

可以看出整个Mapping的生成似乎和FakeEnumA没有任何关系，但是实际上是这样的吗？

我们尝试改动FakeEnumA和when的代码，进行如下测试：

&emsp;1.FakeEnumA添加一个新的类型A_3，代码如下：

```
enum class FakeEnumA(var value: Int) {
    A_1(1),
    A_2(2),
    A_3(3)
}
```

&emsp;2.when跳转中，把FakeEnumA.A_1改为FakeEnumA.A_3，代码如下：

```
when (enumA) {
    FakeEnumA.A_3 -> println("I'm A_1")
    FakeEnumA.A_2 -> println("I'm A_2")
    else -> println("else")
}
```
以上代码均正常编译通过，如果整个过程和FakeEnumA没有关系的话，那么应该会正常运行，并输出"I'm A_1"。

但是实际执行却抛出异常：

```
Exception in thread "main" java.lang.NoSuchFieldError: A_3
        at com.share.example.when.EnumTestMainKt$WhenMappings.<clinit>(Unknown Source)
        at com.share.example.when.EnumTestMainKt.main(EnumTestMain.kt:17)
```

在运行时抛出了NoSuchFieldError,说明在编译的时候时候，编译器校验正常通过，但在运行的时候，发现A_3找不到了，抛出异常。

此时的EnumSwitchMapping字节码如下：

```
public final class com.seewo.share.example.when.EnumTestMainKt$WhenMappings {
  public static final int[] $EnumSwitchMapping$0;

  static {};
    Code:
       0: invokestatic  #14                 // Method com/seewo/share/example/when/EnumA.values:()[Lcom/seewo/share/example/when/EnumA;
       3: arraylength
       4: newarray       int
       6: putstatic     #16                 // Field $EnumSwitchMapping$0:[I
       9: getstatic     #16                 // Field $EnumSwitchMapping$0:[I
      12: getstatic     #20                 // Field com/seewo/share/example/when/EnumA.A_3:Lcom/seewo/share/example/when/EnumA;
      15: invokevirtual #24                 // Method com/seewo/share/example/when/EnumA.ordinal:()I
      18: iconst_1
      19: iastore
      20: getstatic     #16                 // Field $EnumSwitchMapping$0:[I
      23: getstatic     #27                 // Field com/seewo/share/example/when/EnumA.A_2:Lcom/seewo/share/example/when/EnumA;
      26: invokevirtual #24                 // Method com/seewo/share/example/when/EnumA.ordinal:()I
      29: iconst_2
      30: iastore
      31: return
}
```

可以看到，是在类的静态初始化域中，对类内的数组对象进行了初始化，并赋值，其中12: getstatic尝试获取EnumA中的A_3时，发现找不到对应的枚举类型。

由此可以推断，该Mapping的在编译时依赖于when中的条件分支（FakeEnumA.A_3和FakeEnumA.A_2）进行生成，而生成时，只取了枚举类型的name，并没有判断是否是同一个枚举类型，最终导致了这个异常。。

__Kotlin 1.3 中，when的实现__

综上所述，已经找到了Kotlin 1.2中会进入错误分支的原因，那么为什么Kotlin 1.3中会恢复正常，进入else分支呢？

我们看一看使用Kotlin 1.3编译后生成的字节码：

```
public final class com.seewo.share.example.when.EnumTestMainKt {
  public static final void main(java.lang.String[]);
    Code:
       0: aload_0
       1: ldc           #9                  // String args
       3: invokestatic  #15                 // Method kotlin/jvm/internal/Intrinsics.checkParameterIsNotNull:(Ljava/lang/Object;Ljava/lang/String;)V
       6: getstatic     #21                 // Field com/seewo/share/example/when/EnumA.A_1:Lcom/seewo/share/example/when/EnumA;
       9: astore_1
      10: getstatic     #21                 // Field com/seewo/share/example/when/EnumA.A_1:Lcom/seewo/share/example/when/EnumA;
      13: invokevirtual #25                 // Method com/seewo/share/example/when/EnumA.ordinal:()I
      16: istore_2
      17: getstatic     #31                 // Field java/lang/System.out:Ljava/io/PrintStream;
      20: iload_2
      21: invokevirtual #37                 // Method java/io/PrintStream.println:(I)V
      24: getstatic     #40                 // Field com/seewo/share/example/when/EnumA.A_2:Lcom/seewo/share/example/when/EnumA;
      27: invokevirtual #25                 // Method com/seewo/share/example/when/EnumA.ordinal:()I
      30: istore_2
      31: getstatic     #31                 // Field java/lang/System.out:Ljava/io/PrintStream;
      34: iload_2
      35: invokevirtual #37                 // Method java/io/PrintStream.println:(I)V
      38: getstatic     #45                 // Field com/seewo/share/example/when/FakeEnumA.A_1:Lcom/seewo/share/example/when/FakeEnumA;
      41: invokevirtual #46                 // Method com/seewo/share/example/when/FakeEnumA.ordinal:()I
      44: istore_2
      45: getstatic     #31                 // Field java/lang/System.out:Ljava/io/PrintStream;
      48: iload_2
      49: invokevirtual #37                 // Method java/io/PrintStream.println:(I)V
      52: getstatic     #48                 // Field com/seewo/share/example/when/FakeEnumA.A_2:Lcom/seewo/share/example/when/FakeEnumA;
      55: invokevirtual #46                 // Method com/seewo/share/example/when/FakeEnumA.ordinal:()I
      58: istore_2
      59: getstatic     #31                 // Field java/lang/System.out:Ljava/io/PrintStream;
      62: iload_2
      63: invokevirtual #37                 // Method java/io/PrintStream.println:(I)V
      66: aload_1
      67: astore_2
      68: aload_2
      69: getstatic     #51                 // Field com/seewo/share/example/when/FakeEnumA.A_3:Lcom/seewo/share/example/when/FakeEnumA;
      72: if_acmpne     88
      75: ldc           #53                 // String I'm A_1
      77: astore_3
      78: getstatic     #31                 // Field java/lang/System.out:Ljava/io/PrintStream;
      81: aload_3
      82: invokevirtual #56                 // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
      85: goto          118
      88: aload_2
      89: getstatic     #48                 // Field com/seewo/share/example/when/FakeEnumA.A_2:Lcom/seewo/share/example/when/FakeEnumA;
      92: if_acmpne     108
      95: ldc           #58                 // String I'm A_2
      97: astore_3
      98: getstatic     #31                 // Field java/lang/System.out:Ljava/io/PrintStream;
     101: aload_3
     102: invokevirtual #56                 // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
     105: goto          118
     108: ldc           #60                 // String else
     110: astore_3
     111: getstatic     #31                 // Field java/lang/System.out:Ljava/io/PrintStream;
     114: aload_3
     115: invokevirtual #56                 // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
     118: return
}
```
首先，生成的字节码中已经没有了EnumSwitchMapping这个类，那么再看看字节码，字节码中也没有了tableswitch，而是使用了if_acmpne进行比较判断。

那么if_acmpne是怎么比较的呢 ↓

```
if_acmpne pops the top two object references off the stack and compares them. If the two object references are not equal (i.e. if they refer to different objects), execution branches to the address (pc + branchoffset), where pc is the address of the if_acmpne opcode in the bytecode and branchoffset is a 16-bit signed integer parameter following the if_acmpne opcode in the bytecode. If the object references refer to the same object, execution continues at the next instruction.
```

可以看到，if_acmpne是对两个对象的引用进行比较，如果是两个对象的不相等，则进行跳转。

由此可见if_acmpne进行的是对象引用的比较，而EnumA.A_1与FakeEnumA.A_1属于不同的对象，那么72: if_acmpne比较EnumA.A_1与FakeEnumA.A_1，发现两者不相等，跳转至88: aload_2，加载FakeNumA.A_2，与EnumA.A_1进行比较，仍然不相等，最终跳转到108进入else。

因此，最终输出了"else"。

## 问题原因

（1）Kotlin 1.2 编译实现when语句时，在value为枚举类型的情况下，未进行类型校验，并且使用的Mapping关系映射+tableswitch实现when条件判断和分支跳转；

（2）Kotlin 1.2 在编译生成Mapping关系映射生成代码时，仅判断枚举类型name，而两个枚举类中的命名一致，导致Mapping关系映射正常生成，但是错误映射到不相等的分支，最终输出"I'm A_1"；

（3）Kotlin 1.3 中，对when的编译实现使用了if_acmpne，进行对象引用比较，代码执行按正常逻辑走入else分支。

## 解决方案

调整when比较的条件，使用相同对象进行比较，解决了该问题。
