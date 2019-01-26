# engine.io 原理详解

> 最近，业务中有使用到 `socket.io`，进行客户端与服务端的实时通信。`socket.io`提供的`API`易上手，对新手友好，这就极大提高了开发者的效率。不过，期间也有遇到很多`socket.io`中的坑，例如，[中文乱码问题](https://cnodejs.org/topic/55fcd7ed152fdd025f0f4eb7)，[服务端NPE问题](https://github.com/mrniko/netty-socketio/pull/603) 等。有些涉及到底层的问题，就势必要理解`socket.io`设计原理，进行排查。所以，总结了一下相关的概念，方便今后更快定位问题。

[socket.io 官网](https://socket.io/)

## 1.概述
`socket.io` 是基于 [Websocket](https://html.spec.whatwg.org/multipage/web-sockets.html#the-websocket-interface) 的Client-Server 实时通信库。

`socket.io`  底层是基于[engine.io](https://github.com/socketio/engine.io)这个库。

因此，在介绍`socket.io`之前，先简单的介绍一些关于`engine.io`相关的知识，方便深入理解。

**依赖关系详见:**
 - [package.json](https://github.com/socketio/socket.io/blob/master/package.json)


## 2. Engine.io 基础

`engine.io` 为 `socket.io`  提供跨浏览器/跨设备的双向通信的底层库。`engine.io` 使用了 `Websocket` 和 `XHR` 方式封装了一套 `socket` 协议。 在低版本的浏览器中，不支持`Websocket`，为了兼容使用长轮询(**polling**)替代。

**相关源码如下:**

- [兼容判断逻辑](https://github.com/socketio/engine.io/blob/master/lib/server.js#L299)
- [polling-jsonp](https://github.com/socketio/engine.io/blob/master/lib/transports/polling-jsonp.js)
- [polling-xhr](https://github.com/socketio/engine.io/blob/master/lib/transports/polling-xhr.js)
- [websocket](https://github.com/socketio/engine.io/blob/master/lib/transports/websocket.js)


## 3. Engine.io 工作流程

**Client**

```javascript
import eio from './engine.io-client'

// 创建一个socket长连接
let socket = new eio.Socket('ws://localhost');
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190126193704226.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMyNDMzNDc=,size_16,color_FFFFFF,t_70)

根据流程图，可以看出:

- 创建长连接的方式有三种: `websocket`、`xhr`、`jsonp`。其中，后两种使用长轮询的方式进行模拟。
- 所谓的长轮询是指，客户端发送一次`request`，当服务端有消息推送时会push一条`response`给客户端。客户端收到`response`后，会再次发送`request`，重复上述过程，直到其中一端主动断开连接为止。


## 4. webSocket 请求头信息 

**下图是建立成功的socket长连接：**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190126194725643.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMyNDMzNDc=,size_16,color_FFFFFF,t_70)

**参数说明**
- `Request URL` 请求服务端地址
- `Request Method` 请求方式 (支持get/post/option)
- `Status Code` 101 Switching Protocols 

[RFC 7231 规范定义](https://tools.ietf.org/html/rfc7231#section-6.2.2)
>规范解释: 当收到101请求状态码时，表明服务端理解并同意客户端请求，更改`Upgrade` header字段。服务端也必须在`response`中，生成对应的`Upgrade`值。

- `Connection` 设置`upgrade` header,通知服务端，该`request`类型需要进行升级为`websocket`。
[upgrade_mechanism 规范](https://developer.mozilla.org/en-US/docs/Web/HTTP/Protocol_upgrade_mechanism)
- `Host` 服务端 hostname
- `Origin` 客户端 hostname:port
- `Sec-WebSocket-Extensions` 客户端向服务端发起请求扩展列表(list)，供服务端选择并在响应中返回
- `Sec-WebSocket-Key` 秘钥的值是通过规范中定义的算法进行计算得出，因此是不安全的，但是可以阻止一些误操作的websocket请求。
- `Sec-WebSocket-Protocol`  指定有限使用的Websocket协议，可以是一个协议列表(list)。服务端在`response`中返回列表中支持的第一个值。
- `Sec-WebSocket-Version`  指定通信时使用的Websocket协议版本。最新版本:13,[历史版本](https://www.iana.org/assignments/websocket/websocket.xml#version-number)
- `Upgrade` 通知服务端，指定升级协议类型为`websocket`

## 5. engine.io 协议解析

- 客户端通过`engine.io`的url建立通信连接
- 服务端在`response`中返回一个`open`的packet，JSON编码数据格式如下:
       1. **sid**: session id （`String`）
       2. **upgrades**:  传输类型(`Array`)
       3. **pingTimeout**:  服务端通信超时配置，客户端用于超时检测(`Number`)
       4. **pingInterval**:  服务端通信定时器配置，客户端用于超时检测(`Number`)
- 收到客户端发送的`ping` packets时，服务端必须定时发送`pong` packets
- 客户端与服务端可以随意交换`message` pakcets
- `Polling`传输可以发送一个`close` pakcet来关闭`socket`,因为他们可能会一直`opening`或`closing`

## 6. URLs

`engine.io` URL的组成如下:

> `/engine.io/[?\<query string>]`

- **engine.io**: 只允许由库自身进行修改
- **query string**: 可选字段，并提供了四个保留字段:
		1. **transport**: 声明传输方式，可选值:<`polling`，`websocket`>
		2. **j**: 如果传输方式为`polling`，但是需要`JSONP`的响应，则`j`必须设置为`JSONP`响应的`index`
		3. **sid**: 如果客户端已经分配了一个`session id`,则`sid`必须包含在**query string**中
		4. **b64**: 如果客户端不支持`XHR2`,则必须在**query string**中加上`b64=1`标识，以通知服务端所有二进制数据应该进行base64编码

## 7. 编码方式
`engine.io`有两种编码方式:

- packet
- payload

**Packet**

编码包可以是**UTF8**或**二进制**数据，编码格式如下:
> \<**包类型id**>[\<**data**>]

例如:
> 2probe

包类型id(packet type id)是一个整型，具体含义如下:

- **0 open**
当打开一个新传输时，服务端检测并发送
- **1 close**
请求关闭传输，但不是主动断开连接
- **2 ping**
客户端发出，服务端应该返回包含相同数据的`pong` packet进行应答
- **3 pong**
服务端发出，用以响应客户端的`ping` packet
- **4 message**
真实数据，客户端和服务端应该调用回调中的`data`

```javascript
// 服务端发送 
send('4HelloWorld')
// 客户端接收数据并调用回调 
socket.on('message', function (data) { console.log(data); });
// 客户端发送 
send('4HelloWorld')
// 服务端接收数据并调用回调 
socket.on('message', function (data) { console.log(data); })
```

- **5 upgrade**
 在`engine.io`切换传输之前，它会测试服务器和客户端是否可以通过此传输进行通信。如果此测试成功，客户端将发送升级数据包，请求服务器刷新旧传输上的缓存并切换到新传输。
- **6 noop**
`noop` packet。主要用于在收到传入的`websocket`连接时强制轮询周期。
	1. 客户端通过新的传输连接
	2. 客户端发送 `2send`
	3. 服务端接收并发送 `3probe`
	4. 客户端结束并发送 `5`
	5. 服务端刷新并关闭旧的传输连接并切换到新传输连接

**Payload**

`Payload`是绑定在一起的一系列编码分组。格式如下:

> \<length1>:\<packet1>[\<length2>:\<packet2>[...]]
- **length**: 表示`packet`的字符长度
- **packet**: 真实数据包

## Transports
`engine.io`支持三种传输方式:

- `websocket`
- `polling`
	- `jsonp`
	- xhr

### Polling
长轮询传输包括客户端向服务器重复发出GET请求以获取数据，以及具有从客户端到服务器的有效负载的POST请求以发送数据。

**XHR**
服务端必须支持`CORS`

**JSONP**
服务器实现必须使用有效的**JavaScript**进行响应。 URL包含必须在响应中使用的**query string**参数`j`。 `j`是整数。

**JSONP**包的格式:
>___eio[\<`j`> ](" \<encoded payload> ");

例如:
>___eio[4]("packet data");


[官方文档](https://socket.io/docs/internals/)
