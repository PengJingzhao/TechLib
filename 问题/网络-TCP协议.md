##  **1. 说说HTTP常用的状态码及其含义？**

![图片](https://mmbiz.qpic.cn/mmbiz_png/e7qSbfa9dbf85nGbgeyBqys0QBiciaKMBSHOibVbYcMhh39x40O69NmKO1pUYbOfeHAEFXeWVVl27BpTpDjbC88SQ/640?wx_fmt=png&wxfrom=13&tp=wxpic)

**不管是不是面试需要，我们都要知道，日常开发中的这几个状态码的含义哈：**

![图片](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409210920138.png)

 

## **2. 说说HTTP的状态码，301和302的区别？**

**思路:**这道题考查的知识点，也是HTTP状态码，302和301都有重定向的含义，但是它们也是有区别的。

**301：（永久性转移）**请求的网页已被永久移动到新位置。服务器返回此响应时，会自动将请求者转到新位置。

**302：（暂时性转移）**服务器目前正从不同位置的网页响应请求，但请求者应继续使用原有位置来进行以后的请求。此代码与响应GET和HEAD请求的301代码类似，会自动将请求者转到不同的位置。

## **3. HTTP 常用的请求方式，区别和用途？**

我们用得比较多就是**GET和POST**啦。

![图片](https://mmbiz.qpic.cn/mmbiz_png/e7qSbfa9dbf85nGbgeyBqys0QBiciaKMBSUicib88RzGmDEzDoRtFwEVaXagZpCQX7e67keRIsV9pzh01Mq2EWTo2g/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

**GET**和**POST**的区别：

- GET是获取数据的； 而POST是提交数据的
- GET用于获取信息，是无副作用的，且可缓存，可收藏； 而POST用于修改服务器上的数据，有副作用，不可缓存。



## **4. 请简单说一下你了解的端口及对应的服务？**

![图片](https://mmbiz.qpic.cn/mmbiz_png/e7qSbfa9dbf85nGbgeyBqys0QBiciaKMBSOmj7KSU6wOliaY7up9vsiaYc50KfOZVejZuccWAw77ApGxZBJZ6Ig5Sw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

6379是redis



## **5. 说下计算机网络体系结构**

计算机网路体系结构呢，有三层：ISO七层模型、TCP/IP四层模型、五层体系结构。大家可以记住这个图，如下

![图片](https://mmbiz.qpic.cn/mmbiz_png/e7qSbfa9dbf85nGbgeyBqys0QBiciaKMBSHG0fRhqCRlV4c0ibQlalZviaxw1udfyd9YwlzIwvibZnLKDnN4Q5UiaIhg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

计算机网络体系结构

5.1 ISO七层模型

ISO七层模型是国际标准化组织（International Organization for Standardization）制定的一个用于计算机或通信系统间互联的标准体系。

**应用层**：网络服务与用户的一个接口，常见的协议有：**HTTP、FTP SMTP SNMP DNS**.

**表示层**：数据的表示、安全、压缩。，确保一个系统的应用层所发送的信息可以被另一个系统的应用层读取。

**会话层**：建立、管理、终止会话,对应主机进程，指本地主机与远程主机正在进行的会话.

**传输层**：定义传输数据的协议端口号，以及流控和差错校验,协议有**TCP UDP**.

**网络层**：进行逻辑地址寻址，实现不同网络之间的路径选择,协议有**ICMP IGMP IP等**.

**数据链路层**：在物理层提供比特流服务的基础上，建立相邻结点之间的数据链路。

**物理层**：建立、维护、断开物理连接

![图片](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409210920006.png)

------

5.2 TCP/IP 四层模型

**应用层**：对应于OSI参考模型的（应用层、表示层、会话层）。

**传输层**: 对应OSI的传输层，为应用层实体提供端到端的通信功能，保证了数据包的顺序传送及数据的完整性。

**网际层**：对应于OSI参考模型的网络层，主要解决主机到主机的通信问题。

**网络接口层**：与OSI参考模型的数据链路层、物理层对应。

![图片](https://mmbiz.qpic.cn/mmbiz_png/e7qSbfa9dbf85nGbgeyBqys0QBiciaKMBS8bz9mfu7U0IibdaMV5Dic7eKMaukjic0J9894VDcmRMU1UPeWRU4gQjRA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

------

5.3 五层体系结构

**应用层**：对应于OSI参考模型的（应用层、表示层、话层）。

**传输层**：对应OSI参考模型的的传输层

**网络层**：对应OSI参考模型的的网络层

**数据链路层**：对应OSI参考模型的的数据链路层

**物理层**：对应OSI参考模型的的物理层。

![图片](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409210920457.png)

------



## **6. TCP和 UDP**

tcp和udp在运输层。

- **TCP 是面向连接**，能保证数据的**可靠**性交付，因此经常用于：

- - **文件传输**
  - HTTP / HTTPS

- **UDP 面向无连接**，它可以随时发送数据，是**不可靠**的，简单又高效，因此经常用于：

- - 包总量较少的通信，如 DNS 、SNMP等
  - **视频会议**、音频等多媒体通信



TCP 用于在传输层有必要实现可靠传输的情况，UDP 用于对高速传输和实时性有较高要求的通信。TCP

和 UDP 应该根据应用目的按需使用。



## **7 如何理解HTTP协议是无状态的**

**思路:**这道题主要考察候选人，是否理解Http协议，它为什么是无状态的呢？如何使它有状态呢？

如何理解无状态这个词呢？

★当浏览器第一次发送请求给服务器时，服务器响应了；如果同个浏览器发起第二次请求给服务器时，它还是会响应，但是呢，服务器不知道你就是刚才的那个浏览器。简言之，服务器不会去记住你是谁，所以是无状态协议。”

可以通过一个生活中的例子，来更好理解并记住它：

**有状态场景：**

```
        小红：今天吃啥子？

        小明：罗非鱼~

        小红：味道怎么样呀？
        
        小明：还不错，好香。
```

**无状态的场景：**

```
        小红：今天吃啥子？

        小明：罗非鱼~

        小红：味道怎么样呀？

        小明：？啊？你说啥？什么鬼？什么味道怎么样？
```

**Http加了Cookie的话**：

```
        小红：今天吃啥子？

        小明：罗非鱼~

        小红：你今天吃的罗非鱼，味道怎么样呀？

        小明：还不错，好香。
```



## **8. Session和Cookie和token的区别。**

**我们先来看Session和Cookie和token的概念吧：**

**Cookie**是保存在客户端的一小块文本串的数据。客户端向服务器发起请求时，服务端会向客户端发送一个Cookie，客户端就把Cookie保存起来。在客户端下次向同一服务器再发起请求时，Cookie被携带发送到服务器。服务器就是根据这个Cookie来确认身份的。cookie不安全

**session**（翻译：会话）指的就是服务器和客户端一次会话的过程。Session利用Cookie进行信息处理的，当客户端浏览器第一次访问服务器时，服务器会为浏览器创建一个sessionid，将sessionid放到Cookie中，存在客户端浏览器。当用户离开网站时，储存的session会被销毁。

Session对象存储着特定用户会话所需的属性及配置信息。

**token**：Token是服务端生成的一串字符串，以作客户端进行请求的一个**令牌**，当第一次登录后，服务器生成一个Token便将此Token返回给客户端，以后客户端只需带上这个Token前来请求数据即可，无需再次带上用户名和密码。

作用机制：**服务器并不保存token**，而是通过数据签名的方法，对数据用算法（如SHA-256）与私钥进行签名后作为Token，当Token发送给服务器时，服务会通过相同的算法与密钥进行签名，如果和Token中的签名相同，服务器就知道用户已经登录过了，并且可以直接得到用户的userID。（服务器用cpu的计算时间来换取了储存空间）



**Session 和Cookie的区别主要有这些：**

1. session是安全的，存储在服务端；cookie是不安全的，存储在客户端
2. session可以存任意的数据类型；   cookie只能存储字符串类型
3. session的有效期很短，关闭浏览器或者超时 都会失效 ； cookie可以长时间保存
4. seesion的存储空间很大  ；   cookie存储空间小（只有4kB）

![图片](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409210920763.png)



## **9. 说下HTTP/1.0，1.1，2.0的区别**

我们记住**HTTP/1.0**默认是短连接，可以强制开启，HTTP/1.1默认长连接，HTTP/2.0采用**多路复用**就差不多啦。

**HTTP/1.0**  ：

- 默认使用**短连接**，每次请求都需要建立一个TCP连接。它可以设置Connection: keep-alive 这个字段，强制开启长连接。

**HTTP/1.1**  ：

- 使用长连接的方式改善了 HTTP/1.0 短连接造成的性能开销。
- 支持管道（pipeline）网络传输，只要第一个请求发出去了，不必等其回来，就可以发第二个请求出去，可以减少整体的响应时间。

**HTTP/2.0**  ：

- HTTP/2 协议是基于 HTTPS 的
- 头部压缩：HTTP/2 会**压缩头**（Header）如果你同时发出多个请求，协议会帮你**消除重复的部分**
- 二进制格式：头信息和数据体都是二进制，并且统称为帧（frame）：**头信息帧（Headers Frame）和数据帧（Data Frame）**。
- 多路复用：**客户端和浏览器可以并发多个请求或回应，而不用按照顺序一一对应**。
- 服务器推送



## **10. http是什么**

**我的答案如下**：

HTTP，即**超文本传输协议**，是一个基于TCP/IP通信协议来传递明文数据的协议。

HTTP会存在这**几个问题**：

- 请求信息是明文传输，容易被窃听截取。
- 没有验证对方身份，存在被冒充的风险
- 数据的完整性未校验，容易被中间人篡改

为了解决Http存在的问题，Https出现啦



## **11. Https是什么？**

考察的知识点是HTTPS的工作流程，大家需要回答这几个要点，**公私钥、数字证书、加密、对称加密、非对称加密**。

**HTTPS= HTTP+SSL/TLS**，可以理解Https是身披SSL 的HTTP。

![图片](https://mmbiz.qpic.cn/mmbiz_png/e7qSbfa9dbf85nGbgeyBqys0QBiciaKMBS7D6xiciaLeicialw6athYaDlXhUoNIrCzNAkYGVQ1icg5BGJRzBaz7zSljA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

1。客户端发起Https请求，连接到服务器的443端口。

2。服务器必须要有一套数字证书（证书内容有公钥、证书颁发机构、失效日期等）。

3。服务器将自己的数字证书发送给客户端（公钥在证书里面，私钥由服务器持有）。

4。客户端收到数字证书之后，会验证证书的合法性。如果证书验证通过，就会生成一个随机的对称密钥，用证书的公钥加密。

5。客户端将公钥加密后的密钥发送到服务器。

6。服务器接收到客户端发来的密文密钥之后，用自己之前保留的私钥对其进行非对称解密，解密之后就得到客户端的密钥，然后用客户端密钥对返回数据进行对称加密，酱紫传输的数据都是密文啦。

7。服务器将加密后的密文返回到客户端。

8。客户端收到后，用自己的密钥对其进行对称解密，得到服务器返回的数据。



## **12.  说说什么是数字签名？什么是数字证书？**

公钥和个人等信息，经过Hash摘要算法加密，形成消息摘要；

将消息摘要拿到拥有公信力的认证中心（CA），用它的私钥对消息摘要加密，形成**数字签名**。

公钥和个人信息、数字签名共同构成**数字证书**



**13. 对称加密与非对称加密有什么区别**

**对称加密**：加密和解密使用同一把密钥。加密算法是DES、AES

- 效率高，不安全

**非对称加密**：是加密和解密使用不同的密钥。加密用公钥，解密用私钥。加密算法是RSA

- 安全，但是效率低

一般用非对称加密



**14. HTTP 与 HTTPS 的区别**

结合http和http是特性来回答:

- http是不安全的，https是安全的
- http的端口啊80，https是443
- http不需要证书，https需要证书
- http的报文没有加密，https是加密的



**15. 说说DNS的解析过程？**

![图片](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409210920240.png)

1. 首先会查找**浏览器的缓存**和**本地DNS服务器**，看看是否能找到**www.baidu.com**对应的IP地址，找到就直接返回，否则进行下一步。
2. 本地DNS服务器向**根域名服务器**发送请求，根域名服务器返回负责**.com**的顶级域名服务器的IP地址的列表
3. 本地DNS服务器再向**顶级域名服务器**发送一个请求，顶级域名服务器返回负责**.baidu**的权威域名服务器的IP地址列表
4. 本地DNS服务器再向**权威域名服务器**发送一个请求，权威域名服务器返回**www.baidu.com**所对应的IP地址      

![图片](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409210920370.png)



**16. 从浏览器输入url到显示主页的过程（也是http的请求过程）**

**思路:**这道题主要考察的知识点是HTTP的请求过程，**DNS解析，TCP三次握手，四次挥手这几个要点**，我们都可以讲下。

```
    1.DNS解析，查找域名对应的IP地址。

    2.与服务器通过三次握手，建立TCP连接

    3.向服务器发送HTTP请求

    4.服务器处理请求，返回网页内容

    5.浏览器解析，并渲染页面

    6.TCP四次挥手，连接结束
```



**17. tcp的连接：三次握手**

![图片](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409210920608.png)

![图片](https://mmbiz.qpic.cn/mmbiz_png/e7qSbfa9dbf85nGbgeyBqys0QBiciaKMBSkIVguReUDznibicqzDyFtaob5wOne9EIsyibPiabarYlZwhhFttJlnjz5w/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)



![图片](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409210920475.png)

面试这样答 :

 **一次握手：客户端向服务端申请建立连接**  2个东西

- 客户端会随机生成序号 seq = x  
- 把SYN 标记为 1 ：表示  “ 请求建立新连接 ”。发送给服务端

**二次握手：服务端回复客户端，确认连接请求**  4个东西

- 随机生成服务端的序号 seq = y
- 把确认号ack 设置为 x + 1
-  SYN标志置为1
- ACK 标志都置为1，  确认对方的连接有效

**三次握手：客户端确认服务端，发送 ACK 标志为1的数据包**  3个东西

- 把确认包ACK设置为1 ， 确认对方的连接有效
- 客户端的序号置为x+1
- 确认号ack 为 服务端序号y + 1



那么顺便说一下：

为什么是三次呢？（和为什么不是2次、为什么不是4次是一样的）

因为tcp协议是面向连接的可靠协议。两次不安全，四次没必要。

第一次握手的目的是：服务端：知道客户端有发送能力

第二次握手的目的是：客户端：知道服务端有发送能力，也有接收能力。

那么问题来了，目前还未得知的结论是：服务端有没有接收能力呢？

假设现在两次握手就建立连接了，如果失效的连接请求又突然传给服务端，服务端就认为这个连接是可用的。此时服务端就会给失效的客户端发送资源，这就导致了很多端口白白的开着，资源十分浪费。

所以第三次握手的决定性就体现了：    

第三次握手的目的是：服务端：知道客户端有接收能力

总的来说，三次才能保证双方具有接收和发送的能力



**18.TCP四次挥手，连接结束**

![图片](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409210920889.png)

假设客户端先发起关闭请求。

**一次挥手**：客户端发送FIN=1 段，会生成一个序列号seq=u。**（客户端进入 FIN_WAIT_1 状态）**

**二次握手**：服务端会发送 ACK =1 报文，设置确认号ack=u+1，也会生成一个服务端的序号 seq = v 。**（客户端进入FIN_WAIT_2状态）**

**三次握手**：等待服务端处理完数据后，服务端发送 FIN=1报文，且生成序列号seq=w，设置确认号ack=u+1。**（服务端进入 LAST_ACK 的状态）**

**四次握手**：客户端回复ACK=1应答报文，设置确认号ack=w+1，生成序号seq=u+1。**（客户端进入TIME_WAIT状态 : 客户端在经过2MSL一段时间后，自动进入CLOSED状态）**

此后，双方都进入closed关闭状态。



**19. TIME_WAIT状态，是面试的高频考点**

为什么客户端第四次握手时，发送 ACK 之后不直接关闭，而是要等一阵子才关闭。

原因就是，**要确保服务端是否已经收到了我们的 ACK 报文**，如果没有收到的话，**服务端会重新发 FIN 报文**给客户端，客户端再次收到 ACK 报文之后，就知道之前的 ACK 报文丢失了，然后再次发送 ACK 报文。

```
  TIME_WAIT 持续的时间，最少是一个报文的来回时间2MSL, 即两倍的MSL(1MSL一般是30秒)。一般会设置一个计时，如果过了这个计时没有再次收到 FIN 报文，则代表对方成功就是 ACK 报文，此时处于 CLOSED 状态。
```



**20. TCP半连接队列和全连接队列**

TCP进入三次握手前，**服务端**会从**CLOSED**状态变为**LISTEN**状态 , 同时在内部创建了两个队列：**半连接队列（SYN队列）**和  **全连接队列（ACCEPT队列）**。

回忆下 TCP三次握手的图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/e7qSbfa9dbf85nGbgeyBqys0QBiciaKMBS1ZuNfSdGqohFsbKhZdkQJZ8rlIKrDucy3RqkWYpsmpD3icf4fK5308A/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

- **半连接队列（SYN队列）：**

  ​    TCP三次握手时，客户端发送SYN到服务端，服务端收到之后，便回复   **ACK和SYN**，状态由**LISTEN变为SYN_RCVD**，此时这个连接就被推入了**SYN队列**，即半连接队列

- **全连接队列（AXCCEPT队列）：**

  ​    当客户端回复ACK , 服务端接收后，**三次握手**就完成了。这时连接会等待被具体的应用取走，在被取走之前，它被推入**ACCEPT队列**，即全连接队列。

  

**21. TCP的拥塞控制**

四种算法：**慢启动、拥塞避免、拥塞发生、快速恢复**

拥塞控制是防止过多的数据包注入到网络中，避免出现网络负载过大的情况



它跟**流量控制**又有什么区别呢？

拥塞控制是**作用于网络的，防止过多的数据包注入到网络中，避免出现网络负载过大的情况**

流量控制是**作用于接收端的**，根据**接收端的实际接收能力控制发送速度**，防止分组丢失的



## **22.TCP的流量控制**

TCP 提供一种机制可以让**发送端**根据**接收端的实际接收能力**，来控制发送的数据量

目的就是让发送端的发送速度不要太快，要让接收端来得及接收


### [1.1. 基本术语](#_1-1-基本术语)

1. **结点 （node）**：网络中的结点可以是计算机，集线器，交换机或路由器等。
2. **链路（link ）** : 从一个结点到另一个结点的一段物理线路。中间没有任何其他交点。
3. **主机（host）**：连接在因特网上的计算机。
4. **ISP（Internet Service Provider）**：因特网服务提供者（提供商）

![ISP (Internet Service Provider) Definition](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409210952179.png)