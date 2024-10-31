# 业务层ACK机制

一、前言
----

> `SMC` 定理（`Single-Message Communication` ）：端到端的通讯系统，消息不丢不重是不可能的。

**在 `IM` 系统设计中，同样面临这两大难题：**

1.  **消息丢失** ：用户A 发送一条消息给用户B，用户B 没有收到消息
2.  **消息重复** ：用户A 发送一条消息给用户B，用户B 收到两条消息

**实际上并没有打破 `SMC` 定理，只是在不同层做了不同处理：**

1.  **系统层面** ：消息重复了
2.  **业务层面** ：用户感知不到

### （1）消失丢失有哪几种情况？

**先从一个场景里来看下哪些环节可能存在丢消息的风险：**

> 举个栗子：用户 A 给用户 B 发送一条消息。

![2022-02-0621-07-19.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d9344aa6eb224f65958ea1e7a06215e7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

**根据上述时序图，发消息可分为两个部分：**

1.  **发送方** ：用户A发送消息到服务端，对应步骤1、2、3
2.  **接收方** ：服务端推送消息给用户B，对应步骤4

那么按照发送方和接收方来分别讨论。

#### 1）发送方

发送失败，超时重发

**消息丢失，问题分析：**

1.  **步骤1** ，用户A发送消息到服务端：可能网络波动等原因发送失败
2.  **步骤2** ，服务端存储消息：可能服务端宕机、`DB` 宕机等
3.  **步骤3** ，服务端返回消息给用户A：可能服务端宕机、用户A断连等

**解决方案：**

1.  **步骤1** ：客户端超时重发即可。
2.  **步骤2** ：客户端超时重发即可。
3.  **步骤3** ：服务端超时重发，客户端去重。

  

#### 2）接收方

**消息丢失，问题分析：**

*   **步骤4** ，服务端推送消息给用户B：
    *   服务端宕机
    *   用户 B 的设备在把消息写入本地 `DB` 时，出现异常导致没能成功入库

**解决方案：** **参考 `TCP` 协议的 `ACK` 机制，实现一套业务层的 `ACK` 协。**

  

### （2）解决丢失的方案：业务层 `ACK` 机制

接收方成功接收消息时，要向发送方，返回一个标准ACK数据包

**`ACK` （`Acknowledge` ，确认）** ：在 `TCP` 协议中，默认提供了 `ACK` 机制，通过一个协议自带的标准的 `ACK` 数据包，来对通信方接收的数据进行确认，告知通信发送方已经确认成功接收了数据。

**问题：为什么有了 `TCP` 协议本身的 `ACK` 机制，为什么还需要业务层的 `ACK` 机制？**

需要手动的业务上的ACK机制来保证消息处理成功

> **存在这种场景** ：客户端接收消息成功了，但处理消息或存入 `DB` 时候出现问题，导致用户看不到这个消息。
>
> 这个时候就需要 业务层 上的 `ACK` 保证了。

**业务层 `ACK` 机制如下：**

![2022-02-0622-43-36.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/03b6e01f70fe44bd846d9e57217e9fb3~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

**服务端会维护一个 “等待 `ACK` 队列”：**

SequenceId

*   服务端推送消息给用户B：消息中包含消息 `Id` （`msgId` ）
*   **服务端将当前消息加入到 “等待 `ACK` 队列”**
*   **用户B 成功接收到消息后，会给服务端发送一个 `ACK` 包（包含对应消息 `Id` ）**
*   **若服务端长时间没有收到对应的 `ACK` 包，会尝试重发**
*   **服务端重发一定次数后仍没收到对应 `ACK` 包，则：将消息存入离线消息font>**

**小结：`ACK` 机制 + 超时重传 + 去重，能解决大部分问题。**

  


二、`MQTT` 实战
-----------

**`MQTT` 中有 3 种可靠通信 `QoS` 级别：**

*   `QoS 0` ：表示消息最多收到一次，即消息可能丢失，但是不会重复。
*   **`QoS 1` ：表示消息至少收到一次，即消息保证送达，但是可能重复。**
*   **`QoS 2` ：表示消息只会收到一次，即消息有且只有一次。**

**对应 `Client` 端使用：

````java
public class DefaultMqttListener implements IMqttListener, Runnable {

    long count = 0;
    long start = System.currentTimeMillis();
    private HashMap serverDetailsHash;

    public DefaultMqttListener(HashMap serverProp) {
        this.serverDetailsHash = serverProp;
    }

    CallbackConnection myconnection;

    @Override
    public void init() {
        MQTT mqtt = new MQTT();
        String user = env("APOLLO_USER", (String) serverDetailsHash.get("userName"));    //No I18N
        String password = env("APOLLO_PASSWORD", (String) serverDetailsHash.get("password"));    //No I18N
        String host = env("APOLLO_HOST", (String) serverDetailsHash.get("hostName"));    //No I18N
        int port = Integer.parseInt(env("APOLLO_PORT", (String) serverDetailsHash.get("port")));
        try {
            mqtt.setHost(host, port);
            mqtt.setUserName(user);
            mqtt.setPassword(password);
            final CallbackConnection connection = mqtt.callbackConnection();
            myconnection = connection;
            connection.listener(new org.fusesource.mqtt.client.Listener() {
                public void onConnected() {
                }

                public void onDisconnected() {
                }

                public void onFailure(Throwable value) {
                    value.printStackTrace();
                    System.exit(-2);
                }

                public void onPublish(UTF8Buffer topic, Buffer msg, Runnable ack) {
                    long time = System.currentTimeMillis();
                    callback(topic, msg, ack, connection, time);
                }
            });
            connection.connect(new Callback<Void>() {
                @Override
                public void onSuccess(Void value) {
                    NmsLogMgr.M2MERR.log("MQTT Listener connected in ::::", Log.SUMMARY);
                    ArrayList getTopics = (ArrayList) serverDetailsHash.get("Topics");
                    for (int i = 0; i < getTopics.size(); i++) {
                        HashMap getTopic = (HashMap) getTopics.get(i);
                        String topicName = (String) getTopic.get("topicName");
                        String qosType = (String) getTopic.get("qosType");
                        Topic[] topic = {new Topic(topicName, getQosType(qosType))};
                        connection.subscribe(topic, new Callback<byte[]>() {
                            public void onSuccess(byte[] qoses) {
                            }

                            public void onFailure(Throwable value) {
                                value.printStackTrace();
                                System.exit(-2);
                            }
                        });
                    }
                    //Topic[] topics = {new Topic("adminTest", QoS.AT_LEAST_ONCE),new Topic("adminTest1", QoS.AT_LEAST_ONCE)};
                }

                @Override
                public void onFailure(Throwable value) {
                    value.printStackTrace();
                    System.exit(-2);
                }
            });

            // Wait forever..
            synchronized (Listener.class) {
                while (true) {
                    Listener.class.wait();
                }

            }
        } catch (URISyntaxException e1) {
            // TODO Auto-generated catch block
            e1.printStackTrace();
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
    }

    private static String env(String key, String defaultValue) {
        String rc = System.getenv(key);
        if (rc == null) {
            return defaultValue;
        }
        return rc;
    }

    @Override
    public void callback(UTF8Buffer topic, Buffer msg, Runnable ack, CallbackConnection connection, long time) {
        // TODO Auto-generated method stub
        try {
            String Message = msg.utf8().toString();
            MQTTMessage mqttMsg = new MQTTMessage();
            mqttMsg.setMQTTMessage(Message);
            mqttMsg.setTime(time);
            mqttMsg.setTopic(topic);
            mqttMsg.sethostName((String) serverDetailsHash.get("hostName"));
            MQTTCacheManager.mgr.addToCache(mqttMsg);
        } catch (Exception e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }

    }

    @Override
    public void close() {
        // TODO Auto-generated method stub
        NmsLogMgr.M2MERR.log("myconnection closed", Log.SUMMARY);
        myconnection.disconnect(new Callback<Void>() {
            @Override
            public void onSuccess(Void value) {
                System.exit(0);
            }

            @Override
            public void onFailure(Throwable value) {
                value.printStackTrace();
                System.exit(-2);
            }
        });

    }

    @Override
    public void run() {
        this.init();
        // TODO Auto-generated method stub
    }

    public QoS getQosType(String name) {
        Properties qosContainer = new Properties();
        qosContainer.put("0", QoS.AT_MOST_ONCE);
        // qosContainer.put("1", QoS.AT_LEAST_ONCE);
        // qosContainer.put("2", QoS.EXACTLY_ONCE);
        QoS qosName = (QoS) qosContainer.get(name);
        return qosName;
    }
}

````

笔记：IM 消息丢失的解决方案
===============

大部分场景中，业务层ACK确认机制 + 消息重传机制 + 消息完整性检查，能解决消息丢失的问题。

## 业务层的ACK

ACK是确认字符（Acknowledge character）的意思，TCP协议默认提供了ACK机制，如果接收方成功接收到数据，就会回复一个ACK数据，表示发送方发出的数据已确认接收无误，在“三次握手”、“四次挥手”中经常见到。

 ACK确认机制：TCP传输时将每个字节的数据都进行编号，即序列号。TCP传输的过程中，每次接收方收到数据后，都会对传输方进行确认应答，也就是发送ACK报文。 这个ACK报文当中带有对应的确认序列号，告诉发送方，接收到了哪些数据，下一次的数据从哪里发。 有了序列号能够将接收到的数据根据序列号排序，并且去掉重复序列号的数据。这也是TCP传输可靠性的保证之一。

## 消息重传机制

发送方发送一部分数据后，都会等待接收方发送的ACK报文，并解析ACK报文，判断数据是否传输成功。发送方迟迟收不到ACK报文的原因可能有两个：

*   数据在传输过程中由于网络原因等直接全体丢包，接收方没有接收到；
*   接收方接收到了响应的数据，但是发送的ACK报文响应却由于网络原因丢包了。

超时重传机制：

发送方在发送完数据后等待一个时间，如果在超时时间内没有接收到ACK报文，就重新发送数据。如果是上述的第一个原因，接收方收到二次重发的数据后，便进行ACK应答。如果是第二个原因，接收方发现接收的数据已存在，就直接丢弃，仍旧发送ACK应答。

业务层的ACK确认机制参考了TCP的ACK确认机制，其策略是IM服务器在推送消息时，携带一个标识SID（安全标识符，类似TCP的sequenceId），推送出消息后会将当前消息添加到“待ACK消息列表”，客户端B成功接收完消息后，会给IM服务器回一个业务层的ACK包，包中携带有本条接收消息的SID，IM服务器接收后，会从“待ACK消息列表”记录中删除此条消息，本次推送才算真正结束。

业务层的消息重传机制也参考了TCP协议的重传机制，IM服务器的“等待ACK队列”一般会维护一个超时计时器，一定时间内如果没有收到用户B发回的ACK包，就从“等待ACK队列”中重新拉取并进行重推。

为什么有了TCP协议本身的ACK机制，还需要业务层的ACK机制？ 这是因为TCP属于传输层，而IM服务属于应用层。 TCP的ACK保证网络传输层的可靠性，即消息是否送达，但不能保证数据能够被应用层正确可靠处理；业务层ACK进行消息是否送达和是否正确处理的逻辑，达到不丢消息、消息不重复的目的。

## 消息完整性检查

在上面列举的丢失消息的第 4 种可能性中，如果步骤 4 IM服务器将消息推送出去后就宕机了，而这条消息又因为某些原因丢失了，服务器由于宕机无法触发重传机制，导致用户B收不到该消息。可以采取“时间戳比对”机制进行完整性检查。

![](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409091632959.jpeg)

消息去重
-----

消息重复的原因 在上面列举的丢失消息的几种可能性中，第 3 种可能性存在一种场景，步骤 4 将消息成功推送给用户B，但步骤 3 因为某些原因导致超时、用户A收不到响应，这个时候会触发重传机制，用户A重新发送请求，用户B可能会收到重复消息。

消息重复的解决方案 IM服务器推送消息时，携带一个Sequence ID，这个Sequence ID在本次连接会话中唯一，同时针对同一条消息不变。当接收方接收到消息后，会根据这个Sequence ID来进行业务层的去重，可以有效地保证消息的不重复。

小结
--

- 消息丢失：业务层的ACK机制、重传机制和完整性检查
- 消息重复：客户端的去重机制


