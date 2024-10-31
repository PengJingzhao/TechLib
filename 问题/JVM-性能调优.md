##  性能调优

###  熟悉哪些JVM调优参数

X或者XX开头的都是非转标准化参数:

![](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409141730580.webp)

意思就是说准表化参数不会变，非标准化参数可能在每个JDK版本中有所变化，但是就目前来看X开头的非标准化的参数改变的也是非常少。


        格式：-XX:[+-]<name> 表示启用或者禁用name属性。  
    例子：-XX:+UseG1GC（表示启用G1垃圾收集器）  


###  堆设置

    * -Xms 初始堆大小，ms是memory start的简称 ，等价于-XX:InitialHeapSize 
    
    * -Xmx 最大堆大小，mx是memory max的简称 ，等价于参数-XX:MaxHeapSize 

注意：在通常情况下，服务器项目在运行过程中，堆空间会不断的收缩与扩张，势必会造成不必要的系统压力。所以在生产环境中，JVM的Xms和Xmx要设置成一样的，能够避免GC在调整堆大小带来的不必要的压力。

    * -XX:NewSize=n 设置年轻代大小 
    
    * -XX:NewRatio=n 设置年轻代和年老代的比值。如:-XX:NewRatio=3，表示年轻代与年老代比值为1：3，年轻代占整个年轻代年老代和的1/4，默认新生代和老年代的比例=1:2。 
    
    * -XX:SurvivorRatio=n 年轻代中Eden区与两个Survivor区的比值。注意Survivor区有两个，默认是8，表示 

Eden:S0:S1=8:1:1

如：-XX:SurvivorRatio=3，表示Eden：Survivor=3：2，一个Survivor区占整个年轻代的1/5。

    * -XX:MaxPermSize=n 设置持久代大小 
    
    * -XX:MetaspaceSize 设置元空间大小 

###  收集器设置

    * -XX:+UseSerialGC 设置串行收集器 
    
    * -XX:+UseParallelGC 设置并行收集器 
    
    * -XX:+UseParalledlOldGC 设置并行年老代收集器 
    
    * -XX:+UseConcMarkSweepGC 设置并发收集器 

###  垃圾回收统计信息

    * -XX:+PrintGC 
    
    * -XX:+PrintGCDetails 
    
    * -XX:+PrintGCTimeStamps 
    
    * -Xloggc:filenameGC日志输出到文件里filename，比如：-Xloggc:/gc.log 

###  并行收集器设置

    * -XX:ParallelGCThreads=n 设置并行收集器收集时使用的CPU数。并行收集线程数。 
    
    * -XX:MaxGCPauseMillis=n 设置并行收集最大暂停时间 
    
    * -XX:GCTimeRatio=n 设置垃圾回收时间占程序运行时间的百分比。公式为1/(1+n) 
    
    * -XX:MaxGCPauseMillis=n设置并行收集最大暂停时间 

###  并发收集器设置

    * -XX:+CMSIncrementalMode 设置为增量模式。适用于单CPU情况。 
    
    * -XX:ParallelGCThreads=n 设置并发收集器年轻代收集方式为并行收集时，使用的CPU数。并行收集线程数。 

###  其他

    * -XX:+PrintCommandLineFlags查看当前JVM设置过的相关参数 

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmJ1HNpMNf9EXSojfRe7WLTRwWvoPcxsl4obxxibBiags2bjYQRia5hichqA/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

###  Dump异常快照  

    * -XX:+HeapDumpOnOutOfMemoryError 
    
    * -XX:HeapDumpPath 

堆内存出现OOM的概率是所有内存耗尽异常中最高的，出错时的堆内信息对解决问题非常有帮助，所以给JVM设置这个参数(-XX:+HeapDumpOnOutOfMemoryError)，让JVM遇到OOM异常时能输出堆内信息，并通过（-XX:+HeapDumpPath）参数设置堆内存溢出快照输出的文件地址，这对于特别是对相隔数月才出现的OOM异常尤为重要。


        -Xms10M -Xmx10M -Xmn2M -XX:SurvivorRatio=8 -XX:+HeapDumpOnOutOfMemoryError   
    -XX:HeapDumpPath=D:\study\log_hprof\gc.hprof  


    * -XX:OnOutOfMemoryError 

表示发生OOM后，运行jconsole.exe程序。这里可以不用加“”，因为jconsole.exe路径Program
Files含有空格。利用这个参数，我们可以在系统OOM后，自定义一个脚本，可以用来发送邮件告警信息，可以用来重启系统等等。


        -XX:OnOutOfMemoryError="C:\Program Files\Java\jdk1.8.0_151\bin\jconsole.exe"  


##  JVM 调优常见目标

JVM 调优目标：使用较小的内存占用来获得较高的吞吐量或者较低的延迟。

程序在上线前的测试或运行中有时会出现一些大大小小的 JVM 问题，比如 cpu load 过高、请求延迟、tps
降低等，甚至出现内存泄漏（每次垃圾收集使用的时间越来越长，垃圾收集频率越来越高，每次垃圾收集清理掉的垃圾数据越来越少）、内存溢出导致系统崩溃，因此需要对
JVM 进行调优，使得程序在正常运行的前提下，获得更高的用户体验和运行效率。

这里有几个比较重要的指标：

    * 内存占用：程序正常运行需要的内存大小。 
    * 延迟：由于垃圾收集而引起的程序停顿时间。 
    * 吞吐量：用户程序运行时间占用户程序和垃圾收集占用总时间的比值。 

当然，和 CAP
原则一样，同时满足一个程序内存占用小、延迟低、高吞吐量是不可能的，程序的目标不同，调优时所考虑的方向也不同，在调优之前，必须要结合实际场景，有明确的的优化目标，找到性能瓶颈，对瓶颈有针对性的优化，最后进行测试，通过各种监控工具确认调优后的结果是否符合目标。

##  有哪些调优工具？

###  JPS

用 jps（JVM process Status）可以查看虚拟机启动的所有进程、执行主类的全名、JVM启动参数，比如当执行了 JPSTest 类中的
main 方法后（main 方法持续执行），执行 jps -l可看到下面的JPSTest类的 pid 为 31354，加上 -v
参数还可以看到JVM启动参数。

###  jstat

用 jstat（JVM Statistics Monitoring Tool）监视虚拟机信息 jstat -gc pid 500 10：每 500
毫秒打印一次 Java 堆状况（各个区的容量、使用容量、gc 时间等信息），打印 10 次。jstat
还可以以其他角度监视各区内存大小、监视类装载信息等，具体可以 google jstat 的详细用法。

###  jmap

用 jmap（Memory Map for Java）查看堆内存信息 执行 jmap -histo pid
可以打印出当前堆中所有每个类的实例数量和内存占用，如下，class name 是每个类的类名（[B 是 byte 类型，[C是 char 类型，[I 是
int 类型），bytes 是这个类的所有示例占用内存大小，instances 是这个类的实例数量。

执行 jmap -dump 可以转储堆内存快照到指定文件，比如执行：


        jmap -dump:format=b,file=/data/jvm/dumpfile_jmap.hprof 3361  


可以把当前堆内存的快照转储到 dumpfile_jmap.hprof 文件中，然后可以对内存快照进行分析。

###  jconsole、jvisualvm

利用 jconsole、jvisualvm 分析内存信息（各个区如 Eden、Survivor、Old 等内存变化情况），如果查看的是远程服务器的
JVM，程序启动需要加上如下参数：


        "-Dcom.sun.management.jmxremote=true"   
    "-Djava.rmi.server.hostname=12.34.56.78"   
    "-Dcom.sun.management.jmxremote.port=18181"   
    "-Dcom.sun.management.jmxremote.authenticate=false"   
    "-Dcom.sun.management.jmxremote.ssl=false"  


下图是 jconsole 界面，概览选项可以观测堆内存使用量、线程数、类加载数和 CPU
占用率；内存选项可以查看堆中各个区域的内存使用量和左下角的详细描述（内存大小、GC 情况等）；线程选项可以查看当前 JVM
加载的线程，查看每个线程的堆栈信息，还可以检测死锁；VM 概要描述了虚拟机的各种详细参数。

###  第三方工具

MAT、GChisto、GCViewer、JProfiler、arthas、async-profile。

##  JVM 调优经验总结

JVM 配置方面，一般情况可以先用默认配置（基本的一些初始参数可以保证一般的应用跑的比较稳定了），在测试中根据系统运行状况（会话并发情况、会话时间等），结合
gc 日志、内存监控、使用的垃圾收集器等进行合理的调整，当老年代内存过小时可能引起频繁 Full GC，当内存过大时 Full GC 时间会特别长。

那么 JVM 的配置比如新生代、老年代应该配置多大最合适呢？答案是不一定，调优就是找答案的过程，物理内存一定的情况下，新生代设置越大，老年代就越小，Full
GC 频率就越高，但 Full GC 时间越短；相反新生代设置越小，老年代就越大，Full GC 频率就越低，但每次 Full GC 消耗的时间越大。

建议如下：

-Xms 和 -Xmx 的值设置成相等，堆大小默认为 -Xms 指定的大小，默认空闲堆内存小于 40% 时，JVM 会扩大堆到 -Xmx 指定的大小；空闲堆内存大于 70% 时，JVM 会减小堆到 -Xms 指定的大小。如果在 Full GC 后满足不了内存需求会动态调整，这个阶段比较耗费资源。 

    * 新生代尽量设置大一些，让对象在新生代多存活一段时间，每次 Minor GC 都要尽可能多的收集垃圾对象，防止或延迟对象进入老年代的机会，以减少应用程序发生 Full GC 的频率。 
    * 老年代如果使用 CMS 收集器，新生代可以不用太大，因为 CMS 的并行收集速度也很快，收集过程比较耗时的并发标记和并发清除阶段都可以与用户线程并发执行。 
    * 方法区大小的设置，1.6 之前的需要考虑系统运行时动态增加的常量、静态变量等，1.7 只要差不多能装下启动时和后期动态加载的类信息就行。 

代码实现方面，性能出现问题比如程序等待、内存泄漏除了 JVM 配置可能存在问题，代码实现上也有很大关系：

    * 避免创建过大的对象及数组：过大的对象或数组在新生代没有足够空间容纳时会直接进入老年代，如果是短命的大对象，会提前出发 Full GC。 
    * 避免同时加载大量数据，如一次从数据库中取出大量数据，或者一次从 Excel 中读取大量记录，可以分批读取，用完尽快清空引用。 
    * 当集合中有对象的引用，这些对象使用完之后要尽快把集合中的引用清空，这些无用对象尽快回收避免进入老年代。 
    * 可以在合适的场景（如实现缓存）采用软引用、弱引用，比如用软引用来为 ObjectA 分配实例：SoftReference  objectA=new SoftReference  (); 在发生内存溢出前，会将 objectA 列入回收范围进行二次回收，如果这次回收还没有足够内存，才会抛出内存溢出的异常。 

避免产生死循环，产生死循环后，循环体内可能重复产生大量实例，导致内存空间被迅速占满。

* 尽量避免长时间等待外部资源（数据库、网络、设备资源等）的情况，缩小对象的生命周期，避免进入老年代，如果不能及时返回结果可以适当采用异步处理的方式等。 