## 全局对象

下面这些方法可以直接在模块中使用，值得注意的一点是，其中有一些对象是挂载在全局作用域中的，有一些则是挂载在具体的模块作用域中。

## Class: Buffer

- 函数

用于处理二进制数据。

## __dirname

- 字符串

当前脚本执行时所在的目录：

```js
console.log(__dirname);
// /Users/mjr
```

`__dirname` 实际上并不属于全局，而是存在于每一个模块之中。

## __filename

- 字符串

代码所在文件的文件名，返回一个绝对路径。对主程序来说，这个值和命令行中使用的文件名未必一致。在模块内，该值为模块文件的路径。

下面代码是在 `/Users/mjr` 目录下使用 `node example.js` 启动的：

```js
console.log(__filename);
// /Users/mjr/example.js
```

`__filename` 实际上并不属于全局，而是存在于每一个模块之中。

## clearInterval(t)

取消使用 `setInterval()` 创建的定时器。

## clearTimeout(t)

取消使用 `setTimeout()` 创建的定时器。

## console

- 对象

用于输出数据到 stdout 和 stderr。

## exports

该对象是对 `module.exports` 的引用，更多有关 `exports` 和 `module.exports` 的用法请参考本文档的 `module` 模块。

`exports` 实际上并不属于全局，而是存在于每一个模块之中。

## global

- 对象，全局命名对象

在浏览器中，顶层作用域为全局作用域，这意味着在全局作用域定义一个变量的话，这个变量就自动变成了全局变量。在 Node.js 中有所不同的是，顶层作用于并不是全局作用域，在 Node.js 模块内声明的变量属于模块内部的局部变量。

## module

- 对象

该对象是对当前模块的引用。`module.exports` 主要用来定义模块暴漏给外部的变量，便于其他模块通过 `require()` 获取这些变量。

`module` 实际上并不属于全局，而是存在于每一个模块之中。

## process

- 对象

进程对象。

## require()

- 函数

该方法用于加载模块。

#### require.cache

- 对象

该对象缓存请求到的模块，通过删除该对象的键值对，可以在下次 `require()` 模块时重新加载模块。

#### require.extensions

<div class="s s0"></div>

#### require.resolve()

使用内部的 `require()` 机制查找模块的位置，并不加载模块，只会返回模块的文件名。

## setInterval(cb, ms)

参见 timer 模块的 setInterval()。

## setTimeout(cb, ms)

参考 timer 模块的 setTimeout()。

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