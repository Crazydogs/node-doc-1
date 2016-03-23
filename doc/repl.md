## REPL

<div class="s s2"></div>

Read-Eval-Print-Loop（REPL）既可用作独立的程序，也可以集成到其他程序中。REPL 提供了交互式运行 JavaScript 的方式，可以用来调试或测试代码。

从命令行执行 `node` 命令就可以进入 REPL，该 REPL 提供了一个简化版的 emacs 行编辑模式：

```js
$ node
Type '.help' for options.
> a = [ 1, 2, 3];
[ 1, 2, 3 ]
> a.forEach((v) => {
...   console.log(v);
...   });
1
2
3
```

如果要使用高级行编辑器，可以通过环境变量 `NODE_NO_READLINE=1` 启动 Node.js，在该编辑器中可以使用 `rlwrap`。

此外，你也可以将将其添加到 `bashrc` 文件中：

```js
alias node="env NODE_NO_READLINE=1 rlwrap node"
```

## 环境变量

以下环境变量可以用于调整内建的 REPL（通过 `node` 或 `node -i` 启动）：

- `NODE_REPL_HISTORY`，该变量用于指定 REPL 历史记录的存放位置，默认为用户主页目录的 `.node_repl_history`。
- `NODE_REPL_HISTORY_SIZE`，默认值为 `1000`，用于限制历史记录的数量，必须为正数
- `NODE_REPL_MODE`，可选值为 sloppy/strict/magin，默认值为 magic。默认在严格模式下运行只适用于严格模式的语句。

## 永久化历史记录

默认情况下，REPL 为将 REPL 中的回话激励保存在 `.node_repl_history` 文件中。如果想禁用该特性，可以通过修改环境变量 `NODE_REPL_HISTORY=""` 实现。

#### NODE_REPL_HISTORY_FILE

<div class="s s0"></div>

## REPL 特性

在 REPL 中，通过 `Control + D` 可推出 REPL 环境。REPL 支持输入多行语句以及本地或全局变量的 tab 补全。

核心模块会被按需加载到 REPL 中，比如使用 `fs` 模块时，系统会自动通过 `require()` 将 fs 模块加载为 `global.fs`。

特殊变量 `_` 表示上一次执行的结果：

```js
> [ 'a', 'b', 'c' ]
[ 'a', 'b', 'c' ]
> _.length
3
> _ += 1
4
```

REPL 支持对任意全局变量的访问。开发者可以通过将变量绑定给 `context` 对象显式暴漏给 REPL：

```js
// repl_test.js
const repl = require('repl');
var msg = 'message';

repl.start('> ').context.m = msg;
```

在 `context` 对象中的属性，可以在 REPL 中以本地变量的形式使用：

```js
$ node repl_test.js
> m
'message'
```

下面是一些特殊的 REPL 命令：

- `.break`，当输入多行语句时，可能会后悔，此时使用 `.break` 可以重新开始
- `.clear`，该命令用于将 `context` 对象重置为空对象，并清空多行表达式
- `.exit`，该命令用于关闭 I/O stream
- `.help`，该命令用于显示 REPL 中的特殊命令 
- `.save`，该命令用于将当前 REPL 会话保存为文件
- `.load`，该命令用于将会话文件加载到当前 REPL 中

下面是一些特殊的 REPL 快捷键：

- `Control-C`，类似 .break，用于关闭当前 REPL 会话
- `Control-D`，类似 .exit 命令
- `tab`，显示全局和本地变量

#### 自定义对象

REPL 模块内部使用了 `util.inspect()` 打印值。不过，如果对象下挂载 `inspect()` 方法，那么 `util.inspect()` 就会调用对象的 `inspect()` 方法。

在下面的代码中，假设你在对象上定义了一个 `inspect()` 方法：

```js
> var obj = {foo: 'this will not show up in the inspect() output'};
undefined
> obj.inspect = () => {
...   return {bar: 'baz'};
... };
[Function]
```

然后在 REPL 中查看 `obj`，就会发现 REPL 自动调用了 `inspect()` 方法：

```js
> obj
{bar: 'baz'}
```

## Class: REPLServer

该类继承自 `Readline Interface`，并拥有一下事件：

#### 事件：'exit'

- `function () {}`

当用户退出 REPL 时就会触发该事件，常见的退出方式包括输入 `.exit` 命令退出 REPL，按两次 Control + C 发送 `SIGINT` 信号，或者按 Control + D 向 `input` stream 发送 `end` 信号。

```js
replServer.on('exit', () => {
  console.log('Got "exit" event from repl!');
  process.exit();
});
```

#### 事件：'reset'

- `function (context) {}`

当 REPL 的 context 被重置时触发该事件。在 REPL 中输入 `.clear` 命令就会触发该事件。如果开发者使用 `{ useGlobal: true }` 启动 REPL，那么就不会触发该事件。

```js
// Extend the initial repl context.
var replServer = repl.start({ options ... });
someExtension.extend(r.context);

// When a new context is created extend it as well.
replServer.on('reset', (context) => {
  console.log('repl has a new context');
  someExtension.extend(context);
});
```

#### replServer.defineComman(keyword, cmd)

- `keyword`，字符串
- `cmd`，对象或函数

该方法用于定义 REPL 可用的命令，此类命令以 `.` 开头。`cmd` 对象拥有以下属性：

- `help`，在 REPL 中输入 `.help` 时显示的辅助信息，可选属性
- `action`，函数，接收一个字符串参数。在 REPL 调用自定义命令时，该函数绑定到 REPLServer 的实例上，必选属性

如果传给 `cmd` 对象的是一个函数而不是对象，那么该函数会被视为是 `action`。

```js
// repl_test.js
const repl = require('repl');

var replServer = repl.start();
replServer.defineCommand('sayhello', {
  help: 'Say hello',
  action: function(name) {
    this.write(`Hello, ${name}!\n`);
    this.displayPrompt();
  }
});
```

在 REPL 中执行的效果：

```js
> .sayhello Node.js User
Hello, Node.js User!
```

#### replServer.displayPrompt([preserveCursor])

- `preserveCursor`，布尔值

该方法与 `readline.prompt` 类似，不同之处在于使用省略号添加了缩进。`preserveCursor` 参数将会被传递给 `readline.prompt`。该方法类似于 `defineCommand` 命令。在 REPL 内部使用该方法渲染每一行的提示符。

## repl.start([options])

该方法创建和返回一个 `REPLServer` 实例，该实例继承自 `Readline Interface`，其中 `options` 参数是一个对象，包含以下属性：

- `prompt`，所有 I/O 的提示符和 stream，默认值为 `>`
- `input`，监听的可读 stream，默认值为 `process.stdin`
- `output`，写入 readline 数据的可写 stream，默认值为 `process.stdout`
- `terminal`，如果需要 `input` 和 `output` stream 的行为类似于 TTY 并使用 ANSI/VT1000 编码，则设置该属性为 true。默认在 `output` stream 实例化时执行 `isTTY` 检查
- `eval`，函数，用于计算每一行的语句。默认是 `eval()` 的异步封装。
- `useColors`，布尔值，指定是否允许 `writer` 函数输出颜色。如果设置了其他的 `writer` 函数，则该属性无效。默认值为 REPL 的 `terminal`。
- `useGlobal`，如果值为 `true`，REPL 会使用 `global` 对象而不是独立的 `context` 对象执行脚本，默认值为 false
- `ignoreUndefined`，如果只为 `true`，则 REPL 不会输出 `undefined` 值，默认值为 `false`
- `writer`，用于格式化输出结果的函数，默认值为 `util.inspect()`
- `replMode`，该属性决定以何种模式运行所有的命令，包括严格模式、默认模式和混合模式（`magic`），可选值包括：
    - `repl.REPL_MODE_SLOPPY`，sloppy 模式
    - `repl.REPL_MODE_STRICT`，严格模式
    - `repl.REPL_MODE_MAGIC`，首先会尝试默认模式，如果失败则使用严格模式

开发者也可以使用自定义的 `eval` 函数，该函数需要包含一下特性：

```js
function eval(cmd, context, filename, callback) {
  callback(null, result);
}
```

对于 tab 补全，系统会使用 `.scope` 命令调用 `eval`，然后返回一个包含作用域名的数组用于自动补全。

同一个 Node.js 实例可以运行多个 REPL，所有的 REPL 共享全局对象，但拥有独立的 I/O。

下面代码演示了在 stdin、Unix socket 和 TCP socket 上启动一个 REPL：

```js
const net = require('net');
const repl = require('repl');
var connections = 0;

repl.start({
  prompt: 'Node.js via stdin> ',
  input: process.stdin,
  output: process.stdout
});

net.createServer((socket) => {
  connections += 1;
  repl.start({
    prompt: 'Node.js via Unix socket> ',
    input: socket,
    output: socket
  }).on('exit', () => {
    socket.end();
  })
}).listen('/tmp/node-repl-sock');

net.createServer((socket) => {
  connections += 1;
  repl.start({
    prompt: 'Node.js via TCP socket> ',
    input: socket,
    output: socket
  }).on('exit', () => {
    socket.end();
  });
}).listen(5001);
```

从命令行运行上段代码将会在 stdin 启动一个 REPL。其他 REPL 可以通过 Unix socket 或 TCP socket 连接到此 REPL。`telnet` 常用于连接 TCP scoket，`socat` 既可以连接 Unix 也可以连接 TCP socket。

从一个基于 socket 的 Unix 服务器启动 REPL 的话，那么可以直接连接到长时间运行的 Node.js 进程，而无需重启该进程。

有关通过 `net.Server` 和 `net.Socket` 实例运行全功能 REPL 的实例，请参考 [https://gist.github.com/2209310](https://gist.github.com/2209310)。

有关通过 `curl(1)` 运行 REPL 的实例，请参考 [https://gist.github.com/2053342](https://gist.github.com/2053342)。


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