---
title: Node.js 事件循环，定时器和 process.nextTick()
layout: docs.hbs
---

# Node.js 事件循环，定时器和 `process.nextTick()`

## 什么是事件循环

事件循环是 Node.js 处理非阻塞 I/O 操作的机制——尽管 JavaScript 是单线程处理的——当有可能的时候，它们会把操作转移到系统内核中去。

因为目前大多数内核都是多线程的，所以它们可以在后台处理多种操作。当其中的一个操作完成的时候，内核通知 Node.js 将适合的回调函数添加到 *轮询* 队列中等待时机执行。我们在本文后面会进行详细介绍。

## 事件循环机制解析

当 Node.js 启动后，它会初始化事件循环，处理已提供的输入脚本（或丢入 [REPL][]，本文不涉及到），它可能会调用一些异步的 API、调度定时器，或者调用 `process.nextTick()`，然后开始处理事件循环。

下面的图表展示了事件循环操作顺序的简化概览。

```
   ┌───────────────────────────┐
┌─>│           timers          │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │           poll            │<─────┤  connections, │
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │
   └───────────────────────────┘
```

*注意：每个框被称为事件循环机制的一个阶段。*

每个阶段都有一个 FIFO 队列来执行回调。虽然每个阶段都是特殊的，但通常情况下，当事件循环进入给定的阶段时，它将执行特定于该阶段的任何操作，然后执行该阶段队列中的回调，直到队列用尽或已经执行到最大的回调数。当该队列已用尽或达到回调限制，事件循环将移动到下一阶段，以此类推。

由于这些操作中的任何一个都可能调度 _更多的_ 操作并且在 **轮询（poll）** 阶段被处理的新事件会被内核排列， 并且在处理轮询中的事件时，轮询事件可以排队。因此，长时间运行的回调可以允许轮询阶段运行长于计时器的 **阈值（threshold）**。有关详细信息，请参阅 [**计时器**](#timers) 和 [**轮询**](#poll) 部分。

_**注意：** 在 Windows 和 Unix/Linux 实现之间存在细微的差异，但这对演示来说并不重要。最重要的部分在这里。实际上有七或八个步骤，但我们关心的是 Node.js 实际上使用以上的某些步骤。_

## 阶段概述

* **定时器**：本阶段执行已经被 `setTimeout()` 和 `setInterval()` 的调度回调函数。
* **待定回调**：执行延迟到下一个循环迭代的 I/O 回调。
* **idle, prepare**：仅系统内部使用。
* **轮询**：检索新的 I/O 事件;执行与 I/O 相关的回调（几乎所有情况下，除了关闭的回调函数，那些由计时器和 `setImmediate()` 调度的之外），其余情况 node 将在适当的时候在此阻塞。
* **检测**：`setImmediate()` 回调函数在这里执行。
* **关闭的回调函数**：一些关闭的回调函数，如：`socket.on('close', ...)`。

在每次运行的事件循环之间，Node.js 检查它是否在等待任何异步 I/O 或计时器，如果没有的话，则完全关闭。

## 阶段的详细概述

### <!--timers-->定时器

计时器可以 _在回调后面_ 指定 **阈值**，而不是用户希望回调执行的确切时间。因为在经过指定的一段时间间隔后，
计时器回调将被尽可能早地运行。但是，操作系统调度或其它正在运行的回调可能会延迟它们。

_**注意**：[**轮询** 阶段](#poll) 控制何时定时器执行。_

例如，您调度了一个在 100 毫秒后执行回调的定时器，并且您的脚本开始异步读取文件，这会耗费 95 毫秒:

```js
const fs = require('fs');

function someAsyncOperation(callback) {
  // Assume this takes 95ms to complete
  fs.readFile('/path/to/file', callback);
}

const timeoutScheduled = Date.now();

setTimeout(() => {
  const delay = Date.now() - timeoutScheduled;

  console.log(`${delay}ms have passed since I was scheduled`);
}, 100);

// do someAsyncOperation which takes 95 ms to complete
someAsyncOperation(() => {
  const startCallback = Date.now();

  // do something that will take 10ms...
  while (Date.now() - startCallback < 10) {
    // do nothing
  }
});
```

当事件循环进入 **轮询（poll）** 阶段时，它有一个空队列（此时 `fs.readFile()` 尚未完成），因此它将等待剩下的毫秒数，直到达到最快的一个计时器阈值为止。当它等待 95 毫秒过后，`fs.readFile()` 完成读取文件，它的那个需要 10 毫秒才能完成的回调将被添加到 **轮询** 队列中并执行。当回调完成时，队列中不再有回调，此时事件循环机制将发现计时器最快的阈值（100ms）的已经达到，然后将回到 **计时器** 阶段，以执行定时器的回调。在本示例中，您将看到调度计时器到它的回调被执行之间的总延迟将为 105 毫秒。

注意：为了防止 **轮询** 阶段事件循环陷入吃不饱的状态，[libuv][]（实现 Node.js
事件循环和平台的所有异步行为的 C 函数库）在停止轮询以获得更多事件之前，还有一个硬性的最大值（依赖于系统）。

### 挂起的回调函数

此阶段对某些系统操作（如 TCP 错误类型）执行回调。例如，如果 TCP 套接字在尝试连接时接收到 `ECONNREFUSED`，则某些 \*nix 的系统希望等待报告错误。这将被排队以在 **挂起的回调** 阶段执行。

### <!--poll-->轮询

**轮询** 阶段有两个重要的功能：

1. 计算应该阻塞和轮询 I/O 的时间。
2. 然后，处理 **轮询** 队列里的事件。

当事件循环进入 **轮询** 阶段且 _没有被调度的计时器时_ ，将发生以下两种情况之一：

* _如果 **轮询** 队列 **不是空的**_ ，事件循环将循环访问回调队列并同步执行它们，直到队列已用尽，或者达到了与系统相关的硬性限制。

* _如果 **轮询** 队列 **是空的**_ ，还有两件事发生：
  * 如果脚本被 `setImmediate()` 调度，则事件循环将结束 **轮询** 阶段，并继续 **检查** 阶段以执行那些被调度的脚本。

  * 如果脚本 **未被** `setImmediate()`调度，则事件循环将等待回调被添加到队列中，然后立即执行。

一旦 **轮询** 队列为空，事件循环将检查 _已达到时间阈值的计时器_。如果一个或多个计时器已准备就绪，则事件循环将绕回计时器阶段以执行这些计时器的回调。

### 检查阶段

此阶段允许人员在 **轮询** 阶段完成后立即执行回调。如果轮询阶段变为空闲状态，并且脚本使用 `setImmediate()` 后被排列在队列中，则事件循环可能继续到 **检查** 阶段而不是等待。

`setImmediate()` 实际上是一个在事件循环的单独阶段运行的特殊计时器。它使用一个 libuv API 来安排回调在 **轮询** 阶段完成后执行。

通常，在执行代码时，事件循环最终会进入轮询阶段，在该阶段它将等待传入连接、请求等。但是，如果回调已使用 `setImmediate()`调度过，并且轮询阶段变为空闲状态，则它将结束此阶段，并继续到检查阶段而不是继续等待轮询事件。

### 关闭的回调函数

如果套接字或处理函数突然关闭（例如 `socket.destroy()`），则`'close'` 事件将在这个阶段发出。否则它将通过 `process.nextTick()` 发出。

## `setImmediate()` 对比 `setTimeout()`

`setImmediate()` 和 `setTimeout()` 很类似，但是基于被调用的时机，他们也有不同表现。

* `setImmediate()` 设计为一旦在当前 **轮询** 阶段完成， 就执行脚本。
* `setTimeout()` 在最小阈值（ms 单位）过后运行脚本。

执行计时器的顺序将根据调用它们的上下文而异。如果二者都从主模块内调用，则时序将受进程性能的约束（这可能会受到计算机上其他正在运行应用程序的影响）。

例如，如果运行以下不在 I/O 周期（即主模块）内的脚本，则执行两个计时器的顺序是非确定性的，因为它受进程性能的约束：

```js
// timeout_vs_immediate.js
setTimeout(() => {
  console.log('timeout');
}, 0);

setImmediate(() => {
  console.log('immediate');
});
```

```
$ node timeout_vs_immediate.js
timeout
immediate

$ node timeout_vs_immediate.js
immediate
timeout
```

但是，如果你把这两个函数放入一个 I/O 循环内调用，setImmediate 总是被优先调用：

```js
// timeout_vs_immediate.js
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  setImmediate(() => {
    console.log('immediate');
  });
});
```

```
$ node timeout_vs_immediate.js
immediate
timeout

$ node timeout_vs_immediate.js
immediate
timeout
```

使用 `setImmediate()` 相对于`setTimeout()` 的主要优势是，如果`setImmediate()`是在 I/O 周期内被调度的，那它将会在其中任何的定时器之前执行，跟这里存在多少个定时器无关。

## `process.nextTick()`

### 理解 `process.nextTick()`

您可能已经注意到 `process.nextTick()` 在图示中没有显示，即使它是异步 API 的一部分。这是因为 `process.nextTick()` 从技术上讲不是事件循环的一部分。相反，它都将在当前操作完成后处理 `nextTickQueue`， 而不管事件循环的当前阶段如何。这里所谓的 *操作* 被定义为来自底层 C/C++ 处理器的转换，和需要处理的 JavaScript 代码的执行。

回顾我们的图示，任何时候在给定的阶段中调用 `process.nextTick()`，所有传递到 `process.nextTick()` 的回调将在事件循环继续之前解析。这可能会造成一些糟糕的情况，因为**它允许您通过递归 `process.nextTick()`调用来“饿死”您的 I/O**，阻止事件循环到达 **轮询** 阶段。

### 为什么会允许这样？

为什么这样的事情会包含在 Node.js 中？一部分因为它是一个设计理念，即尽管不是必需的情况下， API 应该始终是异步的。以此代码段为例：

```js
function apiCall(arg, callback) {
  if (typeof arg !== 'string')
    return process.nextTick(
      callback,
      new TypeError('argument should be string')
    );
}
```

代码段进行参数检查。如果不正确，则会将错误传递给回调函数。这个 API 最近进行了更新，允许传递参数给 `process.nextTick()`，这将允许它接受任何在回调函数位置之后的参数，并将参数传递给回调函数作为回调函数的参数，这样您就不必嵌套函数了。

我们正在做的是将错误传回给用户，但仅在执行用户的其余代码之后。通过使用`process.nextTick()`，我们保证 `apiCall()` 始终在用户代码的其余部分*之后*和在让事件循环继续进行*之前*执行其回调函数。为了实现这一点，JS 调用栈被允许展开，然后立即执行提供的回调，允许进行递归调用 `process.nextTick()`，而不触碰 `RangeError: 超过 V8 的最大调用堆栈大小` 限制。

这种设计原理可能会导致一些潜在的问题。
以此代码段为例：

```js
let bar;

// this has an asynchronous signature, but calls callback synchronously
function someAsyncApiCall(callback) {
  callback();
}

// the callback is called before `someAsyncApiCall` completes.
someAsyncApiCall(() => {
  // since someAsyncApiCall has completed, bar hasn't been assigned any value
  console.log('bar', bar); // undefined
});

bar = 1;
```

用户将 `someAsyncApiCall()` 定义为具有异步签名，但实际上它是同步运行的。当调用它时，提供给 `someAsyncApiCall()` 的回调是在事件循环的同一阶段内被调用，因为 `someAsyncApiCall()` 实际上并没有异步执行任何事情。结果，回调函数在尝试引用 `bar`，但作用域中可能还没有该变量，因为脚本尚未运行完成。

通过将回调置于 `process.nextTick()` 中，脚本仍具有运行完成的能力，允许在调用回调之前初始化所有的变量、函数等。它还具有不让事件循环继续的优点，适用于让事件循环继续之前，警告用户发生错误的情况。下面是上一个使用 `process.nextTick()` 的示例：

```js
let bar;

function someAsyncApiCall(callback) {
  process.nextTick(callback);
}

someAsyncApiCall(() => {
  console.log('bar', bar); // 1
});

bar = 1;
```

这又是另外一个真实的例子：

```js
const server = net.createServer(() => {}).listen(8080);

server.on('listening', () => {});
```

只有传递端口时，端口才会立即被绑定。因此，可以立即调用 `'listening'` 回调。问题是 `.on('listening')` 的回调在那个时间点尚未被设置。

为了绕过这个问题，`'listening'` 事件被排在 `nextTick()` 中，以允许脚本运行完成。这让用户设置所想设置的任何事件处理器。

## `process.nextTick()` 对比 `setImmediate()`

就用户而言，我们有两个类似的调用，但它们的名称令人费解。

* `process.nextTick()` 在同一个阶段立即执行。
* `setImmediate()` 在事件循环的接下来的迭代或 'tick' 上触发。

实质上，这两个名称应该交换，因为 `process.nextTick()` 比 `setImmediate()` 触发得更快，但这是过去遗留问题，因此不太可能改变。如果贸然进行名称交换，将破坏 npm 上的大部分软件包。每天都有更多新的模块在增加，这意味着我们要多等待每一天，则更多潜在破坏会发生。尽管这些名称使人感到困惑，但它们本身名字不会改变。

*我们建议开发人员在所有情况下都使用 `setImmediate()`，因为它更容易理解。*

## 为什么要使用 `process.nextTick()`?

有两个主要原因：

1. 允许用户处理错误，清理任何不需要的资源，或者在事件循环继续之前重试请求。

2. 有时有让回调在栈展开后，但在事件循环继续之前运行的必要。

以下是一个符合用户预期的简单示例：

```js
const server = net.createServer();
server.on('connection', (conn) => {});

server.listen(8080);
server.on('listening', () => {});
```

假设 `listen()` 在事件循环开始时运行，但 listening 的回调被放置在 `setImmediate()` 中。除非传递过主机名，才会立即绑定到端口。为使事件循环继续进行，它必须命中 **轮询** 阶段，这意味着有可能已经接收了一个连接，并在侦听事件之前触发了连接事件。

另外一个示例直接从 `EventEmitter` 继承，并且在构造函数内部触发一个事件：

```js
const EventEmitter = require('events');

class MyEmitter extends EventEmitter {
  constructor() {
    super();
    this.emit('event');
  }
}

const myEmitter = new MyEmitter();
myEmitter.on('event', () => {
  console.log('an event occurred!');
});
```

你不能立即从构造函数中触发事件，因为脚本尚未处理到用户为该事件分配回调函数的地方。因此，在构造函数本身中可以使用 `process.nextTick()` 来设置回调，以便在构造函数完成后发出该事件，这是预期的结果：

```js
const EventEmitter = require('events');

class MyEmitter extends EventEmitter {
  constructor() {
    super();

    // use nextTick to emit the event once a handler is assigned
    process.nextTick(() => {
      this.emit('event');
    });
  }
}

const myEmitter = new MyEmitter();
myEmitter.on('event', () => {
  console.log('an event occurred!');
});
```

[libuv]: https://libuv.org/
[REPL]: https://nodejs.org/api/repl.html#repl_repl
