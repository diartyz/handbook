## 区块格式化上下文

区块格式化上下文（Block Formating Context，BFC）是 Web 页面的可视 CSS 渲染的一部分，是块级盒子的布局过程发生的区域，也是浮动元素与其他元素交互的区域。

下列方式会创建块格式化上下文：

- display 值为 flow-root 的元素
- 文档的根元素（`<html>`）
- 浮动元素（即 float 值不为 none 的元素）
- 绝对定位元素（即 position 值为 absolute 或 fixed 的元素）
- 行内块元素（即 display 值为 inline-block 的元素）
- 表格单元格（即 display 值为 table-cell 的元素，HTML 表格单元格默认为该值）
- 表格标题（即 display 值为 table-caption 的元素，HTML 表格标题默认为该值）
- 匿名表格单元格元素（即 display 值为 table、table-row、 table-row-group、table-header-group、table-footer-group（分别是 HTML table、row、tbody、thead、tfoot 的默认属性）或 inline-table 的元素）
- overflow 值不为 visible 或 clip 的块元素
- contain 值为 layout、content 或 paint 的元素
- 弹性元素（display 值为 flex 或 inline-flex 元素的直接子元素）
- 网格元素（display 值为 grid 或 inline-grid 元素的直接子元素）
- 多列容器（元素的 column-count 或 column-width 不为 auto，包括 column-count 为 1）
- column-span 为 all 的元素始终会创建一个新的 BFC，即使该元素没有包裹在一个多列容器中。

## BFC 功能

格式化上下文影响布局。它将：

- 包含内部浮动
- 排除外部浮动
- 阻止外边距重叠

## 弹性/网格容器

弹性/网格容器（display：flex/grid/inline-flex/inline-grid）建立新的弹性/网格格式化上下文， 除布局外，它与区块格式化上下文类似。

弹性/网格容器中没有可用的浮动子级，但排除外部浮动和阻止外边距重叠仍然有效。

## 参考

- https://developer.mozilla.org/zh-CN/docs/Web/Guide/CSS/Block_formatting_context
