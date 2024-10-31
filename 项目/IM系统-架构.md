# IM系统架构

![](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409061808034.jpeg)

## 业务功能

### 发送消息

服务器将消息写入消息队列，由相应程序持久化消息以及写扩散给所有订阅者

### 接收消息

直接从服务器获取消息

## 客户端

直接与用户交互，同时兼具发送和接受消息的功能

## 接入层

## 架构演进

### v1

发送方，接收方，IM服务器

![img](https://learn.lianglianglee.com/专栏/即时消息技术剖析与实战/assets/3a685f9d3f363cb7f748b73b6f8a6147.png)

![img](https://learn.lianglianglee.com/专栏/即时消息技术剖析与实战/assets/41bd1f4c74e980f88cd1a2247f292133.png)

![img](https://go-goim.github.io/docs/prologue/introduction/send_rec_msg.png)

![img](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409151818335.webp)
