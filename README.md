t
学习－－－打造聊天室

用Socket.io制作实时聊天室

WebSockets代码

HTTP/长轮询/WebSockets

传统的HTTP协议
客户端没发送请求给服务器之前，服务器是不会主动作出响应的，客户端发送请求后服务器才作出响应，而且是单向的
如果要持续获取资源地不断重复请求，所以后来就有了长轮询的方法

长轮询(Long polling)：(即通过定时器，定时请求接口，获取到最新的信息后，将内容更新到页面中)客户端让HTTP请求保持一段时间，也就是保持持久连接，不过这里就会出问题，服务器不得不腾出资源来给这个长轮询，即使没有数据的时候也得保持连接
 

 然后就有了webSockets
  WebSockets是专门设计来做"实时"应用的协议，WebSockets在开始的时候依旧使用HTTP,只不过后面保持TCP持久连接

WebSockets的请求URI用的是ws或者wss
"ws:""//"host[":"port]path["?"query]
"wws:""//"host[":"port]path["?"query]

###  原理
客户端如果要发起WebSocket请求，就需要在请求首部里作出说明
Connection 的值写成Upgrade
UPgrade的值写成websocket
服务器在收到请求后就会连接要求升级，要升级成什么直接在Upgrade里找，结果找到websocket
于是服务器就知道客户端要建立WebSocket连接
另外客户端还会在请求里加入Sec-WebSocket-Key.这个Key提供服务器来验证是否收到一个有效的WebSockets请求,还有Sec-WebSocket-Version(版本号)

当服务器作出响应的时候，需要发出101 Switching Protocols,毕竟还没正式作出响应，所以是1开头的状态码，相应里的Connection 和Upgrade首部值是和请求一样的，来表明验证了连接已经被升级了
接着还会在响应里加上Sec-WebSocket-Accept,它的值是根据请求里key的值来生成的。
Sec-WebSocket-Accept表示服务器同意建立连接，这里要注意不同的客户端环境会导致请求首部的不同
建立连接后就是正式传输数据了

``` server.js
const WebSocket = require('ws');  //引用刚刚安装的模块
const wss  =  new WebSocket.Server({port:3000}); //生成一个WebSockets实例,并指明使用的端口
//单一连接ws
wss.on('connection',ws => {
	console.log('有个人进来了');
//当前端打开页面，后端的控制台提示"有个人进来了"

//数据传输
   ws.on('message',data => {
		ws.send(data + '举头望明月，低头思故乡,')
	 })



//当前端页面关闭，后端的控制台提示"ta又走了！呜呜呜"
	ws.on('close',() => {
		console.log('ta又走了！呜呜呜')
	})
})

```

```index.html
<script>
			const ws = new WebSocket('ws://localhost:3000');
			ws.addEventListener('open',()=>{
				console.log('连接上服务器了');

				ws.send('床前明月光，原来是惠康;'); //发送数据给服务器
			})
			//当html打开后，查看控制台，打印出"连接上服务器了”
			//前端和后端的控制台都有提示了
			
			//监听message，
			ws.addEventListener('message',({data}) => {
				console.log(data); //床前明月光，原来是惠康;举头望明月，低头思故乡
			})
		</script>

```

### Socket.io

```server.js
const app = require('express')();
const server = require('http').createServer(app);
const {Server} = require('socket.io');
const io = new Server(server);
//基于http模块来生成服务器实例，方便HTTP和HTTPS两个版本的写法
//而且还可以使用express框架的路径写法

//当接收到请求的时候,就把html文件作为响应的文件
app.get('/',(req,res) => {
	res.sendFile(__dirname + '/index.html');
});

io.on('connection',socket => {
	console.log('有个connection进入了聊天室');

	socket.on('chat message',msg => {
		console.log(msg);
		//在chat message 事件下用io实例来触发一下chat message
		//并且把从客户端接收到的数据发送出去
		io.emit('chat message',msg);

	})

	socket.on('disconnect',()=>{
		console.log('connection离开了')
	});
});

server.listen('3000',() => '服务器开启');

```
```index.html
<!DOCTYPE html>
<html>
	<head>
		<meta charset="UTF-8">
		<meta http-equiv="X-UA-Compatible" content="IE=edge">
		<meta name="viewport" content="width=device-width,initial-scal=1.0">
		<title>Document</title>
		<style>
			body {
				background: #f2f2f2;
			}
			form {
				width: 98%;
				display: flex;
			}
			input{
				width: 100%;
				cursor: pointer;
				font-size: 20px;
			}
			ul {
				width: 98%;
				list-style-type: none;
				padding: 0;
				margin-top: 10px;
			}
			li {
				margin: 5px;
				padding: 5px;
				color: #00B4CC;
				font-size: 20px;
				background-color: white;
			}
		</style>
	</head>
	<body>
		<form>
			<input type="text">
			<button>发送</button>
		</form>
		<ul></ul>
		<!-- //在前端的JS部分生成WebSocket实例 -->
		<!-- 首先引入socket.io客户端脚本，这个地方拉的是服务器，不是静态资源， -->
		<script src="/socket.io/socket.io.js"></script>
		<script>
			const socket = io();
			const form = document.querySelector('form');
			const input = document.querySelector('input');
			const ul = document.querySelector('ul');

			form.addEventListener('submit',e => {
				//如果有提交动作的时候，先取消默认动作防止刷新页面
				//如果输入框有内容的时候，就触发chat message事件
				e.preventDefault();
				if(input.value) {
					socket.emit('chat message',input.value);
					//客户端发送消息给服务器,服务器收到后还得广播给所有人
					//这样才能更新聊天室的内容
					input.value = '';  
				}
			});

			//如果服务器发来了chat message 的数据，就生成新的li元素，把数据内容放入li元素里面
			socket.on('chat message',msg => {
				const li = document.createElement('li');
				li.textContent = msg;
				ul.appendChild(li);
			})
			//然后打开多个页面，输入内容，其他页面也会有同步的消息
		</script>
	</body>

</html>

```



