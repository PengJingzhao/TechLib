![cover_image](https://mmbiz.qpic.cn/sz_mmbiz_jpg/aVSVOgASMhAGqESALlN3cJb3SbAFw7TsArGcWMYib56KdJeT9szrxX5S7S4WR19nXmKlHNkXxibnBhT8PyibCXOSw/0?wx_fmt=jpeg)

#  SpringBoot（十）整合WebSocket实时双向通信

原创  夏壹  [ 夏壹分享 ](javascript:void\(0\);)

__ _ _ _ _

项目中经常会用到消息推送功能，关于推送技术的实现，我们通常会联想到轮询，虽然这些技术能够实现。

但是需要反复连接，对于服务资源消耗过大，随着技术的发展，HtML5定义了WebSocket协议，能更好的节省服务器资源和带宽，并且能够更实时地进行通讯。

本文将介绍如何采用websocket实现消息推送。

##  WebSocket 概述

####  1\. 什么是WebSocket？

WebSocket 是一种在单个 TCP 连接上进行全双工通讯的协议。

它允许服务器主动向客户端推送信息，客户端也可以随时向服务器发送信息，实现了真正的双向实时通信。

这种通信方式特别适用于需要高频数据交换的Web应用程序，如实时聊天、在线游戏、实时数据监控等。

WebSocket 的工作原理大致如下：

  1. 1\. **握手阶段** ：WebSocket 连接是通过 HTTP/HTTPS 协议进行的初始握手来建立的。在这个阶段，客户端发送一个特殊的 HTTP 请求到服务器，请求中包含了升级协议的头部信息（如 ` Upgrade: websocket ` 和 ` Connection: Upgrade ` ），并指定了 WebSocket 版本（如 RFC 6455）。服务器解析这个请求，如果同意升级，就会响应一个状态码为 101 的 HTTP 响应，表示协议切换，然后连接就升级为了 WebSocket 连接。 

  2. 2\. **数据传输阶段** ：一旦 WebSocket 连接建立，客户端和服务器之间就可以开始通过 TCP 连接发送和接收数据了。WebSocket 传输的数据被封装在“帧”（Frame）中，每个帧都包含了数据和控制信息（如数据长度、掩码、数据载荷等）。WebSocket 支持文本（通过 UTF-8 编码的字符串）和二进制（如 Blob、ArrayBuffer）数据格式。 

  3. 3\. **连接保持** ：WebSocket 连接是持久的，除非被显式关闭（通过发送关闭帧或 TCP 连接关闭）。这种持久连接减少了网络延迟和带宽消耗，因为不需要频繁地建立和关闭连接。 

  4. 4\. **错误处理** ：WebSocket 协议也定义了一系列错误处理机制，以应对各种异常情况，如连接失败、数据错误、协议错误等。 

####  webSocket协议是什么？

Socket -> TCP连接。

webSocket -> 是基于 TCP 的一种新的网络协议。

它实现了浏览器与服务器全双工通信——浏览器和服务器只需要完成一次握手，两者之间就可以创建持久性的连接， 并进行双向数据传输。

####  2\. WebSocket 和 Socket 的区别

特性/方面  |  WebSocket  |  Socket   
---|---|---  
**应用层协议** |  基于HTTP协议，通过HTTP协议进行握手后，升级为WebSocket协议进行通信  |  没有明确的应用层协议，常用于传输TCP或UDP数据，需要开发者自行定义通信协议   
**传输数据类型** |  支持文本（如JSON、XML）和二进制类型的数据（如Blob、ArrayBuffer）  |  通常用于传输二进制数据，如图片、视频、文件等，但也可以传输文本数据   
**客户端和服务器通信** |  支持实时双向通信，一旦连接建立，双方可以即时发送和接收数据  |  支持双向通信，但通常需要通过轮询（Polling）或长连接（如HTTP长轮询、长轮询AJAX）来实现实时性，效率较低   
**传输效率** |  比传统的轮询技术更加高效和可靠，减少了不必要的网络请求和延迟，适合实时性要求高的应用  |  传输效率较低，特别是在需要频繁更新数据的情况下，因为需要不断发送请求或维持长连接，可能导致较高的服务器负载和客户端资源消耗   
**应用场景** |  适用于实时通信场景，如在线游戏、在线聊天、推送通知、实时监控、股票行情等  |  适用于需要传输大量二进制数据的场景，如文件传输、视频流、音频流、大型数据库同步等   
**连接管理** |  WebSocket协议提供了自动的连接管理功能，包括连接建立、保持和关闭  |  Socket连接的管理需要开发者自行处理，包括连接的建立、数据的发送接收、错误处理、连接关闭等   
**跨域支持** |  WebSocket原生支持跨域通信，但需要服务器端的CORS（跨源资源共享）支持  |  Socket连接通常受限于同源策略，跨域通信需要额外的设置或代理服务器   
**浏览器兼容性** |  大多数现代浏览器都支持WebSocket，但老旧浏览器可能不支持  |  Socket通信通常依赖于TCP/IP协议，与浏览器无关，但浏览器中的WebSocket API提供了在浏览器中使用Socket的方式   
**安全性** |  WebSocket支持通过WSS（WebSocket Secure）协议进行加密通信，类似于HTTPS  |  Socket通信的安全性取决于所使用的协议和加密措施，TCP/IP本身不提供加密，但可以通过SSL/TLS等协议进行加密   

###  3\. Spring Boot 集成 WebSocket

####  3.1 添加依赖

在 Spring Boot 项目中，需要添加 Spring Boot 的 WebSocket 支持依赖。通常使用 Spring Boot 的 `
spring-boot-starter-websocket ` 。


​    
```xml
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-websocket</artifactId>  
</dependency>
```

####  3.2 创建 WebSocket 配置类

配置 WebSocket 的端点（Endpoint）和消息处理器。


​    
```java
import org.springframework.context.annotation.Configuration;  
import org.springframework.messaging.simp.config.MessageBrokerRegistry;  
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;  
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;  
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;  
  
@Configuration  
@EnableWebSocketMessageBroker  
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {  
  
    @Override  
    public void registerStompEndpoints(StompEndpointRegistry registry) {  
        // 注册一个Stomp的端点，客户端可以通过这个端点来连接到WebSocket服务器  
        registry.addEndpoint("/websocket").withSockJS();  
    }  
  
    @Override  
    public void configureMessageBroker(MessageBrokerRegistry registry) {  
        // 配置消息代理的前缀  
        registry.enableSimpleBroker("/topic");  
        // 设置一个应用目标前缀，即发送消息到带有"/app"前缀的目的地时，消息会发送到消息代理，再由消息代理发送给订阅了相应主题的客户端  
        registry.setApplicationDestinationPrefixes("/app");  
    }  
}
```

####  3.3 创建消息处理器

处理来自客户端的消息和向客户端发送消息。


​    
```java
import org.springframework.messaging.handler.annotation.MessageMapping;  
import org.springframework.messaging.handler.annotation.SendTo;  
import org.springframework.stereotype.Controller;  
  
@Controller  
public class WebSocketController {  
  
    @MessageMapping("/hello")  
    @SendTo("/topic/greetings")  
    public String greeting(String message) throws Exception {  
        // 处理客户端发来的消息，并返回需要发送给客户端的消息  
        return "Hello, " + HtmlUtils.htmlEscape(message) + "!";  
    }  
}
```

####  3.4 服务器主动给客户端发送消息

服务器可以通过 ` SimpMessagingTemplate ` 来主动向客户端发送消息。


​    
```java
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.messaging.simp.SimpMessagingTemplate;  
import org.springframework.stereotype.Component;  
  
@Component  
public class NotificationService {  
  
    @Autowired  
    private SimpMessagingTemplate messagingTemplate;  
  
    public void sendNotification(String message) {  
        // 向订阅了/topic/notifications的客户端发送消息  
        messagingTemplate.convertAndSend("/topic/notifications", message);  
    }  
}
```

###  4\. 最后

Spring Boot 通过 ` spring-boot-starter-websocket ` 提供了对 WebSocket
的支持，通过简单的配置和注解即可实现全双工通信。

WebSocket 适用于需要实时数据交互的场景，可以有效提升用户体验。

在实际应用中，可以根据需要配置 WebSocket 的各种参数，如连接超时时间、消息大小限制等。
