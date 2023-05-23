## let const

let 语句声明一个块级作用域的局部变量，并可以初始化为一个值（可选）。
const 非常类似用 let 语句定义的变量，但常量的值是无法（通过重新赋值）改变的。
const 必须有一个初始值。

## 与 var 的区别

let/const 声明的变量作用域是块作用域，而 var 声明的变量作用域是全局或者函数作用域。
let/const 声明的变量不会在作用域中被提升，它是在编译时才初始化。
let/const 不会在全局声明时创建 window 对象的属性。

## 暂时性死区

从一个代码块的开始直到代码执行到声明变量的行之前，let 或 const 声明的变量都处于“暂时性死区”（Temproral dead zone，TDZ）中。

## 参考

- https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/let
- https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/const
