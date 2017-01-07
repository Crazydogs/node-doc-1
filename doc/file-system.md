## 文件系统

<div class="s s2"></div>

文件系统模块是一个封装了标准 POSIX 文件 I/O 操作的集合。通过 `require('fs')` 可以加载该模块。该模块中的所有方法都有异步执行和同步执行两个版本。

异步执行的方法都接收一个回调函数作为最后一个参数。传递给回调函数的参数取决于具体的异步方法，但第一个参数通常用于接收异步方法的错误信息。如果异步执行没有问题，则第一个表示错误信息的参数通常为 `null` 或 `undefined`。

当执行同步方法时，发生的任何错误都会直接抛出。开发者可以使用 `try/catch` 捕获异常，或者直接放纵异常继续向上传播。

下面代码演示了如何使用异步执行的方法：

```js
const fs = require('fs');

fs.unlink('/tmp/hello', (err) => {
  if (err) throw err;
  console.log('successfully deleted /tmp/hello');
});
```

下面代码演示了如何使用同步执行的方法：

```js
const fs = require('fs');

fs.unlinkSync('/tmp/hello');
console.log('successfully deleted /tmp/hello');
```

由于异步方法没有固定的执行顺序，所以下面的这段代码很有可能会抛出错误：

```js
fs.rename('/tmp/hello', '/tmp/world', (err) => {
  if (err) throw err;
  console.log('renamed complete');
});
fs.stat('/tmp/world', (err, stats) => {
  if (err) throw err;
  console.log(`stats: ${JSON.stringify(stats)}`);
});
```

这是因为 `fs.stat` 方法有可能会在 `fs.rename` 之前执行，正确的方式是嵌套执行这段代码：

```js
fs.rename('/tmp/hello', '/tmp/world', (err) => {
  if (err) throw err;
  fs.stat('/tmp/world', (err, stats) => {
    if (err) throw err;
    console.log(`stats: ${JSON.stringify(stats)}`);
  });
});
```

在负载较重的进程中，建议使用异步执行的方法，这是因为同步方法会阻塞进程的执行，在同步方法结束之前，所有的连接和任务都会处于挂起状态。

在该模块中可以使用文件的相对路径，但是需要牢记，相对路径的相对目标是 `process.cwd()`。

该模块中的大多数函数都允许开发者忽略回调函数。如果忽略了回调函数，系统就会使用一个默认的回调函数，用于抛出执行过程中出现的错误。为了获取函数调用点的原始堆栈追踪信息，需要开启环境变量 `NODE_DEBUG`：

```js
$ cat script.js
function bad() {
  require('fs').readFile('/');
}
bad();

$ env NODE_DEBUG=fs node script.js
fs.js:66
        throw err;
              ^
Error: EISDIR, read
    at rethrow (fs.js:61:21)
    at maybeCallback (fs.js:79:42)
    at Object.fs.readFile (fs.js:153:18)
    at bad (/path/to/script.js:2:17)
    at Object.<anonymous> (/path/to/script.js:5:1)
    <etc.>
```

## Class: fs.FSWatcher

`fs.watch()` 返回的对象都属于该类型。

#### 事件：'change'

- `event`，字符串，改变的 fs 类型
- `filename`，字符串，修改后的文件名

当监视的文件或目录发生变化时触发该事件。

#### 事件：'error'

- `error`，Error 实例

发生错误时触发该事件。

#### watcher.close()

该方法用于停止监控 `fs.FSWatcher` 指定的文件或目录。

## Class: fs.ReadStream

`ReadStream` 是一个可读的 Stream 实例。

#### 事件：'open'

- `fd`，数值，传递给 ReadStream 的文件描述符的值

当 ReadStream 中的文件打开时，触发该事件。

#### readStream.path

该属性表示表示 stream 读取的文件路径。

## Class: fs.Stats

`fs.stat()`、`fs.lstat()`、`fs.fstat()` 及相应的同步版本都会返回该类型的对象：

- stats.isFile()
- stats.isDirectory()
- stats.isBlockDevice()
- stats.isCharacterDevice()
- stats.isSymbolicLink() (only valid with fs.lstat())
- stats.isFIFO()
- stats.isSocket()

对于普通文件来说，`util.inspect(stats)` 返回类似如下所示的内容：

```js
{
  dev: 2114,
  ino: 48064969,
  mode: 33188,
  nlink: 1,
  uid: 85,
  gid: 100,
  rdev: 0,
  size: 527,
  blksize: 4096,
  blocks: 8,
  atime: Mon, 10 Oct 2011 23:24:11 GMT,
  mtime: Mon, 10 Oct 2011 23:24:11 GMT,
  ctime: Mon, 10 Oct 2011 23:24:11 GMT,
  birthtime: Mon, 10 Oct 2011 23:24:11 GMT
}
```

值得注意的是，这里的 `atime`、`mtime`、`birthtime` 和 `ctime` 都是 Date 对象的实例，需要使用合适的方法来比较相互之间的大小。`getTime()` 方法返回自 1970 年 1 月 1 日至今为止的毫秒数，这个方法基本满足大多数的比较条件。此外还有一些方法可以显示额外的信息，更多方法请参考 [MDN JavaScript Reference](https://developer.mozilla.org/en/JavaScript/Reference/Global_Objects/Date)。

#### Stat 的事件值

stat 对象中的时间值有如下含义：

- `atime`，访问时间，表示文件的最后一次访问时间，调用系统函数 mknod(2)、utimes(2) 和 read(2) 会修改该值。
- `mtime`，内容修改时间，表示文件内容的最后一次修改时间，调用系统函数 mknod(2)、utimes(2) 和 write(2) 会修改该值。
- `ctime`，修改时间，表示文件的最后一次修改时间，包括但不限于内容的修改，调用系统函数 chmod(2)、chown(2)、link(2)、mknod(2)、rename(2)、unlink(2)、utimes(2)、read(2) 和 write(2) 会修改该值。
- `birthtime`，创建时间，表示文件创建的时间，只在文件创建时初始化该值。如果文件的创建时间不可用，那么该值可能是 `ctime` 或 `1970-01-01T00:00Z`。在 Darwin 或其他 FreeBSD 系统中，会通过 `utimes(2)` 将 `atime` 设置的比 `birthtime` 更早。

在 Node.js v0.12 版本之前，Windows 系统的 ctime 函数 birthtime。注意在 Node.js v0.12 中，ctime 并不是文件创建时间，在 Unix 系统中，它一直都不是文件创建时间。

## Class: fs.WriteStream

`WriteStream` 是一个可写的 Stream 实例。

#### 事件：'open'

- `fd`，数值，传递给 WriteStream 的文件描述符的值

当 WriteStream 中的文件打开时，触发该事件。

#### writeStream.bytesWritten

当前写入的字节数，不包含等待写入的数据。

#### writeStream.path

该属性表示表示 stream 写入的文件路径。

## fs.access(path[, mode], callback)

该方法用于测试用户是否对 `path` 指定的文件有足够的权限。可选参数 `mode` 是一个数值，用于指定需要检查的权限。以下变量是 `mode` 的可选值，通过 OR 可以指定多个值：

- `fs.F_OK`，可见权限，常用于检查文件是否存在，但如果文件的权限是 `rwx` 则不会返回任何值，改制为 `mode` 的默认值
- `fs.R_OK`，可读权限
- `fs.W_OK`，可写权限
- `fs.X_OK`，可执行权限，该权限对 Windows 无效

该函数的最后一个参数是一个回调函数 `callback`，其接收一个引用错误信息的参数。如果任一访问权限不足，就会抛出该错误。下面代码检查了当前进程是否可以对文件 `/etc/passwd` 进行读写：

```js
fs.access('/etc/passwd', fs.R_OK | fs.W_OK, (err) => {
  console.log(err ? 'no access!' : 'can read/write');
});
```

## fs.accessSync(path[, mode])

该方法是 `fs.access()` 的同步执行版本，当访问权限不足时会直接抛出错误。

## fs.appendFile(file, data[, options], callback)

- `file`，字符串或数值，文件名或文件描述符
- `data`，字符串或 Buffer 实例
- `options`，对象或字符串
    - `encoding`，字符串或 null，默认值为 `utf-8`
    - `mode`，数值，默认值为 `0o666`
    - `flag`，字符串，默认值为 `a`
- `callback`，函数

该方法以同步的方式向指定文件添加数据，如果文件不存在则先创建该文件再添加内容。`data` 可以是一个字符串或 Buffer 实例：

```js
fs.appendFile('message.txt', 'data to append', (err) => {
  if (err) throw err;
  console.log('The "data to append" was appended to file!');
});
```

如果 `options` 是一个字符串，则用来指定编码格式：

```js
fs.appendFile('message.txt', 'data to append', 'utf8', callback);
```

任何指定的文件描述符必须保持打开用于添加数据。注意，指定的文件描述符不能自动关闭。

## fs.appendFileSync(file, data[, options])

该方法是 `fs.appendFile()` 的同步执行版本，返回 `undefined`。

## fs.chmod(path, mode, callback)

该方法是 `chmod(2)` 的异步执行版本，其回调函数只接收一个引用错误信息的参数。

## fs.chmodSync(path, mode)

该方法是 `chmod(2)` 的同步执行版本，返回 `undefined`。

## fs.chown(path, uid, gid, callback)

该方法是 `chown(2)` 的异步执行版本，其回调函数只接收一个引用错误信息的参数。

## fs.chownSync(path, uid, gid)

该方法是 `chown(2)` 的同步执行版本，返回 `undefined`。

## fs.close(fd, callback)

该方法是 `close(2)` 的异步执行版本，其回调函数只接收一个引用错误信息的参数。

## fs.closeSync(fd)

该方法是 `close(2)` 的同步执行版本，返回 `undefined`。

## fs.createReadStream(path[, options])

该方法返回一个新建的 ReadStream 对象。

注意，虽然默认情况下一个可读的 Stream 实例的大小由 `highWaterMark` （16 kb）决定，但该方法返回的 Stream 对象的默认大小为 64 kb。

`options` 参数是一个对象或字符串，默认值为：

```js
{
  flags: 'r',
  encoding: null,
  fd: null,
  mode: 0o666,
  autoClose: true
}
```

`options` 可以包含 `start` 和 `end` 两个属性，用于从文件中读取指定范围内的内容而不是读取文件的内容。`encoding` 参数可以是任何 Buffer 接收的编码格式。

如果指定了 `fd`，`ReadStream` 将会忽略 `path` 参数并使用该文件描述符，这意味着将不会触发 `open` 事件。注意，这里的 `fd` 应该可以阻塞进程，非阻塞的 `fd` 应该使用 `net.Socket` 处理。

如果 `autoClose` 的值为 false，那么文件描述符就不会被自动关闭，即使执行过程中出现错误也不会关闭。开发者有责任去管理文件描述的关闭和开启，并确保没有遗漏任何文件描述符。`autoClose` 的默认值为 true，当出现 `error` 或读到 `end` 时都会自动关闭文件描述符。

`mode` 用于设置文件模式（权限和粘滞位），但只对已创建的文件有效。

下面代码演示了读取 100 bytes 文件的最后 10 bytes：

```js
fs.createReadStream('sample.txt', {start: 90, end: 99});
```

如果 `options` 参数是一个字符串，则用于指定编码格式。

## fs.createWriteStream(path[, options])

该方法返回一个 `WriteStream` 对象。

`options` 是一个对象或字符串，默认值为：

```js
{
  flags: 'w',
  defaultEncoding: 'utf8',
  fd: null,
  mode: 0o666,
  autoClose: true
}
```

`options` 中可以包含一个 `start` 属性，用于指定数据写入的起始位置。当修改文件的内容时，需要设置 `flag` 为 `r+`，而不是使用默认的模式 `w`。`defaultEncoding` 可以是任何 Buffer 接收的编码格式。

如果指定了 `fd`，`WriteStream` 将会忽略 `path` 参数并使用该文件描述符，这意味着将不会触发 `open` 事件。注意，这里的 `fd` 应该可以阻塞进程，非阻塞的 `fd` 应该使用 `net.Socket` 处理。

如果 `autoClose` 的值为 false，那么文件描述符就不会被自动关闭，即使执行过程中出现错误也不会关闭。开发者有责任去管理文件描述的关闭和开启，并确保没有遗漏任何文件描述符。`autoClose` 的默认值为 true，当出现 `error` 或读到 `end` 时都会自动关闭文件描述符。

如果 `options` 参数是一个字符串，则用于指定编码格式。

## fs.exists(path, callback)

<div class="s s0">
使用 fs.stat() 后 fs.access() 代替。
</div>

## fs.existsSync(path)

<div class="s s0">
使用 fs.statSync() 或 fs.accessSync() 代替。
</div>

## fs.fchmod(fd, mode, callback)

该方法是 `fchmod(2)` 的异步执行版本，其回调函数只接收一个引用错误信息的参数。

## fs.fchmodSync(fd, mode)

该方法是 `fchmod(2)` 的同步执行版本，返回 `undefined`。

## fs.fchown(fd, uid, gid, callback)

该方法是 `fchown(2)` 的异步执行版本，其回调函数只接收一个引用错误信息的参数。

## fs.fchownSync(fd, uid, gid)

该方法是 `fchown(2)` 的同步执行版本，返回 `undefined`。

## fs.fdatasync(fd, callback)

该方法是 `fdatasync(2)` 的异步执行版本，其回调函数只接收一个引用错误信息的参数。

## fs.fdatasyncSync(fd)

该方法是 `fdatasync(2)` 的同步执行版本，返回 `undefined`。

## fs.fstat(fd, mode, callback)

该方法是 `fstat(2)` 的异步执行版本，其回调函数接收两个参数 `(err, stats)`，其中 `stat` 是一个 `fs.Stats` 对象。`fstat()` 等同于 [stat()](https://nodejs.org/dist/latest-v5.x/docs/api/fs.html#fs_fs_stat_path_callback)，唯一不同的这里的文件可以由文件描述符 `fd` 指定。

## fs.fstatSync(fd)

该方法是 `fstat(2)` 的同步执行版本，返回 `fs.Stats` 的实例。

## fs.fsync(fd, callback)

该方法是 `fsync(2)` 的异步执行版本，其回调函数只接收一个引用错误信息的参数。

## fs.fsyncSync(fd)

该方法是 `fsync(2)` 的同步执行版本，返回 `undefined`。

## fs.ftruncate(fd, len, callback)

该方法是 `ftruncate(2)` 的异步执行版本，其回调函数只接收一个引用错误信息的参数。

## fs.ftruncateSync(fd, len)

该方法是 `ftruncate(2)` 的同步执行版本，返回 `undefined`。

## fs.futimes(fd, callback)

该方法根据传入的文件描述符 `fd` 修改文件的时间戳。

## fs.futimesSync(fd)

该方法是 `fs.futimesSync` 的同步执行版本，返回 `undefined`。

## fs.lchmod(path, mode, callback)

该方法是 `lchmod(2)` 的异步执行版本，其回调函数只接收一个引用错误信息的参数。

该方法只在 Mac OS X 下可用。

## fs.lchmodSync(path, mode)

该方法是 `lchmod(2)` 的同步执行版本，返回 `undefined`。

## fs.link(scrpath, dstpath, callback)

该方法是 `link(2)` 的异步执行版本，其回调函数只接收一个引用错误信息的参数。

## fs.linkSync(scrpath, dstpath)

该方法是 `link(2)` 的同步执行版本，返回 `undefined`。

## fs.lstat(path, callback)

该方法是 `lstat(2)` 的异步执行版本，其回调函数接收两个参数 `(err, stats)`，其中 `stat` 是一个 `fs.Stats` 对象。`lstat()` 等同于 [stat()](https://nodejs.org/dist/latest-v5.x/docs/api/fs.html#fs_fs_stat_path_callback)，唯一不同的是，当 `path` 是一个软链接时，显示的是软链接的状态而不是对应文件的状态。

## fs.lstatSync(path)

该方法是 `lstat(2)` 的同步执行版本，返回 `fs.Stats` 的实例。

## fs.mkdir(path[, mode], callback)

该方法是 `mkdir(2)` 的异步执行版本，其回调函数只接收一个引用错误信息的参数，其中 `mode` 的默认值为 `0o777`。

## fs.mkdirSync(path[, mode])

该方法是 `mkdir(2)` 的同步执行版本，返回 `undefined`。

## fs.open(path, flags[, mode], callback)

该方法是 `open(2)` 的异步执行版本，其中 `flags` 包含以下可选值：

- `'r'`，以只读模式打开，如果文件不存在则抛出错误
- `'r+'`，以读写模式代开，如果文件不存在则抛出错误
- `'rs'`，以同步只读模式打开，通知操作系统忽略本地文件系统的缓存。这一功能常用于打开 NFS 挂载的文件，从而忽略可能已失效的文件缓存。同时，这一功能会严重影响 I/O 的性能，所以应该慎用该标志。注意，这里的 `fs.open()` 不是同步阻塞方法，如果你想使用同步执行的方法，请使用 `fs.openSync()`。
- `'rs+'`，以同步读写模式打开文件
- `'w'`，以只写模式打开文件，如果文件不存在则创建该文件，如果文件存在则覆盖该文件
- `'wx'`，类似 'w'，但如果路径不存在则抛出错误
- `'w+'`，以读写模式打开文件，如果文件不存在则创建该文件，如果文件存在则覆盖该文件
- `'wx+'`，类似 'w+'，但如果路径不存在则抛出错误
- `'a'`，以追加数据的模式打开文件，如果文件不存在则创建该文件
- `'ax'`，类似 'a'，但如果路径不存在则抛出错误
- `'a+'`，以读写模式打开文件，如果文件不存在则创建该文件
- `'ax+'`，类似 'a+'，但如果路径不存在则抛出错误

`mode` 参数用于配置文件的权限和粘滞位，且只在文件存在时有效。该参数的默认值为 `0666`，表示可读写。

回调函数接收两个参数 `(err, fd)`。

排异标志 `x`（对应 open(2) 中的 `O_EXCL`）用于确保 `path` 是新建的。在 POSIX 系统中，即使 `path` 是一个软链接且指向不存在的文件，也会被视为存在。该排异标志尚不能确定在网络文件系统是否有效。

`flags` 可以是 `open(2)` 指定的数值。常用的变量可以通过 `require('constants')` 获得。在 Windows 系统中，标志会被转换为 CreateFileW 接受的等效标志，比如，`O_WRONLY` 到 `FILE_GENERIC_WRITE`, or `O_EXCL|O_CREAT` 到 `CREATE_NEW`。

在 Linux 系统中，当文件以添加模式打开时，不能指定写入的位置。kernal 会忽略位置参数，并总是将数据添加到文件的尾部。

## fs.openSync(path, flags[, mode])

该方法是 `fs.open()` 的同步执行版本，返回一个表示文件描述符的数值。

## fs.read(fd, buffer, offset, length, position, callback)

该方法根据 `fd` 从文件中读取数据。

`buffer` 是一个 Buffer 实例，用于存储从文件中读取到的数据。

`offset` 表示向 Buffer 实例写入数据的起始位置。

`length` 是一个数值，用于表示读取的字节量。

`position` 是一个数值，用于表示文件读取的起始位置，如果该值为 `null`，则从文件的当前位置开始读取。

`callback` 是一个回调函数，接收三个参数 `(err, bytesRead, buffer)`。

## fs.readdir(path, callback)

该方法是 `readdir(3)` 的异步执行版本，用于读取一个目录的内容。`callback` 接收两个参数 `(err, files)`，其中 `files` 是一个数组，数组成员为当前目录下的文件名，不包含 `.` 和 `..`。

## fs.readdirSync(path)

该方法是 `readdir(3)` 的同步执行版本，返回一个不包含 `.` 和 `..` 的文件名数组。

## fs.readFile(file[, options], callback)

- `file`，字符串或数值，文件名或文件描述符
- `options`，对象或字符串
    - `encoding`，字符串或 null，默认值为 null
    - `flag`，字符串，默认值为 `r`
- `callback`，函数

该方法以异步方式读取文件的整个内容：

```js
fs.readFile('/etc/passwd', (err, data) => {
  if (err) throw err;
  console.log(data);
});
```

回调函数接收两个参数 `(err, data)`，其中 `data` 是文件的内容。

如果未指定 `encoding`，则返回原始的 Buffer 数据。

如果 `options` 是一个字符串，则该字符串表示编码格式：

```js
fs.readFile('/etc/passwd', 'utf8', callback);
```

任何指定的文件描述符必须支持可读模式。注意，指定的文件描述符不会自动关闭。

## fs.readFileSync(file[, options])

该方法是 `fs.readFile` 的同步执行版本，返回 `file` 的内容。

如果指定了可选参数 `encoding`，则该方法返回一个字符串，否则返回 Buffer 数据。

## fs.readlink(path, callback)

该方法是 `readlink(2)` 的异步执行版本，其中回调函数接收两个参数 `(err, linkString)`。

## fs.readlinkSync(path)

该方法是 `readlink(2)` 的同步执行版本，以字符串形式返回软链接的值。

## fs.realpath(path[, cache], callback)

该方法是 `realpath(2)` 的异步执行版本，其中 `callback` 接收两个参数 `(err, resolvedPath)`。可以使用 `process.cwd` 解析相对路径。`cache` 是一个映射路径的字面量，常用于强制解析路径或避免对已知真实路径调用额外的 `fs.stat`：

```js
var cache = {'/etc':'/private/etc'};
fs.realpath('/etc/passwd', cache, (err, resolvedPath) => {
  if (err) throw err;
  console.log(resolvedPath);
});
```

## fs.readSync(fd, buffer, offset, length, position)

该方法是 `fs.read()` 的同步执行版本，返回 `bytesRead` 的数值。

## fs.realpathSync(path[, cache])

该方法是 `realpath(2)` 的同步执行版本，返回解析后的路径。`cache` 是一个映射路径的字面量，常用于强制解析路径或避免对已知真实路径调用额外的 `fs.stat`。

## fs.rename(oldPath, newPath, callback)

该方法是 `rename(2)` 的异步执行版本，其回调函数只接收一个引用错误信息的参数

## fs.rename(oldPath, newPath)

该方法是 `rename(2)` 的同步执行版本，返回 `undefined`。

## fs.rmdir(path, callback)

该方法是 `rmdir(2)` 的异步执行版本，其回调函数只接收一个引用错误信息的参数

## fs.rmdir(path)

该方法是 `rmdir(2)` 的同步执行版本，返回 `undefined`。

## fs.stat(path, callback)

该方法是 `stat(2)` 的异步执行版本，其回调函数接收两个参数 `(err, stats)`，其中 `stats` 是一个 `fs.Stats` 对象。更多信息请参考 [fs.Stats](https://nodejs.org/dist/latest-v5.x/docs/api/fs.html#fs_class_fs_stats)。

## fs.statSync(path)

该方法是 `stat(2)` 的同步执行版本，返回一个 `fs.Stats` 的实例。

## fs.symlink(target, path[, type], callback)

该方法是 `symlink(2)` 的异步执行版本，其回调函数只接收一个引用错误信息的参数。`type` 的只可以是 'dir'、'file' 或 'junction'，默认值为 'file'，且该参数只适用于 Windows 系统，在其他平台会被忽略。注意，Windows 连接点必须是绝对路径。当使用 `junction` 时，`target` 参数会被自动解析为绝对路径：

```js
fs.symlink('./foo', './new-port');
```

上面代码创建了一个名为 `new-port` 指向 `foo` 的软链接。

## fs.symlinkSync(target, path[, type])

该方法是 `symlink(2)` 的同步执行版本，返回 `undefined`。

## fs.truncate(path, len, callback)

该方法是 `truncate(2)` 的异步执行版本，其回调函数只接收一个引用错误信息的参数。当第一个参数是文件描述符时，系统调用 `fs.ftruncate()`。

## fs.truncateSync(path, len)

该方法是 `truncate(2)` 的同步执行版本，返回 `undefined`。

## fs.unlink(path, callback)

该方法是 `unlink(2)` 的异步执行版本，其回调函数只接收一个引用错误信息的参数

## fs.unlink(path)

该方法是 `unlink(2)` 的同步执行版本，返回 `undefined`。

## fs.unwatchFile(filename[, listener])

该方法用于停止监控 `filename` 的修改。如果指定了 `listener`，则值移除该 `listener`，否则则移除所有的 `listener`，完全地停止监控 `filename` 的修改。

调用 `fs.unwatchFile()` 停止监视未被监视的文件不会触发错误，本质上只是一个无效操作。

注意，`fs.watch()` 比 `fs.watchFile()` 和 `fs.unwatchFile()` 更加高效。应尽可能使用 `fs.watch()` 替代 `fs.watchFile()` 和 `fs.unwatchFile()`。

## fs.utimes(path, atime, mtime, callback)

该方法根据指定的 `path` 修改文件的时间戳。

注意，参数 `atime` 和 `mtime` 遵循以下规则：

- 如果它们的值是字符串类型的数值，比如 '123456789'，那么会被自动转换为数值类型
- 如果它们的值是 `NaN` 或 `Infinity`，那么它们就会被 `Date.now()` 的值覆盖。

## fs.utimesSync(path, atime, mtime)

该方法是 `fs.utimes()` 的同步执行版本，返回 `undefined`。

## fs.watch(filename[, options][, listener])

该方法用于监视 `filename` 的变动，其中 `filename` 可以是一个文件也可以是一个目录，返回值为 `fs.FSWatcher` 对象。

该方法的第二个参数 `options` 为对象类型的可选参数，该对象包含两个属性 `persistent` 和 `recursive`，均为布尔值。`persistent` 用于指定是否让进程持续运行。`recursive` 用于指定是监控所有的子目录还是之间空当前目录。该参数只有当指定的目标是目录时才会生效，且只支持部分平台（详见下文的 Caveats）。

`options` 的默认值是是 `{ persistent: true, recursive: false }`。

用作监听器的回调函数接收两个参数 `(event, filename)`，其中 `event` 要么是 `rename` 要么就是 `change`，`filename` 则是触发事件的文件名。

#### Caveats

`fs.watch` 方法在不同的平台上的表现并不一致，甚至在某些情况下是不可用的。

其中递归操作只适用于 OS X 和 Windows 系统。

#### Availability

这些特性依赖于操作系统底层提供的文件变动的通知接口：

- 在 Linux 系统中，使用 `inotify`
- 在 BSD 系统中，使用 `kqueue`
- 在 OS X 系统中，对文件使用 `kqueue`，对目录使用 `FSEvents`
- 在 SunOS 系统中（包括 Solaris 和 SmartOS），使用 `event ports`
- 在 Windows 系统中，该特性取决于 `ReadDirectoryChangesW`

如果因为某些原因导致底层接口不可用，那么 `fs.watch` 也将不可用。举例来说，在网络文件系统中（NFS/SMB 等），对文件变动的监视往往收效甚微。

开发者仍可使用 `fs.watchFile`，该方法使用了状态轮询，所以较慢和不可靠。

#### 文件名参数

回调函数接收的 `filename` 参数只在 Linux 和 Windows 系统中有效。即使在支持该参数的平台上，也无法完全保证可以获得 `filename`。因此，永远不要假想一定可以获得 `filename` 属性，并且为 `filename` 为 null 的情况设置降级措施：

```js
fs.watch('somedir', (event, filename) => {
  console.log(`event is: ${event}`);
  if (filename) {
    console.log(`filename provided: ${filename}`);
  } else {
    console.log('filename not provided');
  }
});
```

## fs.watchFile(filename[, options], listener)

该方法用于监视 `filename` 的变动，每当 `filename` 被访问时都会调用一次回调函数 `listener`。

可选参数 `options` 是一个对象，可以忽略。`options` 可以包含一个布尔值属性 `persistent`，用于指定是否让进程持续运行。`options` 还可以包含一个 `interval` 属性，用于指定轮询周期。`options` 的默认值是 `{ persistent: true, interval: 5007 }`。

回调函数 `listener` 接收两个参数，分别是当前状态和先前状态：

```js
fs.watchFile('message.text', (curr, prev) => {
  console.log(`the current mtime is: ${curr.mtime}`);
  console.log(`the previous mtime was: ${prev.mtime}`);
});
```

这里的状态对象都属于 `fs.Stat` 的实例。

如果你想同时监听文件访问和文件修改事件，可以通过比较 `curr.mtime` 和 `prev.mtime` 来实现。

注意，当 `fs.watchFile` 抛出 `ENOENT` 错误时，那么就会只调用一次监听器，且所有字段被重置为 0。在 Windows 系统中，`blksize` 和 `blocks` 会被赋值为 `undefined`，而不是 0。如果随后创建了文件，那么就会再次调用监听器设置状态对象。Node.js v0.10 之后支持该功能。

此外，值得注意的是，`fs.watch()` 比 `fs.watchFile()` 和 `fs.unwatchFile()` 更加高效。应尽可能使用 `fs.watch()` 替代 `fs.watchFile()` 和 `fs.unwatchFile()`。

## fs.write(fd, buffer, offset, length[, position], callback)

该方法用于向指定的 `fd` 写入 Buffer 数据。

`offset` 和 `length` 用于提取 `buffer` 中指定位置的数据。

`position` 指定在文件中开始写入的位置。如果 `typeof position !== 'number'`，那么就会从当前位置开始写入，更多信息请参考 [pwrite(2)](http://man7.org/linux/man-pages/man2/pwrite.2.html)。

回调函数 `callback` 接收三个参数 `(err, written, buffer)`，其中 `written` 用于指定写入的字节量。

注意，如果未执行完回调函数就多次执行 `fs.write` 是不安全的，对于这种需求，建议使用 `fs.createWriteStream`。

在 Linux 系统中，当文件以追加模式打开时，不能指定写入的位置。kernel 会忽略位置参数，并总是将数据追加到文件的尾部。

## fs.write(fd, data[, position[, encoding]], callback)

该方法用于向指定的 `fd` 写入 `data`。如果 `data` 是一个 Buffer 实例，则内容会被强制转换为字符串格式。

`position` 指定在文件中开始写入的位置。如果 `typeof position !== 'number'`，那么就会从当前位置开始写入，更多信息请参考 [pwrite(2)](http://man7.org/linux/man-pages/man2/pwrite.2.html)。

`encoding` 用于指定字符串的编码格式。

回调函数 `callback` 接收三个参数 `(err, written, buffer)`，其中 `written` 用于指定写入的字节量。

与写入 `buffer` 数据不同的是，如果数据是字符串，则必须一次性全部写入，这是因为字节的偏移量和字符的偏移量是不同的。

注意，如果未执行完回调函数就多次执行 `fs.write` 是不安全的，对于这种需求，建议使用 `fs.createWriteStream`。

在 Linux 系统中，当文件以追加模式打开时，不能指定写入的位置。kernel 会忽略位置参数，并总是将数据追加到文件的尾部。

## fs.writeFile(file, data[, options], callback)

- `file`，字符串或数值，传入的文件名或文件描述符
- `data`，字符串或 Buffer 实例
- `options`，对象或字符串
    - `encoding`，字符串或 null，默认值为 `utf-8`
    - `mode`，数值，默认值为 `0o66`
    - `flag`，字符串，默认值为 `w`
- `callback`，函数

该方法以异步执行的方式向文件中写入数据，如果文件已存在，则替换该文件。`data` 可以是字符串或 Buffer 实例。

如果 `data` 是 Buffer 实例，则 `encoding` 参数，该参数的默认值为 `utf8`。

```js
fs.writeFile('message.txt', 'Hello Node.js', (err) => {
  if (err) throw err;
  console.log('It\'s saved!');
});
```

如果 `options` 是一个字符串，则用于指定编码格式；

```js
fs.writeFile('message.txt', 'Hello Node.js', 'utf8', callback);
```

传入的文件描述符必须允许写入。

注意，如果未执行完回调函数就多次执行 `fs.write` 是不安全的，对于这种需求，建议使用 `fs.createWriteStream`。此外，还需注意指定的文件描述符并不会自动关闭。

## fs.writeFileSync(file, data[, options])

该方法是 `fs.writeFile()` 的同步执行版本，返回 `undefined`。

## fs.writeSync(fd, buffer, offset, length[, position])
## fs.writeSync(fd, data[, position[, encoding]])

该方法是 `fs.write()` 的同步执行版本，返回一个数值，该数值表示成功写入的字节量。

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