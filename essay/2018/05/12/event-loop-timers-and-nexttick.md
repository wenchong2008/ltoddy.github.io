# The Node.js Event Loop, Timers, and process.nextTick()

## 什么是事件循环(event loop):

事件循环允许Node.js执行非阻塞I/O操作, 尽管,JavaScript是单线程.
 
由于现代大多数系统内核是多线程的,他们可以在后台执行多个操作.当某一个操作完成时,系统内核告诉Node.js,以便可以将适当的回调(callback)添加到轮询(poll)队列中去执行.

## 事件循环说明

当Node.js启动的时候,它会初始化事件循环,处理提供可能可能造成异步API调用,调度定时器(timers)任务或调用process.nextTick()脚本,然后开始处理事件循环.

下图显示了事件循环操作顺序(简易图解).

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

*提示:每个方格中的内容都是Node.js事件循环的一个阶段.*

每个阶段都有一个执行回调的FIFO队列,虽然每个阶段都有其特定的方式,但通常,事件循环进入一个特定阶段,它将执行这个特殊阶段的任何操作,然后执行该阶段的队列中的回调,
直到队列中的回调全部执行完或执行了最大回调数量.当队列耗尽或者回调达到限制,事件循环将会移动到下一个阶段,以此往复.

由于这些任何操作可能调度更多的操作,并且新的事件在轮询阶段由内核安排进队列,当轮询事件处理队列时轮询事件可以被入队.
因此,长时间运行回调可以使轮询阶段的运行时间远远超过计时器的阈值.

*注意:在windows和Unix/Linux实现中,有一些轻微的差别.*

## 阶段概述:

- **timers**: 这个阶段会执行setTimeout()和setInterval()调度的回调.
- **pending callbacks**: 执行延迟到下一个循环迭代的I/O回调.
- **idle,prepare**:只在内部使用.
- **poll**:检索新的I/O事件.执行I/O相关的回调函数,适当时,node将会在此阻塞.
- **check**: setImmediate()回调在此阶段被调用.
- **close callbacks**:一些关闭回调,比如: socket.on('close', ....)

在每次运行事件循环之间,Node.js检查它是否等待任何异步I/O或定时器,并在没有任何异常时关闭.

## 详细说明每个阶段:

### timers (定时器阶段)

定时器指定阈值,之后可以执行提供回调,而不是人们希望执行的确切时间.定时器回调将在指定的时间过后按照预定的时间运行;然而,操作系统调度或运行其他回调可能会延迟它们.

*注意:技术上,轮询阶段来控制定时器何时被执行*

举个例子,假设你安排了一个超时时间为100ms的阈值后执行,然后你的脚本异步读取一个耗时95ms的文件.

    ```javascript
    const fs = require('fs');

    function someAsyncOperation(callback) {
        // 假设这个任务花费95ms完成
        fs.readFile('/path/to/file', callback);
    }

    const timeoutScheduled = Date.now();

    setTimeout(() => {
        const delay = Date.now() - timeoutScheduled;

        console.log(`${delay}ms hava passed since I was scheduled.`);
    }, 100);

    // do someAsyncOperation which takes 95ms to complete.
    someAsyncOperation(() => {
        const startCallback = Date.now();

        // do something that will take 10ms...
        while (Date.now() - startCallback < 10) {
            // do nothing
        }
    });
    ```

当事件循环进入轮询阶段,它有一个空队列(fs.readFile()尚未完成).所以它会等待剩余的ms(毫秒)数,直到达到最快的定时器阀值.
当它等待了95毫秒,fs.readFile()完成读取文件并且它的回调函数被添加到轮询队列后执行.当回调完成时,队列中没有更多的回调,
所以事件循环将会查看已经达到最快定时器的阀值,然后回调定时器(timers)阶段执行定时器的回调.在这个例子中,
你将会看到被调度的定时器和它执行的回调之间总延迟是105ms.

*注意:为了防止轮询阶段阻塞循环,在停止轮询更多事件之前,libuv也有一个固定的最大值(具体由系统决定).*

### pending callbacks (等待处理回调阶段)

这个阶段执行系统操作的回调.例如如果当尝试连接时一个TCP套接字接收到*ECONNREFUSED*,一些*nix系统想要等待报告这个错误.
这将会在等待处理回调(pending callbacks)阶段入队.

### poll (轮询)

轮询(poll)阶段有两个主要的功能:

- 计算应该阻塞和轮询I/O的时间,然后
- 处理轮询队列中的事件.

当事件循环进入轮询阶段并且没有定时器调度时,会发生下面两件事之一:

- 如果轮询队列不为空,则事件循环将遍历其回调队列,同步执行它们,直到队列耗尽或达到系统相关限制.

- 如果轮询队列为空,则会发生另外两件事之一:
    - 如果脚本已通过setImmediate()进行调度,则事件循环将结束轮询阶段并继续执行检查阶段以执行这些预定脚本.
    - 如果脚本没有通过setImmediate()进行调度,则事件循环将等待将回调添加到队列中,然后立即执行它们.

定时器执行的顺序取决于它们被调用的上下文.如果两者都是在主模块内调用的,那么时序将受到进程性能的限制(可能会受到计算机上运行的其他应用程序的影响).

例如，如果我们运行不在I/O周期内的以下脚本(即主模块),两个定时器的执行顺序是非确定性的,因为它受进程执行的约束.

    ```javascript
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

但是，如果在I/O周期内执行这两个调用，则immediate回调总是首先执行:

    ```
    $ node timeout_vs_immediate.js
    immediate
    timeout

    $ node timeout_vs_immediate.js
    immediate
    timeout
    ```

使用setImmediate()优于setTimeout()的主要优点是setImmediate()将始终在任何定时器之前执行(如果在I/O周期内进行调度),而不考虑存在多少个定时器.

### process.nextTick()

#### 理解 process.nextTick()

您可能已经注意到process.nextTick()没有在图解中展示,即使它是异步API的一部分.这是因为,process.nextTick()在技术上不属于事件循环的一部分,反而,
*nextTickQueue*将在当前操作完成后处理，而不管当前阶段的事件循环.

回头看看我们的图解,任何时候你在给定阶段调用process.nextTick(),传递给process.nextTick()的所有回调将在事件循环继续之前解析.
这会造成一些坏情况,因为这会允许你递归调用process.nextTick()来阻塞你的I/O,从而防止事件循环到达轮询阶段.

#### 为什么这会被允许?

为什么像这样的东西会被加入到Node.js中?其中一部分是一种设计哲学,即即使不需要,API也应该始终是异步的.

以此代码片段为例.

    ```javascript
    function apiCall(arg, callback) {
        if (typeof arg !== 'string') {
            return process.nextTick(callback, new TypeError('argument should be string.'));
        }
    }
    ```

片段进行参数检查,如果不正确,它会将错误传递给回调,它将会将error传递给回调.
最近更新的API允许将参数传递给process.nextTick(),允许它将回调后传递的任何参数作为参数传播给回调函数,因此您不必嵌套函数。

通过使用process.nextTick(),我们保证apiCall()总是在用户代码的其余部分之后并且允许继续进行事件循环之前运行其回调.
为了实现这个,JS调用堆栈被允许展开,然后立即执行提供的回调,允许对process.nextTick()进行递归调用,但不会达到RangeError：超出v8的最大调用堆栈大小。

这个哲学可能会导致一些潜在的问题.以此代码片段为例.

    ```javascript
    let bar;

    // this has an asynchronous signature, but calls callback synchronously
    function someAsyncApiCall(callback) {
      callback();
    }

    // the callback is called before `someAsyncApiCall` completes.
    someAsyncApiCall(() => {
      console.log('bar', bar); // undefined
    });

    bar = 1;
    ```

用户定义someAsyncApiCall()具有异步签名,但它实际上是同步运行的.
当它被调用时，提供给someAsyncApiCall()的回调在事件循环的相同阶段被调用,因为someAsyncApiCall()实际上并不会异步执行任何操作.
因此，回调会尝试引用bar,即使它在范围中可能没有该变量,因为该脚本无法运行到完成状态.

通过将回调放在process.nextTick()中,脚本仍然有能力完成运行,允许在调用回调之前初始化所有变量，函数等.
它还具有不允许事件循环继续的优点.在事件循环被允许继续之前，用户被告知错误可能是有用的。

以下是使用process.nextTick()的示例：

    ```javascript
    let bar;

    function someAsyncApiCall(callback) {
      process.nextTick(callback);
    }

    someAsyncApiCall(() => {
      console.log('bar', bar); // 1
    });

    bar = 1;
    ```

这是另一个真实的例子：

    ```javascript
    const net = require('net');

    const server = net.createServer(() => {}).listen(8080);

    server.on('listening', () => {});
    ```

当只有一个端口通过时,该端口会立即绑定.所以,'listening'回调可以立即被调用.问题是.on('listening')回调不会在那个时候设置.

为了解决这个问题,'listen'事件在nextTick()中排队等待脚本运行完成.这允许用户设置他们想要的任何事件处理程序.

### process.nextTick() vs setImmediate()

*我们建议开发人员在所有情况下使用setImmediate().*

### 为什么使用process.nextTick()?

主要有两个原因:

- 允许用户处理错误,清理任何不需要的资源,或者可能在事件循环继续之前再次尝试请求.
- 有时需要在调用堆栈解除之后但事件循环继续之前允许回调运行.

一个简单的例子：

    ```javascript
    const net = require('net');

    const server = net.createServer();
    server.on('connection', (conn) => {});

    server.listen(8080);
    server.on('listening', () => {});
    ```

假设listen()在事件循环的开始处运行,但监听回调放置在setImmediate()中.
除非传递主机名,否则绑定到端口将立即发生.
要继续进行事件循环,它必须进入轮询阶段,这意味着有一个非零的机会可以收到连接,允许在监听事件之前触发连接事件.

另一个例子是运行一个构造函数,该函数构造函数是从EventEmitter继承的,并且它想要在构造函数中调用一个事件:

    ```javascript
    const EventEmitter = require('events');
    const util = require('util');

    function MyEmitter() {
      EventEmitter.call(this);
      this.emit('event');
    }

    util.inherits(MyEmitter, EventEmitter);

    const myEmitter = new MyEmitter();
    myEmitter.on('event', () => {
      console.log('an event occurred!');
    });
    ```

您不能立即从构造函数发出事件,因为脚本不会处理到用户为该事件分配回调的位置.
因此，在构造函数本身中,您可以使用process.nextTick()来设置回调,以在构造函数完成后发出事件，这将提供预期的结果:

    ```javascript
    const EventEmitter = require('events');
    const util = require('util');

    function MyEmitter() {
      EventEmitter.call(this);


      // use nextTick to emit the event once a handler is assigned
      process.nextTick(() => {
        this.emit('event');
      });
    }

    util.inherits(MyEmitter, EventEmitter);

    const myEmitter = new MyEmitter();
    myEmitter.on('event', () => {
      console.log('an event occurred!');
    });
    ```