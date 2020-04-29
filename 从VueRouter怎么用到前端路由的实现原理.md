# 从 VueRouter 怎么用到前端路由的实现原理

> Vue 的强大周边生态保障了我们构建应用的基础。VueRouter 作为路由控制的重要一环，在非服务端渲染的 SPA 上扮演着重要的角色。
>
> 由于 VueRouter 本身有着详尽的 [官方文档](https://router.vuejs.org/zh/) ，所以我在这里讲述太多文档之中有的内容也没有意义，主要是总结一下值得记录的 API 使用方式，尤其是在一些特定业务场景下。



## 主要由前端承载的权限路由控制的解决思路

有时候我们的应用中一些页面要求用户必须登录后才可访问，那么显然在进入该路由前，我们应该根据一些前端存储的（持久化的）信息确定用户是否登录。

现在你脑子中可能会浮现很多种想法 ... 

其中有一种会是和 Cookie 相关的么？ HTTP-Only 的限制策略导致这在 VueRouter 的 JS 环境中不一定读取得到，那么如果

- 你采用的是 Cookie-Token 的形式，确保要能够读取该 Cookie

- 你采用的是比较传统的 Cookie-Session ID 的登录策略，表示状态的 Session ID 应该被存储，例如存入 LocalStorage ...

- 你采用了 JSON-Web-Token 的登录策略，那么你的 Token 也应当被持久化。

进而，我们最好是给这些需要登录到页面维护一份「 **路由白名单** 」，然后当路由切换时，利用 VueRouter 提供的 API 拦截并检查当前是否登录，**有的时候甚至还要根据用户类型做权限等级的区分**，这个需要用到的拦截 API 就是 `router.beforeEach` 一类的钩子函数：

```js
router.beforeEach((to, from, next) => {
  // ...
});
```

- `to` 是马上要进入的目标路由

- `from` 是正在离开的源路由

- `next` **一定要调用**，用来完成当前这一层中间拦截的逻辑。

  而 `next` 函数的参数多样使得我们可以作出不同的反应：

  - `next()` 就是正常无误，表示确认跳转
  - `next(false)` 即会中断当前的路由变化
  - `next({ path: '/xxx' })` 即跳转到另外的路由上，这个参数对象的属性格式与 `router.push` 方法的参数一致。



# _SPA_ 前端路由的实现原理：

## 基于 Hash 实现：

**优点：** 兼容性好，能够兼容到较低版本的 IE 浏览器。

**缺点：** URL 中有一个丑丑的 `#`，虽然我个人并不觉得丑 😓

我们经常在 url 中看到 `#`，这个 `#` 有两种情况，一个是我们所谓的 **锚点**，比如典型的 "回到顶部按钮"、Github README.md 上各个标题之间的跳转等，而路由里的 `#` 不叫锚点，我们称之为 `Hash`，大型框架的路由系统一开始大多都是哈希实现的。

同样我们需要一个随哈希变化而触发的事件：`hashchange`

如果你想看代码示例，[可以查看这里的一个简易版演示。](https://codepen.io/shenqingchuan/pen/qBOmyza?__cf_chl_jschl_tk__=d647d952cbe968603a20a1e1a631c769c1cadc3e-1587958413-0-AVY4zCLe73bh7IsISYCW0krjuiSyyDWOc121NfGzhVHbZyvk-YSidxKTISerKvH_1i_n0q9tBtvfiyDjuxIqDkVGb4NBlXt25gjoKS4qSkNna-p29h36OyyvohRA33eEfW0jf6Djlr2jReBaCCuiIhPtaM6UG77lGFcllWZKIqNB1d0gOM-cQtil3iqT1gLGqKe4vblezrfMlv_qfOX-D-DYWRDerI61GyYAzpjmplbZ7jaQg806QfhvYUPE3nnBkj-UlbDtELjHPlvaJihhTf5FKsUZNRpj4MUKT_FyUcUlpHe8-S0PZld7tZiteRilHflloFPjxG_XuxrSjAIT418RW_oOUg-zivQkcytoJ7xd)

## 基于 History API 实现

[参见 MDN 文档](https://developer.mozilla.org/zh-CN/docs/Web/API/History) ， `History`  API 允许操作浏览器的曾经在标签页或者框架里访问的会话历史记录。

这其中起主要影响的是：`history.pushState()` 与 `history.replaceState()` 

这两个 API 都接收三个参数，分别是：

- **状态对象（state object）** ： 一个 JavaScript 对象，与用 `pushState()` 方法创建的新历史记录条目关联。

  无论何时用户导航到新创建的状态，`popstate` 事件都会被触发，并且事件对象的 `state` 属性都包含历史记录条目的状态对象的拷贝。

- **标题（title）** ： FireFox 浏览器目前会忽略该参数，虽然以后可能会用上。

  考虑到未来可能会对该方法进行修改，传一个空字符串会比较安全。或者，你也可以传入一个简短的标题，标明将要进入的状态。

- **地址（URL）** ： 新的历史记录条目的地址。

  浏览器不会在调用 `pushState()` 方法后加载该地址，但之后可能会试图加载，例如用户重启浏览器。

  新的 URL 不一定是绝对路径；如果是相对路径，它将以当前URL为基准；

  传入的 URL与当前 URL 应该是 **同源的** ，否则，`pushState()` 会抛出异常。

  该参数是可选的；不指定的话则为文档当前 URL。



上面这两个 API 的其他相同之处是：**两个 API 都会操作浏览器的历史记录，而不会引起页面的刷新。**

不同之处在于 `pushState()` 会增加一条新的历史记录，而 `replaceState()` 则会替换当前的历史记录。