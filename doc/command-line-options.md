## 控制台

<div class="s s2"></div>

`console` 模块提供了一些简单的调试输出函数，与浏览器提供的 consle 方法类似。

该模块主要对外开放了两个组件：

- `Console` 类，该类提供了 `console.log()` / `console.error` 和 `console.warn()` 方法，常用于向 Node.js stream 输出信息。
- 全局的 `console` 实例，用于向 stdout 和 stderr 输出信息。因为该对象是全局的，所以无需使用 `require('console')` 调用。

```js
console.log('hello world');
// 输出结果: hello world, to stdout
console.log('hello %s', 'world');
// 输出结果: hello world, to stdout
console.error(new Error('Whoops, something bad happened'));
// 输出结果: [Error: Whoops, something bad happened], to stderr

const name = 'Will Robinson';
console.warn(`Danger ${name}! Danger!`);
// 输出结果: Danger Will Robinson! Danger!, to stderr
```

下面是使用 `Console` 类的示例：

```js
const out = getStreamSomehow();
const err = getStreamSomehow();
const myConsole = new console.Console(out, err);

myConsole.log('hello world');
// 输出结果: hello world, to out
myConsole.log('hello %s', 'world');
// 输出结果: hello world, to out
myConsole.error(new Error('Whoops, something bad happened'));
// 输出结果: [Error: Whoops, something bad happened], to err

const name = 'Will Robinson';
myConsole.warn(`Danger ${name}! Danger!`);
// 输出结果: Danger Will Robinson! Danger!, to err
```

虽然 `Console` 类的 API 与浏览器中的 `console` 相似，但 Node.js 开发 `Console` 类的目的并不是复制浏览器的 `console` 功能。

## 异步 VS 同步 console

`console` 函数默认情况下都是异步执行的，除非要将信息输出到文件中。硬盘已经很快了，操作系统通常使用高速缓存进行读写，但仍然不可避免发生进程阻塞的情况。

## Class: Console

通过 `require('console').Console` 或 `console.Console` 的方法可以访问 `Console` 类，它常用于创建简单的可配置输出流的日志记录器：

```js
const Console = require('console').Console;
const Console = console.Console;
```

#### new Console(stdout[, stderr])

该方法通过传入一个或两个可写的 stream 实例来创建一个新的 `Console` 实例。`stdout` 是一个可写的 stream 实例，用于打印日志或输出信息。`stderr` 常用于输出提示和错误。如果没有传入 `stderr`，将使用 `stdout` 输出提示和错误：

```js
const output = fs.createWriteStream('./stdout.log');
const errorOutput = fs.createWriteStream('./stderr.log');
// custom simple logger
const logger = new Console(output, errorOutput);
// use it like console
var count = 5;
logger.log('count: %d', count);
// in stdout.log: count 5
```

全局的 `console` 是一个特殊的 `Console` 实例，它输出的信息会发送给 `process.stdout` 和 `process.stderr`，等同于：

```js
new Console(process.stdout, process.stderr);
```

#### console.assert(value[, message][, ...])

该方法是一个简单的断言测试，常用于测试 `value` 是否为真值。如果不是真值，则抛出 `AssertionError`。如果传入了可选参数 `message`，则使用 `util.format()` 格式优化该参数，输出格式化后的字符串：

```js
console.assert(true, 'does nothing');
// OK
console.assert(false, 'Whoops %s', 'didn\'t work');
// AssertionError: Whoops didn't work
```

#### console.dir(obj[, options])

该方法使用 `util.inspect()` 处理 `obj` 参数，并将字符串结果输送给 `stdout`。该方法忽略 `obj` 参数中自定义的 `inspect()` 方法。可选参数对象 `options` 可以用来修改字符串某些部分：

- `showHidden`，如果为 true，则显示对象的不可枚举和 symbol 属性，默认值为 false
- `depth`，指定 `inspect()` 方法格式化 `object` 时的深度。这对于格式化复杂对象很有用。默认值为 2，如果不限制深度，可以设置为 `null
- `colors`，如果值为 true，则使用 ANSI 颜色码对输出信息加以美化。默认值为 false

#### console.error([data][, ...])

该方法用于将消息传送给 `stderr`，接受多个参数，第一个参数可以包含占位符，其他参数可以用来替换占位符，实际上所有的参数都传递给了 `util.format()` 函数：

```js
const code = 5;
console.error('error #%d', code);
// 输出结果: error #5, to stderr
console.error('error', code);
// 输出结果: error 5, to stderr
```

#### console.info([data][, ...])

该方法等同于 `console.log()`。

#### console.log([data][, ...])

该方法用于将信息传送给 `stdout`，接受多个参数，第一个参数可以包含占位符，其他参数可以用来替换占位符，实际上所有的参数都传递给了 `util.format()` 函数：

```js
var count = 5;
console.log('count: %d', count);
  // Prints: count: 5, to stdout
console.log('count: ', count);
  // Prints: count: 5, to stdout
```

#### console.time(label)

该方法用于启动一个计时器，计算某个操作的执行时间，每个计时器都有特定的 `label` 标签。向 `console.timeEnd()` 中传入同一个 `label` 可以停止计时器，并输出有关执行时间的信息到 stdout 中。计时器的精度是亚毫秒级的。

#### console.timeEnd(label)

该方法用于停止计时器，并输出有关执行时间的信息到 stdout：

```js
console.time('100-elements');
for (var i = 0; i < 100; i++) {
  ;
}
console.timeEnd('100-elements');
// prints 100-elements: 225.438ms
```

#### console.trace(message[, ...])

传送有关堆栈跟踪的信息到 stderr 中，首先是一个字符串 `Trace:`，接下来是使用 `util.format()` 格式化的堆栈跟踪信息：

```js
console.trace('Show me');
// Prints: (stack trace will vary based on where trace is called)
//  Trace: Show me
//    at repl:2:9
//    at REPLServer.defaultEval (repl.js:248:27)
//    at bound (domain.js:287:14)
//    at REPLServer.runBound [as eval] (domain.js:300:12)
//    at REPLServer.<anonymous> (repl.js:412:12)
//    at emitOne (events.js:82:20)
//    at REPLServer.emit (events.js:169:7)
//    at REPLServer.Interface._onLine (readline.js:210:10)
//    at REPLServer.Interface._line (readline.js:549:8)
//    at REPLServer.Interface._ttyWrite (readline.js:826:14)
```

#### console.warn([data][, ...])

该方法等同于 `console.error()`。

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