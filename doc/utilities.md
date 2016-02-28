## 工具集

<div class="s s2"></div>

下面这些函数位于 `util` 模块，需要使用 `require()` 方法加载和访问。

`util` 模块设计之初的目的是为 Node.js 的内部 API 服务。该模块中的大部分辅助函数对开发者开发程序也很有帮助。如果你发现这些函数并不能满足你的需求，那么建议你开发自己的辅助工具集。我们无意创建一个功能齐全的 `util` 模块，只会在该模块中编写对 Node.js 内部 API 开发有用的函数。

## util.debuglog(setion)

- `section`，字符串，待调试的程序代码
- 返回值类型：Function，记录函数

该方法常用于创建一个限制性函数，该函数根据是否有 `NODE_DEBUG` 环境变量向 stderr 输出数据。如果 `section` 的名称出现在环境变量中，那么就会返回一个类似 `console.error()` 的函数，否则返回一个空操作。

```js
var debuglog = util.debuglog('foo');

var bar = 123;
debuglog('hello from foo [%d]', bar);
```

如果程序在 `NODE_DEBUG=foo` 的环境中运行，那么它就会输出类似下面这样的信息：

```js
FOO 3245: hello from foo [123]
```

这里的 `3245` 是进程 ID。如果程序运行的环境中没有这样的变量，则不会输出任何信息。此外，你可以使用逗号分隔多个 `NODE_DEBUG` 环境变量，比如 `NODE_DEBU=fs,net,tls`。

## util.deprecate(function, string)

该方法用来标明某个函数不能再被使用：

```js
const util = require('util');

exports.puts = util.deprecate(() => {
  for (var i = 0, len = arguments.length; i < len; ++i) {
    process.stdout.write(arguments[i] + '\n');
  }
}, 'util.puts: Use console.log instead');
```

最终返回一个修改后的函数，调用该函数时默认会发出一次警示。

如果开启了 `--no-deprecation` 选项，则该函数为空操作。通过布尔类型的 `process.noDeprecation`，可以在程序运行中配置该参数，需要注意的是，必须在模块加载前执行该配置。

如果开启了 `--trace-deprecation` 选项，则该函数第一次被调用时，会在控制台输出提示和堆栈跟踪信息。通过布尔类型的 `process.traceDeprecation`，可以在程序运行中配置该参数。

如果开启了 `--throw-deprecation` 选项，则该函数被调用时抛出错误。通过布尔类型的 `process.throwDeprecation`，可以在程序运行中配置该参数。`process.throwDeprecation` 优先级高于 `process.traceDeprecation`。

## util.format(format[, ...])

该方法类似 `printf` 函数，返回一个格式化后的字符串。

第一个参数是一个字符串，包含一个或多个占位符。支持的占位符包括以下几种：

- `%s`，字符串
- `%d`，数值，整数或浮点数
- `%j`，JSON，如果参数中包含循环引用，则使用 `[Circular]` 表示
- `%%`，百分号

如果占位符缺少对应的参数，则直接输出该占位符：

```js
util.format('%s:%s', 'foo'); 
// 'foo:%s'
```

如果参数多余占位符，则该参数被转换为字符串（如果参数是对象或者 symbol，则使用 `util.inspect()` 进行转换），然后使用空格连接成一个字符串：

```js
util.format('%s:%s', 'foo', 'bar', 'baz'); 
// 'foo:bar baz'
```

如果第一个参数不包含占位符，则 `util.format()` 返回一个字符串，该字符串由所有参数连接而成，参数之间以空格分隔。参数转换时，使用 `util.inspect()` 方法进行转换：

```js
util.format(1, 2, 3); 
// '1 2 3'
```

## util.inherits(constructor, superConstructor)

该方法使 `constructor` 构造函数继承 `superConstructor` 构造函数的原型方法。这里的 `superConstructor` 也可以通过 `constructor.super_` 属性得到。

```js
const util = require('util');
const EventEmitter = require('events');

function MyStream() {
    EventEmitter.call(this);
}

util.inherits(MyStream, EventEmitter);

MyStream.prototype.write = function(data) {
    this.emit('data', data);
}

var stream = new MyStream();

console.log(stream instanceof EventEmitter); 
// true
console.log(MyStream.super_ === EventEmitter); 
// true

stream.on('data', (data) => {
  console.log(`Received data: "${data}"`);
})
stream.write('It works!'); 
// Received data: "It works!"
```

## util.inspect(object[, options])

该方法返回 `object` 参数的字符串形式，常用于代码调试。

可选的参数对象 `options` 可以用来改变字符串格式化后的样式：

- `showHidden`，如果为 true，则显示对象的不可枚举和 symbol 属性，默认值为 false
- `depth`，指定 `inspect()` 方法格式化 `object` 时的深度。这对于格式化复杂对象很有用。默认值为 2，如果不限制深度，可以设置为 `null
- `colors`，如果值为 true，则使用 ANSI 颜色码对输出信息加以美化。默认值为 false
- `customInspect`，如果值为 false，则 `object` 对象上自定义的 `inspect(depth, opts)` 函数不会被执行，默认值为 true。

下面代码演示了如何审查 `util` 对象的所有属性：

```js
const util = require('util');

console.log(util.inspect(util, { showHidden: true, depth: null }));
```

该方法被调用时，`object` 对象可以使用自定义的 `inspect(depth, opts)` 方法，该方法接收两个参数，一个是当前的递归深度，另一个是传递给 `util.inspect()` 的可选对象。

#### 自定义 util.inspect 颜色

`util.inspect()` 的输出颜色可由 `util.inspect.styles` 和 `util.inspect.colors` 进行配置。

`util.inspect.styles` 与 `util.inpect.colors` 中的颜色一一对应。常见数据类型的默认高亮样式：

- Number，黄色
- Boolean，黄色
- String，绿色
- Date，洋红
- Regexp，红色
- Null，黑体，
- Undefined，灰色，
- Special，青色，目前 special 只包括 Function 类型
- Name，默认无样式

系统预定义的颜色包括：`white` / `grey` / `black` / `blue` / `cyan` / `green` / `magenta` / `red` 和 `yellow`。此外，还有 `bold` / `italic` / `underline` 和 `inverse` 样式。

#### 自定义对象的 inspect() 函数

对象也可以自定义 `inspect(depth)` 函数，当 `util.inspect()` 方法审查该对象时就会调用这一自定义的方法：

```js
const util = require('util');

var obj = { name: 'nate' };
obj.inspect = function(depth) {
  return `{${this.name}}`;
};

util.inspect(obj);
// "{nate}"
```

也可以从自定义的方法返回另一个对象，`util.inspect()` 方法会返回根据该对象返回格式化的字符串，非常类似 `JSON.stringify()` 的工作方式：

```js
var obj = { foo: 'this will not show up in the inspect() output' };
obj.inspect = function(depth) {
  return { bar: 'baz' };
};

util.inspect(obj);
// "{ bar: 'baz' }"
```

## util.log(string)

在 `stdout` 输出的同时带上时间戳。

```js
require('util').log('Timestamped message.');
```

## 以下函数已被抛弃

<div class="s s0"></div>

- util.debug(string)
- util.error([...])
- util.isArray(object)
- util.isBoolean(object)
- util.isBuffer(object)
- util.isDate(object)
- util.isError(object)
- util.isFunction(object)
- util.isNull(object)
- util.isNullOrUndefined(object)
- util.isNumber(object)
- util.isObject(object)
- util.isPrimitive(object)
- util.isRegExp(object)
- util.isString(object)
- util.isSymbol(object)
- util.isUndefined(object)
- util.print([...])
- util.pump(readableStream, writableStream[, callback])
- util.puts([...])

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