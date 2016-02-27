## 字符串解码器

<div class="s s2"></div>

使用该模块前，需要通过 `require('string_decoder')` 加载该模块。字符串解码器常用于将 Buffer 数据解码为 String。它是 `buffer.toString` 方法的一个简单实现，但额外提供了对 `utf8` 的支持。 

```js
const StringDecoder = require('string_decoder').StringDecoder;
const decoder = new StringDecoder('utf8');

const cent = new Buffer([0xC2, 0xA2]);
console.log(decoder.write(cent));

const euro = new Buffer([0xE2, 0x82, 0xAC]);
console.log(decoder.write(euro));
```

## Class: StringDecoder

接受一个 `encoding` 参数，默认值为 `utf8`。

#### decoder.end()

返回所有留在缓冲区中的尾字节。

#### decoder.write()

返回解码后的字符串。

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