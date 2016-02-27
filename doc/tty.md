## TTY

<div class="s s2"></div>

TTY 模块包含 `tty.ReadStream` 和 `tty.WriteStream` 两个类。在大多数情况下，开发者都需要直接调用该模块。

当 Node.js 检测到当前环境处于 TTY 上下文时，则 `process.stdin` 会指向一个 `tty.ReadStream` 实例，`process.stdout` 会指向一个 `tty.WriteStream` 实例。检测 Node.js 是否处于 TTY 上下文有一个更方便的方式，那就是检查 `process.stdout.isTTY`：

```bash
$ node -p -e "Boolean(process.stdout.isTTY)"
true
$ node -p -e "Boolean(process.stdout.isTTY)" | cat
false
```

## Class: ReadStream

该类是 `net.socket` 的子类，表示 TTY 中的可读部分。在大多数情况下，任何 Node.js 程序（`isatty(0)` 为 true）中的 `process.stdin` 都是唯一的 `tty.ReadStream` 实例。 

#### rs.isRaw

布尔值，默认值为 false，标示当前 `tty.ReadStream` 实例是否为 `raw` 状态。

#### rs.setRawMode(mode)

`mode` 的值必须为 true 或者 false，该值决定了 `tty.ReadStream` 为原始设备或默认设置，并且它也决定了 `isRaw` 的值。

## Class: WriteStram

该类是 `net.socket` 的子类，表示 TTY 中的可写部分。在大多数情况下，任何 Node.js 程序（`isatty(0)` 为 true）中的 `process.stdout` 都是唯一的 `tty.WriteStream` 实例。 

#### 事件：'resize'

`columns` 或 `rows` 属性变化时，`refreshSize()` 方法就会触发该事件：

```js
process.stdout.on('resize', () => {
  console.log('screen size has changed!');
  console.log(`${process.stdout.columns}x${process.stdout.rows}`);
});
```

#### ws.columns

数值，标示 TTY 当前的列数，该属性会被 `resize` 事件更新。

#### ws.rows

数值，标示 TTY 当前的行数，该属性会被 `resize` 事件更新。

## tty.isatty(fd)

如果 `fd` 参数和终端有关，那么该方法返回 `true`，否则返回 `false`。

## tty.setRawMode(mode)

<div class="s s0">
该方法已被废除，请使用 `tty.ReadStream#setRawMode()` 替代，比如 `process.stdin.setRawMode`。
</div>

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