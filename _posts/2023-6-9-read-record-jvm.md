---
title: JVM（深入理解JVM 虚拟机 读书笔记）
date: 2023-6-9 17:19:29 +0800 # 2022-01-01 13:14:15 +0800 只写日期也行；不写秒也行；这样也行 2022-03-09T00:55:42+08:00
categories: [JVM]
tags: [java,gc,jvm]     # TAG names should always be lowercase

# 以下默认false
math: true
mermaid: true
pin: true
---

# 3. 垃圾收集器与内存分配策略

## 3.2 对象存活性判断
### 3.2.1 引用计数法

如果某个对象被引用了，对象的引用计数器就 +1，取消引用就 -1，*存在无法解决循环引用问题，必须依靠额外的工作来保证GC*，**python、COM等使用是引用计数 + 其他辅助方法**

### 3.2.2 可达性分析算法

*GC Roots* ：可达性引用链分析

+ 虚拟机栈内，局部变量的引用
+ 方法区内，静态变量、常量的引用
+ 本地方法栈 对象的引用
+ *虚拟机内部的引用、系统类加载器*
+ *同步锁持有的对象 ，标记为 GC Roots*

### 3.2.4 finialize （对象被回收前，可能会被调用、但是只会被调用一次）

`finalize` ，在对象回收时，会判断是否需要调用 finalize 方法，*在finalize方法内*，对象依然可以重新链接 回 *GC Roots*

### 3.2.5 回收方法区

**方法区 的回收，并没有在 Java 虚拟机规范中严格规定，但是 一般都是会有的**

+ *新生代的一次垃圾回收，效率很高，可以回收 70%-90%* 的 空间
+ *方法区数据，回收成本低，主要是 类对象、常量的回收 *

**方法区中类型的回收**

+ 该类 的所有实例 都已经被回收，*没有对象，存在指向 该类的指针了，不会调用其中的方法了*
+ 加载该类的 *类加载器* 被回收 ，不能再生成 类对象了
+ *java.lang.Class* 对象，没有被引用，无法通过反射构造来实例化对象了。

## 3.3 垃圾收集算法
### 3.3.1 垃圾分代收集理论

分代收集假说

+ 弱分代假说：绝大多数对象的留存时间其实是非常短的
	+ 如果 一个 区域中的大多数对象，都是 *朝生夕灭*，那么针对这个区域的垃圾回收，*就应该更多的去关注存活的对象（每次仅有一小部分）*，能够使用较低的代价，回收大量的空间。
+ 强分代假说：熬过越多次 GC 过程的对象，*越不容易消亡*
	+ 如果一个区域中的对象，大多都是 *难以消亡的对象*，那么这个区域的垃圾回收 频率就可以降低了。

**部分收集 Partial GC**：只收集 Java 堆的一部分，比如说，新生代、老年代等
**全区收集：Full GC**：收集整个 Java 堆 和 方法区

### 3.3.2 标记清除算法（Mark-Sweep 算法）

标记： 对象是否存活
清除：清除掉非存活的对象
**缺点**
+ 性能不稳定，Java 堆中，如果包含大量的对象，大部分需要回收，那么耗时就比较长
+ 存在 内存碎片化的问题，*标记清除后会产生大量不连续的对象，导致在分配大对象的时候，没有足够的连续空间*

### 3.3.3 标记复制算法

### 3.3.4 标记整理算法

为什么标记整理算法是一个吞吐量优先的算法 ：虽然标记清除算法，每次GC 花费的时间非常短，但是分配内存 带来的额外时间消耗，*也是要计算到吞吐量内的*，标记清除算法，会导致每次分配空间都需要 *额外的辅助操作，比如说空闲链表等*。这个累加时间其实是超过标记整理算法的时间的，*但是标记复制算法，每次的GC 的延迟低，用户线程等待时间更短。*

## 3.4HotSpot 算法实现 细节

### 3.4.1 根节点枚举

目前大多数的商业垃圾回收器，采用的方法都是 *GC Roots* + 可达性分析 算法。首先必须要找到哪些是 *GC Roots*

+ **迄今为止，大多数的 垃圾回收器，在进行根节点枚举的时候，都是需要STW，暂停用户线程的**
+ *JVM 会记录所有 保存着引用的位置 OopMap*
	+ 记录存有 *引用* 的地址，并且记录当前引用的有效范围
+ 在 *OopMap* 协助下，HotSpot 虚拟机可以快速的完成对 GC Roots 的枚举工作

![QQ图片20230403091533 (小).jpg](../../assets/img/post/2023-6/QQ20230403091533-.jpg)

### 3.4.2 安全点（各个线程必须都处于安全点的时候，才会进行 垃圾回收）

OopMap，记录的是引用内容，但是引用变化的 频率非常高（GC 后，引用对象的地址就发生了变化），如果要为每一个 引用记录位置，成本非常大。

+ `SafePoint`：只有在特定的位置，才保留了所有的记录信息（所有的引用信息）。所以用户线程必须要在安全点，才能正常暂停。
+ *是否具有让程序长时间执行的特征*：？
+ 如何让所有的用户线程，都到达最新的安全点，然后停顿。
	+ **抢先式中断**：不需要用户线程配合，直接中断所有的用户线程，检查哪些没有达到安全点，就继续执行。
	  **主动式中断**：当需要进行垃圾回收中断线程时，不直接中断，*而是设置标志位*，由用户线程控制，当标志位为 true 时，就在最新的安全点暂停。

### 3.4.3 安全区域（在安全区域内，引用关系不发生变化）

主要针对 无法响应，虚拟机中断的一些线程，比如说陷入了 `sleep` 中。就需要 一个扩展的 安全点（安全区域），在这个区域中，任何位置进行垃圾回收都是安全的。

### 3.4.4 RememberSet（跨代收集） 和 卡表

针对 Partial 收集器，只收集一部分的区域（新时代、老年代），为了防止对其他区域的全扫描，而诞生的 *Remember Set*，标记另外区域哪些是存在跨代引用的。

+ **Remember Set**：记录 *非收集区域*，指向收集区域内的 指针集合的 抽象数据结构。
+ **记录精度**：Remember Set 中 可以保存，所有的对象指针，但是成本就非常高，一般会选择粒度更小的办法
	+ **字长精度（Word）**：一个指针的长度，记录了所有的 跨代引用指针
	+ **对象精度**：记录了哪些对象含有跨代引用
	+ **卡精度**：将非收集区域进行分区，记录了哪些区域含有 *跨代引用*，通常可以通过 **Card Table（卡表）** 的方式来实现
+ **卡表**：卡表中的每一个元素对应着，其标识的内存区域中，一个特定大小的内存块，被称为 卡页（Card Page）
+ `CARD_TABLE[Address >> 9] = 0`，可以发现，每一个 卡页的大小为 2^8 也就是 512 字节。
+ 只要卡页中存在着跨代引用的对象，都标记卡页为 脏的。

### 3.4.5 写屏障（初始化和维护卡表）

卡表 *何时* 变脏的：当有其他 分代区域，*引用了本区域的对象*，对应的其他分代区域的卡表就应该设置为 *dirty*，应该是在对象赋值的时候，就需要变脏卡表。

**写屏障（Writer Barrier）**：可以看成是 对 赋值语句执行一个 AOP 切面，在引用对象赋值的时候，就会产生一个 *Around* 类型的 通知，**提供外部程序执行额外的操作**。赋值语句的前后，都在 *写屏障* 的覆盖范围内。

+ 赋值前的屏障叫做：（Pre-Write Barrier），写前屏障
+ 赋值后的屏障 叫做 （Post-Barrier） 写后屏障

![1680486965726 (小).jpg](../../assets/img/post/2023-6/1680486965726-.jpg)

+ *伪共享*：因为 卡页只占一个字节，一个 Cache Line 中（64KB），可以包含 64 个卡页，也就导致了，修改其中一个卡页，因为缓存一致性协议，会导致其他 CPU 的 相同缓存行失效（锁定了 64 * 512 = 32KB）大小的区域

### 3.4.6 并发的可达性分析

可达性分析算法理论：要求全过程都基于一个 *能保证一致性的快照* 中 才能够进行分析

+ 三色标记法：将遍历对象图中遇到的对象，按照 *是否访问过* 分成三个颜色
	+ **白色**：对象尚未被垃圾收集器访问过，*可达性分析的开始阶段，所有的对象都是白色*
	+ **黑色**：对象已经被 垃圾收集器访问过了，并且这个对象的所有引用也被访问过了。*黑色表示访问过，所以应该是安全存活的*（黑色节点不可能直接连接一个白色节点）
	+ **灰色**：对象已经被 垃圾收集器访问过，但是 *这个对象至少还有一个引用*没有被扫描过。
	+ *白 ---- 灰 ---- 黑*
+ 如果 垃圾回收线程 和 用户 线程时并发执行的，就会存在问题，当用户线程修改了 引用关系时，可能发生两种情况
	+ **本来应该被 GC 的对象，本次中没有GC**，当 黑色扫描过之后，断开了一个黑色与黑色的连接，导致一个引用不可达。
	+ **本来不应该 GC 的对象，本次中被GC了**（*以下两个条件必须同时满足*）
		+ **赋值语句，插入了一条或者，多条从黑色对象到白色对象的连接**：因为从原则上来说，不允许从黑色直接链接到白色，黑色 必须先 链接到灰色，然后再往前推进。*由白色直接链接上黑色*，导致了 白色节点无法被访问到。
		+ **赋值器删除了，链接到白色节点的所有的灰色对象的直接或者间接引用**
+ 解决方案：破坏以上两个条件中的任意一个
	+ `增量更新`：破坏第一个条件，插入黑色到白色的连接时，*将新插入的连接记录下来*，扫描结束后，再重新遍历一下记录的这些新的连接，*黑色对象一旦插入的新的节点，就需要转换为灰色，再次遍历*
		+ **记录的是新增加的 引用连接**，在并发过程结束后，需要处理这些新连接，防止出现错误的连接
		+ *新的连接，从黑色到白色，那么再次扫描的时候，白色结点就需要变为灰色了*
	+ `原始快照（SATB）`：如果要删除 灰色到白色的引用，需要*记录下删除的链接*，随后再进行处理（重新遍历一次）
		+ **记录的是原始的引用关系，需要处理的是 删除了 灰色到白色的连接的情况，在并发标记结束后，需要重新遍历这些快照，方式错标（主要是将需要保留的对象，标记为了删除）**
		+ *无论是否删除这个引用关系，并发扫描结束后，还是会遍历删除的引用，也就是遍历 快照，而不是实时的连接结果*

## 3.5 经典垃圾收集器

### 3.5.1 Serial 收集器

Serial 直译就是单一的，也就是说使用的单线程的收集方式 **并且，需要挂起用户线程**

+ Serial 收集器，针对新生代使用 *标记-复制* 算法，针对 老年代 使用 *标记-整理*算法
+ **缺点在于，需要暂停用户线程，这种割裂感，在耗时较长时无法接受**
+ **但是针对一些，单核处理器、线程数较少的程序，Serial 也有优点，简单消耗内存少，并且没有频繁的线程上下文切换的问题**
+ *Serial 收集器，对于运行在客户端模式下的虚拟机，是一个很好的选择*

### 3.5.2 ParNew 收集器

**Serial** 收集器存在一些问题，为了解决这些问题，*各种虚拟机开发商 在后续的开发过程中提出了不同的方式*
**ParNew**：收集器，最大的特点就是不再使用单一线程来收集，而是使用多线程来收集，*但是依然是存在STW* 的情况。（**使用多个GC线程**）

+ *是大多数服务端模式下 HotSpot 虚拟机首选的 新生代收集器*
+ 只有 ParNew 收集器（新生代），可以和 CMS 收集器（老年代） 配合工作，*ParNew ，Serial 和 CMS，使用了同一套垃圾收集器的分代框架*，而Parallel Scavenge 和 G1 收集器都没有复用这部分框架。
+ **默认开启，处理器核心数相同的线程数**，

### 3.5.3 Parallel Scavenge 收集器 （吞吐量优先的收集器）

+ **吞吐量** ：高吞吐量，可以最高效率地利用处理器资源，尽快完成程序的运算任务，适合在后端运算，而不需要太多交互的任务。
+ **停顿时间**：停顿时间越短，越适合需要与用户交互、*保证服务响应时间* 的程序，良好的响应速度，能够提升用户体验。

`-XX:MaxGCPauseMillis`：控制最大停顿时间，虚拟机 尽力将停顿时间控制在这个参数内，**问题在于，停顿时间的缩小是以吞吐量和新时代空间为代价的，为了尽可能的虽小停顿时间，系统会调小新时代的内存大小，停顿时间会变短，但是次数会增加，吞吐量会降低**
`-XX:GCTimeRatio`：直接设置吞吐量大小，是一个 0~100 的 整数，表示 垃圾收集时间的占比。

### 3.5.4 Serial Old（Serial 的老年代版本）

主要供 **客户端** 下的 `HotSpot` 虚拟机使用，*或者是，作为 CMS 收集器的一个后备预案，简单来说，在发生并发收集失败的情况下，就需要启用 Serial old 收集器*

### 3.5.5 Parallel Old 收集器（也是 Parallel 的老年代版本）

**各个垃圾收集器的发展和繁荣，是和垃圾收集器的适配性非常相关的**

+ Serial 可以 和 Serial old 搭配
+ Parallel 可以 和 CMS 搭配
+ Parallel Scavenge 只能和 *Serial Old* 搭配（不能和 CMS搭配）
+ **所以 Parallel Old 收集器的出现，使得 Parallel Scavenge 可以 搭配一个 并发收集的老年代收集器**
	+ *在 处理器资源紧张、注重吞吐量的场合*：可以使用 Parallel Scavenge 和 Parallel Old 收集器搭配。

### 3.5.6 CMS 收集器 （Concurrent Mark Sweep）并发标记清除收集器

**获取最短停顿时间为目标的收集器**，和 Java 应用场景密切相关，Java 大多数用在 互联网 和 B/S 服务端上，*会较多的关注服务的响应时间*，希望系统停顿尽可能的端，给用户带来更好的交互体验。

1. 初始标记（*需要STW*） ：
	+ 枚举 GC Roots，和 其能直接链接到的对象，数量较少，停顿时间很短
2. 并发标记 ：
	+ **耗时最长**：从 GC Roots 直接关联到的对象开始，*遍历整个对象图*，耗时最长的部分，但是不需要停顿用户线程，可以和垃圾收集器并发执行
3. 重新标记 ：
	+ *修正并发标记期间，因用户线程而发送标记变动的*，简单来说，就是增量更新（写屏障那一部分，会通过 postWrite 方法，将删除掉的引用加入到待遍历列表中）
4. 并发清除 ：
	+ 实际上的清除过程，*和用户线程并发执行*，清理删除掉标记阶段，被标记为死亡的 *对象*（**不需要移动存活对象**）

![image.png](../../assets/img/post/2023-6/20230404150944.png)

**缺点**：

+ CMS收集器，对处理器资源非常敏感，（*面向并发设计的程序对处理器都很敏感*）：**并发阶段，虽然不会暂停用户线程，但是会因为占用了一部分线程数（处理资源），导致应用程序变慢**，一般的，CMS 垃圾收集线程 = （核心数 + 3）/4，如何核心线程数大于 4，会启用 *一个垃圾处理线程*，成本约为 25%，但是如果核心线程数小于4，比如说只有两个核心，计算得到的，依然是需要一个线程来处理，就需要花费 50% 的计算资源。*可能会导致用户程序的执行速度不稳定，忽慢忽快*。
+ **CMS 收集器无法处理浮动垃圾**：并发清除阶段，用户程序依然在运行，*也会生成垃圾*，这个步骤中产生的垃圾，本次垃圾回收无法处理，只有留到下一次处理（*称为浮动垃圾*）
+ **并发清理阶段，用户程序依然还在运行，就不能等到 老年代空间满了才进行收集，必须要要给，用户线程留一部分空间（线程运行是需要内存的）**，所以 CMS 一般会有一个收集阈值 （*`-XX:CMSInitiatingOccur-pancyFraction`*)，超过这个阈值，CMS收集器必须要执行一次垃圾收集。*这个阈值是不好设置的，因为太小了，就会导致频繁的老年代垃圾收集，太大了，会出现一个 Concurrent Mode Failue，因为并发清除时，没办法给用户线程提供更多内存了，导致触发一个 Full GC*，会暂停掉用户线程，并且临时启用 Serial 来进行垃圾处理。
+ **CMS 收集器，基于标记-清除算法**：本身是没有 整理内存这个过程的，*原因也很简单，需要并发清除，就没办法进行 复制和整理（无法并发的移动存活对象）*。会出现内存碎片，并且*提前触发 Full GC* 的情况。

### 3.5.7 Garbage First 收集器 （G1 收集器，分区收集器）

**面向局部收集** 和 **基于 Region 收集**，主要面向 *大内存，服务端的垃圾收集器*，号称在未来可能取代 CMS 垃圾收集器。

#### 停顿时间可控的 G1 垃圾收集器

**停顿时间模型**：支持在一个长度为 M 毫秒的时间片段内，消耗在 垃圾收集线程上的时间不超过 N 毫秒。也就是可以控制垃圾收集的 **吞吐量**，G1 收集器，从使用之初，就可以 替代 *Parallel Scaveng* 收集器。

#### 分区收集思想 （Region）

+ 如何实现，停顿时间可控。G1 收集器不再针对 **整个堆** 做垃圾收集，而是针对 分区做垃圾收集，*如果分区大小可控*，分区数量可控，那么总的停顿时间，大致也是可控的。（**每次收集的基本单位为 Region，根据 region 大小 和 数量，就可以预估一个 停顿时间**）
+ **G1 收集器，并不收集 整个堆**，而是收集 回收价值最大的 Region（*回收耗时 + 回收的空间*），哪一块垃圾最多，回收收益最大，就回收哪一块。 `效益最大化`

**分区**：将整个 Java 堆，分为大小固定的（1MB ~ 32MB）大小的 Region

+ 每一个 Region 都可以扮演 Eden、Survivor、Old （不限制 Region 的分代，但是需要有标记，可以判断为哪一代），以便于针对不同的 *分代*，使用不同的 垃圾回收算法
+ *Humongous* 分区，G1 垃圾回收算法，增加了一个 Humongous 分
+ 区，**专门用来保存大对象（大小超过分区大小的一半**）这个区域的垃圾回收，和老年代垃圾回收一致。
+ *每一个分区有一个经验性的回收价值，为回收成本和回收收益（耗时和回收所得空间），G1收集器会跟踪每一个 Region 的回收价值，并且维护一个 优先级列表*
+ **每次回收，会在MaxGCPauseMillis 内，回收 价值最高的几个 Region，达到最大效益。**

#### G1 的 存在问题

##### 卡表维护问题（主要是针对跨区收集）

在传统的 分代收集器中，只需要维护 两个卡表即可，但是 在 G1 中，需要维护的卡表数量为 Region 的数量，这部分内存空间大概会占到 堆内存的 10% ~ 20%（只要当前分区内，有对其他分区的引用，当前分区内的对应卡表需要被设置为 dirty），**G1 的卡表类似于一个hash 表，维护了一个双向的结构。**

+ *传统卡表记录的是我指向谁*，当前分区对其他分区的引用，标记的是其他分区。
+ G1 卡表维护了一个 哈希表 （key 为别的region 的起始地址，value 是一个集合，存储的元素是 卡表的索引号）
+ *其他 Region 指向，本区域 Region 的指针，并且 value 记录了到底指向的是哪一块区域*

##### 并发标记问题

G1 和 CMS 都存在并发标记问题，也就是初始标记完成后，再次发送的标记变化，*依然是通过写屏障来解决的*，
CMS：采用的是写后屏障，对应的就是 增量更新算法
G1：使用的是 写前屏障来完成的，对应的算法就是 SATB，**原始快照**的方式

**TAMS 指针 ：** 指向的是当前 Region 的空白区域，*也就是并发标记过程中，新对象的分配在这里开始*，TAMS 指针后的对象，默认是 存活的也就是（在标记过程中，产生的对象）。*存在TAMS 内存不足的问题，当并发标记过程中，没办法分配空间了，回收的速度跟不上创建新对象的速度，就会触发 STW 停止用户线程，进行Full GC*

**可靠的停顿预测模式（通过收集各种信息，预测一个这个区域回收大概需要多久**

##### 标记过程

![image.png](../../assets/img/post/2023-6/20230404161253.png)

+ **初始标记阶段**：同样是需要 STW的，也是枚举所有 GC Roots 及其能够直接关联到的对象 ，*修改 TAMS 指针*，让下一阶段用户线程 并发运行。（*可以正确在可用的 Region 中分配对象*）
+ **并发标记**：和 CMS 类似，根据上一阶段的 遍历，得到GC Roots 直接关联对象，以此为起点，遍历整个对象图。
+ **最终标记**：**需要暂停用户线程，处理 SATB 中，记录的有变动的对象**
+ **筛选回收**：*G1 的核心阶段，需要暂停用户线程，更新Region 的统计信息，根据停顿时间，选择多个价值最高的 Region 构成回收集，进行垃圾回收*，需要暂停用户线程，需要对数据进行复制，**但是这种暂停是可控的，不会超过最大停顿时间**，并且这种暂停 用户线程可以回收更多的内存空间，减少碎片话内存，*可以保证吞吐量*。（`可控的停顿时间`）

停顿时间也不能设置得无限小，默认参数为 *200ms*，如果设置得太小，每次回收的空间就很少，导致垃圾回收的效率，跟不上分配内存的速率，从而 *提早的触发 full gc*。

**G1 收集器开始，垃圾收集器，不再关注与整堆收集，而是有一个目标，只要垃圾回收的速度，可以赶上对象分配内存的速度就可以了，没必要整堆 回收，浪费时间。**

#### G1 和 CMS

**G1 回收器**

+ 首先，G1 收集器使用的是，*标记-复制算法*，整个处理过程中，不会产生内存碎片。可以说从另外一个方面提高了吞吐量。
+ G1 收集器可配置，用户期望（可以接受的）停顿时间，可以在吞吐量和 延迟上做到一个平衡。
+ *缺点在于 G1 收集器，需要占用大量的 堆空间，额外的卡表，以及其他内存消耗，在10% ~ 20 %之间*
**CMS回收器**：

## 3.6 低延迟垃圾收集器

衡量垃圾收集器的三大指标：**内存占用**、**吞吐量**、**延迟**

+ 内存占用 和 吞吐量占用都是随着 计算机硬件的发展而不断提高的，*从内存占用来说，现在的内存空间远大于 以前的，那么高一点的内存占用基本上是可以接受的*，从吞吐量来说，随着计算机硬件速度的提高，处理性能越高，整个吞吐量，也会提高。
+ **延迟则相反，内存越大，就需要更多的时间来进行垃圾回收，所以目前的垃圾回收器越来越关注垃圾回收 的延迟**

### 3.6.1 Shenadoah 收集器

在 G1 收集器上的一个递进

#### 相同之处

+ 都是采用 基于 Region 的队内布局，不再是分代、而是分区了
+ 都有保存着大对象的 Humongous 区域
+ **默认的回收策略的，都是需要回收价值最大的几个 Region**

#### 不同之处

+ **Shenadoah** 默认不再使用分代搜集算法，整个收集过程，不会涉及到分代。
+ **Shenadoah**：最大的特点在于，采用并发整理，而不是阻塞的整理，可以很大程度上降低延迟
+ *Shenadoah 摒弃了，在G1 收集器中，占用大量堆空间的 Remember Set*（在G1 中，每一个 Region 都有一个独立的 Remember Set），导致G1 会占用 10% ~ 20 % 的堆空间，在 *Shenadoah中，使用了一个全局的 连接矩阵，（i，j）表示 i 对 j 区域对象有引用关系 *

#### 基本处理流程 （九大过程、重点在引用更新部分）

##### 并发标记阶段
1. **初始标记阶段**：和之前的收集器类似，首先标记 GC Roots 及其可以直接关联到的对象 ，**需要 STW，但是耗时比较短**，时间可控，主要和 方法区大小、虚拟机栈大小相关，*和堆本身的大小没有关系*
2. **并发标记阶段** ：

	+ 类似之前，*用户线程* 和 *垃圾回收线程*，是 并发执行的，这个过程耗时较长，但是不会影响用户的正常操作（*目的技术延迟最低*）

3. **最终标记阶段**

	+ 类似之前，因为在并发标记过程中，可能存在一些引用关系 发生改变的对象，通过 **写屏障**记录下来（*增量更新 或者 是 原始快照 （SATB）*
	+ **类似于 G1，会筛选出，回收价值最大的 Region 组成回收集（Collection Set）**
	+ **这个阶段，也是需要 STW 的**

##### 并发回收阶段
4. **并发清理**：这个阶段，主要处理的是 *Region 内没有一个存活对象的情况，所有的 对象都需要被回收了*，这个过程，可以并发，因为没有修改任何的对象。（*Immediate Garbage Region*）
5. **并发回收**：处理 Region 内既存在 存活对象，又存在非存活对象的情况。**主要目的，就是将这些Region，处理为 （Immediate Garbage Region）**，*将 回收集 中 Region 中的， 存活对象*，复制到新的 *Region*，**需要注意，此时内存中，存在两个相同的对象，一个是原始对象，一个是新复制的对象**，对象复制到新的Region后，相对的，之前的引用就需要 发生变化，再次发送的读写行为，需要主动去访问 *新的对象*。这个过程 *Shenandoah* 会使用 读屏障、和转发指针来实现。

	+ 总结一些，并发回收过程结束后，所有的 Collection Set 中，都是无效对象了，需要存活的对象已经被拷贝到新的Region中，老的 Region 对象的对象头上，还有一个转发指针指向新的对象。（*接下来，需要，更新引用，和 回收空间*）

##### 并发引用更新阶段（旧对象更新到新对象）
6. **初始引用更新**：这个阶段只是一个标志性的阶段，本质上没有进行任何的处理，是一个 **安全点**，保证所有的并发回收过程已经结束了，现在需要开启对象的引用更新了。（*会有短暂的停顿*）
7. **并发引用更新**：正式开始修改对象的引用，*和用户线程并发执行*，线性的扫描内存物理地址，将旧对象的引用替换为新对象。**时间的长短取决于，内存中对象的引用数量**

	+ *实际的执行过程中，会通过连接矩阵，来优化整个扫描过程，保证扫描的快速性*

8. **最终引用更新：** 更新的是，*非堆区* 的引用，比如说方法区的引用，栈空间内的引用，本质上也就是 GC Roots
9. **并发清理**：所有引用更新完毕之后，整个 `Collection Set` 中的所有 Region 都是 *Immediate Garbage Region* 了，可以对这些区域进行清理了。
#### Brook Pointer（转发指针，位于对象的头部 的一个新的指针）

Forwarding Pointer，转发指针，实现对象移动和 用户程序并发。

+ *基于操作系统，其实也可以实现，主要是通过 中断的机制，COW 就是 类似*
+ **基于中断机制的处理，需要 内核态 和 用户态的切换，**，并且需要操作系统 层面的支持
**Brooks Pointer**：在每一个对象的，对象头前面加入一个指针，*正常情况下，这个转发指针，指向是自身，而在垃圾回收过程中，这个指针会指向新的对象*

+ 只要 旧的对象还存在，用户线程，就可以通过 旧对象的转发指针访问到新的对象
+ **但是需要保证，创建新对象 和 写入 Brooks Pointer 的整个过程需要是原子的，否则会出现写入旧对象的情况(Brook 使用的是 CAS 的方式）**
+ **转发指针的使用**：要向让对象访问到转发指针，就必须要加入 *读写屏障*，并且在读写屏障中，加入转发处理，简单来说，就是 JVM 翻译的过程中，见到 读写语句，就会 添加一步 *访问 转发指针*
	+ *写屏障还可以接收，但是读取对象，这个在 Java 中发生最 频繁的操作，如果加上屏障，将会大大影响性能，在 JDK13 中，计划将内存屏障 修改为基于 引用对象的内存屏障，如果是基本数据类型，就不需要访问 转发指针*
![image.png](../../assets/img/post/2023-6/20230405162811.png)

### 3.6.2 ZGC （Z garbage collector）

ZGC 的目标，也是为了尽可能的降低 *延迟*

![image.png](../../assets/img/post/2023-6/20230405170443.png)

#### 着色指针（将信息保存到指针中的技术）

**ZGC**：只支持 64 位 系统，将 64 位 虚拟地址划分为了多个块，

+ **ZGC将对象存活信息存储在42~45位中，这与传统的垃圾回收并将对象存活信息放在对象头中完全不同。（将信息保存在 对象指针中）**
+ *一般的内存地址都是没有使用所有的 64 位的，Linux 一般为 48 位，也有其他的，46 位*
+ *ZGC 的着色指针，需要利用这 46 位 中的 4位，来标记一个 当前指针的状态。*
![image.png](../../assets/img/post/2023-6/20230405200450.png)

![image.png](../../assets/img/post/2023-6/20230405200722.png)

ZGC 使用的 空间换时间的方式：在对象分配初期，就同时在 虚拟内存中，申请了三个地址，*对象在堆中有一个地址，M0、M1、Remapped 三个虚拟位置都有一个地址*，用虚拟的 空间来交换时间。

> 读屏障是JVM向应用代码插入一小段代码的技术。当应用线程从堆中读取对象引用时，就会执行这段代码。需要注意的是，仅“从堆中读取对象引用”才会触发这段代码。

**JVM 在处理 引用对象的读取的时候，会使用到的技术**

+ *ZGC中读屏障的代码作用*：在对象标记和转移过程中，用于确定对象的引用地址是否满足条件，并作出相应动作。

```java
Object o = obj.FieldA // 从堆中读取引用，需要加入屏障
<Load barrier>
Object p = o // 无需加入屏障，因为不是从堆中读取引用
o.dosomething() // 无需加入屏障，因为不是从堆中读取引用
int i = obj.FieldB //无需加入屏障，因为不是对象引用
```

#### 访问过程 （这个过程是会修改对象的 指针的）

举个例子，如果在初始状态下，有一个

```java
A a = B.a
// 在初始标记后，a 对象本质是一个指针，标记过后，这个指针就变了，就被染色了 M0 或者 是 M1，这里是交替执行的。
```

![image.png](../../assets/img/post/2023-6/20230405201313.png)

+ **初始化**：整个内存地址空间 视图被设置为 *Remmapped*。程序正常运行，满足一定条件，程序进行标记阶段
+ **并发标记**：第一次进入标记阶段，*视图为 M0*，如果对象被 GC 标记线程、或者应用线程访问过，*对象的地址视图 就 从 Remmapped 调整为 M0*，所以，在并发标记阶段，结束之后，对象地址要么是 M0，
	+ **如果是 M0**，就表示这个对象是存活的。**（基于 ZGC 的多重映射机制，Remmapped、M1、M0 这个三个虚拟地址指向的是同一个物理地址，只是简单的修改 引用，不会对程序带来任何问题）**，`所有被搜索过的对象、都会被修改引用`，
	+ **如果对象是 Remmapped 视图**，说明对象是不活跃的，可以被清理掉
+ **并发转移阶段（Reallocate 重分配阶段 ）**：标记结束后就进入 *转移阶段*，此时地址视图再次被设置为Remapped。如果对象被GC转移线程或者应用线程访问过，那么就将对象的地址视图从M0调整为Remapped。
	+ 其实，在标记阶段存在两个地址视图M0和M1，上面的过程显示只用了一个地址视图。
	+ 之所以设计成两个，是为了区别 *前一次标记* 和 *当前标记*。也即，第二次进入并发标记阶段后，地址视图调整为M1，而非M0。
	+ 前一次标记结束之后，所有的垃圾已经被清理的，**但是没有将原始的引用重新指向新分配的引用，而这些重 映射的过程就会在下一次的，标记过程实现**

#### ZGC 具体流程

+ **并发标记阶段**：和之前的垃圾回收器都是类似的，*初始表示 STW、并发标记 、STW 的 最终标记*，唯一的区别在于 ZGC 的 并发标记，直接修改 **引用指针（ reference 64bit）**，修改指针的 M0、M1 比特位
	+ *并发标记，会交替使用 M0、M1 两个 bit位，如果上一次是 M0，本次就是 M1*
+ **并发预备重分配**：ZGC 同样存在一个，*选择最优的 Region* 的阶段，这个阶段中，需要统计得到 本次垃圾回收到底需要回收哪些 Region，组成 *重分配集（ Relocation Set ）*，重分配集，定义和 G1的最优 Region，存在差别，
	+ 因为本身 ZGC 就是进行 **全堆扫描的** ，重分配集合只是代表，当前的 Region 的区域会被释放掉，里面的存活对象会被复制到新的 Region，（整堆扫描的）
	+ ZGC每次回收都会扫描所有的 Region，用范围更大的扫描成本换取省去G1中记忆集的维护成本。
+ **并发重分配阶段**：核心阶段，这个阶段，需要将 *重分配集* 中的存活对象，复制到新的 Region 上，**并且将旧的地址 和新的地址的映射，保存到 Region 的 转发表中**
	+ *因为有染色指针，所以很快的，就可以确定一个 对象是否处于 重分配集合中（ 因为需要访问对象地址，根据 染色指针的标记位，就可以很好的判断）*
	+ 如果 用户线程此时，访问了一个 重分配 的对象，*就会被预先 插入的 读屏障 所截获*，根据 Region 转发表，找到新的 对象地址 *并且会更新当前的引用地址*
	+ **最为复杂的部分**
	+ **一旦 重分配 Region 的 存活对象被复制完毕，在保留 转发表的前提下，可以清空整个 Region，作为之后使用**
+ **并发重映射**：并发重映射，就是 修复引用的过程，将旧的引用修改为新的引用，*而这个过程是完全可以，保留到下一个 并发标记阶段执行，因为并发标记阶段，会对整个 堆进行遍历，所以可以自动修复重映射*
![image.png](../../assets/img/post/2023-6/20230405205344.png)

+ `Remapped`：bit 位，代表当前的数据，是经过重映射的，也就是 当前数据 *是最新的，可以访问*
	+ *Read Barrier* ：读屏障会在 访问数据时，进行判断，当前的 Remapped 比特位是不是 1，如果是代表 数据时经过重映射的，可以直接使用
	+ **如果 不是，则判断当前访问的对象，是不是在重分配集 里，如果没有在，说明我们此次GC不需要处理这个对象，手动将 Remapped 比特位置为 1**
	+ *现在 可以知道了当前的访问对象是，需要重分配的，但是问题在于，到底当前对象重分配了没有*，此时就需要从，**转发表中访问，是否有这个 转发项**，如果有证明 已经被 重分配了，如果没有就代表，还没有重分配，*这个时候，Read Barrier 需要负责对 对象进行重分配，并且保存 映射关系到转发表*，
		+ **需要注意，重分配 的 GC 线程，不负责 修改对象引用，所以说，如果一个对象被重分配了，只有在下一次访问到的时候，才会被 重新映射（一直维持 M1 状态）**
	+ **最后**：从转发表中，获取对象的新地址，并且更新当前的引用地址，**重新设置 remap 标志位**，最后返回引用对象。
	+ *在 下一次 GC 的 标记阶段，因为上一层使用的是 M1，本次就会使用 M0，作为标记，并且，针对上一次GC，还没有来得及 Remap 的 引用，重新进行 Remap*，**并且释放掉 转发表**

#### 缺点

+ **收集时间长，可能出现浮动垃圾**：因为每次扫描需要整堆扫描，就很可能出现，持续时间长，导致 *分配新对象的速度 快于 垃圾回收的速度，从而产生浮动垃圾*
