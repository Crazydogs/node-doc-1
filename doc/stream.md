# Stream

<div class="s s2"></div>

stream 是一个在 Node.js 上使用流数据的抽象接口，提供了一些基础的 API，
方便我们基于它实现流式的接口。

Node.js 提供了很多原生的流对象，如 HTTP 服务的 request，process 模块的 stdout 等。

流可以是可读的，可写的或者是同时可读写的。所有的流都是 EventEmitter 实例。

可以通过下面这种方式加载 `stream` 模块

```js
    const stream = require('stream');
```

尽管所有的 Node.js 用户都必须弄明白 stream 是如何工作的，但 stream 模块本身只对正在创建新类型流的开发人员最有用。
其他开发者很少须要直接使用 stream 模块。

## 本文档的结构

本文档分为两个主要部分，和一个[附加注释](#附加注释)部分。
- 第一部分介绍了开发者需要在开发中[使用 steam 所涉及的 API](#使用流涉及的-api)。
- 第二部分介绍了开发者[创建自定义 stream 所需要的 API](#创建自定义-stream-所需要的-api)。

## Stream 的类型

在 Node.js 中 stream 一共有 4 种基本类型

- [Readable](#streamreadable-类) 可以读取数据的流(如 fs.createReadStream())
- [Writable](#streamwritable-类) 可以写入数据的流(如 fs.createWriteStream())
- [Duplex](#streamduplex-类) 同时可读又可写的流(如 net.Socket())
- [Transform](#streamtransform-类) 在写入和读取过程中对数据进行修改变换的 Duplex
流(如 zlib.createDeflate())

### 对象模式

通过 Node API 创建的流，只能够对字符串或者 buffer 对象进行操作。但其实流的实现是可以基于其他的
Javascript 类型(除了 null, 它在流中有特殊的含义)的。这样的流就处在 "对象模式" 中。

在创建流对象的时候，可以通过提供 objectMode 参数来生成对象模式的流。
试图将现有的流转换为对象模式是不安全的。

### 缓冲区

[Readable](#streamreadable-类) 和 [Writable](#streamwritable-类) 流都会将数据储存在内部的缓冲区中。
缓冲区可以分别通过 `writable._writableState.getBuffer()` 和 `readable._readableState.buffer` 来访问。

缓冲区中能容纳的数据数量由 stream 构造函数的 `highWaterMark` 选项决定。对于普通的流来说，
`highWaterMark` 选项表示总共可容纳的比特数。对于对象模式的流，该参数表示可以容纳的对象个数。

当一个可读实例调用 [stream.push()](#readablepushchunk-encoding) 方法的时候，
数据将会被推入缓冲区。如果没有数据的消费者出现，调用 [stream.read()](#readable_readsize)
方法的话，数据就会一直留在缓冲队列中。

如果可读实例内部的缓冲区大小达到了创建时由 `highWaterMark` 指定的阈值，
可读流就会暂时停止从底层资源汲取数据，直到当前缓冲的数据成功被消耗掉
(也就是说，流停止调用内部用来填充缓冲区的 readable._read() 方法)。

在一个在可写实例上调用 [writable.write(chunk)](#writablewritechunk-encoding-callback)
方法的时候，数据会写入可写流的缓冲区。如果缓冲区的数据量低于 `highWaterMark` 设定的值，
调用 `writable.write()` 方法会返回 `true`，否则 write 方法会返回 `false`。

stream 模块的 API，特别是 [stream.pipe()](#readablepipedestination-options)，
最主要的目的就是将数据的流动缓冲到一个可接受的水平，不让不同速度的数据源之间的差异导致内存被占满。

[Duplex](#streamduplex-类) 流和 [Transform](#streamtransform-类) 流都是同时可读写的，
所以他们会在内部维持两个缓冲区，分别用于读取和写入，这样就可以允许两边同时独立操作，
维持高效的数据流。比如说 `net.Socket` 是一个 [Duplex](#streamduplex-类) 流，Readable 端允许从 socket 获取、
消耗数据，Writable 端允许向 socket 写入数据。数据写入的速度很有可能与消耗的速度有差距，
所以两端可以独立操作和缓冲是很重要的。

#使用流涉及的 API

几乎所有的 Node.js 应用，无论多简单，多多少少都会以某种方式用到流。下面是一个在
实现 Http 服务的 Node 应用中对流的使用。

```js
    const http = require('http');

    const server = http.createServer( (req, res) => {
      // req is an http.IncomingMessage, which is a Readable Stream
      // res is an http.ServerResponse, which is a Writable Stream

      let body = '';
      // Get the data as utf8 strings.
      // If an encoding is not set, Buffer objects will be received.
      req.setEncoding('utf8');

      // Readable streams emit 'data' events once a listener is added
      req.on('data', (chunk) => {
        body += chunk;
      });

      // the end event indicates that the entire body has been received
      req.on('end', () => {
        try {
          const data = JSON.parse(body);
          // write back something interesting to the user:
          res.write(typeof data);
          res.end();
        } catch (er) {
          // uh oh!  bad json!
          res.statusCode = 400;
          return res.end(`error: ${er.message}`);
        }
      });
    });

    server.listen(1337);

    // $ curl localhost:1337 -d '{}'
    // object
    // $ curl localhost:1337 -d '"foo"'
    // string
    // $ curl localhost:1337 -d 'not json'
    // error: Unexpected token o
```

Writable 流(如上例中的 res)暴露了 `write()` 和 `end()` 这样的接口，用于向流中写入数据。

Readable 流使用 EventEmitter 的 API 来通知应用，流中有可读取的数据了。
有多种方式可以获取这些数据。

Readable 流和 Writable 流都以各种方法使用 EventEmitter API 来传达流的当前状态。

[Duplex](#streamduplex-类) 流和 [Transform](#streamtransform-类) 流都同时是
[Readable](#streamreadable-类) 流与 [Writable](#streamwritable-类) 流。

向流中写入数据或者消耗数据的应用并不需要直接实现流的接口，而且通常并不需要调用
`require(stream)`

想要创造新的 stream 类型的开发者请参考本文档的[自定义 stream 所需要的 API]
(#创建自定义-stream-所需要的-api)


## Writable Streams

Writable Streams，可写流，是对数据写入的目标的一种抽象。

常见的可写流包括：

- 在客户端的 HTTP request
- 在服务端的 HTTP responses
- 可写文件流(fs.createWriteStream)
- zlib 流
- crypto 流
- TCP sockets
- child process stdin
- process.stdout, process.stderr

注意：上面列举的部分列子实际上是双工 stream，即同时实现了可读流的接口

所有的可写流都实现了 `stream.Writable` 类定义的接口。

虽然各种可写流在不同的方面有所区别，但所有的可写流都遵循下面这样的基本使用方式。

```js
    const myStream = getWritableStreamSomehow();
    myStream.write('some data');
    myStream.write('some more data');
    myStream.end('done writing data');
```

### stream.Writable 类
添加于 v0.9.4

#### close 事件
添加于 v0.9.4

当流或者流的基础资源(比如文件描述符)被关闭，就会触发 `close` 事件。该事件表示，
没有其他事件会触发，也不会再进行更多的计算。

并不是所有的可写流都会触发 `close` 事件

#### drain 事件
添加于 v0.9.4

如果调用 [stream.write(chunk)](#writablewritechunk-encoding-callback) 
的时候返回了 `false`，那么当可以重新开始向流中写入数据的时候，就会触发 `drain` 事件。

```js
    // Write the data to the supplied writable stream one million times.
    // Be attentive to back-pressure.
    function writeOneMillionTimes(writer, data, encoding, callback) {
      let i = 1000000;
      write();
      function write() {
        var ok = true;
        do {
          i--;
          if (i === 0) {
            // last time!
            writer.write(data, encoding, callback);
          } else {
            // see if we should continue, or wait
            // don't pass the callback, because we're not done yet.
            ok = writer.write(data, encoding);
          }
        } while (i > 0 && ok);
        if (i > 0) {
          // had to stop early!
          // write some more once it drains
          writer.once('drain', write);
        }
      }
    }
```

#### error 事件
添加于 v0.9.4

- 参数 \<Error\>

`error` 事件会在写入/传输数据发生错误的时候触发。事件的回调函数在调用的时候，
会接受一个 `Error` 参数。

#### finish 事件
添加于 v0.9.4

在执行 [stream.end()](#writableendchunk-encoding-callback) 之后，会触发 `finish`
事件，此时所有数据都应该已经写入底层。

```js
    const writer = getWritableStreamSomehow();
    for (var i = 0; i < 100; i ++) {
      writer.write(`hello, #${i}!\n`);
    }
    writer.end('This is the end\n');
    writer.on('finish', () => {
      console.error('All writes are now complete.');
    });
```

#### pipe 事件
添加于 v0.9.4

- 参数 `src` \<[stream.Readable](#streamreadable-类)\> 流向此可写流的源

`pipe` 事件会在一个可读流调用 [stream.pipe()](#readablepipedestination-options)，
将一个可写流添加到他的目标集合的时候触发。

```js
    const writer = getWritableStreamSomehow();
    const reader = getReadableStreamSomehow();
    writer.on('pipe', (src) => {
      console.error('something is piping into the writer');
      assert.equal(src, reader);
    });
    reader.pipe(writer);
```

#### unpipe 事件
添加于 v0.9.4

- 参数 `src` <[Readable](#streamreadable-类) Stream> 停止流向此可写流的源

`pipe` 事件会在一个可读流调用 [stream.unpipe()](#readableunpipedestination)，
将一个可写流从他的目标集合移除的时候触发。

```js
    const writer = getWritableStreamSomehow();
    const reader = getReadableStreamSomehow();
    writer.on('unpipe', (src) => {
      console.error('Something has stopped piping into the writer.');
      assert.equal(src, reader);
    });
    reader.pipe(writer);
    reader.unpipe(writer);
```

#### writable.cork()
添加于 v0.11.2
`writable.cork()` 方法强制让已接受到的数据留在内存中，直到调用 `writable.uncork()`
或者 `writable.end()` 时才开始将数据写入目标。

`writable.cork()` 的主要功能是避免很多小数据块依次写入流中的情况，就不用在内部缓冲区生成备份，
以至于影响性能。在这种情况下，实现了 `writable._writev()`
方法的实例可以用更好的方式执行写入操作。

#### writable.end([chunk][, encoding][, callback])
添加于 v0.9.4

- 参数 `chunk` \<String\> | \<Buffer\> | \<any\> 可选的将要写入的数据，
对于处于正常模式的流来说，chunk 必须是字符串或者 Buffer 对象，而对于处于对象模式的流，
chunk 可以是除了 null 之外的任意 JavaScript 值
- 参数 `encoding` \<String\> 如果 chunk 是字符串，此参数为其编码
- 参数 `callback` \<Function\> 可选的回调函数，当流完成时调用。

调用 `writable.end()` 方法表示流将不会再有新的数据写入。可选的 chunk 和 encoding
参数允许在流关闭之前写入最后一块数据。如果提供了可选的 callback 参数，这个函数会成为
[finish](#finish-事件) 事件的一个监听器。

在调用 writable.end() 之后调用 writable.write() 会抛出一个错误。

```js
    // write 'hello, ' and then end with 'world!'
    const file = fs.createWriteStream('example.txt');
    file.write('hello, ');
    file.end('world!');
    // writing more now is not allowed!
```

#### writable.setDefaultEncoding(encoding)
添加于 v0.11.15

- 参数 `encoding` \<String\> 新的默认编码
- 返回值 `this`

`writable.setDefaultEncoding()` 方法用于设置[可写流](#streamwritable-类)的默认编码

#### writable.uncork()
添加于 v0.11.2

`writable.uncork()` 方法用于将被 `writable.cork()` 锁定在缓冲区中的数据释放。

当使用 `writable.cork()` 和 `writable.uncork()` 方法来管理可写流的缓冲区时，建议使用
`process.nextTick()` 方法来调用 `writable.uncork()`。这样做允许在一个 Node.js
循环批量处理所有 `writable.write()` 调用。

```js
    stream.cork();
    stream.write('some ');
    stream.write('data ');
    process.nextTick(() => stream.uncork());
```

如果 `writable.cork()` 方法在同一个流上连续调用了多次，那么必须调用同样次数的
`writable.uncork()` 方法才能将缓冲区的数据刷新到底层。

```js
    stream.cork();
    stream.write('some ');
    stream.cork();
    stream.write('data ');
    process.nextTick(() => {
      stream.uncork();
      // The data will not be flushed until uncork() is called a second time.
      stream.uncork();
    });
```

#### writable.write(chunk[, encoding][, callback])

- 参数 chunk \<String\> | \<Buffer\> 待写入的数据
- 参数 encoding \<String\> 如果 chunk 是字符串，则此参数为其编码
- 参数 callback \<Function\> 当数据块被刷新到底层的时候触发
- 返回值 \<Boolean\> 如果流想要等 `drain` 事件触发后，再写入新的数据，则返回 `false`，
否则返回 `true`

`writable.write()` 方法将数据写入流，并在数据完成处理之后调用提供的回调函数。
如果发生了错误，回调函数可能可以接收到一个 error 作为其第一个参数。如果想要确保监听到错误，
请为 `error` 事件添加监听器。

返回值表示写入的块是否已经在内部缓冲区，并且缓冲区已经达到了创建流时设置的
`highWaterMark`。如果返回了 `false`，应该停止继续尝试向流写入数据，直到流发射了
`drain` 事件。

处于对象模式的可写流会忽略 `encoding` 参数。

## Readable Streams

可读流是对数据来源的一种抽象。

常见的可读流包括：

- 在客户端的 HTTP 返回
- 在服务端的 HTTP 请求
- 可读文件流(fs.createReadStream)
- zlib 流
- crypto 流
- TCP sockets
- child process stdout 和 stderr
- process.stdin

所有的可读流都实现了 `stream.Readable` 定义的接口。

### 两种模式

可读流可以工作在两种模式，流模式和暂停模式。

在流模式下，流会从底层系统尽快地读取数据，然后通过 `EventEmitter` 的事件接口提供给应用。

在暂停模式下，必须显示的调用 [stream.read](#readablereadsize) 从流中读取数据。

处在暂停模式的流可以通过下面的方式切换到流模式

- 给流的 [data](#data-事件) 事件添加监听器
- 调用流的 [stream.resume()](#readableresume) 方法
- 调用 [stream.pipe()](#readablepipedestination-options) 方法将数据送入一个可写流

还可以通过下面的方式切回暂停模式

- 流没有 pipe 的目标时调用 [stream.pause()](#readablepause) 方法
- 流如果有 pipe 的目标，须要移除所有 [data](#data-事件) 事件的监听器，并通过
[stream.unpipe()](#readableunpipedestination) 方法移除所有的 pipe 目标。

一个重要的概念是，如果没有消耗或忽略数据的机制的话，可读流将不会生成数据。
如果数据的消费者失效了或者被移除，可读流将会尝试停止生成数据。

注意：由于向后兼容的原因，移除 [data](#data-事件) 事件监听器并不会将流暂停。另外，如果还有
pipe 的目标的话，调用 [stream.pause()](#readablepause) 方法并不能保证当 pipe 目标索取数据时，
流还能保持在暂停状态。

注意：如果可读流切换到流模式，但却没有数据的消费者接收数据，则数据会流失。
如在没有监听 `data` 事件就调用 `readable.resume()` 方法，或者从一个可读流移除
`data` 事件监听器，就会发生这种情况。

### 三种状态

可读流的两种操作模式，是对其内部更复杂的状态管理的简化抽象。

具体来说就是，在任意时间点，可读流都处于下面三种状态中的一种：

- `readable._readableState.flowing = null`
- `readable._readableState.flowing = false`
- `readable._readableState.flowing = true`

当 `readable._readableState.flowing` 为 `null` 时，没有提供消耗可读流数据的机制，
因此流并不会生成数据。

为 `data` 事件添加监听器，或者调用 `readable.pipe()` 方法，又或者调用 `readable.resume()`
方法，将会把流的 `readable._readableState.flowing` 切换为 `true`，此时，可读流开始生成数据，
并在数据生成时主动发射事件。

调用 `readable.pause()`，`readable.unpipe()` 或者接收 `back pressure` 事件会将流的
`readable._readableState.flowing` 设置为 `false`，暂时停止事件的流动，
但并不会停止数据生成。

当 `readable._readableState.flowing` 为 `false` 时，数据可能会在流的内部缓冲区累积。

### 选择一种方法

可读流API在多个 Node.js 版本中不断改进，并提供了多种消耗流数据的方法。一般来说，
开发者应该选择使用其中一种，而不应该使用多个方法来从单个流中消耗数据。

建议对大多数用户使用 `readable.pipe()` 方法，它提供了最简单的消耗流数据的方法。 
需要对数据传输和生成进行更细粒度控制的开发者可以使用 `EventEmitter` 和
`readable.pause() / readable.resume()` API。

### stream.Readable 类
添加于 v0.9.4

#### close 事件
添加于 v0.9.4

当流或者流依赖的底层资源(如文件描述符)关闭时，会发射 `close` 事件。该事件表示，
不会再有新的事件触发，也不会有进一步的计算。

并不是所有的可读流都会发射 `close` 事件。

#### data 事件
添加于 v0.9.4

- 参数 chunk \<Buffer\> | \<String\> | \<any\> 数据块。对于处在对象模式的流，chunk
可以是除了 `null` 之外任意的 JavaScript 值，否则 chunk 只能是字符串或者 `Buffer`
对象。

`data` 事件会在流放弃对一个数据块的所有权时触发。可读流通过调用 `readable.pipe()`
或 `readable.resume()` 或添加 `data` 监听器切换到流模式来放弃数据的所有权。调用
`readable.read()`，释放出一个数据块的时候，也会触发 `data` 事件。

给一个没有显式暂停的流添加 `data` 事件监听将会把流切换到流模式。
一旦数据可用了就会开始传递。

如果使用 `readable.setEncoding()` 方法为流指定了默认编码，回调函数会被传入字符串作为参数，
否则就会传入 `Buffer` 对象。

```js
    const readable = getReadableStreamSomehow();
    readable.on('data', (chunk) => {
      console.log(`Received ${chunk.length} bytes of data.`);
    });
```

#### end 事件
添加于 v0.9.4

`end` 事件会在可读流已经没有更多数据提供的时候触发。

注意：除非数据被完全消耗，否则 `end` 事件是不会触发的。可以通过切换到流模式，
或者不停调用 [stream.read()](#readablereadsize) 方法耗尽数据使其触发。

```js
    const readable = getReadableStreamSomehow();
    readable.on('data', (chunk) => {
      console.log(`Received ${chunk.length} bytes of data.`);
    });
    readable.on('end', () => {
      console.log('There will be no more data.');
    });
```

#### error 事件
添加于 v0.9.4

- 参数 \<Error\>

`error` 事件可以在任意时间触发。通常是由于底层内部错误或者流尝试推送无效数据块导致的。

监听器回调函数将会传入一个 `Error` 对象作为参数。

#### readable 事件
添加于 v0.9.4

当流有可用的数据可以被获取时，会触发 `readable` 事件。某些情况下，给 `readable`
事件添加监听器会导致一些数据被读入内部的缓冲区。

```js
    const readable = getReadableStreamSomehow();
    readable.on('readable', () => {
      // there is some data to read now
    });
```

在到达流数据结尾但还没发出 `end` 事件之前，会先触发 `readable` 事件。

`readable` 事件表示了流有新的信息，可能是新的数据可用，或者到达流数据的末尾。
如果是前者，调用 [stream.read()](#readablereadsize) 将会返回可用的数据，如果是后者，
[stream.read()](#readablereadsize) 会返回 `null`。在下面的示例中， `foo.txt` 是一个空文件：

```js
    const fs = require('fs');
    const rr = fs.createReadStream('foo.txt');
    rr.on('readable', () => {
      console.log('readable:', rr.read());
    });
    rr.on('end', () => {
      console.log('end');
    });
```

运行代码会输出：

```
    $ node test.js
    readable: null
    end
```

通常来说，使用 `readable.pipe()` 或者 `data` 事件优于使用 `readable` 事件。

#### readable.isPaused()

- 返回值 \<Boolean\>

`readable.isPaused()` 方法返回当前可读流所处的状态。此方法主要是被 `readable.pipe()`
方法的底层机制调用，一般来说是没有理由直接使用此方法的。

```js
    const readable = new stream.Readable

    readable.isPaused() // === false
    readable.pause()
    readable.isPaused() // === true
    readable.resume()
    readable.isPaused() // === false
```

#### readable.pause()
添加于 v0.9.4

- 返回值 `this`

`readable.pause()` 方法会让处于流模式的可读流停止发射 [data](#data-事件) 事件，并退出流模式。
新的可用数据将会留在内部的缓冲区中

```js
    const readable = getReadableStreamSomehow();
    readable.on('data', (chunk) => {
      console.log(`Received ${chunk.length} bytes of data.`);
      readable.pause();
      console.log('There will be no additional data for 1 second.');
      setTimeout(() => {
        console.log('Now data will start flowing again.');
        readable.resume();
      }, 1000);
    });
```

#### readable.pipe(destination[, options])
添加于 v0.9.4

- 参数 `destination` \<[stream.Writable](#streamwritable-类)\> 写入数据的目标
- 参数 `options` \<Object\> pip 操作的参数
    - `end` \<Boolean\> 是否在可读流结束的时候关闭可写流，默认为 `true`

`readable.pipe()` 方法将一个可写流附到可读流上，同时将可写流切换到流模式，
并把所有数据推给可写流。数据流会被自动管理，所以不用担心可写流被快速的可读流打满溢出。

下面这个例子中，可读流讲所有数据写到 `file.txt` 文件中。

```js
    const readable = getReadableStreamSomehow();
    const writable = fs.createWriteStream('file.txt');
    // All the data from readable goes into 'file.txt'
    readable.pipe(writable);
```

可以将多个可写流附加到单个可读流。

`readable.pipe()` 方法返回值是对 pipe 目标的引用，以便使用链式调用：

```js
    const r = fs.createReadStream('file.txt');
    const z = zlib.createGzip();
    const w = fs.createWriteStream('file.txt.gz');
    r.pipe(z).pipe(w);
```

默认情况下，当可读源发射 `end` 事件的时候，目标可写流会自动调用
[stream.end()](#writableendchunk-encoding-callback) 方法，导致可写流不能再写入。
如果想阻止此默认行为，须要将 `end` 选项设置为 false，让目标可写流保持打开状态，
像下面的例子：

```js
    der.pipe(writer, { end: false });
    reader.on('end', () => {
      writer.end('Goodbye\n');
    });
```

一个重要的警告是，如果可读流抛出错误，目标可写流并不会自动关闭。如果发生错误，
则必须手动关闭每一个流，以防止内存泄露。

注意：prcess.stderr 和 process.stdout 可写流在 Node.js 进程退出之前从不关闭，
不管传入什么选项都会被忽视。

#### readable.read([size])
添加于 v0.9.4

- 参数 size \<Number\> 可选参数，指定要读取的数据量。
- 返回值 \<String\> | \<Buffer\> | \<Null\>

`readable.read()` 方法从内部缓冲区抓取并返回数据。如果没有可用数据，则返回 `null`。
数据默认以 `Buffer` 对象返回，除非使用 `readable.setEncoding()` 方法设定了编码，
或者流在对象模式下运行。

`size` 参数指定了要读取的字节数。如果没有那么多字节的数据可用，除非已经到了数据的末尾，
否则将会返回 `null`。如果到达了流末尾，将返回保留在内部缓冲区的所有数据
(即使这些数据已经超过了指定字节数)。

如果未指定 `size` 参数，此方法将返回包含在内部缓冲区中的所有数据。

只有在暂停模式下的留才可以调用 `readable.read()` 方法。在流模式下，`readable.read()`
会自动被调用，直到内部缓冲区耗尽。

```js
    const readable = getReadableStreamSomehow();
    readable.on('readable', () => {
      var chunk;
      while (null !== (chunk = readable.read())) {
        console.log(`Received ${chunk.length} bytes of data.`);
      }
    });
```

一般来说，建议开发者避免使用 `readable` 和 `readable.read()` 方法来支持 `readable.pipe()`
或 `data` 事件的使用。

对象模式下的流调用 `read.read(size)` 总是返回单个对象，无视 `size` 参数的值。

注意：如果 `readable.read()` 方法返回了一个数据块，还会触发一个 `data` 事件。

注意：在 `end` 事件触发后调用 [stream.read([size])](#readablereadsize)
方法将返回 `null`，不会产生错误。

#### readable.resume()
添加于 v0.9.4

- 返回值 `this`

`readable.resume()` 方法让一个显式暂停的流重新开始发射 [data](#data-事件) 事件，切换到流模式。

`readable.resume()` 方法可以用于完全消耗掉数据流，而实际上并不对数据做任何处理，
可参考下面的例子：

```js
    getReadableStreamSomehow()
      .resume()
      .on('end', () => {
        console.log('Reached the end, but did not read anything.');
      });
```

#### readable.setEncoding(encoding)
添加于 v0.9.4

- 参数 `encoding` \<String\> 要使用的编码
- 返回值 `this`

`readable.setEncoding()` 方法可以设置从可读流读取的数据的默认字符编码。

设置编码会让流数据以字符串的形式传递而不是默认的 `Buffer` 对象。例如，调用
`readable.setEncoding('utf8')` 会让输出的数据按 UTF-8 解析，并以字符串传递。
调用 `readable.setEncoding('hex')` 会让数据按照十六进制格式编码。

可读流可以正确的处理流中的多字节字符，如果只是简单的拉取 `Buffer` 对象，
会导致多字节字符不适当地解码。

可以使用 `readable.setEncoding(null)` 来禁用编码。这在处理二进制数据，
或分布在多个块上的大型多字节字符时很有效。

```js
    const readable = getReadableStreamSomehow();
    readable.setEncoding('utf8');
    readable.on('data', (chunk) => {
      assert.equal(typeof chunk, 'string');
      console.log('got %d characters of string data', chunk.length);
    });
```

#### readable.unpipe([destination])
添加于 v0.9.4

- 参数 `destination` \<[stream.Writable](#streamwritable-类)\> 可选参数，指定要解绑的流。

`readable.unpipe()` 方法会移除之前使用 [stream.pipe()](#readablepipedestination-options)
附加的可写流。

如果没有指定 `destination` 参数，则会移除附加的所有可写流。

如果指定了 `destination` 参数，但指定的目标并没有附加在可读流上，则此方法不做任何操作。

```js
    const readable = getReadableStreamSomehow();
    const writable = fs.createWriteStream('file.txt');
    // All the data from readable goes into 'file.txt',
    // but only for the first second
    readable.pipe(writable);
    setTimeout(() => {
      console.log('Stop writing to file.txt');
      readable.unpipe(writable);
      console.log('Manually close the file stream');
      writable.end();
    }, 1000);
```

#### readable.unshift(chunk)
添加于 v0.9.11

- 参数 `chunk` \<Buffer\> | \<String\> 要退回可读流的数据

`readable.unshift()` 方法将一个数据块推回内部缓冲区。此方法主要用于数据被不应该消耗数据的代码消耗了，
须要撤销这次数据消耗的情况，退回缓冲区让数据可以传到其他正确的地方去。

注意：`stream.unshift(chunk)` 方法无法在 `end` 事件触发或者抛出运行错误之后调用。

使用 `stream.unshift()` 的开发者通常应该考虑使用 [Transform](#streamtransform-类) 
流代替。更多信息，可以参考本文档第二部分
[开发者创建自定义 stream 所需要的 API](#创建自定义-stream-所需要的-api)。

```js
    // Pull off a header delimited by \n\n
    // use unshift() if we get too much
    // Call the callback with (error, header, stream)
    const StringDecoder = require('string_decoder').StringDecoder;
    function parseHeader(stream, callback) {
      stream.on('error', callback);
      stream.on('readable', onReadable);
      const decoder = new StringDecoder('utf8');
      var header = '';
      function onReadable() {
        var chunk;
        while (null !== (chunk = stream.read())) {
          var str = decoder.write(chunk);
          if (str.match(/\n\n/)) {
            // found the header boundary
            var split = str.split(/\n\n/);
            header += split.shift();
            const remaining = split.join('\n\n');
            const buf = Buffer.from(remaining, 'utf8');
            stream.removeListener('error', callback);
            // set the readable listener before unshifting
            stream.removeListener('readable', onReadable);
            if (buf.length)
              stream.unshift(buf);
            // now the body of the message can be read from the stream.
            callback(null, header, stream);
          } else {
            // still reading the header.
            header += str;
          }
        }
      }
    }
```

注意：`stream.unshift(chunk)` 与 [stream.push(chunk)](#readablepushchunk-encoding)
方法不同，并不会改变流的内部状态来结束读取过程。在读取数据期间(比如在一个自定义流的
[stream._read()](#readable_readsize) 中)使用 `stream.unshift()` 方法可能会导致意外的结果。
在调用 `readable.unshift()` 后立即调用 [stream.push(chunk)](#readablepushchunk-encoding)
方法会正确的设置流的内部状态。但最好还是不要在读取过程中调用此方法。

#### readable.wrap(stream)
添加于 v0.9.4

- 参数 `stream` <Stream> 旧的可读流

在 0.10 版本之前的 Node.js 中，并没有实现当前定义的流模块的 API。(详见兼容性部分)

在使用旧版的 Node.js 库，使用只有 `stream.pause()` 方法和 `data` 事件发射的流时，
可以用 `readable.wrap()` 方法将其包装成一个新的可读流。

很少会有使用 `readable.wrap()` 方法的时候，它主要是方便与较早的 Node.js 应用和库交互的。

例如：

```js
    const OldReader = require('./old-api-module.js').OldReader;
    const Readable = require('stream').Readable;
    const oreader = new OldReader;
    const myReader = new Readable().wrap(oreader);

    myReader.on('readable', () => {
      myReader.read(); // etc.
    });
```

## Duplex and Transform Streams

### stream.Duplex 类
添加于 v0.9.4

双工流(Duplex stream) 是同时实现了可读和可写接口的流。

常见的双工流包括：

- TCP sockets
- zlib streams
- crypto streams

### stream.Transform 类
添加于 v0.9.4

Transform 流是输出以某种方式依赖于输入的双工流。作为双工流，他也实现了可读于可写的接口。

常见的 Transform 流包括：

- zlib streams
- crypto streams

# 创建自定义 stream 所需要的 API

stream 模块的 API 设计使其可以用 JavaScript 的原型继承简单地实现自定义的流。

首先，开发者须要声明一个新的 JavaScript 类，扩展自四个基本类之一(`stream.Writable`,
`stream.Readable`, `stream.Duplex`, `stream.Transform`)，并确保调用了父类的构造函数：

```js
    const Writable = require('stream').Writable;

    class MyWritable extends Writable {
      constructor(options) {
        super(options);
      }
    }
```

这个新的类须要实现一个或多个特定的方法，具体取决于要创建流的类型，详见下表：

| 用途                | 类               | 实现的方法             |
| -------------------|------------------|-----------------------|
| 只读                |Readable          |_read                  |
| 只写                |Writable          |_write, _writev        |
| 读写                |Duplex            |_read, _write, _writev |
| 写数据读结果         |Transform         |_transform, _flush     |

注意：实现中请不要调用流模块的"公用"方法(在 `使用流涉及的 API` 部分中讲到的)。
这样做可能会导致消耗流数据是产生不良的副作用。

## 构造简单的流

对于很多简单的情况，可以不通过继承，直接通过传递适当的选项调用构造函数来创建
`stream.Writable`, `stream.Readable`, `stream.Duplex`, `stream.Transform` 实例。

例如：

```js
    const Writable = require('stream').Writable;

    const myWritable = new Writable({
      write(chunk, encoding, callback) {
        // ...
      }
    });
```

## 实现一个可写流

可以通过继承扩展 `stream.Writable` 类来实现一个可写流。

自定义流须必须调用 `new stream.Writable([options])` 构造函数并实现 `writable._write()`
方法，也可以实现可选的 `writable._writev()` 方法。

### 构造函数 new stream.Writable([options])

- 参数 `options` <Object>
    - `highWaterMark` <Number> 指定缓存数量达到什么水平的时候 `stream.write()`
    开始返回 `false`。默认是 `16384`(16kb) 或对象模式的流是 `16`(个对象)
    - `decodeStrings` <Boolean> 指定是否在将字符串数据传给 `_write()` 方法前解码，
    默认为 `true`
    - `objectMode` <Boolean> 决定调用 `stream.write(anyObj)` 是否合法。如果设置了此属性，
    则可以向流中写入除了字符串和 `Buffer` 对象之外的其他 JavaScript 值。默认为
    `false`
    - `write` <Function> 对 `stream._write()` 的实现
    - `writev` <Function> 对 `stream._writev()` 的实现

例如：

```js
    const Writable = require('stream').Writable;

    class MyWritable extends Writable {
      constructor(options) {
        // Calls the stream.Writable() constructor
        super(options);
      }
    }
```

或者使用 ES6 之前的风格构造：

```js
    const Writable = require('stream').Writable;
    const util = require('util');

    function MyWritable(options) {
      if (!(this instanceof MyWritable))
        return new MyWritable(options);
      Writable.call(this, options);
    }
    util.inherits(MyWritable, Writable);
```

或者使用简化构造方法

```js
    const Writable = require('stream').Writable;

    const myWritable = new Writable({
      write(chunk, encoding, callback) {
        // ...
      },
      writev(chunks, callback) {
        // ...
      }
    });
```

### writable._write(chunk, encoding, callback)

- 参数 `chunk` <Buffer> | <String> 将要被写入的数据。除非 `decodeStrings` 被设为
`false`，否则都会是 `Buffer` 对象
- 参数 `encoding` <String> 如果 chunk 为字符串，那此参数为字符串的编码，如果
chunk 是 `Buffer` 对象或者流处于对象模式，`encoding` 参数被忽略
- 参数 `callback` <Function> 在完成数据块的处理后须调用此函数(带可选的 error 参数)

所有的可写流都必须提供 `stream._write()` 的实现，用于将数据写入底层。

注意：Transform 流会提供自己的 `stream._write()` 函数实现。

注意：**此函数不应该被直接调用。**子类只是提供它的实现，它只应被可写流的内部方法调用。

`callback` 函数必须被调用，以通知对数据块的处理完成或是出现错误。如果发生错误，
则传入一个错误对象作为他的第一个参数，否则传入 `null` 即可。

须要注意的是，在调用 `writable._write()` 到 `callback` 函数被调用期间，调用
`writable.write()` 写入的数据将会被放入缓冲区。调用 `callback` 函数将会发射一个
`drain` 事件。如果流具备一次处理多个数据块的能力，则应该实现 `wratable._writev()`
方法。

如果设置了 `decodeStrings` 参数，那么 chunk 可能是一个字符串而不是一个 `Buffer`
对象，`encoding` 参数将表示其字符编码。这是为了支持一些针对特定编码进行优化的实现。
如果将 `decodeStrings` 显式设置为 `true`(此处存疑，原文为 false)，则可以安全地忽略
`encoding` 参数，chunk 将会是 `Buffer` 对象。

`writable._write()` 方法带有下划线前缀，说明它是一个内部方法，只提供给定义它的类内部使用，
开发者不应直接调用它。

### writable._writev(chunks, callback)

- 参数 `chunks` <Array> 将要被写入的数据块，每一块数据的格式会是这样：`{ chunk:
..., encoding: ...}`。
- 参数 `callback` <Function> 接受一个可选的错误参数的回调函数，须要在处理完数据块后调用

注意：**此函数不应该被直接调用。**子类只是提供它的实现，它只应被可写流的内部方法调用。

`writable._writev()` 函数是 `writeable.write()` 的补充，用于同时接收多个数据块，
并进行处理。如果实现了此方法, 将会用当前缓冲区内的所有数据作为参数来调用它。

`writable._writev()` 方法带有下划线前缀，说明它是一个内部方法，只提供给定义它的类内部使用，
开发者不应直接调用它。

### 可写流中的错误处理

建议通过向 `callback` 函数传入错误对象作为第一个参数，来处理 `writable._write()`
和 `writable._writev()` 过程中发生的错误。这样可写流就会抛出 `error` 事件。
在 `writable._write()` 中直接抛出错误可能会导致与预期不一致的行为，具体取决于如何使用流。
使用回调可确保一致和可预测的错误处理。

```js
    const Writable = require('stream').Writable;

    const myWritable = new Writable({
      write(chunk, encoding, callback) {
        if (chunk.toString().indexOf('a') >= 0) {
          callback(new Error('chunk is invalid'));
        } else {
          callback();
        }
      }
    });
```

### 一个可写流示例

下面展示一个非常简单(可以说没有意义)的自定义可写流实现。 虽然这个可写流实例没有任何实际的用处，
但它展示了自定义可写流实例的每个必需元素：

```js
    const Writable = require('stream').Writable;

    class MyWritable extends Writable {
      constructor(options) {
        super(options);
      }

      _write(chunk, encoding, callback) {
        if (chunk.toString().indexOf('a') >= 0) {
          callback(new Error('chunk is invalid'));
        } else {
          callback();
        }
      }
    }
```

## 实现一个可读流

可以通过继承扩展 `stream.Readable` 类来实现一个可写流。

自定义流须必须调用 `new stream.Readable([options])` 构造函数并实现 `writable._write()`
方法。

### 构造函数 new stream.Readable([options])

- 参数 `options` \<Object\>
    - `highWaterMark` <Number> 缓冲区最大容量，默认为 16384(16kb)，对于对象模式的流，
    默认为 16(个对象)。达到最大容量后停止从底层系统汲取数据。
    - `encoding` <String> 如果指定了此参数，数据将会按照此编码解码为字符串，
    默认值为 `null`
    - `objectMode` <Boolean> 流是否工作在流模式下。即调用 `stream.read(n)` 返回一个值，
    而不是返回指定长度的字符串或 `Buffer` 对象。
    - `read` <Function> 对 `stream._read()` 的实现

例如：
```js
    const Readable = require('stream').Readable;

    class MyReadable extends Readable {
      constructor(options) {
        // Calls the stream.Readable(options) constructor
        super(options);
      }
    }
```

或者使用 ES6 之前的风格构造：

```js
    const Readable = require('stream').Readable;
    const util = require('util');

    function MyReadable(options) {
      if (!(this instanceof MyReadable))
        return new MyReadable(options);
      Readable.call(this, options);
    }
    util.inherits(MyReadable, Readable);
```

或者使用简化构造方法

```js
    const Readable = require('stream').Readable;

    const myReadable = new Readable({
      read(size) {
        // ...
      }
    });
```

### readable._read(size)

- 参数 `Size` \<Number\> 要异步读取的数据量。

注意：**此函数不应该被直接调用。**子类只是提供它的实现，它只应被可读流的内部方法调用。

所有的可读流都必须提供自己的 `radable._read()` 方法实现，用于从底层系统汲取数据。

当 `readable._read()` 被调用时，如果底层数据可用，则应该通过 `this.push(dataChunk)`
方法将数据推入可读流的缓冲队列中。`_read()` 将持续的将数据汲取到缓冲队列中，
直到 `readable.push()` 返回 `false`。再次调用 `_read()` 将使其重新开始重复以上过程。

注意：`readable._read()` 被调用后，直到调用 `readable.push` 之前，都不会再次被调用。

`size` 参数并不是强制性的。对于读取操作是简单地返回数据的情况，可以用 `size`
参数来决定要提取多少数据。其他情况下可以直接忽略此参数，当有数据可用就直接返回。
没有必要等到可用数据达到 `size` 指定大小再去调用 `stream.push(chunk)`

`readable._read()` 方法以下划线为前缀，表明它属于定义它的类的内部，
并且不应该被用户程序直接调用。

### readable.push(chunk[, encoding])

- 参数 `chunk` \<Buffer\> | \<Null\> | \<String\> 将要推入读取队列的数据。
- 参数 `encoding` \<String\> chunk 的编码。必须是一个合法 Buffer 编码，如 `utf8`
或 `ascii` 等
- 返回值 \<Boolean\> 如果还需要继续推入更多数据则返回 `ture`，否则返回 `false`。

当 chunk 是 `Buffer` 对象或字符串时，数据块将被添加到内部缓冲区以供流的消费者使用。 
如果 chunk 是 `null` 则表示流的末尾(EOF)，之后不能写入更多的数据。

如果流处在暂停模式下，那么使用 `readable.push()` 添加的数据在 `readable` 事件触发后，
可以通过调用 `readable.read()` 方法获取。

如果流处在流模式下，`readable.push()` 添加的数据将通过发射一个 `data` 事件交付出去。

`readable.push()` 方法的设计非常灵活。例如，我们要使用一个底层数据源，
可以使用一个自定义可读流来包装它，以提供暂停/恢复功能和数据的回调机制。
如下所示

```js
    // source is an object with readStop() and readStart() methods,
    // and an `ondata` member that gets called when it has data, and
    // an `onend` member that gets called when the data is over.

    class SourceWrapper extends Readable {
      constructor(options) {
        super(options);

        this._source = getLowlevelSourceObject();

        // Every time there's data, push it into the internal buffer.
        this._source.ondata = (chunk) => {
          // if push() returns false, then stop reading from source
          if (!this.push(chunk))
            this._source.readStop();
        };

        // When the source ends, push the EOF-signaling `null` chunk
        this._source.onend = () => {
          this.push(null);
        };
      }
      // _read will be called when the stream wants to pull more data in
      // the advisory size argument is ignored in this case.
      _read(size) {
        this._source.readStart();
      }
    }
```

注意 `readable.push()` 方法只能在可读流实例中调用，并且只能在 `readable._read()`
中调用。

### 可读流中的错误处理

建议在 `readable._read()` 过程中发生的错误通过 `error` 事件抛出，而不是直接抛出。
从 readable._read() 内部直接抛出错误可能导致预期和不一致的行为，
具体取决于流是以流模式还是以暂停模式运行。 使用 `error` 事件可确保一致且可预测的错误处理。

```js
    const Readable = require('stream').Readable;

    const myReadable = new Readable({
      read(size) {
        if (checkSomeErrorCondition()) {
          process.nextTick(() => this.emit('error', err));
          return;
        }
        // do some work
      }
    });
```

### 一个可读流示例

以下是可读流的基本示例，其以升序从 1 到 1,000,000 输出数字，然后结束。

```js
    const Readable = require('stream').Readable;

    class Counter extends Readable {
      constructor(opt) {
        super(opt);
        this._max = 1000000;
        this._index = 1;
      }

      _read() {
        var i = this._index++;
        if (i > this._max)
          this.push(null);
        else {
          var str = '' + i;
          var buf = Buffer.from(str, 'ascii');
          this.push(buf);
        }
      }
    }
```
## 实现一个双工流

双工流是同时可读写的流，如 TCO socket 连接。

因为 JavaScript 不支持多重继承，所以要继承 `stream.Duplex` 类来实现双工流
(而不是同时继承 `steam.Writable` 类和 `stream.Readable` 类)。

注意：`stream.Duplex` 类原型继承自 `stream.Readable`，并寄生自 `stream.Writable`。
不过 `instanceof` 操作符对两个类都有效，因为在 `stream.Writable` 的 `Symbol.hasInstance`
进行了重写。

自定义双工流必须调用构造函数 `new stream.Duplex([options])`，并实现 `readable._read()`
和 `writable._write()` 方法。

### new stream.Duplex(options)

- 参数 `options` \<Object\> 传给可读与可写构造器的选项，拥有以下字段
    - `allowHalfOpen` \<Boolean\> 默认为 `true`。如果设置为 `false`，
    流将会在可写端关闭的时候自动关闭可读端，反之亦然。
    - `readableObjectMode` \<Boolean\> 默认为 `false`。设置可读端是否以对象模式运行，
    如果 `objectMode` 为 `true` 则没有效果
    - `writableObjectMode` \<Boolean\> 默认为 `false`。设置可写端是否以对象模式运行，
    如果 `objectMode` 为 `true` 则没有效果

例如：

```js
    const Duplex = require('stream').Duplex;

    class MyDuplex extends Duplex {
      constructor(options) {
        super(options);
      }
    }
```

或者使用 ES6 之前的风格构造：

```js
    const Duplex = require('stream').Duplex;
    const util = require('util');

    function MyDuplex(options) {
      if (!(this instanceof MyDuplex))
        return new MyDuplex(options);
      Duplex.call(this, options);
    }
    util.inherits(MyDuplex, Duplex);
```

或者使用简化构造方法

```js
    const Duplex = require('stream').Duplex;

    const myDuplex = new Duplex({
      read(size) {
        // ...
      },
      write(chunk, encoding, callback) {
        // ...
      }
    });
```

### 一个双工流示例

下面是一个简单的例子。假设我们有一个底层的数据源，可读可写，但与 Node.js 的流
API 不兼容。我们可以用一个双工流对它进行包装，用可写流的接口缓冲写入的数据，
并用可读流的接口获取数据。

```js
    const Duplex = require('stream').Duplex;
    const kSource = Symbol('source');

    class MyDuplex extends Duplex {
      constructor(source, options) {
        super(options);
        this[kSource] = source;
      }

      _write(chunk, encoding, callback) {
        // The underlying source only deals with strings
        if (Buffer.isBuffer(chunk))
          chunk = chunk.toString();
        this[kSource].writeSomeData(chunk);
        callback();
      }

      _read(size) {
        this[kSource].fetchSomeData(size, (data, encoding) => {
          this.push(Buffer.from(data, encoding));
        });
      }
    }
```

双工流很重要的一点是，尽管在同一个实例中，它的可读侧和可写侧是相互独立的。

### 双工流的对象模式

对于双工流，objectMode 可以使用 readableObjectMode 和 writableObjectMode
选项为可读端和可写端专门设置。

在下面这个例子中，创建了一个 Transform 流(一种双工流)，其可写端是对象模式，
接受一个数字，并将其按照十六进制转换为字符串从可读端输出。

```js
    const Transform = require('stream').Transform;

    // All Transform streams are also Duplex Streams
    const myTransform = new Transform({
      writableObjectMode: true,

      transform(chunk, encoding, callback) {
        // Coerce the chunk to a number if necessary
        chunk |= 0;

        // Transform the chunk into something else.
        const data = chunk.toString(16);

        // Push the data onto the readable queue.
        callback(null, '0'.repeat(data.length % 2) + data);
      }
    });

    myTransform.setEncoding('ascii');
    myTransform.on('data', (chunk) => console.log(chunk));

    myTransform.write(1);
      // Prints: 01
    myTransform.write(10);
      // Prints: 0a
    myTransform.write(100);
      // Prints: 64
```

## 实现一个 Transform 流

Transform 流是一种双工流，其输出是根据输入计算得到的。比如用于压缩数据的 zlib
流，加密解密数据的 crypto 流等。

注意：Transform 流并不要求输入与输出的大小相同，数据块数量相同，输入输出时间相同。
比如一个 hash 流只会在输入结束的时候输出一个数据块，zlib 流的输出会比其输入大得多或者小得多。

继承 `stream.Transform` 类以实现一个 Transform 流。

`stream.Transform` 原型继承于 `stream.Duplex`，并实现了自己的 `writable._write()`
方法和 `readable._read()` 方法。自定义的 Transform 流实例必须实现 `transform._transform()`
方法，可以实现可选的 `transform.flush()` 方法。

注意：使用 Transform 流的时候要小心，如果可读端数据没有被消耗，向可写端写入数据，
有可能导致可写端进入暂停状态。

### new stream.Transform([options])

- 参数 `options` \<Object\> 传给可读与可写构造器的选项，拥有以下字段
    - transform \<Function\> 对 `stream._transform` 的实现
    - flush \<Function\> 对 `stream._flush` 的实现

例如：

```js
    const Transform = require('stream').Transform;

    class MyTransform extends Transform {
      constructor(options) {
        super(options);
      }
    }
```

或者使用 ES6 之前的风格构造：

```js
    const Transform = require('stream').Transform;
    const util = require('util');

    function MyTransform(options) {
      if (!(this instanceof MyTransform))
        return new MyTransform(options);
      Transform.call(this, options);
    }
    util.inherits(MyTransform, Transform);
```

或者使用简化构造方法

```js
const Transform = require('stream').Transform;

const myTransform = new Transform({
  transform(chunk, encoding, callback) {
    // ...
  }
});
```

### `finish` 和 `end` 事件

`finish` 事件和 `end` 事件分别来自 `stream.Writable` 和 `stream.Readable` 类。
`fiinsh` 事件会在调用 `stream.end()` 之后触发。`end` 事件会在所有数据输出，
并在 `transform._flush()` 调用之后触发。

### transform._flush(callback)

- 参数 `callback` \<Function\> 在剩余的数据都被输出后调用的回调函数(可传递一个错误对象作为参数)

注意：**此函数不应该被直接调用。**子类只是提供它的实现，它只应被可读流的内部方法调用。

某些情况下，Transform 流可能须要在流的末端添加额外的数据。例如 zlib
流会储存一些用于优化压缩输出的内部状态。当流结束的时候，须要将这些状态也进行输出，
这样压缩数据才是完整的。

自定义 Transform 流实现可以实现 `transform._flush()` 方法。 当没有更多写入数据以供消耗时，
在发射 `end` 事件通知可读流结束之前，将调用该方法。

在 `transform._flush()` 实现中，`readable.push()` 方法可能不被调用或调用多次。
当数据操作完成时，必须调用 `callbak` 函数。

`transform._flush()` 方法以下划线为前缀，表明它属于定义它的类的内部，
并且不应该被用户程序直接调用。

### transform._transform(chunk, encoding, callback)

- 参数 `chunk` \<Buffer\> | \<String\> 要被转换的数据。除非 `decodeStrings`
选项设置为 `false`，否则将为 Buffer 对象。
- 参数 `encoding` \<String\> 如果 `chunk` 是字符串，此参数为其编码。如果是 Buffer
对象，此参数为一个特殊值 'buffer'，此情况下可以忽略它。
- 参数 `callback` \<Function\> 一个在数据块处理完成后要调用的回调函数
(可以接受一个错误对象作为参数)

注意：**此函数不应该被直接调用。**子类只是提供它的实现，它只应被可读流的内部方法调用。

所有Transform流实现必须提供一个 `_transform()` 方法来接受输入并产生输出。
`transform._transform()` 的实现须要接收写入的数据，计算输出，然后使用 `readable.push()`
方法将输出传递到可读部分。

对于单个数据块输入，`transform.push()` 方法可以被调用 0 次或多次，这取决于想要输出多少数据块。

可以接受输入但并不产生任何输出。

必须在当前数据块被完全处理后调用 `callback` 函数。如果处理过程中发生了错误，
`callback` 函数的第一个参数必须是一个 `Error` 对象。如果给 `callback` 传入了第二个参数，
它将会被转发到 `readable.push()` 方法，也就是说下面这两段是等价的：

```js
    transform.prototype._transform = function (data, encoding, callback) {
      this.push(data);
      callback();
    };

    transform.prototype._transform = function (data, encoding, callback) {
      callback(null, data);
    };
```

`transform._transform()` 方法以下划线为前缀，表明它属于定义它的类的内部，
并且不应该被用户程序直接调用。

### stream.PassThrough 类

`stream.PassThrough` 是一个简单的 Transform 流，简单地将输入字节传递到输出。
它的目的主要是用于示例和测试。但某些情况下，`stream.PassThrough` 可以作为一种构建块的新型流。

# 附加注释

## 与旧版 Node.js 间的兼容性

在 Node.js 0.10 之前，可读流接口比较简单，但是功能还不完善。

- 之前的可读流不会等待调用 `stream.read()` 方法，`data` 事件会立即开始发射。须要做一些额外的工作，
接收数据并保存到缓冲区，防止数据流失。
- `stream.pause()` 方法并不能保证使流暂停。这意味着即使是暂停的流，也有必要做好接收数据的准备。

在 Node.js v0.10 中添加了 Readable 类。为了和旧的 Node.js 程序兼容，可读流在添加
`data` 事件监听器或者调用 `stream.resume()` 方法时会切换到流模式。现在即使不添加额外的
`readable` 事件监听或者调用 `stream.read()` 方法，也不用担心数据丢失了。

虽然大部分的应用可以正常工作，但是在这些时候会引入边界情况：

- 没有添加任何 `data` 时间的监听。
- `stream.resume()` 方法没有被调用。
- 流没有被 pipe 到任何可写流。

例如，考虑以下代码：
```js
    // WARNING!  BROKEN!
    net.createServer((socket) => {

      // we add an 'end' method, but never consume the data
      socket.on('end', () => {
        // It will never get here.
        socket.end('The message was received but was not processed.\n');
      });

    }).listen(1337);
```

在 0.10 之前的版本，传入的数据会被直接丢弃，而在之后的版本中，socket 会保持在暂停模式。

解决这个问题的方法是在流开始的时候调用一次 `stream.resume()`。

除了这样，还可以使用 `readable.wrap()` 方法包装 0.10 之前的可读流，来确保切换到流模式。

```js
    // Workaround
    net.createServer((socket) => {

      socket.on('end', () => {
        socket.end('The message was received but was not processed.\n');
      });

      // start the flow of data, discarding it.
      socket.resume();

    }).listen(1337);
```

## readable.read(0)

在某些情况下，有必要触发底层可读流机制的刷新，而不实际消耗任何数据。此时，
可以调用 `readable.read(0)` 方法，它总是返回null。

如果内部缓冲区大小低于 `highWaterMark`，并且流当前没有在读取数据，调用
`stream.read(0)` 将触发底层的 `stream._read()` 调用。

大多数应用都不需要这么做，但在Node.js中，特别是在可读流类内部，有这样做的情况，

## readable.push('')

不推荐使用 `readable.push('')`。

将零字节字符串或缓冲区推送到非对象模式的流有一个有趣的副作用。因为它是对 `readable.push()`
的调用，调用将结束读取过程。 但是，因为参数是一个空字符串，所以并没有数据被添加到可读缓冲区，
所以没有什么数据可供消费。
