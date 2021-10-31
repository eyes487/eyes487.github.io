

## 块级格式化上下文（BFC）

### 什么是BFC

一个块格式化上下文（block formatting context） 是Web页面的可视化CSS渲染出的一部分。它是块级盒布局出现的区域，也是浮动层元素进行交互的区域。 （MDN）

浮动，绝对定位元素，非块盒的块容器（例如，inline-blocks，table-cells和table-captions）和'overflow'不为'visible'的块盒会为它们的内容建立一个新的块格式化上下文  （CSS规范）

BFC全称”Block Formatting Context”, 中文为“块级格式化上下文”。啪啦啪啦特性什么的，一言难尽，大家可以自行去查找，我这里不详述，免得乱了主次，总之，记住这么一句话：BFC元素特性表现原则就是，内部子元素再怎么翻江倒海，翻云覆雨都不会影响外部的元素。所以，避免margin穿透啊，清除浮动什么的也好理解了。 （张鑫旭）[https://www.zhangxinxu.com/wordpress/2015/02/css-deep-understand-flow-bfc-column-two-auto-layout/]

上面这些都没有很好的解释BFC是什么，总的来说，BFC没有定义，它只有特性/功能


### 如何创建BFC

* 根元素（<html>）
* 浮动元素（元素的 float 不是 none）
* 绝对定位元素（元素的 position 为 absolute 或 fixed）
* 行内块元素（元素的 display 为 inline-block）
* overflow 计算值(Computed)不为 visible 的块元素
* display 值为 flow-root 的元素
* 弹性元素（display 为 flex 或 inline-flex 元素的直接子元素）

* 表格单元格（元素的 display 为 table-cell，HTML表格单元格默认为该值）
* 表格标题（元素的 display 为 table-caption，HTML表格标题默认为该值）
* 匿名表格单元格元素（元素的 display 为 table、table-row、 table-row-group、table-header-group、table-footer-group（分别是HTML table、row、tbody、thead、tfoot 的默认属性）或 inline-table）
* contain 值为 layout、content 或 paint 的元素
* 网格元素（display 为 grid 或 inline-grid 元素的直接子元素）
* 多列容器（元素的 column-count 或 column-width (en-US) 不为 auto，包括 column-count 为 1）
* column-span 为 all 的元素始终会创建一个新的BFC，即使该元素没有包裹在一个多列容器中（标准变更，Chrome bug）。

### BFC的用途

#### BFC的一些特性
1.内部的box会在垂直方向，一个接一个地放置
2.box垂直方向的距离由margin决定，同一个bfc的两个相邻margin会发生重叠
3.bfc的区域不会与float box重叠
4.bfc就是页面上的一个隔离的独立容器，容器里的子元素不会影响外面的元素，反之也是如此
5.计算bfc的高度时，浮动元素也会计算在内



* 用 BFC 包住浮动元素（模拟清除浮动）。比如设置 overflow:hidden (但是这样会产生副作用),更好的方法：clear:botn
http://js.jirengu.com/rozaxufetu/1/edit?html,css,output


* 用 float + div 做左右自适应布局，更好的办法，可以使用display:flex实现
http://js.jirengu.com/felikenuve/1/edit?html,css,output


* 上下margin 重叠
创建两个BFC


### css硬件加速

### css渲染 https://zhuanlan.zhihu.com/p/31311515