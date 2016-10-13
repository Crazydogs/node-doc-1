## 使用方式

```sh
node [options] [v8 options] [script.js | -e "script"] [arguments]
```

有关 options 的详细信息请参考 [《命令行参数》]() 一章。

## 示例

使用 Node.js 创建 web 服务器：

```js
const http = require('http');

const hostname = '127.0.0.1';
const port = 3000;

const server = http.createServer((req, res) => {
    res.statusCode = 200;
    res.setHeader('Content-Type', 'text/plain');
    res.end('Hello World\n');
});

server.listen(port, hostname, () => {
    console.log(`Server running at http://${hostname}:${port}/`);
});
```

将上述代码保存为 `example.js`，然后在终端中调用 `node` 命令解释执行：

```sh
$ node example.js
Server running at http://127.0.0.1:3000/
```

文档中的所有示例都可以使用上面的方法进行测试。
