## Node.js 工作流程
1. 引入required 模块
2. 创建服务器
3. 接收请求并响应

运行`node helloworld.js`命令，能够在本地8888号端口建立http服务。
```javascript
var http = require("http");
http.createServer(function (request, response) {
    response.writeHead(200, {'Content-Type': 'text/plain'});
    response.end('Hello World\n');
}).listen(8888);

console.log('Server running at http://127.0.0.1:8888/');
```
## Node.js事件循环

```javascript
// 引入 events 模块，用来表示各种各样的请求
var events = require('events');
// 创建 eventEmitter 对象，作为请求的发送者
var eventEmitter = new events.EventEmitter();
 
// 创建事件处理程序，如果连接成功则在终端上返回结果，并触发eventEmitter的emit功能，表示data_received事件
var connectHandler = function connected() {
   console.log('连接成功。');
  
   // 触发 data_received 事件 
   eventEmitter.emit('data_received');
}
 
// 绑定 connection 事件处理程序
eventEmitter.on('connection', connectHandler);
 
// 使用匿名函数绑定 data_received 事件。data_received时间将会被这一行操作所绑定。
eventEmitter.on('data_received', function(){
   console.log('数据接收成功。');
});
 
// 触发 connection 事件 
eventEmitter.emit('connection');
 
console.log("程序执行完毕。");
```

## EventEmitter类型
events模块中的对象，EventEmitter，其核心是事件的触发和监听器功能的封装。

总体的工作流程实际上是异步非阻塞的，相当于烧开水的时候你可以看电视，等开水壶叫了，你再去把开水壶端过来。

下面这个函数阐述了EventEmitter的工作流程：绑定 + 触发，看上去挺硬件的。（当然，睡眠了一秒钟左右）。
```javascript
//event.js 文件
var EventEmitter = require('events').EventEmitter; 
var event = new EventEmitter(); 
event.on('some_event', function() { 
    console.log('some_event 事件触发'); 
}); 
setTimeout(function() { 
    event.emit('some_event'); 
}, 1000); 
```
EventEmitter支持多个事件的监听。
```javascript
//event.js 文件
var events = require('events'); 
var emitter = new events.EventEmitter();

emitter.on('someEvent', function(arg1, arg2) { 
    console.log('listener1', arg1, arg2); 
}); 

emitter.on('someEvent', function(arg1, arg2) { 
    console.log('listener2', arg1, arg2); 
}); 

emitter.emit('someEvent', 'arg1 参数', 'arg2 参数'); 
```
运行后可以看到两个事件的监听器回调函数先后调用（一个事件的两个监听器回调函数），这就是EventEmitter最简单的用法。

关于EventEmitter的其他API见菜鸟教程网站。

综合的实例，在code文件夹里头。很好理解。
### Error事件
在遇到异常时会触发Error事件，但是如果没有Error Handler，Node.js将会报错并退出，返回错误信息。

一般来说，我们要为会触发Error事件的对象设置监听器，避免遇到错误后整个程序奔溃。
```javascript
var events = require('events');
var emitter = new events.EventEmitter();
emitter.emit('error');
```
### 关于继承
Event模块实际上是很多模块的父亲，他们继承了Event的特性

## Node.js Buffer缓冲区
Javascript语言只有字符串数据类型，没有二进制数据类型，但是在处理TCP流或者文件流时，必须使用到二进制数据，因此在Node.js中定义了一个buffer类，用来存放二进制数据的缓存区。

其对应V8堆内存之外的一块原始内存。

看API上似乎没有特别复杂的，可以跳过。
## Node.js Stream
```javascript
var fs = require("fs");
var data = '';

// 创建可读流
var readerStream = fs.createReadStream('input.txt');

// 设置编码为 utf8。
readerStream.setEncoding('UTF8');

// 处理流事件 --> data, end, and error
readerStream.on('data', function(chunk) {
   data += chunk;
});

readerStream.on('end',function(){
   console.log(data);
});

readerStream.on('error', function(err){
   console.log(err.stack);
});

console.log("程序执行完毕");
```
最终的结果居然是：
```
PS D:\books\links_stored_notes\frontend\javascript\code\stream> node stream.js
程序执行完毕
山东菏泽曹县。
```
这说明read操作耗时很长，就这样。

主要来说，遵循data，end，error这样的模式就能够完成了。

