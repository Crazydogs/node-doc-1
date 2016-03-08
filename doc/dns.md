## DNS

<div class="s s2"></div>

`dns` 模块封装的函数主要分为两类：

1. 第一类函数使用操作系统底层提供的功能解析 DNS，无需执行任何网络通信。这一类主要包含一个函数：`dns.lookup()`。开发者如果想像同一操作系统下的其他应用程序一样解析域名，那么就应该使用 `dns.lookup()`。

```js
const dns = require('dns');

dns.lookup('nodejs.org', (err, addresses, family) => {
  console.log('addresses:', addresses);
});
```

1. 第二类函数需要连接到 DNS 服务器解析域名，而且需要一直使用网络解析 DNS 请求。除 `dns.lookup()` 函数之外，`dns` 模块中的其他函数都属于这一类。这些函数所使用的配置文件与 `dns.lookup()` 的配置文件（比如 `/etc/hosts`）不同。不愿意使用操作系统底层提供的域名解析机制的开发者可以使用此类函数。

下面代码演示了如何解析 `nodejs.org` 并反向解析返回的 IP 地址：

```js
const dns = require('dns');

dns.resolve4('nodejs.org', (err, addresses) => {
  if (err) throw err;

  console.log(`addresses: ${JSON.stringify(addresses)}`);

  addresses.forEach((a) => {
    dns.reverse(a, (err, hostnames) => {
      if (err) {
        throw err;
      }
      console.log(`reverse for ${a}: ${JSON.stringify(hostnames)}`);
    });
  });
});
```

## dns.getServers()

该方法返回一个字符串数组，数组内容是用于域名解析的 IP 地址。

## dns.lookup(hostname[, options], callback)

该方法用于将主机名（比如 `nodejs.org`）解析为第一个匹配到的 A 记录（IPv4）或 AAAA 记录（IPv6）。可选参数 `options` 可以是对象或整数。如果未指定 `options` 参数，那么 IPv4 和 IPv6 地址都有效。如果 `options` 是一个整数，那么它必须是 `4` 或 `6`。

如果 `options` 是一个对象，那么它需要包含以下属性：

- `family`，数值，协议族，如果存在，值必须为整数 4 或 6；如果不存在，则表示同时接收 IPv4 和 IPv6
- `hints`，数值，如果存在，则该值为一个或多个的 `getaddrinfo` 标记；如果不存在，表示不向 `getaddrinfo` 传递标记。多个标记使用 `OR` 运算符分隔。
- `all`，布尔值，如果值为 true，则回调函数返回一个数组，该数组包含所有解析后的地址；否则返回一个单一地址。默认值为 false

所有的属性都是可选的，下面是一个简单示例：

```js
{
  family: 4,
  hints: dns.ADDRCONFIG | dns.V4MAPPED,
  all: false
}
```

`callback` 回调函数接收三个参数 `(err, address, family)`，其中 `address` 是一个字符串形式的 IPv4 或 IPv6 地址，`family` 参数的值为 4 或 6，用于表示 `address` （无需初始化时传递给 `lookup()` 函数）的协议族。

当 `all` 的值为 true 时，该回调函数的参数将会变为 `(err, addresses)`，其中，`address` 参数是一个对象数组，数组中的每个对象都拥有 `address` 和 `family` 两个属性。

当出现错误时，`err` 参数的值为一个 Error 实例，其中 `err.code` 为错误码。需要牢记的是，不仅仅主机名不存在的时候，而且在 lookup() 函数执行错误时（没有可用的文件描述符），错误码都有可能会被赋值为 `ENOENT`。

`dns.lookup()` 并没有任何处理 DNS 协议的逻辑，它使用了操作系统所提供的功能来解析地址。对于不同的 Node.js 程序来说，该方法的执行结果可能会有略微但重要的差异。

#### 受支持的 getaddrinfo 标记

下面标记都可作为 `hints` 字段的值传递给 `dns.lookup()`：

- `dns.ADDRCONFIG`：返回当前系统支持的地址类型，比如，如果当前系统至少配置了一个 IPv4 地址，那么就会返回一个 IPv4 地址。局部回送位址（loopback address）不计在内。
- `dns.V4MAPPED`：如果指定了 IPv6 协议族，但没有找到 IPv6，那么就返回由 IPv4 映射的 IPv6 地址。注意，这一标记在某些操作系统上无效，比如 FreeBSD 10.1。

## dns.lookupServce(address, port, callback)

该方法使用操作系统的底层 `getnameinfo` 将传入的 `address` 和 `port` 解析为主机名和服务名。

`callback` 回调函数接收三个参数 `(err, hostname, service)`。这里的 `hostname` 和 `service` 参数都是字符串类型，比如 `localhost` 和 `http`。

当出现错误时，`err` 的值是一个 Error 对象，其中 `err.code` 是错误码：

```js
const dns = require('dns');
dns.lookupService('127.0.0.1', 22, (err, hostname, service) => {
  console.log(hostname, service);
    // Prints: localhost ssh
});
```

## dns.resolve(hostname[, rrtype], callback)

根据 `rrtype` 指定的记录类型使用 DNS 协议将主机名解析为一个数组。

`rrtype` 的有效值包括：

- `'A'`，IPv4 地址，rrtype 的默认值
- `'AAAA'`，IPV6 地址
- `'MX'`，邮件交换记录
- `'TXT'`，文本记录
- `'SRV'`，SRV 记录
- `'PTR'`，用来反向查找 IP
- `'NS'`，域名服务器记录
- `'CNAME'`，别名记录
- `'SOA'`，授权记录的初始值

`callback` 回调函数接收两个参数 `(err, address)`。执行成功时，`address` 参数为一个数组，数组成员的类型由记录类型决定。

当出现错误时，`err` 的值为一个 Error 对象，其中 `err.code` 是一个错误码。

## dns.resolve4(hostname, callback)

该方法使用 DNS 协议根据主机名解析 IPv4 地址（A 记录）。传入 `callback` 回调函数的 `addresses` 参数为一个 IPv4 地址的数组，比如 `['74.125.79.104', '74.125.79.105', '74.125.79.106']`。

## dns.resolve6(hostname, callback)

该方法使用 DNS 协议根据主机名解析 IPv6 地址（AAAA 记录）。传入 `callback` 回调函数的 `addresses` 参数为一个 IPv6 地址的数组。

## dns.resolveCname(hostname, callback)

该方法使用 DNS 协议根据主机名解析 `CNAME`。传入 `callback` 回调函数的 `addresses` 参数为主机名的别名，比如 `['bar.example.com']`。

## dns.resolveMx(hostname, callback)

该方法使用 DNS 协议根据主机名解析邮件交换协议（MX 记录）。传入 `callback` 回调函数的 `addresses` 参数为一个对象数组，数组中的对象都包含 `priority` 和 `exchange` 两个属性，比如 `[{priority: 10, exchange: 'mx.example.com'}, ...]`。

## dns.resolveNs(hostname, callback)

该方法使用 DNS 协议根据主机名解析域名服务器记录（NS 记录）。传入 `callback` 回调函数的 `addresses` 参数为一个主机名可用的域名服务器记录数组，比如 `['ns1.example.com', 'ns2.example.com']`。

## dns.resolveSoa(hostname, callback)

该方法使用 DNS 协议根据主机名解析授权记录的初始值（SOA 记录）。传入 `callback` 回调函数的 `addresses` 参数为一个包含以下属性的对象：

- `nsname`
- `hostmaster`
- `serial`
- `refresh`
- `retry`
- `expire`
- `minttl`

```js
{
  nsname: 'ns.example.com',
  hostmaster: 'root.example.com',
  serial: 2013101809,
  refresh: 10000,
  retry: 2400,
  expire: 604800,
  minttl: 3600
}
```

## dns.resolveSrv(hostname, callback)

该方法使用 DNS 协议根据主机名解析 SRV 记录。传入 `callback` 回调函数的 `addresses` 参数为一个对象数组，数组中的对象都包含以下属性：

- `priority`
- `weight`
- `port`
- `name`

```js
{
  priority: 10,
  weight: 5,
  port: 21223,
  name: 'service.example.com'
}
```

## dns.resolveTxt(hostanme, callback)

该方法使用 DNS 协议根据主机名解析文本查询（TXT 记录）。传入 `callback` 回调函数的 `addresses` 参数为一个二维数组，比如 `[ ['v=spf1 ip4:0.0.0.0 ', '~all' ] ]`。每一个二级数组都包含一条记录的 TXT 块。根据实际需求，可以合并和分开独立使用。

## dns.reverse(ip, callback)

该方法用于执行反向 DNS 查询，将 IPv4 或 IPv6 地址解析为一个主机名的地址。

`callback` 回调函数接收两个参数 `(err, hostnames)`，其中，`hostnames` 是根据传入的 `ip` 解析得来的主机名数组。

当出现错误时，`err` 的值是一个 Error 对象，其中 `err.code` 是错误码。

## dns.setServers(servers)

该方法用于指定一组 IP 地址作为解析服务器。`servers` 参数是一个 IPv4 或 IPv6 地址的数组。

如果地址中包含端口信息，将会被自动移除。

如果传入的地址无效，则将会抛出错误。

不能在 DNS 查询时执行 `dns.setServers()` 方法。

### 错误码

每一次 DNS 查询都会返回下列错误码中的一个：

- `dns.NODATA`: DNS 服务器返回无数据应答
- `dns.FORMERR`: DNS 服务器认为查询各式错误
- `dns.SERVFAIL`: DNS 服务器返回常规失败
- `dns.NOTFOUND`: 未找到域名
- `dns.NOTIMP`: DNS 服务器为实现当前请求的操作
- `dns.REFUSED`: DNS 服务器拒绝查询
- `dns.BADQUERY`: DNS 查询格式错误
- `dns.BADNAME`: 主机名格式错误
- `dns.BADFAMILY`: 不支持的地址族
- `dns.BADRESP`: DNS 回复格式错误
- `dns.CONNREFUSED`: 无法连接到 DNS 服务器
- `dns.TIMEOUT`: 连接 DNS 服务器超时
- `dns.EOF`: 文件结尾
- `dns.FILE`: 读取文件发生错误
- `dns.NOMEM`: 内存溢出
- `dns.DESTRUCTION`: 信道已被销毁
- `dns.BADSTR`: 字符串格式错误
- `dns.BADFLAGS`: 非法标识
- `dns.NONAME`: 给定的主机名不是数值类型
- `dns.BADHINTS`: 非法 Hints 标识
- `dns.NOTINITIALIZED`: c-ares 库尚未初始化
- `dns.LOADIPHLPAPI`: iphlpapi.dll 加载失败
- `dns.ADDRGETNETWORKPARAMS`: 无法找到 GetNetworkParams 函数
- `dns.CANCELLED`: DNS 查询已取消

## 开发提示

虽然 `dns.lookup()` 和 `dns.resolve*()/dns.reverse()` 函数都是用于关联网络名和网络地址，但它们的内部机制还是存在细微不同。差异虽小但在不同的 Node.js 程序中可能会有严重的影响。

#### dns.lookup()

本质上说，`dns.lookup()` 像大多数程序一样都是使用了操作系统的底层函数。举例来说，`dns.lookup()` 会像 `ping` 命令一样解析域名。在大多数的类 POSIX 操作系统上，可以通过修改 `nsswitch.conf(5)`或 `resolv.conf(5)` 中的配置修改 `dns.lookup()` 函数的行为，但需要注意的是，上述修改会影响所有运行在当前操作系统上的程序。

虽然从 JavaScript 的角度看 `dns.lookup()` 是异步调用，但实际上在 libuv 的线程池中是通过同步调用 `getaddrinfo(3)` 实现。因为 libuv 线程池是大小固定的，也就是说，如果因为某些因素导致 `getaddrinfo(3)` 函数长时间运行，那么线程池中其他操作的性能就会降低。为了降低这一风险，一个有效的方法是增大环境变量 `'UV_THREADPOOL_SIZE'` 的值（默认值为 4），从而增加线程池的大小。更多信息请参考[ libuv 的官方文档](http://docs.libuv.org/en/latest/threadpool.html)。

#### dns.resolve(), dns.resolve*() 和 dns.reverse()

这些方法和 `dns.lookup()` 的机制完全不一样。它们不使用 `getaddrinfo(3)`，而是通过网络进行 DNS 查询。通常来说，这里的网络通信都是异步执行的，并不会使用到 libuv 的线程池。

因此，这些函数的操作不会对 libuv 线程池中的其他进程产生类似 `dns.lookup()` 函数所产生的负面影响。

这些函数也不会用到 `dns.lookup()` 函数所用到的配置文件，比如，它们不会使用 `/etc/hosts` 的配置文件。

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