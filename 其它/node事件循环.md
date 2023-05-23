在每次循环的间隙，Node.js 检查它是否在等待任何异步 I/O 或计时器，如果没有，它将关闭该事件循环。

nextTick 不管事件循环的当前阶段如何，都会在当前操作完成后立即执行回调。

## timers 阶段：执行 setTimeout() 和 setInterval() 的回调函数

## pending callbacks 阶段：执行延迟到下一个循环迭代的 I/O 回调

## idle, prepare 阶段：仅在内部使用

## poll 阶段：检索新的 I/O 事件，执行与 I/O 相关的回调；node 将在适当的时候阻塞在这里

- 如果 poll 队列不为空，Node.js 将同步地执行队列中的回调函数，直到队列为空或达到系统限制。
- 如果 poll 队列为空，以下两件事情将发生之一：
  - 如果脚本已安排 setImmediate() 回调，则 Node.js 将结束 poll 阶段并继续检查是否有计时器应该被执行。
  - 如果脚本未安排 setImmediate() 回调，则 Node.js 将等待回调被添加到队列中并立即执行。

一旦 poll 队列为空，事件循环将检查 timers 是否达到阈值，
如果有一个或多个计时器已准备就绪，则事件循环将绕回到 timers 阶段以执行这些计时器的回调。

## check 阶段：执行 setImmediate() 的回调函数

## close callbacks 阶段：执行关闭的回调函数

## 参考

- https://juejin.cn/post/7010308647792148511
- https://nodejs.org/zh-cn/docs/guides/event-loop-timers-and-nexttick
