![cover_image](https://mmbiz.qpic.cn/mmbiz_jpg/hyAMveHd7seichdUdcrSXK0rku2gb4Ekib0xUVNaG6CBTQvFcZib10rE3prT5jyOIvCoSicalxDTgBLDzmic9Tf0ia6g/0?wx_fmt=jpeg)

#  万字长文扫盲JUC基础(下)  

#  Callable接口

  

##  与 Runable 接口的比较

  * Runable 接口 
    * 线程执行使用 run()方法没有返回值 
    * run()方法 只能捕获并处理异常 
  * Callable 接口 
    * 线程执行使用 call()方法有返回值 
    * call()方法可以抛出异常 

##  使用 FutureTask 实现 Callable 接口


​    
    class MyThread1 implements Runnable{  
        @Override  
        public void run() {  
      
        }  
    }  
      
    class MyThread2 implements Callable{  
      
        @Override  
        public Integer call() throws Exception {  
            System.out.println(Thread.currentThread().getName()+"come in callable");  
            return 200;  
        }  
    }  
    public class InterfaceComparison {  
      
        public static void main(String[] args) throws ExecutionException, InterruptedException {  
            //使用Runable创建一个线程  
            new Thread(new MyThread1(),"runable").start();  
            //使用Callable创建一个线程  
            //new Thread(new MyThread2(),"callable").start();  
      
            //futureTask  
            FutureTask<Integer> futureTask1 = new FutureTask<>(new MyThread2());  
            //lamada表达式  
            FutureTask<Integer> futureTask = new FutureTask<>(()->{  
                System.out.println(Thread.currentThread().getName()+"come in callable");  
                return 1024;  
            });  
            new Thread(futureTask,"future").start();  
            while(!futureTask.isDone()){  
                System.out.println("wait....");  
            }  
            System.out.println(futureTask.get());//get方法获取返回值  
            System.out.println(Thread.currentThread().getName()+"come over");  


  * 结果 

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLoB3zYl71jjLlZEuOdBu733CPFBv0FBQ8gF5OsuxjIntpXNq3FBvbPw/640?wx_fmt=png&from=appmsg)
image.png

  * 连续调用两次 get 

    
    
    System.out.println(futureTask.get());  
    System.out.println(futureTask.get());  
    

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLngl0DTYKG8ia2panrKggUztb08Pok9U6nHeq1j8CwbK5dxjXPQjEOBQ/640?wx_fmt=png&from=appmsg)
image.png

  * 开启两个线程 

    
    
    new Thread(futureTask,"future").start();  
    new Thread(futureTask1,"future1").start();  
    
    System.out.println(futureTask.get());  
    System.out.println(futureTask1.get());  
    System.out.println(Thread.currentThread().getName()+"come over");  
    

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLLMc7phwQy4B4AQLeNxmQpCx6vBvKKGChsTlzBUQPurN0lico7xdiagsA/640?wx_fmt=png&from=appmsg)

#  JUC辅助类

  

##  减少计数CountDownLatch

  

###  常规写法,有问题


​    
     //6个同学陆续离开教室之后,班长锁门  
        public static void main(String[] args) {  
            //6个同学离开教室之后  
            for (int i = 1; i <= 6; i++) {  
                new Thread(()->{  
                    System.out.println(Thread.currentThread().getName()+" 号同学离开了教室");  
                },String.valueOf(i)).start();  
            }  
            System.out.println(Thread.currentThread().getName()+"班长锁门走人了");  
        }  


![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLfgkNr2icrONuIfwfU6vLVSRChL6OLTicJ8F5ic2yojM1n9m2CMCuWIR7w/640?wx_fmt=png&from=appmsg)
image.png

  * 有的同学被锁在教室 

###  引入 CountDownLatch


​    
    //6个同学陆续离开教室之后,班长锁门  
    public static void main(String[] args) throws InterruptedException {  
        //6个同学离开教室之后  
        CountDownLatch countDownLatch = new CountDownLatch(6);  
        for (int i = 1; i <= 6; i++) {  
            new Thread(()->{  
                System.out.println(Thread.currentThread().getName()+" 号同学离开了教室");  
                countDownLatch.countDown();//每次让计数器-1  
            },String.valueOf(i)).start();  
        }  
        //countDown不为0,阻塞  
        countDownLatch.await();  
        System.out.println(Thread.currentThread().getName()+"班长锁门走人了");  
    }  


  * 效果 

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLsdNicHVZx8uboGZ8DyVH9TxiaFz6122TPaNYpOibCXyRfb8dnj9PwI5JA/640?wx_fmt=png&from=appmsg)

##  循环栅栏CyclicBarrier

  * 代码 

    
    
       //案例:集齐7科龙珠可以转换神龙  
        //1.创建固定值  
        private static final Integer NUMBER = 7;  
        public static void main(String[] args) {  
            //2.创建CyclicBarrier  
            CyclicBarrier cyclicBarrier = new CyclicBarrier(NUMBER, () -> {  
                System.out.println("集齐7科龙珠可以转换神龙");  
            });  
            //3.收集龙珠  
            for (int i = 1; i <= 6; i++) {  
                //7个线程  
                new Thread(()->{  
                    System.out.println(Thread.currentThread().getName()+" 星龙被收集到了");  
                    try {  
                        cyclicBarrier.await();  
                    } catch (Exception e) {  
                        e.printStackTrace();  
                    }  
                },String.valueOf(i)).start();  
            }  
        }  


  * 效果 

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLBiaibAeJxgpJskaZoyia1z1cqAP2mrYuhvfdlJhnCvDj3IpyUibxFiboXXA/640?wx_fmt=png&from=appmsg)
image.png

  * 将循环次数修改为 6 

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLgjVZFOQuvY5zgUGVv6n9psf9tPsMhVQA9e41KFTsnFNq3PGLRjkDSw/640?wx_fmt=png&from=appmsg)
image.png

  * 未到达屏障点,一直处于 await()状态 

##  信号灯Semphore


​    
     //6辆汽车,停3个停车位  
        public static void main(String[] args) {  
            //创建Semphore,设置许可数量  
            Semaphore semaphore = new Semaphore(3);  
      
            //模拟6辆汽车  
            for (int i = 1; i <= 6; i++) {  
                new Thread(()->{  
                    try {  
                        //抢占车位  
                        semaphore.acquire();  
                        System.out.println(Thread.currentThread().getName()+" 抢到了车位");  
                        TimeUnit.SECONDS.sleep(new Random().nextInt(5));  
                        //设置随机停车时间  
                        System.out.println(Thread.currentThread().getName()+" ----离开了车位");  
                    } catch (InterruptedException e) {  
                        e.printStackTrace();  
                    }finally {  
                        //释放车位  
                        semaphore.release();  
                    }  
                },String.valueOf(i)).start();  
            }  
        }  


  * 效果 

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLAw8lRIicqNVTCPIyRGncnJvNGKR0icUAEZgPmCYic2PDH7njMUCp3ErkQ/640?wx_fmt=png&from=appmsg)

#  ReentrantReadWriteLock(读写锁)

  

##  读写锁概述

  

###  乐观锁与悲观锁

> ★
>
> 乐观锁和悲观锁是两种不同的并发控制策略，用于处理多个线程对共享资源的访问
>
> ”

  

###  悲观锁（Pessimistic Locking）：

  * 思想： 
    * 悲观锁的思想是，在整个访问期间，认为数据会被其他线程修改，因此在访问数据之前，会先对数据进行加锁，确保其他线程无法修改。 
  * 实现方式： 
    * 在数据库中，悲观锁通常通过数据库的锁机制（如行级锁、表级锁）来实现。 
    * 在Java编程中，synchronized 关键字和 ReentrantLock 类都是悲观锁的实现。 
  * 优点和缺点： 
    * 优点：简单，易于理解和实现。 
    * 缺点：性能相对较低，因为在整个访问期间，其他线程可能被阻塞。 

###  乐观锁（Optimistic Locking）：

  * 思想 
    * 乐观锁的思想是，在整个访问期间，认为数据不会被其他线程修改。因此，不对数据进行加锁，而是在更新数据时检查是否有其他线程对数据进行了修改。 
  * 实现方式： 
    * 在数据库中，乐观锁通常通过版本号（Version Number）或时间戳(Timestamp）等机制实现。 
    * 在 Java 编程中，乐观锁常用于基于版本的控制，例如在使用 Hibernate 进行数据库访问时，可以通过 @V  ‍  ersion 注解实现乐观锁。 
  * 优点和缺点： 
    * 优点：性能相对较高，因为在大多数情况下，不需要加锁，只有在更新时才进行检查。 
    * 缺点：实现相对复杂，需要解决冲突和处理失败更新的情况。 

###  使用场景

  * 悲观锁适用于： 
    * 数据更新频繁，冲突概率较高的情况。 
    * 读操作相对较少，写操作较多的情况。 
  * 乐观锁适用于： 
    * 数据更新相对较少，读操作频繁的情况。 
    * 冲突概率较低，因为大多数情况下不需要加锁。 
  * 在实际应用中，选择悲观锁还是乐观锁取决于具体业务场景和性能需求。在分布式环境中，乐观锁通常更受欢迎，因为它对系统的可伸缩性更友好。 

###  原理图

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLRP3ibTpOqc9wevdXfAV3OkaZmu8KZmLqibYDRRicFhnW93hFbOEibYibMKQ/640?wx_fmt=png&from=appmsg)

###  表锁与行锁

> ★
>
>   * 行锁的死锁概率通常比表锁高。
>     *
>     原因是行锁是更细粒度的锁，它锁定的是数据表中的某一行，而表锁则锁定整个数据表。因此，当多个事务尝试访问同一行数据时，如果使用行锁，可能会出现一个事务在等待另一个事务释放锁的情况，这就可能导致死锁。
>     *
>     相比之下，表锁锁定整个表，只要有一个事务在使用表，其他事务就必须等待，这就减少了死锁的可能性。但是，表锁的缺点是它会阻止其他所有事务访问整个表，这可能会降低并发性能。
>     *
>     然而，这并不是说行锁就一定比表锁差，事实上，行锁由于其更细的粒度，通常可以提供更高的并发性能。只是在处理死锁问题时，需要更小心地设计事务，以避免出现死锁。
>
”

  

##  读写锁案例

> ★
>
> 读锁是共享锁,写锁是独占锁,两者都会发生死锁
>
> ”

  * 读锁发生死锁的案例 

![](https://mmbiz.qpic.cn/mmbiz_jpg/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLhnLwTxP3lmJ608e3me1ibNkN8LxHibIsoIUzczE3Y3G8qibeXLnNdalEg/640?wx_fmt=jpeg&from=appmsg)

  * 写锁发生死锁的案例 

![](https://mmbiz.qpic.cn/mmbiz_jpg/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLZAt1FGbMFJmq57Dgiavb9jOa1Da9WicnGhichaoRlwuG9b0U9S0PZSPYw/640?wx_fmt=jpeg&from=appmsg)

##  读写锁发生死锁的场景

  

###  编写资源类


​    
    public class MyCache {  
        //创建map集合  
        private Map<String,Object> map = new HashMap<>();  
      
        //放数据  
        public void put(String key,Object value){  
            System.out.println(Thread.currentThread().getName()+" 正在进行写操作 "+ key);  
            //暂停一会  
            try {  
                TimeUnit.MICROSECONDS.sleep(300);  
            } catch (InterruptedException e) {  
                e.printStackTrace();  
            }  
            //放数据  
            map.put(key,value);  
            System.out.println(Thread.currentThread().getName()+" 写完了 " + key);  
        }  
        //取数据  
        public Object get(String key){  
            Object result = null;  
            System.out.println(Thread.currentThread().getName()+" 正在进行读取操作 "+ key);  
            //暂停一会  
            try {  
                TimeUnit.MICROSECONDS.sleep(300);  
            } catch (InterruptedException e) {  
                e.printStackTrace();  
            }  
            //放数据  
            result = map.get(key);  
            System.out.println(result!=null?  
                    Thread.currentThread().getName()+" 取完了------- " + key  
                    :Thread.currentThread().getName()+" 没取到--------- " + key);  
            return result;  
        }  
    }  




###  编写多线程测试代码


​    
      public static void main(String[] args) {  
            MyCache myCache = new MyCache();  
            //创建多线程放数据  
            for (int i = 1; i <=5 ; i++) {  
                final Integer num = i;  
                new Thread(()->{  
                    myCache.put(String.valueOf(num),String.valueOf(num));  
                },String.valueOf(i)).start();  
            }  
            //创建多线程取数据  
            for (int i = 1; i <=5 ; i++) {  
                final Integer num = i;  
                new Thread(()->{  
                    myCache.get(String.valueOf(num));  
                },String.valueOf(i)).start();  
            }  
        }  




###  结果

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLQRcqfbicZjkx9hql50uzsA0zND2ku80olWK9JRejajFKM1F7FCriawhA/640?wx_fmt=png&from=appmsg)

##  JUC 解决读写锁死锁的发生 ReadWriteLock 类

  

###  修改资源类


​    
    public class MyCache {  
        //创建map集合  
        private Map<String,Object> map = new HashMap<>();  
      
        //创建读写锁对象  
        private ReadWriteLock readWriteLock = new ReentrantReadWriteLock();  
        //放数据  
        public void put(String key,Object value){  
            //写操作,加写锁  
            readWriteLock.writeLock().lock();  
            System.out.println(Thread.currentThread().getName()+" 正在进行写操作 "+ key);  
            //暂停一会  
            try {  
                TimeUnit.MICROSECONDS.sleep(300);  
                //放数据  
                map.put(key,value);  
                System.out.println(Thread.currentThread().getName()+" 写完了 " + key);  
            } catch (InterruptedException e) {  
                e.printStackTrace();  
            }finally {  
                readWriteLock.writeLock().unlock();  
            }  
        }  
        //取数据  
        public Object get(String key){  
            //读取数据,加读锁  
            readWriteLock.readLock().lock();  
            Object result = null;  
            try {  
                System.out.println(Thread.currentThread().getName()+" 正在进行读取操作 "+ key);  
                //暂停一会  
                TimeUnit.MICROSECONDS.sleep(300);  
                //放数据  
                result = map.get(key);  
                System.out.println(result!=null?  
                        Thread.currentThread().getName()+" 取完了------- " + key  
                        :Thread.currentThread().getName()+" 没取到--------- " + key);  
            } catch (InterruptedException e) {  
                e.printStackTrace();  
            }finally {  
                //释放读锁  
                readWriteLock.readLock().unlock();  
            }  
            return result;  
        }  
    }  




###  运行测试看结果

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLnhJr0tmq0bFAiaQh2AiaOVib6v0fWFcv4e48ddk7eS6GiaIRxA0jOOr4HQ/640?wx_fmt=png&from=appmsg)

##  读写锁的演变

> ★
>
> 一个资源可以被多个读线程访问,或者可以被一个写线程访问,但是不能够同时存在读写线程 读写互斥,读读共享
>
> ”

![](https://mmbiz.qpic.cn/mmbiz_jpg/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLLI9xvsC1kPaeme1jSDYtAbibOsaAibXcjmfoc5X3qRjXicx289xTyY5cg/640?wx_fmt=jpeg&from=appmsg)

##  锁降级

> ★
>
> 将写入锁将为读锁
>
> ”

  * jdk8 的降级过程 
    * 获取写锁->获取读锁->释放写锁.....->释放读锁 
  * 读锁不能升级为写锁 
  * 读写锁降级演示 

    
    
     //写锁降级为读锁  
        public static void main(String[] args) {  
            //可重入读写锁对象  
            ReentrantReadWriteLock reentrantReadWriteLock = new ReentrantReadWriteLock();  
            //创建读写锁  
            ReentrantReadWriteLock.ReadLock readLock = reentrantReadWriteLock.readLock();  
            ReentrantReadWriteLock.WriteLock writeLock = reentrantReadWriteLock.writeLock();  
            //演示锁降级  
            writeLock.lock(); //获取写锁  
            System.out.println("get write lock");  
            readLock.lock(); //获取读锁  
            System.out.println("get read lock");  
            writeLock.unlock(); //释放写锁  
            readLock.unlock(); //释放读锁  
        }  


![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLMTx5dBuBY5qcDYsTSUkESbbED1dtZmxSOpyNxWkjNbvAODibLWMzwjA/640?wx_fmt=png&from=appmsg)
image.png

  * 读锁升级写锁,出现死锁 

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyL7YmkPgtAbxCmCoBdBvmrtDtiazfhdQPy0picmibuibTXeRFXdS5eCgxICw/640?wx_fmt=png&from=appmsg)  
![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLbpBRNibPUllFlqAjExPicmjAqOBOuajbF35vt0GZHibJQjuaINNDKJ7OQ/640?wx_fmt=png&from=appmsg)

  * 进入无限等待...... 

#  BlockingQueue阻塞队列

  

##  常见的 BlockingQueue

  * ArrayBlockingQueue(常用) 
    * 由数组结构组成的有界阻塞队列。 
  * LinkedBlockingQueue(常用) 
    * 由链表结构组成的有界（但大小默认值为integer.MAX_VALUE）阻塞队列。 
  * ArrayBlockingQueue和LinkedBlockingQueue是两个最普通也是最常用的阻塞队列，一般情况下，在处理多线程间的生产者消费者问题，使用这两个类足以。 
  * DelayQueue 
    * 使用优先级队列实现的延迟无界阻塞队列。 
  * PriorityBlockingQueue 
    * 支持优先级排序的无界阻塞队列。 
  * SynchronousQueue 
    * 不存储元素的阻塞队列，也即单个元素的队列。 
  * LinkedTransferQueue 
    * 由链表组成的无界阻塞队列。 
  * LinkedBlockingDeque 
    * 由链表组成的双向阻塞队列 

##  小结

> ★
>
>   1. 在多线程领域：所谓阻塞，在某些情况下会挂起线程（即阻塞），一旦条件满足，被挂起的线程又会自动被唤起
>   2. 为什么需要BlockingQueue?
>
”

在concurrent包发布以前，在多线程环境下，我们每个程序员都必须去自己控制这些细节，尤其还要兼顾效率和线程安全，而这会给我们的程序带来不小的复杂度。使用后我们不需要关心什么时候需要阻塞线程，什么时候需要唤醒线程，因为这一切BlockingQueue都给你一手包办了

  

##  核心方法

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLE1Rj4ibTAxYgBQ8pEAn8y13ndahvyKV9avpbAWr7tCMk8nOOMicJMVrg/640?wx_fmt=png&from=appmsg)  
![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLSRib9a5S2b1yv3HAqibV0waIFBRCARicKeBjxib0M7GsxExSPp88M4bIWw/640?wx_fmt=png&from=appmsg)

##  核心方法演示


​    
        public static void main(String[] args) throws InterruptedException {  
            //创建阻塞对列  
            BlockingQueue<String> blockingQueue = new ArrayBlockingQueue(3);  
            //group add()  
    //        System.out.println(blockingQueue.add("a"));  
    //        System.out.println(blockingQueue.add("b"));  
    //        System.out.println(blockingQueue.add("c"));  
    ////        System.out.println(blockingQueue.element());  
    ////        System.out.println(blockingQueue.add("d"));  
    //        System.out.println(blockingQueue.remove());  
    //        System.out.println(blockingQueue.remove());  
    //        System.out.println(blockingQueue.remove());  
    //        System.out.println(blockingQueue.remove());  
      
            //group  offer(e)  
    //        System.out.println(blockingQueue.offer("a"));  
    //        System.out.println(blockingQueue.offer("b"));  
    //        System.out.println(blockingQueue.offer("c"));  
    ////        System.out.println(blockingQueue.offer("d"));  
    //        System.out.println(blockingQueue.poll());  
    //        System.out.println(blockingQueue.poll());  
    //        System.out.println(blockingQueue.peek());  
    //        System.out.println(blockingQueue.poll());  
      
            //group put(e)  
    //        blockingQueue.put("a");  
    //        System.out.println(blockingQueue.take());  
    //        blockingQueue.put("b");  
    //        blockingQueue.put("c");  
    //        System.out.println(blockingQueue.take());  
    //        blockingQueue.put("d");  
    //        System.out.println(blockingQueue.take());  
    //        System.out.println(blockingQueue.take());  
      
            //group offer-time_out  
            System.out.println(blockingQueue.offer("a"));  
            System.out.println(blockingQueue.offer("b"));  
            System.out.println(blockingQueue.offer("c"));  
            System.out.println(blockingQueue.offer("d",3L, TimeUnit.SECONDS));  
        }  




#  ThreadPool线程池

  

##  线程池使用方式

  * 一池 N 线程 
    * ` Executors.newFixedThreadPool(int) `

    
    
     public static void main(String[] args) {  
            //一池n线程  
            //场景描述:银行5个窗口,处理10个顾客的业务  
            ExecutorService threadPool = Executors.newFixedThreadPool(5);  
            try {  
                for (int i = 1; i <= 10; i++) {  
                    //执行  
                    threadPool.execute(()->{  
                        System.out.println(Thread.currentThread().getName()+" 办理业务");  
                    });  
                }  
            } catch (Exception e) {  
                e.printStackTrace();  
            } finally {  
                //关闭资源  
                threadPool.shutdown();  
            }  
        }  
    

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLBT7xkAXvSsOZ6ZhZUD18Lox8BJfRN0HmFLtdvTknPK3SgHCzSebYhQ/640?wx_fmt=png&from=appmsg)
image.png

  * 一个任务一个任务执行,一池一线程 
    * ` Executors.newSingleThreadExecutor() `

    
    
    ExecutorService threadPool = Executors.newSingleThreadExecutor();  
    

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyL0kT6nr1B1hATib38JnDvavLicZguJ5QZZ4G9j7yyonsXiaibayAfVZCUZQ/640?wx_fmt=png&from=appmsg)
image.png

  * 线程池根据需求创建线程,可扩容 
    * ` Executors.newCachedThreadPool() `

    
    
    ExecutorService threadPool = Executors.newCachedThreadPool();  
    

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLDgyiaJM3rYNnSu8f2NibV0IFouPnDNmFncVno6c6YNlaZOxjQlUuZ41g/640?wx_fmt=png&from=appmsg)

##  线程池的七个参数


​    
        public ThreadPoolExecutor(int corePoolSize, 		//常驻线程线程数量(核心)  
                                  int maximumPoolSize,		//最大线程数量  
                                  long keepAliveTime,		//存活时间  
                                  TimeUnit unit,			//存活时间单位  
                                  BlockingQueue<Runnable> workQueue,	//阻塞队列  
                                  ThreadFactory threadFactory,			//线程工厂  
                                  RejectedExecutionHandler handler		//拒绝策略  
                                 ) {	  
            if (corePoolSize < 0 ||  
                maximumPoolSize <= 0 ||  
                maximumPoolSize < corePoolSize ||  
                keepAliveTime < 0)  
                throw new IllegalArgumentException();  
            if (workQueue == null || threadFactory == null || handler == null)  
                throw new NullPointerException();  
            this.corePoolSize = corePoolSize;  
            this.maximumPoolSize = maximumPoolSize;  
            this.workQueue = workQueue;  
            this.keepAliveTime = unit.toNanos(keepAliveTime);  
            this.threadFactory = threadFactory;  
            this.handler = handler;  
        }  




##  线程池底层工作流程与拒绝策略

![](https://mmbiz.qpic.cn/mmbiz_jpg/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLM6c9cmicEwvfCkjMpiapiaX1kk7WVW8RpG9Bib2SBqxopmbRxzD1BjEHicg/640?wx_fmt=jpeg&from=appmsg)

  * 当线程池对象执行 execute()方法之后才创建线程 
    * 常驻线程 corePool 数量为 2 
    * 阻塞队列的任务为 3 
    * 最大线程数是 5 
  * 线程优先使用线程池中的常驻线程,但常驻线程用完之后第三线程会进入阻塞队列中进行等待 
  * 单队列满了之后,第六个任务来时会在线程池中进行创建,直至线程池满 
    * 3,4,5 还在等待,第六个线程会优先执行 
  * 当线程池满,阻塞队列也满之后第九个任务来时,会执行拒绝策略 

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLZ8BhueLZUpvCnwqu0icoRbNDX3ceMXwlvEeNGibdxZGVmgBSMibSaiaQuw/640?wx_fmt=png&from=appmsg)

##  自定义线程线程池

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLKDtc6JaNUC3tFGPcf2jTzDLibkoMdzF0FHl3QlEKh0HKPkVRI3CG2ug/640?wx_fmt=png&from=appmsg)
image.png

  * 自定义线程池 

    
    
       //自定义线程池创建  
        public static void main(String[] args) {  
            ThreadPoolExecutor threadPool = new ThreadPoolExecutor(  
                    2,  
                    5,  
                    2L,  
                    TimeUnit.SECONDS,  
                    new ArrayBlockingQueue<>(3),  
                    Executors.defaultThreadFactory(),  
                    new ThreadPoolExecutor.AbortPolicy());  
            
            try {  
                for (int i = 1; i <= 10; i++) {  
                    //执行  
                    threadPool.execute(()->{  
                        System.out.println(Thread.currentThread().getName()+" 办理业务");  
                    });  
                }  
            } catch (Exception e) {  
                e.printStackTrace();  
            } finally {  
                //关闭资源  
                threadPool.shutdown();  
            }  
        }  


  * 效果 

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLtaF4f2CSwx17Uhs8mDhB4ZDvnGKGITia0n4JyXoThjM6OQc2ia7eCoQg/640?wx_fmt=png&from=appmsg)

#  Fork/Join分支合并框架

  * 基于分支合并框架实现分治法求 1+...100 

    
    
    public class MyTask extends RecursiveTask<Integer> {  
    
        //拆分的差值不能超过10  
        private static final Integer DIFFERENCE = 10;  
        private Integer begin;  //拆分起始值  
        private Integer end;    //拆分结束值  
        private int result;    //结果值  
          
        //创建有参构造  
        public MyTask(Integer begin, Integer end) {  
            this.begin = begin;  
            this.end = end;  
        }  
          
        //拆分与合并过程  
        @Override  
        protected Integer compute() {  
            //判断相加的两个值是否大于10  
            if ((end-begin)<=DIFFERENCE) {  
                //相加  
                for (int i = begin; i <=end; i++) {  
                    result += i;  
                }  
            }else{  
                //进一步拆分  
                int middle = (begin+end)/2;  
                //拆分左边  
                MyTask myTask01 = new MyTask(begin, middle);  
                //拆分右边  
                MyTask myTask02 = new MyTask(middle+1, end);  
                //执行拆分  
                myTask01.fork();  
                myTask02.fork();  
                //合并结果  
                result = myTask01.join()+ myTask02.join();  
            }  
            return result;  
        }  
    }  
    
  * 测试代码 

    
    
     public static void main(String[] args) throws ExecutionException, InterruptedException {  
            //创建MyTask对象  
            MyTask myTask = new MyTask(0, 100);  
            //创建分支池对象  
            ForkJoinPool forkJoinPool = new ForkJoinPool();  
            ForkJoinTask<Integer> forkJoinTask = forkJoinPool.submit(myTask);  
            //获取合并结果  
            Integer result = forkJoinTask.get();  
            System.out.println(result);  
            forkJoinPool.shutdown();  
        }  


  * 结果 

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyLxTibAeDPiaCAFuaoEMrbEx8cz7Z0oHZFHpJ1TUtg41InNdTYpa6QCRLw/640?wx_fmt=png&from=appmsg)

#  CompletableFuture异步回调

  * 继承实现结构 

![](https://mmbiz.qpic.cn/mmbiz_png/hyAMveHd7sfbwGpppHrhqvOurTUfOnyL6FR5rBKxfEU0TX7oH4OP2JCIYsEGvjfoamAllwSRMycDVHPUre5NDA/640?wx_fmt=png&from=appmsg)
image.png

  * 没有返回值的异步调用 

    
    
        private static void nonReturnValue() throws InterruptedException, ExecutionException {  
            //异步调用没有返回值  
            CompletableFuture<Void> completableFuture = CompletableFuture.runAsync(()->{  
                System.out.println(Thread.currentThread().getName()+" completableFuture1");  
            });  
            completableFuture.get();  
        }  


  * 有返回值的异步调用 

    
    
        private static void returnValue() {  
            //异步调用有返回值  
            CompletableFuture<Integer> completableFuture = CompletableFuture.supplyAsync(() -> {  
                System.out.println(Thread.currentThread().getName() + " completableFuture02");  
                int i = 1/0; //模拟异常  
                return 1024;  
            });  
            completableFuture.whenComplete((t,u)->{  
                System.out.println("t = " + t);  
                //u为异常信息  
                System.out.println("u = " + u);  
            });  
        }  




预览时标签不可点

[ 阅读原文 ](javascript:;)

微信扫一扫  
关注该公众号





****



****



×  分析

  收藏

