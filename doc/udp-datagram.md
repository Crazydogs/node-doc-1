## UDP / Datagram Sockets

<div class="s s2"></div>

`dgram` 模块封装了对 UDP Datagram sockets 的实现。

```js
const dgram = require('dgram');
const server = dgram.createSocket('udp4');

server.on('error', (err) => {
  console.log(`server error:\n${err.stack}`);
  server.close();
});

server.on('message', (msg, rinfo) => {
  console.log(`server got: ${msg} from ${rinfo.address}:${rinfo.port}`);
});

server.on('listening', () => {
  var address = server.address();
  console.log(`server listening ${address.address}:${address.port}`);
});

server.bind(41234);
// server listening 0.0.0.0:41234
```

## Class: dgram.Socket

`dgram.Socket` 是一个封装了和数据报相关的函数的 `EventEmitter` 实例对象。

使用 `dgram.createSocket()` 可以创建新的 `dgram.Socket` 实例，且不需要使用 `new` 关键字。

#### 事件：'close'

当 socket 使用 `close()` 关闭之后就会触发 `close` 事件。一旦触发该事件，socket 就不再触发 `message` 事件。

#### 事件：'error'

- `exception`，Error 实例

当出现任意错误时就会触发 `error` 事件，该事件的监听器函数只接收一个错误对象。

#### 事件：'listening'

当 socket 开始监听数据报信息时就会触发 `listening` 事件，且在 UDP socket 创建时就可以触发该事件。

#### 事件：'message'

- `msg`，Buffer 实例，消息
- `rinfo`，对象，远程地址信息

当 socket 接收到新的数据报时就会触发 `message` 事件。该事件的处理函数接收两个参数：`msg` 和 `rifno`，其中 `msg` 参数是一个 Buffer 实例，`rinfo` 参数是一个对象，该对象包含了发报者的地址属性 address/family/port：

```js
socket.on('message', (msg, rinfo) => {
  console.log('Received %d bytes from %s:%d\n', msg.length, rinfo.address, rinfo.port);
});
```

#### socket.addMembership(multicastAddress[, multicastInterface])

- `multicastAddress`，字符串
- `multicastInterface`，字符串

该方法根据给定的 `multicastAddress`（使用 `IP_ADD_MEMBERSHIP` socket 选项） 通知 kernel 加入广播组。如果未指定 `multicastInterface` 参数，操作系统会尝试为所有的有效网络接口添加广播关系。

#### socket.address()

该方法返回一个包含 socket 地址信息的对象。对于 UDP socket，该对象包含 address、family 和 port 属性。

#### socket.bind([port][, address][, callback])

- `port`，整数
- `address`，字符串
- `callback`，函数，不接收任何参数，完成绑定时调用

对于 UDP socket，该方法用于让 `dgram.Socket` 监听 `port` 和 `address` 指定接口的数据报信息。如果未指定 `port`，则操作系统绑定一个随机接口。如果未指定 `address`，则操作系统会尝试监听所有的地址。绑定完成之后会立即出发 `listening` 事件并调用 `callback` 回调函数。

注意，同时为 `socket.bind()` 设置 `listening` 事件监听器和 `callback` 回调函数虽然没有坏处，但略显多余。

socket 绑定数据报之后就会让 Node.js 进程持续运行以接收数据报信息。

如果绑定失败，则触发 `error` 事件。极少数情况下，会抛出一个 Error 实例，比如尝试绑定一个已关闭的 socket。

下面代码演示了监听 41234 端口的 UDP 服务器：

```js
const dgram = require('dgram');
const server = dgram.createSocket('udp4');

server.on('error', (err) => {
  console.log(`server error:\n${err.stack}`);
  server.close();
});

server.on('message', (msg, rinfo) => {
  console.log(`server got: ${msg} from ${rinfo.address}:${rinfo.port}`);
});

server.on('listening', () => {
  var address = server.address();
  console.log(`server listening ${address.address}:${address.port}`);
});

server.bind(41234);
// server listening 0.0.0.0:41234
```

#### socket.bind(options[, callback])

- `options`，对象，包含以下属性
    - `port`，数值
    - `address`，字符串
    - `exclusive`，布尔值
- `callback`，函数

对于 UDP socket，该方法用于让 `dgram.Socket` 监听 `options` 参数中 `port` 和 `address` 所指定的接口的数据报信息。如果未指定 `port`，则操作系统绑定一个随机接口。如果未指定 `address`，则操作系统会尝试监听所有的地址。绑定完成之后会立即出发 `listening` 事件并调用 `callback` 回调函数。

只有 `dgram.Socket` 配合 `cluster` 模块使用时，`options` 对象中的可选属性 `exclusive` 才有效。如果 `exclusive` 的值为 false（默认值），cluster worker 将会使用相同的底层 socket 句柄，并允许连接处理共享的任务；如果 `exclusive` 的值为 true，则不允许共享句柄，即使共享端口也会抛出错误。

下面代码演示了 socket 监听独享端口的示例：

```js
socket.bind({
  address: 'localhost',
  port: 8000,
  exclusive: true
});
```

#### socket.close([callback])

该方法用于关闭底层 socket 并停止监听数据。如果指定了 `callback`，则该回调函数会被系统自动添加为 `close` 事件的监听器。

#### socket.dropMembership(multicastAddress[, multicastInterface])

- `multicastAddress`，字符串
- `multicastInterface`，字符串

该方法根据给定的 `multicastAddress`（使用 `IP_ADD_MEMBERSHIP` socket 选项） 通知 kernel 离开广播组。当 socket 关闭或者进程终端时，kernel 会自动调用该方法，所以对于大多数的应用来说，基本不需要主动调用该方法。

如果未指定 `multicastInterface` 参数，操作系统会尝试为所有的有效网络接口移除广播关系。

#### socket.send(msg[, offset, length], port, address[, callback])

- `msg`，Buffer 实例或字符串或数组，用于发送的消息
- `offset`，整数，指定在 Buffer 实例中数据发送的起始位置
- `length`，整数，消息的字节量
- `port`，整数，分发端口
- `address`，字符串，分发主机名或 IP 地址
- `callback`，函数，消息发送之后调用的回调函数

该方法通过 socket 广播数据报，使用时必须指定 `port` 和 `address`。

`msg` 参数包含了发送的消息，且根据 `msg` 的类型不同，系统会做不同的处理。如果 `msg` 是一个 Buffer 实例，那么 `offset` 和 `length` 属性就用来指定数据发送的起始位置及长度。如果 `msg` 是一个字符串，则会被系统自动转换为 `utf8` 格式的 Buffer 实例。当消息中含有多字节字符时，`offset` 和 `length` 参数指定的就不再试字符位置了，而是字节长度。如果 `msg` 是一个数组，那么一定不要指定 `offset` 和 `length

`address` 参数是一个字符串。如果 `address` 是一个主机名，那么 DNS 就会解析该主机的地址；如果未指定 `address` 或者该参数是一个字符串，那么系统就会自动为其赋值为 `127.0.0.1` 或 `::1`。

如果 socket 之前没有通过 `bind()` 方法进行绑定，那么 socket 就会被分配一个随机接口并绑定到所有的接口地址，比如在 udp4 socket 中是 `0.0.0.0`，在 udp6 socket 中是 `::0`。

可选参数 `callback` 可用于记录 DNS 错误或指定何时复用 `buf` 对象。注意，DNS 查询会推迟消息发送的是时间，这段延迟多于 Node.js 事件循环一次的时间。

确定数据报被发送的唯一方式就是使用 `callback`。如果发送过程中出现了错误且指定了 `callback`，那么 `callback` 的第一个参数就包含了该错误。如果未指定 `callback`，那么在 `socket` 对象上就会触发 `error` 事件。

`offset` 和 `length` 是可选参数，但如果指定了其中之一，那么就必须指定另一个。此外，只有 `msg` 是 Buffer 实例或字符串时，才可以传递该参数。

下面代码演示了如何使用 `localhost` 上的随机端口发送 UDP 包：

```js
const dgram = require('dgram');
const message = new Buffer('Some bytes');
const client = dgram.createSocket('udp4');
client.send(message, 41234, 'localhost', (err) => {
  client.close();
});
```

下面代码演示了如何使用 `localhost` 上的随机端口发送包含多个 Buffer 实例的 UDP 包：

```js
const dgram = require('dgram');
const buf1 = new Buffer('Some ');
const buf2 = new Buffer('bytes');
const client = dgram.createSocket('udp4');
client.send([buf1, buf2], 41234, 'localhost', (err) => {
  client.close();
});
```

发送多个 Buffer 实例的快慢取决于开发者的应用程序和操作系统，所以开发者需要测试一下相关的性能。通常来说，这种方式还是比较快的。

#### 关于 UDP 数据报的长短

一个 `IPv4/6` 数据报的最大长度取决于 `MTU`（Maximum Transmission Unit）和 `Payload Length` 的值：

- `Payload Length` 的长度为 16 个字节，这意味着一个包含网络头信息和数据（65,507 bytes = 65,535 − 8 bytes UDP header − 20 bytes IP header）的有效负荷不能超过 64K。这对于环回接口（loopback interfaces）没有什么问题，但是对于大多数的主机和网络就不那么够用了。
- `MTU` 是链路层对数据报消息支持的最大长度。对于 IPv4 连接，无论是完整接收还是碎片接收，允许的 MTU 最小值是 68 个八位字节，IPv4 的建议值为 576 个八位字节。对于 IPv6，最小值是 1280 个八位字节，最小片段重组缓存值是 1500 个八位字节。68 个八位字节太小了，所以对于大多数的链路层最小值都是 1500 个八位字节。

开发者无法预先知道一个包所经过的所有链接的 MTU。如果发送的数据报大于接收者的 MTU 上限，那么包就会被静默丢弃，既没有任何反馈也不会到大目的地。

#### socket.setBroadcast(flag)

- `flag`，布尔值

该方法用于设置请清除 socket 的 `SO_BROADCAST` 选项。如果值为 true，则 UDP 包可能会被发往本地的广播地址。

#### socket.setMulticastLoopback(flag)

- `flag`，布尔值

该方法用于设置请清除 socket 的 `SO_BROADCAST` 选项。如果值为 true，则组播包也会被本地接口接收。

#### socket.setMulticastTTL(ttl)

- `ttl`，整数

该方法用于设置 `IP_MULTICAST_TTL`。虽然通常来说 TTL 表示生存时间 `Time to Live`，但在这里它表示一个包允许经过的 IP 跳跃点数量。每一个路由或网关都会减少包的 TTL 值，如果 TTL 的值为 0，那么该包将不会被转发。通常使用网络探测器或多播时会修改 TTL 的值。

`socket.setTTL()` 的参数是 1~255 之间的数值，用于表示跳跃点的数量，在大多数系统上默认值为 64 且可以被人为的改动。

#### socket.ref()

默认情况下，绑定 socket 会从 socket 创建到打开的过程阻塞 Node.js 进行的执行。`socket.unref()` 方法可用于从引用计数中去除 socket，以保持 Node.js 进程的活跃。`socket.ref()` 方法用于将 socket 添加到引用计数中并回复为默认行为。

反复调用 `socket.ref()` 并不会有额外的效果。

由于 `socket.ref()` 返回一个 socket 的引用，所以该方法之后可以进行链式调用。

#### socket.unref()

默认情况下，绑定 socket 会从 socket 创建到打开的过程阻塞 Node.js 进行的执行。`socket.unref()` 方法可用于从引用计数中去除 socket，以保持 Node.js 进程的活跃，并且即使 socket 仍在监听，也可以退出进程。

反复调用 `socket.unref()` 并不会有额外的效果。

由于 `socket.unref()` 返回一个 socket 的引用，所以该方法之后可以进行链式调用。

#### 异步 socket.bind()

从 Node.js v0.10 开始， `dgram.Socket#bind()` 方法被修改成了异步执行的方法。类似如下的同步执行代码：

```js
const s = dgram.createSocket('udp4');
s.bind(1234);
s.addMembership('224.0.0.114');
```

必须修改成接收回调函数的方法：

```js
const s = dgram.createSocket('udp4');
s.bind(1234, () => {
  s.addMembership('224.0.0.114');
});
```

## dgram 模块方法

#### dgram.createSocket(options[, callback])

- `options`，对象
- `callback`，函数，和 `message` 事件所绑定
- 返回值类型：dgram.Socket 实例

该方法用于创建一个 `dgram.Socket` 对象。`options` 参数是一个包含 `type` 必选属性，`udp4` 或 `udp6` 两者之一以及可选布尔属性 `reuseAddr` 的对象。

如果 `reuseAddr` 的值为 true，即使其他进程已经绑定了 socket，`socket.bind()` 也可以复用地址。`reuseAddr` 的默认值为 false。可选参数 `callback` 会和 `message` 事件绑定。

一旦创建了 socket，调用 `socket.bind()` 就会让 socket 开始监听数据报消息。当`socket.bind()` 没有接收到 `address` 和 `port` 参数时，系统就会将 socket 绑定到所有的接口地址的随机端口上。使用 `socket.address().address` 和 `socket.address().port` 可以获取相应的绑定地址和端口信息。

#### dgram.createSocket(type[, callback])

- `type`，字符串，udp4 和 udp6 中的一个
- `callback`，函数，和 `message` 事件绑定
- 返回值类型：dgram.Socket 实例

该方法根据指定的 `type` 创建一个 `dgram.Socket` 对象。`type` 参数可以是 udp4 和 udp6 中的一个。可选参数 `callback` 会和 `message` 事件绑定。

一旦创建了 socket，调用 `socket.bind()` 就会让 socket 开始监听数据报消息。当`socket.bind()` 没有接收到 `address` 和 `port` 参数时，系统就会将 socket 绑定到所有的接口地址的随机端口上。使用 `socket.address().address` 和 `socket.address().port` 可以获取相应的绑定地址和端口信息。

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