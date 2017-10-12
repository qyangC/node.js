1.指明依赖项

`package.json`:

```json
{
  "name":"chatrooms",
  "version":"0.0.1",
  "description": "Minimalist multiroom chat server",
  "dependencies": {
    "socket.io": "~2.0.3",
    "mime":"~2.0.3"
  }
}
```

然后使用npm install进行安装，在node_modules中可以看到该程序依赖项。

2.变量声明(并且发送文件数据及错误响应)：

server.js:

```javascript
/**
 * Created by 180 on 2017/10/10.
 */
var http=require('http');//内置的http模板提供了HTTP服务器和客户端功能
var fs=require('fs');//内置的path模板提供了与文件系统路径相关的功能
var path=require('path');//内置的http模板提供了HTTP服务器和客户端功能
var mime=require('mime');//附加的mime模板有根据文件扩展名得出MIME类型的能力
var cache={};//cache是用来缓存文件内容的对象
function send404(response) {
    response.writeHead(404,{'Content-Type':'text/plain'});
    response.write('Error 404:resource not found.');
    response.end();
}//所请求的文件不存在时发出404错误。
function sendFile(response,filePath,fileContents) {
    response.writeHead(
        200,
        {'Content-Type':mime.lookup(path.basename(filePath))}
    );
    response.end(fileContents);
}//提供文件数据服务.
function serveStatic(response,cache,absPath) {
    if(cache[absPath]){         //检查文件是否缓存在内存中
        sendFile(response,cache,cache[absPath])//从内存中返回文件
    }
    else{
        fs.exists(absPath,function (exists) {//检查文件是否存在
            if(exists){
                fs.readFile(absPath,function (err,data) {//从硬盘中读取文件
                    if(err){
                        send404(response);
                    }else{
                        cache[absPath]=data;
                        sendFile(response,absPath,data);//从硬盘中读取文件并返回
                    }
                })
            }
            else{
                send404(response);//发送HTTP 404响应
            }
        })
    }
}//Node程序通常会把常用的数据缓存到内存里。这个函数是为了确认文件是否缓存，
// 如果是，就返回它。如果文件还没被缓存，它会从硬盘中读取并返回它.
//如果文件不存在，则返回一个HTTP 404错误作为响应.
```

3.创建HTTP服务器

server.js:

```javascript
var server=http.createServer(function (request,response) {//创建HTTP服务器，用匿名函数定义对每个请求的处理行为
    var filePath=false;
    if(request.url=='/'){
        filePath='public/index.html';//确定返回的默认HTML文件
    }else {
        filePath='public'+request.url;//将URL路径转为文件的相对路径
    }
    var absPath='./'+filePath;
    serveStatic(response,cache,absPath);//返回静态文件
});
```

4.启动HTTP服务器

```javascript
server.listen(4000,function () {
    console.log("server listening on post 4000.");
});
```

访问http://127.0.0.1:4000/public/index.html，页面上显示“Error 404:resource not found.”,已经添加了静态文件处理逻辑，但还没添加那些静态文件.

5.添加静态文件

index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Chat</title>
    <link rel="stylesheet" href="./stylesheets/style.css">
</head>
<body>
<div id="content">
    <div id="room"></div>
        <div id="room-list"></div>
        <div id="messages"></div>
    <form id="send-form">
        <input id="send-messsage" />
        <input type="submit" value="Send" id="send-button" />

        <div id="help">
            Chat commands:
            <ul>
                <li>Change nickname:<code>/nick [username]</code></li>
                <li>Join/create room:<code>/join [room name]</code></li>
            </ul>
        </div>
    </form>
</div>
<script src="/socket.io/socket.io.js"></script>
<script src="http://code.jquery.com/jquery-1.8.0.min.js"></script>
<script src="javascripts/chat.js"></script>
<script src="javascripts/chat_ui.js"></script>
</body>
</html>
```

style.css:

```css
body{
    padding: 50px;
    font:14px "Lucida Grande",Helvetica,Arial,sans-serif;
}
a{
    color: #00B7FF;
}
#content{
    width: 800px;
    margin-left: auto;
    margin-right: auto;
}
#room{
    background-color: #ddd;
    margin-bottom: 1em;
}
#messages{
    width: 690px;
    height: 300px;
    overflow: auto;
    background-color: #eee;
    margin-bottom: 1em;
    margin-right: 10px;
}
```

可以访问浏览器,看到基本的视觉布局搭建完成.

6.socket.io

建立连接处理逻辑chat_server.js:

```js
/**
 * Created by 180 on 2017/10/10.
 */
var socketio=require('socket.io');
var io;
var guestNumber=1;
var nicknames={};
var namesUsed=[];
var currentRoom={};

exports.listen=function (server) {
    io=socketio.listen(server);//启动Socket.IO服务器，允许打搭载已有的HTTP服务器上
    io.set('log level',1);
    io.sockets.on('connection',function (socket) {//定义每个用户连接的处理逻辑
        guestNumber=assignGuestName(socket,guestNumber,nicknames,namesUsed);
        //在用户连接上来时赋予其一个访问名
        joinRoom(socket,'Lobby');//在用户连接上来时把他放到聊天室Lobby里
        handleMessageBroadcasting(socket,nicknames);
        handleNameChangeAttempts(socket,nicknames,namesUsed);
        handleRoomJoining(socket);
        //处理用户消息更名，聊天室的创建和变更
        socket.io('rooms',function () {
            socket.emit('rooms',io.sockets.manager.rooms);
        });
        handleClientDisconnection(socket,nicknames,namesUsed);
    });
};
```

