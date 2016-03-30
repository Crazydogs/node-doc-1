## net

<div class="s s2"></div>

`net` 模块封装了一系列的异步网络操作，常用于创建服务器和客户端（也被称为 stream）。通过 `require('net')` 可以引用该模块。

## Class: net.Server

该类常用于创建 TCP 或本地服务器。

`net.Server` 是一个拥有以下事件的 `EventEmitter` 实例：

#### 事件：'close'

当服务器关闭时触发该事件。注意，如果存在连接，那么直到连接关闭才会触发该事件。

#### 事件：'connection'

- `net.Socket` 实例，连接对象

当创建新连接时触发该事件，其中 `socket` 是 `net.Socket` 的实例。

#### 事件：'error'

- `Error` 实例

当出现错误时触发该事件。发生该事件之后会立即触发 `close` 事件。

#### 事件：'listening'

当服务器通过 `server.listening` 绑定之后就会触发该事件。

#### server.address()

该方法返回一个对象，包含操作系统提供的绑定地址、地址族和服务器端口等信息。常用于查找那个端口已被系统绑定。该对象的常见格式：`{ port: 12346, family: 'IPv4', address: '127.0.0.1' }`。

```js
var server = net.createServer((socket) => {
  socket.end('goodbye\n');
}).on('error', (err) => {
  // handle errors here
  throw err;
});

// grab a random port.
server.listen(() => {
  address = server.address();
  console.log('opened server on %j', address);
});
```

在 `listening` 事件触发之前不要调用 `server.address()`。

#### server.close([callback])

该方法用于停止服务器接收新的连接请求并保持现有的连接。该方法是异步执行的，当服务器所有的连接都关闭之后就会触发 `close` 事件。可选参数 `callback` 在 `close` 事件触发后立即被调用。当服务器执行 `close` 事件时没有处于运行状态，那么就会生成一个错误传递给 `close` 事件的回调函数的第一个参数。

#### server.connections

<div class="s s0">
使用 server.getConnections() 替代。
</div>

#### server.getConnections(callback)

该方法以异步执行的方法获取当前服务器的并发连接数，且只在 socket 发送给子进程之后才有效。

回调函数接受 `err` 和 `count` 两个参数。

#### server.listen(handle[, backlog][, callback])

- `handle`，对象
- `backlog`，数值
- `callback`，函数

`handle` 对象可以是一个 server、socket（拥有 `_handle` 属性）或 `{fd: <n>}` 对象。

该方法让服务器根据指定的 `handle` 接收连接，但是系统会假定文件描述符或 handle 已经绑定给了端口或域 socket。

在 Windows 平台上不支持对文件描述符的监听。

该方法是以异步方式执行的。当服务器绑定以后，就会触发 `listening` 事件。参数 `callback` 会被系统绑定为 `Listening` 事件的处理器。

参数 `backlog` 的行为与 `server.listen(port[, hostname][, backlog][, callback])` 中的 `backlog` 一致。

#### server.listen(options[, callback])

- `options`，对象，包含以下属性
    - `port`，数值
    - `host`，字符串
    - `backlog`，数值
    - `path`，字符串
    - `exclusive`，布尔值
- `callback`，函数

`options` 对象中的 `port`、`host` 和 `backlog` 属性，以及可选参数 `callback`，它们的行为和 `server.listen(port[, hostname][, backlog][, callback])` 中相应的属性一致。此外，`path` 可用于指定一个 UXIN socket。

如果 `exclusive` 的值为 false（默认值），cluster worker 将会使用相同的底层 socket 句柄，并允许连接处理共享的任务；如果 `exclusive` 的值为 true，则不允许共享句柄，即使共享端口也会抛出错误。

```js
server.listen({
  host: 'localhost',
  port: 80,
  exclusive: true
});
```

#### server.listen(path[, backlog][, callback])

- `path`，字符串
- `backlog`，数值
- `callback`，函数

该方法根据指定的 `paht` 创建一个本地的 socket 服务器用于监听连接请求。

该方法是以异步方式执行的。当服务器绑定以后，就会触发 `listening` 事件。参数 `callback` 会被系统绑定为 `Listening` 事件的处理器。

在 UNIX 系统上，本地域名通常是 UNIX 域名，路径是以文件的路径名。当使用该方法创建文件时需要遵循相关的命名约定和权限检查，并且在文件系统中可见，直到文件被取消关联。

在 Windows 系统上，本地域名使用命名管道实现，路径必须以 `\\?\pipe\` 或 `\\.\pipe\` 开头，支持任意字符，但是稍后系统会对管道名做一些处理，比如解析 `..` 序列。除去这些表象之外，管道命名空间实际上是扁平的。管道并不会持久存在，当对它们的最后一个引用关闭后，管道就会被移除。一定不要忘记 JavaScript 字符串转义需要使用两个反斜线指定路径：

```js
net.createServer().listen(path.join('\\\\?\\pipe', process.cwd(), 'myctl'))
```

参数 `backlog` 的行为和 `server.listen(port[, hostname][, backlog][, callback])` 方法中相应的参数一致。

#### server.listen(port[, hostname][, backlog][, callback])

该方法根据指定的 `port` 和 `hostname` 接收连接。当没有传入 `hostname` 时，如果 IPv6 可用，那么服务器就会接收任何来自 IPv6 地址（`::`）的连接，否则接收 IPv4 地址（`0.0.0.0`）。如果 `port` 的值为 0，则系统分配一个随机端口。

`backlog` 参数指定连接等待队列的最大长度。该参数的实际值有操作系统的 sysctl 配置绝对，比如 Linux 上的 `tcp_max_syn_backlog` 和 `somaxconn`。该参数的默认是 511，而不是 502。

该方法是以异步方式执行的。当服务器绑定以后，就会触发 `listening` 事件。参数 `callback` 会被系统绑定为 `Listening` 事件的处理器。

某些用户在运行时可能会遇到 `EADDRINUSE` 错误，这通常是由于其他运行中的服务器占用了请求的端口。一种处理方法就是等会再次尝试运行：

```js
server.on('error', (e) => {
  if (e.code == 'EADDRINUSE') {
    console.log('Address in use, retrying...');
    setTimeout(() => {
      server.close();
      server.listen(PORT, HOST);
    }, 1000);
  }
});
```

注意，Node.js 中的所有 socket 都已经被设置了 `SO_REUSEADDR`。

#### server.listenning

该属性是一个布尔值，用于表示服务器是否正在监听连接请求。

#### server.maxConnections

设置该参数可以限制服务器的最大连接数。

对于 socket 向 `child_process.fork()` 创建的子进程发送消息这一情况，不建议设置该属性。

#### server.ref()

该方法是 `unref()` 方法的对立方法，在默认情况下，如果服务器已经调用了 `unref()`，那么在此服务器上调用 `ref()` 并不会结束程序。此外，反复调用 `ref()` 并没有额外的效果。

该方法返回一个 server 实例。

#### server.unref()

如果某个服务器是事件系统中唯一存在的服务器，那么在该服务器上调用 `unref` 将不会允许程序退出。此外，反复调用 `unref()` 并没有额外的效果。

该方法返回一个 server 实例。

## Class: net.Socket

该对象是对 TCP 或本地 socket 的抽象。`net.Socket` 的实例实现双工 Stream 接口。用户可以通过 `connect()` 方法创建该对象，将其视为一个客户端，或者由 Node.js 创建，由服务器的 `connection` 方法传递给用户。

#### new net.Socket([options])

该构造器用于创建一个初始化一个新的 socket 对象。

`options` 是一个包含以下默认值的对象：

```js
{
    fd: null,
    allowHalfOpen: false,
    readable: false,
    writable: false
}
```

`fd` 用于指定现有的 socket 文件描述符。将 `readable` 和 `writeable` 设置为 `true`，将允许对 socket 的读写，注意，只有当传入 `fd` 时才有效。`allowHalfOpen` 参数和 `createServer()` 和 `end` 事件有关。

`net.Socket` 是拥有以下事件的 `EventEmitter` 实例：

#### 事件：'close'

- `had_error`，布尔值，如果 socket 发生了传输错误，则该值为 `true`

当 socket 完全关闭时触发该事件，且只触发一次。参数 `had_error` 是一个布尔值，用于表示 socket 关闭时是否发生了传输错误。

#### 事件：'connect'

当 socket 连接建立成功时触发该事件。

#### 事件：'data'

- `Buffer` 实例

当接收到数据时触发该事件。参数 `data` 可以是 Buffer 实例或字符串，并通过 `socket.setEncoding()` 编码数据。

注意，如果 `Socket` 触发了 `data` 事件但没有给该事件设置监听器的时候，将会丢失数据。

#### 事件：'drain'

当写入的 buffer 为空时触发该事件。该事件可用于控制上传。

其他信息请参考 `socket.write()` 的返回值。

#### 事件：'end'

当 socket 的另一端发送 FIN 包时触发该事件。

默认情况下（`allowHalfOpen == false`），当 socket 将队列中的数据写完时就会销毁它的文件描述符。不过，如果设置 `allowHalfOpen == true` 了，则 socket 将不会自动调用 `end()`，而是允许用户随意写入数据，但是最终要开发者自己调用 `end()` 事件。

#### 事件：'error'

- `Error` 实例

当出现错误时就会触发该事件。该事件发生之后会立即触发 `close` 事件。

#### 事件：'lookup'

在主机名解析之后、连接建立之前触发该事件，且不适用于 UNIX socket。

- `err`，Error 实例或 Null，错误对象
- `address`，字符串，IP 地址
- `family`吗，字符串或 Null，地址类型

#### 事件：'timeout'

如果 socket 的空闲时间超时，则触发该事件，该事件只用于提醒 socket 处于空闲状态。开发者必须手动关闭连接。

#### socket.address()

该方法返回一个对象，包含操作系统提供的绑定地址、地址族和服务器端口等信息。常用于查找那个端口已被系统绑定。该对象的常见格式：`{ port: 12346, family: 'IPv4', address: '127.0.0.1' }`。

#### socket.bufferSize

该变量是 `net.Socket` 的一个属性，常用于 `socket.write()`，帮助用户获取更快的运行速度。由于网络连接有时候太慢，所以计算机不能一直高速向 socket 写入数据。Node.js 将排队数据写入 socket 并在网络可用时发送这些数据（轮询 socket 文件描述符直到变为可写）。

这种内部机制将会让内存占用增大，所以可以通过该属性控制缓存的字符数量（这里的字符串数量约等于准备写入的字节量，但在缓存中还有一些字符串，且这些字符串使用了惰性编码，所以实际的字节量是不可知的）。

如果开发者遇到了增长较快且较大的 `bufferSize`，那么应该尝试使用 `pause()` 和 `resume()` 方法对数据流进行节流操作。

#### socket.bytesRead

该属性表示接收到的字节量。

#### socket.bytesWritten

该属性表示发送出去的字节量。

#### socket.connect(options[, connectListener])

该方法为指定的 socket 打开连接。

对于 TCP socket，`options` 参数是一个对象，包含以下参数：

- `port`，客户端需要连接的端口
- `host`，客户端需要连接的主机，默认值为 `localhost`
- `localAddress`，为网络连接绑定的本地接口
- `localPort`，为网络连接绑定的本地端口
- `family`，IP 协议栈的版本，默认值为 `4`
- `lookup`，自定义的查找方法，默认值为 `dns.lookup`

对于本地域名的 socket，`options` 参数是一个对象，包含以下参数：

- `path`，客户端需要连接的路径。

通常来说用不到该方法，因为 `net.createConnection` 已经打开了 socket。只有开发者实现自定义的 Socket 时才需要使用该方法。

该方法是异步执行的。当 socket 建立之后就会触发 `connect` 事件。如果连接出现问题，就不会触发 `connect` 事件，而是触发 `error` 事件并抛出异常。

参数 `connectListener` 会被绑定为 `connect` 事件的监听器。

#### socket.connect(path[, connectListener])
#### socket.connect(port[, host][, connectListener])

该方法和 `socket.connect(options\[, connectListener\])` 类似，其中 `options` 可以是 `{port: port, host: host}` 或 `{path: path}`。

#### socket.destory()

该方法用于确保当前 socket 没有 I/O 活动，且只有在发生错误时才需要该方法。

#### socket.send([data][, encoding])

该方法用于半关闭 socket，比如发送 FIN 包。执行该方法后，socket 仍有可能发送数据。

如果指定了 `data`，那么该方法相当于先调用了 `socket.write(data, encoding)`，又调用了 `socket.end()`。

#### socket.localAddress

该属性是一个字符串，用于表示远程服务器连接到的本地 IP 地址。比如，如果你正在监听 `0.0.0.0`，而客户端链接到了 `192.168.1.1`，那么这个值就会变成 `192.168.1.1`。

#### socket.localPort

该属性是一个数值，表示本地端口，比如 `80` 和 `21`。

#### socket.pause()

该方法用于暂停读取数据，执行之后会触发 `data` 事件，常用于控制上传。

#### socket.ref()

该方法是 `unref()` 方法的对立方法，在默认情况下，如果服务器已经调用了 `unref()`，那么在此服务器上调用 `ref()` 并不会结束程序。此外，反复调用 `ref()` 并没有额外的效果。

该方法返回一个 socket 实例。

#### socket.remoteAddress

该属性是一个字符串，表示远程 IP 地址。比如 `'74.125.127.100'` 或 `'2001:4860:a005::68'`。如果 socket 已经被销毁，那么该属性的值可能是 undefined。

#### socket.remoteFamily

该属性是一个字符串，表示远程 IP 族，要么是 'IPv4'，要么就是 'IPv6'。

#### socket.resume()

该方法用户恢复调用了 `pause()` 方法的 socket 继续读取数据。

#### socket.setEncoding([encoding])

该方法用于设置作为可读 Stream 的 socket 的编码格式。

#### socket.setKeepAlive([enable][, initialDelay])

该方法用于开启或关闭 `keep-alive` 功能，其参数用于在存活探测分节（keepalive probe）第一次发送给空闲 socket 时，设置初始化的延迟时间。`enable` 的默认值为 `false`。

通过设置 `initialDelay` 可以控制接收到最后一个数据报和第一个存活探测分节之间的延迟时间。如果该参数的值wie 0，则系统会使用默认值或之前的值，默认值为 0。

该方法返回一个 socket 实例。

#### socket.setNoDelay([noDelay])

该方法用于禁用延迟传送算法。默认情况下，TCP 连接会使用延迟传送算法，会在数据发送之前缓存它们。如果 `noDelay` 的值为 `true`，则每次调用 `socket.write()` 时就会理解销毁数据。`noDelay` 的默认值为 `true`。

该方法返回一个 socket 实例。

#### socket.setTimeout(timeout[, callback])

该方法用于设置 socket 的空闲时间。默认情况下，`net.Socket` 没有超时限制。

当发生超时时，socket 会接收到一个 `timeout` 事件，但连接不会中断。开发者必须手动使用 `end()` 或 `destory()` 结束 socket。

如果 `timeout` 的值为 0，那么就会禁用闲置超时。

可选参数 `callback` 将会被绑定为 `timeout` 事件的监听器。

该方法返回一个 socket 实例。

#### socket.unref()

如果某个服务器是事件系统中唯一存在的服务器，那么在该服务器上调用 `unref` 将不会允许程序退出。此外，反复调用 `unref()` 并没有额外的效果。

该方法返回一个 socket 实例。

#### socket.write(data[, encoding][, callback])

该方法用于通过 socket 发送数据。第二个参数指定了字符串的编码格式，默认编码格式为 UTF8。

如果所有数据都成功刷新到了内核缓存中，该函数返回 `true`，如果所有的货部分数据加入到了内存中，则会返回 `false`。当缓存为空时，就会触发 `drain` 事件。

当所有数据都写完后，就会执行可选参数 `callback`，不过可能不是立即调用。

## net.connect(options[, connectListener])

该方法是一个工厂方法，会返回一个新的 `new.Socket` 实例并自动根据 `options` 参数进行连接。

`options` 会被传送 `net.Socket` 的构造器和 `socket.connect` 方法。

`connectListener` 参数会被绑定为 `connect` 事件的监听器。

下面的代码演示了之前描述的 echo 服务器的客户端：

```js
const net = require('net');
const client = net.connect({port: 8124}, () => {
  // 'connect' listener
  console.log('connected to server!');
  client.write('world!\r\n');
});
client.on('data', (data) => {
  console.log(data.toString());
  client.end();
});
client.on('end', () => {
  console.log('disconnected from server');
});
```

如果要连接到 `/tmp/echo.sock`，只需要修改第二行：

```js
const client = net.connect({path: '/tmp/echo.sock'});
```

## net.connect(path[, connectListener])

该方法是一个工厂方法，会返回一个新的 `new.Socket` 实例并自动根据 `path` 参数进行连接。

`connectListener` 参数会被绑定为 `connect` 事件的监听器。

## net.connect(port[, host][, connectListener])

该方法是一个工厂方法，会返回一个新的 `new.Socket` 实例并自动根据 `port` 和 `host` 参数进行连接。

如果未指定 `host` 参数，则会使用 `localhost`。

`connectListener` 参数会被绑定为 `connect` 事件的监听器。

## net.createConnection(options[, connectListener])

该方法是一个工厂方法，会返回一个新的 `new.Socket` 实例并自动根据 `options` 参数进行连接。

`options` 会被传送 `net.Socket` 的构造器和 `socket.connect` 方法。

`connectListener` 参数会被绑定为 `connect` 事件的监听器。

下面的代码演示了之前描述的 echo 服务器的客户端：

```js
const net = require('net');
const client = net.createConnection({port: 8124}, () => {
  //'connect' listener
  console.log('connected to server!');
  client.write('world!\r\n');
});
client.on('data', (data) => {
  console.log(data.toString());
  client.end();
});
client.on('end', () => {
  console.log('disconnected from server');
});
```

如果要连接到 `/tmp/echo.sock`，只需要修改第二行：

```js
const client = net.connect({path: '/tmp/echo.sock'});
```

## net.createConnection(path[, connectListener])

该方法是一个工厂方法，会返回一个新的 `new.Socket` 实例并自动根据 `path` 参数进行连接。

`connectListener` 参数会被绑定为 `connect` 事件的监听器。

## net.createConnection(port[, host][, connectListener])

该方法是一个工厂方法，会返回一个新的 `new.Socket` 实例并自动根据 `port` 和 `host` 参数进行连接。

如果未指定 `host` 参数，则会使用 `localhost`。

`connectListener` 参数会被绑定为 `connect` 事件的监听器。

## net.createServer([options][, connectionListener])

该方法用于创建新的服务器，其中，`connectListener` 参数会被绑定为 `connect` 事件的监听器。

`options` 参数是一个包含以下默认值的对象：

```js
{
    allowHalfOpen: false,
    pauseOnConnect: false
}
```

如果 `allowHalfOpen === true`，当 socket 的另一端发送了一个 FIN 包时，socket 不会自动发送 FIN 包。socket 会变成不可读但可写的状态。开发者需要显式调用 `end()`。

如果 `allowHalfOpen === true`，与 socket 有关的所有连接都会被暂停，它的 handle 也不会读取任何数据。它允许在原始进程不读取数据的情况下，让连接在进程之间传递。从暂停的 socket 读取数据之前需要先调用 `resume()`。

下面代码演示了一个监听 8124 端口的 echo 服务器：

```js
const net = require('net');
const server = net.createServer((c) => {
  // 'connection' listener
  console.log('client connected');
  c.on('end', () => {
    console.log('client disconnected');
  });
  c.write('hello\r\n');
  c.pipe(c);
});
server.on('error', (err) => {
  throw err;
});
server.listen(8124, () => {
  console.log('server bound');
});
```

通过 `talnet` 命令测试：

```js
telnet localhost 8124
```

如果要监听 `/tmp/echo.sock` socket，只需修改最后三行代码：

```js
server.listen('/tmp/echo.sock', () => {
  console.log('server bound');
});
```

使用 `nc` 连接到 UNIX 域名 socket 服务器：

```bash
nc -U /tmp/echo.sock
```

## net.isIP(input)

该方法用于测试 `input` 是否是一个 IP 地址。如果 `input` 是无效的的字符串，则返回 0，如果是 IPv4，则返回 4，如果是 IPv6，则返回 6。

## net.isIPv4(input)

如果 `input` 是 IPv4 的 IP 地址，则返回 true，否则返回 false。

## net.isIPv6(input)

如果 `input` 是 IPv6 的 IP 地址，则返回 true，否则返回 false。

<style>
.s {
    margin: 1.5rem 0;
    padding: 10px 20px;
    color: white;
    border-radius: 5px;
}
.s:before {
    display: block;
    font-size: 2rem;
    font-weight: 900;
}
.s0 {
    background-color: #C04848;
}
.s0:before {
    content: "接口稳定性: 0 - 已过时";
}
.s1 {
    background-color: #F07241;
}
.s1:before {
    content: "接口稳定性: 1 - 实验中";
}
.s2 {
    background-color: #457D97;
}
.s2:before {
    content: "接口稳定性: 2 - 稳定";
}
.s3 {
    background-color: #14C3A2;
}
.s3:before {
    content: "接口稳定性: 3 - 已锁定";
}
</style>