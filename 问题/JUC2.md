![cover_image](https://mmbiz.qpic.cn/mmbiz_jpg/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D4QWiadjRCKNHofK8VoAhV1C7ddplm5GlODvzDalFQWKAQBv7ttejP85g/0?wx_fmt=jpeg)

#  万字长文扫盲JUC基础(上)  

#  JUC概述

  

##  什么是JUC

> ❝
>
> java.utils.concurrent Java并发编程工具包的简称,是一个处理线程的工具包,JDK1.5开始出现
>
> ❞

  

##  线程和进程的概念

  

###  进程和线程

  * 进程(process) 

> ❝
>
>   * 进程是计算机中关于某数据集合的一次运行活动
>   * 是系统进行资源分配和调度的基本单位
>   * 进程是线程的容器
>
❞

  * 线程(thread) 

> ❝
>
>   * 是操作系统能够进行调度的最小单位
>   * 被包含于进程之中,是进程在实际运行的单位
>   * 一条线程指的是进程中一个单一顺序的控制流,
>   * 一个进程中可以并发多个线程,每条线程执行不同的任务
>
❞

  * 小结 
    * 进程是指在操作系统中正在运行的一个应用程序,程序一旦运行就是进程,进程是系统资源分配的最小单位 
    * 线程系统分配处理器时间资源的基本单元,换而言之是进程之内独立运行的一个单元执行流,是程序执行的最小单位 

###  线程的状态

  * 线程状态枚举类 

    
    
    ````java
    public enum State{  
        NEW,//新建  
        RUNNABLE,//就绪  
        BLOCKED,//阻塞  
        WAITING,//一直等待  
        TIMED_WAITTING,//超时等待  
        TERMINATED;//终结  
    }  
    ````
    
    

  

###  wait和sleep

  1. sleep是Thread的静态方法,wait是Object的方法,任何对象实例都能调用 
  2. sleep不会释放锁,也不需要占用锁,wait会释放锁,调用它的前提是当前线程占有锁(代码要在synchronized中) 
  3. 在哪睡,在哪醒,均可被interrupted方法中断 

###  并发和并行

  1. 并发同一瞬间,多个线程同时抢占一个cpu 
  2. 并行同一时段,多个线程同时执行(多cpu) 

###  管程monitor

  1. 管程——OS中称为监视器,在Java中称为锁,是一种同步机制, 
     * 保证同一时刻中,只有一个线程访问被保护的数据或代码 
  2. JVM同步基于进入和退出 
     * 使用管程对象实现,每个对象均有一个monitor对象(随着Java对象的创建和销毁) 
     * 管程对象的作用,对临界区资源进行加锁(进入时加锁,退出时解锁) 

###  用户线程与守护线程

  1. 用户线程==>自定义的线程 
  2. 守护线程==>example: 垃圾回收机制 GC 
  3. code segment 


​    
```java
    public static void main(String[] args) {  
        Thread threadName = new Thread(() -> {  
            System.out.println(Thread.currentThread().getName()   
                               + "::"+Thread.currentThread().isDaemon());  
            while (true) {  
  
            }  
        }, "threadName");  
        threadName.start();//用户线程  
  
        System.out.println(Thread.currentThread().getName()+"::"+"over");//主线程  
    }  
```


![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D4r851iahbU14ic6msC2Tygt0c2fyVPXUCu8PAQEJ9G83dlldgXQ64AprQ/640?wx_fmt=png&from=appmsg)
image.png

  * 现象说明 
    * 主线程结束了,用户线程还在运行,jvm存活 
    * 没有用户线程了,都是守护线程,jvm 结束 
    
  * 设置守护线程 

    ````java
    
            //设置守护线程  
            threadName.setDaemon(true);  
            threadName.start();  
    ````
    
    ​      


![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D4XNvnXYWvenBdCsCUDN109bYLR4oDZ5AahkP8RcSWC0SCAjKb6LXQ5w/640?wx_fmt=png&from=appmsg)

#  Lock接口

> ❝
>
> 多线程编程过程(高内聚)
>
>   * 创建资源类,设置属性与方法
>   * 在资源类中
>     * 判断
>     * 干活
>     * 通知
>   * 创建多个线程,调用资源类的操作方法
>   * 防止虚假唤醒问题
>
❞

  

##  复习Synchronized

  

###  syschronized

> ❝
>
> synchronized 是Java中的关键字,是一种同步锁 可修饰
>
>   * 代码块
>   * 方法
>
❞

  * synchronized 关键字 
    * 可修饰一个代码快,称为同步代码块,作用范围==>大括号之内,作用对象==>调用此代码块的对象 
    * 可修饰一个方法,称为同步方法,作用范围==>整个方法,作用对象==>调用这个方法的对象可修饰一个静态方法,作用范围==>整个静态方法,作用对象==>这个类的所有对象 
    * 可修饰一个类,其作用范围是synchronized后面括号之内,作用对象==>类的所有对象 

###  卖票案例

> ❝
>
> 3个售票员,卖出30张票
>
> ❞

  * 创建资源类,定义属性和操作方法 

    ````java
    public class Ticket {  
        /**  
         *  票数  
         */  
        private Integer number = 30;  
    
        /**  
         *  卖票方法 基础版本  
         *  使用同步监视器 synchronized  
         */  
        public synchronized void saleV1() {  
            //判断 是否有票  
            if (number > 0) {  
                System.out.println(Thread.currentThread().getName() +   
                                   ": 卖出 " + number-- + ",剩下" + number);  
            }  
        }  
    }  
    ````
    
    
    
  * 创建多个线程,调用资源类的操作方法 

    ````java
    
    public class SaleTicket {  
        //step2  
        public static void main(String[] args) {  
            //创建Ticket对象  
            Ticket ticket = new Ticket();  
            //创建线程  
            Thread thread1 = new Thread(new Runnable() {  
                @Override  
                public void run() {  
                    for (int i = 0; i < 40; i++) {  
                        ticket.saleV1();  
                    }  
                }  
            },"sale01");  
            Thread thread2 = new Thread(new Runnable() {  
                @Override  
                public void run() {  
                    for (int i = 0; i < 40; i++) {  
                        ticket.saleV1();  
                    }  
                }  
            },"sale02");  
            Thread thread3 = new Thread(new Runnable() {  
                @Override  
                public void run() {  
                    for (int i = 0; i < 40; i++) {  
                        ticket.saleV2();  
                    }  
                }  
            },"sale03");  
            thread1.setPriority(1);  
            thread2.setPriority(5);  
            thread1.start();  
            thread2.start();  
            thread3.start();  
        }  
    }  
    ````
    
    

  

###  多线程编程步骤

  * 创建资源类,创建属性和操作方法 
  * 创建多线程调用资源类方法 

##  什么是Lock接口

  

###  Lock接口的介绍

> ❝
>
> API->java.utils-concurrent.locks
>
>   * 为锁和等待条件提供一个框架的接口和类,不同于内置的同步和监视器
>   * Lock实现提供了比使用synchronized方法和语句可获得更广泛的锁定操作
>   * 接口 Lock所有已知实现类：
>     * ReentrantLock,
>     * ReentrantReadWriteLock.ReadLock,
>     * ReentrantReadWriteLock.WriteLock
>   * [ ] 与Synchronized关键字的区别(面试题)
>     * 竞争不激烈时,二者性能差不多
>     * 大量线程同时竞争时,Lock性能远超synchronized
>     * Lock不是Java语言内置的,是一个类,可以实现同步,synchronized是Java语言关键字
>     *
>     synchronized无需用户手动释放锁,发生异常时,synchronized会自动释放线程所占用的锁,Lock需要手动释放锁,当发生异常时若没有主动通过unLock释放锁将导致死锁现象,为避免死锁通常在finally中进行unLock
>     * Lock可以让等待锁的线程响应中断,synchronized需要一直等待,不能够响应中断
>     * 通过Lock可以知道是否成功获取锁,Synchronized无从得知
>     * Lock可以提高多个线程进行读操作的效率
>     * 从性能上讲
>
❞

  

###  Lock实现可重入锁

  

###  实现卖票案例

````java
*/  public class Ticket { 
        	//可重入锁 对象  
    	private final ReentrantLock lock = new ReentrantLock();  
            //票数  
         private Integer number = 30;  
    /**  
     *  卖票方法 手动锁管理版本  
     *  使用可重入锁 对象 lock上锁,unlock解锁  
     */  
      
    public void saleV2() {  
        lock.lock();//上锁  
        try {  
            //判断  
            if (number > 0) {  
                System.out.println(Thread.currentThread().getName() +   
                                   ": 卖出 " + number-- + ",剩下" + number);  
            }  
        } catch (Exception e) {  
            e.printStackTrace();  
        } finally {  
            lock.unlock();//解锁  
        }  
    }  
    
    }
````

#  线程间通信

  

##  案例

> ❝
>
>   * 有两个线程
>   * 资源类, 定义一个初始值为0的变量
>   * 线程1,获取值,执行+1
>   * 线程2,获取值,执行-1
>
❞


​    
​      

```java
    public class SyncThreadCommunicationShareResource {  
​        /**  
​         * 初始值  
​         */  
​        private Integer number = 0;  
​        /**  
​         * 加1的方法  
​         */  
​        public synchronized void increment() throws InterruptedException {  
​            //判断  
​            if (number != 0) {//判断number值是否是0,是0干活,不是0等待  
​                //对于某一个参数的版本,实现中断和虚假唤醒是可能的,  
​                // 此方法应存在于循环中,在非循环代码中存在虚假唤醒问题  
​                this.wait();  
​            }  
​            //干活  
​            this.number++;//number为0执行++  
​            System.out.println(Thread.currentThread().getName() + "::"+this.number);  
​            //通知  
​            this.notifyAll();  
​        }  
​        /**  
​         * 减1的方法  
​         */  
​        public synchronized void decrement() throws InterruptedException {  
​            //判断  
​            if (number != 1) {  
​                this.wait();  
​            }  
​            //干活  
​            this.number--;  
​            System.out.println(Thread.currentThread().getName() + "::"+this.number);  
​            //通知  
​            this.notifyAll();  
​        }  
​      
​    }  


​    
​    
​    public class SyncThreadCommunicationDemo {  
​        public static void main(String[] args) {  
​            SyncThreadCommunicationShareResource shareResource = new SyncThreadCommunicationShareResource();  
​            /**  
​             * 创建线程  
​             */  
​            new Thread(()->{  
​                for (int i = 0; i < 10; i++) {  
​                    try {  
​                        shareResource.increment();  
​                    } catch (InterruptedException e) {  
​                        e.printStackTrace();  
​                    }  
​                }  
​            },"increment").start();  
​      
​            new Thread(()->{  
​                for (int i = 0; i < 10; i++) {  
​                    try {  
​                        shareResource.decrement();  
​                    } catch (InterruptedException e) {  
​                        e.printStackTrace();  
​                    }  
​                }  
​            },"decrement").start();  
​            //-------------------------------------  
​            new Thread(()->{  
​                for (int i = 0; i < 10; i++) {  
​                    try {  
​                        shareResource.increment();  
​                    } catch (InterruptedException e) {  
​                        e.printStackTrace();  
​                    }  
​                }  
​            },"increment02").start();         
	new Thread(()->{  
            for (int i = 0; i < 10; i++) {  
                try {  
                    shareResource.decrement();  
                } catch (InterruptedException e) {  
                    e.printStackTrace();  
                }  
            }  
        },"decrement02").start();  
    }  
}  
```


![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D4kkPZVQaVW57yKKuoPv3SR6S68iacuDtI7agyKx2WAZIrtqMMjTSGXwQ/640?wx_fmt=png&from=appmsg)

##  虚假唤醒问题

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D49ERcYIjy57fmS9J9s3UUN8StSS1qg9JZ1ia60YQUqqY7eMntcTuYSiaw/640?wx_fmt=png&from=appmsg)  
![](https://mmbiz.qpic.cn/mmbiz_jpg/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D492fb1QfuCTjFx9jherricrRI5rUI0LHax67VEic7ZwwG0kek4kzGdXcQ/640?wx_fmt=jpeg&from=appmsg)

  * wait 在哪睡就在哪醒 
  * 解决方法将 if 判断改成 while 判断,不管 wait 什么时候醒都需要经过 while 判断 

![](https://mmbiz.qpic.cn/mmbiz_gif/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D4DKia8UMRZicjb1JxWVa1RnKEkWHx9Vcc9t57WDR6lZYVDOcchWPycNnw/640?wx_fmt=gif&from=appmsg)

##  线程间的通信基于 Lock 实现


​    
​    //创建资源类  
​    public class LockThreadCommunicationShareResource {  
​        /**  
​         * 初始值  
​         */  
​        private Integer number = 0;  
​        //创建lock  
​        private Lock lock  = new ReentrantLock();  
​        private Condition condition = lock.newCondition();  
​      
​        //执行+1  
​        public void increment(){  
​            lock.lock();  
​            try {  
​                while(number!=0){  
​                    condition.await();  
​                }  
​                number++;  
​                System.out.println(Thread.currentThread().getName() + "::"+this.number);  
​                condition.signalAll();  
​            } catch (Exception e) {  
​                e.printStackTrace();  
​            } finally {  
​                lock.unlock();  
​            }  
​        }  
​        //执行-1  
​        public void decrement(){  
​            lock.lock();  
​            try {  
​                while(number!=1){  
​                    condition.await();  
​                }  
​                number--;  
​                System.out.println(Thread.currentThread().getName() + "::"+this.number);  
​                condition.signalAll();  
​            } catch (Exception e) {  
​                e.printStackTrace();  
​            } finally {  
​                lock.unlock();  
​            }  
​        }  
​      
    }  


​    
​    
​    public static void main(String[] args) {  
​        //        demo1();  
​        LockThreadCommunicationShareResource shareResource  
​        = new LockThreadCommunicationShareResource();  
​        Thread thread0 = new Thread(() -> {  
​            for (int i = 0; i < 10; i++) {  
​                shareResource.increment();  
​            }  
​        }, "lockIncrement");  
​        Thread thread1 = new Thread(() -> {  
​            for (int i = 0; i < 10; i++) {  
​                shareResource.increment();  
​            }  
​        }, "lockIncrement01");  
​        Thread thread2 = new Thread(() -> {  
​            for (int i = 0; i < 10; i++) {  
​                shareResource.decrement();  
​            }  
​        }, "lockDecrement");  
​        Thread thread3 = new Thread(() -> {  
​            for (int i = 0; i < 10; i++) {  
​                shareResource.decrement();  
​            }  
​        }, "lockIDecrement01");  
​        thread0.start();  
​        thread1.start();  
​        thread2.start();  
​        thread3.start();  
​    }  


![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D4vZOwMeNHGibyl5GpC3ENCfibjhXSib5ZBfethlBXg22icJhnJGsgbXlic3A/640?wx_fmt=png&from=appmsg)

##  小结

  

#  线程间定制化通信

  

##  资源类


​    
​    //创建 资源类  
​    public class CustomThreadCommunityShareResource {  
​        //定义标志位  
​        private Integer flag =1; // 1 thread01 2 thread02 3 thread03  
​        //创建lock锁  
​        private  Lock lock = new ReentrantLock();  
​      
​        //创建3个condition  
​        private Condition c1 = lock.newCondition();  
​        private Condition c2 = lock.newCondition();  
​        private Condition c3 = lock.newCondition();  
​      
        //打印5次,参数为第几轮  
        public void printFive(Integer loop){  
            //上锁  
            lock.lock();  
            try {  
                //判断  
                while(flag!=1){  
                    //等待  
                    c1.await();  
                }  
                //打印5次  
                for (int i = 0; i < 5; i++) {  
                    System.out.println(Thread.currentThread().getName()  
                                       +"::"+i+" :轮数: "+loop);  
                }  
                //通知  
                flag = 2;//修改标志位  
                c2.signal();//通知thread02线程  
            } catch (Exception e) {  
                e.printStackTrace();  
            } finally {  
                //释放锁  
                lock.unlock();  
            }  
        }  
        public void printTen(Integer loop){  
            lock.lock();  
            try {  
                //判断  
                while(flag!=2){  
                    c2.await();  
                }  
                for (int i = 0; i < 10; i++) {  
                    System.out.println(Thread.currentThread().getName()  
                                       +"::"+i+" :轮数: "+loop);  
                }  
                flag = 3;  
                c3.signal();  
            } catch (Exception e) {  
                e.printStackTrace();  
            } finally {  
                lock.unlock();  
            }  
        }  
        public void printFifteen(Integer loop){  
            lock.lock();  
            try {  
                //判断  
                while(flag!=3){  
                    c3.await();  
                }  
                for (int i = 0; i < 15; i++) {  
                    System.out.println(Thread.currentThread().getName()  
                                       +"::"+i+" :轮数: "+loop);  
                }  
                flag = 1;  
                c1.signal();  
            } catch (Exception e) {  
                e.printStackTrace();  
            } finally {  
                lock.unlock();  
            }  
        }  
    }  




##  测试代码


​    
​      public static void main(String[] args) {  
​            CustomThreadCommunityShareResource shareResource = new CustomThreadCommunityShareResource();  
​            new Thread(()->{  
​                for (int i = 0; i < 10; i++) {  
​                    shareResource.printFive(i);  
​                }  
​            },"AA").start();  
​            new Thread(()->{  
​                for (int i = 0; i < 10; i++) {  
​                    shareResource.printTen(i);  
​                }  
​            },"BB").start();  
​            new Thread(()->{  
​                for (int i = 0; i < 10; i++) {  
​                    shareResource.printFifteen(i);  
​                }  
​            },"CC").start();  
​      
​        }  




##  结果

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D49erhSQ4jTsxYvBAsKLeTjyn2YEKz0STJLnsxwDzK4v2hUETtq7YM3A/640?wx_fmt=png&from=appmsg)

##  生产者与消费者模型

  

##  小结

  

#  集合的线程安全

  

##  集合线程不安全演示

   ````java
    public static void main(String[] args) {  
   ​            //创建ArrayList  
   ​            List<String> list = new ArrayList<>();  
   ​            //并发修改异常  
   ​            for (int i = 0; i < 100; i++) {  
   ​                new Thread(()->{  
   ​                    //向集合添加内容  
   ​                    list.add(UUID.randomUUID().toString().toString().substring(0,8));  
   ​                    //从集合中获取内容  
   ​                    System.out.println(list);  
   ​                },String.valueOf(i)).start();  
   ​            }  
   ​      
           }  
   ````




![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D4tGqSFz5xGSpNK5SU1Usks5ej8Pgu1TCD1MasszJELu49ypvhshoXQQ/640?wx_fmt=png&from=appmsg)

###  解决方案-Vector

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D48osicSG5mcxOm8r5h0Ca8JVzrbKs0mr8uSXXa2FliaKBibQ11X99yFOaw/640?wx_fmt=png&from=appmsg)  
![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D4Da46VreB2NRhO6BTJ01CsJ4q0H8gDrb4KbqbpeN356GrGTQfhtH6xg/640?wx_fmt=png&from=appmsg)

###  解决方案-Collections

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D4XpZgdoyANUAntHjf21wdAJttsiaSauaG914svEhdnJZ2xC9iaAj4OTmA/640?wx_fmt=png&from=appmsg)

###  解决方案-CopyOnWriteArrayList(常用 JUC)

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D47UugWg3wSoAfKwTtp4cdmZHRntiazTLdqXLCRMruMia2QqYaryK6k26g/640?wx_fmt=png&from=appmsg)  
![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D4VdulmvdbYm0JnODPsiaXxnxZDGoT4DSJnSOZr0ulCI7O9BpRyDv0qOg/640?wx_fmt=png&from=appmsg)

  

##  HashSet (无序,不可重复)线程不安全

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D47Yqzs5DS300YicoPC5ibiat9qZo7hdgeZLv6QxMFbQR3Ot6Lsmxbrza1w/640?wx_fmt=png&from=appmsg)  
![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D4oRYnCfEicM4qG509yh0VTwXztBDYtF4j2UqiatFj3QfIF8hMQBMMSPNg/640?wx_fmt=png&from=appmsg)

###  使用ConcurrentSkipListSet 进行解决

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D44PEiczdllOrUKzWiciccvvM0zGTxjCkdT2DqkvUESNiaT1lNZflCEAo1Cg/640?wx_fmt=png&from=appmsg)

##  HashMap 线程不安全

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D4Vb0aia2SHgw11gyySA1ejdic5ydRmEyD2qopnNYXpvICChYuxXt4MTXw/640?wx_fmt=png&from=appmsg)  
![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D4ic0FnqazjzMttr26ErOcsOXJ94eKa53OKiaHSYHGfibicxXDlQcAdxlYTg/640?wx_fmt=png&from=appmsg)

###  使用 ConcurrentHashMap 进行解决

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D4fSwMWXVxrlPiaCKzj2Y9SvZWk8u8GNrlWV2mJTHXGEXzZb1ObRvXxicg/640?wx_fmt=png&from=appmsg)

#  多线程锁

  

##  同步锁的8钟情况


​    
​    public class Phone {  
​        public synchronized void sendSMS() throws InterruptedException {  
​            TimeUnit.SECONDS.sleep(4);  
​            System.out.println("----------sendSMS");  
​        }  
​        public synchronized void sendEmail() throws InterruptedException {  
​            System.out.println("----------sendEmail");  
​        }  
​        public void getHello() {  
​            System.out.println("----------getHello");  
​        }  
​    }  


​    
​    
​     public static void main(String[] args) throws InterruptedException {  
​            Phone phone = new Phone();  
​    //        Phone phone2 = new Phone();  
​            new Thread(()->{  
​                try {  
​                    phone.sendSMS();  
​                } catch (InterruptedException e) {  
​                    e.printStackTrace();  
​                } finally {  
​                }  
​            },"AA").start();  
​      
​            Thread.sleep(100);  
​      
            new Thread(()->{  
                try {  
                    phone.sendEmail();  
    //                phone2.sendEmail();  
    //                phone.getHello();  
      
                } catch (InterruptedException e) {  
                    e.printStackTrace();  
                } finally {  
                }  
            },"BB").start();  
        }  


​    
​    
​    /**  
​     * 1.标准访问,先打印短信还是邮件  
​     * ----------sendSMS  
​     * ----------sendEmail  
​     * 2.停4秒在短信方法内,先打印短信还是邮件  
​     * ----------sendSMS  
​     * ----------sendEmail  
​     *  
​     * 对于1和2,synchronized锁的范围是当前对象this,执行时,其它方法只能等待  
​     * 3.新增普通的hello方法,先是打印短信还是邮件  
​     * ----------getHello 与锁无光  
​     * ----------sendSMS  
​     * 4.现在两部手机,先是打印短信还是邮件  
​     * ----------sendEmail  
​     * ----------sendSMS  
​     * 两个对象,synchronized锁的范围是各种对象,先执行Email,SMS在sleep  
​     * 5.两个静态同步方法,1部手机,先是打印短信还是邮件  
​     * ----------sendSMS  
​     * ----------sendEmail  
​     * 6.两个静态同步方法,2部手机,,先是打印短信还是邮件  
​     * ----------sendSMS  
​     * ----------sendEmail  
​     *  
​     * 对于5和6 static synchronized 锁住的是当前类的Class对象  
​     * 7.1个静态同步方法,1个普通同步方法,1部手机,先是打印短信还是邮件  
​     * ----------sendEmail  
​     * ----------sendSMS  
​     * 8.1个静态同步方法,1个普通同步方法,2部手机,先是打印短信还是邮件  
​     * ----------sendEmail  
​     * ----------sendSMS  
​     * 对于7和8是两个不同范围的不同的锁,static是大门的锁,plain是房间的锁  
​     */  




##  公平锁与非公平锁

  * 源码视图 

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D4BJwdBYPibPiaU4hfE1vH9dwvBCQpLRH5r60HKqTOicwJxmYRflzXuvfnQ/640?wx_fmt=png&from=appmsg)  
![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D42ad7XfToLUsyrml4cicxMWunGPiaBzdLMN5znVRaN2vnWVQVVKeMictrg/640?wx_fmt=png&from=appmsg)


​    
​    private final ReentrantLock lock = new ReentrantLock(true);  
​    //默认值为flase,非公平锁,将fair置为true此时lock就变成公平锁了  


  * 非公平锁 
    * 会出现线程饥饿现象 
    * 效率较高 
  * 公平锁 
    * 相对比较公平,每个线程都能分配到资源 
    * 效率先对较低 
  * 阅读源码 

    
    
    //ReentrantLock类源码  
    
    
    	/**  
    	 * Creates an instance of {@code ReentrantLock}.  
    	 * This is equivalent to using {@code ReentrantLock(false)}.  
    	 */  
    	public ReentrantLock() {  
    	    sync = new NonfairSync();  
    	}  
    	  
    	/**  
    	 * Creates an instance of {@code ReentrantLock} with the  
    	 * given fairness policy.  
    	 *  
    	 * @param fair {@code true} if this lock should use a fair ordering policy  
    	 */  
    	//传入的fair为true创建公平锁,否则使用默认无参构造  
    	public ReentrantLock(boolean fair) {  
    	    sync = fair ? new FairSync() : new NonfairSync();  
    	}  
    
    
    
    
        /**  
         * Sync object for non-fair locks  
         */  
        //非公平锁进入之后直接执行  
        static final class NonfairSync extends Sync {  
            private static final long serialVersionUID = 7316153563782823691L;  
          
            /**  
             * Performs lock.  Try immediate barge, backing up to normal  
             * acquire on failure.  
             */  
            final void lock() {  
                if (compareAndSetState(0, 1))  
                    setExclusiveOwnerThread(Thread.currentThread());  
                else  
                    acquire(1);  
            }  
          
            protected final boolean tryAcquire(int acquires) {  
                return nonfairTryAcquire(acquires);  
            }  
        }  
          
        /**  
         * Sync object for fair locks  
         */  
        //公平锁进入之后先进行进行礼貌询问  
        static final class FairSync extends Sync {  
            private static final long serialVersionUID = -3000897897090466540L;  
          
            final void lock() {  
                acquire(1);  
            }  
          
            /**  
             * Fair version of tryAcquire.  Don't grant access unless  
             * recursive call or no waiters or is first.  
             */  
            protected final boolean tryAcquire(int acquires) {  
                final Thread current = Thread.currentThread();  
                int c = getState();  
                if (c == 0) {  
                    //进行询问  
                    if (!hasQueuedPredecessors() &&  
                        compareAndSetState(0, acquires)) {  
                        //没人,执行操作  
                        setExclusiveOwnerThread(current);  
                        return true;  
                    }  
                }  
                //有人,进行排队  
                else if (current == getExclusiveOwnerThread()) {  
                    int nextc = c + acquires;  
                    if (nextc < 0)  
                        throw new Error("Maximum lock count exceeded");  
                    setState(nextc);  
                    return true;  
                }  
                return false;  
            }  
        }  
    

  

##  可重入锁

  * synchronized(隐式上锁) 和 Lock(显示上锁) 都是可重入锁 
  * 什么是可重入锁? 

    
    
    public class ReentrantLockDemo {  
        public synchronized void add(){  
            add();  
        }  
        //synchronized  
        public static void main(String[] args) {  
            new ReentrantLockDemo().add();  
        }  
    }  
    

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D49nNia1fkecXf7NictwChINlyhOMTz09Zz3fVLoPCEMiaib8QbuWW1zRgHg/640?wx_fmt=png&from=appmsg)
image.png

  * 现象解释 
    * 由于 synchronized 是一个可重入锁,因此调用 add 之后,任然可以递归调用 
    * 加入 synchronized 不是可重入锁则,add 调用之后其它方法进入等待,则无法出现递归栈溢出异常 
  * condiction.signal 的描述 

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D4GxyHRDED5s5k9acibzcM7U0PtS4SjRz1WADqr4sMDkFdiczf4JGKJxkw/640?wx_fmt=png&from=appmsg)

##  死锁

> ❝
>
>
> 死锁（Deadlock）是指在多线程或多进程的系统中，两个或多个进程或线程由于相互等待对方释放资源而无法继续执行的一种阻塞现象。在死锁状态下，每个进程或线程都在等待其他进程或线程释放资源，导致所有参与的进程或线程都无法继续执行下去。
>
> ❞

  * 死锁通常发生在多个进程或线程之间共享多个资源的情况下，它是一种常见但严重的并发问题。 
  * 死锁的发生需要满足四个必要条件，也称为死锁的四个必要条件： 

> ❝
>
>   * 互斥条件（Mutual Exclusion）：
>     * 资源不能被共享，一次只能被一个进程或线程占用。
>   * 占有并等待条件（Hold and Wait）：
>     * 一个进程或线程可以持有一个资源，并等待另一个资源。
>   * 不可剥夺条件（No Preemption）：
>     * 资源只能在持有者释放资源后才能被其他进程或线程抢占。
>   * 循环等待条件（Circular Wait）：
>     * 存在一个进程或线程的资源等待链，形成一个循环。
>
❞

当以上四个条件同时满足时，就有可能导致死锁的发生。死锁是一种难以调试和解决的问题，因此在设计并发程序时需要特别小心，避免引入可能导致死锁的条件。

![](https://mmbiz.qpic.cn/mmbiz_jpg/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D4gDzd4zzRJHNy3VGOibIDMNRGiaiare86KEBicLTB3ZOKpcCqnYVxlNZhZA/640?wx_fmt=jpeg&from=appmsg)

> ❝
>
>
> 解决死锁的方法包括避免死锁、检测死锁和解除死锁。避免死锁的方法通常涉及对资源的动态分配和合理使用，以确保死锁的四个必要条件不同时满足。检测死锁通常涉及周期性地检查系统中是否存在死锁，一旦检测到死锁，可以采取相应的措施解除死锁。解除死锁的方法包括撤销进程、回滚事务或通过抢占资源等。
>
> ❞

  

###  演示


​    
​    public class DealLock {  
​        static Object a = new Object();  
​        static Object b = new Object();  
​      
​        public static void main(String[] args) {  
​            new Thread(() -> {  
​                synchronized (a) {  
​                    System.out.println(Thread.currentThread().getName() + "持有锁a试图获取锁b");  
​                    try {  
​                        TimeUnit.SECONDS.sleep(1);  
​                    } catch (InterruptedException e) {  
​                        e.printStackTrace();  
​                    }  
​                    synchronized (b) {  
​                        System.out.println(Thread.currentThread().getName() + "获取到锁b");  
​                    }  
​                }  
​            }, "A").start();  
​      
            new Thread(() -> {  
                synchronized (b) {  
                    System.out.println(Thread.currentThread().getName() + "持有锁b试图获取锁a");  
                    try {  
                        TimeUnit.SECONDS.sleep(1);  
                    } catch (InterruptedException e) {  
                        e.printStackTrace();  
                    }  
                    synchronized (a) {  
                        System.out.println(Thread.currentThread().getName() + "获取到锁a");  
                    }  
                }  
            }, "B").start();  
        }  
    }  


​    

  * 现象 

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sdOqy4Sj4LlMtQ2UP9XA9D44ayaYua2HVtcgg8RyibGDFkchic5mU7lIoAFFnIHZaKzakAq9bI5tWVQ/640?wx_fmt=png&from=appmsg)
image.png

  * 验证是否是死锁 
    * jps+jstack 

    
    
    PS D:\Desktop\JUC_workspace> jps -l  
    12084 top.ljzstudy.lock.DealLock  
    18260  
    20716 org.jetbrains.jps.cmdline.Launcher  
    5996 jdk.jcmd/sun.tools.jps.Jps  
    PS D:\Desktop\JUC_workspace> jstack 12084  
    2024-01-27 14:29:17  
    Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.131-b11 mixed mode):  
    
    "DestroyJavaVM" #14 prio=5 os_prio=0 tid=0x0000000002f53800 nid=0x5c2c waiting on condition [0x0000000000000000]  
       java.lang.Thread.State: RUNNABLE  
    
    "B" #13 prio=5 os_prio=0 tid=0x0000000024b73000 nid=0x5718 waiting for monitor entry [0x00000000254cf000]  
       java.lang.Thread.State: BLOCKED (on object monitor)  
            at top.ljzstudy.lock.DealLock.lambda$main$1(DealLock.java:43)  
            - waiting to lock <0x0000000740da2ee0> (a java.lang.Object)  
            - locked <0x0000000740da2ef0> (a java.lang.Object)  
            at top.ljzstudy.lock.DealLock$$Lambda$2/1831932724.run(Unknown Source)  
            at java.lang.Thread.run(Thread.java:748)  
    
    "A" #12 prio=5 os_prio=0 tid=0x0000000024b16800 nid=0x5e9c waiting for monitor entry [0x00000000253cf000]  
       java.lang.Thread.State: BLOCKED (on object monitor)  
            at top.ljzstudy.lock.DealLock.lambda$main$0(DealLock.java:29)  
            - waiting to lock <0x0000000740da2ef0> (a java.lang.Object)  
            - locked <0x0000000740da2ee0> (a java.lang.Object)  
            at top.ljzstudy.lock.DealLock$$Lambda$1/990368553.run(Unknown Source)  
            at java.lang.Thread.run(Thread.java:748)  
    
    "Service Thread" #11 daemon prio=9 os_prio=0 tid=0x0000000022f73800 nid=0x2e44 runnable [0x0000000000000000]  
       java.lang.Thread.State: RUNNABLE  
    
    "C1 CompilerThread3" #10 daemon prio=9 os_prio=2 tid=0x0000000022ece800 nid=0x4810 waiting on condition [0x0000000000000000]  
       java.lang.Thread.State: RUNNABLE  
    
    "C2 CompilerThread2" #9 daemon prio=9 os_prio=2 tid=0x0000000022ece000 nid=0x38d0 waiting on condition [0x0000000000000000]  
       java.lang.Thread.State: RUNNABLE  
    
    "C2 CompilerThread1" #8 daemon prio=9 os_prio=2 tid=0x0000000022eca000 nid=0x58e8 waiting on condition [0x0000000000000000]  
       java.lang.Thread.State: RUNNABLE  
    
    "C2 CompilerThread0" #7 daemon prio=9 os_prio=2 tid=0x0000000022ec5000 nid=0x1060 waiting on condition [0x0000000000000000]  
       java.lang.Thread.State: RUNNABLE  
    
    "Monitor Ctrl-Break" #6 daemon prio=5 os_prio=0 tid=0x0000000022eb8000 nid=0x2f2c runnable [0x00000000244ce000]  
       java.lang.Thread.State: RUNNABLE  
            at java.net.SocketInputStream.socketRead0(Native Method)  
            at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)  
            at java.net.SocketInputStream.read(SocketInputStream.java:171)  
            at java.net.SocketInputStream.read(SocketInputStream.java:141)  
            at sun.nio.cs.StreamDecoder.readBytes(StreamDecoder.java:284)  
            at sun.nio.cs.StreamDecoder.implRead(StreamDecoder.java:326)  
            at sun.nio.cs.StreamDecoder.read(StreamDecoder.java:178)  
            - locked <0x0000000740e2b040> (a java.io.InputStreamReader)  
            at java.io.InputStreamReader.read(InputStreamReader.java:184)  
            at java.io.BufferedReader.fill(BufferedReader.java:161)  
            at java.io.BufferedReader.readLine(BufferedReader.java:324)  
            - locked <0x0000000740e2b040> (a java.io.InputStreamReader)  
            at java.io.BufferedReader.readLine(BufferedReader.java:389)  
            at com.intellij.rt.execution.application.AppMainV2$1.run(AppMainV2.java:49)  
    
    "Attach Listener" #5 daemon prio=5 os_prio=2 tid=0x0000000022e20000 nid=0x5598 waiting on condition [0x0000000000000000]  
       java.lang.Thread.State: RUNNABLE  
    
    "Signal Dispatcher" #4 daemon prio=9 os_prio=2 tid=0x0000000022e1f000 nid=0x166c runnable [0x0000000000000000]  
       java.lang.Thread.State: RUNNABLE  
    
    "Finalizer" #3 daemon prio=8 os_prio=1 tid=0x000000000304c000 nid=0x1d9c in Object.wait() [0x000000002415f000]  
       java.lang.Thread.State: WAITING (on object monitor)  
            at java.lang.Object.wait(Native Method)  
            - waiting on <0x0000000740c08ec8> (a java.lang.ref.ReferenceQueue$Lock)  
            at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:143)  
            - locked <0x0000000740c08ec8> (a java.lang.ref.ReferenceQueue$Lock)  
            at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:164)  
            at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:209)  
    
    "Reference Handler" #2 daemon prio=10 os_prio=2 tid=0x000000000304b000 nid=0x20e0 in Object.wait() [0x000000002405e000]  
       java.lang.Thread.State: WAITING (on object monitor)  
            at java.lang.Object.wait(Native Method)  
            - waiting on <0x0000000740c06b68> (a java.lang.ref.Reference$Lock)  
            at java.lang.Object.wait(Object.java:502)  
            at java.lang.ref.Reference.tryHandlePending(Reference.java:191)  
            - locked <0x0000000740c06b68> (a java.lang.ref.Reference$Lock)  
            at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:153)  
    
    "VM Thread" os_prio=2 tid=0x0000000021737000 nid=0x5efc runnable  
    
    "GC task thread#0 (ParallelGC)" os_prio=0 tid=0x0000000002f69000 nid=0x2240 runnable  
    
    "GC task thread#1 (ParallelGC)" os_prio=0 tid=0x0000000002f6a800 nid=0x4990 runnable  
    
    "GC task thread#2 (ParallelGC)" os_prio=0 tid=0x0000000002f6c000 nid=0x50b0 runnable  
    
    "GC task thread#3 (ParallelGC)" os_prio=0 tid=0x0000000002f6e800 nid=0x37cc runnable  
    
    "GC task thread#4 (ParallelGC)" os_prio=0 tid=0x0000000002f70800 nid=0x2d70 runnable  
    
    "GC task thread#5 (ParallelGC)" os_prio=0 tid=0x0000000002f72000 nid=0x51c0 runnable  
    
    "GC task thread#6 (ParallelGC)" os_prio=0 tid=0x0000000002f75000 nid=0x13d4 runnable  
    
    "GC task thread#7 (ParallelGC)" os_prio=0 tid=0x0000000002f76000 nid=0x4e9c runnable  
    
    "GC task thread#8 (ParallelGC)" os_prio=0 tid=0x0000000002f77800 nid=0x5ab4 runnable  
    
    "GC task thread#9 (ParallelGC)" os_prio=0 tid=0x0000000002f78800 nid=0x5760 runnable  
    
    "VM Periodic Task Thread" os_prio=2 tid=0x0000000022fd8000 nid=0x694 waiting on condition  
    
    JNI global references: 319  
    
    
    Found one Java-level deadlock:  
    =============================  
    "B":  
      waiting to lock monitor 0x0000000021740be8 (object 0x0000000740da2ee0, a java.lang.Object),  
      which is held by "A"  
    "A":  
      waiting to lock monitor 0x0000000021743528 (object 0x0000000740da2ef0, a java.lang.Object),  
      which is held by "B"  
    
    Java stack information for the threads listed above:  
    ===================================================  
    "B":  
            at top.ljzstudy.lock.DealLock.lambda$main$1(DealLock.java:43)  
            - waiting to lock <0x0000000740da2ee0> (a java.lang.Object)  
            - locked <0x0000000740da2ef0> (a java.lang.Object)  
            at top.ljzstudy.lock.DealLock$$Lambda$2/1831932724.run(Unknown Source)  
            at java.lang.Thread.run(Thread.java:748)  
    "A":  
            at top.ljzstudy.lock.DealLock.lambda$main$0(DealLock.java:29)  
            - waiting to lock <0x0000000740da2ef0> (a java.lang.Object)  
            - locked <0x0000000740da2ee0> (a java.lang.Object)  
            at top.ljzstudy.lock.DealLock$$Lambda$1/990368553.run(Unknown Source)  
            at java.lang.Thread.run(Thread.java:748)  
    
    Found 1 deadlock.  
    
    

  

* * *

预览时标签不可点

[ 阅读原文 ](javascript:;)

微信扫一扫  
关注该公众号





****



****



×  分析

  收藏

