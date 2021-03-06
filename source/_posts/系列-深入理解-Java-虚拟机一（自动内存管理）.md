---
title: 系列 | 深入理解Java虚拟机一（自动内存管理机制）
date: 2019-08-20 00:17:35
categories: JVM
tags: JVM
---

> 首发于微信公众号：**BaronTalk**，欢迎关注！

书籍真的是常读常新，古人说「书读百遍其义自见」还是蛮有道理的。周志明老师的这本《深入理解 Java 虚拟机》我细读了不下三遍，每一次阅读都有新的收获，每一次阅读对 Java 虚拟机的理解就更进一步。因而萌生了将读书笔记整理成文的想法，一是想检验下自己的学习成果，对学习内容进行一次系统性的复盘；二是给还没接触过这部好作品的同学推荐下，在阅读这部佳作之前能通过我的文章一窥书中的精华。
<!-- more -->
原想着一篇文章就够了，但写着写着就发现篇幅大大超出了预期。看来还是功力不够，索性拆成了六篇文章，分别从**自动内存管理机制**、**类文件结构**、**类加载机制**、**字节码执行引擎**、**程序编译与代码优化**、**高效并发**六个方面来做更加细致的介绍。本文先说说 Java 虚拟机的自动内存管理机制。

## 一. 运行时数据区

Java 虚拟机在执行 Java 程序的过程中会把它所管理的内存区域划分为若干个不同的数据区域。这些区域都有各自的用途，以及创建和销毁的时间，有些区域随着虚拟机进程的启动而存在，有些区域则是依赖线程的启动和结束而建立和销毁。Java 虚拟机所管理的内存被划分为如下几个区域：

<div align="center"><img src="https://resources.baronzhang.com/blog/jvm/1/运行时数据区.png" width="60%"/></div>

### 程序计数器

程序计数器是一块较小的内存区域，可以看做是当前线程所执行的字节码的行号指示器。在虚拟机的概念模型里，字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成。「属于线程私有的内存区域」

### Java 虚拟机栈

就是我们平时所说的栈，每个方法被执行时，都会创建一个栈帧（Stack Frame）用于存储局部变量表、操作栈、动态链接、方法出口等信息。**每个方法从被调用到执行完成的过程，就对应着一个栈帧在虚拟机栈中从出栈到入栈的过程。**「属于线程私有的内存区域」

> **局部变量表**：局部变量表是 Java 虚拟机栈的一部分，存放了编译器可知的基本数据类型（boolean、byte、char、short、int、float、long、double）、对象引用（reference 类型，不等同于对象本身，根据不同的虚拟机实现，它可能是一个指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄或者其他与此对象相关的位置）和 returnAddress 类型（指向了一条字节码指令的地址）。

### 本地方法栈

和虚拟机栈类似，只不过虚拟机栈为虚拟机执行的 Java 方法服务，本地方法栈为虚拟机使用的 Native 方法服务。「属于线程私有的内存区域」

### Java 堆

对大多数应用而言，Java 堆是虚拟机所管理的内存中最大的一块，是被所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一作用就是存放对象实例，几乎所有的对象实例都是在这里分配的（不绝对，在虚拟机的优化策略下，也会存在栈上分配、标量替换的情况，后面的章节会详细介绍）。Java 堆是 GC 回收的主要区域，因此很多时候也被称为 GC 堆。从内存回收的角度看，Java 堆还可以被细分为新生代和老年代；再细一点新生代还可以被划分为 Eden Space、From Survivor Space、To Survivor Space。从内存回收的角度看，线程共享的 Java 堆可能划分出多个线程私有的分配缓冲区（Thread Local Allocation Buffer，TLAB）。「属于线程共享的内存区域」

### 方法区

用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。「属于线程共享的内存区域」

> **运行时常量池**: 运行时常量池是方法区的一部分，Class 文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息就是常量池（Constant Pool Table），用于存放在编译期生成的各种字面量和符号引用。

> **直接内存**：直接内存（Direct Memory）并不是虚拟机运行时数据区的一部分，也不是 Java 虚拟机规范中定义的内存区域。Java 中的 NIO 可以使用 Native 函数直接分配堆外内存，然后通过一个存储在 Java 堆中的 DiectByteBuffer 对象作为这块内存的引用进行操作。这样能在一些场景显著提高性能，因为避免了在 Java 堆和 Native 堆中来回复制数据。直接内存不受 Java 堆大小的限制。

## 二. 对象的创建、内存布局及访问定位

前面介绍了 Java 虚拟机的运行时数据区，了解了虚拟机内存的情况。接下来我们看看对象是如何创建的、对象的内存布局是怎样的以及对象在内存中是如何定位的。

### 2.1 对象的创建

要创建一个对象首先得在 Java 堆中（不绝对，后面介绍虚拟机优化策略的时候会做详细介绍）为这个要创建的对象分配内存，分配内存的过程要保证并发安全，最后再对内存进行相应的初始化，这一系列的过程完成后，一个真正的对象就被创建了。

#### 内存分配

先说说内存分配，当虚拟机遇到一条 new 指令时，首先将去检查这个指令的参数是否能够在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已被加载、解析和初始化。如果没有，那必须先执行相应的类加载过程。在类加载检查通过后，虚拟机将为新生对象分配内存。对象所需的内存大小在类加载完成后便可完全确定，为对象分配内存空间的任务等同于把一块确定大小的内存从 Java 堆中划分出来。

<div align="center"><img src="https://resources.baronzhang.com/blog/jvm/1/JVM 创建对象.png" width="45%"/></div>

在 Java 堆中划分内存涉及到两个概念：**指针碰撞（Bump the Pointer）**、**空闲列表（Free List）**。

* 如果 Java 堆中的内存绝对规整，所有用过的内存都放在一边，空闲的内存放在另一边，中间放着一个指针作为分界点的指示器，那所分配的内存就仅仅是把指针往空闲空间那边挪动一段与对象大小相等的距离，这种分配方式称为**「指针碰撞」**。

* 如果 Java 堆中的内存并不是规整的，已使用的内存和空闲的内存相互交错，那就没办法简单的进行指针碰撞了。虚拟机必须维护一个列表来记录哪些内存是可用的，在分配的时候从列表中找到一块足够大的空间划分给对象实例，并更新列表上的记录，这种分配方式称为**「空闲列表」**。

选择哪种分配方式是由 Java 堆是否规整来决定的，而 Java 堆是否规整又由所采用的垃圾收集器是否带有压缩整理功能决定。

![JVM 内存分配的两种方式](https://resources.baronzhang.com/blog/jvm/1/内存分配的两种方式.png)

#### 保证并发安全

对象的创建在虚拟机中是一个非常频繁的行为，哪怕只是修改一个指针所指向的位置，在并发情况下也是不安全的，可能出现正在给对象 A 分配内存，指针还没来得及修改，对象 B 又同时使用了原来的指针来分配内存的情况。解决这个问题有两种方案：

* 对分配内存空间的动作进行同步处理（采用 CAS + 失败重试来保障更新操作的原子性）；

* 把内存分配的动作按照线程划分在不同的空间之中进行，即每个线程在 Java 堆中预先分配一小块内存，称为本地线程分配缓冲（Thread Local Allocation Buffer, TLAB）。哪个线程要分配内存，就在哪个线程的 TLAB  上分配。只有 TLAB 用完并分配新的 TLAB 时，才需要同步锁。

![内存分配时保证线程安全的两种方式](https://resources.baronzhang.com/blog/jvm/1/内存分配时保证线程安全的两种方式.png)

#### 初始化

内存分配完后，虚拟机要将分配到的内存空间初始化为零值（不包括对象头），如果使用了 TLAB，这一步会提前到 TLAB 分配时进行。这一步保证了对象的实例字段在 Java 代码中可以不赋初始值就直接使用。

接下来设置对象头（Object Header）信息，包括对象是哪个类的实例、如何找到类的元数据、对象的 Hash、对象的 GC 分代年龄等。

这一系列动作完成之后，紧接着会执行 <init> 方法，把对象按照程序员的意图进行初始化，这样一个真正意义上的对象就产生了。

JVM 中对象的创建过程大致如下图：

<div align="center"><img src="https://resources.baronzhang.com/blog/jvm/1/对象创建的完整过程.png" width="70%"/></div>

### 2.2 对象的内存布局

在 HotSpot 虚拟机中，对象在内存中的布局可以分为 3 块：**对象头（Header）**、**实例数据（Instance Data）**和**对齐填充（Padding）**。

#### 对象头

对象头包含两部分信息，第一部分用于存储对象自身的运行时数据，比如哈希码（HashCode）、GC 分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳等，这部分数据称之为 Mark Word。对象头的另一部分是类型指针，即对象指向它的类元数据指针，虚拟机通过它来确定对象是哪个类的实例；如果是数组，对象头中还必须有一块用于记录数组长度的数据。（并不是所有所有虚拟机的实现都必须在对象数据上保留类型指针，在下一小节介绍「对象的访问定位」的时候再做详细说明）。

#### 实例数据

对象真正存储的有效数据，也是在程序代码中所定义的各种字段内容。

#### 对齐填充

无特殊含义，不是必须存在的，仅作为占位符。

### 2.3 对象的访问定位

Java 程序需要通过栈上的 reference 信息来操作堆上的具体对象。根据不同的虚拟机实现，主流的访问对象的方式主要有**句柄访问**和**直接指针**两种。

#### 句柄访问

Java 堆中划分出一块内存来作为句柄池，reference 中存储的就是对象的句柄地址，而句柄中包含了对象实例数据与类型数据各自的具体地址信息。

<div align="center"><img src="https://resources.baronzhang.com/blog/jvm/1/对象访问定位-句柄.png" width="75%"/></div>

使用句柄访问的好处就是 reference 中存储的是稳定的句柄地址，在对象被移动时只需要改变句柄中实例数据的指针，而 reference 本身不需要修改。

#### 直接指针

在对象头中存储类型数据相关信息，reference 中存储的对象地址。

<div align="center"><img src="https://resources.baronzhang.com/blog/jvm/1/对象访问定位-直接指针.png" width="75%"/></div>

使用直接指针访问的好处是速度更快，它节省了一次指针定位的开销。由于对象访问在 Java 中非常频繁，因此这类开销积少成多也是一项非常可观的执行成本。HotSpot 中采用的就是这种方式。

## 三. 垃圾回收器与内存分配策略

在前面我们介绍 JVM 运行时数据区的时候说过，程序计数器、虚拟机栈、本地方法栈 3 个区域随线程而生，随线程而死；栈中的栈帧随着方法的进入和退出而有条不紊的执行着入栈和出栈的操作。每一个栈帧中分配多少内存基本上在数据结构确定下来的时候就已经知道了，因此这几个区域内存的分配和回收是具有确定性的，所以不用过度考虑内存回收的问题，因为在方法结束或者线程结束时，内存就跟着回收了。

而 Java 堆和方法区则不一样，一个接口中的多个实现类需要的内存可能不一样，一个方法中的多个分支需要的内存也可能不一样，我们只有在程序运行期才能知道会创建哪些对象，这部分内存的分配和回收是动态的，垃圾收集器要关注的就是这部分内存。

### 3.1 对象回收的判定规则

垃圾收集器在做垃圾回收的时候，首先需要判定的就是哪些内存是需要被回收的，哪些对象是「存活」的，是不可以被回收的；哪些对象已经「死掉」了，需要被回收。

#### 引用计数法

判断对象存活与否的一种方式是「引用计数」，即对象被引用一次，计数器就加 1，如果计数器为 0 则判断这个对象可以被回收。但是引用计数法有一个很致命的缺陷就是它无法解决循环依赖的问题，因此现在主流的虚拟机基本不会采用这种方式。

#### 可达性分析算法

可达性分析算法又叫根搜索算法，该算法的基本思想就是通过一系列称为「GC Roots」的对象作为起始点，从这些起始点开始往下搜索，搜索所走过的路径称为引用链，当一个对象到 GC Roots 对象之间没有任何引用链的时候（不可达），证明该对象是不可用的，于是就会被判定为可回收对象。

<div align="center"><img src="https://resources.baronzhang.com/blog/jvm/1/可达性分析.png" width="65%"/></div>

在 Java 中可作为 GC Roots 的对象包含以下几种：

* 虚拟机栈（栈帧中的本地变量表）中引用的对象；
* 方法区中类静态属性引用的对象；
* 方法区中常量引用的对象；
* 本地方法栈中 JNI（Native 方法）引用的对象。

#### Java 中是四种引用类型

无论是通过引用计数器还是通过可达性分析来判断对象是否可以被回收都设计到「引用」的概念。在 Java 中，根据引用关系的强弱不一样，将引用类型划为强引用（Strong Reference）、软引用（Soft Reference）、弱引用（Weak Reference）和虚引用（Phantom Reference）。

**强引用**：`Object obj = new Object()`这种方式就是强引用，只要这种强引用存在，垃圾收集器就永远不会回收被引用的对象。

**软引用**：用来描述一些有用但非必须的对象。在 OOM 之前垃圾收集器会把这些被软引用的对象列入回收范围进行二次回收。如果本次回收之后还是内存不足才会触发 OOM。在 Java 中使用 SoftReference 类来实现软引用。

**弱引用**：同软引用一样也是用来描述非必须对象的，但是它的强度比软引用更弱一些，被弱引用关联的对象只能生存到下一次垃圾收集发生之前。当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。在 Java 中使用 WeakReference 类来实现。 

**虚引用**：是最弱的一种引用关系，一个对象是否有虚引用的存在完全不影响对象的生存时间，也无法通过虚引用来获取一个对象的实例。一个对象使用虚引用的唯一目的是为了在被垃圾收集器回收时收到一个系统通知。在 Java 中使用 PhantomReference 类来实现。

#### 生存还是死亡，这是一个问题

在可达性分析中判定为不可达的对象，也不一定就是「非死不可的」。这时它们处于「缓刑」阶段，真正要宣告一个对象死亡，至少需要经历两次标记过程：

第一次标记：如果对象在进行可达性分析后被判定为不可达对象，那么它将被第一次标记并且进行一次筛选。筛选的条件是此对象是否有必要执行 finalize() 方法。对象没有覆盖 finalize() 方法或者该对象的 finalize() 方法曾经被虚拟机调用过，则判定为没必要执行。

第二次标记：如果被判定为有必要执行 finalize() 方法，那么这个对象会被放置到一个 F-Queue 队列中，并在稍后由虚拟机自动创建的、低优先级的 Finalizer 线程去执行该对象的 finalize() 方法。但是虚拟机并不承诺会等待该方法结束，这样做是因为，如果一个对象的 finalize() 方法比较耗时或者发生了死循环，就可能导致 F-Queue 队列中的其他对象永远处于等待状态，甚至导致整个内存回收系统崩溃。finalize() 方法是对象逃脱死亡命运的最后一次机会，如果对象要在 finalize() 中挽救自己，只要重新与 GC Roots 引用链关联上就可以了。这样在第二次标记时它将被移除「即将回收」的集合，如果对象在这个时候还没有逃脱，那么它基本上就真的被回收了。

#### 方法区回收

前面介绍过，方法区在 HotSpot 虚拟机中被划分为永久代。在 Java 虚拟机规范中没有要求方法区实现垃圾收集，而且方法区垃圾收集的性价比也很低。

方法区（永久代）的垃圾收集主要回收两部分内容：废弃常量和无用的类。

废弃常量的回收和 Java 堆中对象的回收非常类似，这里就不做过多的解释了。

类的回收条件就比较苛刻了。要判定一个类是否可以被回收，要满足以下三个条件：

1. 该类的所有实例已经被回收；
2. 加载该类的 ClassLoader 已经被回收；
3. 该类的 Class 对象没有被引用，无法再任何地方通过反射访问该类的方法。

### 3.2 垃圾回收算法

#### 标记-清除算法

正如标记-清除的算法名一样，该算法分为「标记」和「清除」两个阶段：

首先标记出所有需要回收的对象，在标记完成后回收所有被标记的对象。标记-清除算法是一种最基础的算法，后续其它算法都是在它的基础上基于不足之处改进而来的。它的不足体现在两方面：一是效率问题，标记和清除的效率都不高；二是空间问题，标记清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致以后程序的运行过程中又要分配较大对象是，无法找打足够的连续内存而不得不提前出发下一次 GC。

<div align="center"><img src="https://resources.baronzhang.com/blog/jvm/1/标记-清除算法.png" width="60%"/></div>

#### 复制算法

为了解决效率问题，于是就有了复制算法，它将内存一分为二划分为大小相等的两块内存区域。每次只使用其中的一块。当这一块用完时，就将还存活的对象复制到另一块上面，然后再把已使用过的内存空间一次清理掉。这样做的好处是不用考虑内存碎片问题了，简单高效。只不过这种算法代价也很高，内存因此缩小了一半。

<div align="center"><img src="https://resources.baronzhang.com/blog/jvm/1/复制算法.png" width="60%"/></div>

现在的商业虚拟机都采用这种算法来回收新生代，在 IBM 的研究中新生代中的对象 98% 都是「朝生夕死」，所以并不需要按照 1:1 的比例来划分空间，而是将内存分为一块较大的 Eden 空间和两块较小的 Survivor 空间，每次使用 Eden 和其中一块 Survivor。当回收时，将 Eden 和 Survivor 中还存活的对象一次性复制到另一块 Survivor 空间上，最后清理掉 Eden 和刚才用过的 Survivor 空间。 HotSpot 默认 Eden 和 Survivor 的大小比例是 8:1，也就是每次新生代中可用的内存为整个新生代容量的 90%（80%+10%），只有 10% 会被浪费。当然，98% 的对象可回收只是一般场景下的数据，我们没办法保证每次回收后都只有不多于 10% 的对象存活，当 Survivor 空间不够用时，需要依赖其它内存（这里指老年代）进行分配担保。如果另外一块 Survivor 空间没有足够空间存放上一次新生代收集下来存活的对象时，这些对象将直接通过分配担保机制进入老年代。

#### 标记-整理算法

通过前面对复制-收集算法的介绍我们知道，其对老年代这种对象存活时间长的内存区域就不适用了，而标记整理的算法就比较适用这一场景。

标记-整理算法的标记过程与「标记-清除」算法一样，但是后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。

<div align="center"><img src="https://resources.baronzhang.com/blog/jvm/1/标记-整理算法.png" width="60%"/></div>

#### 分代回收算法

当前商业虚拟机的垃圾搜集都采用「分代回收」算法，这种算法并没有什么新的思想，只是根据对象存活周期的不同将内存划分为几块。一般是将 Java 堆分为新生代和老年代，这样可以根据各个年代的特点采用最合适的搜集算法。

在新生代中，每次垃圾收集时都发现有大批对象死去，只有少量存活，那就选用复制算法，只需要付出少量存活对象的复制成本就可以完成收集。

而老年代中因为对象存活率高、没有额外空间对它进行分配担保，就必须使用「标记-清除」或者「标记-整理」算法来进行回收。

### 3.3 内存分配与回收策略

所谓自动内存管理，最终要解决的也就是内存分配和内存回收两个问题。前面我们介绍了内存回收，这里我们再来聊聊内存分配。

对象的内存分配通常是在 Java 堆上分配（随着虚拟机优化技术的诞生，某些场景下也会在栈上分配，后面会详细介绍），对象主要分配在新生代的 Eden 区，如果启动了本地线程缓冲，将按照线程优先在 TLAB 上分配。少数情况下也会直接在老年代上分配。总的来说分配规则不是百分百固定的，其细节取决于哪一种垃圾收集器组合以及虚拟机相关参数有关，但是虚拟机对于内存的分配还是会遵循以下几种「普世」规则：

#### 对象优先在 Eden 区分配

多数情况，对象都在新生代 Eden 区分配。当 Eden 区分配没有足够的空间进行分配时，虚拟机将会发起一次 Minor GC。如果本次 GC 后还是没有足够的空间，则将启用分配担保机制在老年代中分配内存。

这里我们提到 Minor GC，如果你仔细观察过 GC 日常，通常我们还能从日志中发现 Major GC/Full GC。

* **Minor GC** 是指发生在新生代的 GC，因为 Java 对象大多都是朝生夕死，所有 Minor GC 非常频繁，一般回收速度也非常快；

* **Major GC/Full GC** 是指发生在老年代的 GC，出现了 Major GC 通常会伴随至少一次 Minor GC。Major GC 的速度通常会比 Minor GC 慢 10 倍以上。

#### 大对象直接进入老年代

所谓大对象是指需要大量连续内存空间的对象，频繁出现大对象是致命的，会导致在内存还有不少空间的情况下提前触发 GC 以获取足够的连续空间来安置新对象。

前面我们介绍过新生代使用的是标记-清除算法来处理垃圾回收的，如果大对象直接在新生代分配就会导致 Eden 区和两个 Survivor 区之间发生大量的内存复制。因此对于大对象都会直接在老年代进行分配。

#### 长期存活对象将进入老年代

虚拟机采用分代收集的思想来管理内存，那么内存回收时就必须判断哪些对象应该放在新生代，哪些对象应该放在老年代。因此虚拟机给每个对象定义了一个对象年龄的计数器，如果对象在 Eden 区出生，并且能够被 Survivor 容纳，将被移动到 Survivor 空间中，这时设置对象年龄为 1。对象在 Survivor 区中每「熬过」一次 Minor GC 年龄就加 1，当年龄达到一定程度（默认 15） 就会被晋升到老年代。

#### 动态对象年龄判断

为了更好的适应不同程序的内存情况，虚拟机并不是永远要求对象的年龄必需达到某个固定的值（比如前面说的 15）才会被晋升到老年代，而是会去动态的判断对象年龄。如果在 Survivor 区中相同年龄所有对象大小的总和大于 Survivor 空间的一半，年龄大于等于该年龄的对象就可以直接进入老年代。

#### 空间分配担保

在新生代触发 Minor GC 后，如果 Survivor 中任然有大量的对象存活就需要老年队来进行分配担保，让 Survivor 区中无法容纳的对象直接进入到老年代。


## 写在最后

对于我们 Java 程序员来说，虚拟机的自动内存管理机制为我们在编码过程中带来了极大的便利，不用像 C/C++ 等语言的开发者一样小心翼翼的去管理每一个对象的生命周期。但同时我们也丧失了内存控制的管理权限，一旦发生内存泄漏如果不了解虚拟机的内存管理原理，就很难排查问题。希望这篇文章能对大家理解 Java 虚拟机的内存管理机制有所帮助。如果想对 Java 虚拟机有更进一步的了解，推荐大家去读周志明老师的《深入理解 Java 虚拟机：JVM 高级特性与最佳实践》这本书。

好了，关于 Java 虚拟机的自动内存管理机制就介绍到这里，下一篇我们来聊聊「类文件结构」。

**参考资料：**

* 《深入理解 Java 虚拟机：JVM 高级特性与最佳实践（第 2 版）》

---

> 如果你喜欢我的文章，就关注下我的公众号 **BaronTalk** 、 [**知乎专栏**](https://zhuanlan.zhihu.com/baron) 或者在 [**GitHub**](https://github.com/BaronZ88) 上添个 Star 吧！
>   
> * 微信公众号：**BaronTalk**
> * 知乎专栏：[https://zhuanlan.zhihu.com/baron](https://zhuanlan.zhihu.com/baron)  
> * GitHub：[https://github.com/BaronZ88](https://github.com/BaronZ88)

<div align="center"><img src="https://resources.baronzhang.com/blog/common/gzh3.png" width="85%"/></div>