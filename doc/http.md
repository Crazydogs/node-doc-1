## HTTP

<div class="s s2"></div>

通过 `require('htt')` 加载 HTTP 模块可以创建和使用 HTTP 服务器和客户端。

Node.js 中的 HTTP 接口对协议有良好的支持，包括那些传统上比较难处理的特性，比如大规模、块编码信息。这些接口不会缓存完整的请求或响应，从而允许开发者使用数据流。

HTTP 的头信息类似如下所示：

```js
{ 
    'content-length': '123',
    'content-type': 'text/plain',
    'connection': 'keep-alive',
    'host': 'mysite.com',
    'accept': '*/*' 
}
```

头信息中的键都是小写的，值是不能修改的。

为了支持尽可能多的 HTTP 应用程序，Node.js 的 HTTP API 通常都是比较偏底层的。通常只能处理流和信息。它可以将信息解析成报文头或报文体，但不能直接报文头和报文体。

查看 [message.headers](https://nodejs.org/dist/latest-v5.x/docs/api/http.html#http_message_headers) 可以了解更多如何处理重复报文头的信息。

接收到的原始报文头会被保存在 `rawHeaders` 属性中，该属性通常是一个类似 `[key, value, key2, value2, ...]` 的数组，比如之前提到的头信息对象有可能包含如下所示的 `rasHeaders`：

```js
[ 
    'ConTent-Length', '123456',
    'content-LENGTH', '123',
    'content-type', 'text/plain',
    'CONNECTION', 'keep-alive',
    'Host', 'mysite.com',
    'accepT', '*/*' 
]
```

## Class: http.Agent

HTTP Agent 常用于 HTTP 客户端请求的 socket 池中。

HTTP Agent 默认为客户端的请求设置 `Connection:keep-alive`。如果 socket 处于空闲状态，则 socket 将会自动被关闭，这也就是说，使用 `keep-alive` 简化了开发者对 Node.js 请求池的管理。

如果开发者决定使用 HTTP `keep-alive`，可以通过

开发者可以通过创建 HTTP Agent 对象时修改初始化属性使用 HTTP `keep-alive`，该对象将会复用空闲的 socket。此类对象都会被明确标记，防止它们常驻 Node.js 的进程。不过，通过 `destory()` 方法显式注销 `keep-alive` 的对象是一个更好的做法。

当 socket 触发一个 `close` 或者 `agentRemove` 事件时，agent 池就会移除该 socket。这也就是说，如果开发者想让 socket 长时间运行，且在运行后自动关闭，那么就可以通过触发这两类事件实现：

```js
http.get(options, (res) => {
  // Do stuff
}).on('socket', (socket) => {
  socket.emit('agentRemove');
});
```

此外，也可以通过设置 `agent: false` 不适用资源池：

```js
http.get({
  hostname: 'localhost',
  port: 80,
  path: '/',
  agent: false  // create a new agent just for this one request
}, (res) => {
  // Do stuff with response
})
```

## new Agent([options])

- `options`，对象，用于配置 agent 对象的可选参数，该对象拥有以下属性
    - `keepAlive`，布尔值，决定允许其他请求复用 socket 池中空闲的 socket，默认值为 `false`
    - `keepAliveMsecs`，数值，当 `keepAlive` 为 true 时，该属性用于决定通过在线的 socket 发送 TCP KeepAlive 的频率，默认值为 1000
    - `macSockets`，数值，决定每个主机允许的最大 socket 量，默认值为 `Infinity`
    - `maxFreeSockets`，数值，当 `keepAlive` 为 true 时，该属性决定空闲状态下开启 socket 的最大量，默认值为 `256`

用于 `http.request()` 的 `http.globalAgent()` 在默认情况下会对每一个 agent 对象使用默认值。如果要设置其中某个对象的属性，必须创建自定义的 `http.Agent` 对象：

```js
const http = require('http');
var keepAliveAgent = new http.Agent({ keepAlive: true });
options.agent = keepAliveAgent;
http.request(options, onResponseCallback);
```

#### agent.createConnection(optionsp[, callback])

该方法用于创建一个用于 HTTP 请求的 socket 或 stream 实例。

默认情况下，该方法和 `[net.createConnection()][]` 相同。但是，该方法可以通过修改 Agent 实例的默认配置，进而获得更适合的 Agent 对象。

有两种方式可以获取到 socket 或 stream 实例：一是通过该方法的返回值获取；一是从 `callback` 的参数获取。

`callabck` 接收两个参数 `(err, stream)`。

#### agent.destory()

该方法用与注销 Agent 实例所使用的全部 socket。

通产来说开发者无需显式注销 socket。不过，如果是 `keep-alive` 的 Agent 实例，那么最好由开发者显式注销 socket。因为对于此类 socket，除非服务器终端这些它们，否则它们就会一直存在。

#### agent.freeSockets

该属性是一个对象，包含了 `keep-alive` 的 Agent 实例所拥有的空闲 socket。不建议修改该属性。

#### agent.getName(options)

该方法为一组请求设置一个独一无二的标识符，用于标志一个链接是否可以复用。对于 http agent，该方法返回 `host:port:localAddress`；对于 https agent，该方法的返回值会包含 CA、cert、密钥和其他 HTTPS/TLS 所指定的 socket 复用配置项。

#### agent.maxFreeSockets

该属性的默认值为 256。对于 `keep-alive` 的 HTTP Agent 实例，该属性用于指定空闲状态下开启 socket 的最大量。

#### agent.maxSockets

该属性的默认值为 `Infinity`，用于指定同一源地址上 Agent 实例所能打开的 socket 的最大并发量。原地址可以是 `host:port` 或 `host:port:localAddress`。

#### agent.requrests

该属性是一个对象，包含尚未分配给 socket 的请求队列。不建议修改该属性。

#### agent.sockets

该属性是一个对象，包含 Agent 实例正在使用 socket。不建议修改该属性。

## Class: http.ClientRequest

该对象由 `http.request()` 创建并作为返回值返回到外部。该对象用于表示一个头信息已加入处理队列的请求。虽然头信息已加入请求队列，但仍然可以通过 `setHeader(name, value)`、`getHeader(name)` 和 `removeHeader(name)` 来修改。当第一次发送数据或关闭连接时，头信息会随数据块一起发送出去。

通过监听 `response` 事件可以接收相应信息。当请求对象接收到响应头信息之后，就会触发 `response` 事件。`response` 事件接收一个参数，该参数是 `http.IncomingMessage` 的实例。

在 `response` 事件发生期间，可以给响应对象添加监听器，特别是用于监听 `data` 事件的监听器。

如果未对 `response` 事件设置处理函数，则响应信息就会被忽略。不过，如果添加了 `response` 事件的处理函数，则无论是使用 `response.read()`，或者添加 `data` 事件处理器，还是调用 `.resume()` 方法，都必须对响应对象的数据进行处理。只有数据经过处理之后，才会触发 `end` 事件。此外，除非读取数据，否则将会占用不必要的内容，直至触发 `process out of memory` 错误。

注意，Node.js 不会的对内容和消息体的长度进行比较。

该请求对象实现自 `Writable Stream` 接口，它是拥有可以触发如下事件 EventEmitter 实例：

#### 事件：'abort'

- `function () {}`

当客户端取消请求时触发该事件，且该方法只在第一次调用 `abort()` 时触发。

#### 事件：'checkExpectation'

- `function (request, response) {}`

每当接请求对象收到预期的 http 头消息（不包含 `Expect:100-continue`）时就会触发该事件。如果没有监听该事件，服务器就会自动发送一个包含 417 预期失败的响应对象。

注意，当该事件被触发并处理之后，`request` 事件将不会再被触发。

#### 事件：'connect'

- `function (response, socket, head) { }`

每次服务器通过 `CONNECT` 方法响应请求对象时，都会触发该事件。如果该事件未被监听，接收到 `CONNECT` 方法的客户端就会关闭连接。

下面代码演示了如何在客户端和服务器端监听 `connect` 事件：

```js
const http = require('http');
const net = require('net');
const url = require('url');

// Create an HTTP tunneling proxy
var proxy = http.createServer( (req, res) => {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('okay');
});
proxy.on('connect', (req, cltSocket, head) => {
  // connect to an origin server
  var srvUrl = url.parse(`http://${req.url}`);
  var srvSocket = net.connect(srvUrl.port, srvUrl.hostname, () => {
    cltSocket.write('HTTP/1.1 200 Connection Established\r\n' +
                    'Proxy-agent: Node.js-Proxy\r\n' +
                    '\r\n');
    srvSocket.write(head);
    srvSocket.pipe(cltSocket);
    cltSocket.pipe(srvSocket);
  });
});

// now that proxy is running
proxy.listen(1337, '127.0.0.1', () => {

  // make a request to a tunneling proxy
  var options = {
    port: 1337,
    hostname: '127.0.0.1',
    method: 'CONNECT',
    path: 'www.google.com:80'
  };

  var req = http.request(options);
  req.end();

  req.on('connect', (res, socket, head) => {
    console.log('got connected!');

    // make a request over an HTTP tunnel
    socket.write('GET / HTTP/1.1\r\n' +
                 'Host: www.google.com:80\r\n' +
                 'Connection: close\r\n' +
                 '\r\n');
    socket.on('data', (chunk) => {
      console.log(chunk.toString());
    });
    socket.on('end', () => {
      proxy.close();
    });
  });
});
```

#### 事件：'continue'

- `function () {}`

当请求对象包含 `Expect: 100-continue` 时，服务器通常会发送一个包含 `100 Continue` 的 HTTP 响应信息，此时就会触发该事件。该事件的作用是告诉客户端可以发送请求体了。

#### 事件：'response'

- `function (response) {}`

当请求对象接收到响应信息时就会触发该事件，且只触发一次，其中 `response` 参数是 `http.IncomingMessage` 的实例。

可选配置项：

- `host`，请求的服务器域名或 IP 地址
- `port`，远程服务器的端口号
- `socketPath`，Unix 域名 Socket（`host:port` 或 socketPath）

#### 事件：'socket'

- `function (socket) {}`

当一个 socket 被分配给请求兑现过之后就会触发该事件。

#### 事件：'upgrade'

- `function (response, socket, head) {}`

每次服务器相应一个 upgrade 请求时就会触发该事件。如果为监听该事件，收到 upgrade 头信息的客户端就会关闭连接。

下面代码演示了如何在客户端和服务器端监听 `connect` 事件：

```js
const http = require('http');

// Create an HTTP server
var srv = http.createServer( (req, res) => {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('okay');
});
srv.on('upgrade', (req, socket, head) => {
  socket.write('HTTP/1.1 101 Web Socket Protocol Handshake\r\n' +
               'Upgrade: WebSocket\r\n' +
               'Connection: Upgrade\r\n' +
               '\r\n');

  socket.pipe(socket); // echo back
});

// now that server is running
srv.listen(1337, '127.0.0.1', () => {

  // make a request
  var options = {
    port: 1337,
    hostname: '127.0.0.1',
    headers: {
      'Connection': 'Upgrade',
      'Upgrade': 'websocket'
    }
  };

  var req = http.request(options);
  req.end();

  req.on('upgrade', (res, socket, upgradeHead) => {
    console.log('got upgraded!');
    socket.end();
    process.exit(0);
  });
});
```

#### request.abort()

该方法用于终止请求。调用该方法将会使剩余的相应信息被丢弃且销毁 socket。

#### request.end([data][, encoding][, callback])

该方法用于结束发送请求，如果调用该方法时消息体的某些部分尚未发送，则这些部分将会被丢弃给 Stream 实例；如果请求是分块发送的，则调用该方法会直接发送终止块 `'0\r\n\r\n'`。

如果指定了 `data` 参数，相当于先调用了 `response.write(data, encoding)`，然后调用了 `request.end(callback)`。

如果指定了 `callback`，则当 request stream 结束时调用该回调函数。

#### request.flushHeaders()

该方法用于刷新请求头信息。

处于性能方面的原因，Node.js 通常会缓存请求头信息，除非开发者显式调用 `request.end()` 或写入第一个请求数据块。然后，系统就会将请求头信息和数据打包进一个 TCP 包。

虽然通常这已经满足了开发者的需求（可以节省一次 TCP 握手），但是发送第一块数据的时间可能会有点久。`request.flushHeaders()` 便于开发者忽略这些优化而发起请求。

#### request.setNoDelay([noDelay])

当请求对象分配到 socket 并连接之后就会调用 `socket.setNoDelay()` 方法。

#### request.setSocketKeepAlive([enable][, initialDelay])

当请求对象分配到 socket 并连接之后就会调用 `socket.setKeepAlive()` 方法。

#### request.setTimeout(timeout[, callback])

当请求对象分配到 socket 并连接之后就会调用 `socket.setTimeout()` 方法。

- `timeout`，数值，决定请求超时的时间
- `callback`，函数，请求超时后调用的回调函数

#### request.write(chunk[, encoding][, callback])

该方法用于发送请求体的数据块。多次调用该方法，可以从客户端向服务端发送请求体。对于发送请求体这种任务，建议创建请求对象时配置头信息 `['Transfer-Encoding', 'chunked']`。

`chunk` 参数可以是一个 Buffer 实例或字符串。

可选参数 `encoding` 只有当 `chunk` 为字符串时才可用，默认值为 `utf8`。

可选参数 `callback` 只有当数据被刷新掉时才会触发。

该函数的返回类型为 request 对象。

## Class: http.Server

该类继承自 `net.Server`，可以触发以下事件：

#### 事件：'checkContinue'

- `function (request, response) {}`

每当接请求对象收到 `Expect:100-continue` http 头消息时就会触发该事件。如果没有监听该事件，服务器就会自动发送一个包含 100 Continue 的响应对象。

如果客户端需要继续发送请求体或者生成适当的 HTTP 响应（比如 400，无效请求），则可以通过调用 `response.writeContinue()` 处理该事件。

注意，当该事件被触发并处理之后，`request` 事件将不会再被触发。

#### 事件：'clientError'

- `function (exception, socket) {}`

如果客户端在连接过程中触发一个 `error` 事件，就会被转义到该事件上。

触发错误的 `socket` 对象是 `net.Socket` 的实例。

#### 事件：'close'

- `function () {}`

当服务器关闭时触发该事件。

#### 事件：'connect'

- `function (request, socket, head) {}`

每次客户端通过 `CONNECT` 方法发起请求时，都会触发该事件。如果该事件未被监听，接收到 `CONNECT` 方法的客户端就会关闭连接。

- `request` 参数是 request 事件中的请求对象
- `socket` 是服务器和客户端之间的网络套接字
- `head` 是一个 Buffer 实例，隧道 stream 的第一个数据块，值可能为空

触发该事件之后，请求的 socket 将不会给 `data` 事件绑定监听器，这也就是说开发者需要显式声明 `data` 事件的监听器，用于处理经 socket 发送给服务器的数据。

#### 事件：'connection' 

- `function (socket) {}`

新建 TCP 流时会触发该事件。`socket` 是一个 `net.Socket` 的实例。通常开发者并不会使用该事件，但有时候协议解析器绑定 socket 的方式，并不会让 socket 触发 `readable` 事件。`socket` 也可以通过 `request.connection` 方法获得。

#### 事件：'request'

- `function (request, response) {}`

每次接收到请求对象时都会触发该事件。注意，每个链接可能有多个请求对象。`request` 是 `http.IncomingMessage` 的实例，`response` 是 `http.ServerResponse` 的实例。

#### 事件：'upgrade'

- `function (request, socket, head) {}`

每次客户端请求 http upgrade 时就会触发该事件。如果为监听该事件，收到 upgrade 头信息的客户端就会关闭连接。

- `request` 参数是 request 事件中的请求对象
- `socket` 是服务器和客户端之间的网络套接字
- `head` 是一个 Buffer 实例，隧道 stream 的第一个数据块，值可能为空

触发该事件之后，请求的 socket 将不会给 `data` 事件绑定监听器，这也就是说开发者需要显式声明 `data` 事件的监听器，用于处理经 socket 发送给服务器的数据。

#### server.close([callback])

该方法用于终止服务器接收新的连接请求。

#### server.listen(handle[, callback])

- `handle`，对象
- `callback`，函数

`handle` 对象可以是一个 server、socket（任意以下划线开头的 `_handle` 成员） 或 `{fd: <n>}` 对象。

使用该方法会锁定无法根据指定的 `handle` 接收连接，并默认为文件描述符或者 `handle` 已经绑定到了端口和域名 socket 上。

Windows 不支持对文件描述符的监听。

该方法是异步执行的方法，最后一个参数 `callback` 用于为 `listening` 时间添加监听器。

该方法的返回值是一个 `server` 对象。

#### server.listen(path[, callback])

给方法启动一个 UNIX socket 服务器，根据指定的 `path` 监听连接请求。

该方法是异步执行的方法，最后一个参数 `callback` 用于为 `listening` 时间添加监听器。

#### server.listen(port[, hostname][, backlog][, callback])

该方法根据指定的 `port` 和 `hostname` 接收连接。如果未指定 `hostname`，则当 IPv6 可用时，服务器会接受任意的 IPv6 地址，否则就接收任意的 IPv4 地址。如果端口值为 0，则随机分配一个端口号。

如果要监听 UNIX socket，则需要使用文件名，而不是端口或域名。

`backlog` 参数指定了等待连接队列的最大长度。该参数的实际值由操作系统通过 sysctl 配置项决定，比如 Linux 中的 `tcp_max_syn_backlog` 和 `somaxconn`。该参数的默认值为 `511`，而不是 522。

该方法是异步执行的方法，最后一个参数 `callback` 用于为 `listening` 时间添加监听器。

#### server.listening

该属性是一个布尔值，用于标识服务器是否在监听连接请求。

#### server.maxHeadersCount

该属性限制请求头的最大数量，默认值为 1000。如果值为 0，则表示不限制。

#### server.setTimeout(msecs, callback)

- `msecs`，数值
- `callback`，函数

该方法为 socket 设置超时时间，并触发 `timeout` 事件。如果触发了超时，则参数为 socket。

如果 server 对象监听了 `timeout` 事件并设置了监听器，那么超时时就会调用该监听器，且参数为超时的 socket。

默认你情况下，服务器的超时事件是两分钟，如果 socket 超时就会别自动销毁。不过，如果开发者为服务器的 `timeout` 事件设定了自定义的监听器，则由开发者负责销毁 socket。

#### server.timeout

- 数值，默认值为 120000（两分钟）

该属性指定 socket 处于空闲状态的超时时间。

注意，socket 超时的逻辑在连接时初始化，所以之后该属性的修改只对新创建的连接有效，不会影响现有的 socket。

如果该属性的值为 0，则对连接禁用任何类型的超时处理。

## Class: http.ServerResponse

该对象由 HTTP 服务器创建，而不是由开发者创建。它常作为 `request` 事件的第二个参数。

该响应对象实现自 `Writable Stream` 接口，它是拥有可以触发如下事件 EventEmitter 实例：

#### 事件：'close'

- `function () {}`

有两种行为会触发该事件：一是在调用 `response.end()` 之前中断底层连接；一是刷新数据。

#### 事件：'finish'

- `function () {}`
 
响应信息发送之后会触发该事件。更准确地说，当响应信息的最后一部分被操作系统传输给网络时触发该事件。这并不表示客户端是否收到了数据。

该事件触发之后，响应对象不会再触发其他事件。

#### response.addTrailers(headers)

该方法用于给 response 对象添加 HTTP trailing header（信息尾部的头信息）。

只有响应对象使用了块编码时，才会触发 Trailer。如果没有使用块编码（比如使用 HTTP/1.0 的 request），则会被毫无声息地抛弃。

注意，如果要触发 Trailer，则 HTTP 需要发送 `Trailer` 头信息，该头信息包含以下属性和值：

```js
response.writeHead(200, { 'Content-Type': 'text/plain','Trailer': 'Content-MD5' });
response.write(fileData);
response.addTrailers({'Content-MD5': '7895bf4b8828b55ceaf47747b4bca667'});
response.end();
```

如果修改头信息时使用了无效字符作为键值，则会导致系统抛出 TypeError。

#### response.end([data][, encoding][,callback])

该方法用于通知服务器所有的额响应信息已经发送完成。每一个响应对象都必须调用该方法。

如果指定了 `data`，则相当于先调用了 `resposne.write(data, encoding)`，然后调用了 `response.end(callback)`。

如果指定了 `callback`，那么在响应信息流结束后就会调用该回调函数。

#### response.finished

该属性是一个布尔值，用于标识响应过程是否结束，默认值为 `false`。执行 `response.end()` 之后，该值变为 `true`。

#### response.getHeader(name)

该方法用于读取已进入队列尚未发送给客户端的头信息。注意，这里的 `name` 需要注意大小写。该方法只能在头信息刷新之后调用。

```js
var contentType = response.getHeader('content-type');
```

#### response.headerSent

该属性是一个只读的布尔值。如果头信息已被发送，则值为 true，否则为 false。

#### response.removeHeader(name)

该方法从头待发送队列中删除指定的头信息。

```js
response.removeHeader('Content-Encoding');
```

#### response.sendDate

当该属性为 true 时，如果头信息没有日期，则会自动在响应信息中生成日期并发送出去，默认值为 true。

建议只在测试时禁用该属性，因为 HTTP 需要知道响应信息的

#### response.setHeader(name, value)

该方法用于配置头信息。如果该头信息即将被发送，则相应的值会被替换。如果你需要发送多个名字相同的头信息，可以使用字符串数组。

```js
response.setHeader('Content-Type', 'text/html');

// or
response.setHeader('Set-Cookie', ['type=ninja', 'language=javascript']);
```

如果修改头信息时使用了无效字符作为键值，则会导致系统抛出 TypeError。

使用 `response.setHeader()` 配置头信息之后，这些头信息会被其他头信息合并并传递给 `response.writeHead()`，但是传递给 `response.writeHead()` 的头信息优先级较高。

```js
// returns content-type = text/plain
const server = http.createServer((req,res) => {
  res.setHeader('Content-Type', 'text/html');
  res.setHeader('X-Foo', 'bar');
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('ok');
});
```

#### response.setTimeout(msecs,callback)

- `msecs`，数值
- `callback`，函数

该方法为 socket 设置超时时间。如果传入了回调函数，则该回调函数会被视为 `timeout` 事件的监听器。

如果请求对象、响应对象或服务器监听了 `timeout` 事件，那么 socket 超时时就会被销毁。如果开发者为服务器、请求对象或响应对象的 `timeout` 事件设定了自定义的监听器，则由开发者负责销毁 socket。

该方法的返回值是 `resposne` 对象。

#### response.statusCode

当使用隐式头信息（没有显式调用 `response.writeHead()` 的头信息）时，该属性决定了头信息刷新后发送给客户端的状态码。

```js
response.statusCode = 404;
```

响应头信息发送给客户端之后，该属性表示已发送的状态码。

#### response.statusMessage

当使用隐式头信息（没有显式调用 `response.writeHead()` 的头信息）时，该属性决定了头信息刷新后发送给客户端的状态信息。如果该属性的值以 `undefined` 发送，则会使用标准的状态码的状态信息。

```js
response.statusMessage = 'Not found';
```

响应头信息发送给客户端之后，该属性表示已发送的状态信息。

#### response.write(chunk[, encoding][, callback])

如果调用该方法之前没有调用 `response.writeHead()`，那么该方法将会切换到隐式头信息模式并刷新隐式头信息。

该方法用于发送一块响应体。该方法可以被多次调用，以保证响应体成功发送出去了。

`chunk` 可以是一个字符串或 Buffer 实例。如果 `chunk` 是一个字符串，那么第二个参数则指定了该字符串转化为字节码 stream 的编码格式。`encoding` 的默认值为 `utf8`。数据块刷新时会调用回调函数 `callback`。 

注意，这是原始的 HTTP body，和 higher-level multi-part body 编码无关。

第一次调用 `response.write()` 时，将会发送缓存中的头信息和第一块 body 信息给客户端。第二次调用 `response.write()` 时，Node.js 将会默认独立发送流数据，也就是说，响应信息已经缓存在第一块 body 中了。

如果所有的数据已经成功刷新进了 kenel buffer，那么该方法返回 `true`；如果尚有数据存在于内存中，则该方法返回 `false`。当 buffer 为空时将会触发 `drain` 事件。

#### response.writeContinue()

该方法向客户端发送一个 HTTP1.1 100 Continue 的消息，用于指示应该发送请求体了。

#### response.writeHead(statusCode[, statusMessage][, headers])

该方法向请求对象发送一个响应头。状态码是一个三位的 HTTP 状态码，比如 404。最后一个参数 `headers` 是响应头信息。可选参数 `statusMessage` 是易于辨识状态信息。

```js
var body = 'hello world';
response.writeHead(200, {
  'Content-Length': body.length,
  'Content-Type': 'text/plain' });
```

每次消息只能调用该方法一次，且必须在 `response.end()` 之前调用。

如果你在调用 `response.write()` 和 `response.end()` 之后调用该方法，那么就会产生隐式或可变的头信息。

使用 `response.setHeader()` 配置头信息之后，这些头信息会被其他头信息合并并传递给 `response.writeHead()`，但是传递给 `response.writeHead()` 的头信息优先级较高。

```js
// returns content-type = text/plain
const server = http.createServer((req,res) => {
  res.setHeader('Content-Type', 'text/html');
  res.setHeader('X-Foo', 'bar');
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('ok');
});
```

注意，`content-length` 是字节数而不是字符数。上述代码之所以正常运行，是因为这里的 `hello world` 只是字节字符。如果 body 包含高级编码字符，就必须使用 `Buffer.byteLength()` 根据指定的编码格式计算字节数。此外，Node.js 并不会 content 和 body 的长度是否一致。

如果修改头信息时使用了无效字符作为键值，则会导致系统抛出 TypeError。

## Class: http.IncomingMessage

`IncomingMessage` 对象可以由 `http.Server` 或 `http.ClientRequest` 创建，并且可以作为 `request` 和 `response` 事件监听器的第一个参数。它可以用于访问响应的状态、头信息和数据。

该对象实现了 `Readable Stream` 接口，具有以下事件、方法和属性：

#### 事件：'close'

- `function () {}`

当底层链接关闭时触发该事件。和 `end` 事件一样，该事件对于每个响应对象只发生一次。

#### message.headers

该属性是包含请求或响应头信息的对象。

该属性内部以键值对的形式存储头信息，头信息的键名必须是小写：

```js
// Prints something like:
//
// { 'user-agent': 'curl/7.22.0',
//   host: '127.0.0.1:8000',
//   accept: '*/*' }
console.log(request.headers);
```

根据头信息中的键名，对于重复的原始头信息分为以下多种处理方式：

- 重复的 age、authorization、content-length、content-type、etag、expires、from、host、if-modified-since、if-unmodified-since、last-modified、location、max-forwards、proxy-authorization、referer、retry-after, or user-agent 会被丢弃
- `set-cookie` 是一个数组，重复的键值会被添加到该数组中
- 对于头信息其他键名的键值，使用 `,` 拼接在一起

#### message.httpVersion

该属性表示客户端支持的 HTTP 版本，通常为 '1.1' 或 '1.0'。

此外，也可以通过 `message.httpVersionMajor` 获取大版本号，通过 `message.httpVersionMinor` 获取小版本号。

#### message.method

该属性只对来自 `http.Server` 的请求有效。

请求方法是一个只读字符串，比如 'GET'、'DELETE'。

#### message.rawHeaders

该属性是一个接收到的原始请求或响应头信息的列表。

注意，键名和键值同处于该列表中，偶数索引为键名，奇数索引为键值。

头信息的键名不限制大小写，即使存在重复的键名也不会被合并。

```js
// Prints something like:
//
// [ 'user-agent',
//   'this is invalid because there can be only one',
//   'User-Agent',
//   'curl/7.22.0',
//   'Host',
//   '127.0.0.1:8000',
//   'ACCEPT',
//   '*/*' ]
console.log(request.rawHeaders);
```

#### message.rawTrailers

该属性包含了接收到的原始请求或响应 trailer 的键值对，且只存在与 `end` 事件中。

#### message.setTimeout(msecs, callback)

- `msecs`，数值
- `callback`，函数

相当于调用 `message.connection.setTimeout(msecs, callback)`。

该方法的返回值是 `message` 对象。

#### message.statusCode

该属性只对来自 `http.ClientRequest` 的响应有效。

该属性的值是 3 位 HTTP 响应状态码，比如 `404`。

#### message.statusMessage

该属性只对来自 `http.ClientRequest` 的响应有效。

该属性的值用于表示 HTTP 响应状态信息，比如 `OK` 或 `Internal Server Error`。

#### message.socket

该属性表示与当前链接有关的 `net.Socket` 对象。

在 HTTPS 中，使用 `request.socket.getPeerCertificate()` 方法可以获取客户端的验证资料。

#### message.trailers

该属性包含了请求或响应的 trailer 对象，且只存在与 `end` 事件中。

#### message.url

该属性只对来自 `http.Server` 的请求有效。

该属性表示发起请求的 URL，值为字符串形式。该属性只包含白哦是实际 HTTP 请求的 URL，比如有如下所示的请求：

```js
GET /status?name=ryan HTTP/1.1\r\n
Accept: text/plain\r\n
\r\n
```

则 `request.url` 的值为：

```js
'/status?name=ryan'
```

如果你想讲 URL 拆解开来，可以使用 `require(url).parse(request.url)` 来处理：

```js
$ node
> require('url').parse('/status?name=ryan')
{
  href: '/status?name=ryan',
  search: '?name=ryan',
  query: 'name=ryan',
  pathname: '/status'
}
```

如果你想去除查询字符串中的参数，可以使用 `require('querystring').parse` 或给 `require('url').parse` 方法传递 `true` 参数：

```js
$ node
> require('url').parse('/status?name=ryan', true)
{
  href: '/status?name=ryan',
  search: '?name=ryan',
  query: {name: 'ryan'},
  pathname: '/status'
}
```

#### http.METHODS

- 数组

该属性是一个数组，包含解析器所支持的 HTTP 方法。

#### http.STATUS_CODES

- 对象

该对象包含所有标准的 HTTP 响应状态码，以及相应的描述信息，比如 `http.STATUS_CODES[404] === 'Not Found'`。

#### http.createClient([port][, host])

<div class="s s0">
使用 http.request() 替代。
</div>

#### http.createServer([requestListener])

该方法返回一个 `http.Server` 新建的实例。

`requestListener` 是一个函数，将会被系统自动添加给 `request` 事件。

#### http.get(options[, callback])

由于大部分的请求都是没有 body 的 `GET` 请求，所以 Node.js 提供了这一方法。该方法与 `http.request()` 唯一的不同在于，它的默认 HTTP 方法就是 `GET`，并会自动调用 `req.end()`。

```js
http.get('http://www.google.com/index.html', (res) => {
  console.log(`Got response: ${res.statusCode}`);
  // consume response body
  res.resume();
}).on('error', (e) => {
  console.log(`Got error: ${e.message}`);
});
```

#### http.globalAgent

该属性是 Agent 的全局性实例，也是所有 HTTP 客户端请求的默认目标。

#### http.request(options[, callback])

Node.js 和每个服务器之间又存在链接，用于发送 HTTP 请求。开发者也可以使用该方法发出请求。

`options` 可以是对象或字符串。如果是字符串，则系统调用 `url.parse()` 解析该参数。`options` 包含的属性包括：

- `protocal`，使用的协议，默认值为 `http:`
- `hsot`，发出请求的服务器的域名或 IP 地址
- `hostname`，`host` 的别名，就 `url.parse()` 的支持度而言，`hostname` 要优于 `host`
- `family`，解析 host 或 hostname 时使用的 IP 地址簇。有效值为 4 和 6。如果未指定该参数，则 IPv4 和 IPv6 都可以使用。
- `port`，远程服务器的端口号，默认值为 80
- `localAddress`，绑定网络连接的本地接口
- `socketPath`，Unix Domain Socket（host:port 或 socketPath）
- `method`，字符串形式的 HTTP 请求方法，默认值为 `GET`
- `path`，请求路径，默认值为 `/`。如果存在查询字符串，则应该包含查询字符串，比如 `'/index.html?page=12'`。如果请求路径包含非法字符，则系统会抛出异常。目前，只有空格是不允许的，未来可能对此进行修改。
- `headers`，包含请求头信息的对象。
- `auth`，基本的验证信息，比如用于计算验证头信息的 `user:password`
- `agent`，用于控制 Agent 的行为。当使用 Agent 时，请求默认使用 `Connection: keep-alive`。可选值包括：
    - `undefined`，默认值，在这个主机和端口上使用 `http.globalAgent`
    - `Agent`，对象，在 `Agent` 中显式使用传入的参数
    - `false`，退出 Agent 连接池，默认请求 `Connection: close`
- `createConnection`，函数，当未使用 `agent` 参数时，该方法用于生成 request 需要的 socket 或 stream。该方法可以避免创建一个只为了重写 `createConnection` 函数的自定义的 Agent。

可选参数 `callback` 会被设为 `response` 事件的一次性监听器。

`http.request()` 方法返回一个 `http.ClientRequest` 的实例。`ClientRequest` 的实例是一个可写 stream。如果开发者需要使用 `POST` 方法上传文件，那么就可以将其写入到 `ClientRequest` 对象中。

```js
var postData = querystring.stringify({
  'msg' : 'Hello World!'
});

var options = {
  hostname: 'www.google.com',
  port: 80,
  path: '/upload',
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded',
    'Content-Length': postData.length
  }
};

var req = http.request(options, (res) => {
  console.log(`STATUS: ${res.statusCode}`);
  console.log(`HEADERS: ${JSON.stringify(res.headers)}`);
  res.setEncoding('utf8');
  res.on('data', (chunk) => {
    console.log(`BODY: ${chunk}`);
  });
  res.on('end', () => {
    console.log('No more data in response.')
  })
});

req.on('error', (e) => {
  console.log(`problem with request: ${e.message}`);
});

// write data to request body
req.write(postData);
req.end();
```

注意，在上面代码的最后调用了 `req.end()`。对于 `http.request` 方法，必须调用 `req.end()` 方法显式结束请求，即使未向请求体写入任何数据。

如果请求过程中出现了任何错误（DNS 解析、TCP 级别或 HTTP 解析），就会在返回的请求对象中触发一个 `error` 事件。对于所有的 `error` 事件，如果没有注册监听器，则该错误会抛出到全局。

下面是几个需要特别注意的头信息：

- 发送一个包含 `Connection: keep-alive` 的头信息会让 Node.js 保持服务器的链接，知道下次请求的到来
- 发送一个包含 `Content-length` 的头信息会替换默认的块编码格式
- 发送一个包含 `Expect` 的头信息会立即发送请求头。通常来说，如果发送的是 `Expect: 100-continue'`，则开发者需要设置超时时间并监听 `continue` 事件。
- 发送一个包含验证信息的头信息会替代使用 `auth` 参数计算出来的验证信息

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