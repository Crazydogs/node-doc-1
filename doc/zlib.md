## Zlib

<div class="s s2"></div>

通过 `require('zilb')` 的方式可以访问 `zlib` 模块，该模块提供了对 Gzip/Gunzip，Deflate/Inflate 和 DeflateRaw/InflateRaw 类的绑定，这些类接收相同的参数，都属于可读写的 Stream 实例。

## 实例

下面代码演示了通过 `fs.ReadStream` 管道传输到 zlib stream 在传输到 `fs.WriteStream` 所实现的文件解压缩流程：

```js
const gzip = zlib.createGzip();
const fs = require('fs');
const inp = fs.createReadStream('input.txt');
const out = fs.createWriteStream('input.txt.gz');

inp.pipe(gzip).pipe(out);
```

下面代码演示了使用这些方法一步实现数据解压缩的过程：

```js
const input = '.................................';
zlib.deflate(input, (err, buffer) => {
  if (!err) {
    console.log(buffer.toString('base64'));
  } else {
    // handle error
  }
});

const buffer = new Buffer('eJzT0yMAAGTvBe8=', 'base64');
zlib.unzip(buffer, (err, buffer) => {
  if (!err) {
    console.log(buffer.toString());
  } else {
    // handle error
  }
});
```

在 HTTP 客户端或服务端使用该模块时，请使用 `accept-encoding` 发送请求，使用 `context-encoding` 头信息发送响应。

注意：这些示例的目的只是用于演示该模块的基础概念。Zlib 的编码过程是非常耗时的，所以开发者应该缓存编码结果。更多有关 zlib 模块中对 speed/memory/compression 的权衡和考虑，请参考 [Memory Usage Tuning](https://nodejs.org/dist/latest-v5.x/docs/api/zlib.html#zlib_memory_usage_tuning)。

```js
// client request example
const zlib = require('zlib');
const http = require('http');
const fs = require('fs');
const request = http.get({ host: 'izs.me',
                         path: '/',
                         port: 80,
                         headers: { 'accept-encoding': 'gzip,deflate' } });
request.on('response', (response) => {
  var output = fs.createWriteStream('izs.me_index.html');

  switch (response.headers['content-encoding']) {
    // or, just use zlib.createUnzip() to handle both cases
    case 'gzip':
      response.pipe(zlib.createGunzip()).pipe(output);
      break;
    case 'deflate':
      response.pipe(zlib.createInflate()).pipe(output);
      break;
    default:
      response.pipe(output);
      break;
  }
});

// server example
// Running a gzip operation on every request is quite expensive.
// It would be much more efficient to cache the compressed buffer.
const zlib = require('zlib');
const http = require('http');
const fs = require('fs');
http.createServer((request, response) => {
  var raw = fs.createReadStream('index.html');
  var acceptEncoding = request.headers['accept-encoding'];
  if (!acceptEncoding) {
    acceptEncoding = '';
  }

  // Note: this is not a conformant accept-encoding parser.
  // See http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.3
  if (acceptEncoding.match(/\bdeflate\b/)) {
    response.writeHead(200, { 'content-encoding': 'deflate' });
    raw.pipe(zlib.createDeflate()).pipe(response);
  } else if (acceptEncoding.match(/\bgzip\b/)) {
    response.writeHead(200, { 'content-encoding': 'gzip' });
    raw.pipe(zlib.createGzip()).pipe(response);
  } else {
    response.writeHead(200, {});
    raw.pipe(response);
  }
}).listen(1337);
```

## 内存调优

ZLIB 模块的配置文件存放于 `zili/zconf.h`，通过修改该文件可以调整 Node.js 系统的性能。

deflate 压缩算法需要的内存计算方式：

```js
(1 << (windowBits+2)) +  (1 << (memLevel+9))
```

默认情况下，总共需要 128K（windowBits = 15）+ 128K（memLevel = 8），再加上一些小对象占用的字节。

举例来说，如果开发者想将默认的内存大小从 256K 减少为 128K，那么就可以传入参数：

```js
{ windowBits: 14, memLevel: 7 }
```

当然，这么做会降低压缩等级。

inflate 解压算法需要的内存计算方式：

```js
1 << windowBits
```

默认情况下，总共需要 32K（windowBits = 15），再加上一些小对象占用的字节。

上面的计算都排除了内部输出缓存 `chunkSize`，其默认值为 16K。

zlib 的压缩速度主要受 `level` 的影响，压缩级别越高，压缩效果就会越好，但压缩时间就会越长，反之异然。

通常来说，可用内存越多，意味着对 zlib 模块在内存的交换越少，通过一次 `write` 操作所能处理的数据也就越多，所以内存的可用量也是一个影响压缩速度的重要因素。

## 常量

所有定义在 `zlib.h` 中的变量也可以通过 `require('zlib')` 获取。对于常规操作，开发者通常无需配置这些变量。本节内容主要选取自 [zlib 的文档](http://zlib.net/manual.html#Constants)，更多相关信息请参考 [http://zlib.net/manual.html#Constants](http://zlib.net/manual.html#Constants)。

允许刷新的值：

- zlib.Z_NO_FLUSH
- zlib.Z_PARTIAL_FLUSH
- zlib.Z_SYNC_FLUSH
- zlib.Z_FULL_FLUSH
- zlib.Z_FINISH
- zlib.Z_BLOCK
- zlib.Z_TREES

以下是压缩、解压函数的返回值，负值表示错误，正值表示特殊但正常的事件：

- zlib.Z_OK
- zlib.Z_STREAM_END
- zlib.Z_NEED_DICT
- zlib.Z_ERRNO
- zlib.Z_STREAM_ERROR
- zlib.Z_DATA_ERROR
- zlib.Z_MEM_ERROR
- zlib.Z_BUF_ERROR
- zlib.Z_VERSION_ERROR

压缩级别：

- zlib.Z_NO_COMPRESSION
- zlib.Z_BEST_SPEED
- zlib.Z_BEST_COMPRESSION
- zlib.Z_DEFAULT_COMPRESSION

压缩策略：

- zlib.Z_FILTERED
- zlib.Z_HUFFMAN_ONLY
- zlib.Z_RLE
- zlib.Z_FIXED
- zlib.Z_DEFAULT_STRATEGY

`data_type` 的可选值：

- zlib.Z_BINARY
- zlib.Z_TEXT
- zlib.Z_ASCII
- zlib.Z_UNKNOWN

deflate 压缩算法：

- zlib.Z_DEFLATED

初始化 zalloc/zfree/opaque：

- zlib.Z_NULL

## Class Options

每一个类都接收一个可选对象作为配置参数，且所有的配置参数都是可选的。

注意，有些可选属性只对压缩过程有效，对解压过程无效：

- flush (默认值 zlib.Z_NO_FLUSH)
- chunkSize (默认值 16*1024)
- windowBits
- level (只对压缩有效)
- memLevel (只对压缩有效)
- strategy (只对压缩有效)
- dictionary (只对 deflate/inflate 算法有效, 默认为空目录）

更多有关 `deflateInit2` 和 `inflateInit2` 的信息请参考 [http://zlib.net/manual.html#Advanced](http://zlib.net/manual.html#Advanced)。

## Class: zlib.Deflate

使用 deflate 算法压缩数据。

## Class: zlib.DeflateRaw

使用 deflate 算法压缩数据，且不添加 zlib 头信息。

## Class: zlib.Gunzip

使用 gzip 算法解压数据。

## Class: zlib.Gzip

使用 gzip 算法压缩数据。

## Class: zlib.Inflate

使用 inflate 算法解压数据。

## Class: zlib.InflateRaw

使用 deflate 算法解压数据。

## Class: zlib.Unzip

通过自动检测头信息解压 gzip 或 deflate 数据。

## Class: zlib.Zlib

`zilb` 模块并不会输出该类，之所以在这里列出，是因为它是前天解压缩类的基类。

#### zlib.flush([kind], callback)

`kind` 的默认值为 `zlib.Z_FULL_FLUSH`。

该方法用于输入缓冲数据，请谨慎使用该方法，因为过早的刷新会严重影响压缩算法的效率。

#### zlib.params(level, strategy, callback)

该方法用于动态更新压缩级别和压缩策略，且只对 deflate 算法有效。

#### zlib.reset()

该方法用于重置解压缩算法，且只对 inflate/defalte 算法有效。

## zlib.createDeflate(options)

该方法根据指定的 `options` 返回一个新的 Defalte 对象。

## zlib.createDeflateRaw(options)

该方法根据指定的 `options` 返回一个新的 DeflateRaw 对象。

## zlib.createGunzip(options)

该方法根据指定的 `options` 返回一个新的 Gunzip 对象。

## zlib.createGzip(options)

该方法根据指定的 `options` 返回一个新的 Gzip 对象。

## zlib.createInflate(options)

该方法根据指定的 `options` 返回一个新的 Inflate 对象。

## zlib.createInflateRaw(options)

该方法根据指定的 `options` 返回一个新的 InfalteRaw 对象。

## zlib.createUnzip(options)

该方法根据指定的 `options` 返回一个新的 Unzip 对象。

## 简便方法

以下所有方法的第一个参数都是 Buffer 实例或字符串，第二个参数是一个可选的 `options`，第三个参数是一个回调函数，其形式为 `callback(error, result)`。

每一个方法都有一个 `*Sync` 版本，接收相同的参数，但不接收回调函数。

#### zlib.deflate(buf[, options], callback)
#### zlib.deflateSync(buf[, options])

该方法使用 Defalte 压缩 Buffer 实例或字符串。

#### zlib.deflateRaw(buf[, options], callback)
#### zlib.deflateRawSync(buf[, options])

该方法使用 DefalteRaw 压缩 Buffer 实例或字符串。

#### zlib.gunzip(buf[, options], callback)
#### zlib.gunzipSync(buf[, options])

该方法使用 Gunzip 解压 Buffer 实例或字符串。

#### zlib.gzip(buf[, options], callback)
#### zlib.gzipSync(buf[, options])

该方法使用 Gzip 解压 Buffer 实例或字符串。

#### zlib.inflate(buf[, options], callback)
#### zlib.inflateSync(buf[, options])

该方法使用 Infalte 解压 Buffer 实例或字符串。

#### zlib.inflateRaw(buf[, options], callback)
#### zlib.inflateRawSync(buf[, options])

该方法使用 InflateRaw 解压 Buffer 实例或字符串。

#### zlib.unzip(buf[, options], callback)
#### zlib.unzipSync(buf[, options])

该方法使用 Unzip 解压 Buffer 实例或字符串。

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