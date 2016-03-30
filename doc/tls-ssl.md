## TLS(SSL)

<div class="s s2"></div>

通过 `require('tls')` 可以加载该模块。

`tls` 模块使用 OpenSSL 为安全传输层和安全套接层提供了加密的流通讯。

TLS/SSL 是一个公钥/私钥基础架构。每一个客户端和服务器都必须拥有一个私钥。创建私钥的操作如下所示：

```bash
openssl genrsa -out ryans-key.pem 2048
```

所有的服务器和部分客户端需要拥有整数。证书是由认证中心或以自认证的方法签发的公钥。获取证书的第一步是创建一个 `证书签发请求（Certificate Signing Requrest, CSR）` 文件：

```js
openssl req -new -sha256 -key ryans-key.pem -out ryans-csr.pem
```

使用 CSR 以自认证的方式创建的证书：

```js
openssl x509 -req -in ryans-csr.pem -signkey ryans-key.pem -out ryans-cert.pem
```

此外，开发者也可以将 CSR 文件发送给认证中心获取证书。

对于完全前向保密（Perfect forward secrecy），需要生成 Diffie-Hellman 参数：

```js
openssl dhparam -outform PEM -out dhparam.pem 2048
```

创建 .pfx 或者 .p12：

```js
openssl pkcs12 -export -in agent5-cert.pem -inkey agent5-key.pem \
      -certfile ca-cert.pem -out agent5.pfx
```

- `in`，整数
- `inkey`，私钥
- `certfile`，将所有的 CA 证书合并为一个文件，比如 `cat ca1-cert.pem ca2-cert.pem > ca-cert.pem`

## ALPN / NPN / SNI

ALPN (Application-Layer Protocol Negotiation Extension), NPN (Next Protocol Negotiation) 和 SNI (Server Name Indication) 都是 TLS 握手的扩展:

- `ALPN/NPN`，允许同一个 TLS 服务器使用多种协议，比如 HTTP，SPDY，HTTP/2
- `SNI`，允许同一个 TLS 服务器使用多个 SSL 整数不同的主机名

## 降低客户端发起的 renegotiation 攻击

TLS 协议允许客户端协商 TLS 会话的某些部分。不幸的是，会话协商需要大量的服务端资源，而这有可能让其成为服务器拒绝（denial-of-service）攻击的媒介。

为了降低这种潜在的危险，会话协商没十分钟只能发起三次。当超出限制时，`tls.TLSSocket` 的实例就会抛出一个错误。这项限制是可配置的：

- `tls.CLIENT_RENEG_LIMIT`，会话协商限制，默认值为 3
- `tls.CLIENT_RENEG_WINDOW`，会话协商窗口的限制时间，单位为秒，默认为 10 分钟

除非开发者了解正在做的事情，否则不要轻易修改默认配置。

为了测试服务器，可以通过 `openssl s_client -connect address:port` 连接到服务器，然后敲几次 `R<CR>`（字母 R + 回车键）看看。

## 修改默认的 TLS 加密套件

Node.js 内建了一套开启和禁用 TLS 加密的套件。在最新的 Node.js 中默认的加密套件是：

```js
ECDHE-RSA-AES128-GCM-SHA256:
ECDHE-ECDSA-AES128-GCM-SHA256:
ECDHE-RSA-AES256-GCM-SHA384:
ECDHE-ECDSA-AES256-GCM-SHA384:
DHE-RSA-AES128-GCM-SHA256:
ECDHE-RSA-AES128-SHA256:
DHE-RSA-AES128-SHA256:
ECDHE-RSA-AES256-SHA384:
DHE-RSA-AES256-SHA384:
ECDHE-RSA-AES256-SHA256:
DHE-RSA-AES256-SHA256:
HIGH:
!aNULL:
!eNULL:
!EXPORT:
!DES:
!RC4:
!MD5:
!PSK:
!SRP:
!CAMELLIA
```

通过 `--tls-cipher-list` 参数在命令行中可以修改这些默认参数。举例来说，下面的命令将 `ECDHE-RSA-AES128-GCM-SHA256:!RC4` 设为了默认的 TLS 加密套件：

```js
node --tls-cipher-list="ECDHE-RSA-AES128-GCM-SHA256:!RC4"
```

注意，Node.js 中默认的加密套件是精心筛选过的，通常具有最佳的安全性和最轻的风险。修改默认的加密套件有可能严重影响应用程序的安全性。除非绝对需要，否则不要使用 `--tls-cipher-list` 参数。

## 完全前向保密

`前向保密（Forward Secrecy）` 和 `完全前向保密（Perfect Fowward Secrecy）` 描述了密钥协商（比如密钥交换）方法的特点。事实上，即使服务器的私钥被窃取了，那么窃取者必须努力后去每次会话的密钥对才能完整破解会话。

完全正向保密本质上是通过每次握手时为密钥协商随机生成密钥对实现的。实现这一技术的方法被称为 `ephemeral`。

目前有两种方法可以实现完全前向保密：

- `DHE`，Diffie Hellman 密钥协商协议的 ephemeral 版本
- `ECDHE`，Elliptic Curve Diffie Hellman 密钥协商协议的 ephemeral 版本

ephermeral 的缺点主要集中在性能上，这是因为生成密钥的过程非常耗费资源。

## Class: CryptoStream

<div class="s s0">
使用 tls.TLSSocket 替代。
</div>

## Class: SecurePair

该类可由 `tls.createSecurePair` 返回。

#### 事件：'secure'

当 SecurePair 成功创阿金了安全连接之后就会触发该事件。

与检查服务器的 `secureConnection` 事件相似，`pair.cleartext.authorized` 常用于确认证书是否经过授权。

## Class: tls.Server

该类是 `net.Server` 的子类，且两者拥有相同的方法。该类接收使用 TLS 或 SSL 加密过的连接而不是原始的 TCP 连接。

#### 事件：'clientError'

- `function (exception, tlsSocket) {}`

如果客户端连接在安全建立之前触发了 `error` 事件，则客户端连接将会被转发到该方法。

`tlsSocket` 是错误的发起源 `tls.TLSSocket`。

#### 事件：'newSession'

- `function (sessionId, sessionData, callback) {]`

该事件在创建 TLS 会话之后触发。可用于在外部储存中存储会话数据。最终必须调用 `callback`，否则安全连接将不会发送或接收任何数据。

注意，此类事件监听器只有在连接建立之后添加才会生效。

#### 事件：'OCSPRequest'

- `function (certificate, issuer, callback) {}`

当客户端发送证书状态请求时就会触发该事件。开发者可以通过解析的服务器的整数获取 OCSP url 和整数 id，然后通过调用 `callback(null, resp)` 方法获取 OCSP 响应，其中 `resp` 是 Buffer 实例。`certificate` 和 `issuer` 都是最主要的的 DER-representations Buffer 实例和发行者的整数。它们可用于获取 OCSP 整数 id 和 OCSP 终结点 url。

此外，还可以调用 `callback(null, null)`，表示没有 OCSP 响应。

典型流程：

1. 客户端连接服务器并发送 `OCSPRequest`（通过 ClientHello 中的状态信息扩展）
1. 服务器接收请求并调用 `OCSPRequest` 事件监听器
1. 服务器根据 certificate 或 issuer 抓取 OCSP url，并对 CA 执行 OCSP 请求
1. 服务器从 CA 接收 `OCSPResponse` 并将通过 `callback` 发送给客户端
1. 客户端验证响应，要么销毁 socket 要么执行一次握手

注意，如果证书是自认证（self-signed）或发证者不在根证书列表中，那么 `issuer` 的值可能是 `null`。

注意，此类事件监听器只有在连接建立之后添加才会生效。

注意，开发者可以通过 npm 模块解析证书，比如 [asn1.js](https://npmjs.org/package/asn1.js)。

## 事件：'resumeSession'

- `function (sessionId, callback) {}`

当客户端要恢复先前的 TLS 会话时就会触发该事件。事件监听器也许会根据指定的 `sessionId` 到外部储存中查找，一旦查找完成就会调用 `callback(null, sessionData)`。如果会话不能恢复（比如不存在这个会话），那么系统可能会调用 `callback(null, null)`。调用 `callback(err)` 将会中断连接并销毁 socket。

注意，此类事件监听器只有在连接建立之后添加才会生效。

下面代码演示了如何恢复 TLS 会话：

```js
var tlsSessionStore = {};
server.on('newSession', (id, data, cb) => {
  tlsSessionStore[id.toString('hex')] = data;
  cb();
});
server.on('resumeSession', (id, cb) => {
  cb(null, tlsSessionStore[id.toString('hex')] || null);
});
```

#### 事件：'secureConnection'

- `function (tlsSocket) {}`

当新连接成功握手后会触发该事件，参数 `tslSocket` 是 `tls.TLSSocket` 的实例，该参数拥有 stream 的常见方法和事件。

`socket.authorized` 是一个布尔值，用于表示客户端是否已经被服务器的证书中心校验通过。如果 `socket.authorized` 的值为 false，那么就会抛出 `socket.authorizationError`，用以说明授权失败。值得提醒的是：开发者的链接是否可以被服务器接收，很大因素取决于 TLS 服务器的配置，也就是说，开发者未被授权的链接有可能会被接收。

`socket.npnProtocol` 是一个字符串，包含了已选择的 NPN 协议，`socket.alpnProtocol` 也是一个字符串，包含了已选择的 ALPN 协议。当同时接收到 NPN 和 ALPN 扩展时，ALPN 的优先级高于 NPN，且下一个协议由 ALPN 选择。当 ALPN 没有可选择的协议时，返回 false。

`socket.servername` 是一个字符串，包含了 SNI 请求的服务器名。

#### server.addContext(hostname, context)

当客户端请求的 SNI 主机名匹配 `hostanme` 时，该方法添加的 context 就会生效。`context` 包含 key / cert / ca 和其他来自 `tsl.createSecureContext()` 中 `options` 参数所包含的属性。

#### server.address()

该方法返回操作系统记录的绑定地址、地址族和服务器接口。

#### server.close([callback])

该方法中断服务器接收新的链接。该方法是异步执行的，当服务器触发 `close` 事件后，服务器将中断和退出执行。此外，开发者可以向 `close` 事件传递一个 `callback` 作为监听器。

#### server.connections

该属性是一个数值，表示服务器的并发连接数。

#### server.getTicketKeys()

该方法返回一个 Buffer 实例，示例中包含 TLS Session Tickets 加密和解密所用到的密钥。

#### server.listen(port[, hostname][, callback])

该方法根据指定的 `port` 和 `hostname` 接收连接。当没有传入 `hostname` 时，如果 IPv6 可用，那么服务器就会接收任何来自 IPv6 地址（`::`）的连接，否则接收 IPv4 地址（`0.0.0.0`）。如果 `port` 的值为 0，则系统分配一个随机端口。

该方法是异步执行的，当服务器被绑定后系统会调用回调函数 `callback`。

#### server.setTicketKeys(keys)

该方法用于更新 TLS Session Tickets 加密和解密所用到的密钥。

注意，buffer 的大小必须为 48 字节，更多使用信息请查看 `ticketKeys` 的可选配置。

注意，修改后的密钥只对新的服务器连接有效，现有的或当前正在执行的服务器连接不受影响。

##### server.maxConnections

该属性可以限制服务器的连接数。

## Class: tls.TLSSocket

该类是 `net.Socket` 的包装类，用于处理写入数据的透明传输和所有的 TLS 会话协商。

`tls.TLSSocket` 的实例实现双工 Stream 接口，该实例拥有 stream 实例常见的方法和事件。

当连接打开时，返回 TLS 连接原数据的该类成员方法只会返回数据。

#### new tls.TLSSocket(socket[, options])

该构造器根据传入的 TCP socket 创建一个新的 TLSSocket。

参数 `socket` 是 net.Socket 的实例。

可选参数 `options` 是一个对象，包含以下属性：

- `secureContext`，来自 tls.createSecureContext() 的 TLS 上下文对象
- `isServer`，如果值为 true，则 TLS socket 将会在服务器模式下初始化，默认值是 false
- `server`，可选的额 net.Server 实例
- `requestCert`，可选，参见 tls.createSecurePair()
- `rejectUnauthorized`，可选，参见 tls.createSecurePair()
- `NPNProtocols`，可选，参见 tls.createServer()
- `ALPNProtocols`，可选，参见 tls.createServer()
- `SNICallback`，可选，参见 tls.createServer()
- `session`，可选，一个包含 TLS 会话的 Buffer 实例
- `requestOCSP`，可选，如果值为 `true`，则 OCSP 状态请求扩展将会被添加到 client hello 中，并在安全通讯创建之前在 socket 上触发 `OCSPResponse` 事件

#### 事件：'OCSPRespnse'

- `function (response) {}`

如果设置了 `requestOCSP` 属性就会触发该事件。`response` 是一个 Buffer 对象，包含了服务器的 OCSP 响应。

传统上来说，`response` 是一个服务器 CA 签发的签发对象，包含了服务器证书吊销状态等信息。

#### 事件：'secureConnect'

当连接成功握手后会触发该事件。无论服务器的证书是否授权，都会调用该事件监听器。如果是否测试 `tlsSocket.authorized` 来查看服务器证书的合法性完全取决于开发者。如果 `tlsSocket.authorized === false`，那么就可以从 `tlsSocket.authorizationError` 查看错误信息。此外，如果使用了 ALPN 或 NPN，那么可以通过 `tlsSocket.alpnProtocal` 或 `tlsSocket.npnProtocol` 来查看会话协商协议。

#### tlsSocket.address()

该方法返回一个对象，包含操作系统提供的绑定地址、地址族和服务器端口等信息。常用于查找那个端口已被系统绑定。该对象的常见格式：`{ port: 12346, family: 'IPv4', address: '127.0.0.1' }`。

#### tlsSocket.authorized

该属性是一个布尔值，如果对等实体证书（peer certificate）由指定的 CA 签发，那么就会返回 true，否则返回 false。

#### tlsSocket.authorizationError

该属性表示对等实体证书验证失败的原因。只有当 `tlsSocket.authorized === false` 时，该属性才会可用。

#### tlsSocket.encrypted

该属性是一个静态的布尔值，总是等于 true。常用于区分 TLS socket 和常规对象。

#### tlsSocket.getCipher()

该方法返回一个对象，用于白哦是加密名和第一个定义该加密名的 SSL/TLS 的协议版本，比如 `{ name: 'AES256-SHA', version: 'TLSv1/SSLv3' }`.

更多信息请参考 [https://www.openssl.org/docs/manmaster/ssl/SSL_CIPHER_get_name.html](https://www.openssl.org/docs/manmaster/ssl/SSL_CIPHER_get_name.html) 中的 `SSL_CIPHER_get_name()` 和 `SSL_CIPHER_get_version()`。

#### tlsSocket.getEphemeralKeyInfo()

该方法返回一个对象，表示客户端完全前向保密连接的 ephemeral 密钥交换参数中的类型、名称和大小。如果不是 ephemeral 密钥交换则返回一个空对象。由于只支持客户端 socket，所以如果在服务器 socket 上调用该方法则会返回 null。该方法支持的类型包括 `DH` 和 `ECDH`。`name` 属性只在 `ECDH` 中可用：

```js
{ type: 'ECDH', name: 'prime256v1', size: 256 }
```

#### tlsSocket.getPeerCertificate([detailed])

该方法返回一个包含对等实体证书的对象，该对象包含的属性与证书的内容相对应。如果 `detailed === true`，则返回包含 `issuer` 属性的完整链路，如果返回的只是没有 issuer 属性的顶级证书，则返回 false。

```js
{ subject:
   { C: 'UK',
     ST: 'Acknack Ltd',
     L: 'Rhys Jones',
     O: 'node.js',
     OU: 'Test TLS Certificate',
     CN: 'localhost' },
  issuerInfo:
   { C: 'UK',
     ST: 'Acknack Ltd',
     L: 'Rhys Jones',
     O: 'node.js',
     OU: 'Test TLS Certificate',
     CN: 'localhost' },
  issuer:
   { ... another certificate ... },
  raw: < RAW DER buffer >,
  valid_from: 'Nov 11 09:52:22 2009 GMT',
  valid_to: 'Nov  6 09:52:22 2029 GMT',
  fingerprint: '2A:7A:C2:DD:E5:F9:CC:53:72:35:99:7A:02:5A:71:38:52:EC:8A:DF',
  serialNumber: 'B9B0D332A1AA5635' 
}
```

如果对等实体没有提供证书，则返回 null 或空对象。

#### tlsSocket.getProtocol()

该方法返回一个字符串，表示当前连接会话协商后的 SSL/TLS 协议版本。如果已连接 socket 没有完整的握手进程，则返回 `unknown`；如果是服务器连接或未连接的客户端 socket，则返回 null：

```js
'SSLv3'
'TLSv1'
'TLSv1.1'
'TLSv1.2'
'unknown'
```

更多信息请查看 [https://www.openssl.org/docs/manmaster/ssl/SSL_get_version.html]( https://www.openssl.org/docs/manmaster/ssl/SSL_get_version.html)。

#### tlsSocket.getSession()

该方法返回使用 ASN.1 加密后的 TLS 会话，如果没有经过会话协商，则返回 `undefined`。当重连到服务器时，使用 Cloud 可以加快握手过程。

#### tlsSocket.getTLSTicket()

注意，该方法只对客户端 TLS socket 有效，常用于调试程序，通过向 `tls.connect()` 传递 `session` 参数可以复用会话。

该方法返回使用 TLS session ticket，如果没有经过会话协商，则返回 `undefined`。

#### tlsSocket.localAddress

该属性是一个字符串，表示本地 IP 地址。

#### tlsSocket.localPort

该属性是一个数值，表示本地端口。

#### tlsSocket.remoteAddress

该属性是一个字符串，表示远程 IP 地址，比如 `74.125.127.100` 或 `2001:4860:a005::68`。

#### tlsSocket.remoteFamily

该属性是一个字符串，表示远程 IP 族，比如 `IPv4` 或 `IPv6`。

#### tlsSocket.remotePort

该属性是一个数值，表示远程端口，比如 `443`。

#### tlsSocket.renegotiate(options, callback)

该方法用于初始化 TLS 会话协商进程，其中 `options` 参数包含了 rejectUnauthorized、requestCert 字段。如果会话协商成功，则 `callback(err)` 中的 `err` 的值为 `null`。

注意，在安全连接建立之后，该方法可用于请求对等实体证书。

注意，当在服务端运行时，如果超过了 `handshakeTimeout` 的限制时间，则 socket 会被销毁并抛出错误。

#### tlsSocket.setMaxSendFragment(size)

该方法用于设置 TLS 片段的最大值，默认值和最大值都是 16384，最小值是 512，如果设置成功则返回 true，否则返回 false。

片段体积越小，则客户端缓存的等待时间越短：大体积的片段会被 TLS 层缓存，直到所有的片段都被接收且通过了完整性测试；大体积的片段需要多次往返传输，且处理进程会由于保丢失或重排序等原因被延迟。不过，小体积的片段增加了 TLS 帧字节和 CPU 负载，这也会降低服务器的吞吐量。

## tls.connect(options[, callback])
## tls.connect(port[, host][, options][, callback])

该方法根据 `port` 和 `host` 或 `options.port` 和 `options.host`（如果没有传入 host 信息，则使用 localhost）创建一个新的客户端连接。`options` 是一个包含以下属性的对象：

- `host`，客户端需要连接的主机
- `port`，客户端需要连接的端口
- `socket`，根据指定的 socket 创建安全连接，而不是创建新的 socket。如果指定了该参数，则系统会忽略 host 和 port 参数
- `path`，创建连接到该路径的 Unix socket，如果指定了该参数，则系统会忽略 host 和 port 参数
- `pfx`，该参数是一个字符串或 Buffer 实例，包含了 PFX 或 PKCS12 格式的客户端私钥、证书和 CA 证书
- `key`，该参数是一个字符串或 Buffer 实例，包含了 PEM 格式的客户端私钥
- `passphrase`，该参数是一个字符串格式的私钥或 pfx 密码
- `cert`，该参数是一个字符串或 Buffer 实例，包含了 PEM 格式的客户端私钥
- `ca`，该参数是一个 PEM 格式的字符串、Buffer 实例、字符串数组，用于表示可信证书。如果没有设置该参数，则系统使用众多周知的几个根 CA，比如 VeriSign。它们常用于授权连接。
- `ciphers`，该参数是一个字符串，指定了使用或不使用的密码，使用 `:` 分隔。使用的默认密码和 `tls.createServer()` 一样
- `rejectUnauthorized`，如果值为 true，则根据指定的 CA 列表校验服务器整数。如果校验失败，则触发 `error ` 事件，其中 `err.code` 表示 OpenSSL 的错误代码，默认值为 true
- `NPNProtocols`，该参数是一个字符串数组或 Buffer 实例，表示支持的 NPN 协议。Buffer 实例需要遵循类似 `0x05hello0x05world` 的格式，即第一个字节是下一个协议名的长度（传入的数组则简单多了：`['hello', 'world']`）。
- `ALPNProtocols`，该参数是一个字符串数组或 Buffer 实例，表示支持的 ALPN 协议。Buffer 实例需要遵循类似 `0x05hello0x05world` 的格式，即第一个字节是下一个协议名的长度（传入的数组则简单多了：`['hello', 'world']`）。
- `servername`，用于 SNI TLS 扩展的服务器名字
- `checkServerIndentity(servername, cert)`，该方法覆盖了系统的相关功能，用来根据证书检查服务器的主机名。如果校验失败，则返回一个错误，否则返回 udnefined
- `secureProtocol`，使用的 SSL 方法，比如 `SSLv3_method` 强制使用 SSL 第三版。该属性的值取决于安装的 OpenSSL 和 SSL_METHODS 常量
- `secureContext`，一个来自 `tls.createSecureContext(...)` 的可选 TLS 上下文对象。可用于缓存客户端证书、密钥和 CA 证书
- `session`，一个包含 TLS 会话的 Buffer 实例
- `minDHSize`，允许接收的 TLS 连接的 DH 参数的最小字节量。如果服务器提供的 DH 参数小于该值，则 TLS 连接被销毁并抛出错误，默认值为 1024

回调函数 `callback` 会被绑定为 `secureConnect` 事件的事件处理器。

`tls.connect()` 返回一个 `tls.TLSSocket` 对象。

下面代码演示了一个 echo 服务器：

```js
const tls = require('tls');
const fs = require('fs');

const options = {
  // These are necessary only if using the client certificate authentication
  key: fs.readFileSync('client-key.pem'),
  cert: fs.readFileSync('client-cert.pem'),

  // This is necessary only if the server uses the self-signed certificate
  ca: [ fs.readFileSync('server-cert.pem') ]
};

var socket = tls.connect(8000, options, () => {
  console.log('client connected',
              socket.authorized ? 'authorized' : 'unauthorized');
  process.stdin.pipe(socket);
  process.stdin.resume();
});
socket.setEncoding('utf8');
socket.on('data', (data) => {
  console.log(data);
});
socket.on('end', () => {
  server.close();
});
```

或者：

```js
const tls = require('tls');
const fs = require('fs');

const options = {
  pfx: fs.readFileSync('client.pfx')
};

var socket = tls.connect(8000, options, () => {
  console.log('client connected',
              socket.authorized ? 'authorized' : 'unauthorized');
  process.stdin.pipe(socket);
  process.stdin.resume();
});
socket.setEncoding('utf8');
socket.on('data', (data) => {
  console.log(data);
});
socket.on('end', () => {
  server.close();
});
```

## tls.createSecureContext(details)

该方法用于创建一个凭证对象，其中 `details` 包含以下属性:

- `pfx`，该属性是一个字符串或 Buffer 实例，包含了 PFX 或 PKCS12 格式的客户端私钥、证书和 CA 证书
- `key`，该属性是一个字符串或 Buffer 实例，包含了 PEM 格式的客户端私钥。如果值为数组，则支持多个密钥使用不同的算法。该值也可以是纯密钥数组，或对象数组，比如 `{pem: key, passphrase: passphrase}`
- `passphrase`，该属性是一个字符串格式的私钥或 pfx 密码
- `cert`，该属性是一个字符串或 Buffer 实例，包含了 PEM 格式的客户端私钥
- `ca`，该属性是一个 PEM 格式的字符串、Buffer 实例、字符串数组，用于表示可信证书。如果没有设置该属性，则系统使用众多周知的几个根 CA，比如 VeriSign。它们常用于授权连接。
- `crl`，该属性要么是一个字符串，要么是一个 PEM 格式的 CRL 列表
- `ciphers`，该属性是一个字符串，指定了使用或不使用的密码，更多有关格式的信息请参考 [https://www.openssl.org/docs/apps/ciphers.html#CIPHER_LIST_FORMAT](https://www.openssl.org/docs/apps/ciphers.html#CIPHER_LIST_FORMAT)
- `honorCipherOrder`，当选择加密算法时，使用服务器配置替代客户端配置。

如果没有指定 `ca`，那么 Node.js 就会使用默认的、来自 [http://mxr.mozilla.org/mozilla/source/security/nss/lib/ckfw/builtins/certdata.txt](http://mxr.mozilla.org/mozilla/source/security/nss/lib/ckfw/builtins/certdata.txt) CA 公信列表。

## tls.createSecurePair([context][, isServer][, requestCert][, rejectUnauthorized][, options])

该方法新建一个包含两个 stream 的安全对对象，其中一个用于读写加密数据，一个用于读写明文数据。通常来说，加密 stream 用于传递或接受加密数据流，明文 stream 用于替代初始化加密 stream。

- `credentials`，一个来自 `tls.createSecureContext()` 的安全上下文对象
- `isServer`，该参数是一个布尔值，用于表示是否以服务器或客户端代开当前的 tls 连接
- `requestCert`，该参数是一个布尔值，用于表示服务器是否应该从连接的客户端请求证书，只对服务器连接有效。
- `rejectUnauthorized`，该参数是一个布尔值，用于表示服务器是否应该自动拒绝携带无效证书的客户端，只对开启 `requestCert` 选项的服务器有效
- `options`，一个包含常见 SSL 选项的对象

`tls.createSecurePair()` 返回一个 SecurePair 对象，该对象包含了 `cleartext` 和 `encrypted` stream 属性。

注意，`cleartext` 和 `tls.TLSSocket` 具有相同的 API。

## tls.createServer(options[, secureConnectionListener])

该方法用于创建一个新的 `tls.Server`。`secureConnectionListener` 会被绑定为 `secureConnection` 事件的监听器。`options` 对象包含以下属性：

- `pfx`，该属性是一个字符串或 Buffer 实例，包含了 PFX 或 PKCS12 格式的客户端私钥、证书和 CA 证书
- `key`，该属性是一个字符串或 Buffer 实例，包含了 PEM 格式的客户端私钥。如果值为数组，则支持多个密钥使用不同的算法。该值也可以是纯密钥数组，或对象数组，比如 `{pem: key, passphrase: passphrase}`
- `passphrase`，该属性是一个字符串格式的私钥或 pfx 密码
- `cert`，该属性是一个字符串或 Buffer 实例，包含了 PEM 格式的客户端私钥
- `ca`，该属性是一个 PEM 格式的字符串、Buffer 实例、字符串数组，用于表示可信证书。如果没有设置该属性，则系统使用众多周知的几个根 CA，比如 VeriSign。它们常用于授权连接。
- `crl`，该属性要么是一个字符串，要么是一个 PEM 格式的 CRL 列表
- `ciphers`，该参数是一个字符串，指定了使用或不使用的密码，使用 `:` 分隔，默认的加密套件包括：

```js
ECDHE-RSA-AES128-GCM-SHA256:
ECDHE-ECDSA-AES128-GCM-SHA256:
ECDHE-RSA-AES256-GCM-SHA384:
ECDHE-ECDSA-AES256-GCM-SHA384:
DHE-RSA-AES128-GCM-SHA256:
ECDHE-RSA-AES128-SHA256:
DHE-RSA-AES128-SHA256:
ECDHE-RSA-AES256-SHA384:
DHE-RSA-AES256-SHA384:
ECDHE-RSA-AES256-SHA256:
DHE-RSA-AES256-SHA256:
HIGH:
!aNULL:
!eNULL:
!EXPORT:
!DES:
!RC4:
!MD5:
!PSK:
!SRP:
!CAMELLIA
```

默认的加密套件使用 GCM 支持 Chrome's modern cryptography setting，使用 ECDHE 和 DHE 加密完全前向保密，并提供了一些向后兼容性。

考虑到 [specific attacks affecting larger AES key sizes](https://www.schneier.com/blog/archives/2009/07/another_new_aes.html)，128 位的 AES 加密算法要优于 192 位或 256 位的 AES。

对于那些使用了不安全或已抛弃的加密算法（RC4、基于 DES）并不能使用默认的配置完成握手。如果开发者一定要支持这些浏览器，可以参考 [TLS 建议的兼容性加密套件](https://wiki.mozilla.org/Security/Server_Side_TLS)，更多信息请参考 [OpenSSL cipher list format documentation](https://www.openssl.org/docs/apps/ciphers.html#CIPHER_LIST_FORMAT)。

- `ecdhCurve`，该参数是一个字符串，表示一个用于 ECDH 密钥协商的 named curve，或者是一个 false，表示禁用 ECDH。默认值是 `prime256v1`（NIST P-256）。使用 `crypto.getCurves()` 可以后去可用的 curve name 列表。在最新的版本中，使用 `openssl ecparam -list_curves` 命令也可以显示可用 elliptic curve 的名字和描述信息。
- `dhparam`，该参数是一个包含 Diffie Hellman 参数的字符串或 Buffer 实例，用于完全前向保密。使用 `openssl dhparam` 可以创建该参数。它的密钥长度要大于等于 1024 位，否则系统会抛出异常。强烈建议使用 2048 位或更长的长度增强安全性。如果忽略了该属性或者为无效属性，则会被系统忽略且不能使用 DHE 加密算法。
- `handshakeTimeout`，如果 SSL/TLS 握手时间超时就会终止连接，默认值为 120 秒。当握手超时时，`tls.Server` 对象会触发一个 `clientError` 的事件
- `honorCipherOrder`，如果值为 true，则在选在加密算法时，使用服务器的配置替代客户端的配置
- `requestCert`，如果值为 true，则服务器将会请求客户端发送证书并校验证书，默认值为 false
- `rejectUnauthorized`，如果值为 true，则根据指定的 CA 列表校验服务器证书。只有当 `requestCert` 的值为 true 时，该参数才会生效，默认值为 false
- `NPNProtocols`，该参数是一个字符串数组或 Buffer 实例，表示支持的 NPN 协议。
- `ALPNProtocols`，该参数是一个字符串数组或 Buffer 实例，表示支持的 ALPN 协议。当服务器同时从客户端接收到 NPN 和 ALPN 扩展时，ALPN 的优先级高于 NPN，且服务器不会向客户端发送 NPN 扩展
- `SNICallback(serveranme, cb)`，如果客户端支持 SNI TLS 扩展就会调用该方法。该方法接收两个参数：`servername` 和 `cb`。`SNICallback` 应该调用 `cb(null, ctx)`，其中 `ctx` 是一个 SecureContext 实例。如果没有指定 `SNICallback`，系统就会使用默认的高阶 API 回调函数。
- `sessionTimeout`，该参数是一个整数，用于指定服务器创建 TLS 会话标识符和 TLS session tickets 的超时时间。
- `ticketKeys`，该参数是一个 48 字节的 Buffer 实例，由 16 字节的前缀、16 字节的 hmac 密钥和 16 字节的 AES 密钥组成。开发者可以使用它从多个 tls 服务器接收 session tickets。注意，该属性会自动在集群模块的 worker 之间共享。
- `sessionIdContext`，该参数是一个字符串，包含了一个不透明的会话恢复标识符。如果 `requestCert === true`，则默认值为命令行生成的 MD5 哈希值（在 FIPS 模式下，使用缩减的 SHA1 哈希值）。否则，不提供默认值。
- `secureProtocol`，使用的 SSL 方法，比如 `SSLv3_method` 强制使用 SSL 第三版。该属性的值取决于安装的 OpenSSL 和 SSL_METHODS 常量

下面代码演示了一个 echo 服务器：

```js
const tls = require('tls');
const fs = require('fs');

const options = {
  key: fs.readFileSync('server-key.pem'),
  cert: fs.readFileSync('server-cert.pem'),

  // This is necessary only if using the client certificate authentication.
  requestCert: true,

  // This is necessary only if the client uses the self-signed certificate.
  ca: [ fs.readFileSync('client-cert.pem') ]
};

var server = tls.createServer(options, (socket) => {
  console.log('server connected',
              socket.authorized ? 'authorized' : 'unauthorized');
  socket.write('welcome!\n');
  socket.setEncoding('utf8');
  socket.pipe(socket);
});
server.listen(8000, () => {
  console.log('server bound');
});
```

或者：

```js
const tls = require('tls');
const fs = require('fs');

const options = {
  pfx: fs.readFileSync('server.pfx'),

  // This is necessary only if using the client certificate authentication.
  requestCert: true,

};

var server = tls.createServer(options, (socket) => {
  console.log('server connected',
              socket.authorized ? 'authorized' : 'unauthorized');
  socket.write('welcome!\n');
  socket.setEncoding('utf8');
  socket.pipe(socket);
});
server.listen(8000, () => {
  console.log('server bound');
});
```

使用 `openssl s_client` 可以连接到服务器进行测试：

```bash
openssl s_client -connect 127.0.0.1:8000
```

## tls.getCiphers()

该方法返回一个数组，数组内包含系统支持的 SSL 加密算法的名称：

```js
var ciphers = tls.getCiphers();
console.log(ciphers); // ['AES128-SHA', 'AES256-SHA', ...]
```

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