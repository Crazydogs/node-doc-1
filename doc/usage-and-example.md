## 使用方式

```bash
node [options] [v8 options] [script.js | -e "script"] [arguments]
```

详细信息请参考本文档 Cammand Line Options 一章。

## 概述

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

将上述代码保存到 `example.js` 文件中，然后在终端中使用 `node` 命令解析执行：

```bash
$ node example.js
Server running at http://127.0.0.1:3000/
```

文档中的所有示例都可以使用这种方法进行测试。