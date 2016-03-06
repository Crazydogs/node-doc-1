## Punycode

<div class="s s2"></div>

Node.js 自 v0.6.2 版本加入了 [Punycode.js](https://mths.be/punycode)，通过 `require('punycode')` 的方式访问。

## punycode.decode(string)

将一个纯 ASCII 的 Punycode 字符串转换为 Unicode 字符串：

```js
// decode domain name parts
punycode.decode('maana-pta'); // 'mañana'
punycode.decode('--dqo34k'); // '☃-⌘'
```

## punycode.encode(string)

将一个 Unicode 字符串转换为纯 ASCII 的 Punycode 字符串：

```js
// encode domain name parts
punycode.encode('mañana'); // 'maana-pta'
punycode.encode('☃-⌘'); // '--dqo34k'
```

## punycode.toASCII(domain)

将一个 Unicode 域名字符串转换为 Punycode 字符串，域名中只有非 ASCII 的部分会被转换：

```js
// encode domain names
punycode.toASCII('mañana.com'); // 'xn--maana-pta.com'
punycode.toASCII('☃-⌘.com'); // 'xn----dqo34k.com'
```

## punycode.toUnicode(domain)

将一个 Punycode 的域名字符串转换为 Unicode 字符串，域名中只有 Punycode 的部分会被转换：

```js
// decode domain names
punycode.toUnicode('xn--maana-pta.com'); // 'mañana.com'
punycode.toUnicode('xn----dqo34k.com'); // '☃-⌘.com'
```

## punycode.ucs2

#### punycode.ucs2.decode(string)

根据传入的 `string` 参数创建一个数值形式的编码数组。由于 JavaScript 内部使用 UCS-2，所以该函数会根据 UTF-16 将一对  surrogate halves 转换为一个编码：

```js
punycode.ucs2.decode('abc'); // [0x61, 0x62, 0x63]
// surrogate pair for U+1D306 tetragram for centre:
punycode.ucs2.decode('\uD834\uDF06'); // [0x1D306]
```

## punycode.ucs2.encode(codePoints)

将编码数组转换为字符串：

```js
punycode.ucs2.encode([0x61, 0x62, 0x63]); // 'abc'
punycode.ucs2.encode([0x1D306]); // '\uD834\uDF06'
```

## punycode.version

返回字符串形式的 Punycode 版本号。

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