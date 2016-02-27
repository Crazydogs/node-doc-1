## 定时器

<div class="s s3"></div>

定时器模块中的所有方法都是全局方法，无需使用 `require()` 引入即可使用。

## clearImmediate(immediateObject)

取消一个 immediate 定时器。

## clearInterval(intervalObject)

取消一个 interval 定时器。

## clearTimeout(timeoutObject)

取消一个 timeout 定时器。

## ref()

如果之前 `unref()` 了定时器，可以使用 `ref()` 请求定时器保持打开状态。如果之前已经使用 `ref()` 掉用过定时器，则再次调用无效。 

该方法返回一个定时器。

## setImmediate(callback[, arg][, arg][, ...])

该方法用于设定在 I/O 事件回调之后、在 setTimeout 和 setInterval 之前触发的定时器。返回值 `immediateObject` 常用于 `clearImmediate()` 方法取消 immediate 定时器。此外，该方法接收不定量的可选参数，这些参数会被传递给回调函数，作为回调函数的参数。

回调函数根据它们创建的顺序依次加入到回调队列之中。回调队列在事件循环中遍历执行，如果向正在执行的回调中加入一个 immediate 定时器，那么该定时器不会立即执行，而是等到下一轮事件循环时执行。

## setInterval(callback, delay[, arg][, ...])

该方法用于设定每隔 `delay` 毫秒执行一次的 `callback` 回调函数。返回值 `intervalObject` 常用于 `clearInterval()` 方法取消 interval 定时器。此外，该方法接收不定量的可选参数，这些参数会被传递给回调函数，作为回调函数的参数。

根据浏览器的表现来看，如果 `delay` 大于 2147483647 毫秒或小于 1，该值将会被重置为 1。

## setTimeout(callback, delay[, arg][, ...])

该方法用于设定 `delay` 毫秒后执行 `callback` 回调函数。返回值 `timeoutObject` 常用于 `clearTimeout()` 方法取消 timeout 定时器。此外，该方法接收不定量的可选参数，这些参数会被传递给回调函数，作为回调函数的参数。

该回调函数延迟触发的时间可能并不等于 `delay`。Node.js 无法保证回调函数准确触发的时间以及触发的顺序，只能保证回调函数触发的时间尽可能接近 `delay` 时间。

根据浏览器的表现来看，如果 `delay` 大于 2147483647 毫秒或小于 1，则该定时器会被立即执行，且该值将会被重置为 1。

## unref()

`setTimeout()` 和 `setInterval()` 方法返回的值都拥有一个实例方法 `timer.unref()`，该方法允许开发者创建一个定时器，但如果事件循环队列中只有这一个定时器，其将不会执行。如果定时器已经调用过 `unref()`，则再次调用时没有效果。

使用 `unref()` 创建的 `setTimeout` 定时器是一个独立的定时器，它会唤醒事件循环机制，创建过多的此类定时器会影响事件循环机制的性能。

该方法返回一个定时器。 

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