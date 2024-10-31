使用轮询&长轮询实现网页聊天室
===============================================================================================

前言
==

　　 如果有一个需求，让你构建一个网络的聊天室，你会怎么解决？

　　 首先，对于`HTTP`请求来说，`Server`端总是处于被动的一方，即只能由`Browser`发送请求，`Server`才能够被动回应。

　　 也就是说，如果`Browser`没有发送请求，则`Server`就不能回应。

　　 并且`HTTP`具有无状态的特点，即使有长链接（Connection请求头）的支持，但受限于`Server`的被动特性，要有更好的解决思路才行。

轮询
==

基本概念
----

　　 根据上面的需求，最简单的解决方案就是不断的朝`Server`端发送请求，`Browser`获取最新的消息。

　　 对于前端来说一般都是基于`setInterval`来做，但是轮询的缺点非常明显：

> 1.  Server需要不断的处理请求，压力非常大
> 2.  前端数据刷新不及时，setInterval间隔时间越长，数据刷新越慢，setInterval间隔时间越短，Server端的压力越大

　　 ![image-20201219204119137](https://images-1302522496.cos.ap-nanjing.myqcloud.com/img/image-20201219204119137.png)\>

示例演示
----

　　 以下是用`Flask`和`Vue`做的简单实例。

　　 每个用户打开该页面后都会生成一个随机名字，前端采用轮询的方式更新记录。

　　 后端用一个列表存储最近的聊天记录，最多存储100条，超过一百条截取最近十条。

　　 总体流程就是前端发送过来的消息都放进列表中，然后前端轮询时就将整个聊天记录列表获取到后在页面进行渲染。

　　 ![image-20201221140125610](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409071002285.png)

　　 缺点非常明显，仅仅有两个用户在线时，后端的请求就非常频繁了：

　　 ![image-20201221141051340](https://images-1302522496.cos.ap-nanjing.myqcloud.com/img/image-20201221141051340.png)

　　 后端代码：

```python
import uuid

from faker import Faker
from flask import Flask, request, jsonify

fake = Faker(locale='zh_CN')  # 生成随机名
app = Flask(__name__)

notes = []  # 存储聊天记录，100条
```


​    
```python
@app.after_request  # 解决CORS跨域请求
def cors(response):
    response.headers['Access-Control-Allow-Origin'] = "*"
    if request.method == "OPTIONS":
        response.headers["Access-Control-Allow-Headers"] = "Origin,Content-Type,Cookie,Accept,Token,authorization"
    return response
```


​    
```python
@app.route('/get_name', methods=["POST"])
def get_name():
    """
    生成随机名
    """
    username = fake.name() + "==" + str(uuid.uuid4())
    return jsonify(username)
```


​    
```python
@app.route('/send_message', methods=["POST"])
def send_message():
    """
    发送信息
    """
    username, tag = request.json.get("username").rsplit("==", maxsplit=1)  # 取出uuid和名字
    message = request.json.get("message")
    time = request.json.get("time")

    dic = {
        "username": username,
        "message": message,
        "time": time,
        "tag": tag + time,  # 前端:key唯一标识
    }

    notes.append(dic)  # 追加聊天记录

    return jsonify({
        "status": 1,
        "error": "",
        "message": "",
    })
```


​    
```python
@app.route('/get_all_message', methods=["POST"])
def get_all_message():
    """
    获取聊天记录
    """
    global notes
    if len(notes) == 100:
        notes = notes[90:101]
    return jsonify(notes)
```


​    
```python
if __name__ == '__main__':
    app.run(threaded=True)  # 开启多线程
```


​    

　　 前端代码`main.js`：

```javascript
import Vue from 'vue'
import App from './App.vue'
import router from './router'
import store from './store'
import axios from "axios"
import moment from 'moment'

Vue.prototype.$moment = moment
moment.locale('zh-cn')
Vue.prototype.$axios = axios;
```


​    
```javascript
Vue.config.productionTip = false

new Vue({
  router,
  store,
  render: h => h(App)
}).$mount('#app')
```


​    

　　 前端代码`Home.vue`：

```vue
<template>
  <div class="index">

    <div>{{ title }}</div>
    <article id="context">
      <ul>
        <li v-for="(v,index) in all_message" :key="index">
          <p>{{ v.username }}&nbsp;{{ v.time }}</p>
          <p>{{ v.message }}</p>
        </li>
      </ul>
    </article>
    <textarea v-model.trim="message" @keyup.enter="send"></textarea>
    <button type="button" @click="send">提交</button>
  </div>
</template>

<script>

export default {
  name: 'Home',
  data() {
    return {
      BASE_URL: "http://127.0.0.1:5000/",
      title: "聊天交流群",
      username: "",
      message: "",
      all_message: [],
    }
  },
  mounted() {
    // 获取用户名
    this.get_user_name();
    // 轮询，获取信息
    setInterval(this.get_all_message, 3000);
  },
  methods: {
    // 获取用户名
    get_user_name() {
      this.$axios({
        method: "POST",
        url: this.BASE_URL + "get_name",
        responseType: "json",
      }).then(response => {
        this.username = response.data;

      })
    },
    // 发送消息
    send() {
      if (this.message) {
        this.$axios({
          method: "POST",
          url: this.BASE_URL + "send_message",
          data: {
            message: this.message,
            username: this.username,
            time: this.$moment().format("YYYY-MM-DD HH:mm:ss"),
          },
          responseType: "json",
        });
        this.message = "";
      }
    },
    // 轮询获取消息
    get_all_message() {
      this.$axios({
        method: "POST",
        url: this.BASE_URL + "get_all_message",
        responseType: "json",
      }).then(response => {
        this.all_message = response.data;
        // 使用宏队列任务，拉滚动条
        let context = document.querySelector("#context");
          setTimeout(() => {
            context.scrollTop = context.scrollHeight;
          },)

      })
    },
  }
}
</script>

<style scoped>
* {
  margin: 0;
  padding: 0;
  list-style: none;
  box-sizing: border-box;
}

.index {
  display: flex;
  flex-flow: column;
  justify-content: flex-start;
  align-items: center;
}

.index div:first-child {
  margin: 0 auto;
  background: rebeccapurple;
  padding: 10px;
  display: flex;
  justify-content: center;
  align-items: center;
  color: aliceblue;
  width: 80%;
}

.index article {
  margin: 0 auto;
  height: 300px;
  border: 1px solid #ddd;
  overflow: auto;
  width: 80%;
  font-size: .9rem;
}

.index article ul li {
  margin-bottom: 10px;
}

.index article ul li p:last-of-type {
  text-indent: 1rem;
}

.index textarea {
  outline: none;
  resize: none;
  width: 80%;
  height: 100px;
  border: 1px solid #ddd;
  margin-bottom: 10px;
}

.index button {
  width: 10%;
  height: 30px;
  align-self: flex-end;
  transform: translate(-100%);
  background: forestgreen;
  color: white;
  outline: none;
}
</style>
```


长轮询
===

基本概念
----

　　 轮询是不断的发送请求，`Server`端显然受不了。

　　 这时候就可以使用长轮询的机制，即为每一个进入聊天室的用户（与`Server`端建立连接的用户）创建一个队列，每个用户轮询时都去询问自己的队列，如果没有新消息就等待，如果后端一旦接收到新消息就将消息放入所有的等待队列中返回本次请求。

　　 长轮询是在轮询基础上做的，也是不断的访问服务器，但是服务器不会即刻返回，而是等有新消息到来时再返回，或者等到超时时间到了再返回。

> 1.  Server端采用队列，为每一个请求创建一个专属队列
> 2.  Server端有新消息进来，放入每一个请求的队列中进行返回，或者等待超时时间结束捕获异常后再返回

　　 ![image-20201219204347371](https://images-1302522496.cos.ap-nanjing.myqcloud.com/img/image-20201219204347371.png)

示例演示
----

　　 使用长轮询实现聊天室是最佳的解决方案。

　　 前端页面打开后的流程依旧是生成随机名字，后端立马为这个随机名字拼接上uuid后创建一个专属的队列。

　　 然后每次发送消息时都将消息装到每个用户的队列中，如果有队列消息大于1的说明该用户已经下线，将该队列删除即可。

　　 获取最新消息的时候就从自己的队列中获取，获取不到就阻塞，获取到就立刻返回。

　　 后端代码：

```python
import queue
import uuid

from faker import Faker
from flask import Flask, request, jsonify

fake = Faker(locale='zh_CN')  # 生成随机名
app = Flask(__name__)

notes = []  # 存储聊天记录，100条

# 用户消息队列
user_queue = {

}

# 已下线用户
out_user = []
```


​    
```python
@app.after_request  # 解决CORS跨域请求
def cors(response):
    response.headers['Access-Control-Allow-Origin'] = "*"
    if request.method == "OPTIONS":
        response.headers["Access-Control-Allow-Headers"] = "Origin,Content-Type,Cookie,Accept,Token,authorization"
    return response
```


​    
```python
@app.route('/get_name', methods=["POST"])
def get_name():
    """
    生成随机名，还有管道
    """
    username = fake.name() + "==" + str(uuid.uuid4())
    q = queue.Queue()
    user_queue[username] = q  # 创建管道 {用户名+uuid:队列}
    return jsonify(username)
```


​    
```python
@app.route('/send_message', methods=["POST"])
def send_message():
    """
    发送信息
    """
    username, tag = request.json.get("username").rsplit("==", maxsplit=1)  # 取出uuid和名字
    message = request.json.get("message")
    time = request.json.get("time")

    dic = {
        "username": username,
        "message": message,
        "time": time,
    }

    for username, q in user_queue.items():
        if q.qsize() > 1:  # 用户已下线,五条阻塞信息，加入下线的用户列表中
            out_user.append(username)  # 不能循环字典的时候弹出元素
        else:
            q.put(dic)  # 将最新的消息放入管道中

    if out_user:
        for username in out_user:
            user_queue.pop(username)
            out_user.remove(username)
            print(username + "已下线，弹出消息通道")

    notes.append(dic)  # 追加聊天记录

    return jsonify({
        "status": 1,
        "error": "",
        "message": "",
    })
```


​    
```python
@app.route('/get_all_message', methods=["POST"])
def get_all_message():
    """
    获取聊天记录
    """
    global notes
    if len(notes) == 100:
        notes = notes[90:101]
    return jsonify(notes)
```


​    
```python
@app.route('/get_new_message', methods=["POST"])
def get_new_message():
    """
    获取最新的消息
    """
    username = request.json.get("username")
    q = user_queue[username]
    try:
        # 获取不到就阻塞,不立即返回
        new_message_dic = q.get(timeout=30)
    except queue.Empty:
        return jsonify({
            "status": 0,
            "error": "没有新消息",
            "message": "",
        })
    return jsonify({
        "status": 1,
        "error": "",
        "message": new_message_dic
    })
```


​    
```python
if __name__ == '__main__':
    app.run()
```


​    

　　 前端代码`main.js`：

```javascript
import Vue from 'vue'
import App from './App.vue'
import router from './router'
import store from './store'
import axios from "axios"
import moment from 'moment'

Vue.prototype.$moment = moment
moment.locale('zh-cn')
Vue.prototype.$axios = axios;

Vue.config.productionTip = false

new Vue({
  router,
  store,
  render: h => h(App)
}).$mount('#app')
```


　　 前端代码`Home.vue`：

```vue
<template>
  <div class="index">

    <div>{{ title }}</div>
    <article id="context">
      <ul>
        <li v-for="(v,index) in all_message" :key="index">
          <p>{{ v.username }}&nbsp;{{ v.time }}</p>
          <p>{{ v.message }}</p>
        </li>
      </ul>
    </article>
    <textarea v-model.trim="message" @keyup.enter="send" :readonly="status"></textarea>
    <button type="button" @click="send">提交</button>
  </div>
</template>

<script>

export default {
  name: 'Home',
  data() {
    return {
      BASE_URL: "http://127.0.0.1:5000/",
      title: "聊天交流群",
      username: "",
      message: "",
      status: false,
      all_message: [],
    }
  },
  mounted() {
    // 获取用户名
    this.get_user_name();

    // 异步队列，确认用户名已获取到
    setTimeout(() => {
      // 加载聊天记录
      this.get_all_message();
      // 长轮询
      this.get_new_message();

    }, 1000)

  },
  methods: {
    // 获取用户名
    get_user_name() {
      this.$axios({
        method: "POST",
        url: this.BASE_URL + "get_name",
        responseType: "json",
      }).then(response => {
        this.username = response.data;
      })
    },
    // 发送消息
    send() {
      if (this.message) {
        this.$axios({
          method: "POST",
          url: this.BASE_URL + "send_message",
          data: {
            message: this.message,
            username: this.username,
            time: this.$moment().format("YYYY-MM-DD HH:mm:ss"),
          },
          responseType: "json",
        });

        this.message = "";
      }
    },
    // 页面打开后，第一次加载聊天记录
    get_all_message() {
      this.$axios({
        method: "POST",
        url: this.BASE_URL + "get_all_message",
        responseType: "json",
      }).then(response => {
        this.all_message = response.data;
        // 控制滚动条
        let context = document.querySelector("#context");
        setTimeout(() => {
          context.scrollTop = context.scrollHeight;
        },)
      })
    },
    get_new_message() {
      this.$axios({
        method: "POST",
        // 发送用户名
        data: {"username": this.username},
        url: this.BASE_URL + "get_new_message",
        responseType: "json",
      }).then(response => {
        if (response.data.status === 1) {
          // 添加新消息
          this.all_message.push(response.data.message);
          // 控制滚动条
          let context = document.querySelector("#context");
          setTimeout(() => {
            context.scrollTop = context.scrollHeight;
          },)

        }
        // 递归
        this.get_new_message();
      })
    }
  }
}
</script>

<style scoped>
* {
  margin: 0;
  padding: 0;
  list-style: none;
  box-sizing: border-box;
}

.index {
  display: flex;
  flex-flow: column;
  justify-content: flex-start;
  align-items: center;
}

.index div:first-child {
  margin: 0 auto;
  background: rebeccapurple;
  padding: 10px;
  display: flex;
  justify-content: center;
  align-items: center;
  color: aliceblue;
  width: 80%;
}

.index article {
  margin: 0 auto;
  height: 300px;
  border: 1px solid #ddd;
  overflow: auto;
  width: 80%;
  font-size: .9rem;
}

.index article ul li {
  margin-bottom: 10px;
}

.index article ul li p:last-of-type {
  text-indent: 1rem;
}

.index textarea {
  outline: none;
  resize: none;
  width: 80%;
  height: 100px;
  border: 1px solid #ddd;
  margin-bottom: 10px;
}

.index button {
  width: 10%;
  height: 30px;
  align-self: flex-end;
  transform: translate(-100%);
  background: forestgreen;
  color: white;
  outline: none;
}
</style>
```

