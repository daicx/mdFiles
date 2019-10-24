---
title: 基于WebSocket的研究
date: 2019-05-15 00:12:12
tags:
- websocket
categories:
- 通信机制
---



# 基于WebSocket的Stomp的研究

## WebSocket与Stomp

WebSocket是通过单个TCP连接提供全双工通信信道的计算机通信协议.实现的是客户端与服务端的双向通信.STOMP(Simple Text-Orientated Messaging Protocol) 面向消息的简单文本协议.

WebSocket处在TCP上非常薄的一层，会将字节流转换为文本/二进制消息，因此，对于实际应用来说，WebSocket的通信形式层级过低，因此，可以在 WebSocket 之上使用 STOMP协议，来为浏览器 和 server间的 通信增加适当的消息语义。

如何理解 STOMP 与 WebSocket 的关系： 
1) HTTP协议解决了 web 浏览器发起请求以及 web 服务器响应请求的细节，假设 HTTP 协议 并不存在，只能使用 TCP 套接字来 编写 web 应用，你可能认为这是一件疯狂的事情； 
2) 直接使用 WebSocket（SockJS） 就很类似于 使用 TCP 套接字来编写 web 应用，因为没有高层协议，就需要我们定义应用间所发送消息的语义，还需要确保连接的两端都能遵循这些语义； 

3) 同 HTTP 在 TCP 套接字上添加请求-响应模型层一样，STOMP 在 WebSocket 之上提供了一个**基于帧的线路格式层**，用来定义消息语义；

综上: 

1. TCP套接字(底层协议)->http(高层协议)
2. WebSocket(底层协议)->Stomp(高层协议)

stomp帧格式:

```
>>> SEND
destination:/app/chat
content-length:2

21
>>> SUBSCRIBE
id:sub-0
destination:/user/22/notifications

<<< MESSAGE
destination:/user/22/notifications
content-type:text/plain;charset=UTF-8
subscription:sub-0
message-id:kikxsuk1-0
content-length:4138
```



## 客户端API

### 1.引入js包

<script src="https://cdn.bootcss.com/stomp.js/2.3.3/stomp.min.js"></script>
<script src="https://cdn.bootcss.com/sockjs-client/1.1.4/sockjs.min.js"></script>

### 2.连接

```html
// 建立连接对象（还未发起连接）
var socket=new SockJS("/endpointChatServer");

// 获取 STOMP 子协议的客户端对象
var stompClient = Stomp.over(socket);

// 向服务器发起websocket连接并发送CONNECT帧
stompClient.connect(
    {},
function connectCallback (frame) {
    // 连接成功时（服务器响应 CONNECTED 帧）的回调方法
        document.getElementById("state-info").innerHTML = "连接成功";
        console.log('已连接【' + frame + '】');
        stompClient.subscribe('/topic/getResponse', function (response) {
            showResponse(response.body);
        });
    },
function errorCallBack (error) {
    // 连接失败时（服务器响应 ERROR 帧）的回调方法
        document.getElementById("state-info").innerHTML = "连接失败";
        console.log('连接失败【' + error + '】');
    }
);
```

说明:

1) socket连接对象也可通过WebSocket(不通过SockJS)连接.

```
var socket=new WebSocket("/endpointChatServer");
```

2) stompClient.connect()方法签名：

```
client.connect(headers, connectCallback, errorCallback);
```

其中 
headers表示客户端的认证信息，如：

```
var headers = {
  login: 'mylogin',
  passcode: 'mypasscode',
};
```

若无需认证，直接使用空对象 “{}” 即可；

connectCallback 表示连接成功时（服务器响应 CONNECTED 帧）的回调方法； 
errorCallback 表示连接失败时（服务器响应 ERROR 帧）的回调方法，非必须；

### 3.断开连接

若要从客户端主动断开连接，可调用 disconnect() 方法:

```
client.disconnect(function () {
   alert("See you next time!");
};
```

该方法为异步进行，因此包含了回调参数，操作完成时自动回调；

### 4.心跳机制

若使用STOMP 1.1 版本，默认开启了心跳检测机制，可通过client对象的heartbeat field进行配置（默认值都是10000 ms）：

```
client.heartbeat.outgoing = 20000;  // client will send heartbeats every 20000ms
client.heartbeat.incoming = 0;      // client does not want to receive heartbeats from the server
// The heart-beating is using window.setInterval() to regularly send heart-beats and/or check server heart-beats
```

### 5.发送消息

连接成功后，客户端可使用 send() 方法向服务器发送信息：

```
client.send(destination url, headers, body);
```

其中 
destination url 为服务器 controller中 @MessageMapping 中匹配的URL，字符串，必须参数； 
headers 为发送信息的header，JavaScript 对象，可选参数； 
body 为发送信息的 body，字符串，可选参数；

例:

```
//header里面的参数,在服务端用注解:@Headers Map<String,String> headers接收.
client.send("/queue/test", {priority: 9}, "Hello, STOMP");
//url里面的参数,服务端用注解:@DestinationVariable("appkey") String appkey接收.
client.send("/queue/test/{appkey}", {}, "Hello, STOMP");

```

6.消息的订阅

STOMP 客户端要想接收来自服务器推送的消息，必须先订阅相应的URL，即发送一个 SUBSCRIBE 帧，然后才能不断接收来自服务器的推送消息； 
订阅和接收消息通过 subscribe() 方法实现：

```
subscribe(destination url, callback, headers)

```

其中 
destination url 为服务器 @SendTo 匹配的 URL，字符串； 
callback 为每次收到服务器推送的消息时的回调方法，该方法包含参数 message； 
headers 为附加的headers，JavaScript 对象；什么作用？ 
该方法返回一个包含了id属性的 JavaScript 对象，可作为 unsubscribe() 方法的参数；

例:

```
var headers = {ack: 'client', 'selector': "location = 'Europe'"};
var  callback = function(message) {
  if (message.body) {
    alert("got message with body " + message.body)
  } else {
    alert("got empty message");
  }
});
var subscription = client.subscribe("/queue/test", callback, headers);

```

### 7.取消订阅

```
var subscription = client.subscribe(...);

subscription.unsubscribe();

```

### 8.JSON的支持

STOMP 帧的 body 必须是 string 类型，若希望接收/发送 json 对象，可通过 JSON.stringify() and JSON.parse() 实现； 
例：

```
var quote = {symbol: 'APPL', value: 195.46};
client.send("/topic/stocks", {}, JSON.stringify(quote));

client.subcribe("/topic/stocks", function(message) {
var quote = JSON.parse(message.body);
alert(quote.symbol + " is at " + quote.value);
});

```

9.事物支持

STOMP 客户端支持在发送消息时用事务进行处理： 
举例说明：

```
// start the transaction
// 该方法返回一个包含了事务 id、commit()、abort() 的JavaScript 对象
var tx = client.begin();
// send the message in a transaction
// 最关键的在于要在 headers 对象中加入事务 id，若没有添加，则会直接发送消息，不会以事务进行处理
client.send("/queue/test", {transaction: tx.id}, "message in a transaction");
// commit the transaction to effectively send the message
tx.commit();
// tx.abort();

```

### 10.消息接收确认

默认情况下，在将消息传递到客户机之前，服务器将自动确认STOMP消息。

客户端可以选择通过订阅目的地来处理消息确认，并将ack头集指定给客户端或单独的客户端。

在这种情况下，客户机必须使用message.ack()方法通知服务器它已经确认了消息。

```HTML
var subscription = client.subscribe("/queue/test",
  function(message) {
    // do something with the message
    ...
    // and acknowledge it
    message.ack();
  },
  {ack: 'client'}
);

```

## 服务端

### 1.引入依赖

```
//基于spring-boot-2.2.0.M2版本
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-websocket</artifactId>
</dependency>

```

### 2.加入配置文件

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketStompConfig extends AbstractWebSocketMessageBrokerConfigurer{
@Override
public void registerStompEndpoints(StompEndpointRegistry registry) {
	//为 /endpointChatServer 路径启用SockJS功能
	registry.addEndpoint("/endpointChatServer").setAllowedOrigins("*").withSockJS();
}
@Override
public void configureMessageBroker(MessageBrokerRegistry registry)
{	
	//表明在topic、queue、users这三个域上可以向客户端发消息。
	registry.enableSimpleBroker("/topic","/queue","/user");
    //客户端向服务端发起请求时，需要以/app为前缀。
	registry.setApplicationDestinationPrefixes("/app");
    //给指定用户发送一对一的消息前缀是/user。
	registry.setUserDestinationPrefix("/user");
}
}

```

### 3.消息传递的三种用例:

#### 客户端:

```
/*STOMP*/
var url = 'http://localhost:8080/stomp';
var sock = new SockJS(url);
var stomp = Stomp.over(sock);

var strJson = JSON.stringify({'message': 'Marco!'});

//默认的和STOMP端点连接
/*stomp.connect("guest", "guest", function (franme) {

});*/

var headers={
    username:'admin',
    password:'admin'
};

stomp.connect(headers, function (frame) {

    //发送消息
    //第二个参数是一个头信息的Map，它会包含在STOMP的帧中
    //事务支持
    var tx = stomp.begin();
    stomp.send("/app/marco", {transaction: tx.id}, strJson);
    tx.commit();


    //订阅服务端消息 subscribe(destination url, callback, headers)
    stomp.subscribe("/topic/marco", function (message) {
        var content = message.body;
        var obj = JSON.parse(content);
        console.log("订阅的服务端消息：" + obj.message);
    }, {});


    stomp.subscribe("/app/getShout", function (message) {
        var content = message.body;
        var obj = JSON.parse(content);
        console.log("订阅的服务端直接返回的消息：" + obj.message);
    }, {});


    /*以下是针对特定用户的订阅*/
    var adminJSON = JSON.stringify({'message': 'ADMIN'});
    /*第一种*/
    stomp.send("/app/singleShout", {}, adminJSON);
    stomp.subscribe("/user/queue/shouts",function (message) {
        var content = message.body;
        var obj = JSON.parse(content);
        console.log("admin用户特定的消息1：" + obj.message);
    });
    /*第二种*/
    stomp.send("/app/shout", {}, adminJSON);
    stomp.subscribe("/user/queue/notifications",function (message) {
        var content = message.body;
        var obj = JSON.parse(content);
        console.log("admin用户特定的消息2：" + obj.message);
    });

    //若使用STOMP 1.1 版本，默认开启了心跳检测机制（默认值都是10000ms）
    stomp.heartbeat.outgoing = 20000;

    stomp.heartbeat.incoming = 0; //客户端不从服务端接收心跳包
});

```

#### 3.1广播

```
@MessageMapping("/marco")
@SendTo("/topic/marco")
  public Shout stompHandle(Shout shout){
      LOGGER.info("接收到消息：" + shout.getMessage());
      Shout s = new Shout();
      s.setMessage("Polo!");
      return s;
  }
 //方式2
    @Autowired
    private SimpMessagingTemplate messagingTemplate;
  /**
  * 广播消息，不指定用户，所有订阅此的用户都能收到消息
  * @param shout
  */
  @MessageMapping("/broadcastShout")
  public void broadcast(String shout) {
      messagingTemplate.convertAndSend("/topic/shouts", shout);
  }

```

#### 3.2 点对点聊天

```
 @Autowired
    private SimpMessagingTemplate messagingTemplate;
 @MessageMapping("/chat")
    public void handleChat( String msg) {
        messagingTemplate
        .convertAndSendToUser("hjx", "/queue/notifications",
        		msg+ "-send:" + msg);
    }

```

#### 3.3 客户端与服务端的通信

前端

```
var sock = new SockJS("http://127.0.0.1:8080/endpointChatServer");
		var stomp = Stomp.over(sock);
		var headers = {
				platform : 'mylogin',
				name : 'mypasscode',
		};
		stomp.connect({}, function(frame) {
		//此处订阅了点对点的通信,注意22为返回用户id
			stomp.subscribe("/user/22/notifications", handleNotification)
		})
stomp.send("/app/appkey/chat-test/22", headers,reader.result)

```

后端

config:

```
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
	
	@Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
		//表明在topic、queue、users这三个域上可以向客户端发消息。
				registry.enableSimpleBroker("/topic","/user");
		        //客户端向服务端发起请求时，需要以/app为前缀。
				registry.setApplicationDestinationPrefixes("/app");
		        //给指定用户发送一对一的消息前缀是/user。
				registry.setUserDestinationPrefix("/user");
    }
 
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/endpointChatServer").setAllowedOrigins("*").withSockJS();
    }

    
}

```

使用:

```
@Autowired
	// 通过SimpMessagingTemplate模板向浏览器发送消息。如果是广播模式，可以直接使用注解@SendTo
private SimpMessagingTemplate simpMessagingTemplate;

@MessageMapping("/{appkey}/chat-test/{to}")
	public void handleChatTest(String message,@DestinationVariable("to") String to,@Headers Map<String, String> header) {
		System.out.println("开始chat");
		//此处的to对应上面的用户id,22
		simpMessagingTemplate.convertAndSendToUser( to,"/notifications",header+message);
		System.out.println("结束chat");
	}

```

