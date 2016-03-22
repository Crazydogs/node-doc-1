## Modules

<div class="s s3"></div>

Node.js 内置了模块加载系统，文件和模块具有一一对应的关系。举例来说，`foo.js` 文件加载了同一目录下的 `circle.js` 模块，则在 `foo.js` 中可以向如下代码所示加载外部模块：

```js
const circle = require('./circle.js');
console.log( `The area of a circle of radius 4 is ${circle.area(4)}`);
```

`circle.js` 的内容：

```js
const PI = Math.PI;

exports.area = (r) => PI * r * r;

exports.circumference = (r) => 2 * PI * r;
```

`circle.js` 模块向外暴漏了两个方法：area() 和 circuference()。如果要将对象或函数置于模块的顶层作用域，则可以将它们挂载在 `exports` 对象下。

模块的本地变量和内部封装的方法都是私有的，在上面的代码中变量 `PI` 就是 `circle.js` 私有的变量。

如果你只想从模块输出一个包含一切的函数或对象，那么可以使用 `module.exports` 而不是 `exports`。

在下面的代码中，`bar.js` 引用了 `square` 模块，该模块输出了一个构造函数：

```js
const square = require('./square.js');
var mySquare = <a href="http://man7.org/linux/man-pages/man2/square.2.html">square(2)</a>;
console.log(`The area of my square is ${mySquare.area()}`);
```

`square` 模块定义在 `square.js` 中：

```js
// assigning to exports will not modify module, must use module.exports
module.exports = (width) => {
  return {
    area: () => width * width
  };
}
```

`require('module')` 定义了 Node.js 的模块系统。

## 访问主模块

由 Node.js 直接访问的入口文件也被叫做主模块，此时 `require.main` 就等于该模块。这也即是说，你可以通过以下代码测试当前文件是否是主文件：

```js
require.main === module
```

假设有一个 `foo.js` 文件，如果运行 `node foo.js`，那么上述代码就会返回 true；如果运行 `require('./foo')`，那么上述代码就会返回 false。

因为 `module` 提供了 `filename` 属性（通常等于 `__filename`），所以当前项目的入口文件可以通过 `require.main.filename` 获得。

## 包管理技巧

Node.js 内置的 `require()` 函数设计之初就是为了支持各种常规的目录结构。类似 dpkg/rpm/npm 的包管理器有助于开发者无需修改 Node.js 的模块即可构建本地的包。

下面我们将给出一个建设性的目录结构：

假设我们有一个文件夹 `/usr/lib/node/<some-package>/<some-version>`，用于包含指定版本的包。

包之间可以相互依赖。为了安装包 `foo`，开发者有可能需要安装指定版本的 `bar`。`bar` 又有可能有其他的依赖，这些依赖甚至会存在冲突或相互引用。

Node.js 首先会查找模块的 realpath（即遇到软链接会解析为真是链接），然后查找存储依赖的 `node_modules` 目录，具体的查找过程如下所以：

1. `/usr/lib/node/foo/1.2.3/`，foo 包，版本为 1.2.3.
1. `/usr/lib/node/bar/4.3.2/`，foo 的依赖包 bar
1. `/usr/lib/node/foo/1.2.3/node_modules/bar`，软链接 `/usr/lib/node/bar/4.3.2/` 解析后获得真实地址
1. `/usr/lib/node/bar/4.3.2/node_modules/*`，bar 所以来的包的软链接

因此，即使遇到循环引用，或者依赖冲突，每一个模块都能得到可用的特定版本的依赖。

当开发者在 `foo` 中使用 `require('bar')` 请求加载 `bar` 后，系统会解析 `bar` 的软链接，获取真实的地址 `/usr/lib/node/foo/1.2.3/node_modules/bar`。然后如果在 `bar` 解析到了 `require('quux')`，那么系统会继续解析软链接拿到真实路径 `/usr/lib/node/bar/4.3.2/node_modules/quux`。

此外，为了优化模块查找效率，我们最好将模块置于 `/usr/lib/node_modules/<name>/<version>` 而不是直接置于 `/usr/lib/node`。

为了在 Node.js 的 REPL 中使用模块，最好将 `/usr/lib/node_modules` 添加给换进变量 `$NODE_PATH`。因为模块查找的 `node_modules` 文件夹使用的都是相对路径，且调用基于文件的真实路径执行 `require()`，所以包实际上可以置于任意位置。

## 

通过 `require()` 加载的模块可以通过 `require.resolve()` 函数获取模块的真实路径。

下面是用伪代码演示的 `require.resolve()` 解析过程:

```js
require(X) from module at path Y
1. If X is a core module,
   a. return the core module
   b. STOP
2. If X begins with './' or '/' or '../'
   a. LOAD_AS_FILE(Y + X)
   b. LOAD_AS_DIRECTORY(Y + X)
3. LOAD_NODE_MODULES(X, dirname(Y))
4. THROW "not found"

LOAD_AS_FILE(X)
1. If X is a file, load X as JavaScript text.  STOP
2. If X.js is a file, load X.js as JavaScript text.  STOP
3. If X.json is a file, parse X.json to a JavaScript Object.  STOP
4. If X.node is a file, load X.node as binary addon.  STOP

LOAD_AS_DIRECTORY(X)
1. If X/package.json is a file,
   a. Parse X/package.json, and look for "main" field.
   b. let M = X + (json main field)
   c. LOAD_AS_FILE(M)
2. If X/index.js is a file, load X/index.js as JavaScript text.  STOP
3. If X/index.json is a file, parse X/index.json to a JavaScript object. STOP
4. If X/index.node is a file, load X/index.node as binary addon.  STOP

LOAD_NODE_MODULES(X, START)
1. let DIRS=NODE_MODULES_PATHS(START)
2. for each DIR in DIRS:
   a. LOAD_AS_FILE(DIR/X)
   b. LOAD_AS_DIRECTORY(DIR/X)

NODE_MODULES_PATHS(START)
1. let PARTS = path split(START)
2. let I = count of PARTS - 1
3. let DIRS = []
4. while I >= 0,
   a. if PARTS[I] = "node_modules" CONTINUE
   c. DIR = path join(PARTS[0 .. I] + "node_modules")
   b. DIRS = DIRS + DIR
   c. let I = I - 1
5. return DIRS
```

## 缓存

系统第一次加载模块时会缓存这些模块，这也就是说，每次调用 `require('foo')` 都会得到相同的返回对象。

缓存有一个很重要的特性的，那就是多次调用 `require('foo')` 并不会让该模块多次运行。根据该特性，当模块返回结束对象之后，即使其他依赖存在循环引用也无所谓。

如果你想多次调用模块的某块代码，最好的方法是从该模块向外输出一个函数，在外部多次调用该函数。

#### 模块缓存警告

缓存是基于模块的文件名进行解析的，所以如果调用位置不同，加载的模块就有可能不同，也就是说在不同的文件中，无法保证 `require('foo')` 每次都返回相同的对象。

此外，在对大小写敏感的操作系统中，不同的文件名有可能指向相同的文件，但缓存仍将其视为不同的模块，继而会多次重载该模块。比如，`require('./foo')` 和 `require('./FOO')` 会返回两个不同的对象，系统并不会检查 `./foo` 和 `./FOO` 是否会指向同一个文件。

## 核心模块

Node.js 内置了一些已经编译成二进制文件的模块，有关这些模块的详细介绍请参考本文的相应章节。

核心模块由 Node.js 源代码定义和实现，保存在 `lib/` 文件夹中。

`require()` 函数总是优先加载核心模块，举例来说，即使存在 `http` 文件，`require('http')` 也总会返回一个 Node.js 内建的 HTTP 模块。

## 循环引用

当 `require()` 出现循环引用时，引用的模块内部可能尚未执行完就返回了值。

我们假设有三个文件，其中 `a.js`：

```js
console.log('a starting');
exports.done = false;
const b = require('./b.js');
console.log('in a, b.done = %j', b.done);
exports.done = true;
console.log('a done');
```

`b.js`：

```js
console.log('b starting');
exports.done = false;
const a = require('./a.js');
console.log('in b, a.done = %j', a.done);
exports.done = true;
console.log('b done');
```

`main.js`：

```js
console.log('main starting');
const a = require('./a.js');
const b = require('./b.js');
console.log('in main, a.done=%j, b.done=%j', a.done, b.done);
```

在上面的代码中，`main.js` 引用了 `a.js`，`a.js` 引用了 `b.js`，当系统继续解析 `b.js` 时，发现 `b.js` 有引用了 `a.js`。为了避免无限循环引用，系统就会将一个 `a.js` 的不完全拷贝返回给 `b.js` 模块，然后 `b.js` 完成相应的解析，最后将 `exports` 对象提供给 `a.js` 模块。

`main.js` 加载这两个模块后完成相应的操作，输出结果如下：

```js
$ node main.js
main starting
a starting
b starting
in b, a.done = false
b done
in a, b.done = true
a done
in main, a.done=true, b.done=true
```

如果你的项目中存在模块的循环引用，建议据此解决。

## 文件模块

如果根据文件名没有找到模块，那么 Node.js 就会尝试使用不同的扩展名去加载模块，比如 `.js`、`.json`，最后是 `.node`。

`.js` 文件会被解析为 JavaScript 文本文件，`.json` 会被解析为 JSON 文本文件，`.node` 文件会被解析为被 `dlopen` 解析过的插件模块。

以 `/` 开头的路径为文件的绝对路径，举例来说，对于 `require('/home/marco/foo.js')`，系统会查找 `/home/marco/foo.js`。

以 `./` 开头的路径为文件的相对路径，表示相对于当前文件所在的目录，举例来说，对于 `foo.js` 文件中的 `require('./circle')`，系统会在 `foo.js` 所在目录下查找 `cicle.js`。

对于不以 `/`、`./` 或 `../` 开头的路径，系统会从核心模块或 `node_modules` 目录查找模块。

如果指定的路径不存在，`require()` 会抛出一个 Error 实例，该实例具有一个值为 `MODULE_NOT_FOUND` 的 `code` 属性。

## 文件夹即模块

将程序和依赖打包到同一个文件夹内并提供一个入口文件，是一种非常便捷的打包方式。这里有三种方式使用 `require()` 加载此类模块。

第一种方式是在根目录创建一个 `package.json` 文件，用于指定 `main` 模块：

```js
{ 
    "name" : "some-library",
    "main" : "./lib/some-library.js" 
}
```

如果该模块位于 `./some-library`，那么 `require('./some-libaray')` 就会尝试加载 `./some-library/lib/some-library.js`。

Node.js 可以正确解析 package.json 配置文件。

如果模块的根目录下没有 package.json，Node.js 就会尝试加载根目录下的 `index.js` 或 `index.node` 文件。举例来说，在某个模块的根目录下没有 package.json，那么 `require('./some-library')` 就会尝试加载：

- `./some-library/index.js`
- `./some-library/index.node`

## 从 node_module 加载模块

如果传给 `require()` 的不是一个原生模块，且不以 `/`、`./` 或 `../` 开头，那么 Node.js 就会从当前模块的父级目录查找 `node_modules`，并尝试从 `node_modules` 加载模块。

如果还是找不到，继续查找上一级目录，如此递归，直到找到或到达系统的根目录。

比如，如果在文件 `'/home/ry/projects/foo.js'` 中调用了 `require('bar.js')`，那么 Node.js 就会一次查找以下文件：

- `/home/ry/projects/node_modules/bar.js`
- `/home/ry/node_modules/bar.js`
- `/home/node_modules/bar.js`
- `/node_modules/bar.js`

这种方式有助于限制依赖的作用范围，避免冲突。

开发者还可以通过添加后缀加载指定的文件或子模块，举例来说，`require('example-module/path/to/file')` 会加载 `example-module` 模块下的 `path/to/file` 文件。模块后缀的路径解析方式和上述模块路径的惊喜方式一致。

## 从全局文件夹加载

Node.js 如果在上述所有地方都找不到模块的话，就会检索环境变量 `NODE_PATH` 下是否存在，在大多数的系统中，`NODE_PATH` 中的路径以冒号分隔，而在 Windows 中，`NODE_PATH` 以逗号分隔。

创建环境变量 `NODE_PATH` 的本意是在上述的模块检索算法失效后检索更多的 Node.js 路径。

虽然现在 Node.js 还支持环境变量 `NODE_PATH`，但该变量已经越来越不重要了，这是因为 Node.js 圈约定俗成的使用本地存放依赖模块。如果模块和 `NODE_PATH` 绑定的话，会让那些不知道 `NODE_PATH` 的用户茫然不知所措。此外，当模块的依赖关系发生变化时，那么检索 `NODE_PATH` 就会加载不同版本的模块，甚至是与预期完全不同的模块。

除了 `NODE_PATH`，Node.js 还会检索以下位置：

1. `$HOME/.node_modules`
2. `$HOME/.node_libraries`
3. `$PREFIX/lib/node`

这里的 `$HOME` 是用户的主页目录，`$PREFIX` 是 Node.js 配置的 `node_prefix`。

上述处理方式大都是由于历史原因形成的。强烈建议开发者将依赖存储于 `node_modules` 文件夹，这将有助于提高系统的解析速度，增强模块的稳定性。

## module 对象

- 对象

在每一个模块中，变量 `module` 就是一个引用当前模块的对象。为了简便起见，可以使用 `exports` 替代 `module.exports`。`module` 实际上并不是一个全局对象，而是存在于每一个模块的本地变量。

#### module.children

- 数组

该属性的值包含了当前模块所加载的 module 对象。

#### module.exports

- 对象

`module.exports` 对象由模块系统所创建。很多开发者想要让模块是类的实例，那么就可以将期望的对象赋值给 `module.exports`。注意，如果赋值给了 `exports`，实际上只是简单地重新绑定到了本地的 `exports` 变量，这有可能会发生意料之外的结果。

举例来说，假设我们有一个模块 `a.js`：

```js
const EventEmitter = require('events');

module.exports = new EventEmitter();

// Do some work, and after some time emit
// the 'ready' event from the module itself.
setTimeout(() => {
  module.exports.emit('ready');
}, 1000);
```

在其他文件中加载该模块：

```js
const a = require('./a');
a.on('ready', () => {
  console.log('module a is ready');
});
```

注意，`module.exports` 的赋值语句必须立即执行，而不能将其置于任何的回调函数之中。在下面的情况下，`module.exports` 的赋值操作是无效的：

```js
// x.js
setTimeout(() => {
  module.exports = { a: 'hello' };
}, 0);
```

加载 `x.js`：

```js
const x = require('./x');
console.log(x.a);
```

#### exports 和 module.exports

模块中的变量 `exports` 最开始的时候是对 `module.exports` 的引用。如果开发者给 `exports` 替换成了对其他变量的引用，那么两者之间就没有任何关系了：

```js
function require(...) {
  // ...
  ((module, exports) => {
    // Your module code here
    exports = some_func;        // re-assigns exports, exports is no longer
                                // a shortcut, and nothing is exported.
    module.exports = some_func; // makes your module export 0
  })(module, module.exports);
  return module;
}
```

如果你无法理解 `exports` 和 `module.exports` 之间的关系，建议你忽略 `exports`，一切都使用 `module.exports`。

#### module.filename

- 字符串

该属性表示模块解析后的文件名。

#### module.id

- 字符串

该属性表示模块的标识符，通产来说就是模块解析后的文件名。

#### module.loaded

- 布尔值

该属性表示模块是已经加载完成还是处于加载过程中。

#### module.parent

- 对象，模块对象

该属性表示第一个加载该模块的文件。

#### module.require(id)

- `id`，字符串
- 返回值类型：对象，模块解析后返回的 `module.exports` 

`module.require()` 方法提供了一种类似原始模块调用 `require()` 的模块加载方式。

注意，开发者必须获得 `module` 对象的引用才可以这么做。因为 `require()` 需要返回 `module.exports`，而 `module` 只代表特定的模块，所以必须显式导出 `module` 才能使用它。


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