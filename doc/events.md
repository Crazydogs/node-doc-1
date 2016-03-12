## 事件

<div class="s s2"></div>

Node.js 的核心 API 大都是基于异步事件驱动架构（asynchronous event-driven architecture）构建的，在这一架构中，触发器（emitter）定期触发某些事件执行监听对象（listener）。

举例来说，每次有连接时，`net.Server` 就会触发一个事件；当文件打开时，`fs.ReadStream` 就会触发一个事件；当数据可访问时，`stream` 就会触发一个事件。

所有可以触发事件的对象都是 `EventEmitter` 类的实例。这些对象通过 `eventEmitter.on()` 函数监听特定的事件。通常来说，事件名称统一使用驼峰字符串表示，但实际上任何有效的 JavaScript 键名都可以做事件名。

当 `EventEmitter` 的实例对象触发事件之后，绑定在该事件之下的所有函数会被以同步执行的方式调用。任何监听器返回的值都会被忽略和抛弃。

下面代码演示了一个简单的 `EventEmitter` 实例及其监听的事件。`eventEmitter.on()` 方法用于注册监听器，`eventEmitter.emit()` 方法用于触发事件。

```js
const EventEmitter = require('events');
const util = require('util');

function MyEmitter() {
  EventEmitter.call(this);
}
util.inherits(MyEmitter, EventEmitter);

const myEmitter = new MyEmitter();
myEmitter.on('event', () => {
  console.log('an event occurred!');
});
myEmitter.emit('event');
```

通过继承，任何对象都可以成为 `EventEmitter` 类的实例。上面代码通过 `util.inherits()` 方法使用传统的原型继承实现了继承机制。此外,也可以使用 ES6 的类来实现：

```js
const EventEmitter = require('events');

class MyEmitter extends EventEmitter {}

const myEmitter = new MyEmitter();
myEmitter.on('event', () => {
  console.log('an event occurred!');
});
myEmitter.emit('event');
```

## 向监听器传递参数和上下文

`eventEmitter.emit()` 方法允许向监听器函数传递一组参数。值得注意的是，`EventEmitter` 调用监听函数时，`this` 仍然指向监听器：

```js
const myEmitter = new MyEmitter();
myEmitter.on('event', function(a, b) {
  console.log(a, b, this);
    // Prints:
    //   a b MyEmitter {
    //     domain: null,
    //     _events: { event: [Function] },
    //     _eventsCount: 1,
    //     _maxListeners: undefined }
});
myEmitter.emit('event', 'a', 'b');
```

虽然可以使用 ES6 的箭头函数创建监听器，但是这样的话，`this` 将会不再指向 `EventEmitter` 的实例对象：

```js
const myEmitter = new MyEmitter();
myEmitter.on('event', (a, b) => {
  console.log(a, b, this);
    // Prints: a b {}
});
myEmitter.emit('event', 'a', 'b');
```

## 异步 VS 同步

`EventListener` 会根据监听顺序以同步的方式调用所有的监听函数。这有助于确保事件触发的合理顺序，避免竞争条件和逻辑错误。在某些情况下，我们可以通过使用 `setImmediate()` 或 `process.nextTick()` 方法以异步的方式调用监听函数：

```js
const myEmitter = new MyEmitter();
myEmitter.on('event', (a, b) => {
  setImmediate(() => {
    console.log('this happens asynchronously');
  });
});
myEmitter.emit('event', 'a', 'b');
```

## 一次性事件

使用 `eventEmitter.on()` 方法注册的监听器对事件的每一次触发做出响应：

```js
const myEmitter = new MyEmitter();
var m = 0;
myEmitter.on('event', () => {
  console.log(++m);
});
myEmitter.emit('event');
// Prints: 1
myEmitter.emit('event');
// Prints: 2
```

使用 `eventEmitter.once()` 方法注册的监听器在事件第一次被触发后就会注销该事件：

```js
const myEmitter = new MyEmitter();
var m = 0;
myEmitter.once('event', () => {
  console.log(++m);
});
myEmitter.emit('event');
// Prints: 1
myEmitter.emit('event');
// Ignored
```

## 错误事件

当 `EventEmitter` 的实例出现错误时，就会触发 `error` 事件，这在 Node.js 中被视为一个特殊事件。

如果 `EventEmitter` 的实例没有监听 `error` 事件，就会将该错误向上传播，直到抛出该错误，打印出堆栈跟踪信息，最终退出当前的 Node.js 进程：

```js
const myEmitter = new MyEmitter();
myEmitter.emit('error', new Error('whoops!'));
// Throws and crashes Node.js
```

为了避免 Node.js 的进程崩溃，开发者可以使用 `domain` 模块（不过该模块已被抛弃）或者注册一个 `process.on('uncaughtException')` 来解决:

```js
const myEmitter = new MyEmitter();

process.on('uncaughtException', (err) => {
  console.log('whoops! there was an error');
});

myEmitter.emit('error', new Error('whoops!'));
// Prints: whoops! there was an error
```

最好的预防措施就是，开发者在使用监听器时始终都应该监听 `error` 事件：

```js
const myEmitter = new MyEmitter();
myEmitter.on('error', (err) => {
  console.log('whoops! there was an error');
});
myEmitter.emit('error', new Error('whoops!'));
// Prints: whoops! there was an erro
```

## Class: EventEmitter

`EventEmitter` 类定义在 `events` 模块中：

```js
const EventEmitter = require('events');
```

所有的事件触发器在注册监听器的时候都会触发 `newListener` 事件，在注销监听器的时候都会触发 `removeListener` 事件。

#### 事件：'newListener'

- `event`，字符串或 Symbol，事件名称
- `listener`，函数，事件处理函数

`EventEmitter` 的实例在监听器被添加到监听列表之前就会首先触发 `newListener` 事件。

`newListener` 时间的监听器接收两个参数，一个是事件名称，另一个是已添加的监听器。

实际上，在添加监听器之前触发该事件有一点负面影响：在 `newListener` 事件中注册的同名事件，将会比外部注册的事件晚触发：

```js
const myEmitter = new MyEmitter();
// Only do this once so we don't loop forever
myEmitter.once('newListener', (event, listener) => {
  if (event === 'event') {
    // Insert a new listener in front
    myEmitter.on('event', () => {
      console.log('B');
    });
  }
});
myEmitter.on('event', () => {
  console.log('A');
});
myEmitter.emit('event');
// Prints:
//   B
//   A
```

#### 事件：'removeListener'

- `event`，字符串或 Symbol，事件名称
- `listener`，函数，事件处理函数

注销监听器之后触发该事件。

#### EventEmitter.listenerCount(emitter, event)

<div class="s s0">
请使用 emitter.listenerCount() 代替该方法。
</div>

#### EventEmitter.defaultMaxListeners

默认情况下，任何事件可以注册 10 个监听器。`EventEmitter` 的实例可以通过 `emitter.setMaxListeners(n)` 方法修改这一限制值。如果要为所有的 `EventEmitter` 实例修改该属性，需要使用 `EventEmitter.defaultMaxListeners` 属性。

需要注意的是，使用 `EventEmitter.defaultMaxListeners` 修改该限制值会影响所有的 `EventEmitter` 实例，包括之前已经注册的 `EventEmitter` 实例。

此外需要留意的是，这并不是一个硬性限制指标。`EventEmitter` 的实例允许添加多个监听器，但是会在控制条输出一个提示 `possible EventEmitter memory leak`。可以使用 `emitter.getMaxListeners()` 和 `emitter.setMaxListeners()` 方法临时避免这种警告：

```js
emitter.setMaxListeners(emitter.getMaxListeners() + 1);
emitter.once('event', () => {
  // do stuff
  emitter.setMaxListeners(Math.max(emitter.getMaxListeners() - 1, 0));
});
```

#### emitter.addListener(event, listener)

emitter.on(event, listener) 的同名函数。

#### emitter.emit(event[, arg1][, arg2][, ...])

该方法以同步的方式调用指定 `event` 的每一个监听器，并将参数传递给它们。

如果 `event` 拥有监听器，则返回 true，否则返回 false。

#### emitter.getMaxListeners()

该方法返回 `EventEmitter` 可以设置的最大监听器数量，该值可以由 `emitter.setMaxListeners(n)` 方法修改，其默认值由 `EventEmitter.defaultMaxListeners` 决定。

#### emitter.listenerCount(event)

- `event`，事件类型

该方返回当前 `event` 类型所监听的监听器数量。

#### emitter.listeners(event)

该方法先拷贝 `event` 事件的监听器列表，然后将其以数组的方式返回：

```js
server.on('connection', (stream) => {
  console.log('someone connected!');
});
console.log(util.inspect(server.listeners('connection')));
// Prints: [ [Function] ]
```

#### emitter.on(event, listener)

该方法用于给指定的 `event` 添加监听器，并将该监听器置于监听器列表的末位。该方法不会检查是否已经添加过该监听器。重复添加相同的 `event` 和 `listener` 会导致该事件和监听器被重复触发：

```js
server.on('connection', (stream) => {
  console.log('someone connected!');
});
```

该方法返回一个对 `EventEmitter` 实例的引用，所以可以使用链式调用。

#### emitter.once(event, listener)

该方法为 `event` 事件添加一个一次性的监听器，该事件第一次触发之后就会被注销：

```js
server.once('connection', (stream) => {
  console.log('Ah, we have our first user!');
});
```

该方法返回一个对 `EventEmitter` 实例的引用，所以可以使用链式调用。

#### emitter.removeAllListener([event])

该方法用于注销所有的事件，或者注销指定的 `event`。

注意，不建议使用该方法注销其他组件或模块监听的事件，尤其是 `EventEmitter` 由其他模块或组件创建的时候。

该方法返回一个对 `EventEmitter` 实例的引用，所以可以使用链式调用。

#### emitter.removeListener(event, listener)

该方法用于移除 `event` 事件的监听器列表中的 `listener`：

```js
var callback = (stream) => {
  console.log('someone connected!');
};
server.on('connection', callback);
// ...
server.removeListener('connection', callback);
```

`removeListener` 对于某个事件的 `listener` 每次只会移除一个，所以如果某个 `listener` 被添加了多次，就需要执行多次 `removeListener` 才能完全注销该监听器。

因为监听器由内部数组进行管理，所以调用该方法会修改监听器在监听列表中的顺序。这并不会影响已调用的监听器，但会影响通过 `emitter.listeners()` 创建的监听器副本，所以可能需要重建该副本。

该方法返回一个对 `EventEmitter` 实例的引用，所以可以使用链式调用。

#### emitter.setMaxListeners(n)

默认情况下，当某个事件设置的监听器超过十个时，EventEmitter 实例就会抛出警告，这有助于开发者找出程序中的内存泄露。显而易见的是，并不是每一个事件的监听器都会少于十个，所以可以使用 `emitter.setMaxListeners()` 方法修改特定 `EventEmitter` 实例的监听器数量。如果限制值为 `Infinity` 或 `0`，则表示不限制监听器的数量。

该方法返回一个对 `EventEmitter` 实例的引用，所以可以使用链式调用。


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