#  深度解析 JDK8 中的数据结构



最近在整理数据结构方面的知识, 系统化看了下Java中常用数据结构, 突发奇想用动画来绘制数据流转过程

** 文末，有个特别好的网站推荐  **

主要基于JDK8, 可能会有些特性与jdk7之前不相同, 例如LinkedList LinkedHashMap中的双向列表不再是回环的.

HashMap中的单链表是尾插, 而不是头插入等等, 后文不再赘叙这些差异, 本文目录结构如下:

![](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr7rVsKUYZ92ChGkzQdP5SHakUjY3v1AtgLJ6aibibM9Yg7Itibj5u3DuuL6h8845SUOK6v9nDMvILcXQ/640?wx_fmt=png)

###  **LinkedList**

双向链表

 适用于乱序插入, 删除. 指定序列操作则性能不如ArrayList, 这也是其数据结构决定的.

**add(E) / addLast(E)**

![](https://mmbiz.qpic.cn/mmbiz_gif/tO7NEN7wjr7rVsKUYZ92ChGkzQdP5SHaBdC5PMibExiaLZ3ibAgFQEWWZ6rtI03NbnUxx9JH05Ym6LSfVSuz6DpYA/640?wx_fmt=gif)

**add(index, E)**

这边有个小的优化, 他会先判断index是靠近队头还是队尾, 来确定从哪个方向遍历链入.


```java
  1. if (index < (size >> 1)) {

  2.     Node<E> x = first;

  3.     for (int i = 0; i < index; i++)

  4.         x = x.next;

  5.     return x;

  6. } else {

  7.     Node<E> x = last;

  8.     for (int i = size - 1; i > index; i--)

  9.         x = x.prev;

  10.     return x;

  11. }
```


​    
​    

![](https://mmbiz.qpic.cn/mmbiz_gif/tO7NEN7wjr7rVsKUYZ92ChGkzQdP5SHa6fspeeic3jr8zOsib4icXP56diagQJO1FpRdSsmHIW8KBjyxcBIyYyhJXg/640?wx_fmt=gif)

**靠队尾**

![](https://mmbiz.qpic.cn/mmbiz_gif/tO7NEN7wjr7rVsKUYZ92ChGkzQdP5SHaeEva0A2K9gxjvrTYiawgg6vXPnMKTtvR2QAsmdDDvbiceZaEQfetyqcQ/640?wx_fmt=gif)

**get(index)**

也是会先判断index, 不过性能依然不好, 这也是为什么不推荐用for(int i = 0; i < lengh;
i++)的方式遍历linkedlist, 而是使用iterator的方式遍历.

![](https://mmbiz.qpic.cn/mmbiz_gif/tO7NEN7wjr7rVsKUYZ92ChGkzQdP5SHathJxbAs6jRRRGib6MhtCPrJ9WED8DuftJsA02ImZSC1DoKIa57pGshQ/640?wx_fmt=gif)

![](https://mmbiz.qpic.cn/mmbiz_gif/tO7NEN7wjr7rVsKUYZ92ChGkzQdP5SHaGZLGyvfQv8Yfazuq6wSXc073PS1nG6SicYzWOszp5ibZlItB4j3QYbAQ/640?wx_fmt=gif)

**remove(E)**

![](https://mmbiz.qpic.cn/mmbiz_gif/tO7NEN7wjr7rVsKUYZ92ChGkzQdP5SHay4ChdYfE1nSkRia5g30csomhGzwuBGlVwgSv3IsTvP4UwN4Oekdliabg/640?wx_fmt=gif)

###  **ArrayList**

数组

因此按序查找快, 乱序插入, 删除因为涉及到后面元素移位所以性能慢.

**add(index, E)**

![](https://mmbiz.qpic.cn/mmbiz_gif/tO7NEN7wjr7rVsKUYZ92ChGkzQdP5SHaKQmqgHqbCZNjdwoHPPFvLTHsG8K0WebicZevaO56ib8QZz3iaZrBR3nRA/640?wx_fmt=gif)

**扩容**

一般默认容量是10, 扩容后, 会length*1.5.

![](https://mmbiz.qpic.cn/mmbiz_gif/tO7NEN7wjr7rVsKUYZ92ChGkzQdP5SHaic8ss6LHicZ2cg95q46w4qlqPgTGRlenVb0TfdaXcExOeTmP6WRAUX6g/640?wx_fmt=gif)

**remove(E)**

循环遍历数组, 判断E是否equals当前元素, 删除性能不如LinkedList.

![](https://mmbiz.qpic.cn/mmbiz_gif/tO7NEN7wjr7rVsKUYZ92ChGkzQdP5SHaFA0h3zKbhsGI52A59mG2X3K1jrnrfIwMXzz4V8eeDJuKqlL1eQnGmQ/640?wx_fmt=gif)

###  **Stack**

数组

经典的数据结构, 底层也是数组, 继承自Vector, 先进后出FILO, 默认new Stack()容量为10, 超出自动扩容.

**push(E)**

![](https://mmbiz.qpic.cn/mmbiz_gif/tO7NEN7wjr7rVsKUYZ92ChGkzQdP5SHaAJPPMMCe3X4G9iaSqfia9J0nScH1nK43XK0e72XvgpukmjbZhIO14SFw/640?wx_fmt=gif)

**pop()**

![](https://mmbiz.qpic.cn/mmbiz_gif/tO7NEN7wjr7rVsKUYZ92ChGkzQdP5SHaQOf7cpkxBTiboP4cREkAZjdoJxvUy8mOG5FtsFROQJSjw0ibXf6Xquvw/640?wx_fmt=gif)

**后缀表达式**

Stack的一个典型应用就是计算表达式如 9 + (3 - 1) * 3 + 10 / 2, 计算机将中缀表达式转为后缀表达式, 再对后缀表达式进行计算.

**中缀转后缀**

** 1、  ** 数字直接输出

** 2、  ** 栈为空时，遇到运算符，直接入栈

** 3、  ** 遇到左括号, 将其入栈

** 4、  ** 遇到右括号, 执行出栈操作，并将出栈的元素输出，直到弹出栈的是左括号，左括号不输出。

** 5、  ** 遇到运算符(加减乘除)：弹出所有优先级大于或者等于该运算符的栈顶元素，然后将该运算符入栈

** 6、  ** 最终将栈中的元素依次出栈，输出。

![](https://mmbiz.qpic.cn/mmbiz_gif/XaklVibwUKn5XmeIJ2TrXaKy1nOpDVgafRKzcSkm0GSRZYyneQTkVLmGg2HNz1HkMrGmUjNEsfdjKaOTSjBJdUA/640?wx_fmt=gif)

**计算后缀表达**

** 1、  ** 遇到数字时，将数字压入堆栈

** 2、  ** 遇到运算符时，弹出栈顶的两个数，用运算符对它们做相应的计算, 并将结果入栈

** 3、  ** 重复上述过程直到表达式最右端

** 4、  ** 运算得出的值即为表达式的结果

![](https://mmbiz.qpic.cn/mmbiz_gif/XaklVibwUKn5XmeIJ2TrXaKy1nOpDVgafyP080icsRJsDtIlicTnfPTYBubtGRB4PSX8ib5LjJ6OGNibibhicNBFNMnIw/640?wx_fmt=gif)

###  **队列**

与Stack的区别在于, Stack的删除与添加都在队尾进行, 而Queue删除在队头, 添加在队尾.

**ArrayBlockingQueue**

生产消费者中常用的阻塞有界队列, FIFO.

**put(E)**

![](https://mmbiz.qpic.cn/mmbiz_gif/tO7NEN7wjr7rVsKUYZ92ChGkzQdP5SHasico7kZFmNpBib9nUCAmrFjy7jy6JFKwYEZoQuwvPmX8DK8tR3zfw1Xg/640?wx_fmt=gif)

**put(E) 队列满了**


          1. final ReentrantLock lock = this.lock;
    
      2. lock.lockInterruptibly();
    
      3. try {
    
      4.     while (count == items.length)
    
      5.         notFull.await();
    
      6.     enqueue(e);
    
      7. } finally {
    
      8.     lock.unlock();
    
      9. }


​    
​    

![](https://mmbiz.qpic.cn/mmbiz_gif/tO7NEN7wjr7rVsKUYZ92ChGkzQdP5SHa7WgqayhTmCcd7o4I8nBkOOcncIRYHjSUss3JDTQ2VJI9I9Wb3iayniag/640?wx_fmt=gif)

**take()**

当元素被取出后, 并没有对数组后面的元素位移, 而是更新takeIndex来指向下一个元素.

takeIndex是一个环形的增长, 当移动到队列尾部时, 会指向0, 再次循环.


          1. private E dequeue() {
    
      2.     // assert lock.getHoldCount() == 1;
    
      3.     // assert items[takeIndex] != null;
    
      4.     final Object[] items = this.items;
    
      5.     @SuppressWarnings("unchecked")
    
      6.     E x = (E) items[takeIndex];
    
      7.     items[takeIndex] = null;
    
      8.     if (++takeIndex == items.length)
    
      9.         takeIndex = 0;
    
      10.     count--;
    
      11.     if (itrs != null)
    
      12.         itrs.elementDequeued();
    
      13.     notFull.signal();
    
      14.     return x;
    
      15. }


​    
​    

![](https://mmbiz.qpic.cn/mmbiz_gif/tO7NEN7wjr7rVsKUYZ92ChGkzQdP5SHaWUfnZmCDz5RECic96jVUicyW1NRuoerLl7LLEJqicOMANicib9dxc4rzsyQ/640?wx_fmt=gif)

###  HashMap

数组+单向链表+红黑树

最常用的哈希表, 面试的童鞋必备知识了, 内部通过数组 + 单链表的方式实现. jdk8中引入了红黑树对长度 > 8的链表进行优化, 我们另外篇幅再讲.

**put(K, V *  ** )*

![](https://mmbiz.qpic.cn/mmbiz_gif/tO7NEN7wjr7rVsKUYZ92ChGkzQdP5SHaRicpWopiaNt9KLN8Bzb0YOWkEd7mKBGl7VfQARZN36GyLy0WFnu8Ivicg/640?wx_fmt=gif)

**put(K, V) 相同hash值**

![](https://mmbiz.qpic.cn/mmbiz_gif/tO7NEN7wjr7rVsKUYZ92ChGkzQdP5SHaJjomMDuwe9DUO1lxV06HJxTqjI2CiaryvAD985TqBjehcjXlYMnBOKA/640?wx_fmt=gif)

**resize 动态扩容**

当map中元素超出设定的阈值后, 会进行resize (length * 2)操作, 扩容过程中对元素一通操作, 并放置到新的位置.

具体操作如下:

** 1、  ** 在jdk7中对所有元素直接rehash, 并放到新的位置.

  

** 2、  ** 在jdk8中判断元素原hash值新增的bit位是0还是1, 0则索引不变, 1则索引变成"原索引 + oldTable.length".

  


          1. //定义两条链
    
      2. //原来的hash值新增的bit为0的链，头部和尾部
    
      3. Node<K,V> loHead = null, loTail = null;
    
      4. //原来的hash值新增的bit为1的链，头部和尾部
    
      5. Node<K,V> hiHead = null, hiTail = null;
    
      6. Node<K,V> next;
    
      7. //循环遍历出链条链
    
      8. do {
    
      9.     next = e.next;
    
      10.     if ((e.hash & oldCap) == 0) {
    
      11.         if (loTail == null)
    
      12.             loHead = e;
    
      13.         else
    
      14.             loTail.next = e;
    
      15.         loTail = e;
    
      16.     }
    
      17.     else {
    
      18.         if (hiTail == null)
    
      19.             hiHead = e;
    
      20.         else
    
      21.             hiTail.next = e;
    
      22.         hiTail = e;
    
      23.     }
    
      24. } while ((e = next) != null);
    
      25. //扩容前后位置不变的链
    
      26. if (loTail != null) {
    
      27.     loTail.next = null;
    
      28.     newTab[j] = loHead;
    
      29. }
    
      30. //扩容后位置加上原数组长度的链
    
      31. if (hiTail != null) {
    
      32.     hiTail.next = null;
    
      33.     newTab[j + oldCap] = hiHead;
    
      34. }


​    
​    

![](https://mmbiz.qpic.cn/mmbiz_gif/tO7NEN7wjr7rVsKUYZ92ChGkzQdP5SHaahpWVksvMSQYkrR7prnbBnjeicWxia8v1OZWzZdkmLM8ybEY1H4Y0a5Q/640?wx_fmt=gif)

###  **LinkedHashMap**

额外维护双向链表，用于维护元素之间的插入顺序

继承自HashMap, 底层额外维护了一个双向链表来维持数据有序. 可以通过设置accessOrder来实现FIFO(插入有序)或者LRU(访问有序)缓存.

**put(K, V)**

![](https://mmbiz.qpic.cn/mmbiz_gif/tO7NEN7wjr7rVsKUYZ92ChGkzQdP5SHabxWnLMeVdf3zUKZIKicoeiclYsPWuLzKlicib5NfF5o1a4OBwJbyWEFtFA/640?wx_fmt=gif)

**get(K)**

accessOrder为false的时候, 直接返回元素就行了, 不需要调整位置.

accessOrder为true的时候, 需要将最近访问的元素, 放置到队尾.

![](https://mmbiz.qpic.cn/mmbiz_gif/tO7NEN7wjr7rVsKUYZ92ChGkzQdP5SHa9uNnPrMdSTx0GnqzpytcIPR3xA9HCcb4XkZJPdBP05dsd1ReficvrFQ/640?wx_fmt=gif)

**removeEldestEntry 删除最老的元素**

![](https://mmbiz.qpic.cn/mmbiz_gif/tO7NEN7wjr7rVsKUYZ92ChGkzQdP5SHaJjucGJ79aIWxZw4UJjh7UofOdHj5ibEVuzrJlDpNtg1cvia7lia1HlJVQ/640?wx_fmt=gif)

![](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr511H00l56ND6Ows6h51icxJpkzvCjibkdcpLVOx8Wr6c4azOMiaNAgAIibEbNElMoXicrdtcxLmyyc1Aw/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr511H00l56ND6Ows6h51icxJxLJyjRjlXtItROpkqxj3oy1XrsUfibhd2PKXbBCzcPvpnVjR9jEeA4Q/640?wx_fmt=jpeg)

**推荐一个数据结构的动图网站，挺经典的。**

https://visualgo.net/zh

  

  

往期推荐

  

[ Java 里的 for (;;) 与 while (true)，哪个更快？
](http://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452977600&idx=2&sn=86c73989d2b50d0ce4d29ae0d60ddc59&chksm=88eddaa8bf9a53be4011479a4a6fde076b4704e63d261bc32b200a2cc4b0e6bf16c9d2bda06a&scene=21#wechat_redirect)
[ MySQL每秒57万的写入，带你装逼，带你飞 ！！
](http://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452977580&idx=2&sn=c3ea4309a617159889fbb07a4bf2f731&chksm=88eddac4bf9a53d2d9f46c5ed62215c6d6ece65ae5fec99236b2e2cbf48591ebf9b9d5b8d3cb&scene=21#wechat_redirect)
[ SpringBoot最新版：优雅停机，抢先解读~~ 拒绝kill -9
](http://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452977569&idx=2&sn=d8a2a7701bec29f75f2b8780d44e2a63&chksm=88eddac9bf9a53df1f460f978283376b3bf99e6be0012cb589df6c1c5f3fc14b75c9adf1d586&scene=21#wechat_redirect)

  

**阅读原文：** **** **最新 3625页大厂面试题**

预览时标签不可点

[ 阅读原文 ](javascript:;)

微信扫一扫  
关注该公众号





****



****



×  分析

  收藏

