## process

`process` 是一个全局对象，也是 `EventEmitter` 的实例。

## 事件：'beforeExit'

当 Node.js 清空事件循环机制且没有其他调度任务时就会触发该事件。当没有调度任务时，Node.js 就会退出，但是 `beforeExit` 的事件监听器可以以异步的形式执行，从而保持 Node.js 持续运行。

`beforeExit` 事件并不是进程终端的条件，process.exit() 或者是未捕获的异常才是。`beforeExit` 事件不应该被当做 `exit` 事件，除非开发者需要大量的调度工作。

## 事件：'exit'

当进程将要退出时触发该事件。此时将无法停止事件循环的执行，一旦所有的 `exit` 事件监听器都执行完毕，那么进程就会退出，因此，开发者只应该使用同步执行的监听器。这是一个检查模块状态（比如单元测试）的好时机。回调函数接收一个参数，即进程的退出码。

该事件只有通过 `process.exit()` 显式结束 Node.js 或事件循环的内存耗尽时才会触发。

```js
process.on('exit', (code) => {
  // do *NOT* do this
  setTimeout(() => {
    console.log('This will not run');
  }, 0);
  console.log('About to exit with code:', code);
});
```

## 事件：'message'

- `message`，对象，解析后的 JSON 对象或原始值
- `sendHandle`，Handle 对象，net.Socket 或 net.Server 对象，或 undefined

子进程对象可以通过监听 `message` 事件获取 `ChildProcess.send()` 发送的消息。

## 事件：'rejectionHandled'

当 Promise 被驳回且在事件循环开始后绑定了错误处理函数时，就会触发该事件。该事件包含以下参数：

- `p`，经 `unhandleRejection` 事件处理后绑定驳回函数的 Promise。

Promise 调用链中无法绝对肯定驳回会被处理。由于 Promise 天生就是异步的，所以驳回可以在未来的某个时间被处理，这个时间可能远大于事件循环机制将其移交给 `unhandledRejection` 事件的时间。

另一种处理该问题的方式是这样的，在同步执行的代码中总有一个持续增长的未处理异常列表，但在 Promise 中则有一个可伸缩的未处理异常列表。对于同步执行的代码，`uncaughtException` 事件可以通知开发者未处理异常列表在什么时间增大了；对于异步执行的代码，`unhandledRejection` 事件可以通知开发者未处理异常列表在什么时间增大了，而 `rejectionHandled` 事件则可以通知开发者未处理异常列表在什么时间缩小了。

下面代码演示了使用驳回检测方式记录指定时间内所有被驳回的 Promise 原因：

```js
const unhandledRejections = new Map();
process.on('unhandledRejection', (reason, p) => {
  unhandledRejections.set(p, reason);
});
process.on('rejectionHandled', (p) => {
  unhandledRejections.delete(p);
});
```

该记录的内容会随着时间收缩不定，反映出何时发生驳回，何时驳回被处理。开发者可以使用错误日志记录周期性的（适合耗时应用）或在进程退出前（适合脚本）记录错误。

## 事件：'uncaughtException'

当异常传播到事件循环机制时就会触发该事件。默认情况下，Node.js 对于此类异常会直接将其堆栈跟踪信息输出给 stderr 并结束进程，而为 `uncaughtException` 事件添加一个事件监听器则可以覆盖该默认行为。

```js
process.on('uncaughtException', (err) => {
  console.log(`Caught exception: ${err}`);
});

setTimeout(() => {
  console.log('This will still run.');
}, 500);

// Intentionally cause an exception, but don't catch it.
nonexistentFunc();
console.log('This will not run.');
```

#### 合理使用 'uncaughtException'

注意，`uncaughtException` 是一种处理异常的非常规手段，除非万不得已，不要使用它。该事件不能被认为等同于 `On Error Resume Next`。存在未处理的异常就意味着应用程序处于一种未知的状态。如果不能合理地根据异常信息会发应用程序的状态，那么很有可能会触发其他不可预见的问题。

事件处理器内部抛出的异常将不会被捕获，这些异常将会中断 Node.js 的进程并返回一个非零的退出码，最后输出相应的堆栈信息，这一机制有助于避免无限递归。

在系统抛出异常之后尝试恢复的过程很类似于在电脑升级系统的时候拔掉电源，虽然这么做十次中有九次都没有问题，但只要有一次有问题，系统就会挂掉。

正确使用 `uncaughtException` 的方式是在结束进程前使用同步执行的方法清理分配到的资源（文件描述符、句柄等）。在 `uncaughtException` 事件之后执行常规的恢复操作并不安全。

## 事件：'unhandledRejection'

当 `Promise` 被驳回且没有被绑定错误处理函数时，就会触发该事件。当 Promise 遇到异常时都要将其封装为异常 Promise，可以通过 `promise.catch(...)` 处理此类 Promise，驳回信息也会通过 Promise 调用链逐步向上传播。该事件对检测和跟踪未被处理的驳回 Promise 的堆栈信息很有用。该事件接收以下参数：

- `reason`，携带 Promise 被驳回信息的对象，通常为 Error 实例
- `p` 被驳回的 Promise

下面代码演示了使用控制台记录所有未处理的驳回：

```js
process.on('unhandledRejection', (reason, p) => {
    console.log("Unhandled Rejection at: Promise ", p, " reason: ", reason);
    // application specific logging, throwing an error, or other logic here
});
```

下面代码演示了一个会触发 `unhandledRejection` 事件的驳回逻辑：

```js
somePromise.then((res) => {
  return reportToUser(JSON.pasre(res)); // note the typo (`pasre`)
}); 
// no `.catch` or `.then`
```

下面代码是一个触发 `unhandledRejection` 事件的代码模式：

```js
function SomeResource() {
  // Initially set the loaded status to a rejected promise
  this.loaded = Promise.reject(new Error('Resource not yet loaded!'));
}

var resource = new SomeResource();
// no .catch or .then on resource.loaded for at least a turn
```

对于此类事件，如果你不想像开发错误一样追踪驳回信息，那么你就可以使用 `unhandledRejection` 事件。为了实现这一方案，开发者既可以给 `resource.loaded` 绑定 `.catch(() => {})`，避免触发 `unhandledRejection` 事件，也可以使用 `rejectionHandled` 事件。

## 退出码

如果没有异步操作正在执行，那么 Node.js 正常退出的退出码就是 0，否则则为以下情况：

- `1`，存在未被捕获的重大异常。该退出码表示存在未被捕获的异常，且没有被域名或 `uncaughtException` 事件监听器处理
- `2`，该退出码尚处于保留状态，Bash 使用该状态码表示内建的误用信息
- `3`，JavaScript 语法解析错误。该退出码表示 JavaScript 源代码在 Node.js 启动阶段发生了解析错误。这种错误很少见，且只在 Node.js 的开发阶段会被触发。
- `4`，JavaScript 执行错误。该退出码表示 JavaScript 在 Node.js 启动进程时的失败，失败时系统会返回一个函数值。这种错误很少见，且只在 Node.js 的开发阶段会被触发。
- `5`，重大错误。该退出码表示 V8 里出现的不可修复的重大错误。系统通常会向使用 stderr 发送以 `FATAL ERROR` 开头的消息。
- `6`，非函数的异常处理器。系统中存在未捕获的异常，但是内部的异常捕获器却不是函数类型，无法被调用。
- `7`，异常处理器运行失败。系统中存在未捕获的异常，但是内部的异常捕获器运行时抛出了异常。举例来说，`process.on('uncaughtException')` 或 `domain.on('error')` 处理器就会抛出此类错误。
- `8`，该退出码处于保留状态。在早先的 Node.js 中，退出码 8 表示一个未被捕获的异常。
- `9`，无效参数。该退出码表示传入了未知的参数或指定了无效的值。
- `10`，JavaScript 运行失败。该退出码表示 Node.js 的引导进程部分的 JavaScript 代码存在问题。这种错误很少见，且只在 Node.js 的开发阶段会被触发。
- `12`，无效的调试参数。设置了 `--debug` 或 `--debug-brk` 参数，但选择了错误的端口。
- `>128`，信号推出。如果 Node.js 接收到了 `SIGKILL` 或 `SIGHUP` 信号，那么它就会结束进程且退出码为 128 + 信号码。这是一个标准的 Unix 做法，因为退出码为低位 7 字节的整数，信号码高位 7 字节整数，所以它们的值大于 128。

## 信号事件

当继承接收到信号时就会触发此类事件。查看 `sigaction(2)` 可以获得标准的 POSIX 信号名，比如 `SIGINT`、`SIGHUP` 等。

下面代码演示了如何监听 `SIGINT`：

```js
// Start reading from stdin so we don't exit.
process.stdin.resume();

process.on('SIGINT', () => {
  console.log('Got SIGINT.  Press Control-D to exit.');
});
```

在大多数的终端环境中，使用 `Control-C` 组合键是一个发送 `SIGINT` 的简单方式。

- `SIGUSER1`，Node.js 使用该信号开启调试模式。在调试模式时可以设置监听器，且不会中断调试模式。
- `SIGTERM` 和 `SIGINT`，在非 Windows 平台上有默认的处理函数，用于在使用退出码 `128 + signal number` 退出前重置终端模式。如果这些信号中的某一个设置了监听器，那么就会移除上述的默认行为（Node.js 将不再会退出）。
- `SIGPIPE`，该信号默认会被系统忽略，可以设置一个监听器。
- `SIGHUB`，当 Windows 的控制台窗口退出时会生成该信号，其他平台出现类似情况时，也会生成该信号，详见 `signal(7)`。该信号可以设置一个监听器，不过在 Windows 将会无条件地在 10 秒后终端 Node.js，在非 Windows 的平台上，`SIGHUB` 的默认行为是关闭 Node.js，但如果为该信号设置了监听器，则会移除默认欣慰。
- `SIGTERM`，该信号可以设置监听事件，但不支持 Windows 平台。
- `SIGINT`，该信号支持所有平台，通常使用 `Control+C` 发出。当终端使用原始模式时不会产生该信号。
- `SIGBREAK`，在 Windows 平台上按下 `CTRL + BREAK` 组合键时发出该信号，在非 Windows 平台上可以监听该事件但不能产生该信号。
- `SIGWINCH`，当控制台大小发生变化时发出该信号。在 Windows 平台上，只有当控制台位置发生变化或者使用原始模式运行可读 TTY 时才会向控制台发出该信号。
- `SIGKILL`，在所有平台上都不能对该信号设置监听器，它会无条件地终端 Node.js 进程。
- `SIGSTOP`，不允许对该信号设置监听器

注意，Windows 平台不支持发送信号，但 Node.js 通过 `process.kill()` 和 `child_process.kill()` 等方法提供了一些模拟信号发送的机制。发送信号 `0` 可以测试某个进程是否存在。发送 `SIGINT`、`SIGTERM` 和 `SIGKILL` 将会无条件地终端目标进程。

## process.abort()

该方法可以让系统触发 `abort` 事件，并让 Node.js 退出且生成一个核心文件。

## process.arch

该属性显示当前运行平台的处理器架构，常见值：arm / ia32 / x64。

```js
console.log('This processor architecture is ' + process.arch);
```

## process.argv

该属性是一个包含命令行参数的数组，其数组的第一个元素一定是 `node`，第二个元素一定是 JavaScript 的文件名，剩下元素为其他的命令行参数。

```js
// print process.argv
process.argv.forEach((val, index, array) => {
  console.log(`${index}: ${val}`);
});
```

执行效果：

```bash
$ node process-2.js one two=three four
0: node
1: /Users/mjr/work/node/process-2.js
2: one
3: two=three
4: four
```

## process.chdir(directory)

该方法用于修改进程的当前工作目录，如果执行失败则抛出异常：

```js
console.log(`Starting directory: ${process.cwd()}`);
try {
  process.chdir('/tmp');
  console.log(`New directory: ${process.cwd()}`);
}
catch (err) {
  console.log(`chdir: ${err}`);
}
```

## process.config

该属性是一个对象，包含了编译 Node.js 执行文件的配置参数。该文件类似于运行 `./configure` 脚本时生成的 `config.gupi`。

```js
{
  target_defaults:
   { cflags: [],
     default_configuration: 'Release',
     defines: [],
     include_dirs: [],
     libraries: [] },
  variables:
   {
     host_arch: 'x64',
     node_install_npm: 'true',
     node_prefix: '',
     node_shared_cares: 'false',
     node_shared_http_parser: 'false',
     node_shared_libuv: 'false',
     node_shared_zlib: 'false',
     node_use_dtrace: 'false',
     node_use_openssl: 'true',
     node_shared_openssl: 'false',
     strict_aliasing: 'true',
     target_arch: 'x64',
     v8_use_snapshot: 'true'
   }
}
```

## process.connected

- 布尔值，调用 `process.disconnect()` 之后系统会将该属性设为 false

如果 `process.connected === false`，则进程不再发送消息。

## process.cwd()

该方法返回进程的当前工作目录：

```js
console.log(`Current directory: ${process.cwd()}`);
```

## process.disconnect()

该方法用于关闭与父进程的 IPC 信道，并允许子进程在没有其他连接时平缓结束活动。

该方法和父进程的 `ChildProcess.disconnect()` 完全相同。

如果 Node.js 没有使用 IPC 信道分立子进程，则 `process.disconnect()` 的值为 undefined。

## process.env

该属性是一个对象，包含了开发环境的信息，详见 `environ(7)`。

```js
{ 
    TERM: 'xterm-256color',
    SHELL: '/usr/local/bin/bash',
    USER: 'maciej',
    PATH: '~/.bin/:/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/bin',
    PWD: '/Users/maciej',
    EDITOR: 'vim',
    SHLVL: '1',
    HOME: '/Users/maciej',
    LOGNAME: 'maciej',
    _: '/usr/local/bin/node' 
}
```

开发者可以修改该属性，但是修改后的配置信息并不会影响进程，这也就是说如下操作不会有什么效果：

```bash
$ node -e 'process.env.foo = "bar"' && echo $foo
```

但下面的处理方式会有效果：

```js
process.env.foo = 'bar';
console.log(process.env.foo);
```

赋予 `process.env` 内部属性的值都会被隐式转换为字符串：

```js
process.env.test = null;
console.log(process.env.test);
// => 'null'
process.env.test = undefined;
console.log(process.env.test);
// => 'undefined'
```

使用 `delete` 可以删除 `process.env` 内部的属性：

```js
process.env.TEST = 1;
delete process.env.TEST;
console.log(process.env.TEST);
// => undefined
```

## process.execArgv

该属性包含了一组 Node.js 进程运行时可用的命令行选项。这些选项并不会出现在 `process.argv` 中，且不包含可执行文件、脚本文件名以及文件名之后的所有选项。这些选项有助于使用和父进程相同的参数来分立子进程：

```bash
$ node --harmony script.js --version
```

`process.execArgv` 的值：

```js
['--harmony']
```

`process.argv` 的值：

```js
['/usr/local/bin/node', 'script.js', '--version']
```

## process.execPath

该属性表示启动进程的脚本文件的绝对路径。

```bash
/usr/local/bin/node
```

## process.exit([code])

该方法根据指定的 `code` 参数终端进程。如果缺失参数，则系统默认 `code` 为成功退出码 `0`。

下面代码演示了一种失败的退出：

```js
process.exit(1);
```

执行上述 Node.js 代码的 shell 可以看到退出码 `1`。

## process.exitCode

该属性是一个表示退出码的数值，无论是进程是平稳退出还是使用 `process.exit()` 强制退出，都会返回相应的退出码。

如果使用 `process.exit(code)` 退出进程，则 `code` 将会覆盖 `process.exitCode` 之前的值。

## process.getegid()

注意，该函数只在 POSIX 平台可用，不适用于 Windows、Android 平台。

该方法用于获取进程的有效组身份，详见 `getegid(2)`，且返回值是数值类型的组 id，而不是组名：

```js
if (process.getegid) {
  console.log(`Current gid: ${process.getegid()}`);
}
```

## process.geteuid()

注意，该函数只在 POSIX 平台可用，不适用于 Windows、Android 平台。

该方法用于获取进程的有效用户身份，详见 `geteuid(2)`，且返回值是数值类型的用户 id，而不是用户名：

```js
if (process.geteuid) {
  console.log(`Current uid: ${process.geteuid()}`);
}
```

## process.getgid()

注意，该函数只在 POSIX 平台可用，不适用于 Windows、Android 平台。

该方法用于获取进程的组身份，详见 `getgid(2)`，且返回值是数值类型的组 id，而不是组名：

```js
if (process.getgid) {
  console.log(`Current gid: ${process.getgid()}`);
}
```

## process.getgroups()

注意，该函数只在 POSIX 平台可用，不适用于 Windows、Android 平台。

该方法返回一个包含组 ID 信息的数组。如果存在有效组 ID，则在 POSIX 系统下不确定是否存在该方法返回的数组，而 Node.js 系统会认为一定有该数组。

## process.getuid()

注意，该函数只在 POSIX 平台可用，不适用于 Windows、Android 平台。

该方法用于获取进程的用户身份，详见 `getuid(2)`，且返回值是数值类型的用户 id，而不是用户名：

```js
if (process.getuid) {
  console.log(`Current uid: ${process.getuid()}`);
}
```

## process.hrtime()

该方法以 `[seconds, nanoseconds]` 形式的数组返回高解析度的实时时间。因为它是相对于过去的任意时间而，所以不存在时间漂移。该方法常用于测量代码间歇性执行的性能。

下面代码演示了如何使用 `process.hrtime()` 获取间歇执行的差异时间，这对基准测试非常有用：

```js
var time = process.hrtime();
// [ 1800216, 25 ]

setTimeout(() => {
  var diff = process.hrtime(time);
  // [ 1, 552 ]

  console.log('benchmark took %d nanoseconds', diff[0] * 1e9 + diff[1]);
  // benchmark took 1000000527 nanoseconds
}, 1000);
```

## process.initgroups(user, extra_group)

注意，该函数只在 POSIX 平台可用，不适用于 Windows、Android 平台。

该方法用于读取 `/etc/group` 并初始化群组访问列表，从中选择 `user` 所在的所有群组。这是一个需要超级权限的操作，也就是说开发者需要有 root 权限，或者有 `CAP_SETGID` 的能力。

`user` 是一个用户名或用户 ID，`extra_group` 是一个组名或组 ID。

需要特别留意失去超级权限的情况：

```js
console.log(process.getgroups());        
// [ 0 ]
process.initgroups('bnoordhuis', 1000);  
// switch user
console.log(process.getgroups());        
// [ 27, 30, 46, 1000, 0 ]
process.setgid(1000);                    
// drop root gid
console.log(process.getgroups());        
// [ 27, 30, 46, 1000 ]
```

## process.kill(pid[，signal])

该方法用于向进程发送信号，`pid` 是进程 id，`signal` 是字符串形式的信号，比如 `SIGINT`、`SIGHUP`。如果没有传入 `signal` 参数，那么系统会使用默认值 `SIGTERM`，更多信息请参考信号事件和 `kill(2)`。

如果目标进程不存在，则系统会抛出错误，这里有一个特殊情况，那就是可以使用信号 `0` 测试进程是否存在。如果使用 `pid` 杀死一个进程组，那么 Windows 平台会抛出错误。

注意，虽然该方法的名字是 `process.kill`，但是它只是一个信号发送器，类似系统调用的 `kill` 命令。使用该方法发送信号做得事情实际上比单纯的杀死进程要多很多。

```js
process.on('SIGHUP', () => {
  console.log('Got SIGHUP signal.');
});

setTimeout(() => {
  console.log('Exiting.');
  process.exit(0);
}, 100);

process.kill(process.pid, 'SIGHUP');
```

注意，当 Node.js 接收到 `SIGUSER1` 信号时，系统会启动一个调试器，详见信号事件。

## process.mainModule

该方法是获取 `require.main` 的替代方法。两者的差异在于如果主模块在运行时发生了改变，`require.main` 仍可能引用之前设置的主模块。通常来说，将 `process.mainModule` 和 `require.main` 视为同一个引用是比较安全的。

如果没有入门脚本，那么 `require.main` 的值为 `undefined`。

## process.memoryUsage()

该方法返回一个对象，用于描述 Node.js 进程的内存使用情况，单位为字节：

```js
const util = require('util');

console.log(util.inspect(process.memoryUsage()));
```

输出结果：

```js
{ 
    rss: 4935680,
    heapTotal: 1826816,
    heapUsed: 650472 
}
```

其中，`heapTotal` 和 `heapUsed` 表示 V8 的内存使用情况。

## process.nextTick(callback[, arg][, ...])

- `callback`，函数

当前事件循环完成后，会立即调用回调函数 `callback`。

该方法并不是 `setTimout(fn, 0)` 的同名函数，而是更加高效的方法。该方法在任何 I/O 事件（包括 timers）触发之前执行。

```js
console.log('start');
process.nextTick(() => {
  console.log('nextTick callback');
});
console.log('scheduled');
// Output:
// start
// scheduled
// nextTick callback
```

这一方法对开发 API 非常有用，有助于开发者在对象创建之后 I/O 出现之前分配事件处理器。

```js
function MyThing(options) {
  this.setupOptions(options);

  process.nextTick(() => {
    this.startDoingStuff();
  });
}

var thing = new MyThing();
thing.getReadyForStuff();
// thing.startDoingStuff() gets called now, not before.
```

对 API 来说 100% 的同步执行或 100% 的异步执行都非常重要。请思考一下以下的代码：

```js
// WARNING!  DO NOT USE!  BAD UNSAFE HAZARD!
function maybeSync(arg, cb) {
  if (arg) {
    cb();
    return;
  }

  fs.stat('file', cb);
}
```

这一 API 是有风险的，比如：

```js
maybeSync(true, () => {
  foo();
});
bar();
```

这里无法看清是先执行 `foo()` 还是先执行 `bar()`。

下面的代码实现更加健壮：

```js
function definitelyAsync(arg, cb) {
  if (arg) {
    process.nextTick(cb);
    return;
  }

  fs.stat('file', cb);
}
```

注意，nextTick 队列中的事件完全执行完毕后才会执行 I/O 操作。因此，递归调用 nextTick 回调函数就会阻塞 I/O 操作，类似于 `while(true)` 循环。

## process.pid

该属性拜师进程的 PID：

```js
console.log(`This process is pid ${process.pid}`);
```

## process.platform

该属性表示当前 Node.js 的运行平台，常见值包括：darwin / freebsd / linux / sunos / win32。

```js
console.log(`This platform is ${process.platform}`);
```

## process.release

该属性是一个对象，包含了当前分发版本的元信息，比如源码的 URL 和 headers-only。

`process.release` 包含以下属性：

- `name`，对于 Node.js，该属性的值永远都是字符串形式的 `node`；对于 io.js，该属性的值是 `io.js`
- `sourceUrl`，该属性的值是指向当前版本源代码的 `.tar.gz` 文件的网址
- `headersUrl`，该属性的值是指向包含当前版本头信息的 `.tar.gz` 文件的网址。该文件远小于源代码的文件，可用于编译 Node.js 的插件
- `libUrl`，该属性的值是指向匹配当前版本架构的 `node.lib` 文件的网址，可用于编译 Node.js 的插件。该属性只会出现在 Windows 平台的 Node.js 中，不会出现在其他平台。

```js
{ 
    name: 'node',
    sourceUrl: 'https://nodejs.org/download/release/v4.0.0/node-v4.0.0.tar.gz',
    headersUrl: 'https://nodejs.org/download/release/v4.0.0/node-v4.0.0-headers.tar.gz',
    libUrl: 'https://nodejs.org/download/release/v4.0.0/win-x64/node.lib' 
}
```

开发者根据非分发版本自主构建的 Node.js 版本也许只会包含 `name` 属性，其他属性不确定是否会存在。

## process.send(message[, sendHandle[, options]][, callback])

- `message`，对象
- `sendHandle`，Handle 对象
- `options`，对象
- `callback`，函数
- 返回值类型：布尔值

如果 Node.js 分立的子进程绑定了 IPC 信道，则该子进程可以通过 `process.send()` 方法向父进程发送消息，父进程的 `ChildProcess` 对象可以通过 `message` 事件接收消息。

注意，该方法在内部使用 `JSON.stringify()` 序列化 `message`。

如果 Node.js 新建的子进程没有 IPC 信道，则 `process.send()` 的返回值为 undefined。

## process.setegid(id)

注意，该函数只在 POSIX 平台可用，不适用于 Windows、Android 平台。

该方法用于设置进程的有效组 id，详见 `setegid(2)`。该方法接收一个数值 id 或字符串形式的组名作为参数。如果指定了组名，则该方法会阻塞进程，直到组名被解析为组 ID。

```js
if (process.getegid && process.setegid) {
  console.log(`Current gid: ${process.getegid()}`);
  try {
    process.setegid(501);
    console.log(`New gid: ${process.getegid()}`);
  }
  catch (err) {
    console.log(`Failed to set gid: ${err}`);
  }
}
```

## process.seteuid(id)

注意，该函数只在 POSIX 平台可用，不适用于 Windows、Android 平台。

该方法用于设置进程的有效用户 ID，详见 `seteuid(2)`。该方法接收一个数值 ID 或字符串类型的用户名作为参数。如果指定了用户名，则该方法会阻塞进程，直到用户名被解析为用户 ID。

```js
if (process.geteuid && process.seteuid) {
  console.log(`Current uid: ${process.geteuid()}`);
  try {
    process.seteuid(501);
    console.log(`New uid: ${process.geteuid()}`);
  }
  catch (err) {
    console.log(`Failed to set uid: ${err}`);
  }
}
```

## process.setgid(id)

注意，该函数只在 POSIX 平台可用，不适用于 Windows、Android 平台。

该方法用于设置进程的组 id，详见 `setgid(2)`。该方法接收一个数值 id 或字符串形式的组名作为参数。如果指定了组名，则该方法会阻塞进程，直到组名被解析为组 ID。

```js
if (process.getgid && process.setgid) {
  console.log(`Current gid: ${process.getgid()}`);
  try {
    process.setgid(501);
    console.log(`New gid: ${process.getgid()}`);
  }
  catch (err) {
    console.log(`Failed to set gid: ${err}`);
  }
}
```

## process.setgroups(groups)

注意，该函数只在 POSIX 平台可用，不适用于 Windows、Android 平台。

该方法用于设置组 ID。这是一个需要超级权限的操作，也就是说开发者需要 root 权限或拥有 `CAP_SETGID` 能力。

`groups` 列表可以包含组 ID、组名或者两者的混合。 

## process.setuid(id)

注意，该函数只在 POSIX 平台可用，不适用于 Windows、Android 平台。

该方法用于设置进程的用户 ID，详见 `setuid(2)`。该方法接收一个数值 ID 或字符串类型的用户名作为参数。如果指定了用户名，则该方法会阻塞进程，直到用户名被解析为用户 ID。

```js
if (process.getuid && process.setuid) {
  console.log(`Current uid: ${process.getuid()}`);
  try {
    process.setuid(501);
    console.log(`New uid: ${process.getuid()}`);
  }
  catch (err) {
    console.log(`Failed to set uid: ${err}`);
  }
}
```

## process.stderr

该属性是一个指向 stderr 的可写 stream。

`proces.stderr` 和 `process.stdout` 与 Node.js 中的其他 stream 不同，它们不能被关闭，也不会触发 `finish` 事件，此外，写入数据时会阻塞进程输出数据时会重定向到一个文件。因为磁盘速度通常比较快，而且操作系统会使用回写式缓存，所以阻塞进程的情况很少见。

## process.stdin

该属性是一个 stdin 的可读 stream。

下面代码演示了如何打开表述输入并监听事件的操作：

```js
process.stdin.setEncoding('utf8');

process.stdin.on('readable', () => {
  var chunk = process.stdin.read();
  if (chunk !== null) {
    process.stdout.write(`data: ${chunk}`);
  }
});

process.stdin.on('end', () => {
  process.stdout.write('end');
});
```

作为 Stream 的实例，`process.stdin` 也可以很好的兼容 Node.js v0.10 之前的代码。

在老版的 Stream 模式中，stdin stream 默认是暂停的，所以必须通过调用 `process.stdin.resume()` 去写入数据。注意，也可以通过调用 `process.stdin.resume()` 将 stream 切换为老版模式。

如果你正在创建一个新项目，那么最好使用新的 stream 模式。

## process.stdout

该属性是一个指向 stdout 的可写 stream。

在下面的代码中模拟了 console.log：

```js
console.log = (msg) => {
  process.stdout.write(`${msg}\n`);
};
```

`proces.stderr` 和 `process.stdout` 与 Node.js 中的其他 stream 不同，它们不能被关闭，也不会触发 `finish` 事件，此外，写入数据时会阻塞进程输出数据时会重定向到一个文件。因为磁盘速度通常比较快，而且操作系统会使用回写式缓存，所以阻塞进程的情况很少见。

通过 `process.stderr`、`process.stdout` 或 `process.stdin` 方法读取 `isTTY` 属性，可以检查当前 Node.js 是否运行在 TTY 上下文环境：

```bash
$ node -p "Boolean(process.stdin.isTTY)"
true
$ echo "foo" | node -p "Boolean(process.stdin.isTTY)"
false

$ node -p "Boolean(process.stdout.isTTY)"
true
$ node -p "Boolean(process.stdout.isTTY)" | cat
false
```

## process.title

该属性用于设置或读取 `ps` 中显示的信息。

当使用该方法设置信息时，信息的最大长度由操作系统指定，实际上可能比较短。

在 Linux 和 OS X，该属性的长度受限于二进制名称的长度加上命令行参数的长度，因为它会覆盖参数内存。

虽然 Node.js v0.8 允许较长的进程名称字符串，也支持覆盖环境内存，但是在某些情况下存在不安全或冲突问题。

## process.umask([mask])

该方法用于设置或读取进程文件的掩码。子进程工父进程继承掩码。如果指定了 `mask` 参数，则返回一个旧的掩码，否则返回当前的掩码：

```js
const newmask = 0o022;
const oldmask = process.umask(newmask);
console.log(
  `Changed umask from ${oldmask.toString(8)} to ${newmask.toString(8)}`
);
```

## process.uptime()

该方法返回 Node.js 运行的时间，单位为秒。

## process.version

该属性是一个编译属性，包含 `NODE_VERSION`：

```js
console.log(`Version: ${process.version}`);
```

## process.versions

该属性包含了 Node.js 以及相关依赖的版本信息：

```js
console.log(process.versions);
```

输出结果如下所示：

```js
{ 
    http_parser: '2.3.0',
    node: '1.1.1',
    v8: '4.1.0.14',
    uv: '1.3.0',
    zlib: '1.2.8',
    ares: '1.10.0-DEV',
    modules: '43',
    icu: '55.1',
    openssl: '1.0.1k' 
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