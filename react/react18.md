React 18 已经放弃对 ie11 的支持，将于 2022/06/15 停止支持 ie。

## Render API

新的 root API 还支持 new concurrent renderer（并发模式的渲染），它允许你进入 concurrent mode（并发模式）。

```javascript
const root = document.getElementById("root");

ReactDOM.render(<App />, root);
// -->
ReactDOM.createRoot(root).render(<App />);

ReactDOM.hydrate(<App />, root);
// -->
ReactDOM.hydrateRoot(root, <App />);
```

ReactDOM.render 第三个参数是回调函数，新的 root API 不再支持这个参数。

## setState 自动批处理

React 18 之前，只有在 React 事件处理函数中的会自动批处理。
现在, 在 setTimeout、原生 js 事件等也会自动批处理。

## flushSync

flushSync 可以让你在 React 事件处理函数中强制同步更新。
flushSync 函数内部的多个 setState 仍然为批量更新。

## No Warning about setState on unmounted components

## React 组件的返回值

React 18 之前，如果显式的返回 undefined，控制台会在运行时抛出一个错误
在 React 18 中，不再检查因返回 undefined 而导致崩溃（但是 React 18 的 dts 文件还是会检查）。

## Strict Mode

当你使用严格模式时，React 会对每个组件进行两次渲染，以帮助你发现隐藏的副作用。
在 React 17 中，取消了其中一次渲染的控制台日志，以便让日志更容易阅读。

在 React 18 中，官方取消了这个限制。
如果安装了 React DevTools，第二次渲染的日志信息将显示为灰色，以柔和的方式显示在控制台。

## Suspense 不再需要 fallback 来捕获

## 新的 api

### useId

支持同一个组件在客户端和服务端生成相同的唯一的 ID。

### useSyncExternalStore

### useInsertionEffect

## Concurrent Mode

### startTransition

### useDeferredValue

## 参考

- https://juejin.cn/post/7094037148088664078#heading-23
