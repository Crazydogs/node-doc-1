## Errors

基于 Node.js 开发的应用通常会遇到以下四类错误：

- 标准的 JavaScript 错误：
    - `[EvalError][]`：调用 `eval()` 出现错误时抛出该错误
    - `SyntaxError`：代码不符合 JavaScript 语法规范时抛出该错误
    - `[RangeError][]`：数组越界时抛出该错误
    - `ReferenceError`：引用未定义的变量时抛出该错误
    - `TypeError`：参数类型错误时抛出该错误
    - `[URIError][]`：误用全局的 URI 处理函数时抛出该错误
- 由操作系统底层触发的系统错误，比如尝试打开不存在的文件、尝试通过已关闭的 socket 发送数据等
- 用户自定义的错误，通常在应用程序运行过程中触发
- 断言错误，当 Node.js 检测到逻辑错误时会触发该错误。通常来说，此类错误由 `assert` 模块抛出

Node.js 抛出的所有 JavaScript 和系统错误都是 `Error` 类的实例，每个实例都至少有一个引用该类的属性。

## 错误的传播和拦截

Node.js 提供了多种传播和处理错误的机制，具体的而传播和处理机制需要根据错误类型和调用的 API 类型来确定。

所有的 JavaScript 错误都会被视为异常，使用 `tre / catch` 命令可以立即将其抛出：

```js
// Throws with a ReferenceError because z is undefined
try {
  const m = 1;
  const n = m + z;
} catch (err) {
  // Handle the error here.
}
```

必须配合 `try / catch` 使用 JavaScript 的 `throw` 机制，否则将会立即中断 Node.js 的进程。

除少数情况之外，Node.js 使用同步版本的 API 处理异常。

异步版本的 API 有多种方式记录错误：

- 大多数的异步方法都接收回调函数，回调函数的第一个参数就是抛出的 `Error`。如果第一个参数的值为不是 `null` 且是 Error 的实例，则表示抛出了错误，需要开发者进行处理：

```js
const fs = require('fs');
fs.readFile('a file that does not exist', (err, data) => {
  if (err) {
    console.error('There was an error reading the file!', err);
    return;
  }
  // Otherwise handle the data
});
```

- 调用异步方法的 `EventEmitter` 对象，可以通过监听 `error` 时间捕获错误：

```js
const net = require('net');
const connection = net.connect('localhost');

// Adding an 'error' event handler to a stream:
connection.on('error', (err) => {
  // If the connection is reset by the server, or if it can't
  // connect at all, or on any sort of error encountered by
  // the connection, the error will be sent here.
  console.error(err);
});

connection.pipe(process.stdout);
```

- Node.js 中少数的异步方法仍然采用了 `throw` 机制抛出错误，对此需要使用 `try / catch` 捕获错误。对于此类异步方法，目前尚没有完整的统计列表，请参考本文档中的对 API 的详细说明，采用合适的机制处理错误。

基于 stream 和基于 event emitter 的 API 通常都是通过监听 `error` 事件处理异常的，这些 API 大都封装了一系列的异步操作。

对于所有的 `EventEmitter` 对象，如果未监听 `error` 事件，除非恰当地使用了 `domain` 模块或者为 `process.on('uncaughtException')` 事件设置了处理函数，否则就会上抛错误，导致 Node.js 进程崩溃并报告该错误：

```js
const EventEmitter = require('events');
const ee = new EventEmitter();

setImmediate(() => {
  // This will crash the process because no 'error' event
  // handler has been added.
  ee.emit('error', new Error('This will crash'));
});
```

不能使用 `try / catch` 来捕获此类错误，这是因为捕获行为发生在触发该错误的代码之后。

开发者有必要仔细阅读本文档，了解每一个方法的错误传播方式。

#### Node.js 风格的回调函数

Node.js 中大多数的异步方法都遵循一个被称为 `Node.js 式回调函数` 的通用模式。在这一模式中，回调函数将传给异步方法作为一个参数。无论异步方法是否成功执行，最终都会执行回调函数，且回调函数的第一个参数就是一个 Error 对象。如果没有错误，则回调函数的第一个参数的值为 `null`：

```js
const fs = require('fs');

function nodeStyleCallback(err, data) {
 if (err) {
   console.error('There was an error', err);
   return;
 }
 console.log(data);
}

fs.readFile('/some/file/that/does-not-exist', nodeStyleCallback);
fs.readFile('/some/file/that/does-exist', nodeStyleCallback)
```

不能使用 `try / catch` 来捕获异步方法的错误。对于初学者常见的错误就是在回调函数中使用 `throw` 的方式：

```js
// THIS WILL NOT WORK:
const fs = require('fs');

try {
  fs.readFile('/some/file/that/does-not-exist', (err, data) => {
    // mistaken assumption: throwing here...
    if (err) {
      throw err;
    }
  });
} catch(err) {
  // This will not catch the throw!
  console.log(err);
}
```

上面的错误捕获方式并不会生效，这是因为 `fs.readFile()` 的回调函数是异步调用的。当回调函数被调用时，`try / catch` 代码块已经退出了执行期。回调函数中抛出的错误往往会让 Node.js 进程崩溃。如果恰当地使用了 `domain` 模块或者为 `process.on('uncaughtException')` 事件设置了处理函数，则可以避免此类崩溃。

## Class: Error

JavaScript 的 `Error` 对象并不会显示错误发生的具体环境，但它会显示 `Error` 对象实例化时的堆栈信息，并提供有关的描述信息

在 Node.js 的程序中，无论是系统错误还是 JavaScript 错误，统统是 `Error` 类的实例。

#### new Error(message)

该方法用于创建 `Error` 对象，并根据 `message` 参数配置 `error.message` 属性。如果 `message` 参数是一个对象，则调用 `message.toString()` 转换为字符串。`error.stack` 属性用于表示 `new Error()` 在代码中被调用的信息。对堆栈的跟踪依赖于 [V8 的堆栈跟踪 API](https://github.com/v8/v8/wiki/Stack-Trace-API)。堆栈跟踪只会提供开始执行同步代码的信息或者 `Error.stackTraceLimit` 属性指定范围内的调用帧信息。

#### Error.captureStackTrace(targetObject[, constructorOpt])

该方法用于在 `targetObject` 上创建一个 `.stack` 属性，该属性的值是一个字符串，表示 `Error.captureStackTrace()` 方法被调用的位置信息。

```js
const myObject = {};
Error.captureStackTrace(myObject);
myObject.stack  // similar to `new Error().stack`
```

`ErrorType: message` 后的第一行堆栈跟踪信息是 `targetObject.toString()` 的返回值。

可选参数 `constructorOpt` 是一个函数。如果指定了该函数，则从堆栈跟踪开始，`constructorOpt` 之上或之内的调用帧都会被忽略。

`constructorOpt` 参数常用于对用户隐藏错误发生的细节，举例如下:

```js
function MyError() {
  Error.captureStackTrace(this, MyError);
}

// Without passing MyError to captureStackTrace, the MyError
// frame would should up in the .stack property. by passing
// the constructor, we omit that frame and all frames above it.
new MyError().stack
```

#### Error.stackTraceLimit

`Error.stackTraceLimit` 属性指定了堆栈跟踪器收集堆栈信息的最大容量。

虽然该属性的默认值为 10，但是开发者可以修改为任何有效的 JavaScript 数值。该属性值修改后立即生效。

如果将该属性的值为非数值或负值，则堆栈跟踪器不捕获任何调用帧的信息。

#### error.message

该属性返回一个字符串，即 `new Error(message)` 中的 `message` 参数。传递给构造函数的 `message` 参数将会出现在 `Error` 的堆栈信息的首行。`Error` 对象创建之后再修改该属性也许并不能修改堆栈信息：

```js
const err = new Error('The message');
console.log(err.message);
// Prints: The message
```

#### error.stack

该属性返回一个字符串，表示 Error 对象初始化方面的信息：

```js
Error: Things keep happening!
   at /home/gbusey/file.js:525:2
   at Frobnicator.refrobulate (/home/gbusey/business-logic.js:424:21)
   at Actor.<anonymous> (/home/gbusey/actors.js:400:8)
   at increaseSynergy (/home/gbusey/actors.js:701:6)
```

第一行是固定的 `<error class name>: <error message>`，其后是一系列的调用栈信息（每一行都以 `at` 开头）。每一帧都描述了代码中的一个错误抛出点。V8 会尝试显示每一个函数的函数名，但有时候并无法找到合适的名字。如果 V8 无法确定函数名，那么就会只显示位置信息；如果可以确定函数名，则会同时显示函数名和位置信息，此时位置信息置于尾部的括号中。

有一点非常值得注意，那就是只有 JavaScript 函数会产生调用帧。举例来说，如果将可执行的异步方法传给一个名为 `cheetahify` 的 C++ 插件，则发生错误时，并不会显示该插件有关的堆栈跟踪信息：

```js
const cheetahify = require('./native-binding.node');

function makeFaster() {
  // cheetahify *synchronously* calls speedy.
  cheetahify(function speedy() {
    throw new Error('oh no!');
  });
}

makeFaster(); // will throw:
// /home/gbusey/file.js:6
//     throw new Error('oh no!');
//           ^
// Error: oh no!
//     at speedy (/home/gbusey/file.js:6:11)
//     at makeFaster (/home/gbusey/file.js:5:3)
//     at Object.<anonymous> (/home/gbusey/file.js:10:1)
//     at Module._compile (module.js:456:26)
//     at Object.Module._extensions..js (module.js:474:10)
//     at Module.load (module.js:356:32)
//     at Function.Module._load (module.js:312:12)
//     at Function.Module.runMain (module.js:497:10)
//     at startup (node.js:119:16)
//     at node.js:906:3
```

位置信息为以下类型之一：

- `native`，V8 内部的调用帧
- `plain-filename.js:line:column`，Node.js 内部的调用帧
- `/absolute/path/to/file.js:line:column`，基于 Node.js 的程序或依赖所产生的调用帧

当 `error.stack` 可访问时，该字符串的生成速度较慢。

堆栈跟踪的调用帧数量在小于 `Error.stackTraceLimit`。

## Class: RangeError

该类是 `Error` 类的子类，用于表示函数接收到了指定范围之外的参数，该范围可能是数值范围也可能是一个可选列表：

```js
require('net').connect(-1);
// throws RangeError, port should be > 0 && < 65536
```

参数检验完成后，Node.js 会立即生成和抛出 `RangeError` 实例。

## Class: ReferenceError

该类是 `Error` 类的一个子类，用于表示访问的变量不存在。此类错误往往是由代码中的拼写错误或其他编写问题引起的。

虽然客户端代码可以产生或抛出此类错误，但实际上，此类代码是由 V8 产生和抛出的：

```js
doesNotExist;
// throws ReferenceError, doesNotExist is not a variable in this program.
```

`ReferenceError` 实例都拥有一个 `error.arguments` 属性，该属性的值为一个数组，且数组只有一个字符串，该字符串用于表示当前变量未定义：

```js
const assert = require('assert');
try {
  doesNotExist;
} catch(err) {
  assert(err.arguments[0], 'doesNotExist');
}
```

除非应用程序动态生成和执行代码，否则代码中或依赖环境中的 `ReferenceError` 实例都应该被视为一个 Bug.

## Class: SyntaxError

该类是 `Error` 类的子类，用于表示当前程序代码不符合 JavaScript 语法规范。此类错误通常只会出现在代码评估阶段。代码评估通常指 `eval`、`Function`、`require` 或 `vm` 的执行结果。此类错误通常会中断程序的执行：

```js
try {
  require('vm').runInThisContext('binary ! isNotOk');
} catch(err) {
  // err will be a SyntaxError
}
```

`SyntaxError` 实例无法在触发该错误的上下文中捕获，只能在其他上下文中捕获。

## Class: TypeError

该类是 `Error` 类的子类，用于表示传入的参数不符合函数要求。举例来说，当某个方法只接受字符串参数时，如果传入的参数是一个函数，那么就会抛出该错误：

```js
require('url').parse(function() { });
// throws TypeError, since it expected a string
```

参数检验完成后，Node.js 会立即生成和抛出 `TypeError` 实例。

## 异常 VS 错误

JavaScript 中的异常都是一个值，表示无效的操作或 `throw` 的对象。这些值要么是 `Error` 的实例，要么就是继承自 `Error`。所有由 Node.js 或 JavaScript 运行环境抛出的异常都是 Error 的实例。

有一些异常无法在 JavaScript 层面上捕获。此类异常通常会让 Node.js 进程崩溃，比如在 C++ 层调用 `abort()` 或 `assert()` 都会产生这种异常。

## 系统错误

在程序的运行环境中抛出异常时，就会产生系统错误。通常来说，当应用程序违反操作系统的约束条件时就会出触发此类错误，比如试图读取一个不存在的文件或者当前操作的权限过低等等。

系统错误通常在系统调用时产生。在 Unix 系统下，通过执行 `man 2 intro` 或 `man 3 errno` 可以获取完整的错误码列表及其简介。

在 Node.js 中，系统错误丰富了 `Error` 对象，该对象中有专门属性用于描述系统错误。

#### Class: System Error

**error.code && error.errno**

该属性返回一个字符串，表示错误码，通常以大写字母 `E` 开头，详见 `man 2 intro` 命令的解释。

`error.code` 和 `error.errno` 的功能相同，返回的值也相同。

**error.syscall**

该方法返回一个字符串，用于表示失败的系统调用（syscall）。

#### 常见的系统错误

下面的列表并不完整，只是一些开发 Node.js 程序时常见的系统错误：

- `EACCES` (Permission denied): An attempt was made to access a file in a way forbidden by its file access permissions.

- `EADDRINUSE` (Address already in use): An attempt to bind a server (net, http, or https) to a local address failed due to another server on the local system already occupying that address.

- `ECONNREFUSED` (Connection refused): No connection could be made because the target machine actively refused it. This usually results from trying to connect to a service that is inactive on the foreign host.

- `ECONNRESET` (Connection reset by peer): A connection was forcibly closed by a peer. This normally results from a loss of the connection on the remote socket due to a timeout or reboot. Commonly encountered via the http and net modules.

- `EEXIST` (File exists): An existing file was the target of an operation that required that the target not exist.

- `EISDIR` (Is a directory): An operation expected a file, but the given pathname was a directory.

- `EMFILE` (Too many open files in system): Maximum number of file descriptors allowable on the system has been reached, and requests for another descriptor cannot be fulfilled until at least one has been closed. This is encountered when opening many files at once in parallel, especially on systems (in particular, OS X) where there is a low file descriptor limit for processes. To remedy a low limit, run ulimit -n 2048 in the same shell that will run the Node.js process.

- `ENOENT` (No such file or directory): Commonly raised by fs operations to indicate that a component of the specified pathname does not exist -- no entity (file or directory) could be found by the given path.

- `ENOTDIR` (Not a directory): A component of the given pathname existed, but was not a directory as expected. Commonly raised by fs.readdir.

- `ENOTEMPTY` (Directory not empty): A directory with entries was the target of an operation that requires an empty directory -- usually fs.unlink.

- `EPERM` (Operation not permitted): An attempt was made to perform an operation that requires elevated privileges.

- `EPIPE` (Broken pipe): A write on a pipe, socket, or FIFO for which there is no process to read the data. Commonly encountered at the net and http layers, indicative that the remote side of the stream being written to has been closed.

- `ETIMEDOUT` (Operation timed out): A connect or send request failed because the connected party did not properly respond after a period of time. Usually encountered by http or net -- often a sign that a socket.end() was not properly called.

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