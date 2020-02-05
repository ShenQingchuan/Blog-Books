# Node.js 事件循环与异步机制总结

> **觉如的常规唠叨时间：**
>
> 这是把以前几章的学习总结成一文 ...
>
> 学习 JavaScript 的过程中，我经历了一个探秘过程是：从`Promise` 到 `async/await`，最后我发现「**异步**」这些个问题都源自于 JS 的线程模型。
>
> 我们都知道 JavaScript 是一门单线程的语言，像我们这样的很多编程初学者刚开始写的代码，都是一件事接着一件事做，是「**同步**」的逻辑，但是在遇到较高并发场景时，如果顺次进行，任务就会排起长队 ...
>
> 这些任务中，有的是用 CPU 的计算密集型任务，执行起来相对较快，而有的是 I/O 密集型的任务，相对较慢，如果
>
> 阅读 Node.js 的官方文档，你会惊讶于几乎它的每一个 API 都带有「**回调函数**」，这个函数是在 API 的功能执行完成后才被调用的。
>
> 正是因为并发、多线程这样的环境中，我们作为程序员对各个任务执行的情况很难准确把控，做不到很好的任务调度安排，所以才说 "并发的学习是难点"。
>
> 而 "回调" 这种「**预置未来**」的方案大大方便了我们，程序的主体逻辑开始变得清晰而精炼。
>
> 详情请接着看下去吧！

如果你刚开始学习 Node.js，你一定会看到许多介绍中这样写：「 **异步、非阻塞的 I/O** 」，知乎上有个特别形象的例子，我在这里引用过来：

>老张爱喝茶，废话不说，煮开水。
>出场人物：老张，水壶两把（普通水壶，简称水壶；会响的水壶）。
>1. 老张把水壶放到火上，立等水开。（**同步阻塞**）
>老张觉得自己有点傻
>2. 老张把水壶放到火上，去客厅看电视，时不时去厨房看看水开没有。（**同步非阻塞**）
>老张还是觉得自己有点傻，于是学聪明了，买了会响笛的那种水壶。水开后能大声发出 "嘀──" 的噪音。
>3. 老张把响水壶放到火上，立等水开。（**异步阻塞**）
>老张觉得这样傻等意义不大
>4. 老张把响水壶放到火上，去客厅看电视，水壶响之前不再去看它了，响了再去拿壶。（**异步非阻塞**）
>老张觉得自己聪明多了。
>
>
>
>作者：「**愚抄**」
>链接：httpss://www.zhihu.com/question/19732473/answer/23434554
>来源：知乎

但并不是说只要你用了回调函数，程序就异步化了，"回调函数" 本身只是一种实现手段而已。

你可能会说：" 那这个就简单了，我知道回调什么时候用，Node.js 里一定是把 `callback(...args)` 放在那个 API 实现的最后。" 不过真的是这样么？如果它之前的代码很耗时，整个程序岂不是又被卡住了？

所以重点根本不是回调，而是 Node.js 采用了「**事件驱动**」。

Node.js 所有的异步 I/O 操作都会先生成一个事件监听者，在完成时都会发回一个相应的事件到事件队列，然后事件监听者才去执行回调函数。

```js
// 引入 events 模块
var events = require('events');
// 创建 eventEmitter 对象
var eventEmitter = new events.EventEmitter();
 
// 创建事件处理程序
var connectHandler = function connected() {
   console.log('连接成功。');
  
   // 触发 data_received 事件 
   eventEmitter.emit('data_received');
}
 
// 绑定 connection 事件处理程序
eventEmitter.on('connection', connectHandler);
 
// 使用匿名函数绑定 data_received 事件
eventEmitter.on('data_received', function(){
   console.log('数据接收成功。');
});
 
// 触发 connection 事件 
eventEmitter.emit('connection');
 
console.log("程序执行完毕。");
```

```bash
> node main.js
连接成功。
数据接收成功。
程序执行完毕。
```

在 Node.js 启动的时候，步骤分别是：

1. 初始化事件循环

2. 处理可能包含异步 API 调用的输入脚本（用户代码）或进入 [Node REPL](httpss://link.jianshu.com?t=httpss://nodejs.org/api/repl.html#repl_repl)

3. 执行异步 API 的回调、调度定时器或者调用 `process.nextTick()` 

   如此循环往复 ...

```bash
   ┌───────────────────────────┐
┌─>│          timers 定时器        │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │ pending callbacks  等待的回调  │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │          idle, prepare       │
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:     │
│  │           poll  轮询          │<────┤  connections,   │
│  └─────────────┬─────────────┘      │   data, etc.    │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │             check            │
│  └─────────────┬─────────────┘
│  ┌─────────────┴───────────────┐
└──┤ close callbacks 关闭类事件的回调  │
   └─────────────────────────────┘
```

*注意：每个方框都称为事件循环的“**阶段**”。*

每一阶段都有一个先进先出的待执行任务队列。而在每一阶段内部有自己的执行方法，也就是说，当进入其中一个阶段时，会执行任何该阶段自己特定的操作，然后才执行在该阶段的队列中的回调，直到队列里的回调都执行完了或执行的次数达到最大限制。当队列耗尽或执行的次数达到最大限制时，事件循环进入下一个阶段，如此循环。

**定时器**：这一阶段执行由 `setTimeout()` 和 `setInterval()` 设置的回调。
 **I/O 回调**：执行除关闭回调、定时器调度的回调和 `setImmediat()` 以外的几乎所有的回调。
 **ide，prepare**：仅内部使用。
 **轮询**：获取新的 I/O 事件；适当的时候这里会被阻断。
 **check**： `setImmediate()` 的回调。
 **关闭事件回调**：如 `socket.on('close', ...)` 的回调。

在事件循环的每次运行之间， Node.js 会检查是否在等待任何异步 I/O 或定时器，如果两个都没有就自动关闭。



## 各阶段详解

### 1 定时器 Timers

```js
const fs = require('fs');

// 函数 1:
function someAsyncOperation(callback) {
  // 假设 这个读取要花费 95ms
  fs.readFile('/path/to/file', callback);
}

const timeoutScheduled = Date.now();

setTimeout(() => { // 回调 1
  const delay = Date.now() - timeoutScheduled;

  console.log(`自从我被调度，已经过去了${delay}ms`);
}, 100);

someAsyncOperation(() => { // 回调 2
  const startCallback = Date.now();

  // 做点什么，来消耗 10ms
  while (Date.now() - startCallback < 10) {
		// 让循环跑着就好...
  }
});

// 打印出结果：自从我被调度，已经过去了105ms
```

你是不是这么预测的：

1. 函数 1：`0 ~ 95ms` 
2. 回调 1：`100ms +`
3. 回调 2：`105ms`

当事件循环进入轮询阶段，任务队列还是空的（ 因为`fs.readFile()` 还没执行完），所以这里开始一直循环，等待 `fs.readFile()` 执行完后有一个定时器达到阈值。

这里 `95ms` 最快到达，即文件先读完然后回调添加到轮询队列并开始执行，然而该回调任务需要花费 `10ms` 来执行。在这个过程中是在当前这轮循环的 `pending callbacks` 阶段

在执行完这个任务以后进入定时器阶段时发现有定时器阈值到了，可以开始执行了，然后开始执行这个定时器回调。在这个例子里，实际等待时间比指定的等待时间多了 `5ms`。

注意：为防止**轮询**阶段长时间独占循环，使其他阶段长时间得不到执行，[libuv](httpss://libuv.org/) （一个实现了 Node.js 事件循环和平台的所有异步行为的 C 库）在停止轮询更多事件之前，还具有一个固定的最大值（取决于系统）。



### 2 I/O 回调等待 Pending callbacks

此阶段执行上轮残留的回调，和对某些系统操作（如 TCP 错误类型）执行回调。例如，如果TCP套接字在尝试连接时接收到 `ECONNREFUSED`，则某些 `*nix` 系统希望等待报告错误。这将在挂起的回调阶段排队执行。



### 3 ide & prepare

这个阶段仅供 Node.js 内部使用



### 4 轮询 Poll

这个阶段有两个主要的功能：

1. 为阈值已经到了的定时器执行一些回调
2. 处理队列里的事件

 当事件循环进入这个阶段且没有定时器时，则：

- 如果轮询回调队列里不为空，事件循环将遍历回调队列，同步执行队列里的任务直到队列空了或达到依赖于系统的最大值。
- 如果队列为空，则：
  - 如果存在 `setImmediate()` ，事件循环将结束此阶段进入 `check` 阶段来执行 `setImmediate()` 的回调。
  - 如果不存在 `setImmediate()` ，事件循环将等待轮询阶段的回调入队，然后立刻执行这些回调。

一旦轮询队列为空，事件循环将检查是否有阈值到达了的定时器，如果有，事件循环将返回到定时器阶段来执行这些定时器的回调。



###  5 检定 Check

这个阶段允许我们在轮询阶段完成后立刻执行一些回调。如果轮询阶段变为空闲，并且有 `setImmediate()` 的回调排队，那么事件循环可能会继续进入 check 阶段，而不是等待轮询回调入队。

`setImmediate()` 实际上是一个特殊的定时器，它在事件循环的一个单独的阶段中运行。在轮询阶段完成之后，它使用一个 `libuv` API 调度回调执行。

一般来说，随着代码执行，事件循环最终会到达 `check` 阶段，在该阶段等待一个传入连接、请求等。



### 5 关闭类事件回调 Close callbacks

如果一个 socket 或 handle 突然关闭（如：`socket.destroy()` ），这个阶段将发送 `close` 事件。否则这个 `close` 事件将通过 `process.nextTick()` 发送。

> **`setImmediate()` VS `setTimeout()`**
>
> 它们 很像，区别在于执行的时间点：
>
> - `setImmediate()` 在当前轮询阶段完成后执行。
> - `setTimeout()` 在达到所定的时间（单位：ms）以后被执行。
>
> 它们被执行的顺序依赖于它们在上下文中的位置。如果这两个都是在主模块内部调用的，那么定时器将受到进程性能的限制（受运行在这个机器上的其它应用程序影响，比如我们上面那个 `100ms` 就不是很准...）



## 换个角度看

### 分任务队列

所有的任务可以分为同步任务和异步任务，同步任务，顾名思义，就是立即执行的任务，同步任务一般会直接进入到主线程中执行。

而异步任务，就是异步执行的任务，比如 AJAX 网络请求，`setTimeout` 定时函数等都属于异步任务，异步任务会通过任务队列 （Event Queue ）的机制来进行协调。

同步和异步任务分别进入不同的执行环境，同步的进入主线程，即 **主执行栈**，异步的进入 Event Queue 。主线程内的任务执行完毕为空，会去 Event Queue 读取对应的任务，推入主线程执行，上述这个过程的不断重复其实就是我们说的 **事件循环（Event Loop）**

![](https://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-02-05-WeChatabdbee12ee5a7e71794be6fe21bf16b9.png)

这里相信有人会想问，什么是 microtasks ？

规范中规定，task 分为两大类, 分别是 **Macro Task（宏任务）** 和 **Micro Task（微任务）**, 并且每个宏任务结束后, 都要清空所有的微任务。

这里的 Macro Task 也是我们常说的 task ，有些文章并没有做区分，后面提及的 task 皆视为宏任务（macro task）



### 分析示例代码

千言万语，不如就着例子讲来的清楚。下面我们可以按照规范，一步步执行解析，先贴一下例子代码：

```js
console.log('script start');

setTimeout(function() {
  console.log('setTimeout');
}, 0);

Promise.resolve().then(function() {
  console.log('promise1');
}).then(function() {
  console.log('promise2');
});

console.log('script end');
```

1. **整体这段代码作为第一个宏任务** 进入主线程，遇到 `console.log`，输出 `script start`
2. 遇到 `setTimeout`，其回调函数被分发到宏任务 Event Queue 中
3. 遇到 `Promise`，其 `then` 函数被分到到微任务 Event Queue 中,记为 then1，之后又遇到了 `then` 函数，将其分到微任务 Event Queue 中，记为 then2
4. 遇到 `console.log`，输出 `script end`

至此，Event Queue 中存在三个任务，如下表：

| 宏任务 | 微任务 |
| ---------- | ------ |
| setTimeout | then1  |
| -          | then2  |

1. 执行微任务，首先执行 then1，输出 `promise1`, 然后执行 then2，输出 `promise2`，这样就清空了所有微任务

2. 执行 setTimeout 任务，输出 `setTimeout`，至此，输出的顺序是：

   ```txt
   script start 
   script end 
   promise1
   promise2
   setTimeout
   ```

怎么样？这种像探案一样的感觉过瘾吧？那再来个有点难度的：

```js
console.log('script start');

setTimeout(function() {
  console.log('timeout1');
}, 10);

new Promise(resolve => {
    console.log('promise1');
    resolve();
    setTimeout(() => console.log('timeout2'), 10);
}).then(function() {
    console.log('then1')
})

console.log('script end');
```

嘻嘻，我就不在这里贴我做好的流程图啦，[你点击此链接直接打开图片查看答案吧！](https://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-02-05-WeChat879999a8bb40303296aeb5b0714bca2f.png)

> **一个 Tricky：** 按照规范，首先是整个 Script 代码的一次 task，那么除了本身的代码，一些 microtask 因为要在此轮 Tick 中清除，故而优先于 Script 代码中定义的 task 执行，所以如果有需要优先执行的逻辑，放入 microtask 队列会比 task 更早的被执行。



## 语法糖

### async

`async/await`实际上是`Generator`的语法糖。顾名思义，`async`关键字代表后面的函数中有异步操作，`await`表示等待一个异步方法执行完成。声明异步函数只需在普通函数前面加一个关键字`async`即可，如：

```js
async function funcA() {}
```

`async` 函数返回一个Promise对象（如果指定的返回值不是Promise对象， **也返回一个Promise**，只不过立即 `resolve `，处理方式同 `then `方法），因此 `async `函数通过 `return `返回的值，会成为 `then `方法中回调函数的参数：

```js
async function funcA() {
  return 'hello!';
}

funcA().then(value => {
  console.log(value);
})
// hello!
```

单独一个 `async `函数，其实与 Promise 执行的功能是一样的。



### await

而 `await  `顾名思义就是 **异步等待**，它等待的是一个 Promise，因此 `await `后面应该写一个 Promise 对象，如果不是Promise对象，那么会被转成一个立即 `resolve `的 Promise。 `async `函数被调用后就立即执行，但是一旦遇到 `await `就会先返回，等到异步操作执行完成，再接着执行函数体内后面的语句。总结一下就是：`async`函数调用不会造成代码的阻塞，但是`await`会引起`async`函数内部代码的阻塞。看看下面这个例子：

```js
async function foo() {
  console.log('async function is running!');
  const num1 = await 200;
  console.log(`num1 is ${num1}`);
  const num2 = await num1+ 100;
  console.log(`num2 is ${num2}`);
  const num3 = await num2 + 100;
  console.log(`num3 is ${num3}`);
}

foo();
console.log('run me before await!');
// async function is running!
// run me before await!
// num1 is 200
// num2 is 300
// num3 is 400
```

可以看出调用 `async foo`  函数后，它会立即执行，首先输出了`'async function is running!'`，接着遇到了 `await `异步等待，函数返回。先执行`foo() `后面的同步任务，同步任务执行完后，接着 `await` 等待的位置继续往下执行。

别忘了我们上面可是做过 `Promise.resolve.then()` 的分析的哦！当遇到 `await`，这里是 `200` 这样的非 Promise 会被转为直接 `resolve` 的，那么就是推进了微任务队列里，根据机制先退出 `foo` ，然后外层整个 Script 的 task 还没结束，之后... 就不用细细分说了 🤣

可以说，`async` 函数完全可以看作多个异步操作，包装成的一个 Promise 对象，而`await`命令就是内部`then`命令的语法糖。

值得注意的是， `await `后面的 Promise 对象不总是返回 `resolved `状态，只要一个 `await `后面的Promise状态变为 `rejected `，整个 `async `函数都会中断执行，为了保存错误的位置和错误信息，我们需要用 `try...catch` 语句来封装多个 `await `过程，例如：

```js
async function func() {
  try {
    const num1 = await 200;
    console.log(`num1 is ${num1}`);
    const num2 = await Promise.reject('num2 is wrong!');
    console.log(`num2 is ${num2}`);
    const num3 = await num2 + 100;
    console.log(`num3 is ${num3}`);
  } catch (error) {
    console.log(error);
  }
}

func();
// num1 is 200
// 出错了
// num2 is wrong!
```

这个语法糖最好的就是： **可以用同步的方式、思路与格式去写异步的代码！**更加符合编程习惯。



## 参考资料

- [Node.js 官方文档 Guide 英文原版: The Node.js Event Loop, Timers, and `process.nextTick()`](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)

- [【译文】Node.js的事件循环（Event loop）、定时器（Timers）和 process.nextTick()](https://www.jianshu.com/p/3c3bd24e7c12)

- [深入理解JavaScript事件循环机制 - 作者 ChessZhang](https://www.cnblogs.com/yugege/p/9598265.html)

- [理解 async/await - 作者 Leon](https://segmentfault.com/a/1190000015488033)

