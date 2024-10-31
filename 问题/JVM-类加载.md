##  类加载

###  如何查找class文件并导入到JVM中

(1)通过一个类的全限定名获取定义此类的二进制字节流

(2)将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构

(3)在Java堆中生成一个代表这个类的java.lang.Class对象，作为对方法区中这些数据的访问入口

###  获取class文件有哪些方式

` .class ` 文件也是需要查找的，以下是查找.class文件的常用方式：

    1. 从本地文件系统中加载.class文件 
    2. 从jar包中或者war包中加载.class文件 
    3. 通过网络或者从数据库中加载.class文件 
    4. 把一个Java源文件动态编译，并加载 

加载进来后就，系统为这个.class文件生成一个对应的Class对象。

###  生成Class对象的有哪些方式

1.对象获取:调用person类的父类方法getClaass();

2.类名获取，每个类型(包括基本类型和引用)都有一个静态属性，class。

3.Class类的静态方法获取。forName("字符串的类名")写全名，要带包名。 (包名.类名)

###  类加载机制

类加载机制分为三步：

` [Loading] ` 装载：其实就是我们上面查找class文件并导入到JVM中。

` [Linking] ` 连接：就是对整个class内容进行一系列的校验、为一些变量进行数据准备、把字节码中符号进行解析等操作。

` [Initializing] ` 初始化：创建我们使用的对象； ` User user=new User(); `

其中连接又分三个步骤：验证、准备、解析。

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmlM9V9bf85gbhDYDgjoCkUxGCjz20LEwlpF5mEaAnKRSzVxHwaqsLFA/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

###  连接

####  验证

首先肯定是要保证被加载类的正确性，也就是做一些 ` .class ` 文件人的校验罢了；

  * 文件格式验证 

  * 元数据验证 

  * 字节码验证 

  * 符号引用验证 

####  准备

为类的静态变量分配内存空间，并将其初始化为默认值。

比如说User.java中有个变量int a;

```java
 public class InitialDemo {  
        static int a = 10;  
          
        public static void main(String[] args) {  
            System.out.println(a);  
        }  
 }  
```


在这个阶段，会对这些static修饰的变量进行赋值，附一个初始值，这里就是给

` int a =0; `

因为int类型的初始值就是0；如果是String类型，那么初始值就是null。

####  解析

初始值搞定后，还有就是有部分对象引用的，在.class字节码文件中还是符号，得给指定一个真实引用地址。

换言之，把符号引用变成直接引用。

#####  符号引用

符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能够无歧义的定位到目标即可。

例如，在Class文件中通过javap命令能查看，它以

` CONSTANT_Class_info ` 、

` CONSTANT_Fieldref_info ` 、

` CONSTANT_Methodref_info ` 等类型的常量出现。

符号引用与虚拟机的内存布局无关，引用的目标并不一定加载到内存中。在Java中，一个java类将会编译成一个class文件。

在编译时，java类并不知道所引用的类的实际地址，因此只能使用符号引用来代替。

比如： ` org.simple.People ` 类引用了 ` org.simple.Language `
类，在编译时People类并不知道Language类的实际内存地址，因此只能使用符号 ` org.simple.Language `
（假设是这个，当然实际中是由类似于 ` CONSTANT_Class_info ` 的常量来表示的）来表示Language类的地址。

各种虚拟机实现的内存布局可能有所不同，但是它们能接受的符号引用都是一致的，因为符号引用的字面量形式明确定义在Java虚拟机规范的Class文件格式中。

#####  直接引用

直接引用可以是以下三种场景：

（1）直接指向目标的指针（比如，指向“类型”【Class对象】、类变量、类方法的直接引用可能是指向方法区的指针）

（2）相对偏移量（比如，指向实例变量、实例方法的直接引用都是偏移量）

（3）一个能间接定位到目标的句柄

直接引用是和虚拟机的布局相关的，同一个符号引用在不同的虚拟机实例上翻译出来的直接引用一般不会相同。

如果有了直接引用，那引用的目标必定已经被加载入内存中了。

##  类加载器

在装载(Load)阶段，通过类的全限定名获取其定义的二进制字节流，需要借助类装载 器完成，顾名思义，就是用来装载Class文件的。

###  类装载器分类

####  Bootstrap ClassLoader

负责加载$JAVA_HOME中
jre/lib/rt.jar里所有的class或Xbootclassoath选项指定的jar包。由C++实现，不是ClassLoader子类。

####  Extension ClassLoader

负责加载Java平台中扩展功能的一些jar包，包括$JAVA_HOME中jre/lib/*.jar或 -Djava.ext.dirs指定目录下的jar包。

####  App ClassLoader

负责加载classpath中指定的jar包及 Djava.class.path所指定目录下的类和jar包。

####  Custom ClassLoader

通过java.lang.ClassLoader的子类自定义加载class，属于应用程序根据自身需要自定义的ClassLoader，如tomcat、jboss都会根据j2ee规范自行实现ClassLoader。

**图解类加载**

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmicuSClLpqsCJ8g4Asd7Jve5y4w3KPA2mabu0f8LxxEib26FwSYiaSEk8Q/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

####  加载原则

检查某个类是否已经加载：顺序是自底向上，从Custom ClassLoader到BootStrap ClassLoader逐层检

查，只要某个Classloader已加载，就视为已加载此类，保证此类只所有ClassLoader加载一次。

加载的顺序：先查找是否已经加载过，当没有被加载过，则加载的顺序是自顶向下，也就是由上层来逐层尝试加载此类。

java.lang.ClassLoader中很重要的三个方法：

  * loadClass方法 

  * findClass方法 

  * defineClass方法 

#####  loadClass方法 
      

```java
		public Class<?> loadClass(String name) throws ClassNotFoundException {  
            return loadClass(name, false);  
        }  
        protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException{  
            //使用了同步锁，保证不出现重复加载  
            synchronized (getClassLoadingLock(name)) {  
                // 首先检查自己是否已经加载过  
                Class<?> c = findLoadedClass(name);  
                //没找到  
                if (c == null) {  
                    long t0 = System.nanoTime();  
                    try {  
                        //有父类  
                        if (parent != null) {  
                            //让父类去加载  
                            c = parent.loadClass(name, false);  
                        } else {  
                            //如果没有父类，则委托给启动加载器去加载  
                            c = findBootstrapClassOrNull(name);  
                        }  
                    } catch (ClassNotFoundException e) {  
                        // ClassNotFoundException thrown if class not found  
                        // from the non-null parent class loader  
                    } 
                if (c == null) {  
                    // If still not found, then invoke findClass in order  
                    // to find the class.  
                    long t1 = System.nanoTime();  
                    // 如果都没有找到，则通过自定义实现的findClass去查找并加载  
                    c = findClass(name);  
  
                    // this is the defining class loader; record the stats  
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);  
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);  
                    sun.misc.PerfCounter.getFindClasses().increment();  
                }  
            }  
            //是否需要在加载时进行解析  
            if (resolve) {  
                resolveClass(c);  
            }  
            return c;  
        }  
    }  
```


正如loadClass方法所展示的，当类加载请求到来时，先从缓存中查找该类对象，如果存在直接返回，如果不存在则交给该类加载去的父加载器去加载，倘若没有父加载则交给顶级启动类加载器去加载，最后倘若仍没有找到，则使用findClass()方法去加载（关于findClass()稍后会进一步介绍）。从loadClass实现也可以知道如果不想重新定义加载类的规则，也没有复杂的逻辑，只想在运行时加载自己指定的类，那么我们可以直接使用this.getClass().getClassLoder.loadClass("className")，这样就可以直接调用ClassLoader的loadClass方法获取到class对象。

#####  findClass方法

   ````java
       protected Class<?> findClass(String name) throws ClassNotFoundException {  
                   throw new ClassNotFoundException(name);  
       }  
   ````


在JDK1.2之前，在自定义类加载时，总会去继承ClassLoader类并重写loadClass方法，从而实现自定义的类加载类，但是在JDK1.2之后已不再建议用户去覆盖loadClass()方法，而是建议把自定义的类加载逻辑写在findClass()方法中，从前面的分析可知，findClass()方法是在loadClass()方法中被调用的，当loadClass()方法中父加载器加载失败后，则会调用自己的findClass()方法来完成类加载，这样就可以保证自定义的类加载器也符合双亲委托模式。

需要注意的是ClassLoader类中并没有实现findClass()方法的具体代码逻辑，取而代之的是抛出ClassNotFoundException异常，

同时应该知道的是findClass方法通常是和defineClass方法一起使用的。

#####  defineClass方法

    ````java
        protected final Class<?> defineClass(String name, byte[] b, int off, int len,  
                                        ProtectionDomain protectionDomain) throws ClassFormatError{  
                    protectionDomain = preDefineClass(name, protectionDomain);  
                    String source = defineClassSourceLocation(protectionDomain);  
                    Class<?> c = defineClass1(name, b, off, len, protectionDomain, source);  
                    postDefineClass(c, protectionDomain);  
                    return c;  
                }  
    ````



defineClass()方法是用来将byte字节流解析成JVM能够识别的Class对象。通过这个方法不仅能够通过class文件实例化class对象，也可以通过其他方式实例化class对象，如通过网络接收一个类的字节码，然后转换为byte字节流创建对应的Class对象
。

###  如何自定义类加载器

用户根据需求自己定义的。需要继承自ClassLoader,重写方法findClass()。

如果想要编写自己的类加载器，只需要两步：

  * 继承ClassLoader类 
  * 覆盖findClass(String className)方法 

ClassLoader超类的loadClass方法用于将类的加载操作委托给其父类加载器去进行，只有当该类尚未加载并且父类加载器也无法加载该类时，才调用findClass方法。

如果要实现该方法，必须做到以下几点：

1.为来自本地文件系统或者其他来源的类加载其字节码。

2.调用ClassLoader超类的defineClass方法，向虚拟机提供字节码。

##  双亲委派模型

###  什么是双亲委派模型

如果一个类加载器在接到加载类的请求时，先在缓存中查找是否已经加载过，如果没有被加载过，它首先不会自己尝试去加载这个类，而是把这个请求任务委托给父类加载器去完成，依次递归，如果父类加载器可以完成类加载任务，就成功返回；只有父类加载器无法完成此加载任务时，才自己去加载。

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmXQSl1ibTgq0NdppuibcJtGuO59NbKmI0icicvW9mJDyGsHhK3KGUjsib6Jg/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

###  双亲委派模型有什么好处？

Java类随着加载它的类加载器一起具备了一种带有优先级的层次关系。

比如，Java中的Object类，它存放在rt.jar之中,无论哪一个类加载器要加载这个类，最终都是委派给处于模型最顶端的启动类加载器进行加载，因此Object在各种类加载环境中都是同一个类。

如果不采用双亲委派模型，那么由各个类加载器自己取加载的话，那么系统中会存在多种不同的Object类。

可以保证相同的类由同一个类加载器加载

###  打破双亲委派模型的案例

####  tomcat

tomcat 通过 war 包进行应用的发布，它其实是违反了双亲委派机制原则的。简单看一下 tomcat 类加载器的层次结构。

![](https://mmbiz.qpic.cn/mmbiz_png/07BicZywOVtnP2xPtkOAgu6ficjZ3o0tcmlPnpUzicEmXcXKFjmhNbPsBMlVqkGte4XTpHpicsbyCGXiathhRAibsNtQ/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

对于一些需要加载的非基础类，会由一个叫作 WebAppClassLoader 的类加载器优先加载。等它加载不到的时候，再交给上层的 ClassLoader
进行加载。这个加载器用来隔绝不同应用的 .class 文件，比如你的两个应用，可能会依赖同一个第三方的不同版本，它们是相互没有影响的。

如何在同一个 JVM 里，运行着不兼容的两个版本，当然是需要自定义加载器才能完成的事。

那么 tomcat 是怎么打破双亲委派机制的呢？可以看图中的 WebAppClassLoader，它加载自己目录下的 .class
文件，并不会传递给父类的加载器。但是，它却可以使用 SharedClassLoader 所加载的类，实现了共享和分离的功能。

但是你自己写一个 ArrayList，放在应用目录里，tomcat 依然不会加载。它只是自定义的加载器顺序不同，但对于顶层来说，还是一样的。

####  OSGi

OSGi 曾经非常流行，Eclipse 就使用 OSGi 作为插件系统的基础。OSGi
是服务平台的规范，旨在用于需要长运行时间、动态更新和对运行环境破坏最小的系统。

OSGi 规范定义了很多关于包生命周期，以及基础架构和绑定包的交互方式。这些规则，通过使用特殊 Java 类加载器来强制执行，比较霸道。

比如，在一般 Java 应用程序中，classpath 中的所有类都对所有其他类可见，这是毋庸置疑的。但是，OSGi 类加载器基于 OSGi
规范和每个绑定包的 manifest.mf
文件中指定的选项，来限制这些类的交互，这就让编程风格变得非常的怪异。但我们不难想象，这种与直觉相违背的加载方式，肯定是由专用的类加载器来实现的。

随着 jigsaw 的发展（旨在为 Java SE 平台设计、实现一个标准的模块系统），我个人认为，现在的 OSGi，意义已经不是很大了。OSGi
是一个庞大的话题，你只需要知道，有这么一个复杂的东西，实现了模块化，每个模块可以独立安装、启动、停止、卸载，就可以了。

####  SPI

Java 中有一个 SPI 机制，全称是 Service Provider Interface，是 Java 提供的一套用来被第三方实现或者扩展的
API，它可以用来启用框架扩展和替换组件。

后面会专门针对这个写一篇文章，这里就不细说了。

###  双亲委派模型为什么安全？

可以防止一些核心类被相同全限定名的类所覆盖

前面谈到双亲委派机制是为了安全而设计的，但是为什么就安全了呢？举个例子，ClassLoader加载的class文件来源很多，比如编译器编译生成的class、或者网络下载的字节码。

而一些来源的class文件是不可靠的，比如我可以自定义一个java.lang.Integer类来覆盖jdk中默认的Integer类，例如下面这样：      

```java
package java.lang;  

/**  
 * @author 田维常  
 * @version 1.0  
 * @date 2020/11/7 21:18  
 */  
public class Integer {  
    public Integer(int value) {  
        System.exit(0);  
    }  
  
    public static void main(String[] args) {  
        Integer i = new Integer(1);  
        System.err.println(i);  
    }  
}  
```


初始化这个Integer的构造器是会退出JVM，破坏应用程序的正常进行，如果使用双亲委派机制的话，该Integer类永远不会被调用，因为委托BootStrapClassLoader加载后会加载JDK中的Integer类而不会加载自定义的这个，运行main方法，程序并没有执行System.exit(0);

![](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409141708549.webp)

这里使用的是rt.jar里的Integer，并没有使用我们自定义的Integer类，这个案例和前面的能不能自定义一个String类完全一样，这样就保证了安全性。