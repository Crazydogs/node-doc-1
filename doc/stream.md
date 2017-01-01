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

本文档分为两个主要部分，和一个附加注释部分。
第一部分介绍了开发者需要在开发中使用 steam 所涉及的 API。
第二部分介绍了开发者创建自定义 stream 所需要的 API。

## Stream 的类型

在 Node.js 中 stream 一共有 4 种基本类型

- Readable 可以读取数据的流(如 fs.createReadStream())
- Writable 可以写入数据的流(如 fs.createWriteStream())
- Duplex 同时可读又可写的流(如 net.Socket())
- Transform 在写入和读取过程中对数据进行修改变换的 Duplex 流(如 zlib.createDeflate())

### 对象模式

通过 Node API 创建的流，只能够对字符串或者 buffer 对象进行操作。但其实流的实现是可以基于其他的
Javascript 类型(除了 null, 它在流中有特殊的含义)的。这样的流就处在 "对象模式" 中。

在创建流对象的时候，可以通过提供 objectMode 参数来生成对象模式的流。
试图将现有的流转换为对象模式是不安全的。

### 缓冲区

Readable 和 Writable 流都会将数据储存在内部的缓冲区中。缓冲区可以分别通过
`writable._writableState.getBuffer()` 和 `readable._readableState.buffer` 来访问。

缓冲区中能容纳的数据数量由 stream 构造函数的 `highWaterMark` 选项决定。对于普通的流来说，
`highWaterMark` 选项表示总共可容纳的比特数。对于对象模式的流，该参数表示可以容纳的对象个数。

当一个可读实例调用 stream.push() 方法的时候，数据将会被推入缓冲区。如果没有数据的消费者出现，
调用 stream.read() 方法的话，数据就会一直留在缓冲队列中。

如果可读实例内部的缓冲区大小达到了创建时由 `highWaterMark` 指定的阈值，
可读流就会暂时停止从底层资源汲取数据，直到当前缓冲的数据成功被消耗掉
(也就是说，流停止调用内部用来填充缓冲区的 readable._read() 方法)。

在一个在可写实例上调用 writable.write(chunk) 方法的时候，数据会写入可写流的缓冲区。
如果缓冲区的数据量低于 highWaterMark 设定的值，调用 `writable.write()` 方法会返回 `true`，
否则 write 方法会返回 `false`。

stream 模块的 API，特别是 `stream.pipe()`，最主要的目的就是将数据的流动缓冲到一个可接受的水平，
不让不同速度的数据源之间的差异导致内存被占满。

Duplex 流和 Transform 流都是同时可读写的，所以他们会在内部维持两个缓冲区，分别用于读取和写入，
这样就可以允许两边同时独立操作，维持高效的数据流。比如说 `net.Socket` 是一个 Duplex 流，
Readable 端允许从 socket 获取、消耗数据，Writable 端允许向 socket 写入数据。
数据写入的速度很有可能与消耗的速度有差距，所以两端可以独立操作和缓冲是很重要的。

# 使用流涉及的 API

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

Duplex 流和 Transform 流都同时是 Readable 流与 Writable 流。

向流中写入数据或者消耗数据的应用并不需要直接实现流的接口，而且通常并不需要调用
`require(stream)`

想要创造新的 stream 类型的开发者请参考本文档第三部分（自定义 stream 所需要的 API）


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

如果调用 stream.write(chunk) 的时候返回了 `false`，那么当可以重新开始向流中写入数据的时候，
就会触发 `drain` 事件。

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

- 参数 <Error>

`error` 事件会在写入/传输数据发生错误的时候触发。事件的回调函数在调用的时候，
会接受一个 `Error` 参数。

#### finish 事件
添加于 v0.9.4

在执行 stream.end() 之后，会触发 `finish` 事件，此时所有数据都应该已经写入底层。

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

- 参数 `src` <stream.Readable> 流向此可写流的源

`pipe` 事件会在一个可读流调用 `stream.pipe()`，将一个可写流添加到他的目标集合的时候触发。

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

- 参数 `src` <Readable Stream> 停止流向此可写流的源

`pipe` 事件会在一个可读流调用 `stream.unpipe()`，将一个可写流从他的目标集合移除的时候触发。

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

- 参数 `chunk` <String> | <Buffer> | <any> 可选的将要写入的数据，对于处于正常模式的流来说，
chunk 必须是字符串或者 Buffer 对象，而对于处于对象模式的流，chunk 可以是除了
null 之外的任意 JavaScript 值
- 参数 `encoding` <String> 如果 chunk 是字符串，此参数为其编码
- 参数 `callback` <Function> 可选的回调函数，当流完成时调用。

调用 writable.end() 方法表示流将不会再有新的数据写入。可选的 chunk 和 encoding
参数允许在流关闭之前写入最后一块数据。如果提供了可选的 callback 参数，这个函数会成为
`finish` 事件的一个监听器。

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

- 参数 `encoding` <String> 新的默认编码
- 返回值 `this`

`writable.setDefaultEncoding()` 方法用于设置可写流的默认编码

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

- 参数 chunk <String> | <Buffer> 待写入的数据
- 参数 encoding <String> 如果 chunk 是字符串，则此参数为其编码
- 参数 callback <Function> 当数据块被刷新到底层的时候触发
- 返回值 <Boolean> 如果流想要等 `drain` 事件触发后，再写入新的数据，则返回 `false`，
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

在暂停模式下，必须显示的调用 stream.read() 从流中读取数据。

处在暂停模式的流可以通过下面的方式切换到流模式

- 给流的 `data` 事件添加监听器
- 调用流的 `stream.resume()` 方法
- 调用 `stream.pipe()` 方法将数据送入一个可写流

还可以通过下面的方式切回暂停模式

- 流没有 pipe 的目标时调用 `stream.pause()` 方法
- 流如果有 pipe 的目标，须要移除所有 `data` 事件的监听器，并通过 `stream.unpipe()`
方法移除所有的 pipe 目标。

一个重要的概念是，如果没有消耗或忽略数据的机制的话，可读流将不会生成数据。
如果数据的消费者失效了或者被移除，可读流将会尝试停止生成数据。

注意：由于向后兼容的原因，移除 `data` 事件监听器并不会将流暂停。另外，如果还有
pipe 的目标的话，调用 `stream.pause()` 方法并不能保证当 pipe 目标索取数据时，
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

- 参数 chunk <Buffer> | <String> | <any> 数据块。对于处在对象模式的流，chunk
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
或者不停调用 `stream.read()` 方法耗尽数据使其触发。

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

- 参数 <Error>

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
如果是前者，调用 `stream.read()` 将会返回可用的数据，如果是后者，`stream.read()`
会返回 `null`。在下面的示例中， `foo.txt` 是一个空文件：

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

- 返回值 <Boolean>

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

`readable.pause()` 方法会让处于流模式的可读流停止发射 `data` 事件，并退出流模式。
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

- 参数 `destination` <stream.Writable> 写入数据的目标
- 参数 `options` <Object> pip 操作的参数
    - `end` <Boolean> 是否在可读流结束的时候关闭可写流，默认为 `true`

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

默认情况下，当可读源发射 `end` 事件的时候，目标可写流会自动调用 `stream.end()`
方法，导致可写流不能再写入。如果想阻止此默认行为，须要将 `end` 选项设置为 false，
让目标可写流保持打开状态，像下面的例子：

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

- 参数 size <Number> 可选参数，指定要读取的数据量。
- 返回值 <String> | <Buffer> | <Null>

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

注意：在 `end` 事件触发后调用 `stream.read([size])` 方法将返回 `null`，不会产生错误。

#### readable.resume()
添加于 v0.9.4

- 返回值 `this`

`readable.resume()` 方法让一个显式暂停的流重新开始发射 `data` 事件，切换到流模式。

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

- 参数 `encoding` <String> 要使用的编码
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

- 参数 `destination` <stream.Writable> 可选参数，指定要解绑的流。

`readable.unpipe()` 方法会移除之前使用 `stream.pipe()` 附加的可写流。

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

- 参数 `chunk` <Buffer> | <String> 要退回可读流的数据

`readable.unshift()` 方法将一个数据块推回内部缓冲区。此方法主要用于数据被不应该消耗数据的代码消耗了，
须要撤销这次数据消耗的情况，退回缓冲区让数据可以传到其他正确的地方去。

注意：`stream.unshift(chunk)` 方法无法在 `end` 事件触发或者抛出运行错误之后调用。

使用 `stream.unshift()` 的开发者通常应该考虑使用 `Transform` 流代替。更多信息，
可以参考本文档第二部分 `开发者创建自定义 stream 所需要的 API`。

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

注意：`stream.unshift(chunk)` 与 `stream.push(chunk)` 方法不同，并不会改变流的内部状态来结束读取过程。
在读取数据期间(比如在一个自定义流的 _read() 中)使用 `stream.unshift()` 方法可能会导致意外的结果。
在调用 `readable.unshift()` 后立即调用 `stream.push()` 方法会正确的设置流的内部状态。
但最好还是不要在读取过程中调用此方法。

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

### 写操作中的错误处理

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








## Class: stream.Duplex

Duplex stream 是同时实现了 Readable 和 Writable 接口的 stream。

Duplex stream 的示例包括：

- [TCP socket](https://nodejs.org/dist/latest-v5.x/docs/api/net.html#net_class_net_socket)
- [zlib stream](https://nodejs.org/dist/latest-v5.x/docs/api/zlib.html)
- [crypto stream](https://nodejs.org/dist/latest-v5.x/docs/api/crypto.html)

## Class: stream.Readable

Readable stream 接口是对数据源的抽象。换言之，数据由 Readable stream 产生。

Readable stream 并不主动开始发送数据，直到显式表明可以接收数据时才会发送数据（类似于惰性加载）。

Readable stream 拥有两种模式：流动模式（flowing mode）和暂停模式（paused mode）。在流动模式中，数据从操作系统底层获取并尽可能快的传递给开发者；在暂停模式中，开发者必须显式调用 `stream.read()` 才能获取数据。其中，默认使用暂停模式。

注意，如果没有为 `data` 事件设置监听器，没有设置 `stream.pipe()` 的输出对象，那么 stream 就会自动切换为流动模式，且数据会丢失。
 
开发者可以通过以下方式切换为流动模式：

- 添加 `data` 事件处理器监听数据
- 调用 `stream.resume()` 显式打开数据流
- 调用 `stream.pipe()` 将数据发送给 Writable stream

通过以下模式可以切换为暂停模式：

- 调用 `stream.pasue()` 时不传递输出对象
- 移除 `data` 事件处理器且调用 `stream.pasue()` 时不传递输出对象

注意，为了保持向后兼容性，移除 `data` 事件处理器并不会自动暂停 stream。此外，如果存在输出对象，则调用 `stream.pause()` 并不能保证输出对象为空且要求获取数据时仍保持暂停状态。

下面是一个使用 Readable stream 的示例：

- [HTTP responses, on the client](https://nodejs.org/dist/latest-v5.x/docs/api/http.html#http_class_http_incomingmessage)
- [HTTP requests, on the server](https://nodejs.org/dist/latest-v5.x/docs/api/http.html#http_class_http_incomingmessage)
- [fs read streams](https://nodejs.org/dist/latest-v5.x/docs/api/fs.html#fs_class_fs_readstream)
- [zlib streams](https://nodejs.org/dist/latest-v5.x/docs/api/zlib.html)
- [crypto streams](https://nodejs.org/dist/latest-v5.x/docs/api/crypto.html)
- [TCP sockets](https://nodejs.org/dist/latest-v5.x/docs/api/net.html#net_class_net_socket)
- [child process stdout and stderr](https://nodejs.org/dist/latest-v5.x/docs/api/child_process.html#child_process_child_stdout)
- [process.stdin](https://nodejs.org/dist/latest-v5.x/docs/api/process.html#process_process_stdin)

#### 事件：'close'

当 stream 或底层资源（比如文件描述符）关闭时就会触发该事件。该事件也用于指示之后再没有其他事件会被触发，也不会再有任何计算。

并不是所有的 stream 都会触发 `close` 事件。

#### 事件：'data'

- `chunk`，Buffer 实例或字符串，数据块

给未显式暂停的 stream 绑定 `data` 事件监听器会让 stream 切换为流动模式，数据会被可能快的传送出去。

如果开发者只是想从 stream 尽快获取所有的数据，下面是最好的方式：

```js
var readable = getReadableStreamSomehow();
readable.on('data', (chunk) => {
  console.log('got %d bytes of data', chunk.length);
});
```

#### 事件：'end'

当没有数据可以读取时就会触发该事件。

注意，除非所有的数据都被处理了，否则不会触发 `end` 事件。可以通过切换到流动模式或反复调用 `stream.read()` 直到结束来实现。

```js
var readable = getReadableStreamSomehow();
readable.on('data', (chunk) => {
  console.log('got %d bytes of data', chunk.length);
});
readable.on('end', () => {
  console.log('there will be no more data.');
});
```

#### 事件：'error'

- `Error` 实例

如果接受到的数据存在错误就会触发该事件。

#### 事件：'readable'

当可以读取 stream 中的数据块时，就会触发 `readable` 事件。

在某些情况下，如果数据还没有准备好，那么监听 `readable` 事件将会从系统底层读取数据并放入内部缓冲中：

```js
var readable = getReadableStreamSomehow();
readable.on('readable', () => {
  // there is some data to read now
});
```

一旦内部缓存为空且所有的数据准备完成时，就会再次触发 `readable` 事件。

唯一的异常是在流动模式中，stream 数据传送完成时不会触发 `readable` 事件。

`readable` 事件用于表示 stream 接收到了新的消息：要么是新数据可用了要么是 stream 的数据全部发送完毕了。对于前一种情况，`stream.read()` 将会返回新数据；对于后一种情况，`stream.read()` 将会返回 null。举例来说，在下面的代码中，`foo.txt` 是一个空文件：

```js
const fs = require('fs');
var rr = fs.createReadStream('foo.txt');
rr.on('readable', () => {
  console.log('readable:', rr.read());
});
rr.on('end', () => {
  console.log('end');
});
```

运行后的输出结果：

```bash
$ node test.js
readable: null
end
```

#### readable.isPaused()

- 返回值类型：布尔值

该方法返回一个布尔值，表示 readable stream 是否是通过客户端代码（使用 `stream.pause()`）显式暂停的。

```js
var readable = new stream.Readable

readable.isPaused() // === false
readable.pause()
readable.isPaused() // === true
readable.resume()
readable.isPaused() // === false
```

#### readable.pause()

- 返回值类型：`this`

该方法会让处于流动模式中的 stream 停止触发 `data` 事件，并切换为暂停模式。所有可用的数据都会保留在内部缓存中。

```js
var readable = getReadableStreamSomehow();
readable.on('data', (chunk) => {
  console.log('got %d bytes of data', chunk.length);
  readable.pause();
  console.log('there will be no more data for 1 second');
  setTimeout(() => {
    console.log('now data will start flowing again');
    readable.resume();
  }, 1000);
});
```

#### readable.pipe(destination[, options])

- `destination`，stream.Writable 的实例，被写入数据的目标对象
- `options`，对象，包含以下属性：
    - `end`，布尔值，是否读取到了结束符，默认值为 true

该方法从 readable stream 拉取所有的数据，并将其写入 `destination`，整个过程是由系统自动管理的，所以不用担心 `destination` 会被 readable stream 的吞吐量压垮。

`pipe()` 可以接受多个 `distination`。

```js
var readable = getReadableStreamSomehow();
var writable = fs.createWriteStream('file.txt');
// All the data from readable goes into 'file.txt'
readable.pipe(writable);
```

由于该方法返回目标对象 stream，所以可以执行链式调用：

```js
var r = fs.createReadStream('file.txt');
var z = zlib.createGzip();
var w = fs.createWriteStream('file.txt.gz');
r.pipe(z).pipe(w);
```

下面代码模拟了 Unix 的 `cat` 命令：

```js
process.stdin.pipe(process.stdout);
```

默认情况下当源 stream 触发了 `end` 事件之后，就会使用 `destination` 调用 `stream.end()`，所以 `distination` 将不再可写。设置 `options` 为 `{ end: false }` 可以保持 `destination` stream 的开启状态。

下面的代码会让 `writer` 一直处于开启状态，所以 `end` 事件之后还可以写入数据 `Goodbye`：

```js
reader.pipe(writer, { end: false });
reader.on('end', () => {
  writer.end('Goodbye\n');
});
```

注意，`{ end: false }` 对 `process.stderr` 和 `process.stdout` 没有效用，只有进程退出时它们才会被关闭。

#### readable.read([size])

- `size`，数值，可选参数，用于指定读取的数据量
- 返回值类型：字符串、Buffer 实例或 Null

`read()` 从内部缓存读取数据并返回该数据。如果没有可用的数据，则返回 `null`。

如果传入了 `size`，则返回指定长度的数据。如果 `size` 长的数据不可能，除非是最后的数据块（返回剩余所有），否则返回 null。

如果未指定 `size` 参数，则会返回内部缓存中的所有数据。

该方法只能在暂停模式中调用，因为在流动模式中，系统会自动调用该方法直到内部缓存为空。

```js
var readable = getReadableStreamSomehow();
readable.on('readable', () => {
  var chunk;
  while (null !== (chunk = readable.read())) {
    console.log('got %d bytes of data', chunk.length);
  }
});
```

如果该方法返回了数据块，那么它也会触发 `data` 事件。

注意，在 `end` 事件之后调用 `stream.read([size])` 会返回 `null`，而不会抛出任何运行时错误。

#### readable.resume()

- 返回值类型：`this`

该方法用于恢复 readable stream 的 `data` 事件。

该方法会将 stream 切换为流动模式。如果你不想要使用来自 stream 的数据，只想触发 `end` 事件，可以使用 `stream.resume()` 启动数据的流动模式：

```js
var readable = getReadableStreamSomehow();
readable.resume();
readable.on('end', () => {
  console.log('got to the end, but did not read anything');
});
```

#### readable.setEncoding(encoding)

- `encoding`，字符串，使用的编码格式
- 返回值类型：`this`

该方法根据指定的编码格式返回字符串，而不是返回 Buffer 对象。举例来说，如果调用 `readable.setEncoding('utf8')`，则输出的数据就会被解析为 UTF-8 数据，并以字符串的形式返回。如果调用 `readable.setEncoding('hex')`，则会返回十六进制格式的字符串数据。

该方法可以正确处理多字节字符，但如果开发者直接获取 Buffer 数据并使用 `buf.toString(encoding)` 处理，则无法正确处理多字节字符。如果你想以字符串的形式读取数据，那么最好一直使用该方法。

当然，开发者也可以以 `readable.setEncoding(null)` 的形式调用该方法并禁用任何编码格式。该方法对于处理二进制数据或大量的多字节字符串非常有用。

```js
var readable = getReadableStreamSomehow();
readable.setEncoding('utf8');
readable.on('data', (chunk) => {
  assert.equal(typeof chunk, 'string');
  console.log('got %d characters of string data', chunk.length);
});
```

#### readable.unpipe([destination])

- `destination`，stream Writable 实例，可选参数，解除指定 stream

该方法用于移除调用 stream.pipe() 之前的钩子方法。

如果未指定 `destination`，则移除所有的 pipe。

如果指定了 `destination`，但没有对应的 pipe，则该操作无效。

```js
var readable = getReadableStreamSomehow();
var writable = fs.createWriteStream('file.txt');
// All the data from readable goes into 'file.txt',
// but only for the first second
readable.pipe(writable);
setTimeout(() => {
  console.log('stop writing to file.txt');
  readable.unpipe(writable);
  console.log('manually close the file stream');
  writable.end();
}, 1000);
```

#### readable.unshift(chunk)

- `chunk`，Buffer 实例或字符串，将数据块插入到读取队列

如果某个 stream 中的数据已经被解析过了，但是又需要解析前的数据，那么就可以使用该方法进行你操作将 stream 传递给第三方。

注意，`stream.unshift(chunk)` 不能再 `end` 事件之后调用，否则会抛出运行时错误。

如果开发者发现在程序中必须多次调用 `stream.unshift(chunk)`，那么请考虑实现一个 `Transform` stream。

```js
// Pull off a header delimited by \n\n
// use unshift() if we get too much
// Call the callback with (error, header, stream)
const StringDecoder = require('string_decoder').StringDecoder;
function parseHeader(stream, callback) {
  stream.on('error', callback);
  stream.on('readable', onReadable);
  var decoder = new StringDecoder('utf8');
  var header = '';
  function onReadable() {
    var chunk;
    while (null !== (chunk = stream.read())) {
      var str = decoder.write(chunk);
      if (str.match(/\n\n/)) {
        // found the header boundary
        var split = str.split(/\n\n/);
        header += split.shift();
        var remaining = split.join('\n\n');
        var buf = new Buffer(remaining, 'utf8');
        if (buf.length)
          stream.unshift(buf);
        stream.removeListener('error', callback);
        stream.removeListener('readable', onReadable);
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

注意，与 `stream.push(chunk)` 不同，`stream.unshift(chunk)` 不会通过重置 stream 的内部读取状态来中断读取继承。如果在读取期间调用 `unshift()`，则有可能出现未知的结果。如果在调用 `unshift()` 惠州立即调用 `stream.push('')`，将会重置读取状态，但是最好的方式还是在读取过程中不要调用 `unshift()`。

#### readable.wrap(stream)

- `stream`，Stream 实例，旧式的 readable stream

在 Node.js v0.10 之前，Stream 并没有实现完整的 Stream API。

如果你正在使用旧式的 Node.js 库，那么就只能使用 `data` 事件和 `stream.pasue()` 方法，通过 `wrap()` 方法可以创建一个使用旧式 stream 的 Readable stream。

应该尽量少用该函数，该函数存在的价值只是为了便于与旧版的 Node.js 程序和库进行交互。

```js
const OldReader = require('./old-api-module.js').OldReader;
const Readable = require('stream').Readable;
const oreader = new OldReader;
const myReader = new Readable().wrap(oreader);

myReader.on('readable', () => {
  myReader.read(); // etc.
});
```

## Class: stream.Transform

Transform stream 是 Duplex stream，其中输入是由输出计算而来的。它们都实现了 Readable 和 Writable 的接口。

下面是一些使用 Transform stream 的实例：

- [zlib streams](https://nodejs.org/dist/latest-v5.x/docs/api/zlib.html)
- [crypto streams](https://nodejs.org/dist/latest-v5.x/docs/api/crypto.html)

## Class: stream.Writable

Writable stream 接口是对接收数据的 destination 的抽象。

下面是一些使用 writable stream 的实例：

- [HTTP requests, on the client](https://nodejs.org/dist/latest-v5.x/docs/api/http.html#http_class_http_clientrequest)
- [HTTP responses, on the server](https://nodejs.org/dist/latest-v5.x/docs/api/http.html#http_class_http_serverresponse)
- [fs write streams](https://nodejs.org/dist/latest-v5.x/docs/api/fs.html#fs_class_fs_writestream)
- [zlib streams](https://nodejs.org/dist/latest-v5.x/docs/api/zlib.html)
- [crypto streams](https://nodejs.org/dist/latest-v5.x/docs/api/crypto.html)
- [TCP sockets](https://nodejs.org/dist/latest-v5.x/docs/api/net.html#net_class_net_socket)
- [child process stdin](https://nodejs.org/dist/latest-v5.x/docs/api/child_process.html#child_process_child_stdin)
- [process.stdout](https://nodejs.org/dist/latest-v5.x/docs/api/process.html#process_process_stdout)
- [process.stderr](https://nodejs.org/dist/latest-v5.x/docs/api/process.html#process_process_stderr)

#### 事件：'drain'

如果 `stream.write(chunk)` 返回 false，`drain` 事件就会通知开发者何时适合向 stream 写入更多的数据。

```js
// Write the data to the supplied writable stream one million times.
// Be attentive to back-pressure.
function writeOneMillionTimes(writer, data, encoding, callback) {
  var i = 1000000;
  write();
  function write() {
    var ok = true;
    do {
      i -= 1;
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

#### 事件：'error'

- `Error` 实例

如果写入或 pipe 数据的时候出现了错误就会触发该事件。

#### 事件：'finish'

调用 `stream.end()` 并且所有数据都被刷新到系统底层后，就会触发该事件。

```js
var writer = getWritableStreamSomehow();
for (var i = 0; i < 100; i ++) {
  writer.write('hello, #${i}!\n');
}
writer.end('this is the end\n');
writer.on('finish', () => {
  console.error('all writes are now complete.');
});
```

#### 事件：'pipe'

- `src`，stream.Readable，发起 pipe 写入操作的源 stream

当 readable stream 调用 `stream.pipe()` 方法时就会触发该事件，并添加 writable stream 到 `destination` 上。

```js
var writer = getWritableStreamSomehow();
var reader = getReadableStreamSomehow();
writer.on('pipe', (src) => {
  console.error('something is piping into the writer');
  assert.equal(src, reader);
});
reader.pipe(writer);
```

#### 事件：'unpipe'

- `src`，Readable stream，发起 unpipe 写入操作的源 stream

当 readable stream 调用 `stream.unpipe()` 方法时就会触发该事件，并移除 `destination` 中的 writable stream。

```js
var writer = getWritableStreamSomehow();
var reader = getReadableStreamSomehow();
writer.on('unpipe', (src) => {
  console.error('something has stopped piping into the writer');
  assert.equal(src, reader);
});
reader.pipe(writer);
reader.unpipe(writer);
```

#### writable.cork()

该方法强制系统缓存所有写入的数据。

调用 `stream.uncork()` 或 `stream.end()` 时都会刷新缓存数据。

#### writable.end([chunk][, encoding][, callback])

- `chunk`，字符串或 Buffer 实例，写入的数据
- `encoding`，字符串，如果 chunk 是字符串，则该参数指定编码格式
- `callback`，函数，stream 结束时执行的回调函数

当没有数据需要写入到 stream 时可以调用该方法。如果指定了 `callback`，则该参数会被绑定为 `finish` 事件的监听器。

在 `stream.end()` 之后调用 `stream.write()` 将会触发错误。

```js
// write 'hello, ' and then end with 'world!'
var file = fs.createWriteStream('example.txt');
file.write('hello, ');
file.end('world!');
// writing more now is not allowed!
```

#### writable.setDefaultEncoding(encodign)

- `encoding`，字符串，编码格式

该方法用于设置 writable stream 的默认字符串编码格式。

#### writable.uncork()

该方法用于刷新调用 `stream.cork()` 之后缓存的所有数据。

#### writable.write(chunk[, coding][, callback])

- `chunk`，字符串或 Buffer 实例，写入的数据
- `encoding`，字符串，如果 chunk 是字符串，则该参数指定编码格式
- `callback`，函数，数据块刷新后执行的回调函数
- 返回值类型：布尔值，如果数据处理完成则返回 true

该方法用于向系统底层写入数据，并在数据完全写入后指定回调函数。如果出现了错误，将无法确定是否会执行 callback，所以为了检测错误，请监听 `error` 事件。

该方法的返回值用于通知开发者是否应该继续写入数据。如果数据已在内部缓存，那么就会返回 false，否则返回 true。

该返回值主要是建议性的，开发者还是可以继续写入数据，即使返回的是 false。不过，写入的数据会被缓存在内存中，所以最好不要这么做。相反，开发者可以等出发了 `drain` 事件之后再继续写入数据。

## 自定义 Stream 的 API

实现自定义 Stream 的模式：

1. 从恰当的父类创建你需要的子类，`util.inherits()` 方法对此很有帮助
1. 在子类的构造器中合理调用父类的构造器，确保内部机制初始化成功
1. 实现一个或多个方法，如下所示

所要扩展的类和要实现的方法取决于开发者编写的 stream 的类型：

| 用途                | 类               | 实现的方法             |
| -------------------|------------------|-----------------------|
| 只读                |Readable          |_read                  |
| 只写                |Writable          |_write, _writev        |
| 读写                |Duplex            |_read, _write, _writev |
| 写数据读结果         |Transform         |_transform, _flush     |

在开发者的实现代码里，千万不要调用第一部分的代码，否则会引起不利的副作用。

## Class: stream.Duplex

Duplex stream 是可读写的 stream，比如 TCP socket 连接。

注意，`stream.Duplex` 是一个抽象类，也是 `stream._read(size)` 和 `stream._write(chunk, encoding, callback)` 的底层基础。

因为 JavaScript 没有多原型继承机制，所以该类继承自 Readable，而又寄生于 Writable。从而允许开发者实现底层的 `stream._read(n)` 和 `stream._write(chunk, encoding, callback)` 方法来扩展 Duplex 类。

#### new stream.Duplex(options)

- `options` 是一个对象，同时传递给 Writable 和 Readable 的构造器，具有以下属性：
    - `allowHalfOpen`，布尔值，默认值为 true。如果值为 false，则当 writable 端停止时 readable 端的 stream 也会自动停止，反之异然
    - `readableObjectMode`，布尔值，默认值为 false。为 readable stream 设置 `objectMode`。如果 `objectMode === true`，则没有任何效果
    - `writableObjectMode`，布尔值，默认值为 false。为 writable stream 设置 `objectMode`。如果 `objectMode === true`，则没有任何效果

对于继承了 Duplex class 的类，只有正确调用构造器才能确保成功初始化缓存设置。

## Class: stream.PassThrough

该类是 Transform stream 的一个实现，相对而言并不重要，只是简单的将输入的数据传送给输出。创建该类的目的主要是为了演示和测试，但也偶尔用做新型 stream 的构建基块。

## Class: stream.Readable

`stream.Readable` 是一个可扩展的底层抽象类，常服务于 `stream._read(size)` 方法。

#### new stream.Readable([options])

- `options`，对象
    - `highwatermark`，数值，表示内部缓存可以存储的最大字节量，默认值为 16384（16kb），对于 `objectMode` stream，默认值是 16
    - `encoding`，字符串，如果指定了该参数，则 Buffer 实例会被转换为指定格式的字符串，默认值为 `null`
    - `objectmode`，布尔值，表示 stream 的行为是否应该像是对象 stream。也就是说，`stream.read(n)` 返回一个单值，而不是长度为 n 的 Buffer 实例，默认值为 `false`
    - `read`，函数，`stream._read()` 方法的实现

对于继承了 Readable class 的类，只有正确调用构造器才能确保成功初始化缓存设置。

#### readable._read(size)

- `size`，数值，异步读取的字节量

注意，可以实现该方法，但不要直接使用该方法。

该方法使用下划线作为前缀，这表明它是类的内部方法，只应该在 Readable 类内部使用。所有 Readable stream 的实现都应该提供一个 `_read` 方法从系统底层获取资源和数据。

当调用 `_read()` 时，如果资源数据可以使用了，则 `_read()` 的实现应该通过调用 `this.push(dataChunk)` 将数据加入到读取队列中。`_read()` 需要持续读取资源并将其添加到队列中，直到返回 `false` 则停止读取数据。

只有在数据读取停止之后再次调用 `_read()` 才能读取更多的数据并将数据添加到队列。

注意，一旦调用了 `_read()` 方法，那么只有调用 `stream.push()` 之后才会再次调用 `_read()`。

`size` 参数只具有建议性而不具有强制性，该参数只是用来通知读取的数据量。但与具体的实现无关，比如对于 TCP 和 TLS，它们可能会忽略该参数，只要数据可用就会输出数据。举例来说，调用 `stream.push(chunk)` 之前也没必要等到 `size` 长的数据块准备完毕。

#### readable.push(chunk[, encoding])

- `chunk`，Buffer 实例、字符串或 Null，添加到读取队列的数据块
- `encoding`，字符串，字符串的编码格式，必须是一个有效的 Buffer 编码格式，比如 utf8 或 ascii
- 返回值类型：布尔值，用于指定是否应该继续推送数据

注意，Readable 的实现者必须调用该方法，而不是由 Readable stream 的使用调用该方法。

如果传入的值不是 null，则 `push()` 方法将数据块推送到队列，便于随后的 stream 处理器使用。如果传入的是 null，则是向 stream 发送结束信号（EOF），之后将不会再写入数据。

当触发了 `readable` 事件之后，通过 `stream.read()` 可以拉取使用 `push()` 添加的数据。

该方法设计的非常灵活。举例来说，开发者可以使用该方法封装一个底层资源，包含了一些暂停和回复机制，以及一个数据回调函数：

```js
// source is an object with readStop() and readStart() methods,
// and an `ondata` member that gets called when it has data, and
// an `onend` member that gets called when the data is over.

util.inherits(SourceWrapper, Readable);

function SourceWrapper(options) {
  Readable.call(this, options);

  this._source = getLowlevelSourceObject();

  // Every time there's data, we push it into the internal buffer.
  this._source.ondata = (chunk) => {
    // if push() returns false, then we need to stop reading from source
    if (!this.push(chunk))
      this._source.readStop();
  };

  // When the source ends, we push the EOF-signaling `null` chunk
  this._source.onend = () => {
    this.push(null);
  };
}

// _read will be called when the stream wants to pull more data in
// the advisory size argument is ignored in this case.
SourceWrapper.prototype._read = function(size) {
  this._source.readStart();
};
```

#### 实例：计数 stream

这是一个基础的 Readable stream 实例，它从 1 到 1000000 递增的顺序触发数字直到结束：

```js
const Readable = require('stream').Readable;
const util = require('util');
util.inherits(Counter, Readable);

function Counter(opt) {
  Readable.call(this, opt);
  this._max = 1000000;
  this._index = 1;
}

Counter.prototype._read = function() {
  var i = this._index++;
  if (i > this._max)
    this.push(null);
  else {
    var str = '' + i;
    var buf = new Buffer(str, 'ascii');
    this.push(buf);
  }
};
```

#### 实例：简单协议 v1（初始版）

该实例与 [parseHeader 方法](https://nodejs.org/dist/latest-v5.x/docs/api/stream.html#stream_readable_unshift_chunk)类似，但它是一个自定义的 stream。值得注意的是，该方法不会将传入的数据转换为字符串。

实际上，更好的方式是使用 Transform stream 实现该方法，详情请查看 [SimpleProtocol v2](https://nodejs.org/dist/latest-v5.x/docs/api/stream.html#stream_example_simpleprotocol_parser_v2)。

```js
// A parser for a simple data protocol.
// The "header" is a JSON object, followed by 2 \n characters, and
// then a message body.
//
// NOTE: This can be done more simply as a Transform stream!
// Using Readable directly for this is sub-optimal. See the
// alternative example below under the Transform section.

const Readable = require('stream').Readable;
const util = require('util');

util.inherits(SimpleProtocol, Readable);

function SimpleProtocol(source, options) {
  if (!(this instanceof SimpleProtocol))
    return new SimpleProtocol(source, options);

  Readable.call(this, options);
  this._inBody = false;
  this._sawFirstCr = false;

  // source is a readable stream, such as a socket or file
  this._source = source;

  source.on('end', () => {
    this.push(null);
  });

  // give it a kick whenever the source is readable
  // read(0) will not consume any bytes
  source.on('readable', () => {
    this.read(0);
  });

  this._rawHeader = [];
  this.header = null;
}

SimpleProtocol.prototype._read = function(n) {
  if (!this._inBody) {
    var chunk = this._source.read();

    // if the source doesn't have data, we don't have data yet.
    if (chunk === null)
      return this.push('');

    // check if the chunk has a \n\n
    var split = -1;
    for (var i = 0; i < chunk.length; i++) {
      if (chunk[i] === 10) { // '\n'
        if (this._sawFirstCr) {
          split = i;
          break;
        } else {
          this._sawFirstCr = true;
        }
      } else {
        this._sawFirstCr = false;
      }
    }

    if (split === -1) {
      // still waiting for the \n\n
      // stash the chunk, and try again.
      this._rawHeader.push(chunk);
      this.push('');
    } else {
      this._inBody = true;
      var h = chunk.slice(0, split);
      this._rawHeader.push(h);
      var header = Buffer.concat(this._rawHeader).toString();
      try {
        this.header = JSON.parse(header);
      } catch (er) {
        this.emit('error', new Error('invalid simple protocol data'));
        return;
      }
      // now, because we got some extra data, unshift the rest
      // back into the read queue so that our consumer will see it.
      var b = chunk.slice(split);
      this.unshift(b);
      // calling unshift by itself does not reset the reading state
      // of the stream; since we're inside _read, doing an additional
      // push('') will reset the state appropriately.
      this.push('');

      // and let them know that we are done parsing the header.
      this.emit('header', this.header);
    }
  } else {
    // from there on, just provide the data to our consumer.
    // careful not to push(null), since that would indicate EOF.
    var chunk = this._source.read();
    if (chunk) this.push(chunk);
  }
};

// Usage:
// var parser = new SimpleProtocol(source);
// Now parser is a readable stream that will emit 'header'
// with the parsed header data.
```

## Class: stream.Transform

Transform stream 是 Duplex stream，输入和输出具有因果关系，比如 zlib stream 和 crypto stream。

输入和输出没有数据块大小、数据块数量以及到达时间的要求。举例来说，一个哈希 stream 只会在结束时向输出发送一个单一的数据块；一个 zlib stream 则会生成比输入或大或小的输出结果。

Transform class 必须实现 `stream._transform()` 方法，可以选择性地实现 `stream._flush()` 方法。

#### new stream.Transform([options])

- `options`，对象，将会传递给 Writable 和 Readable class 的构造器，包含以下属性：
    - `transform`，函数，是 `stream._transform()` 的实现
    - `flush`，函数，是 `stream._flush()` 的实现

对于继承了 Transform class 的类，只有正确调用构造器才能确保成功初始化缓存设置。

#### 事件：'finish' 和 'end'

`finish` 和 `end` 事件分别来自于 Writable 和 Readable class。调用 `stream.read()` 并使用 `stream._transform()` 方法处理完所有的数据块之后就会触发 `finish()` 事件；`stream._flush()` 调用完内部的回调函数并输出完所有的数据之后触发 `end` 事件。

#### transform._flush(callback)

- `callback`，函数，当开发者刷新完所有的剩余数据之后执行该回调函数

**注意，一定不要直接调用该函数**。可以在子类中实现该方法，且只允许 Transform class 的内部方法调用它。

在某些情况下爱，transform 操作需要在 stream 的最后触发额外的数据。举例来说，一个 zlib 压缩 stream 会存储一些优化压缩结果的内部状态。

在这些情况下，开发者可以实现一个 `_flush()` 方法，该方法会在所有写入的数据被处理之后、触发 `end` 事件结束 readable stream 之前被调用。与 `stream._transform()` 类似，当刷新操作完成之后，会调用 `transform.push(chunk)` 零次或多次，最后调用 `callback`。

该方法名使用了下划线的前缀，表示它是类的内部方法，不应该被开发者的程序直接调用，而是希望开发者在自定义的扩展类中重写该方法。

#### transform._transform(chunk, encoding, callback)

- `chunk`，Buffer 实例或字符串，用于传输的数据块。除非 `decodeStrigns === false`，否则该参数都是 Buffer 实例
- `encoding`，字符串，如果 `chunk` 是一个字符串，则该参数指定字符串的编码格式。如果 chunk 是一个 Buffer 实例，则该参数是一个特殊值 "buffer"，在这种情况下请忽略该值。
- `callback`，函数，当处理完输入的数据块之后将会执行该回调函数

**注意，一定不要直接调用该函数**。可以在子类中实现该该方法，且只允许 Transform class 的内部方法调用它。

所有的 Transform stream 实现都必须提供一个 `_transform()` 方法接收输入并生成输出数据。

`_transform` 可以处理 Transform class 中规定的任何事情，比如处理写入的字节、将它们传给 readable stream、处理异步 I/O等任务。

调用 `transform.push(outputChunk)` 根据输入生成输出数据的次数取决于开发者想要输出的数据量。

之后当前数据块被完全处理之后才可以调用回调函数。注意，输入块或许有也或许没有对应的输出块。如果给回调函数设置了第二个参数，则该参数会被传递给 push 方法。换言之，以下代码相等：

```js
transform.prototype._transform = function (data, encoding, callback) {
  this.push(data);
  callback();
};

transform.prototype._transform = function (data, encoding, callback) {
  callback(null, data);
};
```

该方法名使用了下划线的前缀，表示它是类的内部方法，不应该被开发者的程序直接调用，而是希望开发者在自定义的扩展类中重写该方法。

#### 实例：SimpleProtocol 解析器 v2

上面的简单协议解析器可以使用高阶的 Transform stream class 来实现，处理方式与 `parseHeader` 和 `SimpleProtocol v1` 类似。

在下面的代码中，并没有将输入作为参数，而是将其 pipe 进了解析器，这种方案更符合 Node.js stream 的使用习惯：

```js
const util = require('util');
const Transform = require('stream').Transform;
util.inherits(SimpleProtocol, Transform);

function SimpleProtocol(options) {
  if (!(this instanceof SimpleProtocol))
    return new SimpleProtocol(options);

  Transform.call(this, options);
  this._inBody = false;
  this._sawFirstCr = false;
  this._rawHeader = [];
  this.header = null;
}

SimpleProtocol.prototype._transform = function(chunk, encoding, done) {
  if (!this._inBody) {
    // check if the chunk has a \n\n
    var split = -1;
    for (var i = 0; i < chunk.length; i++) {
      if (chunk[i] === 10) { // '\n'
        if (this._sawFirstCr) {
          split = i;
          break;
        } else {
          this._sawFirstCr = true;
        }
      } else {
        this._sawFirstCr = false;
      }
    }

    if (split === -1) {
      // still waiting for the \n\n
      // stash the chunk, and try again.
      this._rawHeader.push(chunk);
    } else {
      this._inBody = true;
      var h = chunk.slice(0, split);
      this._rawHeader.push(h);
      var header = Buffer.concat(this._rawHeader).toString();
      try {
        this.header = JSON.parse(header);
      } catch (er) {
        this.emit('error', new Error('invalid simple protocol data'));
        return;
      }
      // and let them know that we are done parsing the header.
      this.emit('header', this.header);

      // now, because we got some extra data, emit this first.
      this.push(chunk.slice(split));
    }
  } else {
    // from there on, just provide the data to our consumer as-is.
    this.push(chunk);
  }
  done();
};

// Usage:
// var parser = new SimpleProtocol();
// source.pipe(parser)
// Now parser is a readable stream that will emit 'header'
// with the parsed header data.
```

## Class: stream.Writable

`stream.Writable` 是一个可扩展的抽象类，可用于 `stream._write(chunk, encoding, callback)` 等方法的底层实现。

#### new stream.Writable([options])

- `options`，对象
    - `highwatermark`，数值，当 `stream.write()` 开始返回 `false` 时的缓存级别，默认值是 16384（16kb），对于 `objectMode` stream，默认值是 16
    - `decodeStrings`，布尔值，该参数决定是否在讲字符串传递给 `stream._write()` 之前将其转换为 Buffer，默认值为 true
    - `objectmode`，布尔值，决定 `stream.write(anyObj)` 是否是一个有效操作。如果值为 true，则可以写入任意类型的数据，而不只是 Buffer 和字符串数据，默认值为 `false`
    - `write`，函数，`stream._write()` 的实现
    - `writev`，函数，`stream._writev()` 的实现

对于继承了 Writable class 的类，只有正确调用构造器才能确保成功初始化缓存设置。

#### writable._write(chunk, encoding, callback)

- `chunk`，Buffer 实例或字符串，写入的数据块。除非 `decodeStrings === false`，否则该参数只能是 Buffer 实例
- `encoding`，字符串，如果 `chunk` 是一个字符串，则该参数指定字符串的编码格式。如果 chunk 是一个 Buffer 实例，则该参数是一个特殊值 "buffer"，在这种情况下请忽略该值。
- `callback`，函数，当处理完输入的数据块之后须执行该回调函数（可以带一个 error 参数）

**注意，一定不能直接调用该方法**。可以在子类中实现该方法，且只允许 Writable class 的内部方法调用它。

调用 `callback(err)` 用于通知系统数据写入完成或者出现了错误。

如果在构造器中设置了 `decodeStrings` 选项，那么 `chunk` 就只能是字符串而不能是 Buffer 实例，`encoding` 参数用于表示字符串的编码格式。这种实现是为了优化某些字符串的处理。如果没有显式设置 `decodeStrings === false`，那么系统会忽略 `encoding` 参数，并假设 `chunk` 是一个 Buffer 实例。

该方法名使用了下划线的前缀，表示它是类的内部方法，不应该被开发者的程序直接调用，而是希望开发者在自定义的扩展类中重写该方法。

#### writable._writev(chunks, callback)

- `chunk`，数组，写入的数据。每一个数据块都遵循如下格式：`{ chunk: ..., encoding: ... }`
- `callback`，函数，当处理完数据块之后将会执行该回调函数

**注意，一定不能直接调用该方法**。可以在子类中实现该方法，且只允许 Writable class 的内部方法调用它

该方法是 \_write 方法的补充。当流中须要一次性处理多个数据块的能力时，可以实现它。如果提供了 \_writev 方法，会使用内部缓存中的所有数据块作为参数来调用它。

该方法名使用了下划线的前缀，表示它是类的内部方法，不应该被开发者的程序直接调用，而是希望开发者在自定义的扩展类中重写该方法。

## 简化构造器 API

在某些简单的情况下，不通过继承创建 stream 也大有用处。

通过向构造器传递恰当的方法即可实现这一目标。

#### Duplex

```js
var duplex = new stream.Duplex({
  read: function(n) {
    // sets this._read under the hood

    // push data onto the read queue, passing null
    // will signal the end of the stream (EOF)
    this.push(chunk);
  },
  write: function(chunk, encoding, next) {
    // sets this._write under the hood

    // An optional error can be passed as the first argument
    next()
  }
});

// or

var duplex = new stream.Duplex({
  read: function(n) {
    // sets this._read under the hood

    // push data onto the read queue, passing null
    // will signal the end of the stream (EOF)
    this.push(chunk);
  },
  writev: function(chunks, next) {
    // sets this._writev under the hood

    // An optional error can be passed as the first argument
    next()
  }
});
```

#### Readable

```js
var readable = new stream.Readable({
  read: function(n) {
    // sets this._read under the hood

    // push data onto the read queue, passing null
    // will signal the end of the stream (EOF)
    this.push(chunk);
  }
});
```

#### Transform

```js
var transform = new stream.Transform({
  transform: function(chunk, encoding, next) {
    // sets this._transform under the hood

    // generate output as many times as needed
    // this.push(chunk);

    // call when the current chunk is consumed
    next();
  },
  flush: function(done) {
    // sets this._flush under the hood

    // generate output as many times as needed
    // this.push(chunk);

    done();
  }
});
```

#### Writable

```js
var writable = new stream.Writable({
  write: function(chunk, encoding, next) {
    // sets this._write under the hood

    // An optional error can be passed as the first argument
    next()
  }
});

// or

var writable = new stream.Writable({
  writev: function(chunks, next) {
    // sets this._writev under the hood

    // An optional error can be passed as the first argument
    next()
  }
});
```

## Stream: 内部玄机

#### 缓存

Writable 和 Readable stream 都会在对象内部缓存数据，该对象可以通过 `_writableState.getBuffer()` 或 `_readableState.buffer` 获取。

缓存的总量取决于构造器中接收的 `highWaterMark` 配置信息。

调用 `stream.push(chunk)` 可以将数据缓存到 Readable stream 中。如果数据没有经过 `stream.read()` 处理，就会一直待在内部队列，直到被拉去和处理。

当开发者反复 `stream.write(chunk)` 时就会将数据缓存到 Writable stream 中，直到 `stream.write(chunk)` 返回 `false`。

设计 stream 的初衷，尤其是对于 `stream.pipe()` 方法，是为了在可控范围内限制数据的缓存量，所以即使输入资源和输出的目标对象之间的速度存在差异，都不会过度影响内存的使用。

#### 兼容性

在 Node.js v0.10 之前的版本，Readable stream 的接口非常简单，功能贫乏，实用性不强。

- 在老版本中，系统不会等待开发者调用 `stream.read()` 方法，而是触发 `data` 事件。如果开发者要决定如何处理数据，那么就需要手动缓存数据块
- 在老版本中，`stream.pause()` 只是建议性的方法，而不是绝对有效的。这也就是说，即使 stream 处于暂停状态，开发者仍有可能接收到 `data` 事件。

在 Node.js v0.10，添加了 Readable。为了保持向后兼容性，当添加了 `data` 事件处理器之后或调用了 `stream.resume()` 之后，Readable stream 就会切换为流动模式。这么做的好处是，即使你没有使用 `stream.read()` 或 `readable` 事件，也无需担心会丢失数据块。

虽然大多数的程序都会正常运行，但还是有必要介绍一些边缘用例：

- 没有添加任何 `data` 事件处理器
- 从没调用过 `stream.resume()` 方法
- stream 没有 pipe 到任何 writable destination

举例来说，思考一下下面的代码：

```js
// WARNING!  BROKEN!
net.createServer((socket) => {

  // we add an 'end' method, but never consume the data
  socket.on('end', () => {
    // It will never get here.
    socket.end('I got your message (but didnt read it)\n');
  });

}).listen(1337);
```

在 Node.js v0.10 之前，新传入的消息数据会被直接丢弃。不过从 Node.js v0.10 之后，socket 会保持暂停状态。

这种情况下的变通方法就是使用 `stream.resume()` 启动数据流：

```js
// Workaround
net.createServer((socket) => {

  socket.on('end', () => {
    socket.end('I got your message (but didnt read it)\n');
  });

  // start the flow of data, discarding it.
  socket.resume();

}).listen(1337);
```

为了让新的 Readable stream 切换到流动模式，在 v0.10 之前 stream 可以通过 `stream.wrap()` 包装成 Readable stream。

#### Object Mode

通常来说，stream 只适用于对字符串和 Buffer 实例的操作。

但处于 object mode 的 stream 则可以处理任何 JavaScript 值。

在 object mode 中，不论调用 `stream.read(size)` 时 `size` 是多少，Readable stream 都会返回一个单元素。

在 object mode 中，不论调用 `stream.write(data, encoding)` 时 `encoding` 是什么，Writable stream 都会忽略该参数。

特殊值 `null` 在 object mode 中仍然保持了特殊性。也就是说，对于 object mode 中的 Readable stream，如果 `stream.read()` 返回了 null，表示没有数据了；如果调用 `stream.push(null)`，表示 stream 的数据推送结束了。

Node.js 的核心 stream 没有一个是 object mode stream。object mode 只存在于用户的 stream 库中。

开发者应该在 stream 子类的构造器中设置 `objectMode` 配置信息，在其他地方设置则不安全。

对于 Duplex stream 的 `objectMode` 参数，可以通过 `readableObjectMode` 和 `writableObjectMode` 设置为 readable 或 writable。这些选项可以通过 Transform stream 来实现解析器和序列化器。

```js
const util = require('util');
const StringDecoder = require('string_decoder').StringDecoder;
const Transform = require('stream').Transform;
util.inherits(JSONParseStream, Transform);

// Gets \n-delimited JSON string data, and emits the parsed objects
function JSONParseStream() {
  if (!(this instanceof JSONParseStream))
    return new JSONParseStream();

  Transform.call(this, { readableObjectMode : true });

  this._buffer = '';
  this._decoder = new StringDecoder('utf8');
}

JSONParseStream.prototype._transform = function(chunk, encoding, cb) {
  this._buffer += this._decoder.write(chunk);
  // split on newlines
  var lines = this._buffer.split(/\r?\n/);
  // keep the last partial line buffered
  this._buffer = lines.pop();
  for (var l = 0; l < lines.length; l++) {
    var line = lines[l];
    try {
      var obj = JSON.parse(line);
    } catch (er) {
      this.emit('error', er);
      return;
    }
    // push the parsed object out to the readable consumer
    this.push(obj);
  }
  cb();
};

JSONParseStream.prototype._flush = function(cb) {
  // Just handle any leftover
  var rem = this._buffer.trim();
  if (rem) {
    try {
      var obj = JSON.parse(rem);
    } catch (er) {
      this.emit('error', er);
      return;
    }
    // push the parsed object out to the readable consumer
    this.push(obj);
  }
  cb();
};
```

#### stream.read(0)

在某些情况下，开发者只想刷新底层的 readable stream 机制，而不处理任何数据，那么就可以使用 `stream.read(0)`，该方法返回 null。

如果内部的 Buffer 数据长度小于 `highWaterMark`，且尚未被 stream 读取，那么调用 `stream.read(0)` 就调用发底层的 `stream._read()`。

一般来说没有调用该方法的必要。不过，开发者可能会发现在 Node.js 内部有这样的调用，特别是 Readable stream 的内部。

#### stream.push('')

推送一个零字节的字符串或 Buffer 实例数据会发生一些有趣的现象。因为调用了 `stream.push()`，所以会终止 `reading` 进程。不过，实际上并没有向 readable buffer 添加任何数据，也就不需要开发者处理任何数据了。

虽然现在很少有空数据的情况，但是通过调用 `stream.read(0)` 可以检查是否有任务在处理你的 stream。出于这种目的，你还是有可能调用 `stream.push('')` 的。

到目前为止，只在 `tls.CryptoStream` 类中使用过该手段，但该方法在 Node.js/io.js v1.0 已被抛弃了。如果你必须使用 `stream.push('')` 方法，请先思考是否有其他处理方式，因为这种做法会被视为发生了极其严重的错误。

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
