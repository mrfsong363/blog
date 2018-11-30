---
title: 【JAVA回顾】volatile关键字分析
categories: 编程
tags: JUC
date: 2018-04-08 22:47:45
---

在 Java 并发编程中，要想使并发程序能够正确地执行，必须要保证三条原则，即：原子性、可见性和有序性。只要有一条原则没有被保证，就有可能会导致程序运行不正确。volatile 关键字 被用来保证可见性，即保证共享变量的内存可见性以解决缓存一致性问题。一旦一个共享变量被 volatile 关键字 修饰，那么就具备了两层语义：内存可见性和禁止进行指令重排序。在多线程环境下，volatile 关键字 主要用于及时感知共享变量的修改，并使得其他线程可以立即得到变量的最新值，例如，用于 修饰状态标记量 和 Double-Check (双重检查) 中。
volatile 关键字 虽然从字面上理解起来比较简单，但是要用好不是一件容易的事情。由于 volatile 关键字 是与 内存模型 紧密相关，因此在讲述 volatile 关键字 之前，我们有必要先去了解与内存模型相关的概念和知识，然后回头再分析 volatile 关键字 的实现原理，最后在给出 volatile 关键字 的使用场景。
<!-- more -->

> 原文地址: https://blog.csdn.net/justloveyou_/article/details/53672005

## 一. 内存模型的相关概念

  大家都知道，计算机在执行程序时，每条指令都是在 CPU 中执行的，而执行指令过程中，势必涉及到数据的读取和写入。由于程序运行过程中的临时数据是存放在主存（物理内存）当中的，这时就存在一个问题：由于 CPU 执行速度很快，而从内存读取数据和向内存写入数据的过程跟 CPU 执行指令的速度比起来要慢的多，因此如果任何时候对数据的操作都要通过和内存的交互来进行，会大大降低指令执行的速度。因此，在 CPU 里面就有了 高速缓存（寄存器）。

　　也就是说，在程序运行过程中，会将运算需要的数据从主存复制一份到 CPU 的高速缓存当中，那么， CPU 进行计算时就可以直接从它的高速缓存读取数据和向其中写入数据，当运算结束之后，再将高速缓存中的数据刷新到主存当中。举个简单的例子，比如下面的这段代码：

```java
i = i + 1;
```

　　当线程执行这个语句时，会先从主存当中读取 i 的值，然后复制一份到高速缓存当中，然后 CPU 执行指令对 i 进行加 1 操作，然后将数据写入高速缓存，最后将高速缓存中 i 最新的值刷新到主存当中。

　　这个代码在单线程中运行是没有任何问题的，但是在多线程中运行就会有问题了。在多核 CPU 中，每个线程可能运行于不同的 CPU 中，因此每个线程运行时有自己的高速缓存（对单核 CPU 来说，其实也会出现这种问题，只不过是以线程调度的形式来分别执行的）。本文我们以多核 CPU 为例。

　　比如，同时有两个线程执行这段代码，假如初始时 i 的值为 0，那么我们希望两个线程执行完之后 i 的值变为 2。但是事实会是这样吗？

　　可能存在下面一种情况：初始时，两个线程分别读取 i 的值存入各自所在的 CPU 的高速缓存当中，然后线程 1 进行加 1 操作，然后把 i 的最新值 1 写入到内存。此时线程 2 的高速缓存当中 i 的值还是 0，进行加 1 操作之后，i 的值为 1，然后线程 2 把 i 的值写入内存。

　　最终结果 i 的值是 1，而不是 2 。这就是著名的 **缓存一致性问题** 。通常称这种被多个线程访问的变量为 **共享变量** 。

　　也就是说，**如果一个变量在多个 CPU 中都存在缓存（一般在多线程编程时才会出现），那么就可能存在 _缓存不一致_ 的问题。**

　　为了解决缓存不一致性问题，在 **硬件层面** 上通常来说有以下两种解决方法：

　　1）通过在 **总线加 LOCK# 锁** 的方式 **（在软件层面，效果等价于使用 synchronized 关键字）**；

　　2）通过 **缓存一致性协议** **（在软件层面，效果等价于使用 volatile 关键字）**。

　　在早期的 CPU 当中，是通过在总线上加 LOCK# 锁的形式来解决缓存不一致的问题。因为 CPU 和其他部件进行通信都是通过总线来进行的，如果对总线加 LOCK# 锁的话，也就是说阻塞了其他 CPU 对其他部件访问（如内存），从而使得只能有一个 CPU 能使用这个变量的内存。比如上面例子中， 如果一个线程在执行 i = i + 1，如果在执行这段代码的过程中，在总线上发出了 LCOK# 锁的信号，那么只有等待这段代码完全执行完毕之后，其他 CPU 才能从变量 i 所在的内存读取变量，然后进行相应的操作，这样就解决了缓存不一致的问题。但是上面的方式会有一个问题，**由于在锁住总线期间，其他 CPU 无法访问内存，导致效率低下。**

　　所以，就出现了 **缓存一致性协议** ，其中最出名的就是 Intel 的 MESI 协议。MESI 协议保证了每个缓存中使用的共享变量的副本是一致的。它核心的思想是： **当 CPU 写数据时，如果发现操作的变量是共享变量，即在其他 CPU 中也存在该变量的副本，会发出信号通知其他 CPU 将该变量的缓存行置为无效状态。因此，当其他 CPU 需要读取这个变量时，发现自己缓存中缓存该变量的缓存行是无效的，那么它就会从内存重新读取。**

　　![](http://static.zybuluo.com/Rico123/9ai5r4s44tu3ocj4kzj7nynv/%E7%BC%93%E5%AD%98%E4%B8%80%E8%87%B4%E6%80%A7%E9%97%AE%E9%A2%98.jpg)

* * *

## 二. 并发编程中的三个概念

　　在并发编程中，我们通常会遇到以下三个问题： **原子性问题** ， **可见性问题** 和 **有序性问题** 。我们先看具体看一下这三个概念：

* * *

**1、原子性**

　　**原子性：** 即一个操作或者多个操作 要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。

　　一个很经典的例子就是银行账户转账问题：

　　比如从账户 A 向账户 B 转 1000 元，那么必然包括 2 个操作：从账户 A 减去 1000 元，往账户 B 加上 1000 元。

　　试想一下，如果这两个操作不具备原子性，会造成什么样的后果。假如从账户 A 减去 1000 元之后，操作突然中止。然后又从 B 取出了 500 元，取出 500 元之后，再执行 往账户 B 加上 1000 元 的操作。这样就会导致账户 A 虽然减去了 1000 元，但是账户 B 没有收到这个转过来的 1000 元。所以，这两个操作必须要具备原子性才能保证不出现一些意外的问题。

　　同样地，反映到并发编程中会出现什么结果呢？

　　举个最简单的例子，大家想一下，假如为一个 32 位的变量赋值过程不具备原子性的话，会发生什么后果？

```java
i = 9;
```

　　假若一个线程执行到这个语句时，我们暂且假设为一个 32 位的变量赋值包括两个过程：为低 16 位赋值，为高 16 位赋值。那么就可能发生一种情况：当将低 16 位数值写入之后，突然被中断，而此时又有一个线程去读取 i 的值，那么读取到的就是错误的数据，导致 **数据不一致性** 问题。

* * *

**2、可见性**

　　 **可见性：** 是指当多个线程访问同一个共享变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。

　　举个简单的例子，看下面这段代码：

```java
//线程1执行的代码
int i = 0;
i = 10;

//线程2执行的代码
j = i;
```

　　假若执行 线程 1 的是 CPU1，执行 线程 2 的是 CPU2。由上面的分析可知，当 线程 1 执行 i = 10 这句时，会先把 i 的初始值加载到 CPU1 的高速缓存中，然后赋值为 10，那么在 CPU1 的高速缓存当中 i 的值变为 10 了，却没有立即写入到主存当中。此时，线程 2 执行 j = i，它会先去主存读取 i 的值并加载到 CPU2 的缓存当中，注意此时内存当中 i 的值还是 0，那么就会使得 j 的值为 0，而不是 10。

　　这就是可见性问题，线程 1 对变量 i 修改了之后，线程 2 没有立即看到 线程 1 修改后的值。

* * *

**3、有序性**

　　**有序性：**即程序执行的顺序按照代码的先后顺序执行。举个简单的例子，看下面这段代码：

```java
int i = 0;
boolean flag = false;
i = 1;                //语句1
flag = true;          //语句2
```

　　上面代码定义了一个 int 型 变量，定义了一个 boolean 型 变量，然后分别对两个变量进行赋值操作。从代码顺序上看，语句 1 是在 语句 2 前面的，那么 JVM 在真正执行这段代码的时候会保证 语句 1 一定会在 语句 2 前面执行吗？不一定，为什么呢？这里可能会发生 **指令重排序（Instruction Reorder）**。

　　下面解释一下什么是指令重排序，一般来说，**处理器为了提高程序运行效率，可能会对输入代码进行优化，它不保证程序中各个语句的执行先后顺序同代码中的顺序一致，但是它会保证程序最终执行结果和代码顺序执行的结果是一致的**（单线程情形下）**。**

　　比如上面的代码中，语句 1 和 语句 2 谁先执行对最终的程序结果并没有影响，那么就有可能在执行过程中， 语句 2 先执行而 语句 1 后执行。但是要注意，虽然处理器会对指令进行重排序，但是它会保证程序最终结果会和代码顺序执行结果相同，那么它靠什么保证的呢？再看下面一个例子：

```java
int a = 10;    //语句1
int r = 2;    //语句2
a = a + 3;    //语句3
r = a*a;     //语句4
```

　　这段代码有 4 个语句，那么可能的一个执行顺序是：

　　 　　 　　 　　 　　　![](http://static.zybuluo.com/Rico123/omn745bkm73c4xump83ptmec/%E6%8C%87%E4%BB%A4%E9%87%8D%E6%8E%92%E5%BA%8F.jpg)

　　那么可不可能是这个执行顺序呢： 语句 2　->　语句 1　->　语句 4　->　语句 3

　　答案是不可能，因为处理器在进行重排序时会考虑指令之间的 **数据依赖性**，如果一个指令 Instruction 2 必须用到 Instruction 1 的结果，那么处理器会保证 Instruction 1 会在 Instruction 2 之前执行。

　　虽然 **重排序不会影响单个线程内程序执行的结果**，但是多线程呢？下面，看一个例子：

```java
//线程1:
context = loadContext();   //语句1
inited = true;             //语句2

//线程2:
while(!inited ){
  sleep()
}
doSomethingwithconfig(context);
```

　　上面代码中，由于 语句 1 和 语句 2 没有数据依赖性，因此可能会被重排序。假如发生了重排序，在 线程 1 执行过程中先执行 语句 2，而此时 线程 2 会以为初始化工作已经完成，那么就会跳出 while 循环 ，去执行 doSomethingwithconfig(context) 方法，而此时 context 并没有被初始化，就会导致程序出错。

　　从上面可以看出，**指令重排序不会影响单个线程的执行，但是会影响到线程并发执行的正确性。也就是说，要想使并发程序正确地执行，必须要保证原子性、可见性以及有序性。只要有一个没有被保证，就有可能会导致程序运行不正确。**

* * *

## 三. Java 内存模型

　　在前面谈到了一些关于内存模型以及并发编程中可能会出现的一些问题。下面我们来看一下 **Java 内存模型**，研究一下 Java 内存模型 为我们提供了哪些保证以及在 Java 中提供了哪些方法和机制来让我们在进行多线程编程时能够保证程序执行的正确性。

　　**在 Java 虚拟机规范 中，试图定义一种 **Java 内存模型（Java Memory Model，JMM）** 来屏蔽各个硬件平台和操作系统的内存访问差异，以实现让 Java 程序在各种平台下都能达到一致的内存访问效果。**那么，Java 内存模型 规定了哪些东西呢，它定义了程序中变量的访问规则，往大一点说是定义了程序执行的次序。注意，为了获得较好的执行性能，Java 内存模型并没有限制执行引擎使用处理器的寄存器或者高速缓存来提升指令执行速度，也没有限制编译器对指令进行重排序。也就是说，**在 Java 内存模型 中，也会存在缓存一致性问题和指令重排序的问题。**

　　Java 内存模型 规定所有的变量都是存在主存当中（类似于前面说的物理内存），每个线程都有自己的工作内存（类似于前面的高速缓存）。线程对变量的所有操作都必须在工作内存中进行，而不能直接对主存进行操作，并且每个线程不能访问其他线程的工作内存。

　　举个简单的例子：在 java 中，执行下面这个语句：

```java
i  = 10;
```

　　执行线程必须先在自己的工作线程中对 变量 i 所在的缓存进行赋值操作，然后再写入主存当中，而不是直接将数值 10 写入主存当中。那么，Java 语言本身对原子性、可见性以及有序性 提供了哪些保证呢？

* * *

**1、原子性**

　　在 Java 中，对基本数据类型的变量的 **读取** 和 **赋值** 操作是原子性操作，即这些操作是不可被中断的 ： 要么执行，要么不执行。

　　上面一句话虽然看起来简单，但是理解起来并不是那么容易。看下面一个例子，请分析以下哪些操作是原子性操作：

```java
x = 10;         //语句1
y = x;         //语句2
x++;           //语句3
x = x + 1;     //语句4
```

　　乍一看，有些朋友可能会说上面的四个语句中的操作都是原子性操作。其实 只有 语句 1 是原子性操作，其他三个语句都不是原子性操作。

　　语句 1 是直接将数值 10 赋值给 x，也就是说线程执行这个语句的会直接将数值 10 写入到工作内存中；

　　语句 2 实际上包含两个操作，它先要去读取 x 的值，再将 x 的值写入工作内存。虽然，读取 x 的值以及 将 x 的值写入工作内存这两个操作都是原子性操作，但是合起来就不是原子性操作了；

　　同样的，x++ 和 x = x+1 包括 3 个操作：读取 x 的值，进行加 1 操作，写入新的值。

　　所以，上面四个语句只有 语句 1 的操作具备原子性。也就是说，**只有简单的读取、赋值（而且必须是将数字赋值给某个变量，变量之间的相互赋值不是原子操作）才是原子操作。**

　　不过，这里有一点需要注意：**在 32 位平台下，对 64 位数据的读取和赋值是需要通过两个操作来完成的，不能保证其原子性。但是好像在最新的 JDK 中，JVM 已经保证对 64 位数据的读取和赋值也是原子性操作了。**

　　从上面可以看出，**Java 内存模型只保证了基本读取和赋值是原子性操作，如果要实现更大范围操作的原子性，可以通过 synchronized 和 Lock 来实现。**由于 synchronized 和 Lock 能够保证任一时刻只有一个线程执行该代码块，那么自然就不存在原子性问题了，从而保证了原子性。

* * *

**2、可见性**

　　**对于可见性，Java 提供了 volatile 关键字 来保证可见性。**

　　当一个共享变量被 volatile 修饰时，它会保证修改的值会立即被更新到主存，当有其他线程需要读取时，它会去内存中读取新值。而普通的共享变量不能保证可见性，因为普通共享变量被修改之后，什么时候被写入主存是不确定的，当其他线程去读取时，此时内存中可能还是原来的旧值，因此无法保证可见性。

　　另外，通过 synchronized 和 Lock 也能够保证可见性，synchronized 和 Lock 能保证同一时刻只有一个线程获取锁然后执行同步代码，并且 **在释放锁之前会将对变量的修改刷新到主存当中**，因此可以保证可见性。

* * *

**3、有序性**

  在 Java 内存模型中，允许编译器和处理器对指令进行重排序，但是重排序过程不会影响到单线程程序的执行，却会影响到多线程并发执行的正确性。

  在 Java 中，可以通过 volatile 关键字来保证一定的 “有序性”（具体原理在下一节讲述）。另外，我们千万不能想当然地认为，可以通过 synchronized 和 Lock 来保证有序性，也就是说，不能由于 synchronized 和 Lock 可以让线程串行执行同步代码，就说它们可以保证指令不会发生重排序，这根本不是一个粒度的问题。

另外，Java 内存模型具备一些先天的 “有序性”，即不需要通过任何手段就能够得到保证的有序性，这个通常也称为 **happens-before 原则**。如果两个操作的执行次序无法从 happens-before 原则推导出来，那么它们就不能保证它们的有序性，虚拟机可以随意地对它们进行重排序。

下面就来具体介绍下 happens-before 原则（先行发生原则）：

*   **程序次序规则**：一个线程内，按照代码顺序，书写在前面的操作先行发生于书写在后面的操作；

*   **锁定规则**：一个 unLock 操作先行发生于后面对同一个锁额 lock 操作；

*   **volatile 变量规则：**对一个变量的写操作先行发生于后面对这个变量的读操作；

*   **传递规则**：如果操作 A 先行发生于操作 B，而操作 B 又先行发生于操作 C，则可以得出操作 A 先行发生于操作 C ；

*   **线程启动规则**：Thread 对象的 start() 方法先行发生于此线程的每个一个动作；

*   **线程中断规则**：对线程 interrupt() 方法的调用先行发生于被中断线程的代码检测到中断事件的发生；

*   **线程终结规则**：线程中所有的操作都先行发生于线程的终止检测，我们可以通过 Thread.join() 方法结束、Thread.isAlive() 的返回值手段检测到线程已经终止执行；

*   **对象终结规则**：一个对象的初始化完成先行发生于他的 finalize() 方法的开始。

    这八条原则摘自《深入理解 Java 虚拟机》。这八条规则中，前四条规则是比较重要的，后四条规则都是显而易见的。下面我们来解释一下前四条规则：

    对于程序次序规则来说，我的理解就是一段程序代码的执行在单个线程中看起来是有序的。注意，虽然这条规则中提到 “书写在前面的操作先行发生于书写在后面的操作”，这个应该是程序看起来执行的顺序是按照代码顺序执行的，因为虚拟机可能会对程序代码进行指令重排序。虽然进行重排序，但是最终执行的结果是与程序顺序执行的结果一致的，它只会对不存在数据依赖性的指令进行重排序。因此，在单个线程中，程序执行看起来是有序执行的，这一点要注意理解。事实上，**这个规则是用来保证程序在单线程中执行结果的正确性，但无法保证程序在多线程中执行的正确性。**

    第二条规则也比较容易理解，也就是说无论在单线程中还是多线程中，同一个锁如果出于被锁定的状态，那么必须先对锁进行了释放操作，后面才能继续进行 lock 操作。

    第三条规则是一条比较重要的规则，也是后文将要重点讲述的内容。直观地解释就是，如果一个线程先去写一个变量，然后一个线程去进行读取，那么写入操作肯定会先行发生于读操作。

    第四条规则实际上就是体现 happens-before 原则具备传递性。

***

## 四. 深入剖析 volatile 关键字

**1、volatile 关键字的两层语义**
一旦一个共享变量（类的成员变量、类的静态成员变量）被 volatile 修饰后，那么就具备了两层语义：
** 1）保证了不同线程对共享变量进行操作时的可见性，即一个线程修改了某个变量的值，这个新值对其他线程来说是 **立即可见** 的；**
** 2）禁止进行指令重排序。**

先看一段代码，假如 线程 1 先执行，线程 2 后执行：

```java
//线程1
boolean stop = false;
while(!stop){
    doSomething();
}

//线程2
stop = true;
```

这段代码是很典型的一段代码，很多人在中断线程时可能都会采用这种标记办法。但是事实上，这段代码会完全运行正确么？即一定会将线程中断么？不一定，也许在大多数时候，这个代码能够把线程中断，但是也有可能会导致无法中断线程（虽然这个可能性很小，但是只要一旦发生这种情况就会造成死循环了）。

![](https://img-blog.csdn.net/20170116141631367?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvanVzdGxvdmV5b3Vf/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

下面解释一下这段代码为何有可能导致无法中断线程。在前面已经解释过，如上图所示，每个线程在运行过程中都有自己的工作内存，那么 线程 1 在运行的时候，会将 stop 变量的值拷贝一份放在自己的工作内存当中。那么，当 线程 2 更改了 stop 变量 的值之后，可能会出现以下两种情形：

*   线程 2 对变量的修改没有立即刷入到主存当中；

*   即使 线程 2 对变量的修改立即反映到主存中，线程 1 也可能由于没有立即知道 线程 2 对 stop 变量的更新而一直循环下去。

这两种情形都会导致 线程 1 处于死循环。但是，用 volatile 关键字 修饰后就变得不一样了，如下图所示：

① 使用 volatile 关键字会强制将修改的值立即写入主存;

② 使用 volatile 关键字的话，当 线程 2 进行修改时，会导致 线程 1 的工作内存中缓存变量 stop 的缓存行无效（反映到硬件层的话，就是 CPU 的 L1 或者 L2 缓存中对应的缓存行无效）；

③ 由于 线程 1 的工作内存中缓存变量 stop 的缓存行无效，所以，线程 1 再次读取变量 stop 的值时会去主存读取。

综上，**在 线程 2 修改 stop 值时（当然这里包括两个操作，修改 线程 2 工作内存中的值，然后将修改后的值写入内存），会使得 线程 1 的工作内存中缓存变量 stop 的缓存行无效，然后 线程 1 读取时，会发现自己的缓存行无效从而去对应的主存读取最新的值 。简化一下，通过使用 volatile 关键字，如下图所示，线程会及时将变量的新值更新到主存中，并且保证其他线程能够立即读到该值。**这样，线程 1 读取到的就是最新的、正确的值。
![](https://img-blog.csdn.net/20170319195723202?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvanVzdGxvdmV5b3Vf/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

* * *

下面通过两个例子更好地了解关键字 volatile 的作用。下面先看 **示例 1**

```java
//资源类
class MyList {

    // 临界资源
    private  List list = new ArrayList();

    // 对临界资源的访问
    public void add() {
        list.add("rico");
    }

    public int size() {
        return list.size();
    }
}

// 线程B
class ThreadB extends Thread {

    private MyList list;

    public ThreadB(MyList list) {
        super();
        this.list = list;
    }

    @Override
    public void run() { // 任务 B
        try {
            while (true) {
                if (list.size() == 2) {
                    System.out.println("list中的元素个数为2了，线程b要退出了！");
                    throw new InterruptedException();
                }
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

// 线程A
class ThreadA extends Thread {

    private MyList list;

    public ThreadA(MyList list) {
        super();
        this.list = list;
    }

    @Override
    public void run() { // 任务 A
        try {
            for (int i = 0; i < 3; i++) {
                list.add();
                System.out.println("添加了" + (i + 1) + "个元素");
                Thread.sleep(1000);
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

// 测试
public class Test {

    public static void main(String[] args) {
        MyList service = new MyList();

        ThreadA a = new ThreadA(service);
        a.setName("A");
        a.start();

        ThreadB b = new ThreadB(service);
        b.setName("B");
        b.start();
    }
}
```

运行结果如下所示：

![](http://static.zybuluo.com/Rico123/jukkuql2mr0ea0wkj5e7fns2/volatile%E5%85%B3%E9%94%AE%E5%AD%97%E4%BE%8B%E8%AF%811.png)

* * *

![](http://static.zybuluo.com/Rico123/l1j3s9027uv0m0pd0kmh9yxm/volatile%E5%85%B3%E9%94%AE%E5%AD%97%E4%BE%8B%E8%AF%812.png)

第一个运行结果是在没有使用 volatile 关键字的情况下产生的，第二个运行结果是在使用 volatile 关键字的情况下产生的。

特别地，博友 qq_27571221(王小军 08) 提到， “若将 类 ThreadA 中的 run() 方法中的 Thread.sleep(1000); 去掉，上述两种运行结果都有可能出现。” 事实上，之所以会出现这种情况，究其根本，**是由线程获得 CPU 执行的不确定性引起的**。也就是说，在使用 volatile 关键字修饰共享变量 list 的前提下，去掉代码 Thread.sleep(1000); 后，之所以也会出现第一种运行结果是因为存在这样一种情形：**线程 A 早已运行结束但线程 B 才刚刚开始执行或尚未开始执行，即串行执行的情形。**总的来说，在类 ThreadA 中的 run() 方法中添加 Thread.sleep(1000); 的原因就是 **为了保证线程 A、B 能交替执行，防止上述情形的出现。

* * *

**示例 2：**

```java
public class TestVolatile {

    public static void main(String[] args) {

        ThreadDemo td = new ThreadDemo();
        new Thread(td, "ThreadDemo").start();

        while (true) {
            // 加上下面三句代码的任意一句，程序都会正常结束：
            // System.out.println("!!");                              //...语句1
            // synchronized (TestVolite.class) {}                     //...语句2
            //TestVolite.test2();                                    //...语句3

            // 若只加上下面一句代码，程序都会死循环：
            // TestVolite.test1();                                  //...语句4

            if (td.flag) {
                System.out.println("线程 " + Thread.currentThread().getName()
                        + " 即将跳出while循环体... ");
                break;
            }
        }
    }

    public static void test1() {}

    public synchronized static void test2() {}
}

class ThreadDemo implements Runnable {

    public boolean flag = false;

    @Override
    public void run() {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
        }
        flag = true;
        System.out.println("线程 " + Thread.currentThread().getName() + " 执行完毕： "
                + "置  flag= " + flag + " ...");
    }
}
```

上述程序运行结果如下图：
![](http://static.zybuluo.com/Rico123/txrxamgzpkvpuqfic4i8g0vy/Case0.png)

**下面对该程序分以下 5 种情形进行修改并讨论，如下所示 ： **

* * *

**Case 1：只用 volatile 关键字修饰 类 ThreadDemo 中的共享变量 flag**

运行结果为：
![](http://static.zybuluo.com/Rico123/5ddn0d5op9w0b094zrorzkge/Case%201.png)

* * *

**Case 2：只取消对语句 1 的注释**

运行结果为：
![](http://static.zybuluo.com/Rico123/ezhdml7fgza05f25ddjgdcgr/Case%202.png)

**Case 3：只取消对语句 2 的注释**

运行结果为：
![](http://static.zybuluo.com/Rico123/w12i86014gbs71tiurnuzftx/Case3.png)
* * *

**Case 4：只取消对语句 3 的注释**
运行结果为：
![](http://static.zybuluo.com/Rico123/8jsmj2rn42sxjishrt2v1qn1/Case%204.png)
* * *

**Case 5：只取消对语句 4 的注释**
运行结果为：
![](http://static.zybuluo.com/Rico123/1rkavxzq5s56qgpa4r3s5rre/Case5.png)
* * *

关于上面五种情形，**情形 1** 和 **情形 5** 很容易理解，此不赘述。

但是，对于上面的 **第 2、3、4 三种情形**，可能有很多朋友就不能理解了，特别是 **第 2 种情形**。其实，这三种情形都反映了一个问题：**在我们不使用 volatile 关键字修饰共享变量去保证其可见性时，那么线程是不是始终一直从自己的工作内存中读取变量的值呢？ 什么情况下，线程工作内存中的变量值才会与主存中的同步并取得一致状态呢？**

**事实上，除了 volatile 可以保证内存可见性外，synchronized 也可以保证可见性，因为每次运行 synchronized 块 或者 synchronized 方法都会导致线程工作内存与主存的同步，使得其他线程可以取得共享变量的最新值。也就是说，synchronized 语义范围不但包括 volatile 具有的可见性，也包括原子性，但不能禁止指令重排序，这是二者一个功能上的差异。**说到这里，朋友应该就理解了 **情形 3** 和 **情形 4** 了。但是，**情形 2** 怎么也会导致类似于 **情形 3** 和 **情形 4** 的效果呢？ 因为 System.out.println() 方法里面包含 synchronized 块， 我们看完它的源码就大彻大悟了，如下：

```java
public void println(String x) {
    synchronized (this) {         // synchronized 块
        print(x);
        newLine();
    }
}
```

**_在此，特别感谢 CSDN 博友 Geek-k(u010395144) 所提出的问题，是他的提问，我才能更好的诠释这个问题，更好的提升自己，更好的帮助更多的朋友。_**更多关于 synchronized 关键字 的介绍请移步我的另一篇博文[《Java 并发：内置锁 Synchronized》](http://blog.csdn.net/justloveyou_/article/details/54381099)。更多关于 Java 多线程 方面的知识请移步我的专栏[《Java 并发编程学习笔记》](http://blog.csdn.net/column/details/14542.html)。

* * *

**2、volatile 保证原子性吗？**

从上面知道， volatile 关键字保证了操作的可见性，但是 volatile 能保证对变量的操作是原子性吗？

下面看一个例子：
```java
//线程类
class MyThread extends Thread {
    // volatile 共享静态变量，类成员
    public volatile static int count;

    private static void addCount() {
        for (int i = 0; i < 100; i++) {
            count++;
        }
        System.out.println("count=" + count);
    }

    @Override
    public void run() {
        addCount();
    }
}

//测试类
public class Run {
    public static void main(String[] args) {
        //创建 100个线程并启动
        MyThread[] mythreadArray = new MyThread[100];
        for (int i = 0; i < 100; i++) {
            mythreadArray[i] = new MyThread();
        }

        for (int i = 0; i < 100; i++) {
            mythreadArray[i].start();
        }
    }
}/* Output(循环):
       ... ...
       count=9835
 *///:~
```

大家想一下这段程序的输出结果是多少？也许有些朋友认为是 10000。但是事实上运行它会发现每次运行结果都不一致，都是一个 小于 10000 的数字。可能有的朋友就会有疑问，不对啊，上面是对变量 count 进行自增操作，由于 volatile 保证了可见性，那么在每个线程中对 count 自增完之后，在其他线程中都能看到修改后的值啊，所以有 100 个 线程分别进行了 100 次操作，那么最终 count 的值应该是 100*100=10000。

**这里面就有一个误区了，volatile 关键字能保证可见性没有错，但是上面的程序错在没能保证原子性。可见性只能保证每次读取的是最新的值，但是 volatile 没办法保证对变量的操作的原子性。**在前面已经提到过，自增操作是不具备原子性的，它包括 读取变量的原始值、进行加 1 操作 和 写入工作内存 三个原子操作。那么就是说，这三个子操作可能会分割开执行，所以就有可能导致下面这种情况出现：

假如某个时刻 变量 count 的值为 10，线程 1 对变量进行自增操作，线程 1 先读取了 变量 count 的原始值，然后 线程 1 被阻塞了；然后，线程 2 对变量进行自增操作，线程 2 也去读取 变量 count 的原始值，由于 线程 1 只是对 变量 count 进行读取操作，而没有对变量进行修改操作，所以不会导致 线程 2 的工作内存中缓存变量 count 的缓存行无效，所以 线程 2 会直接去主存读取 count 的值 ，发现 count 的值是 10，然后进行加 1 操作。注意，此时 线程 2 只是执行了 **count + 1** 操作，还没将其值写到 线程 2 的工作内存中去！此时线程 2 被阻塞，线程 1 进行加 1 操作时，注意操作数 count 仍然是 10！然后，线程 2 把 11 写入工作内存并刷到主内存。虽然此时 线程 1 能感受到 线程 2 对 count 的修改，但由于线程 1 只剩下对 count 的**写操作**了，而不必对 count 进行**读操作**了，所以此时 线程 2 对 count 的修改并不能影响到 线程 1。于是，线程 1 也将 11 写入工作内存并刷到主内存。也就是说，两个线程分别进行了一次自增操作后，count 只增加了 1。下图演示了这种情形：
![](https://img-blog.csdn.net/20170319194031555?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvanVzdGxvdmV5b3Vf/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

进一步地，将上述代码修改成下面示例的样子以后，这个问题就迎刃而解：

```java
//线程类
class MyThread extends Thread {
    // 既然使用 synchronized关键字 ，就没必要使用 volatile关键字了
    public static int count;

    //注意必须添加 static 关键字，这样synchronized 与 static 锁的就是 Mythread.class 对象了，
    //也就达到同步效果了
    private synchronized static void addCount() {
        for (int i = 0; i < 100; i++) {
            count++;
        }
        System.out.println("count=" + count);
    }

    @Override
    public void run() {
        addCount();
    }
}

//测试类
public class Run {
    public static void main(String[] args) {
        //创建 100个线程并启动
        MyThread[] mythreadArray = new MyThread[100];
        for (int i = 0; i < 100; i++) {
            mythreadArray[i] = new MyThread();
        }

        for (int i = 0; i < 100; i++) {
            mythreadArray[i].start();
        }
    }
}
```

**使用 Lock 和 Java 1.5 所提供的 java.util.concurrent.atomic 包来保证线程安全性将在后面的博文中进行介绍。**

* * *

## 五. 使用 volatile 关键字的场景

**synchronized 关键字是防止多个线程同时执行一段代码，那么就会很影响程序执行效率；而 volatile 关键字在某些情况下性能要优于 synchronized，但是要注意 volatile 关键字是无法替代 synchronized 关键字的，因为 volatile 关键字无法保证操作的原子性。**通常来说，使用 volatile 必须具备以下两个条件：

**1）对变量的写操作不依赖于当前值；**
**2）该变量没有包含在具有其他变量的不变式中。**

实际上，这些条件表明，**可以被写入 volatile 变量的这些有效值 **独立于任何程序的状态**，包括变量的当前状态。**事实上，上面的两个条件就是保证对 该 volatile 变量 的操作是原子操作，这样才能保证使用 volatile 关键字 的程序在并发时能够正确执行。

**特别地，关键字 volatile 主要使用的场合是:**

**在多线程环境下及时感知共享变量的修改，并使得其他线程可以立即得到变量的最新值。**
* * *

**1、状态标记量**
```java
// 示例 1
volatile boolean flag = false;

while(!flag){
    doSomething();
}

public void setFlag() {
    flag = true;
}
```

```java
// 示例 2
volatile boolean inited = false;

//线程1:
context = loadContext();
inited = true;

//线程2:
while(!inited ){
    sleep()
}
doSomethingwithconfig(context);
```

更多关于 **volatile 在状态标记量方面的应用**，请移步我的博文[《Java 并发：线程间通信与协作》](http://blog.csdn.net/justloveyou_/article/details/54929949)。

* * *

**2、Double-Check (双重检查)**

```java
class Singleton{
    private volatile static Singleton instance = null;

    private Singleton() {

    }

    public static Singleton getInstance() {
        if(instance==null) {
            synchronized (Singleton.class) {
                if(instance==null)
                    instance = new Singleton();
            }
        }
        return instance;
    }
}
```

　　更多关于 **Double-Check (双重检查) 的定义与应用场景** 的介绍，请移步我的博文[《彻头彻尾理解单例模式与多线程》](http://blog.csdn.net/justloveyou_/article/details/64127789)。

* * *

## 六. 小结

关键字 volatile 与内存模型紧密相关，是线程同步的轻量级实现，其性能要比 synchronized 关键字 好。在作用对象和作用范围上， volatile 用于修饰变量，而 synchronized 关键字 用于修饰方法和代码块，而且 synchronized 语义范围不但包括 volatile 拥有的可见性，还包括 volatile 所不具有的原子性，但不包括 volatile 拥有的有序性，即允许指令重排序。因此，在多线程环境下，volatile 关键字 主要用于及时感知共享变量的修改，并保证其他线程可以及时得到变量的最新值。可用以下文氏图表示 synchronized 和 volatile 语义范围：

![](https://img-blog.csdn.net/20170320172306900?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvanVzdGxvdmV5b3Vf/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
