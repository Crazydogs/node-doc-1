## 查询字符串

<div class="s s2"></div>

该模块提供了处理查询字符串的辅助函数。

## querystring.escape

该转义函数供 `querystring.stringify` 使用，必要时可以重写。

## querystring.parse(str[, sep][, eq][, options])

该方法用于将字符串反序列化为对象，第二和第三个参数分别表示分隔符（默认为 '&'）和分配符（默认为 '='）。

`options` 对象可以包含 `maxKeys` 属性（默认值为 1000），该属性用于限制处理过的进程数量，如果该属性的值为 0，则表示不限制。

`options` 对象可以包含 `decodeURIComponent` 属性（默认值为 querystring.unescape），如有需要可用于解码非 UTF-8 编码的字符串。

```js
querystring.parse('foo=bar&baz=qux&baz=quux&corge')
// returns
{ foo: 'bar', baz: ['qux', 'quux'], corge: '' }

// Suppose gbkDecodeURIComponent function already exists,
// it can decode `gbk` encoding string
querystring.parse('w=%D6%D0%CE%C4&foo=bar', null, null,
  { decodeURIComponent: gbkDecodeURIComponent })
// returns
{ w: '中文', foo: 'bar' }
```

## querystring.stringify(obj[, sep][, eq][, options])

该方法用于将对象序列化为字符串，第二和第三个参数分别表示分隔符（默认为 '&'）和分配符（默认为 '='）。

`options` 对象可以包含 `encodeURIComponent` 属性（默认值为 querystring.escape），如有需要可用于转义非 UTF-8 编码的字符串。

```js
querystring.stringify({ foo: 'bar', baz: ['qux', 'quux'], corge: '' })
// returns
'foo=bar&baz=qux&baz=quux&corge='

querystring.stringify({foo: 'bar', baz: 'qux'}, ';', ':')
// returns
'foo:bar;baz:qux'

// Suppose gbkEncodeURIComponent function already exists,
// it can encode string with `gbk` encoding
querystring.stringify({ w: '中文', foo: 'bar' }, null, null,
  { encodeURIComponent: gbkEncodeURIComponent })
// returns
'w=%D6%D0%CE%C4&foo=bar'
```

## querystring.unescape

该编码函数供 `querystring.parse` 使用，必要时可以重写。该函数首先会尝试使用 `decodeURIComponent` 进行解码，如果解码失败会回退到一个安全状态，不会抛出错误的 URL。

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