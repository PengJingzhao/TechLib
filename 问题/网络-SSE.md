# SSE（Server-Sent Events）

`SSE` 是一种允许服务器单向发送事件到客户端的技术，它基于`HTTP`协议，服务器可以推送消息到客户端，但客户端不能向服务器发送消息。

# 实现

直接上代码

```java
@RestController
public class TestController {

    private static final Logger logger = LoggerFactory.getLogger(TestController.class);

    @GetMapping("/test")
    public void test(HttpServletResponse response) {
        response.setContentType("text/event-stream");
        response.setCharacterEncoding("utf-8");

        try (final PrintWriter writer = response.getWriter()) {
            // 要推送的内容
            final String content = "你好，我的朋友，快过年了，提前祝你新年快乐！";
            int len = content.length();
            int endIndex = 0;
            // 每隔2个字符推送一次，模拟打字机效果
            while (endIndex < len) {
                endIndex = Math.min(endIndex + 2, len);
                final String subContent = content.substring(0, endIndex);
                // 将要推送的内容封装成JSON格式，模拟实际开发中的数据格式，非必须
                final JSONObject json = new JSONObject();
                json.put("data", subContent);
                json.put("code", HttpStatus.OK.value());
                // 最后一次推送时，type为finish，表示推送结束，其它情况为add
                final String type = endIndex == len
                        ? "finish"
                        : "add";
                json.put("type", type);
                // 组装成SSE格式的数据，发送给前端，这个格式（data: content\n\n）是固定的，content是自定义的推送内容
                writer.write("data: " + json.toJSONString() + "\n\n");
                writer.flush();
                // 稍微给点停顿，防止数据发送太快，浏览器接收不过来
                TimeUnit.MILLISECONDS.sleep(100);
            }
        } catch (Exception e) {
            Thread.currentThread().interrupt();
            logger.error("流式推送数据异常", e);
        }
    }
}
```

组装数据，格式固定为`"data: " + content + "\n\n"`，这里的数据格式是服务器发送事件（`Server-Sent Events，SSE`）的标准格式。

在 `SSE` 中，数据必须以 `"data: "` 开头，然后是要发送的数据，最后是两个换行符`"\n\n"`。这是`SSE`的规定格式，客户端会根据这种格式来解析服务器发送的事件。

# 接口调试

用接口调试工具（我用的是`Apifox`）调试接口如下：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/403ad1855e2f4e1aa46d5097f6699aa8~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1497&h=894&s=89314&e=png&b=242424)

接口符合`SSE`规范，所以可以被正常识别为事件流推送。

# SSE与其他实时通信技术的比较

### SSE与WebSocket的比较

- 通信方式：SSE是单向的，只能由服务器向客户端发送数据；而WebSocket是双向的，服务器和客户端都可以发送数据。
- 协议：SSE基于HTTP协议，更易于设置和配置；WebSocket是一个独立的协议。
- 数据格式：SSE发送的数据格式固定，必须是"text/event-stream"；而WebSocket可以发送任何类型的数据。
- 连接：SSE在断开连接后可以自动重新连接，而WebSocket需要手动处理重连。
- 浏览器支持：WebSocket的浏览器支持更广泛，几乎所有现代浏览器都支持WebSocket；而SSE在某些旧版本的浏览器（如IE）中不被支持。

### SSE与长轮询的比较

- 效率：SSE更高效，因为它只需要一个HTTP连接，就可以持续地发送数据；而长轮询需要不断地建立和断开HTTP连接。
- 实时性：SSE的实时性更强，因为服务器可以随时发送数据；而长轮询需要客户端不断地发送请求来获取新数据。
- 复杂性：SSE的实现相对简单，只需要服务器按照规定的格式发送数据即可；而长轮询的实现较复杂，需要处理连接的建立和断开，以及错误和超时等问题。
- 浏览器支持：与WebSocket相比，SSE和长轮询的浏览器支持都较差，但长轮询在更多的浏览器中被支持。
- 适用场景：SSE适用于服务器需要主动推送数据的场景；而长轮询适用于客户端需要定期获取新数据，但服务器不需要主动推送数据的场景。

# 注意事项

- 以上一个简单的事件流推送接口就实现好了，但是它的问题是客户端没法干预服务器的推流，如果需要中途停止接收内容，基于以上接口是没法做到的。所以实际项目开发过程中，可以使用`Spring`框架提供的`SseEmitter`，具体用法可以参考 [轻松实现服务器事件推送！Spring SseEmitter 详解](https://juejin.cn/post/7250328942841495613) 。
- 浏览器支持：并非所有浏览器都支持`SSE`，例如，旧版本的Internet Explorer就不支持`SSE`。在使用`SSE`时，需要确保目标用户的浏览器支持这项技术。
- 连接限制：由于`SSE`需要保持长连接，因此可能会占用大量的服务器资源。在使用`SSE`时，需要考虑到这一点，并根据实际情况进行优化。