##  垃圾回收算法有哪些？

###  标记-清除

第一步：就是找出活跃的对象。我们反复强调 GC 过程是逆向的， 根据 GC Roots 遍历所有的可达对象，这个过程，就叫作标记。

第二部：除了上面标记出来的对象以外，其余的都清楚掉。

    * 缺点：标记和清除效率不高，标记和清除之后会产生大量不连续的内存碎片 

![](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409141727240.webp)

###  复制  

新生代使用，新生代分中Eden:S0:S1= 8:1:1，其中后面的1:1就是用来复制的。

当其中一块内存使用完了，就将还存活的对象复制到另外一块上面，然后把已经使用过的内存空间一次

清除掉。

一般对象分配都是进入新生代的eden区，如果Minor
GC还存活则进入S0区，S0和S1不断对象进行复制。对象存活年龄最大默认是15，大对象进来可能因为新生代不存在连续空间，所以会直接接入老年代。任何使用都有新生代的10%是空着的。

    * 缺点：对象存活率高时，复制效率会较低，浪费内存。 

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmhy6Htf7vkqWGOev3lvaIq2QtHmxRVYczpib0avZKzPTBpmLFm9s6mVQ/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

###  标记整理  

它的主要思路，就是移动所有存活的对象，且按照内存地址顺序依次排列，然后将末端内存地址以后的内存全部回收。
但是需要注意，这只是一个理想状态。对象的引用关系一般都是非常复杂的，我们这里不对具体的算法进行描述。我们只需要了解，从效率上来说，一般整理算法是要低于复制算法的。这个算法是规避了内存碎片和内存浪费。

让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。

![](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409141727210.webp)

从上面的三个算法来看，其实没有绝对最好的回收算法，只有最适合的算法。

##  垃圾收集器

垃圾收集器就是垃圾回收算法的实现，下面就来聊聊现目前有哪些垃圾收集器。

###  新生代有哪些垃圾收集器

####  serial

Serial收集器是最基本、发展历史最悠久的收集器，曾经（在JDK1.3.1之前）是虚拟机新生代收集的唯一选择。

它是一种单线程收集器，不仅仅意味着它只会使用一个CPU或者一条收集线程去完成垃圾收集工作，更重要的是其在进行垃圾收集的时候需要暂停其他线程。

优点：简单高效，拥有很高的单线程收集效率

缺点：收集过程需要暂停所有线程

算法：复制算法

应用：Client模式下的默认新生代收集器

收集过程：

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmfRUfShBDLX4qdNIeTHr15Z6RtasXkyo5xPnTBDpJwn5Ej5lyCx2OIA/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

####  ParNew

可以把这个收集器理解为Serial收集器的多线程版本。

优点：在多CPU时，比Serial效率高。

缺点：收集过程暂停所有应用程序线程，单CPU时比Serial效率差。

算法：复制算法

应用：运行在Server模式下的虚拟机中首选的新生代收集器

收集过程：

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmM2cibZkFicVhoX6xs3rosNKOJPEgloDRVWvr8qAhtdPxwv1SElIRDibuA/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

####  Parallel Scanvenge

Parallel Scavenge收集器是一个新生代收集器，它也是使用复制算法的收集器，又是并行的多线程收集

器，看上去和ParNew一样，但是Parallel Scanvenge更关注 系统的吞吐量 ;

吞吐量 = 运行用户代码的时间 / (运行用户代码的时间 + 垃圾收集时间)

比如虚拟机总共运行了120秒，垃圾收集时间用了1秒，吞吐量=(120-1)/120=99.167%。

若吞吐量越大，意味着垃圾收集的时间越短，则用户代码可以充分利用CPU资源，尽快完成程序的运算任务。

可设置参数：


        -XX:MaxGCPauseMillis控制最大的垃圾收集停顿时间，  
    -XX:GC Time Ratio直接设置吞吐量的大小。  


###  老年代有哪些垃圾收集器

####  CMS=Concurrent Mark Sweep

特点：最短回收停顿时间，

回收算法：标记-清除

回收步骤：

    1. 初始标记：标记GC Roots直接关联的对象，速度快 
    2. 并发标记：GC Roots Tracing过程，耗时长，与用户进程并发工作 
    3. 重新标记：修正并发标记期间用户进程运行而产生变化的标记，好事比初始标记长，但是远远小于并发标记 
    4. 表发清除：清除标记的对象 

缺点：对CPU资源非常敏感，CPU少于4个时，CMS岁用户程序的影响可能变得很大，有此虚拟机提供了“增量式并发收集器”；无法回收浮动垃圾；采用标记清除算法会产生内存碎片，不过可以通过参数开启内存碎片的合并整理。

收集过程：

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmTcN7sWju68HXZByDYNgjqVd15MOkU5JpqMZKXB6A2rDriaIJs9yXqYA/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

####  serial old

Serial Old收集器是Serial收集器的老年代版本，也是一个单线程收集器，不同的是采用"标记-整理算

法"，运行过程和Serial收集器一样。

适用场景：JDK1.5前与Parallel Scanvenge配合使用，作为CMS的后备预案;

收集过程：

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmfRUfShBDLX4qdNIeTHr15Z6RtasXkyo5xPnTBDpJwn5Ej5lyCx2OIA/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

####  Parallel old

Parallel Old收集器是Parallel Scavenge收集器的老年代版本，使用多线程和"标记-整理算法"进行垃圾

回收，吞吐量优先;

回收算法：标记-整理

适用场景：为了替代serial old与Parallel Scanvenge配合使用

收集过程：

![](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409141727383.webp)

####  G1 收集器

G1（Garbage first） 收集器是 jdk1.7 才正式引用的商用收集器，现在已经成为JDK9
默认的收集器。前面几款收集器收集的范围都是新生代或者老年代，G1 进行垃圾收集的范围是整个堆内存，它采用 “ 化整为零 ”
的思路，把整个堆内存划分为多个大小相等的独立区域（Region），在 G1 收集器中还保留着新生代和老年代的概念，它们分别都是一部分 Region，如下图：

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmzicLCTRUpcAnrY8Drw5U8qAJ1xhV7OyXKMGwSFFoYtFeic9Dy4mKSL3w/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

每一个方块就是一个区域，每个区域可能是 Eden、Survivor、老年代，每种区域的数量也不一定。JVM 启动时会自动设置每个区域的大小（1M ~
32M，必须是 2 的次幂），最多可以设置 2048 个区域（即支持的最大堆内存为 32M*2048 = 64G），假如设置 -Xmx8g
-Xms8g，则每个区域大小为 8g/2048=4M。

为了在 GC Roots Tracing 的时候避免扫描全堆，在每个 Region 中，都有一个 Remembered Set
来实时记录该区域内的引用类型数据与其他区域数据的引用关系（在前面的几款分代收集中，新生代、老年代中也有一个 Remembered Set
来实时记录与其他区域的引用关系），在标记时直接参考这些引用关系就可以知道这些对象是否应该被清除，而不用扫描全堆的数据。

G1 收集器可以 “ 建立可预测的停顿时间模型 ”，它维护了一个列表用于记录每个 Region
回收的价值大小（回收后获得的空间大小以及回收所需时间的经验值），这样可以保证 G1 收集器在有限的时间内可以获得最大的回收效率。

如下图所示，G1 收集器收集器收集过程有初始标记、并发标记、最终标记、筛选回收，和 CMS 收集器前几步的收集过程很相似：

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmt34t0dS5qlecN66oErc82G4gp9Rs49qTSEB9nKRawc3FZoNj3jUdtA/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

① 初始标记：标记出 GC Roots 直接关联的对象，这个阶段速度较快，需要停止用户线程，单线程执行。

② 并发标记：从 GC Root 开始对堆中的对象进行可达新分析，找出存活对象，这个阶段耗时较长，但可以和用户线程并发执行。

③ 最终标记：修正在并发标记阶段引用户程序执行而产生变动的标记记录。

④ 筛选回收：筛选回收阶段会对各个 Region 的回收价值和成本进行排序，根据用户所期望的 GC
停顿时间来指定回收计划（用最少的时间来回收包含垃圾最多的区域，这就是 Garbage First
的由来——第一时间清理垃圾最多的区块），这里为了提高回收效率，并没有采用和用户线程并发执行的方式，而是停顿用户线程。

适用场景：要求尽可能可控 GC 停顿时间；内存占用较大的应用。可以用 -XX:+UseG1GC 使用 G1 收集器，jdk9 默认使用

####  ZGC收集器

#####  ZGC有什么特点？

ZGC 是最新的 JDK1.11 版本中提供的高效垃圾回收算法，ZGC 针对大堆内存设计可以支持 TB 级别的堆，ZGC 非常高效，能够做到 10ms
以下的回收停顿时间。

这么快的响应，ZGC 是如何做到的呢？这是由于 ZGC 具有以下特点。

    * 第一个：ZGC 使用了着色指针技术，我们知道 64 位平台上，一个指针的可用位是 64 位，ZGC 限制最大支持 4TB 的堆，这样寻址只需要使用 42 位，那么剩下 22 位就可以用来保存额外的信息，着色指针技术就是利用指针的额外信息位，在指针上对对象做着色标记。 
    * 第二个：使用读屏障，ZGC 使用读屏障来解决 GC 线程和应用线程可能并发修改对象状态的问题，而不是简单粗暴的通过 STW 来进行全局的锁定。使用读屏障只会在单个对象的处理上有概率被减速。 
    * 第三个：由于读屏障的作用，进行垃圾回收的大部分时候都是不需要 STW 的，因此 ZGC 的大部分时间都是并发处理，也就是 ZGC 的第三个特点。 
    * 第四个：基于 Region，这与 G1 算法一样，不过虽然也分了 Region，但是并没有进行分代。ZGC 的 Region 不像 G1 那样是固定大小，而是动态地决定 Region 的大小，Region 可以动态创建和销毁。这样可以更好的对大对象进行分配管理。 
    * 第五个：压缩整理。CMS 算法清理对象时原地回收，会存在内存碎片问题。ZGC 和 G1 一样，也会在回收后对 Region 中的对象进行移动合并，解决了碎片问题。 

虽然 ZGC 的大部分时间是并发进行的，但是还会有短暂的停顿。来看一下 ZGC 的回收过程。

#####  ZGC 是如何进行垃圾收集的？

ZGC（Z Garbage
Collector）是一款由Oracle公司研发的，以低延迟为首要目标的一款垃圾收集器。它是基于动态Region内存布局，（暂时）不设年龄分代，使用了读屏障、染色指针和内存多重映射等技术来实现可并发的标记-
整理算法的收集器。

初始状态时，整个堆空间被划分为大小不等的许多 Region，即图中绿色的方块。

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcm7dOCuN1Nkib2S3A7pYribSaSYd5wUxSlXEE4wSC39rB4XrOZzsO16Z1g/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

开始进行回收时，ZGC 首先会进行一个短暂的 STW（Stop The world），来进行 roots 标记。这个步骤非常短，因为 roots
的总数通常比较小。

然后就开始进行并发标记，如上图所示，通过对对象指针进行着色来进行标记，结合读屏障解决单个对象的并发问题。其实，这个阶段在最后还是会有一个非常短的 STW
停顿，用来处理一些边缘情况，这个阶段绝大部分时间是并发进行的，所以没有明显标出这个停顿。

下一个是清理阶段，这个阶段会把标记为不在使用的对象进行回收，如上图所示，把橘色的不在使用的对象进行了回收。

最后一个阶段是重定位，重定位就是对 GC 后存活的对象进行移动，来释放大块的内存空间，解决碎片问题。

重定位最开始会有一个短暂的 STW，用来重定位集合中的 root 对象。暂停时间取决于 root 的数量、重定位集与对象的总活动集的比率。

最后是并发重定位，这个过程也是通过读屏障，与应用线程并发进行的。

## 如何理解垃圾回收

### 解决什么问题

    哪些内存需要回收？
    什么时候回收？
    如何回收？

### 什么是垃圾

- 垃圾是指在**运行程序中没有任何引用指向的对象**，这个对象就是需要被回收的垃圾。
- 如果不及时对内存中的垃圾进行清理，那么，这些垃圾对象所占的内存空间会一直保留到应用程序结束，被保留的空间无法被其他对象使用。甚至可能**导致内存****溢出**。

### 针对哪些区域回收

 垃圾收集器可以对年轻代回收，也可以对老年代回收，甚至是全栈和方法区的回收，其中，
Java 堆是垃圾收集器的工作重点
        从次数上讲：
                频繁收集 Young 区
                较少收集 Old 区
                基本不收集元空间(方法区)

### 如何回收

#### 垃圾回收器

如果说垃圾收集算法是内存回收的方法论,那么收集器就是内存回收的实践者.

实际使用时,可以根据实际的使用场景选择不同的垃圾回收器,这也是 JVM 调优的重要部

