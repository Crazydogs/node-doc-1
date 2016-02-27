## 调试器

<div class="s s2"></div>

Node.js 内置了功能强大的调试工具，该工具可通过 TCP 协议或者内建的调试客户端进行调用。使用时，需要以 `debug` 参数启动 Node.js，后跟要调试的脚本文件路径，调试器启动后会显示一个提示符 `debug>`：

```js
$ node debug myscript.js
< debugger listening on port 5858
connecting... ok
break in /home/indutny/Code/git/indutny/myscript.js:1
  1 x = 5;
  2 setTimeout(() => {
  3   debugger;
debug>
```

Node.js 的调试器客户端现在还不能支持完整的调试命令，仅支持简单的步进和检查。

将 `debugger;` 语句插入到脚本之中，相当于在特定代码行设置了断点。假设我们有如下所示的一段代码：

```js
// myscript.js
x = 5;
setTimeout(() => {
  debugger;
  console.log('world');
}, 1000);
console.log('hello');
```

调试器启动欧，在代码的第四行设置了一个断点，所以程序执行会出现暂停：

```js
$ node debug myscript.js
< debugger listening on port 5858
connecting... ok
break in /home/indutny/Code/git/indutny/myscript.js:1
  1 x = 5;
  2 setTimeout(() => {
  3   debugger;
debug> cont
< hello
break in /home/indutny/Code/git/indutny/myscript.js:3
  1 x = 5;
  2 setTimeout(() => {
  3   debugger;
  4   console.log('world');
  5 }, 1000);
debug> next
break in /home/indutny/Code/git/indutny/myscript.js:4
  2 setTimeout(() => {
  3   debugger;
  4   console.log('world');
  5 }, 1000);
  6 console.log('hello');
debug> repl
Press Ctrl + C to leave debug repl
> x
5
> 2+2
4
debug> next
< world
break in /home/indutny/Code/git/indutny/myscript.js:5
  3   debugger;
  4   console.log('world');
  5 }, 1000);
  6 console.log('hello');
  7
debug> quit
```

通过 REPL 命令可以远程执行调试代码，其中 `next` 命令是单步调试命令，输入 `help` 可以查看完整的调试命令。

## 监视器

在调试过程中，可以同时监视表达式和变量的值。在每一处断点，监视器都会计算监视目标（表达式和变量）的值。输入 `watch('my_expression')` 即可监视表达式，输入 `watchers` 输出当前所有的监视器，输入 `unwatch('my_expression')` 对表达式取消监视。

## 命令

#### 步进

- `cont` 或 `c`，继续执行
- `next` 或 `n`，单步执行
- `step` 或 `s`，step in
- `out` 或 `o`，step out
- `pause`，暂停

#### 断点

- `setBreakpoint()` 或 `sb() -`，当前行设置断点
- `setBreakpoint(line)` 或 `sb(line)`，在 `line` 行设置断点
- `setBreakpoint('fn()')` 或 `sb(...)`，在函数里的第一行设置断点
- `setBreakpoint('script.js', 1)` 或 `sb(...)`，在脚本文件的第一行设置断点
- `clearBreakpoint('script.js', 1)` 或 `cb(...)`，清除脚本文件第一行的断点

下面代码演示了如何在未加载的文件中设置断点：

```js
$ ./node debug test/fixtures/break-in-module/main.js
< debugger listening on port 5858
connecting to port 5858... ok
break in test/fixtures/break-in-module/main.js:1
  1 var mod = require('./mod.js');
  2 mod.hello();
  3 mod.hello();
debug> setBreakpoint('mod.js', 23)
Warning: script 'mod.js' was not loaded yet.
  1 var mod = require('./mod.js');
  2 mod.hello();
  3 mod.hello();
debug> c
break in test/fixtures/break-in-module/mod.js:23
 21
 22 exports.hello = () => {
 23   return 'hello from module';
 24 };
 25
debug>
```

#### 信息显示

- `backtrace` 或 `bt`，显示当前执行帧的回溯信息
- `list(5)`，显示脚本代码的五行上下文，即 `debugger;` 之前的五行和之后的五行
- `watch(expr)`，添加对表达式 `expr` 的监视
- `unwatch(expr)`，移除对表达式 `expr` 的监视
- `watchers`，显示所有的监视器和相应的值
- `repl`，启动调试器的 REPL 环境，并使用调试脚本的上下文信息处理代码逻辑
- `exec expr`，在脚本文件的上下文中调试表达式

#### 执行控制

- `run`，执行脚本（脚本在调试器启动时会自执行）
- `restart`，重新执行脚本
- `kill`，杀死正在执行的脚本

#### 其他

- `scripts`，显示已加载脚本的列表
- `version`，显示 V8 的版本

## 高级用法

此外还有两种方式可以打开调试器：一种方法是使用 `--debug` 参数开启调试器，另一种方法是像 Node.js 进程传递 `SIGUSR1` 信号。

Node.js 进程使用上述方式进入调试模式后，可以使用 Node.js 的调试器通过 PID 或 URI 调试该进程：

- `node debug -p <pid>`，通过 PID 连接到进程
- `node debug <URI>`，- 通过类似 localhost:5858 的 URI 连接到进程

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