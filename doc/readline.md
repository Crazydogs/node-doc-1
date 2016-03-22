## Readline

<div class="s s2"></div>

通过 `require('readline')` 可以加载该模块。Readline 模块提供了逐行读取 stream （比如 `process.stdin`）的能力。

注意，一旦调用该模块，Node.js 程序将不会自动关闭，需要开发者显式关闭接口。下面的代码演示了如何退出程序：

```js
const readline = require('readline');

const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout
});

rl.question('What do you think of Node.js? ', (answer) => {
  // TODO: Log the answer in a database
  console.log('Thank you for your valuable feedback:', answer);

  rl.close();
});
```

## Class: Interface

该类是一个包含输入输出 stream 的 readline 接口。

#### rl.close()

该方法用于关闭 `Interface` 实例，取消对 `input` 和 `output` stream 的控制。调用该方法会触发 `close` 事件。

#### rl.pause()

该方法用于暂停执行 readline 中的 `input` stream，该 stream 稍后可以回复状态继续执行。

注意，该方法不会立即暂停 stream 的操作。调用该方法后将会触发多个事件，包括 `line`。

#### rl.prompt([preserveCursor])

该方法为用户配置好输入环境，并将当前的 `setPrompt` 置于新行，指定用户输入的位置。如果 `preserveCursor` 的值为 `true`，则不允许光标位置被重置为 `0`。

如果 `input` stream 已经暂停，也可以使用 `createInterface` 重置。

如果调用 `createInterface` 时，`output` 的值为 `null` 或 `undefined`，那么就不会写入提示符。

#### rl.question(query, callback)

该方法向用户显示 `query`，并在用户响应后执行 `callback`。

如果 `input` stream 已经暂停，也可以使用 `createInterface` 重置。

如果调用 `createInterface` 时，`output` 的值为 `null` 或 `undefined`，那么就不会写入提示符。

```js
rl.question('What is your favorite food?', (answer) => {
  console.log(`Oh, so your favorite food is ${answer}`);
});
```

#### rl.resume()

该方法用于恢复 `input` stream 的状态。

#### rl.setPrompt(prompt)

该方法用于设置提示符，举例来说，当你在命令行中运行 Node.js 时，你会发现一个 `>` 符号，这就是 Node.js 的提示符。

#### rl.write(data[, key])

该方法用于将 `data` 写入 `output` stream，除非调用 `createInterface` 时，`output` 的值为 `null` 或 `undefined`。`key` 是一个对象，包含了一系列的键名。只有终端是 TTY 时，该方法才可用。

如果 `input` stream 已经暂停，也可以该方法重置。

```js
rl.write('Delete me!');
// Simulate ctrl+u to delete the line written previously
rl.write(null, {ctrl: true, name: 'u'});
```

#### 事件：'close'

- `function () {}`

当调用 `close()` 时会触发该事件。

当 `input` stream 接收到 `end` 事件时也会触发该事件。该事件触发之后，即可将 `Interface` 的实例视为已结束，比如 `input` stream 收到 `^D` 时，就会被认为是 `EOT`。

当 `input` sream 接收到 `^C`（SIGINT）时，如果没有 `SIGINT` 事件监听器，那么也会触发该事件。

#### 事件：'line'

- `function (line) {}`

当 `input` stream 接收到行结束符（`\n`、`\r`、或 `\r\n`）时，就会触发该事件。通常来说，当用户敲下回车键或者 return 键时，就会触发该事件。这对监听用户的输入来说是一个非常有用的钩子。

```js
rl.on('line', (cmd) => {
  console.log(`You just typed: ${cmd}`);
});
```

#### 事件：'pause'

- `function () {}`

当 `input` stream 暂停时触发该事件。此外，当 `input` stream 接收到 `SIGCONT` 事件时也会触发该事件。

```js
rl.on('pause', () => {
  console.log('Readline paused.');
});
```

#### 事件：'resume'

- `function () {}`

当 `input` stream 从暂停中回复过来时触发该事件。

```js
rl.on('resume', () => {
  console.log('Readline resumed.');
});
``` 

#### 事件：'SIGCONT'

- `function () {}`

该事件在 Windows 上不会触发。

当 `input` stream 向后台传送 `^Z`（`SIGTSTP`）时就会触发该事件，然后继续执行 `fs(1)`。该事件只有程序切换到后台前 stream 没有暂停才会触发。

```js
rl.on('SIGCONT', () => {
  // `prompt` will automatically resume the stream
  rl.prompt();
});
```

#### 事件：'SIGINT'

- `function () {}`

当 `input` stream 接收到 `^C`（`SIGINT`）时就会触发该事件。如果接收到 `SIGINT` 却没有 `SIGINT` 事件监听器，那么就会触发 `pause` 事件。

```js
rl.on('SIGINT', () => {
  rl.question('Are you sure you want to exit?', (answer) => {
    if (answer.match(/^y(es)?$/i)) rl.pause();
  });
});
```

#### 事件：'SIGTSTP'

- `function () {}`

该事件在 Windows 上不会触发。

当 `input` stream 向后台传送 `^Z`（`SIGTSTP`）时就会触发该事件。如果接收到 `SIGINT` 却没有 `SIGINT` 事件监听器，那么就会将程序切换到后台。

当程序通过 `fg` 回复时，会触发 `pause` 和 `SIGCONT` 事件。开发者可以使用任意一个事件恢复 stream 的状态。

如果程序切换到后台前 stream 暂停了，那么就不会触发 `pause` 和 `SIGCONT` 事件。

```js
rl.on('SIGTSTP', () => {
  // This will override SIGTSTP and prevent the program from going to the
  // background.
  console.log('Caught SIGTSTP.');
});
```

## 实例：Tiny CLI

下面代码演示了如何使用上述方法创建一个命令行接口：

```js
const readline = require('readline');
const rl = readline.createInterface(process.stdin, process.stdout);

rl.setPrompt('OHAI> ');
rl.prompt();

rl.on('line', (line) => {
  switch(line.trim()) {
    case 'hello':
      console.log('world!');
      break;
    default:
      console.log('Say what? I might have heard `' + line.trim() + '`');
      break;
  }
  rl.prompt();
}).on('close', () => {
  console.log('Have a great day!');
  process.exit(0);
});
```

## 实例：逐行读取文件 stream

一个常见的操作就是使用 readline 的 `input` 选项读取文件 stream。下面带那么演示了如何逐行解析文件：

```js
const readline = require('readline');
const fs = require('fs');

const rl = readline.createInterface({
  input: fs.createReadStream('sample.txt')
});

rl.on('line', (line) => {
  console.log('Line from file:', line);
});
```

## readline.clearLine(stream, dir)

该方法从指定的方法清除 TTY stream 中的当前行，其中 `dir` 有如下可选值：

- `-1`，从光标向左
- `1`，从光标向右
- `0`，正行

## readline.clearScreenDown(stream)

该方法用于清空屏幕上从光标位置开始的内容。

## readline.createInterface(options)

该方法创建一个 readline `Interface` 实例。可选参数 `options` 包含以下属性：

- `input`，监听的可读 stream，必选属性
- `output`，输入 realine 数据的可写 stream，可选属性
- `completer`，可选函数，用于自动补全
- `terminal`，如果需要 `input` 和 `output` stream 的行为类似于 TTY 并使用 ANSI/VT1000 编码，则设置该属性为 true。默认在 `output` stream 实例化时执行 `isTTY` 检查
- `historySize`，指定保留多少行的历史记录，默认值为 30

`completer` 函数接收用户的输入并返回一个数组，该数组包含两条记录：

1. 复核补全的字符串数组
1. 用于匹配的子串

最终结果类似于：`[[substr1, substr2, ...], originalsubstring]`。

```js
function completer(line) {
  var completions = '.help .error .exit .quit .q'.split(' ')
  var hits = completions.filter((c) => { return c.indexOf(line) == 0 })
  // show all completions if none found
  return [hits.length ? hits : completions, line]
}
```

如果给 `completer` 传入两个参数，那么也可以以异步的方式执行：

```js
function completer(linePartial, callback) {
  callback(null, [['123'], linePartial]);
}
```

`createInterface` 常常和 `process.stdin` 和 `process.stdout` 配合使用，用来接收用户的输入：

```js
const readline = require('readline');
const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout
});
```

一旦创建了 readline 的实例之后，开发者通常会监听 `line` 事件。

如果 readline 实例的 `terminal` 的值为 `true` 并且定义了 `output.columns` 属性，那么 `output` stream 就会保持最佳兼容性。当行大小发生变化时（如果是 TTY，则 process.stdout 会自动处理此类变化），会在 `output` stream 上触发 `resize` 事件。

## readline.cursorTo(stream, x, y)

在给定的 TTY stream 中，该方法用于将光标移动到特定位置。

## readline.moveCursor(stream, dx, dy)

在给定的 TTY 中，该方法相对于光标的当前位置移动光标。

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