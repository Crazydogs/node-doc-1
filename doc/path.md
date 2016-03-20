## Path

<div class="s s2"></div>

该模块封装了一些处理和转换文件路径的辅助函数，其中大多数主要用于对路径字符串的转换。文件系统不会去校验路径是否有效。

通过 `require('path')` 可以加载该模块。

## path.basname(p[, ext])

该方法返回路径的最后一部分，类似于 Unix 中的 `basename` 命令：

```js
path.basename('/foo/bar/baz/asdf/quux.html')
// returns 'quux.html'

path.basename('/foo/bar/baz/asdf/quux.html', '.html')
// returns 'quux'
```

## path.delimiter

该方法根据当前平台特性返回路径定界符，比如 `;` 或 `:`。

下面代码演示了在 *nix 下的执行结果：

```js
console.log(process.env.PATH)
// '/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/bin'

process.env.PATH.split(path.delimiter)
// returns ['/usr/bin', '/bin', '/usr/sbin', '/sbin', '/usr/local/bin']
```

下面代码演示了在 Windows 下的执行结果：

```js
console.log(process.env.PATH)
// 'C:\Windows\system32;C:\Windows;C:\Program Files\node\'

process.env.PATH.split(path.delimiter)
// returns ['C:\\Windows\\system32', 'C:\\Windows', 'C:\\Program Files\\node\\']
```

## path.dirname(p)

该方法返回一个路径的目录名，类似于 Unix 的 `dirnaem` 命令：

```js
path.dirname('/foo/bar/baz/asdf/quux')
// returns '/foo/bar/baz/asdf'
```

## path.extname(p)

该方法返回路径的扩展名，该扩展名为路径的最后一个 `.` 到结尾部分的字符串。如果路径最后一部分中没有 `.`，或者第一个字符就是 `.`，则返回一个空字符串：

```js
path.extname('index.html')
// returns '.html'

path.extname('index.coffee.md')
// returns '.md'

path.extname('index.')
// returns '.'

path.extname('index')
// returns ''

path.extname('.index')
// returns ''
```

## path.format(pathObject)

该方法根据传入的 `pathObject` 返回一个字符串形式的路径，与之相对的方法是 `path.parse`：

```js
path.format({
    root : "/",
    dir : "/home/user/dir",
    base : "file.txt",
    ext : ".txt",
    name : "file"
})
// returns '/home/user/dir/file.txt'
```

## path.isAbsolute(path)

该方法用于判断一个路径是否是绝对路径。绝对路径不需要依赖当前共走路径即可定位位置。

在 POSIX 系统下：

```js
path.isAbsolute('/foo/bar') // true
path.isAbsolute('/baz/..')  // true
path.isAbsolute('qux/')     // false
path.isAbsolute('.')        // false
```

在 Windows 系统下：

```js
path.isAbsolute('//server')  // true
path.isAbsolute('C:/foo/..') // true
path.isAbsolute('bar\\baz')  // false
path.isAbsolute('.')         // false
```

注意，如果传入的路径是一个空字符串，与其他 path 模块的方法所不同的是，该方法会返回 `false`。

## path.join([path1][, path2][, ...])

该方法拼接所有的参数，然后标准化出一个路径。

该方法中的参数必须是字符串。在 v0.8 中，非字符串参数会被忽略，但在 v0.10+ 中，会抛出错误。

```js
path.join('/foo', 'bar', 'baz/asdf', 'quux', '..')
// returns '/foo/bar/baz/asdf'

path.join('foo', {}, 'bar')
// throws exception TypeError: Arguments to path.join must be strings
```

注意，如果传入的路径是一个空字符串，与其他 path 模块的方法所不同的是，该方法会返回 `.`（代表当前路径）。

## path.normalize(p)

该方法用于标准化路径，注意 `..` 和 `.` 部分。

如果路径中存在多个斜线，则会被替换为单个斜线；如果路劲尾部存在斜线，则会被保留。在 Windows 中路径使用反斜线分隔。

```js
path.normalize('/foo/bar//baz/asdf/quux/..')
// returns '/foo/bar/baz/asdf'
```

注意，如果传入的路径是一个空字符串，该方法会返回 `.`（代表当前路径）。

## path.parse(pathString)

该方法根据字符串路径返回一个对象。

在 *nix 系统下：

```js
path.parse('/home/user/dir/file.txt')
// returns
{
    root : "/",
    dir : "/home/user/dir",
    base : "file.txt",
    ext : ".txt",
    name : "file"
}
```

在 Windows 系统下：

```js
path.parse('C:\\path\\dir\\index.html')
// returns
{
    root : "C:\\",
    dir : "C:\\path\\dir",
    base : "index.html",
    ext : ".html",
    name : "index"
}
```

## path.posix

该属性决定是否以 POSX 兼容模式执行上述路径处理方法。

## path.relative(from, to)

该方法根据 `from` 和 `to` 解析相对路径。

该方法根据传入的两个绝对路径，计算它们的相对路径。该方法实际上是 `path.resolve` 的逆方法：`path.resolve(from, path.relative(from, to)) == path.resolve(to)`。

```js
path.relative('C:\\orandea\\test\\aaa', 'C:\\orandea\\impl\\bbb')
// returns '..\\..\\impl\\bbb'

path.relative('/data/orandea/test/aaa', '/data/orandea/impl/bbb')
// returns '../../impl/bbb'
```

注意，如果传入的路径存在空字符串，该空字符串会被 `.`（代表当前路径）代替。如果两个路径相同，则返回一个空字符串。

## path.resolve([from...], to)

该方法将 `to` 解析为绝对路径。

如果 `to` 不是相对 `from` 的绝对路径，那么 `to` 就会被添加到 `from` 的右侧，直到找到一个而绝对路径。如果用尽所有的 `from` 都没有找到绝对路径，就会使用当前工作目录。最终的路径是标准化之后，已经去除了斜线，除非结果是根目录。非字符串的 `from` 参数会被忽略。

下面是一个该犯法的用例：

```js
path.resolve('foo/bar', '/tmp/file/', '..', 'a/../subfile')
```

类似于：

```js
cd foo/bar
cd /tmp/file/
cd ..
cd a/../subfile
pwd
```

不同之处在于，不同的路径可以不存在或者是文件。

```js
path.resolve('/foo/bar', './baz')
// returns
'/foo/bar/baz'

path.resolve('/foo/bar', '/tmp/file/')
// returns
'/tmp/file'

path.resolve('wwwroot', 'static_files/png/', '../gif/image.gif')
// if currently in /home/myself/node, it returns
'/home/myself/node/wwwroot/static_files/gif/image.gif'
```

注意，如果传入的路径存在空字符串，该空字符串会被 `.`（代表当前路径）代替。

## path.sep

该属性表示当前平台的文件分隔符是 '\\' 还是 '/'。

在 *nix 系统下：

```js
'foo/bar/baz'.split(path.sep)
// returns ['foo', 'bar', 'baz']
```

在 Windows 系统下：

```js
'foo\\bar\\baz'.split(path.sep)
// returns ['foo', 'bar', 'baz']
```

## path.win32

该属性决定是否以 win32 兼容模式执行上述路径处理方法。

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