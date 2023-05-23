一个 Promise 是一个代理，它代表一个在创建 promise 时不一定已知的值。
它允许你将处理程序与异步操作的最终成功值或失败原因关联起来。
这使得异步方法可以像同步方法一样返回值：异步方法不会立即返回最终值，而是返回一个 promise，以便在将来的某个时间点提供该值。

一个 Promise 必然处于以下几种状态之一：

- 待定（pending）：初始状态，既没有被兑现，也没有被拒绝。
- 已兑现（fulfilled）：意味着操作成功完成。
- 已拒绝（rejected）：意味着操作失败。

如果一个 Promise 已经被兑现或拒绝，即不再处于待定状态，那么则称之为已敲定（settled）。

revolved 意味着该 Promise 已经敲定，或为了匹配另一个 Promise 的最终状态而被“锁定（lock-in）”，进一步解决或拒绝它都没有影响。

## Promise 并发

### Promise.all

在所有传入的 Promise 都被兑现时兑现；在任意一个 Promise 被拒绝时拒绝。

### Promise.allSettled

在所有的 Promise 都被敲定时兑现。

### Promise.any

在任意一个 Promise 被兑现时兑现；在所有的 Promise 都被拒绝时拒绝。

### Promise.race

在任意一个 Promise 被敲定时敲定。换句话说，在任意一个 Promise 被兑现时兑现；在任意一个 Promise 被拒绝时拒绝。

## async 和 await

### async 函数的返回值

- return 结果值：非 thenable、非 promise（不等待）
- return 结果值：thenable（等待一个 then 的时间）
- return 结果值：promise（等待两个 then 的时间）

### await 右值类型区别

- 接非 thenable 类型，会立即添加一个微任务，且不需等待
- 接 thenable 类型，会立即添加一个微任务，且需等待 一个 then 的时间之后执行
- 接 promise 类型，会立即添加一个微任务，且不需等待（TC39 后）

## 手写 Promise

```javascript
const PENDING = 'PENDING'
const FULFILLED = 'FULFILLED'
const REJECTED = 'REJECTED'

class Promise {
  constructor(executor) {
    this.status = PENDING
    this.value = undefined
    this.reason = undefined
    this.onResolvedCallbacks = []
    this.onRejectedCallbacks = []

    const resolve = (value) => {
      if (this.status === PENDING) {
        this.status = FULFILLED
        this.value = value
        this.onResolvedCallbacks.forEach((fn) => fn())
      }
    }
    const reject = (reason) => {
      if (this.status === PENDING) {
        this.status = REJECTED
        this.reason = reason
        this.onRejectedCallbacks.forEach((fn) => fn())
      }
    }

    try {
      executor(resolve, reject)
    } catch (error) {
      reject(error)
    }
  }

  then(onFulfilled, onRejected) {
    onFulfilled =
      typeof onFulfilled === 'function' ? onFulfilled : (value) => value
    onRejected =
      typeof onRejected === 'function'
        ? onRejected
        : (error) => {
            throw error
          }

    const nextPromise = new Promise((resolve, reject) => {
      const resolvePromise = (cb) => {
        try {
          const x = cb(this.value)
          if (x instanceof Promise) {
            x.then(resolve, reject)
          } else {
            resolve(x)
          }
        } catch (error) {
          reject(error)
        }
      }

      if (this.status === FULFILLED) {
        resolvePromise(onFulfilled)
      } else if (this.status === REJECTED) {
        resolvePromise(onRejected)
      } else if (this.status === PENDING) {
        this.onResolvedCallbacks.push(() => {
          resolvePromise(onFulfilled)
        })
        this.onRejectedCallbacks.push(() => {
          resolvePromise(onRejected)
        })
      }
    })

    return nextPromise
  }
}
```

## 参考

- https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise
