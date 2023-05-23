## 基本概念

### Fiber

Fiber 是 React 16 之后默认的协调器，主要做了如下事情：

- 给不同类型的任务分配不同的优先级
- 中断任务，后续重启任务
- 在不再需要的情况下终止任务
- 复用之前已经完成的任务

### Fiber 节点

```javascript
{
  // 跟当前 Fiber 相关本地状态（比如浏览器环境就是 DOM 节点）
  stateNode: any

  // 单链表树结构
  return: Fiber | null // 指向它在 Fiber 节点树中的 parent，用来在处理完这个节点之后向上返回
  child: Fiber | null // 指向它的第一个子节点
  sibling: Fiber | null // 指向它的兄弟节点，兄弟节点的 return 指向同一个父节点

  // 更新相关
  pendingProps: any // 新的变动带来的新的 props
  memoizedProps: any // 上一次渲染完成之后的 props
  updatedQueue: UpdateQueue<any> | null // 该节点变动的更新队列
  memoizedState: any // 上一次渲染时候的 state

  // Schedule 相关
  expirationTime: ExpirationTime // 代表任务在未来的哪个时间点应该被执行
  childExpirationTime: ExpirationTime // 当前节点的子节点中，最近一次的过期时间

  // 在 Fiber 树的更新过程中，每个 Fiber 都会有一个跟其对应的 Fiber
  // 我们称他为 current <==> workInProgress
  alternate: Fiber | null

  // Effect 相关的
  effectTag: SideEffectTag // 代表该 Fiber 要执行的副作用类型
  nextEffect: Fiber | null // 单链表树结构，effect list 会通过这个属性构成一个单链表
  firstEffect: Fiber | null // 代表子树 effect list 中的第一个 Fiber
  lastEffect: Fiber | null // 代表子树 effect list 中的最后一个 Fiber
}
```

### effect

对于宿主组件（DOM 元素），工作包括添加、更新或删除元素。
对于类组件，React 可能需要更新 refs 并调用 componentDidMount 和 componentDidUpdate 生命周期方法。

### expirationTime

在 React 中，为防止某个 update 因为优先级的原因一直被打断而未能执行。
React 会设置一个 ExpirationTime，当时间到了 ExpirationTime 的时候，
如果某个 update 还未执行的话，React 将会强制执行该 update。

每一次 update 之前，Fiber 都会根据当下的时间（通过 requestCurrrentTime 获取）和更新的触发条件为每个更新队列的 Fiber 节点计算到期时间。
到期事件的计算有两种方式，一种是对交互引起的更新做计算 computeInteractiveExpiration，另一种是对普通更新做计算 computeAsyncExpiration。
两者调用的是同一个方法：computeExpirationBucket，只是传入的参数不同。

```javascript
export const HIGH_PRIORITY_EXPIRATION = __DEV__ ? 500 : 150
export const HIGH_PRIORITY_BATCH_SIZE = 100

export const LOW_PRIORITY_EXPIRATION = 5000
export const LOW_PRIORITY_BATCH_SIZE = 250
```

普通更新的 expirationTime 间隔是 25ms，高优先级是 10ms。
React 让两个相近的更新得到相同的 expirationTime，目的就是让这两个 update 能够合并，从而达到批量更新。

### 调度

React 应用更新时，Fiber 从当前处理节点，层层遍历置组件树跟组件，然后开始处理更新。
主要调度逻辑是通过 sheduleWork 来实现的。

1. 通过 fiber.return 属性，从当前 fiber 实例层层遍历至组件树根组件
2. 依次对每一个 Fiber 实例进行到期时间判断，若大于传入的任务到期时间，则将其更新为传入的任务到期时间
3. 调用 requestWork 方法开始处理任务，并传入获取的组件树根组件 FiberRoot 对象和任务到期时间

```javascript
function sheduleWork(fiber, expirationTime: ExpirationTime) {
  return sheduleWorkImpl(fiber, expirationTime, false)
}

function sheduleWorkImpl(
  fiber,
  expirationTime: ExpirationTime,
  isErrorRecovery: boolean,
) {
  let node = fiber

  while (node !== null) {
    if (
      node.expirationTime === noWork ||
      node.expirationTime > expirationTime
    ) {
      node.expirationTime = expirationTime
    }
    if (node.alternate !== null) {
      if (
        node.alternate.expirationTime === noWork ||
        node.alternate.expirationTime > expirationTime
      ) {
        node.alternate.expirationTime = expirationTime
      }
    }
    if (node.return === null) {
      if (node.tag === HostRoot) {
        const root = node.stateNode
        requestWork(root, expirationTime)
      } else {
        return
      }
    }
    node = node.return
  }
}

function requestWork(root: FiberRoot, expirationTime: ExpirationTime) {
  if (
    root.remainingExpirationTime === noWork ||
    root.remainingExpirationTime > expirationTime
  ) {
    root.remainingExpirationTime = expirationTime
  }
  if (expirationTime === Sync) {
    performWork(Sync, null)
  } else {
    scheduleCallbackWithExpirationTime(expirationTime)
  }
}
```

### 更新器（updater）

每一个 React 组件实例化时都会被注入一个 updater，负责协调组件与 React 核心进程的通信。
这使得 setState 在 ReactDOM、React Native、服务端渲染以及测试工具中有不同的实现。

在 ReactDOM 中，updater 使用了 Fiber 协调器，负责：

- 获取 Fiber 实例
- 从调度器获取当前 Fiber 实例的优先级
- 将更新任务划分优先级并根据优先级从高到低依次推入 Fiber 实例的更新队列
- 根据优先级调度更新任务

当我们点击 button 时，click 事件触发。
updater 在 Fiber 节点上添加一个待处理的更新队列，且调用工作，React 进入 render 阶段。

## Render

所有的 fiber 节点都会在 work loop 做处理，这里是它的实现：

```javascript
function workLoop(isYieldy) {
  if (!isYieldy) {
    // Flush work without yielding
    while (nextUnitOfWork !== null) {
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork)
    }
  } else {
    // Flush asynchronous work until the deadline runs out of time.
    while (nextUnitOfWork !== null && !shouldYield()) {
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork)
    }
  }
}
```

nextUnitOfWork 为 null 时，React 将退出工作循环，并准备提交更新。

```javascript
function performUnitOfWork(workInProgress) {
  let next = beginWork(workInProgress)
  if (next === null) {
    next = completeUnitOfWork(workInProgress)
  }
  return next
}

function beginWork(current, workInProgress) {
  switch (workInProgress.tag) {
    // ...
    case FunctionComponent: {
      //...
    }
    case ClassComponent: {
      //...
      return updateClassComponent(current, workInProgress /* ... */) // return workInProgress.child
    }
    case HostComponent: {
      //...
    }
    // ...
  }
}
```

### 子协调

React 进入 finishClassComponent 方法中，对组件返回的孩子执行 diff 算法。
finishClassComponent 返回了当前 Fiber 节点的第一个孩子的 Fiber 节点。
React 更新孩子的 props 是父级执行工作的一部分，它要使用 render 方法返回的 react 元素的数据。

```javascript
function updateClassComponent(current, workInProgress, Component /* ... */) {
  // ...
  const instance = workInProgress.stateNode
  let shouldUpdate
  if (instance === null) {
    // ...
    constructClassInstance(workInProgress, Component /* ... */)
    mountClassInstance(workInProgress, Component /* ... */)
    shouldUpdate = true
  } else if (current === null) {
    shouldUpdate = resumeMountClassInstance(workInProgress, Component /* ... */)
  } else {
    shouldUpdate = updateClassInstance(current, workInProgress /* ... */)
  }

  return finishClassComponent(
    current,
    workInProgress,
    Component,
    shouldUpdate,
    // ...
  )
}

function updateClassInstance(
  current,
  workInProgress,
  ctor,
  newProps,
  // ...
) {
  const instance = workInProgress.stateNode

  const oldProps = workInProgress.memoizedProps
  instance.props = oldProps
  if (oldProps !== newProps) {
    callComponentWillReceiveProps(workInProgress, instance, newProps /* ... */)
  }

  let updateQueue = workInProgress.updateQueue
  if (updateQueue !== null) {
    processUpdateQueue(workInProgress, updateQueue /* ... */)
    newState = workInProgress.memoizedState
  }

  applyDerivedStateFromProps(workInProgress /* ... */)
  newState = workInProgress.memoizedState

  const shouldUpdate = checkShouldComponentUpdate(workInProgress /* ... */)
  if (shouldUpdate) {
    instance.componentWillUpdate(newProps, newState /* ... */)
    workInProgress.effectTag |= Update
    workInProgress.effectTag |= Snapshot
  }

  instance.props = newProps
  instance.state = newState

  return shouldUpdate
}

function completeUnitOfWork(workInProgress) {
  while (true) {
    let returnFiber = workInProgress.return
    let siblingFiber = workInProgress.sibling

    nextUnitOfWork = completeWork(workInProgress)

    if (siblingFiber !== null) {
      return siblingFiber
    } else if (returnFiber !== null) {
      workInProgress = returnFiber
      continue
    } else {
      return null
    }
  }
}

function completeWork(workInProgress) {
  // ...
  switch (workInProgress.tag) {
    case FunctionComponent: {
      // ...
    }
    case ClassComponent: {
      // ...
    }
    case HostComponent: {
      // ...
      updateHostComponent(current, workInProgress /* ... */)
    }
    // ...
  }
  return null
}
```

完成 render 阶段后，就能把完成的副本（或者叫替代-alternate）树赋值给 FiberRoot 上的 finishedWork 属性。
它可以在 render 阶段后立即处理，或者挂起等浏览器空闲时处理。

### Fiber 树的遍历

- 开始：Fiber 从最上面的 React 元素开始遍历。
- 子节点：转到子元素，直到到达叶子元素。
- 兄弟节点：检查是否有兄弟节点元素。如果有，他就遍历兄弟节点元素，然后再到兄弟姐妹的叶子元素。
- 返回：如果没有兄弟节点，那么它就返回到父节点。

## Commit

主要运行在 commit 阶段的方法是 commitRoot，大致如下操作：

- 标记了 Snapshot 作用的节点执行 getSnapshotBeforeUpdate 生命周期方法
- 标记了 Deletion 作用的节点执行 componentWillUnmount 生命周期方法
- 执行所有 DOM 的插入、更新和删除
- 把 finishedWork 树置为当前树
- 标记了 Placement 作用的节点执行 componentDidMount 生命周期方法
- 标记了 Update 作用的节点执行 componentDidUpdate 生命周期方法

```javascript
function commitRoot(root, finishedWork) {
  commitBeforeMutationLifecycles()
  commitAllHostEffects()
  root.current = finishedWork
  commitAllLifeCycles()
}
```

### 前置突变（Pre-mutation）生命周期方法

```javascript
function commitBeforeMutationLifecycles() {
  while (nextEffect !== null) {
    const effectTag = nextEffect.effectTag
    if (effectTag & Snapshot) {
      const current = nextEffect.alternate
      commitBeforeMutationLifeCycles(current, nextEffect)
    }
    nextEffect = nextEffect.nextEffect
  }
}
```

### DOM 更新

```javascript
function commitAllHostEffects() {
  switch (primaryEffectTab) {
    case Placement: {
      commitPlacement(nextEffect)
      // ...
    }
    case PlacementAndUpdate: {
      commitPlacement(nextEffect)
      commitWork(current, nextEffect)
      // ...
    }
    case Update: {
      var current = nextEffect.alternate
      commitWork(current, nextEffect)
      // ...
    }
    case Deletion: {
      commitDeletion(nextEffect)
      // ...
    }
  }
}
```

### 后置突变（Post-mutation）生命周期方法

commitAllLifeCycles 是 React 调用所有剩余生命周期方法（componentDidUpdate 和 componentDidMount）的方法。

```javascript
function commitAllLifeCycles() {
  while (nextEffect !== null) {
    const effectTag = nextEffect.effectTag
    if (effectTag & (Update | Callback)) {
      const current = nextEffect.alternate
      commitLifeCycles(finishedRoot, current, nextEffect)
    }
    if (effectTag & Ref) {
      commitAttachRef(nextEffect)
    }
    nextEffect = nextEffect.nextEffect
  }
}

function commitLifeCycles(finishedRoot, current /* ... */) {
  // ...
  switch (finishedWork.tag) {
    case FunctionComponent: {
      // ...
    }
    case ClassComponent: {
      const instance = finishedWork.stateNode
      if (finishedWork.effectTag & Update) {
        if (current === null) {
          instance.componentDidMount()
        } else {
          instance.componentDidUpdate(
            prevProps,
            prevState,
            // ...
          )
        }
      }
      // ...
    }
    case HostComponent: {
      // ...
    }
    // ...
  }
}
```

## 参考

- https://zhuanlan.zhihu.com/p/424967867
- https://juejin.cn/post/6844903844825006087
- https://juejin.cn/post/6844903844825006093
