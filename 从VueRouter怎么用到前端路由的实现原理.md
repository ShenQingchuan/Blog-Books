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



## VueRouter 源码探究

VueRouter 基于 Vue 组件化的概念，使用 VueRouter 对 Vue 进行组件化的路由管理。也就是使用组件替换的方式更新页面展示。

替换的地方，就是 `<router-view/>` 组件所在地方, `<router-view>` 组件是一个函数式的组件，根据路径和match 补丁算法，找到要渲染的组件进行渲染。

**这种渲染只有在路径变化才会发生。**



为了理清源码各个模块之间的关系，我们用一个思维导图来表示：

![](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-04-27-VueRouter%E6%BA%90%E7%A0%81%E6%9E%B6%E6%9E%84%E5%9B%BE.png)



### Router 的构造方法：

```js
constructor (options: RouterOptions = {}) {
  this.app = null // Vue实例
  this.apps = []  // 存放正在被使用的组件（vue实例），只有destroyed掉的组件，才会从这里移除）
  this.options = options // VueRouter实例化时，传来的参数，也就是router.js中配置项的内容
  this.beforeHooks = []  // 存放各组件的全局beforeEach钩子
  this.resolveHooks = [] // 存放各组件的全局beforeResolve钩子
  this.afterHooks = []   // 存放各组件的全局afterEach钩子
  this.matcher = createMatcher(options.routes || [], this) // 由createMatcher生成的matcher，里面有一些匹配相关方法
  let mode = options.mode || 'hash'  //模式

// 模式的回退或者兼容方式，若设置的mode是history，而js运行平台不支持
supportsPushState 方法，自动回退到hash模式
  this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false
  if (this.fallback) {
    mode = 'hash'
  }

// 若不在浏览器环境下，强制使用abstract模式
  if (!inBrowser) {
    mode = 'abstract'
  }
  this.mode = mode

 //不同模式，使用对应模式 Histroty 管理器去管理 history
  switch (mode) {
    case 'history':
      this.history = new HTML5History(this, options.base)
      break
    case 'hash':
      this.history = new HashHistory(this, options.base, this.fallback)
      break
    case 'abstract':
      this.history = new AbstractHistory(this, options.base)
      break
    default:
      if (process.env.NODE_ENV !== 'production') {
        assert(false, `invalid mode: ${mode}`)
      }
  }
}

```

### `init` 方法：

```js
init (app: any /* Vue component instance */) {

	// assert是个断言，测试 install.installed 是否为真，为真，则说明 VueRouter 已经安装了
  // 这个我们下面 install.js 里会说到
  process.env.NODE_ENV !== 'production' && assert(
    install.installed,
    `not installed. Make sure to call \`Vue.use(VueRouter)\` ` +
    `before creating root instance.`
  )
  //将 Vue实例推到 apps列表中，install里面最初是将 Vue根实例推进去的
  this.apps.push(app)

  // 设置一个销毁时的句柄
  // https://github.com/vuejs/vue-router/issues/2639

	// app被destroyed时候，会$emit ‘hook:destroyed’事件，监听这个事件，执行下面方法
	// 从apps 里将app移除
  app.$once('hook:destroyed', () => {
    // clean out app from this.apps array once destroyed
    const index = this.apps.indexOf(app)
    if (index > -1) this.apps.splice(index, 1)
    // 确保始终有一个主 app 否则我们不呈现这个 router，这里置为 null
    if (this.app === app) this.app = this.apps[0] || null
  })

  if (this.app) {
    return
  }
  // 新增一个history，并添加route监听器
  // 并根据不同路由模式进行跳转。hashHistory 需要监听 hashchange 和 popshate 两个事件，而 HTML5History 监听 popstate 事件。
  this.app = app

  const history = this.history

  if (history instanceof HTML5History) {
    // HTML5History 在 constructor 中包含了监听方法，因此这里不需要像
    // HashHistory 那样 setupListner。
    history.transitionTo(history.getCurrentLocation())
  } else if (history instanceof HashHistory) {
    const setupHashListener = () => {
      history.setupListeners()
    }
    history.transitionTo(
      history.getCurrentLocation(),
      setupHashListener,
      setupHashListener
    )
  }
 
  // 将apps中的组件的 _route 全部更新至最新的
  history.listen(route => {
    this.apps.forEach((app) => {
      app._route = route
    })
  })
}
```

最后这里的这个 `.listen` ：是设置了 `route` 改变之后的响应函数，会在 `confirmTransition` 中的`onComplete` 回调中调用, 并更新当前的 `route` 的值，前面我们提到，`route` 是响应式的，那么当其更新的时候就会去通知组件重新渲染。



### 全局路由钩子

在路由切换的时候这些自定义的钩子函数被调用。可以在 `main.js` 中使用 VueRouter 实例 `$router` 进行调用。比如文章一开头例子中用户身份验证的需求设置的钩子。

```js
// 将回调方法 fn 注册到 beforeHooks 里。
// registerHook 会返回 fn 执行后的callback方法，功能是将 fn 从 beforeHooks 删除。
beforeEach (fn: Function): Function {
  return registerHook(this.beforeHooks, fn)
}
// 将回调方法 fn 注册到 resolveHooks 里。
// registerHook 会返回 fn 执行后的callback方法，功能是将 fn 从 resolveHooks 删除。
beforeResolve (fn: Function): Function {
  return registerHook(this.resolveHooks, fn)
}
// 将回调方法 fn 注册到 afterHooks 里。
// registerHook 会返回，fn 执行后的 callback 方法，功能是将 fn 从 afterHooks 删除。
afterEach (fn: Function): Function {
  return registerHook(this.afterHooks, fn)
}
```

而上面用到的 `registerHook` 方法是：

```js
function registerHook (list: Array<any>, fn: Function): Function {
  list.push(fn)
  return () => {
    const i = list.indexOf(fn)
    if (i > -1) list.splice(i, 1)
  }
}
```

功能是将参数插入 `list` 中，返回一个从 `list` 中删除 `fn` 的方法。



### 主动改变路由的功能 API

通常我们在组件内 `$router.push` 调用的就是这里的 `push` 方法。 

```js
onReady (cb: Function, errorCb?: Function) {
  this.history.onReady(cb, errorCb)
}
// 造成报错
onError (errorCb: Function) {
  this.history.onError(errorCb)
}
// 新增路由跳转
push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  this.history.push(location, onComplete, onAbort)
}
// 路由替换
replace (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  this.history.replace(location, onComplete, onAbort)
}

// 前进n条路由
go (n: number) {
  this.history.go(n)
}
// 回退一步
back () {
  this.go(-1)
}
// 前进一步
forward () {
  this.go(1)
}
```

其中 `onReady` 方法是传入一个回调函数，它会在首次路由跳转完成时被调用。

此方法通常用于等待异步的导航钩子完成，比如在进行**服务端渲染**的时候，例如：

```js
return new Promise((resolve, reject) => { 
  const {app, router} = createApp() 
  router.push(context.url)
  router.onReady(() => { // 页面刷新，检测路由是否匹配 
    const matchedComponents = router.getMatchedComponents() 
    if (!matchedComponents.length) { 
      return reject(new Error('no component matched')) 
    } resolve(app)
  }); 
});
```



### `install.js`

VueRouter 通过 `Vue.use(router)` 来注册插件的时候，会找到插件的 `install` 方法进行执行。

`install` 主要有以下几个目的：

1. 子组件通过从父组件获取 `router` 实例，从而可以在组件内进行路由切换、在路由钩子内设置 `callback`、获取当前 `route` 等操作，所有 `router` 提供的接口均可以调用。 

2. 设置代理，组件内部使用 `$router, $route` 等同于获得当前的`router`实例和 `route` 

3. 为 `route` 设置数据劫持，**看上面的关系图里的 `Object.defineProperty()`**，

   也就是在数值变化时，可以 `notify` 到 `watcher` 

4. 注册路由实例，`<router-view/>` 中注册 `route`

5. 注册全局组件 `RouterView`、`RouterLink`

```js
import View from './components/view'
import Link from './components/link'

export let _Vue

export function install (Vue) {

	// 避免重复安装
  if (install.installed && _Vue === Vue) return
  install.installed = true

  _Vue = Vue

  const isDef = v => v !== undefined
  // 从父节点拿到 registerRouteInstance，注册路由实例
  const registerInstance = (vm, callVal) => {
    let i = vm.$options._parentVnode
    if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
      i(vm, callVal)
    }
  }
  
	// 混入 mixin, 这个每个组件在实例化的时候，都会在对应的钩子里运行这部分代码
  Vue.mixin({
    beforeCreate () {
      if (isDef(this.$options.router)) {
        this._routerRoot = this
        this._router = this.$options.router
				// 调用初始化 init 方法
        this._router.init(this)
				// 设置_route为 Reactive （设置其为响应式的）
        Vue.util.defineReactive(this, '_route', this._router.history.current)
      } else {
				// 子组件从父组件获取 routerRoot
        this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
      }
      registerInstance(this, this)
    },
    destroyed () {
      registerInstance(this)
    }
  })
  // 设置代理 this.$router === this._routerRoot._router，组件内部通过
	// this.$router调用_router
  Object.defineProperty(Vue.prototype, '$router', {
    get () { return this._routerRoot._router }
  })
	// 组件内部通过this.$route调用_route
  Object.defineProperty(Vue.prototype, '$route', {
    get () { return this._routerRoot._route }
  })

  // 注册RouterView、RouterLink组件
  Vue.component('RouterView', View)
  Vue.component('RouterLink', Link)

  const strats = Vue.config.optionMergeStrategies
  // 对路由钩子使用相同的钩子合并策略
  strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created
}
```



### 创建 Matcher

对于 VueRouter 示例构造器，其实就是两件主要的事情，创建 Matcher 和创建 History，我们先来说这头一档子：

```js
import VueRouter from './index'
import { resolvePath } from './util/path'
import { assert, warn } from './util/warn'
import { createRoute } from './util/route'
import { fillParams } from './util/params'
import { createRouteMap } from './create-route-map'
import { normalizeLocation } from './util/location'

// routes 为我们初始化 VueRouter 的路由配置；
// router 就是我们的 VueRouter 实例；
export function createMatcher (routes, router) {
  // pathList 是根据 routes 生成的 path 数组；
  // pathMap 是根据 path 的名称生成的 map；
  // 如果我们在路由配置上定义了 name，那么就会有这么一个 name 的 map；
  const { pathList, pathMap, nameMap } = createRouteMap(routes)
  // 根据新的 routes 生成路由；
  function addRoutes (routes) {
    createRouteMap(routes, pathList, pathMap, nameMap)
  }
  // 路由匹配函数；
  function match (raw, currentRoute, redirectedFrom) {
    // 简单讲就是拿出我们 path, params, query等等；
    const location = normalizeLocation(raw, currentRoute, false, router)
    const { name } = location

    if (name) {
      // 如果有 name 的话，就去 name Map 中去找到这条路由记录；
      const record = nameMap[name]
      if (process.env.NODE_ENV !== 'production') {
        warn(record, `Route with name '${name}' does not exist`)
      }
      // 如果没有这条路由记录就去创建一条路由对象；
      if (!record) return _createRoute(null, location)
      const paramNames = record.regex.keys
        .filter(key => !key.optional)
        .map(key => key.name)

      if (typeof location.params !== 'object') {
        location.params = {}
      }

      if (currentRoute && typeof currentRoute.params === 'object') {
        for (const key in currentRoute.params) {
          if (!(key in location.params) && paramNames.indexOf(key) > -1) {
            location.params[key] = currentRoute.params[key]
          }
        }
      }

      if (record) {
        location.path = fillParams(record.path, location.params, `named route "${name}"`)
        return _createRoute(record, location, redirectedFrom)
      }
    } else if (location.path) {
      location.params = {}
      for (let i = 0; i < pathList.length; i++) {
        const path = pathList[i]
        const record = pathMap[path]
        // 根据当前路径进行路由匹配
        // 如果匹配就创建一条路由对象；
        if (matchRoute(record.regex, location.path, location.params)) {
          return _createRoute(record, location, redirectedFrom)
        }
      }
    }
    // no match
    return _createRoute(null, location)
  }
  
  // ...省略中间一段其他的代码

  function _createRoute (record, location, redirectedFrom) {
    // 根据不同的条件去创建路由对象；
    if (record && record.redirect) {
      return redirect(record, redirectedFrom || location)
    }
    if (record && record.matchAs) {
      return alias(record, location, record.matchAs)
    }
    return createRoute(record, location, redirectedFrom, router)
  }

  return {
    match,
    addRoutes
  }
}
```

首先 `createMatcher` 会根据 `createRouteMap` 生成一份含有对应关系的映射表, `createRouteMap` 的具体实现在	`create-route-map.js` 中，其根据用户定义的 `routes` 配置的 `path`、`alias` 以及 `name`等项来生成对应的路由记录。

返回一个包含 `match` 和 `addRoutes` 两个方法的对象 `match`。

- `match` 是我们实现路由匹配的详细逻辑，会返回匹配的路由对象
- `addRoutes` 是添加路由的方法



### 关于 HashHistory 与 HTML5History

可以参见 [HashHistory 源码](https://github.com/vuejs/vue-router/blob/dev/src/history/hash.js) 和 [HTML5History 源码](https://github.com/vuejs/vue-router/blob/dev/src/history/hash.js) ：它们都是继承自 `base.js` 中的 History。

`HashHistory`  中我们主要关注：

```js
setupListeners () {
  const router = this.router
  const expectScroll = router.options.scrollBehavior
  const supportsScroll = supportsPushState && expectScroll

  if (supportsScroll) {
    setupScroll()
  }

  window.addEventListener(
    supportsPushState ? 'popstate' : 'hashchange',
    () => {
      const current = this.current

      // ensureSlash 路径是否正确，path 的第一为是否是'/'
      if (!ensureSlash()) {
        return
      }

      // transitionTo 调用的父类 History 下的跳转方法，跳转后会对路径进行 hash 化。
      this.transitionTo(getHash(), route => {
        //根据 router 配置的 scrollBehavior，进行滚动到 last pos。
        if (supportsScroll) {
          handleScroll(this.router, route, current, true)
        }
        if (!supportsPushState) {
          replaceHash(route.fullPath)
        }
      })
    }
  )
}

//判断 supportsPushState（浏览器对'pushState'是否支持），true时调用
// push-state.js 中的 pushState 方法。反之直接将当前 url 中 hash 替换。
function pushHash (path) {
  if (supportsPushState) {
    pushState(getUrl(path))
  } else {
    window.location.hash = path
  }
}

// 判断 supportsPushState（浏览器对 'pushState' 是否支持），true时调用
// push-state.js 中的 replaceState 方法。反之直接将当前 url 替换。
function replaceHash (path) {
  if (supportsPushState) {
    replaceState(getUrl(path))
  } else {
    window.location.replace(getUrl(path))
  }
}
```

`hashHitory` 需要同时监听 `hashchange` 和 `popstate` 事件，前者来自 `#hash` 中的 hash 变化触发，后者来自浏览器的 `back`，`go` 按钮的触发。

而 `HTML5History` 下只需监听浏览器的 `popstate` 事件，根据传来的参数获取 `location`，然后进行路由跳转即可，这部分在 `constructor` 中包含，`HTML5History` 实例化的时候就会执行。



## 总结

`hash` 是一个真实的 URL，它利用锚点 `#` 来实现单页面，只要锚点前的地址不变，这个页面就不会刷新,即便刷新了，也能立即获取到路由参数（`#` 后面的内容）

`history` 是一个虚假的 URL, 他就像你用代码去在地址栏敲了URL，**并且没按回车** ，你一旦按了回车，那基本性质就变了，因为浏览器会真的去请求这个 URL。`history.pushState()` 在保留现有历史记录的同时，将 URL 加入到历史记录中。` history.replaceState()` 会将历史记录中的当前页面历史替换为 URL。 

由于 `history.pushState()` 和 `history.replaceState()` 可以改变 URL 同时不会刷新页面，所以在 HTML5 中的 Histroy 接口具备了实现前端路由的能力。

**但需要注意的是，history 在修改 URL 后，虽然页面并不会刷新，但我们在手动刷新，或通过 url 直接进入应用的时候， 服务端是无法识别这个 URL 的。**

因为 Vue 本身生成出来是单页应用，只有一个 HTML 文件，服务端在处理其他路径的 URL 的时候，就会出现 404 的情况。 所以，如果要应用 `history` 模式：如果 URL 匹配不到任何静态资源，则应该至少给 Vue Router 一个通配符来重定向，最好是提供一个自定义的 404 页面：

```js
export default new Router({
  mode: 'history',
  routes: [
    // ...
    {path: "*", redirect: "/404"}
    {
      name: '404',
      path: '/404',
      component: () => import('@/views/notFound.vue')
    },
    // ...other routes
  ],
});
```

此处即设置如果 URL 输入错误或者是 URL 匹配不到任何静态资源，就自动跳到到 404 页面。