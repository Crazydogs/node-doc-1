## URL

该模块提供了解析 URL 的辅助函数，使用 `require('url')` 调用。

## 解析 URL

解析后的 URL 对象通常具有以下的部分或全部字段。如果 URL 字符串中没有，则解析后的 URL 对象也不会存在，下面是一个 URL 的示例：

```js
'http://user:pass@host.com:8080/p/a/t/h?query=string#hash'
```

- `href`，待解析的、完整的 URL，其中协议和主机都是小写形式，举例：'http://user:pass@host.com:8080/p/a /t/h?query=string#hash’
- `protocol`，请求协议，举例：'http:'
- `slashes`，协议冒号之后是否需要反斜线，举例：true or false
- `host`，URL 的主机部分，包含端口信息，举例：'host.com:8080'
- `auth`，URL 的授权信息部分，举例：'user:pass'
- `hostname`，主机的主机名部分，举例：'host.com'
- `port`，主机的端口号，举例：'8080'
- `pathname`，URL 的路径部分，位于主机之后、查询字符串之前，如果该值存在则第一个字符为反斜线，不对字符串解码，举例：'/p/a/t/h'
- `search`，URL 的查询字符串部分, 以问号开头，举例：'?query=string'
- `path`，URL 的路径名和查询字符串，不对字符串解码，举例：'/p/a/t/h?query=string'
- `query`，查询字符串中的参数部分，或者是使用 querystring 解析后的对象，举例：'query=string' or {'query':'string'}
- `hash`，以 # 开头的 URL 片段，举例：'#hash'

#### 转义字符

默认情况下，如果有空格和以下字符出现在 URL 中，则会被自动转义：

```js
< > " ` \r \n \t { } | \ ^ '
```

## url.format(urlObj)

接收一个解析后的 URL 对象，返回一个格式化的 URL 字符串。

下面是格式化流程：

- `href` 会被忽略
- `path` 会被忽略
- `protocol`，允许缺少冒号，只要存在 host 或者 hostname 字段，`http` / `https` / `ftp` / `gopher` / `file` 协议就会被添加 `://` 后缀，其他协议会被添加 `:` 后缀，比如 `mailto` / `xmpp` / `aim` / `sftp` / `foo` 协议等
- `slashes`，如果缺少 host 或者 hostname 字段、协议不属于默认添加 `://` 后缀的协议且需要添加 `://` 后缀，则需要将该值赋值为 true 如果协议需要需要 `://`
- `auth`，存在即有效
- `hostname`，host 缺失时有效
- `port`，host 缺失时有效
- `host`，用来替代 hostname 和 port
- `pathname`，无论是否以反斜线开头，都做相同的梳理
- `query`，search 缺失时有效
- `search`，用于替代 query，无论是否以问号开头，都会做通常的处理
- `hash`，无论是否以 # 开头，都会做通常的处理

## url.parse(urlStr[, parseQueryString][, slashesDenoteHost])

该方法用于接收一个 URL 字符串，返回一个 URL 对象。

如果第二个参数的值为 true，则使用 `queryString` 模块解析查询字符串，`query` 属性指向一个对象，`search` 属性指向一个字符串。如果值为 false，`query` 属性不会被解析或编码。默认值为 false。

如果第三个参数的值为 true，则将 `//foo/bar` 解析为 `{ host: 'foo', pathname: '/bar' }`，而不是 `{ pathname: '//foo/bar' }`。默认值为 false。

## url.resolve(from, to)

接收一个 base URL 和一个 href URL，返回一个跳转后的链接地址：

```js
url.resolve('/one/two/three', 'four')         // '/one/two/four'
url.resolve('http://example.com/', '/one')    // 'http://example.com/one'
url.resolve('http://example.com/one', '/two') // 'http://example.com/two'
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