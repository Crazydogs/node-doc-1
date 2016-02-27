## 操作系统

<div class="s s2"></div>

该模块提供了一些和操作系统有关的辅助函数，使用 `require('os')` 可以加载该模块。

## os.EOL

常量，返回当前操作系统的换行符。

## os.arch()

返回当前操作系统的 CPU 架构，常见值包括：`x64` / `arm` 和 `ia32`，实际上该值引用的是 `process.arch`。

## os.cpus()

返回一个对象数组，每一个对象描述一个 CPU 的数据：

下面是一个 `os.cpus()` 的返回信息：型号，速度和使用时间。

```js
[ { model: 'Intel(R) Core(TM) i7 CPU         860  @ 2.80GHz',
    speed: 2926,
    times:
     { user: 252020,
       nice: 0,
       sys: 30340,
       idle: 1070356870,
       irq: 0 } },
  { model: 'Intel(R) Core(TM) i7 CPU         860  @ 2.80GHz',
    speed: 2926,
    times:
     { user: 306960,
       nice: 0,
       sys: 26980,
       idle: 1071569080,
       irq: 0 } },
  { model: 'Intel(R) Core(TM) i7 CPU         860  @ 2.80GHz',
    speed: 2926,
    times:
     { user: 248450,
       nice: 0,
       sys: 21750,
       idle: 1070919370,
       irq: 0 } },
  { model: 'Intel(R) Core(TM) i7 CPU         860  @ 2.80GHz',
    speed: 2926,
    times:
     { user: 256880,
       nice: 0,
       sys: 19430,
       idle: 1070905480,
       irq: 20 } },
  { model: 'Intel(R) Core(TM) i7 CPU         860  @ 2.80GHz',
    speed: 2926,
    times:
     { user: 511580,
       nice: 20,
       sys: 40900,
       idle: 1070842510,
       irq: 0 } },
  { model: 'Intel(R) Core(TM) i7 CPU         860  @ 2.80GHz',
    speed: 2926,
    times:
     { user: 291660,
       nice: 0,
       sys: 34360,
       idle: 1070888000,
       irq: 10 } },
  { model: 'Intel(R) Core(TM) i7 CPU         860  @ 2.80GHz',
    speed: 2926,
    times:
     { user: 308260,
       nice: 0,
       sys: 55410,
       idle: 1071129970,
       irq: 880 } },
  { model: 'Intel(R) Core(TM) i7 CPU         860  @ 2.80GHz',
    speed: 2926,
    times:
     { user: 266450,
       nice: 1480,
       sys: 34920,
       idle: 1072572010,
       irq: 30 } } ]
```

注意，其中 `nice` 的值只对 UNIX 有效，在 Windows 中该值为 0。

## os.endianness()

返回 CPU 的字节序，如果是大端字节序，返回 `BE`，如果是小端字节序，则返回 `LE`。

## os.freemem()

返回系统空闲内存的大小，单位是字节。

## os.homedir()

返回当前用户的根目录。

## os.hostname()

返回当前操作系统的主机名。

## os.loadavg()

返回一个数组，包含 1，5 和 15 分钟内的负载均衡信息。负载均衡在一定程度上反映了操作系统的活动情况，它的值有操作系统计算得出，通常数值较小。一般来说，负载均衡都会比操作系统中的 CPU 数量小。

负载均衡的概念仅存在于 UNIX 系统中，在 Windows 平台中，该值为 `[0, 0, 0]`。

## os.networkInterfaces()

返回网络接口的列表：

```js
{ lo:
   [ { address: '127.0.0.1',
       netmask: '255.0.0.0',
       family: 'IPv4',
       mac: '00:00:00:00:00:00',
       internal: true },
     { address: '::1',
       netmask: 'ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff',
       family: 'IPv6',
       mac: '00:00:00:00:00:00',
       internal: true } ],
  eth0:
   [ { address: '192.168.1.108',
       netmask: '255.255.255.0',
       family: 'IPv4',
       mac: '01:02:03:0a:0b:0c',
       internal: false },
     { address: 'fe80::a00:27ff:fe4e:66a1',
       netmask: 'ffff:ffff:ffff:ffff::',
       family: 'IPv6',
       mac: '01:02:03:0a:0b:0c',
       internal: false } ] }
```

注意，由于底层实现的原因，该方法只会返回已分配地址的网络接口。

## os.platform()

放回操作系统平台，常见值包括：`darwin` / `freebsd` / `linux` / `sunos` 和 `win32`，实际上该值引用的是 `process.platform()`。

## os.release()

返回操作系统的分发版本号。

## os.tmpdir()

返回操作系统默认的临时文件夹。

## os.totalmem()

返回操作系统内存空间的大小，单位是字节。

## os.type()

返回操作系统的名称，比如 Linux 的名称是 `Linux`，OS X 的名称是 `Darwin`，Windows 的名称是 `Windows_NT`。

## os.uptime()

返回操作系统的运行时间，单位是秒。


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