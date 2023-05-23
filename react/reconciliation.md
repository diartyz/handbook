## 假设

- 两种不同类型的元素会产生不同的树
- 开发者可以通过 key 属性来标识哪些元素在不同的渲染下可以保持稳定

## Diffing 算法

当比较两颗树时，React 首先比较两个根元素。

### 不同类型的元素

React 会销毁旧的树，并且创建新的树。

### 相同类型的元素

React 会保留 DOM 节点，只更新有改变的属性。

当更新 style 属性时，React 只会更新 style 对象中发生改变的属性。

之后，React 会递归地比较子元素。

### 相同类型的组件

React 会递归比较 render() 返回值和之前的返回值。

### Children 的递归

默认情况下，当递归 DOM 节点时，React 会同时遍历两个子元素的列表；当产生差异时，React 会生成一个 mutation。

### Keys

当子元素拥有 key 时，React 使用 key 来匹配原有树上的子元素以及最新树上的子元素。

## 参考

- https://legacy.reactjs.org/docs/reconciliation.html
