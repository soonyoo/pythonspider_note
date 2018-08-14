Ajax全名为Asynchronous JavaScript And XML ,即异步JavaScript与XML，并非一个编程语言，它仅仅是使用了一个集合：

- 浏览器内置的`XMLHttpRequest`对象（用于从web服务器请求数据）
- `JavaScript` 和`HTML DOM` (用于显示或者使用数据)

> Ajax可能是有点误导性的名字，Ajax应用层可能使用XML来传输数据，但是一般更多使用的是`Plain`文本或者`Json`来传输数据。

**一般通过Ajax来实现异步加载数据，以此来更新网站页面**。

如下截图，我们通过不断往下拉页面，页面不断继续刷新。

![](http://ww1.sinaimg.cn/large/67c0b572gy1fu7rawfk3pj20lh0hugoy.jpg)