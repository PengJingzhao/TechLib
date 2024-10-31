##  运行数据区

###  运行时数据区的五个模块

` [1. The pc Register] ` 程序计数器/寄存器

` [2. Java Virtual Machine Stacks] ` Java虚拟机栈

` [3. Heap] ` 堆

` [4. Method Area] ` 方法区

` [5. Native Method Stacks] ` 本地方法栈

####  什么是方法区

方法区是用于存储类结构信息的地方，线程共享，包括常量池、静态变量、构造函数等类型信息，类型信息是由类加载器在类加载时从类.class文件中提取出来的。

官网的介绍；

` https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.5.1 `

![](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409141719302.webp)

从上面的介绍中，我们大致可以得出以下结论：  

    1. 方法区是各个线程共享的内存区域，在虚拟机启动时创建，生命周期和JVM生命周期一样。 
    2. 用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。 
    3. 虽然Java虚拟机规范把方法区描述为堆的一个逻辑部分，但是它却又一个别名叫做Non-Heap(非堆)，目 的是与Java堆区分开来。 
    4. 当方法区无法满足内存分配需求时，将抛出OOM=OutOfMemoryError异常。 

用一段代码来加深印象：          

```java
    /**  
     * @author 老田  
     * @version 1.0  
     * @date 2020/11/5 12:55  
     */  
public class User {  
        private static String a = "";  
        private static final int b = 10;  

}  
```


User.class类信息，以及静态变量a，常量b等信息是存放在方法区的。方法区的实现通常有两种：JDK8前的永久代，以及JDK8后的元空间。

####  什么是寄存器？

The pc Register 也有的翻译为pc寄存器。下面是官网对寄存器的解释，做了一个简要的翻译。


​    The Java Virtual Machine can support many threads of execution at once (JLS §17).   
​    Java虚拟机支持多线程并发  
​    Each Java Virtual Machine thread has its own pc (program counter) register.   
​    每个Java虚拟机线程都拥有一个寄存器  
​    At any point, each Java Virtual Machine thread is executing the code of a single method, namely the current method (§2.6) for that thread.   
​    在任何时候，每个Java虚拟机线程都在执行单个方法的代码，即该线程的当前方法  
​    If that method is not native, the pc register contains the address of the Java Virtual Machine instruction currently being executed.   
​    如果线程正在执行Java方法，则计数器记录的是正在执行的虚拟机字节码指令的地址；  
​    If the method currently being executed by the thread is native, the value of the Java Virtual Machine's pc register is undefined.   
​    如果正在执行的是Native方法，则这个计数器为空。  
​    The Java Virtual Machine's pc register is wide enough to hold a returnAddress or a native pointer on the specific platform.  
​    Java虚拟机的pc寄存器足够宽，可以容纳特定平台上的返回地址或本机指针。  


实际上，程序计数器占用的内存空间很小，由于Java虚拟机的多线程是通过线程轮流切换，并分配处理器执行时间的方式来实现的，在任意时刻，一个处理器只会执行一条线程中的指令。因此，为了线程切换后能够恢复到正确的执行位置，每条线程需要有一个独立的程序计数器(线程私有)。

我们都知道一个JVM进程中有多个线程在执行，而线程中的内容是否能够拥有执行权，是根据CPU调度来的。

假如线程A正在执行到某个地方，突然失去了CPU的执行权，切换到线程B了，然后当线程A再获得CPU执行权的时候，怎么能继续执行呢？

这就是需要在线程中维护一个变量，记录线程执行到的位置，记录本次已经执行到哪一行代码了，当CPU切换回来时候，再从这里继续执行。

####  什么是堆？

堆是Java虚拟机所管理内存中最大的一块，在虚拟机启动时创建，被所有线程共享。Java对象实例以及数组基本上都在堆上分配。官网介绍：


    1. JVM中的所有线程共享这个堆。  
    1. 所有的Java对象实例以及数组都在堆上分配。  
    1. 在虚拟机启动时创建  


在前面类加载阶段我们已经聊过了，在Java堆中生成一个代表这个类的java.lang.Class对象，作为对方法区中这些数据的访问入口。

####  堆在JDK1.7和JDK1.8的变化

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmkibiav5DdRpO7VhFOP6wUqY3icXzhYfpGtPgWibFLebNmnwECpF3dkdiaMQ/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

大家都知道，JVM 在运行时，会从操作系统申请大块的堆内内存，进行数据的存储。但是，堆外内存也就是申请后操作系统剩余的内存，也会有部分受到 JVM
的控制。比较典型的就是一些 native 关键词修饰的方法，以及对内存的申请和处理。

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmsVRNvzqPLzEchAociaLKRMSoTQtdYM4hKDzD0m0NHuPQLJyfibYR6f9A/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在 JVM中，堆被划分成两个不同的区域：新生代 ( ` Young ` )、老年代 ( ` Old ` )。

新生代 ( Young ) 又被划分为三个区域： ` Eden区 ` 、 ` From Survivor区 ` 、 ` To Survivor区 ` 。

**注意** ：很多的文章或者书籍里也称 ` From Survivor区 ` 为S0区， ` To Survivor区 ` 为S1区。

这样划分的目的是为了使JVM能够更好的管理堆内存中的对象，包括内存的分配以及回收。

根据之前对于Heap的介绍可以知道，一般对象和数组的创建会在堆中分配内存空间，关键是堆中有这么多区 域，那一个对象的创建到底在哪个区域呢？

#####  对象创建所在区域

一般情况下，新创建的对象都会被分配到Eden区（朝生夕死），一些特殊的大的对象会直接分配到Old区。

比如有对象A，B，C等创建在Eden区，但是Eden区的内存空间肯定有限，比如有100M，假如已经使用了

100M或者达到一个设定的临界值，这时候就需要对Eden内存空间进行清理，即垃圾收集(Garbage Collect)，

这样的GC我们称之为Minor GC，Minor GC指得是 ` Young区的GC ` 。

经过GC之后，有些对象就会被清理掉，有些对象可能还存活着，对于存活着的对象需要将其复制到Survivor

区，然后再清空Eden区中的这些对象。

` TLAB ` 的全称是 ` Thread Local Allocation Buffer ` ，JVM默认给每个线程开辟一个 buffer
区域，用来加速对象分配。这个 buffer 就放在 Eden 区中。

这个道理和 Java 语言中的ThreadLocal类似，避免了对公共区的操作，以及一些锁竞争。

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmcmdVia73dwzCJHRQcPXs40bkPesqAPU8Il6icpbecAkyMgOEqgThlnzA/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

对象的分配优先在TLAB上 分配，但 TLAB通常都很小，所以对象相对比较大的时候，会在 Eden 区的共享区域进行分配。

TLAB是一种优化技术，类似的优化还有对象的栈上分配（这可以引出逃逸分析的话题，默认开启）。这属于非常细节的优化，不做过多介绍，但偶尔面试也会被问到。

####  Survivor区详解

由图解可以看出，Survivor区分为两块S0和S1，也可以叫做From和To。在同一个时间点上，S0和S1只能有一个区有数据，另外一个是空的。

接着上面的GC来说，比如一开始只有Eden区和From中有对象，To中是空的。

此时进行一次GC操作，From区中对象的年龄就会+1，我们知道Eden区中所有存活的对象会被复制到To区，From区中还能存活的对象会有两个去处。

若对象年龄达到之前设置好的年龄阈值（默认年龄为15岁，可以自行设置参数 ` ‐XX:+MaxTenuringThreshold `
），此时对象会被移动到Old区， 如果Eden区和From区 没有达到阈值的对象会被复制到To区。

此时Eden区和From区已经被清空(被GC的对象肯定没了，没有被GC的对象都有了各自的去处)。

这时候From和To交换角色，之前的From变成了To，之前的To变成了From。也就是说无论如何都要保证名为To的Survivor区域是空的。

Minor GC会一直重复这样的过程，知道To区被填满，然后会将所有对象复制到老年代中。

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmjicYgqnDmMDeiaEjtRhTBHVavTVOUibGjUZy59dng6Ho6fxRKlVT3M19Q/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409141719256.webp)

#####  Old区

从上面的分析可以看出，一般Old区都是年龄比较大的对象，或者相对超过了某个阈值(-XX:PretenureSizeThreshold,默认为15，表示全部进Eden区)的对象。在Old区也会有GC的操作，Old区的GC我们称作为Major
GC。

###  什么是虚拟机栈？

Java虚拟机栈，是线程私有。

每一个线程拥有一个虚拟机栈，每一个栈包含n个栈帧，每个栈帧对应一次一个放调用，

每个栈帧里包含：局部变量表、操作数栈、动态链接、方法出口。

官网介绍


    1. 一个线程的创建也就同事创建一个java虚拟机栈  
       2. Java虚拟机堆栈存储帧  
       3. Java虚拟机堆栈的内存不需要是连续的。  


看一段代码      

```java
public class JavaStackDemo {  
    private void checkParam(String passWd, String userName) {  
        // TODO: 2020/11/6 用户名和密码校验   
    }  
  
    private void getUserName(String passWd, String userName) {  
        checkParam(passWd, userName);  
    }  
  
    private void login(String passWd, String userName) {  
        getUserName(passWd, userName);  
    }  
  
    public static void main(String[] args) {  
        //这里是演示代码，希望大家能结合自己平时写的代码理解，那样会更爽  
        //你就不再死记硬背了  
        JavaStackDemo javaStackDemo = new JavaStackDemo();  
        javaStackDemo.login("老田", "111111");  
    }  
}  
```


启动main方法就是启动了一个线程，JVM中会对应给这个线程创建一个栈。

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcm5FHUsOnoB1UroicggqV3FNWVt1H0FGPxiaWs6J6mMm3wPrKibG12NTRYQ/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

从这个调用过程很容易发现是个先进后出的结构，刚好栈的结构就是这样的。java虚拟机栈就是这么设计的：

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmqricE5U1lUKFmxXobfQTWZ1Ea11GjcrVpLibavm4wIHGEIOX8mDa7Scw/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

每个栈帧表示一个方法的调用：进入方法表示栈帧入栈，栈帧出栈表示方法调用结束。

多线程的话就是这样了：

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmibqe6gM5xw5ibd5c9ZTrUdq1zGBK4Vw7Bia28dIc7dMRdMeeXzTzNN04Q/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

从上面这个图大家会不会觉得这个栈有问题？其实也是有问题的，比如说看下面这段代码

```java
public class JavaStackDemo {  
    public static void main(String[] args) {  
        JavaStackDemo javaStackDemo = new JavaStackDemo();  
        javaStackDemo.test();  
    }  
    //循环调用test方法  
    private void test(){  
        test();  
    }  
}  
```


调用过程如下图：

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmozm7z8THPLu1FTRHuoyZEltuokmUohT6SCMm9n1z9o6U9wJu3N7H0w/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

是不是觉得很无语，调用方法就往栈里加入一个栈帧，这么下去，这个栈得需要多深才能放下，死循环和无限递归呢，岂不是栈里需要无限深度吗?

Java虚拟机栈大小肯定是有限的，所以就会导致一个大家都听说过的栈溢出。

运行上面的代码：

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmojj4fR8PNhEBmvoiaIkQA3AL2G8YQCfKIia3IkTEfwg9ZEThCibVdgRjg/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

###  如何设置Java虚拟机栈的大小呢？

我们可以使用虚拟机参数-Xss 选项来设置线程的最大栈空间，栈的大小直接决定了函数调用的最大可达深度； ` -Xss size `
设置线程堆栈大小（以字节为单位）。附加字母k或K表示KB，m或M表示MB，和g或G表示GB。默认值取决于平台：

  * Linux / x64（64位）：1024 KB 
  * macOS（64位）：1024 KB 
  * Oracle Solaris / x64（64位）：1024 KB 
  * Windows：默认值取决于虚拟内存 

下面的示例以不同的单位将线程堆栈大小设置为1024 KB：


​    -Xss1m （1mb）  
​    -Xss1024k  (1024kb)  
​    -Xss1048576  


回到上面的话题。

####  什么是栈帧？

上面提到过，调用方法就生成一个栈帧，然后入栈。

看一段代码      

```java
public class JavaStackDemo {  
    public static void main(String[] args) {  
        JavaStackDemo javaStackDemo = new JavaStackDemo();  
        javaStackDemo.getUserType(21);  
    }  
  
    public String getUserType(int age) {  
        int temp = 18;  
        if (age < temp) {  
            return "未成年人";  
        }  
        //动态链接  
        //userService.xx();  
        return "成年人";  
    }   
}  
```


既然是和方法有关，那么就可以联想到方法里都有些什么

官网介绍 

每个栈帧拥有自己的本地变量。比如上面代码里的
    int age、int temp  


这些都是本地变量。

每个栈帧都有自己的操作数栈

通过javac编译好JavaStackDemo，然后使用


​    javap -v JavaStackDemo.class >log.txt  


将字节码导入到log.txt中，打开

![](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409141719506.webp)

对getUserType方法里面的字节码做一个解释。有时候本地变量通过javap看不到，可以再javac的时候添加一个参数

` javac -g:vars XXX.class ` 这样就可以把本地变量表给输出来了。


​    
​    指令bipush 18  将18压入操作数栈  
​    istore_2 将栈顶int型数值存入第三个本地变量  
​    iload_1 将第二个int型本地变量推送至栈顶  
​    iload_2 将第三个int型本地变量推送至栈顶  
​    if_icmpge 比较栈顶两int型数值大小, 当结果大于等于0时跳转  
​    ldc 将int,float或String型常量值从常量池中推送至栈顶  
​    areturn 从当前方法返回对象引用  


官网

` https://docs.oracle.com/javase/specs/jvms/se8/html/ `

![](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409141719528.webp)

这些都是字节码指令。  

#####  LocalVariableTable 本地变量表


​    
​    Start  Length  Slot  Name   Signature  
​       0      14     0  this   Lcom/tian/demo/test/JavaStackDemo;  
​       0      14     1   age   I  
​       3      11     2  temp   I  


自己this算一个本地变量，入参age算一个本地变量，方法中的临时变量temp也算一个本地变量。

#####  方法出口

return。如果方法不需要返回void的时候，其实方法里是默认会为其加上一个return;

另外方法的返回分两种：

  * 正常代码执行完毕然后return。 

  * 遇到异常结束 

#####  栈帧总结

  * 方法出口：return或者程序异常 

  * 局部变量表：保存局部变量 

  * 操作数栈：保存每次赋值、运算等信息 

  * 动态链接：相对于C/C++的静态连接而言，静态连接是将所有类加载，不论是否使用到。而动态链接是要用到某各类的时候在加载到内存里。静态连接速度快，动态链接灵活性更高。 

###  什么是本地方法栈？

Native Method Stacks
翻译过来就是本地方法栈，与Java虚拟机栈一样，但这里的栈是针对native修饰的方法的，比如System、Unsafe、Object类中的相关native方法。


   ````java
    public class Object {  
           //native修饰的方法  
           private static native void registerNatives();  
           public final native Class<?> getClass();  
           public native int hashCode();  
           protected native Object clone() throws CloneNotSupportedException;  
           public final native void notify();  
           //.......  
       }      
       public final class System {  
           //native修饰的方法  
           private static native void registerNatives();  
           static {  
               registerNatives();  
           }  
           public static native long currentTimeMillis();  
           private static native void setIn0(InputStream in);  
           private static native void setOut0(PrintStream out);  
           private static native void setErr0(PrintStream err);  
           //.....  
       }  
       public final class Unsafe {  
           //native修饰的方法  
           private static native void registerNatives();  
           public native int getInt(Object var1, long var2);  
           public native void putInt(Object var1, long var2, int var4);  
           public native Object getObject(Object var1, long var2);  
           public native void putObject(Object var1, long var2, Object var4);  
           public native boolean getBoolean(Object var1, long var2);  
           //...  
       }    
   ````




面试常问：JVM运行时区那些和线程有直接的关系和间接的关系，哪些区会发生OOM?

每个区域是否为线程共享，是否会发生OOM

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmZB5UicTTMFJRMccwrt7EXfuhlE9w8K9VbPkgiaaiamegj3NJpnKc7nyyQ/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

###  如何判断对象是垃圾对象？

####  引用计数法

给对象添加一个引用计数器，每当一个地方引用它object时技术加1，引用失去以后就减1，计数为0说明不再引用。

  * 优点：实现简单，判定效率高 
  * 缺点：无法解决对象相互循环引用的问题，对象A中引用了对象B，对象B中引用对象A。 

![](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409141719650.webp)

```java
  public class A {   
    public B b;   
    }  
    public class B {  
    public C c;   
    }   
    public class C {   
    public A a;   
    }  
      
public class Test{  
  private void test(){  
      A a = new A();  
      B b = new B();  
      C c = new C();  
        
      a.b=b;  
      b.c=c;  
      c.a=a;  
  }  
}  
```


####  可达性分析算法

当一个对象到GC Roots没有引用链相连，即就是GC Roots到这个对象不可达时，证明对象不可用。

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmPexiaLtpdpNI2udEMZzjC4ngiaT5wicvfWCmEJuGyPriaIWNicBdu4Oua6w/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

####  GC Roots种类

  * Java 线程中，当前所有正在被调用的方法的引用类型参数、局部变量、临时值等。也就是与我们栈帧相关的各种引用。 

  * 所有当前被加载的 Java 类。 

  * Java 类的引用类型静态变量。 

  * 运行时常量池里的引用类型常量（String 或 Class 类型）。 

  * JVM 内部数据结构的一些引用，比如 sun.jvm.hotspot.memory.Universe 类。 

  * 用于同步的监控对象，比如调用了对象的 wait() 方法。 

    

    ````java
    public class Test{  
        private void test(C c){  
            A a = new A();  
            B b = new B();  
            a.b=b;  
            //这里的a/b/c都是GC Root；  
        }  
    }  
    ````

    

####  对象的引用类型有哪些？

  * 强引用：User user=new User()；我们开发中使用最多的对象引用方式。特点：我们平常典型编码Object obj = new Object()中的obj就是强引用。通过关键字new创建的对象所关联的引用就是强引用。当JVM内存空间不足，JVM宁愿抛出OutOfMemoryError运行时错误（OOM），使程序异常终止，也不会靠随意回收具有强引用的“存活”对象来解决内存不足的问题。对于一个普通的对象，如果没有其他的引用关系，只要超过了引用的作用域或者显式地将相应（强）引用赋值为 null，就是可以被垃圾收集的了，具体回收时机还是要看垃圾收集策略。 
  * 软引用：SoftReference object=new SoftReference(new Object()); 特点：软引用通过SoftReference类实现。软引用的生命周期比强引用短一些。只有当 JVM 认为内存不足时，才会去试图回收软引用指向的对象：即JVM 会确保在抛出 OutOfMemoryError 之前，清理软引用指向的对象。软引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被垃圾回收器回收，Java虚拟机就会把这个软引用加入到与之关联的引用队列中。后续，我们可以调用ReferenceQueue的poll()方法来检查是否有它所关心的对象被回收。如果队列为空，将返回一个null,否则该方法返回队列中前面的一个Reference对象。应用场景：软引用通常用来实现内存敏感的缓存。如果还有空闲内存，就可以暂时保留缓存，当内存不足时清理掉，这样就保证了使用缓存的同时，不会耗尽内存。 
  * 弱引用：WeakReference object=new WeakReference (new Object();ThreadLocal中有使用. 弱引用通过WeakReference类实现。弱引用的生命周期比软引用短。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。由于垃圾回收器是一个优先级很低的线程，因此不一定会很快回收弱引用的对象。弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java虚拟机就会把这个弱引用加入到与之关联的引用队列中。应用场景：弱应用同样可用于内存敏感的缓存。 
  * 虚引用：几乎没见过使用， ReferenceQueue 、PhantomReference 

finalize()方法有什么作用？

这个方法就有点类似，某个人被拍了死刑，但是不一定会死。

即使在可达性分析算法中不可达的对象，也并非一定是“非死不可”的，这时候他们暂时处于“缓刑”阶段，真正宣告一个对象死亡至少要经历两个阶段：

1、如果对象在可达性分析算法中不可达，那么它会被第一次标记并进行一次刷选，刷选的条件是是否需要执行finalize()方法（当对象没有覆盖finalize()或者finalize()方法已经执行过了（对象的此方法只会执行一次）），虚拟机将这两种情况都会视为没有必要执行）。

2、如果这个对象有必要执行finalize()方法会将其放入F-Queue队列中，稍后GC将对F-
Queue队列进行第二次标记，如果在重写finalize()方法中将对象自己赋值给某个类变量或者对象的成员变量，那么第二次标记时候就会将它移出“即将回收”的集合。