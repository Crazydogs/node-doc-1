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

注意，













































































































































































































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