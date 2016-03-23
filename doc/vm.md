## VM

<div class="s s2"></div>

通过 `require('vm')` 可以加载该模块。利用该模块可以立即编译执行或先编译保存后执行 JavaScript 代码。

## Class: Script

该类封装了一些预编译脚本，并在沙盒中运行这些脚本。

#### new vm.Script(code, options)

该方法是一个构造函数，创建一个 `Script` 实例并编译 `code`，但不运行编译后的代码，最终返回的 `vm.Script` 对象存储了编译后的代码。使用下面的函数可以反复运行 `vm.Script` 中的代码。`vm.Script` 中的代码并没有绑定到任何的全局对象上，它们会在运行前绑定，并运行后解绑。

参数 `options` 包含以下属性：

- `filename`，该属性允许开发者更改堆栈跟踪信息中的文件名
- `lineOffset`，该属性用于在堆栈跟踪信息中为行号设置偏移位置
- `columnOffset`，该属性用于在堆栈跟踪信息中为列号设置偏移位置
- `displayErrors`，该属性决定在抛出异常之前，是否向 stderr 输出错误并高亮问题代码。只有编译期的语法问题会被抛出，执行期的问题则有 Script 的函数处理
- `timeout`，用于指定 `code` 执行的超时时间，超时则中断执行，并抛出错误
- `cacheData`，该属性是一个包含 V8 代码缓存数据的 Buffer。`cachedDataRejected` 是布尔类型，其值取决于是否接受 V8 的数据。
- `produceCachedDate`，如果值为 true 且没有 `cachedData` 参数，V8 会尝试为 `code` 缓存代码，如果缓存成功，就会生成一个 Buffer 实例存储代码的缓存数据，该 Buffer 实例也会成为返回的 `vm.Script` 的一个实例。`cachedDataProduced` 是一个布尔类型，其值取决于系统是否成功生成了代码缓存数据。

#### script.renInContext(contextifiedSandbox[, options])

该方法与 `vm.runInContext()` 类似，但后者是挂载在预编译 `Script` 对象下的一个方法。`scirpt.runInContext()` 方法在 `contextifiedSandbox` 中运行 `Script` 编译后的代码并返回相应的结果。运行代码时并不会访问本地作用域。

`script.runInContext` 接收的 `options` 和 `script.runInThisContext()` 一致。

下面代码演示了使用 VM 模块编译后的代码修改全局变量并反复执行的效果：

```js
const util = require('util');
const vm = require('vm');

var sandbox = {
  animal: 'cat',
  count: 2
};

var context = new vm.createContext(sandbox);
var script = new vm.Script('count += 1; name = "kitty"');

for (var i = 0; i < 10; ++i) {
  script.runInContext(context);
}

console.log(util.inspect(sandbox));

// { animal: 'cat', count: 12, name: 'kitty' }
```

注意运行不可信代码时需要万分小心。`script.runInContext()` 虽然很有用，但是运行不可信代码时最好使用一个独立的进程。

#### script.runInNewContext([sandbox][, options])

该方法与 `vm.runInNewContext()` 类似，但后者是挂载在预编译 `Script` 对象下的一个方法。`scirpt.runInNewContext()` 如果没有收到 `sandbox` 就会自建一个沙盒，然后在沙盒中以全局对象的形式运行 `Script` 编译后的代码。运行代码时并不会访问本地作用域。

`script.runInNewContext` 接收的 `options` 和 `script.runInThisContext()` 一致。

下面代码演示了使用 VM 模块编译后的代码修改全局变量并反复执行的效果：

```js
const util = require('util');
const vm = require('vm');

const sandboxes = [{}, {}, {}];

const script = new vm.Script('globalVar = "set"');

sandboxes.forEach((sandbox) => {
  script.runInNewContext(sandbox);
});

console.log(util.inspect(sandboxes));

// [{ globalVar: 'set' }, { globalVar: 'set' }, { globalVar: 'set' }]
```

注意运行不可信代码时需要万分小心。`script.runInNewContext()` 虽然很有用，但是运行不可信代码时最好使用一个独立的进程。

#### script.runInThisContext([options])

该方法与 `vm.runInThisContext()` 类似，但后者是挂载在预编译 `Script` 对象下的一个方法。`scirpt.runInThisContext()` 方法运行 `Script` 编译后的代码并返回相应的结果。运行代码时没有访问本地作用域的权限，但有访问 `global` 对象的权限。

下面代码演示了使用 `script.runInThisContext()` 编译代码并多次运行的效果：

```js
const vm = require('vm');

global.globalVar = 0;

const script = new vm.Script('globalVar += 1', { filename: 'myfile.vm' });

for (var i = 0; i < 1000; ++i) {
  script.runInThisContext();
}

console.log(globalVar);
// 1000
```

`options` 参数接收一下参数：

- `filename`，该属性允许开发者更改堆栈跟踪信息中的文件名
- `lineOffset`，该属性用于在堆栈跟踪信息中为行号设置偏移位置
- `columnOffset`，该属性用于在堆栈跟踪信息中为列号设置偏移位置
- `displayErrors`，该属性决定在抛出异常之前，是否向 stderr 输出错误并高亮问题代码。只有编译期的语法问题会被抛出，执行期的问题则有 Script 的函数处理
- `timeout`，用于指定 `code` 执行的超时时间，超时则中断执行，并抛出错误

## vm.createContext([sandbox])

如果指定了 `sandbox` 对象，那么该使用该对象调用 `vm.runInContext()` 或 `script.runInContext()` 方法。在脚本内部，`sandbox` 会被视为全局对象，包含所有的已有属性以及标准全局对象所拥有的内建对象和方法。在 VM 模块所执行的脚本外部，`sandbox` 不会被修改。

如果指定沙盒对象，那么该方法返回一个新的空沙盒对象。

该方法常用于创建一个可以运行多个脚本的沙盒，比如为一个模拟浏览器创建煞和对象，该沙盒对象包含了全局对象 window 的属性和方法，最后将所有的 `script` 标签置于沙盒中运行。

## vm.isContext(sandbox)

该方法返回一个布尔值，用于判定沙盒对象是否经由 `vm.createContext()` 初始化过。

## vm.runInContext(code, contextifiedSandbox[, options])

`vm.runInContext()` 方法先编译 `code`，然后使用指定的 `contextifiedSandbox` 运行代码并返回结果。运行中的代码没有访问本地作用域的权限。`contextifiedSandbox` 对象必须使用 `vm.createContext()` 创建或初始化，它会被视为 `code` 的全局对象。

`vm.runInContext` 接收的 `options` 和 `vm.runInThisContext()` 一致。

下面代码演示了使用同一个上下文环境编译和执行不同脚本的效果：

```js
const util = require('util');
const vm = require('vm');

const sandbox = { globalVar: 1 };
vm.createContext(sandbox);

for (var i = 0; i < 10; ++i) {
    vm.runInContext('globalVar *= 2;', sandbox);
}
console.log(util.inspect(sandbox));
// { globalVar: 1024 }
```

注意运行不可信代码时需要万分小心。`vm.runInContext()` 虽然很有用，但是运行不可信代码时最好使用一个独立的进程。

## vm.runInDebugContext(code)

`vm.runInDebugContext()` 在 V8 的调试上下文环境中编译和执行 `code`。该方法最主要的场景是获取 V8 的调试对象：

```js
const Debug = vm.runInDebugContext('Debug');
Debug.scripts().forEach(function(script) { console.log(script.name); });
```

注意，这里的调试上下文和对象都是和 V8 的调试器绑定在一起的，所以当 V8 引擎有所改动时可能并不会有警告。

通过 `--expose_debug_as= switch` 也可以暴漏调试对象。

## vm.runInNewContext(code[, sandbox][, options])

`vm.runInNewContext()` 如果没有收到 `sandbox` 就会自建一个沙盒，然后在沙盒中编译 `code`，最后使用沙盒作为全局对象执行编译后的代码并返回结果。

`vm.runInNewContext` 接收的 `options` 和 `vm.runInThisContext()` 一致。

下面代码演示了使用 VM 模块编译后的代码修改全局变量并反复执行的效果：

```js
const util = require('util');
const vm = require('vm');

const sandbox = {
  animal: 'cat',
  count: 2
};

vm.runInNewContext('count += 1; name = "kitty"', sandbox);
console.log(util.inspect(sandbox));
// { animal: 'cat', count: 3, name: 'kitty' }
```

注意运行不可信代码时需要万分小心。`vm.runInNewContext()` 虽然很有用，但是运行不可信代码时最好使用一个独立的进程。

## vm.runInThisContext(code[, options])

`vm.runInThisContext()` 方法用于编译执行 `code` 并返回结果。运行代码时没有访问本地作用域的权限，但有访问 `global` 对象的权限。

下面代码演示了使用 `vm.runInThisContext()` 和 `eval()` 运行相同代码的差别：

```js
const vm = require('vm');
var localVar = 'initial value';

const vmResult = vm.runInThisContext('localVar = "vm";');
console.log('vmResult: ', vmResult);
console.log('localVar: ', localVar);

const evalResult = eval('localVar = "eval";');
console.log('evalResult: ', evalResult);
console.log('localVar: ', localVar);
// vmResult: 'vm', localVar: 'initial value'
// evalResult: 'eval', localVar: 'eval'
```

`vm.runInThisContext()` 并没有访问本地作用域的权限，所以 `localVar` 不会被修改。`eval` 具有访问本地作用域的权限，所以可以修改 `loalVar`。

虽然 `vm.runInThisContext()` 的这种用法很类似直接调用 `eval()`，比如 `(0,eval)('code')`，但是 `vm.runInThisContext()` 的优势在于还可以接收以下参数：

- `filename`，该属性允许开发者更改堆栈跟踪信息中的文件名
- `lineOffset`，该属性用于在堆栈跟踪信息中为行号设置偏移位置
- `columnOffset`，该属性用于在堆栈跟踪信息中为列号设置偏移位置
- `displayErrors`，该属性决定在抛出异常之前，是否向 stderr 输出错误并高亮问题代码。只有编译期的语法问题会被抛出，执行期的问题则有 Script 的函数处理
- `timeout`，用于指定 `code` 执行的超时时间，超时则中断执行，并抛出错误

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