## 工具集

<div class="s s2"></div>

下面这些函数位于 `util` 模块，需要使用 `require()` 方法加载和访问。

`util` 模块设计之初的目的是为 Node.js 的内部 API 服务。该模块中的大部分辅助函数对开发者开发程序也很有帮助。如果你发现这些函数并不能满足你的需求，那么建议你开发自己的辅助工具集。我们无意创建一个功能齐全的 `util` 模块，只会在该模块中编写对 Node.js 内部 API 开发有用的函数。

## util.debuglog(setion)

- `section`，字符串，待调试的程序代码
- 返回值类型：Function，记录函数

该方法常用于


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