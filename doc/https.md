## HTTPS

<div class="s s2"></div>

HTTPS 是一个基于 TLS/SSL 的 HTTP。在 Node.js 中，封装了一个单独的 HTTPS 模块。

## Class: https.Agent

该类与 `http.Agent` 类似。

## Class: https.Server

该类是 `tls.Server` 的子类，其可以触发的事件和 `http.Server` 相似。

#### server.setTimeout(msecs, callback)  

该方法与 `http.Server#setTimeout()` 相似。

#### server.timeout

该方法与 `http.Server#timeout()` 相似。

## https.createServer(options[, requestListener])

该方法返回一个新的 HTTPS web 服务器对象。`options` 参数与 `tls.createServer()` 中的 `options` 参数类似。`requestListener` 是一个参数，会被系统自动设为 `request` 事件的监听器。

```js
// curl -k https://localhost:8000/
const https = require('https');
const fs = require('fs');

const options = {
  key: fs.readFileSync('test/fixtures/keys/agent2-key.pem'),
  cert: fs.readFileSync('test/fixtures/keys/agent2-cert.pem')
};

https.createServer(options, (req, res) => {
  res.writeHead(200);
  res.end('hello world\n');
}).listen(8000);

// or
const https = require('https');
const fs = require('fs');

const options = {
  pfx: fs.readFileSync('server.pfx')
};

https.createServer(options, (req, res) => {
  res.writeHead(200);
  res.end('hello world\n');
}).listen(8000);
```

#### server.close()

该方法与 `http.close()` 方法类似。

#### server.listen(handle[, callback])
#### server.listen(path[, callback])
#### server.listen(port[, host][, backlog][, callback])

上述方法与 `http.listen()` 方法类似。

## https.get(options, callback)

该方法与 `http.get()` 方法类似，但只用于 HTTPS。

`options` 可以是对象或字符串。如果 `options` 是字符串，则系统自动调用 `url.parse()` 方法解析该参数。

```js
const https = require('https');

https.get('https://encrypted.google.com/', (res) => {
  console.log('statusCode: ', res.statusCode);
  console.log('headers: ', res.headers);

  res.on('data', (d) => {
    process.stdout.write(d);
  });

}).on('error', (e) => {
  console.error(e);
});
```

## https.globalAgent

该属性是全局性的 `https.Agent` 实例，是所有 HTTPS 客户端的请求目标。

## https.request(options, callback)

该方法用于向安全的 web 服务器发送请求。

`options` 可以是对象或字符串。如果 `options` 是字符串，则系统自动调用 `url.parse()` 方法解析该参数。

所有能被 `http.request()` 接受的 `options` 对 `https.request()` 都是有效的。

```js
const https = require('https');

var options = {
  hostname: 'encrypted.google.com',
  port: 443,
  path: '/',
  method: 'GET'
};

var req = https.request(options, (res) => {
  console.log('statusCode: ', res.statusCode);
  console.log('headers: ', res.headers);

  res.on('data', (d) => {
    process.stdout.write(d);
  });
});
req.end();

req.on('error', (e) => {
  console.error(e);
});
```

`options` 参数包含以下可选项：

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

也可以指定下述来自 `tls.connect()` 的可选参数。不过，`globalAgent` 会忽略这些选项。

- `pfx`，SSL 使用的证书、私钥和 CA 证书，默认值为 `null`
- `key`，SSL 使用的私钥，默认值为 `null`
- `passphrase`，字符串，私钥或 pfx 的口令，默认值为 `null`
- `cert`，使用的公有 x509 证书，默认值为 `null`
- `ca`，字符串、Buffer 实例、字符串数组或具有 PEM 格式可信证书的 Buffer 实例。如果并不是这几种类型，则使用 "root" CA，比如 VeriSign。它们常用于授权链接。
- `ciphers`，使用或拒用的字符串密码，具体格式请参考 [https://www.openssl.org/docs/apps/ciphers.html#CIPHER_LIST_FORMAT](https://www.openssl.org/docs/apps/ciphers.html#CIPHER_LIST_FORMAT)
- `rejectUnauthorized`，如果该值为 true，则使用提供的 CA 列表验证服务器证书，当验证失败时触发 `error` 事件。验证发生在 HTTP 请求发送之前的链接阶段。该属性的默认值为 true。
- `secureProtocal`，指定使用的 SSL 方法，比如 `SSLv3_method` 强制使用 SSL v3。具体的值取决于安装的 OpenSSL 和 `SSL_METHODS` 变量
- `servername`，SNI（Server Name INdication） TLS 扩展的服务器名字。

为了使用上述可选项，可以创建一个自定义的 `Agent`。

```js
var options = {
  hostname: 'encrypted.google.com',
  port: 443,
  path: '/',
  method: 'GET',
  key: fs.readFileSync('test/fixtures/keys/agent2-key.pem'),
  cert: fs.readFileSync('test/fixtures/keys/agent2-cert.pem')
};
options.agent = new https.Agent(options);

var req = https.request(options, (res) => {
  ...
}
```

或者不适用 `Agent` 退出连接池：

```js
var options = {
  hostname: 'encrypted.google.com',
  port: 443,
  path: '/',
  method: 'GET',
  key: fs.readFileSync('test/fixtures/keys/agent2-key.pem'),
  cert: fs.readFileSync('test/fixtures/keys/agent2-cert.pem'),
  agent: false
};

var req = https.request(options, (res) => {
  ...
}
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