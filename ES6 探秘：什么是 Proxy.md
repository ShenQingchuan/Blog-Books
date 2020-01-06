# ES6 探秘：什么是 Proxy ？

> 欢迎阅览我的 JavaScript沉潜日志，从本文开始，我将开始学习一些 Vue3.0新特性实现的原理。

众所周知，Vue2.x 中基于 `Object.defineProperty()`拦截了对数据的操作，另外数组等对象也有针对性地重写了：

```js
push()
pop()
shift()
unshift()
splice()
sort()
reverse()
```

这几种方法，目的都是实现 **依赖收集、响应式的数据更新**

但是尽管这样，数组的其他属性仍是无法监测的，所以说也还是有局限性的，而 ES6 中的 Proxy 可以劫持整个对象，并返回一个新对象，并且有13种劫持操作。

Vue3.0 果断放弃了对 IE 前世代的兼容，随着 Node 的不断演进和浏览器生态的变化，全面转向 ES6 其实是大势所趋，但作为一个社区内十分火热的框架作出如此坚决的决定也很值得肯定。

以上都是闲话啦~ 我今天有在思否上看到一篇帖子，想一起看的话[链接戳这里哦！](https://segmentfault.com/a/1190000015783546#comment-area)这里是一个富有探索精神的同学向尤大提问的过程记录，下方评论区的一条评论我觉得说的蛮有道理的，**学过 Vue 2 原理的我们都知道，`Object.defineProperty()`是通过拦截 getter 和 setter**，但是想象一下：如果给一个巨大的数组每一个元素都挂监听，时间、空间的复杂度实在太高了吧...




## 先看看类型定义：

Proxy 是 ES6 中自带的类啦，可以直接用以下语句调用：

```js
let p = new Proxy(target, handler)
```

`target`和`handler`具体是什么，我们可以找 Typescript 帮忙。这是我一个很喜欢的学习方法，很想推荐给大家，Typescript 有对函数参数的详细定义，甚至还有很多注释、文档，是实践型学习的不二之选。

![Proxy 的 Typescript描述](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2019-12-22-111257.png)

1. `target` 即是 Proxy 劫持目标的类型，可以是任何类型的对象，包括原生数组，函数，甚至另一个代理。

2. `handler`是一个对象，其属性是当执行一个操作时，定义代理行为的函数。

   它对应的 TS 类型定义如下：

```ts
// lib.es5.d.ts
declare type PropertyKey = string | number | symbol;

// lib.es2015.proxy.d.ts
interface ProxyHandler<T extends object> {
    getPrototypeOf? (target: T): object | null;
    setPrototypeOf? (target: T, v: any): boolean;
    isExtensible? (target: T): boolean;
    preventExtensions? (target: T): boolean;
    getOwnPropertyDescriptor? (target: T, p: PropertyKey): PropertyDescriptor | undefined;
    has? (target: T, p: PropertyKey): boolean;
    get? (target: T, p: PropertyKey, receiver: any): any;
    set? (target: T, p: PropertyKey, value: any, receiver: any): boolean;
    deleteProperty? (target: T, p: PropertyKey): boolean;
    defineProperty? (target: T, p: PropertyKey, attributes: PropertyDescriptor): boolean;
    enumerate? (target: T): PropertyKey[];
    ownKeys? (target: T): PropertyKey[];
    apply? (target: T, thisArg: any, argArray?: any): any;
    construct? (target: T, argArray: any, newTarget?: any): object;
}
```

3. 另外我们还可以通过以下方式创建一个可以撤销的 Proxy 对象：

![revocable 的调用](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2019-12-22-112957.png)



## 详细剖析 Handler 中的常用 API

> 下文中的 `p` 皆为 `target` 被代理后生成的 Proxy 对象

### 1. Handler.get(target, property, receiver)

此方法用来拦截 **对目标的读取操作**：

1. **`target`** 目标对象
2. **`property`** 被获取的属性名
3. **`reciver`** Proxy 或者继承 Proxy 的对象
4. 此方法的**`this`** 上下文为 **`handler`** 对象

当对 `p` 作出以下操作时，会被转发到 `handler.get()`：

- 访问属性时：`p[foo]` 或者 `p.foo`
- 访问原型链属性：`Object.create(p)['foo']`、`Object.create(p).foo`
- 对 `p` 的 `Reflect.get`

```js
class Ele {
  constructor(x) {
    this.x = x
  }
}

const e = new Ele(4)
let p = new Proxy(e, {
  get: (target, property) => {
    return `using get() ${target[property]}`
  }
})
```

```js
// 以下结果皆为：
// using get() 4
console.log( p.x )
console.log( p['x'] )
console.log( Object.create(p).x )
console.log( Object.create(p)['x'] )
console.log( Reflect.get(p, 'x') )
```



### 2. Handler.set(target, property, value, receiver)

此方法用于拦截 **对目标的设置属性值操作**：

1. **`target`** 目标对象

2. **`property `** 被设置的属性名

3. **`value`** 被设置的值

4. **`receiver`** 「解释引用自MDN文档」

   是最初被调用的对象。通常是 `p` 本身，但 `handler` 的  `set` 方法也有可能在原型链上或以其他方式被间接地调用（因此不一定是 `p` 本身）。

   比如，假设有一段代码执行 `obj.name = "jen"`，`obj`不是一个 `Proxy` 且自身不含 `name` 属性，但它的原型链上有一个 `Proxy`，那么那个 `proxy`的 `set` 拦截函数会被调用，此时 `obj` 会作为 `receiver` 参数传进来。

`set` 方法应该返回一个布尔值，返回 `true` 代表此次设置属性成功了，如果返回 `false` 且设置属性操作发生在严格模式下，那么会抛出一个 `TypeError`。

当对 `p` 作出以下操作时，会被转发到 `handler.set()`：

- 指定属性值：`p.foo = bar` 或者 `p['foo'] = bar`
- 指定继承者的属性值：`Object.create(p).foo = bar` 或者 `Object.create(p)['foo'] = bar`
- `Reflect.set(p, 'foo', 'bar')`，以及对继承该 Proxy 对象 `p` 的对象做 `Reflect.set()`操作

我们来定义如下的一个 `set` 劫持函数：

```js
"use strict";
let p = new Proxy({}, {
  set: (target, key, value) => {
    console.log(`using set('${key}', ${
      typeof value === 'string' 
        ? "'" + value + "'" 
        : value})`
    )
    if (key === 'n') {
      if (!Number.isInteger(value)) {
        throw new TypeError("property 'n' must be a integer!")
      } else if (value > 200) {
        throw new RangeError('n is too large')
      }
    }
    target[key] = value
    return true // 表示成功
  }
});
```

下面给出 4 个范例：

```js
p.foo = 1;							// using set('foo', 1)
p['hello'] = 'world'		// using set('hello', 'world')
p.bar = undefined;			// using set('bar', undefined)
```

```js
try {
  p.n = 'abc'									// using set('n', 'abc')
} catch (error) {
  console.log(error.message)	// property 'n' must be a integer!
}
```

```js
try {
  p.n = 202										// using set('n', 202)
} catch (error) {
  console.log(error.message) 	// n is too large
}
```

```js
const child_p = Object.create(p)
child_p.n = 116			// using set('n', 116)
```

```js
Reflect.set(p, 'foo', 'baz')										// using set('foo', 'baz')
Reflect.set(child_p, 'child_foo', 'child_baz')	// using set('child_foo', 'child_baz')
```



### 3. Handler.apply(target, thisArg, args)

此方法用来拦截 **对目标的调用操作**

- **`target`** 目标函数，**必须是可被调用的。也就是说，它必须是一个函数对象，**不符合此要求则会抛出 `TypeError`。
- **`thisArg`** target 函数调用时绑定的 this 对象
- **`args`** target 函数调用时传入的实参列表，应是一个 **类数组**

该方法会拦截以下操作：

1. 直接调用：`p(...args)`
2. `p.apply(thisArg, args) `或 `p.call(thisArg, ...args)`
3. `Reflect.apply(target, thisArg, args)`

 下面我们给出范例：

```js
let sum = (...args) => {
  return args.reduce((a, b) => a + b)
}

let p = new Proxy(sum, {
  apply: (target, thisArg, args) => {
    console.log('it will be: ' + target.name + '(' + args.join(',') + ') * 10')
    return target(...args)
  }
})
```

```js
// 以下输出皆为：
// it will be: sum(1,2,4,6) * 10
// 13
console.log( p(1,2,4,6) )
console.log( p.apply(null, [1,2,4,6]) )
console.log( p.call(null, 1,2,4,6) )
console.log( Reflect.apply(p, null, [1,2,4,6]) )
```



接下来我们会介绍 3 个 **重写操作符** 的拦截函数：

### 4. Handler.construct(target, args, newTarget)

该方法主要用于拦截 **`new`** 操作符。

- **`target`** 目标对象

- **`args`** `contructor` 的参数列表

- **`newTarget`** 这里我认为是即新生成的构造函数

  >  MDN文档中写的是：`最初被调用的构造函数`，我有一些不理解，如果有知道的小伙伴可以在评论区给予我一些建议和指导，不胜感激。

`handler.construct()` **必须返回一个对象**。

该方法可以拦截以下操作：

1. `new p(...args)`
2. `Reflect.construct(target, args, newTarget)`

下面给出范例：

```js
function Person(name, sex) {
  this.name = name
  this.sex = sex
}

let p = new Proxy(Person, {
  construct: (target, args) => {
    console.log(`new ${target.name}(${args.join(', ')})`)
    return new target(...args)
  }
})

let tom = new p('tom', 'M')
// new Person(tom, M)
```



### 5. Handler.has(target, property)

该方法主要用于拦截 **`in`** 操作符。

- **`target`**  目标对象
- **`property`** 需要检查是否存在的属性

`handler.has()` **必须返回一个 `boolean` 类型的结果值**。

该方法可以拦截以下操作：

- 属性查询: `foo in p` 或 `'foo' in p`
- 继承属性查询: `foo in Object.create(p)`
- `with` 检查: `with(p) { (foo); }`
- `Reflect.has(target, property)`

下面是范例：

```js
let obj = {
  foo: 'bar',
  'baz': 123
}

let p = new Proxy(obj, {
  has: (target, property) => {
    console.log(`Check: '${property}' in target`)
    return property in target
  }
})

const propFoo = 'foo'
console.log(propFoo in p) 
// Check: 'foo' in target
// true

console.log('baz' in p)
// Check: 'baz' in target
// true

console.log('zeg' in p)
// Check 'zeg' in target
// false
```



### 6. Handler.deleteProperty(target, property)

该方法用于拦截 **`delete`** 操作符。

- **`target`** 目标对象
- **`property`** 要删除的属性

`handler.deleteProperty()` 必须返回一个 `Boolean` 值，表示是否删除成功。

该方法会拦截以下操作：

1. `delete p.foo` 或者 `delete p['foo']`
2. `Reflect.deleteProperty(target, property)`

**如果目标对象的属性是不可配置的，那么该属性不能被删除。** 违背该原则，`p` 会抛出一个 `TypeError`。

下面给出范例：

```js
let obj = {
  foo: 'bar',
  baz: 123,
  'zag': false,
  remain: 'End'
}

let p = new Proxy(obj, {
  deleteProperty: (target, property) => {
    console.log(`delete target.${property}`)
    Reflect.deleteProperty(target, property)
    return true
  }
})

delete p.foo												// delete target.foo
delete p['baz']											// delete target.baz
Reflect.deleteProperty(p, 'zag')		// delete target.zag

console.log(`p: ${JSON.stringify(p, null, 2)}`)
/*
p: {
  "remain": "End"
}
*/
```





### 7. Handler.defineProperty(target, property, desc)

- **`target`** 目标对象

- **`property`** 要设置的属性

- **`desc`** 待定义或修改的属性的描述对象。

  ```ts
  // desc 的类型定义如下：
  interface PropertyDescriptor {
      configurable?: boolean;
      enumerable?: boolean;
      value?: any;
      writable?: boolean;
      get?(): any;
      set?(v: any): void;
  }
  ```

`handler.defineProperty()` 可以拦截如下几种操作：

1. `Object.defineProperty(target, property, desc)`
2. `Reflect.defineProperty(target, property, desc)`
3. `p.foo = baz` 或 `p['foo'] = baz`

> 如果违背了以下的不变量，`p` 会抛出 `TypeError`
>
> - 如果目标对象不可扩展， 将不能添加属性。
> - 如果一个属性在目标对象中存在对应的属性，那么 `Object.defineProperty(target, property, descriptor)` 将不会抛出异常。
> - 在严格模式下， `false` 作为` handler.defineProperty` 方法的返回值的话将会抛出 `TypeError`异常.

下面给出范例：

```js
let obj = {}

let p = new Proxy(obj, {
  defineProperty: (target, property, desc) => {
    console.log(`define property('${property}') for target: ${desc.value}`)
    Reflect.defineProperty(target, property, desc)
    return true
  }
})

p.foo = 'bar'													// define property('foo') for target: bar
p['num'] = 999												// define property('num') for target: 999
Object.defineProperty(p, 'baz', {
  value: 123,
  writable: true,
  enumerable: true,
  configurable: true
})																		// define property('baz') for target: 123
Reflect.defineProperty(p, 'zag', {
  value: false,
  writable: true,
  enumerable: true,
  configurable: false
})																		// define property('zag') for target: false

console.log(`p: ${JSON.stringify(p, null, 2)}`)
/*
p: {
  "foo": "bar",
  "num": 999,
  "baz": 123,
  "zag": false
}
*/
```



## 参考文档：

- MDN 的中文文档一直是前端开发者学习 JS 的一方宝地，文档写得简明扼要，[戳这里查看 Proxy 的介绍呀！](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy)
- 本文相应的 Github [代码仓库](https://github.com/ShenQingchuan/What-is-ES6-Proxy)
