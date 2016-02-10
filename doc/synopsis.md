## 概述

使用 Node.js 创建 web 服务器：

```js
const http = require('http')

http.createServer( (req, res) => {
    res.writeHead(200, {
        'Content-Type': 'text/plain'
    })
    res.end('Hello World\n')
}).listen(8124)

console.log('Server running at http://localhost:8124')
```

将上述代码保存到 `example.js` 文件中，然后在终端中使用 `node` 命令解析执行：

```bash
$ node example.js
Server running at http://127.0.0.1:8124/
```

文档中的所有示例都可以使用这种方法进行测试。