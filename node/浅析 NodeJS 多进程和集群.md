# 浅析 NodeJS 多进程和集群

### 进程

进程是指在系统中正在运行的一个应用程序。

当我们打开活动监视器或者文件资源管理器时，可以看到每一个正在运行的进程：

![CleanShot 2021-05-20 at 11.39.51@2x](http://makeryi-blog.oss-cn-hangzhou.aliyuncs.com/2021/05/21/cleanshot-20210520-at-1139512x.png)

### 多进程

#### 复制进程

NodeJS 提供了 `child_process` 模块，并且提供了 `child_process.fork()` 函数供我们复制进程。

举个🌰

在一个目录下新建 worker.js 和 master.js 两个文件：

worker.js
```js
const http = require('http');

http.createServer((req, res) => {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello NodeJS!\n');
}).listen(Math.round((1 + Math.random()) * 2000), '127.0.0.1');
```
master.js

```js
const { fork } = require('child_process');
const { cpus } = require('os');

cpus().map(() => {
  fork('./worker.js');
});
```

通过 `node master.js` 启动 master.js，然后通过 `ps aux | grep worker.js` 查看进程的数量，我们可以发现，理想状况下，进程的数量等于 CPU 的核心数，每个进程各自利用一个 CPU，实现多核 CPU 的利用：

![CleanShot 2021-05-20 at 18.16.13@2x](http://makeryi-blog.oss-cn-hangzhou.aliyuncs.com/2021/05/21/cleanshot-20210520-at-1816132x.png)

这是经典的 Master-Worker 模式（主从模式）
![CleanShot 2021-05-20 at 18.00.51@2x](http://makeryi-blog.oss-cn-hangzhou.aliyuncs.com/2021/05/21/cleanshot-20210520-at-1800512x.png)

实际上，fork 进程是昂贵的，复制进程的目的是充分利用 CPU 资源，所以 NodeJS 在单线程上使用了事件驱动的方式来解决高并发的问题。

#### 子进程的创建

`child_process` 模块提供了四个方法创建子进程：

* child_process.spawn(command, args)
* child_process.exec(command, options)
* child_process.execFile(file, args[, callback])
* child_process.fork(modulePath, args)

对比：

![CleanShot 2021-05-20 at 20.13.24@2x](http://makeryi-blog.oss-cn-hangzhou.aliyuncs.com/2021/05/21/cleanshot-20210520-at-2013242x.png)

后三种方法都是 `spawn()` 的延伸。

### 进程间的通信

在 NodeJS 中，子进程对象使用 `send()` 方法实现主进程向子进程发送数据，`message` 事件实现主进程收听由子进程发来的数据。

举个🌰

在一个目录下新建 parent.js 和 child.js 两个文件：

parent.js
```js
const { fork } = require('child_process');
const sender = fork(__dirname + '/child.js');

sender.on('message', msg => {
  console.log('主进程收到子进程的信息：', msg);
});

sender.send('Hey! 子进程');
```
child.js

```js
process.on('message', msg => {
  console.log('子进程收到来自主进程的信息：', msg);
});

process.send('Hey! 主进程');
```

当我们执行 `node parent.js` 时，会出现如下图所示：

![CleanShot 2021-05-20 at 20.41.21@2x](http://makeryi-blog.oss-cn-hangzhou.aliyuncs.com/2021/05/21/cleanshot-20210520-at-2041212x.png)

这样我们就实现了一个最基本的进程间通信。

#### IPC

IPC 即进程间通信，可以让不同进程之间能够相互访问资源并协调工作。
![CleanShot 2021-05-20 at 20.55.07@2x](http://makeryi-blog.oss-cn-hangzhou.aliyuncs.com/2021/05/21/cleanshot-20210520-at-2055072x.png)

实际上，父进程会在创建子进程之前，会先创建 IPC 通道并监听这个 IPC，然后再创建子进程，通过环境变量（NODE_CHANNEL_FD）告诉子进程和 IPC 通道相关的文件描述符，子进程启动的时候根据文件描述符连接 IPC 通道，从而和父进程建立连接。

![CleanShot 2021-05-20 at 21.02.58@2x](http://makeryi-blog.oss-cn-hangzhou.aliyuncs.com/2021/05/21/cleanshot-20210520-at-2102582x.png)


### 句柄传递

句柄是一种可以用来标识资源的引用的，它的内部包含了指向对象的文件资源描述符。

一般情况下，当我们想要将多个进程监听到一个端口下，可能会考虑使用主进程代理的方式处理：
![CleanShot 2021-05-20 at 21.20.01@2x](http://makeryi-blog.oss-cn-hangzhou.aliyuncs.com/2021/05/21/cleanshot-20210520-at-2120012x.png)

然而，这种代理方案会导致每次请求的接收和代理转发用掉两个文件描述符，而系统的文件描述符是有限的，这种方式会影响系统的扩展能力。

所以，为什么要使用句柄？原因是在实际应用场景下，建立 IPC 通信后可能会涉及到比较复杂的数据处理场景，句柄可以作为 `send()` 方法的第二个可选参数传入，也就是说可以直接将资源的标识通过 IPC 传输，避免了上面所说的代理转发造成的文件描述符的使用。

![CleanShot 2021-05-20 at 21.45.36@2x](http://makeryi-blog.oss-cn-hangzhou.aliyuncs.com/2021/05/21/cleanshot-20210520-at-2145362x.png)


以下是支持发送的句柄类型：

* net.Socket
* net.Server
* net.Native
* dgram.Socket
* dgram.Native

#### 句柄发送与还原

NodeJS 进程之间只有消息传递，不会真正的传递对象。

`send()` 方法在发送消息前，会将消息组装成 handle 和 message，这个 message 会经过 `JSON.stringify` 序列化，也就是说，传递句柄的时候，不会将整个对象传递过去，在 IPC 通道传输的都是字符串，传输后通过 `JSON.parse` 还原成对象。

#### 监听共同端口

上图所示，为什么多个进程可以监听同一个端口呢？

原因是主进程通过 send() 方法向多个子进程发送属于该主进程的一个服务对象的句柄，所以对于每一个子进程而言，它们在还原句柄之后，得到的服务对象是一样的，当网络请求向服务端发起时，进程服务是抢占式的，所以监听相同端口时不会引起异常。

### Cluster

引用 Egg.js 官方对 Cluster 的理解：

* 在服务器上同时启动多个进程。
* 每个进程里都跑的是同一份源代码（好比把以前一个进程的工作分给多个进程去做）。
* 更神奇的是，这些进程可以同时监听一个端口

其中：

* 负责启动其他进程的叫做 Master 进程，他好比是个『包工头』，不做具体的工作，只负责启动其他进程。
* 其他被启动的叫 Worker 进程，顾名思义就是干活的『工人』。它们接收请求，对外提供服务。
* Worker 进程的数量一般根据服务器的 CPU 核数来定，这样就可以完美利用多核资源。

```js
const cluster = require('cluster');
const http = require('http');
const numCPUs = require('os').cpus().length;

if (cluster.isMaster) {
  // Fork workers.
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', function(worker, code, signal) {
    console.log('worker ' + worker.process.pid + ' died');
  });
} else {
  // Workers can share any TCP connection
  // In this case it is an HTTP server
  http.createServer(function(req, res) {
    res.writeHead(200);
    res.end("hello world\n");
  }).listen(8000);
}
```

简单来说，`cluster` 模块是 `child_process` 模块和 `net` 模块的组合应用。

cluster 启动时，内部会启动 TCP 服务器，将这个 TCP 服务器端 socket 的文件描述符发给工作进程。

在 `cluster` 模块应用中，一个主进程只能管理一组工作进程，其运作模式没有 `child_process` 模块那么灵活，但是更加稳定：

![CleanShot 2021-05-20 at 22.10.35@2x](http://makeryi-blog.oss-cn-hangzhou.aliyuncs.com/2021/05/21/cleanshot-20210520-at-2210352x.png)

为了让集群更加稳定和健壮，`cluster` 模块也暴露了许多事件：

* fork
* online
* listening
* disconnect
* exit
* setup

这些事件在进程间消息传递的基础了完成了封装，保证了集群的稳定性和健壮性。

### 进程守护#

#### 未捕获异常

当代码抛出了异常没有被捕获到时，进程将会退出，此时 Node.js 提供了 `process.on('uncaughtException', handler)` 接口来捕获它，但是当一个 Worker 进程遇到未捕获的异常时，它已经处于一个不确定状态，此时我们应该让这个进程优雅退出：

* 关闭异常 Worker 进程所有的 TCP Server（将已有的连接快速断开，且不再接收新的连接），断开和 Master 的 IPC 通道，不再接受新的用户请求。
* Master 立刻 fork 一个新的 Worker 进程，保证在线的『工人』总数不变。
* 异常 Worker 等待一段时间，处理完已经接受的请求后退出。


```
+---------+                 +---------+
|  Worker |                 |  Master |
+---------+                 +----+----+
     | uncaughtException         |
     +------------+              |
     |            |              |                   +---------+
     | <----------+              |                   |  Worker |
     |                           |                   +----+----+
     |        disconnect         |   fork a new worker    |
     +-------------------------> + ---------------------> |
     |         wait...           |                        |
     |          exit             |                        |
     +-------------------------> |                        |
     |                           |                        |
    die                          |                        |
                                 |                        |
                                 |                        |
```

#### OOM、系统异常

当一个进程出现异常导致 crash 或者 OOM 被系统杀死时，不像未捕获异常发生时我们还有机会让进程继续执行，只能够让当前进程直接退出，Master 立刻 fork 一个新的 Worker。


## 参考资料

* 《深入浅出 Node.js》
* [Node.js 中文文档](http://nodejs.cn/api/)
* [Egg.js 官方文档](https://eggjs.org/zh-cn/core/cluster-and-ipc.html)