# Vue-next Reactivity 模块探秘

> Vue-next 中的 Reactivity 是一个可独立使用的模块，预计会发布为 @vue/reacitivity，含有诸如 `reactive`、`ref`、`effect`、`computed` 等新的 Composition API。
>
> 它主要负责的是 Vue 的响应式数据系统，用于替代 Vue 2.x 中  `Object.defineProperty` 方案，改用 ES6 的 `Proxy`，并且采用了新的依赖收集方式。
>
> 具体有什么不一样呢？让我们一起来看看吧！

这篇文章的源代码仓库[请戳这里呀！](https://github.com/ShenQingchuan/Anx)基本是按照 `vue-next/packages/reactivity` 中的结构。

## Reactive 篇

既然我们要 「单测驱动」来阅读源码，那么[看看我从源码中选取的这部分相应的几则单测](https://github.com/ShenQingchuan/Anx/blob/master/packages/reactivity/test/reactivity.spec.ts) 

```ts
// 让简单的对象 变为 响应式
test('reacitive for plain value: ', () => {
  const original = {
    a: 123,
    b: 'this is just a string',
    c: false
  }
  const observed = reactive(original)

  expect(observed).not.toBe(original)
  expect(isReactive(original)).toBe(false)
  expect(isReactive(observed)).toBe(true)
  // get
  expect(observed.a).toBe(123)
  // has
  expect('b' in observed).toBe(true)
  // ownKeys
  expect(Object.keys(observed)).toEqual(['a', 'b', 'c'])
})
```

要可以使得一个 JavaScript 的简单对象被转为「响应式」对象，可以看到源码当中还开放了一个 `isReactive` 供给我们来断言某对象是否为响应式。

那么我们来看看 `reactive()` 是怎么实现的：

```ts
export function reactive<T extends object>(target: T): UnwrapNestedRefs<T>
export function reactive(target: object) {
  // 如果只读表中已经有了他，就直接返回该 只读型Proxy
  if (readonlyToRaw.has(target)) {
    return target
  }
  // 目标对象已经被显示标记为了只读
  if (readonlyValues.has(target)) {
    return readonly(target)
  }
  return createReactivityObject(
    target, 
    rawToReactive,
    reactiveToRaw,
    mutableHandlers
  )
}
```

这里使用了两个 WeakMap，分别是 `readonlyToRaw` 和 `readonlyValues`，主要涉及**「只读型」** 值的管理，和转换成为响应式的主要逻辑不相关，故可暂时跳过。

主要逻辑还是在这个 `createReactivityObject` 中，来看看他函数签名中的参数类型：

```ts
export function createReactivityObject(
  target: any,
  toProxy: WeakMap<any, any>,
  toRaw: WeakMap<any, any>,
  baseHandlers: ProxyHandler<any>,
  collectionHandlers: ProxyHandler<any>
)
```

- `target` 直译过来就是「目标」，意为挂载响应式监听的目标，也就是「源对象」
- `toProxy` 和 `toRaw` 是两个 WeakMap，分别对应的是 `<源对象, 代理对象>` 和 `<代理对象, 源对象>`。
- `baseHandlers` 即 我们平常创建 `Proxy` 时配置的 `handler`，如果你对它还不熟悉，我给你[指路 MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy/)
- `collectionHandlers` 其专门对应那些对 **「集合类型」值**创建 Proxy 代理时的 Handler

这就保证了将创建「可变型」与「只读型」响应式对象的两种逻辑抽象成为一个。

```ts
{
  if (!isObject(target)) {
    console.warn(`not an 'Object' type value, can not be observered: ${String(target)}`)
    return target
  }
  // 如果 target 已经转过 Proxy 了（怎么判断？就是靠我们的 toProxy K-V 中的 V）
  let observed = toProxy.get(target)
  if (observed !== void 0) {
    return observed
  }
  // target 本身就是个 Proxy（怎么判断？就是靠我们的 toRaw K-V 中的 K）
  if (toRaw.has(target)) {
    return target
  }
  // 检测 target 是否是可以观察（监听）的
  if (!canObserve(target)) {
    return target
  }

  const handlers = collectionTypes.has(target.constructor)
  	? collectionHandlers
  	: baseHandlers
  observed = new Proxy(target, baseHandlers)
  toProxy.set(target, observed)
  toRaw.set(observed, target)
  return observed
}
```

首先 `target` 如果不是对象这样的「引用类型」，而是简单的 `number | string | boolean ... ` 这些「值类型」，那么我们就不挂载响应式了。

> 在 [ `shared`](https://github.com/ShenQingchuan/Anx/tree/master/packages/shared) 这个文件夹中存放了很多项目中的「公用方法」，都是一些 JavaScript 或 Typescript 中的精妙技巧，很多甚至值得我们背会、熟用，运用到自己的项目中。

```ts
// 判断某值是否为 Object 类型
export const isObject = (val: unknown): val is Record<any, any> =>
  val !== null && typeof val === 'object'
```

另外，如果一个对象是「不可观察」 的，我们也只返回源对象 `target`：

```ts
// 判断是否是可观察的对象
const canObserve = (value: any): boolean => {
  return (
    !value._isAux &&
    !value._isANode &&
    isObservableType(toRawType(value)) &&
    !nonReactiveValues.has(value)
  )
}
```

如果所有的记录中都找不到，预检查也都通过了，那么我们就对此 `target` 新建一个 `Proxy`，并在两个 WeakMap 中记录下对应关系再返回此代理对象。

所以所谓的「**响应式对象**」其实就是 Handlers 中编写了所需操作的 Proxy 对象。

还记得 Vue 2.x 中是使用 `Object.defineProperty` 来覆写属性的 `get / set` 方法的么？曾经想要做劫持需要这么写：

```js
let apple = {}
let val = 3000
Object.defineProperty(apple, 'price', {
  enumerable: true,
  configurable: true,
  get(){
    console.log('apple 属性被读取')
    return val
  },
  set(newVal){
    console.log('apple 属性被修改')
    val = newVal
  }
})
```

但不足就是对诸如 `Array` 这类的对象劫持效果并不理想，所以 Vue 2.x 选择重写数组类原型上的一些常用方法，实现依赖收集、追踪和触发。

然而 Proxy 的 Handlers 比单单这两个 `get / set` 强大得多，可以劫持 13 种不同类型的操作，在第二则单测中，我们测试了「对数组使用 `reactive` 是否有效果」：

```ts
// 让数组变为响应式
test('reacitive for Array: ', () => {
  const original = [{ foo: 1 }]
  const observed = reactive(original)
  expect(observed).not.toBe(original)
  expect(isReactive(observed)).toBe(true)
  expect(isReactive(original)).toBe(false)
  expect(isReactive(observed[0])).toBe(true)
  // get
  expect(observed[0].foo).toBe(1)
  // has
  expect(0 in observed).toBe(true)
  // ownKeys
  expect(Object.keys(observed)).toEqual(['0'])
})
```

那么要探究根据，我们必须看看在 `reactive()` 和 `readonly()` 中传入的 `mutableHandlers` 和 `readonlyHandlers` 中都是怎么拦截的！



### BaseHandler · 基本句柄

#### Get · 劫持对象的属性获取活动：

```ts
function createGetter(isReadonly: boolean = false) {
    // receiver 即是被创建出来的代理对象
    return function get(target: object, key: string | symbol, receiver: object) {
        // 用 Reflect 获取原始数据的相应值
        let res = Reflect.get(target, key, receiver)

        // 如果是js的内置方法，不做依赖收集
        if (isSymbol(key) && builtInSymbols.has(key)) {
            return res
        }

        // 如果是 Ref 型的值，返回 .value
        if (isRef(res)) {
            return res.value
        }

        // 收集依赖
        track(target, TrackOpTypes.GET, key)

        return isObject(res)
            ? isReadonly
                ? // 懒挂载依赖，避免出现循环依赖，同时提高性能
                readonly(res)
                : reactive(res)
            : res
    }
}
```

如果你还不太清楚 `Reflect` 我继续[指路 MDN 相关部分](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Reflect) ，总之一般 `Reflect` 就是和 `Proxy` 想对应的一个只有方法的类，一般`Reflect`就是做一些对源对象的操作。

各个部分注释都很详细了，但是最后一段的那个「懒挂载」我还是要重点拿出来讲一下。我们都知道 Vue 3 的核心性能真的提高很多，懒加载是两个功臣之一。

如果你没有读过 2.x 的响应式原理解析，我建议你还是[在这里面读一下]([https://nlrx-wjc.github.io/Learn-Vue-Source-Code/reactive/object.html#_2-%E4%BD%BFobject%E6%95%B0%E6%8D%AE%E5%8F%98%E5%BE%97%E2%80%9C%E5%8F%AF%E8%A7%82%E6%B5%8B%E2%80%9D](https://nlrx-wjc.github.io/Learn-Vue-Source-Code/reactive/object.html#_2-使object数据变得"可观测"))（这些不是我写的，如果可以还是请支持人家博主，给个 Star 吧。）

如果你读过了这些 Vue 2.x 的源码教程，使用 Option API 在 `data()` 中定义某个对象时：

```js
export default {
  data() {
  	return {
      example: {
        a: 188,
        b: 'Someone Like You',
        c: false
      }
    }
	}
}
```

要知道 `vue-loader`在加载单文件组件（就`.vue`文件中这个导出的对象），是会执行一次 Vue 的构造函数，对 `data()` 函数返回的这个对象执行 `walk`。

在经过 `defineReactive`之后， `example`的三个属性都会挂上依赖，其实这样的依赖收集是比较冗余的，Vue 的项目当中不乏一些不需要在视图中展现，甚至不需要响应式的数据，所以对此做优化是很有必要的！

```ts
track(target, TrackOpTypes.GET, key)
```

`track` 是之后要讲的 `effect.ts` 中导出的 API，在这里你现在只需要理解它执行后的结果就是将 `target[key]` 推进了一个依赖栈，有点像原来 2.x 的 `Dep`。

2.x 源码中对于 `nested object` 即复杂嵌套对象的情况，解决方式是递归 `defineReactive` ，这里也是一样的思路，如果 `target[key]` 仍是 `object` 类型，那么继续使用 `reactive` 转换为响应式。

#### Set · 劫持属性的赋值：

```ts
function createSetter(isReadonly: boolean = false) {
    return function set(
        target: any,
        key: string | symbol,
        value: any,
        receiver: any
    ): boolean {
        if (isReadonly && LOCKED) {
            if (__DEV__) {
                console.warn(
                    `Set operation on key "${String(key)}" failed: target is readonly.`,
                    target
                )
            }
            return true
        }
        // 获取旧值
        const oldValue = (target as any)[key]
        // 如果value是响应式数据，则返回其映射的源数据
        value = toRaw(value)
        // 如果旧值是 Ref 而新的值不是 Ref，那么更新 old 的 .value 并直接结束 set 过程
        if (isRef(oldValue) && !isRef(value)) {
            oldValue.value = value
            return true
        }

        // 代理对象中，是否真的有这个key，没有说明操作是新增
        const hadKey = hasOwn(target, key)
        // 将本次设置行为，反射到原始对象上
        const result = Reflect.set(target, key, value, receiver)
        // 如果是原始数据原型链上的数据操作，不做任何触发监听函数的行为。
        if (target === toRaw(receiver)) {
            if (__DEV__) {
                // 开发环境下，会传给trigger一个扩展数据，包含了新旧值，便于开发环境下做一些调试。
                const extraInfo = { oldValue, newValue: value }
                // 如果不存在key时，说明是新增属性，操作类型为 ADD，否则就是更新，即 SET
                // 存在key，则说明为更新操作，当新值与旧值不相等时，才是真正的更新，进而触发trigger
                if (!hadKey) {
                    trigger(target, TriggerOpTypes.ADD, key, extraInfo)
                } else if (value !== oldValue) {
                    trigger(target, TriggerOpTypes.SET, key, extraInfo)
                }
            } else {
                // 同上述逻辑，只是少了供给 debug 用的 extraInfo
                if (!hadKey) {
                    trigger(target, TriggerOpTypes.ADD, key)
                } else if (value !== oldValue) {
                    trigger(target, TriggerOpTypes.SET, key)
                }
            }
        }
        return result
    }
}
```

这段我 Anx 中的实现暂时剔除了`vue-next`中的 `shallow`，源码当中留下的注释翻译过来是：`在浅层模式中，对象按原样设置，而不考虑是否响应。`

1. **为什么要 `value = toRaw(value)`, 所赋的值为什么要转为源对象？**

   所以说要谨记我们之前说的「**懒挂载**」原则呀！我们并不知道 **所属对象是否需要这个新值变得响应式**，所以我们默认最好是就保存源对象，以节省内存开销，只有至少需要过一次才挂上 Proxy 代理监听：

   ```ts
   const a_original = {
     x: 33
   }
   let a_reactive = reactive(a_original, mutableHandlers)
   
   const b_original = {
     k: a_reactive
   }
   let b_reactive = reactive(b_original, mutableHandlers)
   /*
   b_reactive 的结果是 {
    k: a_original
   }
   */
   ```

   等到某个地方需要 `b_reactive.k`，触发了 `get` 之后发现 `a_original` 非响应式，一查 WeakMap 发现保存过与`a_reactive`响应式对象的关系，「懒挂载」此时不懒惰了，也触发了。

2. **为什么最后一段要包在一个 `if (target === toRaw(reciever))` 之中呢？**

   刚看完 MDN 时，我觉得`receiver`有点儿像是`this`一样的存在，指代着被 Proxy 执行后的代理对象。那代理对象用`toRaw`转化，也就是转为原始对象，自然跟`target`是全等的。

   但是这里还有一个很偏门的知识点：

   > **Receiver**：最初被调用的对象。通常是 proxy 本身，但 handler 的 set 方法也有可能在原型链上或以其他方式被间接地调用，因此不一定是 proxy 本身。

   ```ts
   const child = new Proxy(
     {},
     {
       get(target, key, receiver) {
         return Reflect.get(target, key, receiver)
       },
       set(target, key, value, receiver) {
         Reflect.set(target, key, value, receiver)
         console.log('child', receiver)
         return true
       }
     }
   )
   
   const parent = new Proxy(
     { x: 888 },
     {
       get(target, key, receiver) {
         return Reflect.get(target, key, receiver)
       },
       set(target, key, value, receiver) {
         Reflect.set(target, key, value, receiver)
         console.log('parent', receiver)
         return true
       }
     }
   )
   
   Object.setPrototypeOf(child, parent)
   
   child.x = 4
   
   // 打印结果
   // parent {x: 4}
   // child {a: 4}
   ```

   本来我们想要的结果是只触发 `child` 的 `set`句柄函数，没想到 `parent` 竟然也跟着被触发了，在这种情况下，`parent`其实并没有变更，按道理来说，它确实不应该触发它的监听函数。

   所以为了避免我们的响应式系统中出现这样的冗余操作，方案就是 **"通过用 WeakMap 保存的「代理与原生的对应关系」对触发的起因对象进行断言。"**

#### Other Traps · 其他劫持句柄

```ts
// 劫持属性删除
function deleteProperty(target: any, key: string | symbol): boolean {
    const hadKey = hasOwn(target, key)
    const oldValue = target[key]
    const result = Reflect.deleteProperty(target, key)
    if (result && hadKey) {
        if (__DEV__) {
            trigger(target, TriggerOpTypes.DELETE, key, { oldValue })
        } else {
            trigger(target, TriggerOpTypes.DELETE, key)
        }
    }
    return result
}

// 劫持 in 操作符
function has(target: any, key: string | symbol): boolean {
    const result = Reflect.has(target, key)
    track(target, TrackOpTypes.HAS, key)
    return result
}

// 劫持 Object.keys
function ownKeys(target: any): (string | number | symbol)[] {
    track(target, TrackOpTypes.ITERATE, ITERATE_KEY)
    return Reflect.ownKeys(target)
}
```

这些句柄在「可变型」与「只读型」中都是比较通用的，逻辑也比较简单，就不再赘述了，大家自行阅读吧。



###  CollectionHandler · 集合类型专用句柄

> **为什么它们需要特别对待？**
>
> 因为 Proxy 也是有缺陷的啦！对于 `Map、Set、WeakMap` 这些内置集合类型，当我们使用它们的方法（例如 `entries`、`delete`等）时，这些方法的 `this` 指向的本应该是源对象，但通过代理对象去操作时，`this` 指向的是 **代理对象**。所以就会报如下错：
>
> ```bash
> Uncaught TypeError: 
>   Method Set.prototype.add called on incompatible receiver [object Object]
> ```

#### 如何劫持

那么源码当中给出了怎样的解决方案呢？简单来说，就是 “**移花接木**”！

也就是说，当访问的是一个方法时，我们需要给他重新绑定 `this`。`Instrumentation` 的意思就是 “插桩”，下面这个函数的意思是：**创造一个特殊的 getter，如果是 get 集合类型的方法，那么我给你换一些做了劫持的方法**，意思基本等同于原来 Vue 2.x 时代给数组重写方法。

```ts
function createInstrumentationGetter(
  instrumentations: Record<string, Function>
) {
  return (
    target: CollectionTypes,
    key: string | symbol,
    receiver: CollectionTypes
  ) =>
    Reflect.get(
      hasOwn(instrumentations, key) && key in target
        ? instrumentations
        : target,
      key,
      receiver
    )
}
```

那么这些所谓的 「桩」：`instrumentation` 到底是什么呢？它是一个对象，具有和集合类型一样的方法名，但其实这些方法其实是已经被我们改造过的，我们看一个例子：

```ts
const mutableInstrumentations: Record<string, Function> = {
  get(this: MapTypes, key: unknown) {
    return get(this, key, toReactive)
  },
  get size(this: IterableCollections) {
    return size(this)
  },
  has,
  add,
  set,
  delete: deleteEntry,
  clear,
  forEach: createForEach(false)
}
```

上面这些方法传入时，都是在源码中都重写了的，比如下面的 `get`：

```ts
function get(
  target: MapTypes,
  key: unknown,
  wrap: typeof toReactive | typeof toReadonly
) {
  // 获取原始数据
  target = toRaw(target)
  // 由于 Map可以用对象做 key，所以 key也有可能是个响应式数据，先转为原始数据
  key = toRaw(key)
  // 收集依赖
  track(target, TrackOpTypes.GET, key)
  // 使用原型方法，通过原始数据去获得该key的值。
  // wrap 即传入的 toReceive 或 toReadonly 方法，将获取的 value值转为响应式数据
  return wrap(getProto(target).get.call(target, key))
}
```

注意！在`get`方法中，第一个入参`target`不能跟`Proxy`构造函数的第一个入参混淆。

- `Proxy`函数的第一个入参`target`指的原始数据

- 在这个`get`方法中，这个`target`其实是被代理后的数据。

再看看其他几个例子：

```ts
// 集合类型的 .has() 拦截
function has(this: CollectionTypes, key: unknown): boolean {
  const target = toRaw(this)
  key = toRaw(key)
  track(target, TrackOpTypes.HAS, key)
  return getProto(target).has.call(target, key)
}

// 集合类型的 .size() 拦截
function size(target: IterableCollections) {
  target = toRaw(target)
  track(target, TrackOpTypes.ITERATE, ITERATE_KEY)
  return Reflect.get(getProto(target), 'size', target)
}
```

特别注意：`size` 是一个属性而不是方法哟 ~

如果你对 函数参数中传 `this` 有疑问，敬请前往[「TypeScript 文档此处」](https://www.typescriptlang.org/docs/handbook/functions.html#this-parameters)阅读相关解释。



#### 「迭代器方法」劫持

在许多语言中都有迭代器这个API，基本的思想就是 「不断的 `next()`遍历元素，最后来到最后一个元素 `end()`」，JS 也有对应的实现，具体请查看 [MDN 文档相关内容](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Iterators_and_Generators)。

这里让每次通过 `next()` 获取到的数据都变成响应式：

```ts
function createIterableMethod(method: string | symbol, isReadonly: boolean) {
  return function(this: IterableCollections, ...args: unknown[]) {
    // 获取原始数据
    const target = toRaw(this)
    // 如果是entries方法，或者是 map的迭代方法的话，isPair为true
    // 这种情况下，迭代器方法的返回的是一个[key, value]的结构
    const isPair =
      method === 'entries' ||
      (method === Symbol.iterator && target instanceof Map)
    // 获取原型 调用原型链上的相应迭代器方法
    const innerIterator = getProto(target)[method].apply(target, args)
    const wrap = isReadonly ? toReadonly : toReactive
    // 收集依赖
    track(target, TrackOpTypes.ITERATE, ITERATE_KEY)
    // 返回一个包装的迭代器，它返回从实际迭代器发出的值的响应式版本
    return {
      // 迭代器的实现协议
      next() {
        const { value, done } = innerIterator.next()
        return done
          ? { value, done }
          : {
              value: isPair ? [wrap(value[0]), wrap(value[1])] : wrap(value),
              done
            }
      },
      [Symbol.iterator]() {
        return this
      }
    }
  }
}
```

####   「for-Each」 劫持

```ts
function createForEach(isReadonly: boolean) {
  return function forEach(
    this: IterableCollections,
    callback: Function,
    thisArg?: unknown
  ) {
    const observed = this
    const target = toRaw(observed)
    const wrap = isReadonly ? toReadonly : toReactive
    track(target, TrackOpTypes.ITERATE, ITERATE_KEY)
    // 将传递进来的 callback函数插桩，让传入 callback的数据，转为响应式数据
    function wrappedCallback(value: unknown, key: unknown) {
      // 注意下方 绑定的 this 是响应式对象
      return callback.call(observed, wrap(value), wrap(key), observed)
    }
    return getProto(target).forEach.call(target, wrappedCallback, thisArg)
  }
}
```

`forEach`的用法是一个「参数表示每一个可迭代对象每个元素」的回调函数，`callback.call(observed, wrap(value), wrap(key), observed)`，使这个参数变成了响应式数据。



#### 「写」相关操作

```ts
// 集合类型的 .add() 拦截
function add(this: SetTypes, value: unknown) {
  // 获取原始数据
  value = toRaw(value)
  const target = toRaw(this)
  // 获取原型
  const proto = getProto(target)
  // 用原型的 has 判断是否有这个key
  const hadKey = proto.has.call(target, value)
  // 通过原型方法，增加这个 key
  const result = proto.add.call(target, value)
  // 没有这个 key的话 说明真的是新增，触发 ADD 的监听逻辑
  if (!hadKey) {
    /* istanbul ignore else */
    if (__DEV__) {
      trigger(target, TriggerOpTypes.ADD, value, { newValue: value })
    } else {
      trigger(target, TriggerOpTypes.ADD, value)
    }
  }
  return result
}

// 集合类型的 .set() 拦截
function set(this: MapTypes, key: unknown, value: unknown) {
  value = toRaw(value)
  key = toRaw(key)
  const target = toRaw(this)
  const proto = getProto(target)
  const hadKey = proto.has.call(target, key)
  const oldValue = proto.get.call(target, key)
  const result = proto.set.call(target, key, value)
  /* istanbul ignore else */
  if (__DEV__) {
    const extraInfo = { oldValue, newValue: value }
    if (!hadKey) {
      trigger(target, TriggerOpTypes.ADD, key, extraInfo)
    } else if (hasChanged(value, oldValue)) {
      trigger(target, TriggerOpTypes.SET, key, extraInfo)
    }
  } else {
    if (!hadKey) {
      trigger(target, TriggerOpTypes.ADD, key)
    } else if (hasChanged(value, oldValue)) {
      trigger(target, TriggerOpTypes.SET, key)
    }
  }
  return result
}
```

写相关的操作其实比较容易，相比于基本类型的数据可以通过 `Reflect` 很方便地拿到，这里集合类型需要手动获取原型链并绑定`this`而已。

## Ref 篇

> 有的时候，我们也希望对某些「非对象型」的值做监听，那么根据 Reactive 篇的经验，不难想到方案，即：使其变为对象，值存放在 `value` 字段中，然后挂上 Proxy 代理？

不要急着得出结论，还是一起走进源码耐心看看吧！

### Unwrap · 包裹解套

最应该先看的就是这部分，[用到了 TS 的 infer，如果你不了解建议先看 TS 这部分的官方文档。其实简单点说就是：能推断出某个带泛型的类型带的到底是什么类型。](http://www.typescriptlang.org/docs/handbook/advanced-types.html#type-inference-in-conditional-types)

```ts
// 基础类型无需解套
type BaseTypes = string | number | boolean

// 递归地打开嵌套的值绑定。
export type UnwrapRef<T> = {
  cRef: T extends ComputedRef<infer V> ? UnwrapRef<V> : T
  // T 如果是 Ref<V> 类型，继续解套 V 直到终点
  ref: T extends Ref<infer V> ? UnwrapRef<V> : T
  // T 如果是 Array<V> 类型，继续解套 V 直到终点
  array: T extends Array<infer V> ? Array<UnwrapRef<V>> & UnwrapArray<T> : T
  // T 如果是 object 类型，它的 key 是 K 类型，继续解套 T[K] 直到终点
  object: { [K in keyof T]: UnwrapRef<T[K]> }
}[
  T extends ComputedRef<any> ? 'cRef'
	: T extends Ref ? 'ref'
  : T extends Array<any> ? 'array'
  : T extends Function | CollectionTypes | BaseTypes ? 'ref' // 
  : T extends object ? 'object' : 'ref'
]
```

最先我在开始从`vue-next`中抽取源码时，这部分我没有第一时间取出查看，对需要使用它或与其相关的内容时，我都暂时用其包裹的泛型 `T` 来表示，但在以下这个单测中就出了问题：

```ts
// 如果某对象为数组类型，且数组内元素长度、类型固定，视为 多元组类型：
// ref() 它的结果应该保留 其中各个元素类型信息
it('should keep tuple types', () => {
  const tuple: [number, string, { a: number }, () => number, Ref<number>] = [
    0,
    '1',
    { a: 1 },
    () => 0,
    ref(0)
  ]
  const tupleRef = ref(tuple)

  tupleRef.value[0]++
  expect(tupleRef.value[0]).toBe(1)
  tupleRef.value[1] += '1'
  expect(tupleRef.value[1]).toBe('11')
  tupleRef.value[2].a++
  expect(tupleRef.value[2].a).toBe(2)
  expect(tupleRef.value[3]()).toBe(0)
  tupleRef.value[4]++										// 在这一行 报错了！
  expect(tupleRef.value[4]).toBe(1)
})
```

由于我对 `tuple` 的定义，`tupleRef.value[4]`会是 `Ref<number>`类型，所以 TS 编译器报错说 `++` 这样的自增运算符不可用于此类型。

这个看起来好长好长的 `type`定义的语义，就是`UnwrapRef`可以帮助我们对包裹的值进行 **「解套」**，使得 `Ref` 类型的 `value` 值能像原来的类型一样正常使用。



### Definition · 接口定义

```ts
export interface Ref<T = any> {
  // * 这个字段是必要的，它允许 TS 将一个 Ref与一个恰好有 “value” 字段的普通对象区分开来。
  // 但是，在任意对象上检查符号要比检查普通属性慢得多，
  // 因此我们在实际实现中为 isRef()检查使用 _isRef普通属性。
  // * 而不在接口中声明 _isRef的原因是，我们不希望这个内部字段泄漏到用户空间的自动完成
  // 一个私有符号正好实现了这一点。
  [isRefSymbol]: true
  value: UnwrapRef<T>
}
```



接下来是 `ref()` 这个 Composition API 中主要使用到的 API 的定义：

```ts
// 主要 API：定义 Ref 型的值
export function ref<T extends Ref>(raw: T): T
export function ref<T>(raw: T): Ref<T>
export function ref<T = any>(): Ref<T>
export function ref(raw?: unknown) {
  if (isRef(raw)) {
    return raw
  }
  raw = convert(raw)
  const r = {
    _isRef: true,
    get value() {
      track(r, TrackOpTypes.GET, 'value')
      return raw
    },
    set value(newVal) {
      raw = convert(newVal)
      trigger(
        r,
        TriggerOpTypes.SET,
        'value',
        __DEV__ ? { newValue: newVal } : void 0
      )
    }
  }
  return r
}
```

果然，我们在`ref()`中并没有看到 Proxy API 的操作痕迹，因为这本身就是我们自己创造的对象，其实并不需要 Proxy 这么重的方案，基础值类型的数据本来也没有太多操作，仅仅是 `get` 和 `set` 需要被监听就好了。



### Deconstruction Optimization · 解构优化

现在似乎在咱们的响应式系统中，即使是基础值也可以是响应式的了！但 ... 我们还差点忘了这种情况：

```ts
const p = reactive({
  x: 123,
  y: 0
})
const { x, y } = p
```

这样的情况下，`x` 和 `y` 都是 `number` 类型，被解构出来后，就失去了响应式特性，所以我们需要为此情况提供一种解决方案：**「若要解构，且要保证响应式，那么请全体挂 `Ref`」**，写法如下：

```ts
const p = reactive({
  x: 123,
  y: 0
})
const { x, y } = toRefs(p)
```

但你阅读源码到 `toRefs` 的实现时你可能会感到困惑：「为什么它给每个属性对应的值挂 `Ref` 不是用 `ref()` API 而是用 `toProxyRef`，且这个 `toPrxoyRef`中也没有 `track` 和 `trigger`来做跟踪和触发？」

这是因为 `p` 本身就已经是响应式对象，我们挂 `Ref` 的目的是让`x`和`y`基本值类型变为对象类型，只是套一个壳子而已！

```ts
// 将一个普通对象中某属性对应的值转化为 Ref 型
// 返回该 Ref 型值
function toProxyRef<T extends object, K extends keyof T>(
  object: T,
  key: K
): Ref<T[K]> {
  return {
    _isRef: true,
    get value(): any {
      return object[key]
    },
    set value(newVal) {
      object[key] = newVal
    }
  } as any
}
```

至于什么 `track` 还有 `trigger` 的事儿，在运行到这里的 `get`和`set`时就会触发原来响应式对象 `handler` 里的句柄啦！

顾名思义嘛，为什么要叫 `toProxyRef`？即 **一个去触发 Proxy Handler 的 Ref 壳子**。



## Effect 篇

**Effect** 直译过来是 "影响，效果，达到目的"，是一个由用户自定义的函数，会在依赖被`trigger`触发时执行。

在 2.x 的源码中，依赖收集是通过 `Dep` 和 `Watcher` 两个类完成的，通过在 `defineReactive` 当中对目标对象每一个属性都新建一个 `Dep`，用于保存所有依赖此属性的 **订阅者** `Watcher`。

那么在 3.0 中是怎么做的呢？还是从单测看起吧：

```ts
// 在 响应式值 更新时 触发effect
it('should trigger effect when updating a reactive value', () => {
  const original = {
    a: 0,
    b: 'Hello'
  }
  let observed = reactive(original)
  let dummy;
  effect(() => {
    dummy = observed.a
  })
  expect(dummy).toBe(0)
  observed.a = 1
  expect(dummy).toBe(1)
})
```

仔细读完之后，应该会发现第 11 行：我们本没有初始化 `dummy`，它应该为 `undefined`，但经过了 `effect()` 的定义后，我们却期待它为 `0`，这是为何？

唯一的解释就是我们传入的这个自定义「效果函数」被执行了一次，使 `observed.a` 把值 `0` 赋给了 `dummy`。

“欸！你读取了一次 `a` 属性！应该触发 `observed`的 `get` ！”

嘿嘿 😏️！恭喜你发现了重要的知识点：「**定义了效果函数后，会立即执行一次！**」由此获得一次 `track` 跟踪。

### effect · 定义效果函数的方法

```ts
export function effect<T = any>(
  // 原始函数
  fn: () => T,
  // 配置项
  options: ReactiveEffectOptions = EMPTY_OBJ
): ReactiveEffect<T> {
  // 如果该函数已经是监听函数了，那赋值fn为该函数的原始函数
  if (isEffect(fn)) {
    fn = fn.raw
  }
  // 创建一个监听函数
  const effect = createReactiveEffect(fn, options)
  // 如果不是延迟执行的话，立即执行一次
  if (!options.lazy) {
    effect()
  }
  // 返回该监听函数
  return effect
}
```

```ts
// 创建监听函数的方法
function createReactiveEffect<T = any>(
  fn: () => T,
  options: ReactiveEffectOptions
): ReactiveEffect<T> {
  // 创建监听函数，通过run来包裹原始函数，做额外操作
  const effect = function reactiveEffect(...args: unknown[]): unknown {
    return run(effect, fn, args)
  } as ReactiveEffect
  
  // 省略其他配置代码...
  return effect
}
```

```ts
// 监听函数执行器
function run(effect: ReactiveEffect, fn: Function, args: unknown[]): unknown {
  // 如果这个 active 开关是关上的，那就执行原始方法，并返回
  if (!effect.active) {
    return fn(...args)
  }
  // 如果监听函数栈中并没有此监听函数，则：
  if (!effectStack.includes(effect)) {
    // 先预先清除它可能存在的依赖引用，准备重新添加进依赖栈
    cleanup(effect)
    try {
      // 将本 effect推到 effect栈中
      effectStack.push(effect)
      activeEffect = effect
      // 执行原始函数并返回
      return fn(...args)
    } finally {
      // 执行完以后将 effect从栈中推出
      effectStack.pop()
      activeEffect = effectStack[effectStack.length - 1]
    }
  }
}
```

嚯！这三段代码看得人真头大🤦🏻，梳理一下：

执行`createReactiveEffect()`来创建依赖

- 创建时通过执行 `run()` 方法使得 TS 确认行内创建的 `reactiveEffect` 返回值为 `fn` 的返回值 `T` 类型

  - `run()`方法中除了执行用户定义的「效果函数」，还有对 **依赖栈** 的操作：首先先将 `effect` 推入一个全局栈中，执行原始函数返回结果后将刚才推进来的 `effect` 推出。

    **Q：**那照这么说，执行前进来，执行后出去，为啥还需要判断 `!effectStack.includes(effect)` 呢？

    **A：**如果 `fn()`函数中有对依赖数据的更改，就会导致递归，反复触发 effect 啦！所以我们需要此判断。

那么 ... 我们说了好多次这些个什么`track`和`trigger`了，它们的实现又是怎么样的呢？



```ts
// 这个主要的 WeakMap 用来存储 {target -> {key -> dep} } 目标对象某个 key 属性的依赖.
// 从概念上讲, 把 Dep 依赖作为要一个类更好理解
// 它是一个包含着订阅者们的集合, 但是我们简单存储它为原生 Set 以节省内存开销
type Dep = Set<ReactiveEffect>
type KeyToDepMap = Map<any, Dep>
const targetMap = new WeakMap<any, KeyToDepMap>()
```

### track · 追踪与依赖收集

首先要想让 `track`  收集到依赖，那么得要有容器保存依赖，👆🏻️上面的注释已经写的很清楚了，保存的就是  **目标对象与属性的关系**、 **属性和它的依赖集合**。

因为上面的 `run()` 函数中记录了当前活跃的 `activeEffect`，那么在执行了用户定义的「效果函数」 `fn()` 时触发 `track` 跟踪，将 `activeEffect` 推进 `target[key]` 的依赖集合中。

```ts
// 收集依赖的函数
export function track(
  // 原始数据
  target: object,
  // 操作行为
  type: TrackOpTypes,
  key: unknown
) {
  // 如果 shouldTrack开关关闭，或 effectStack中不存在监听函数，则无需要收集
  if (!shouldTrack || activeEffect === undefined) {
    return
  }
  // 获取 target 的 {key -> deps}，如果无，则初始化
  let depsMap = targetMap.get(target)
  if (depsMap === void 0) {
    targetMap.set(target, (depsMap = new Map()))
  }
  // 获取 effect集合，无则初始化
  let dep = depsMap.get(key)
  if (dep === void 0) {
    depsMap.set(key, (dep = new Set()))
  }
  // 如果集合中没有刚刚获取的最后一个 effect，则将其 add到集合 dep中
  // 并在 effect的 deps中也 push这个 effects集合 dep
  if (!dep.has(activeEffect)) {
    dep.add(activeEffect)
    activeEffect.deps.push(dep)
    if (__DEV__ && activeEffect.options.onTrack) {
      activeEffect.options.onTrack({
        effect: activeEffect,
        target,
        type,
        key
      })
    }
  }
}
```



### trigger · 句柄事件触发

这部分的逻辑简单得令人惊讶😂️，当依赖收集表的三维关系建立完成后，执行 `set`，`delete` 这些方法时就会触发、执行用户在 `effect` 处定义的句柄函数。

```ts
// 触发监听函数的方法
export function trigger(
  target: object,       // 原始数据
  type: TriggerOpTypes, // 写操作类型
  key?: unknown,        // 属性key
  extraInfo?: DebuggerEventExtraInfo // 拓展信息
) {
  // 获取原始数据的响应依赖映射，没有的话，说明没被监听，直接返回
  const depsMap = targetMap.get(target)
  if (depsMap === void 0) {
    // 从未被跟踪过
  	return
  }
}
```

> 这里有个 TypeScript 赋予的特性，为避免环境中 `undefined` 值被污染，可以使用 `void 0` 来表达「空」。

最后我们小结一下，一图流让你看明白整个过程：

![响应式一图流](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-01-19-102528.png)

## Computed 篇

我们还是从主要的 API 的单测看起：

```ts
it('should return updated value', () => {
  const value = reactive<{ foo?: number }>({})
  const cValue = computed(() => value.foo)
  expect(cValue.value).toBe(undefined)
  value.foo = 1
  expect(cValue.value).toBe(1)
})
```

Vue 3 中「计算属性」的写法被 Hooks 化了。我们就像给 `effect`传句柄函数那样， 给 `computed` 传入一个计算函数即可。

接下来我们看看具体实现，之后我们还将讲到对实现中的一些重点难点的几个比较重要、比较有代表性的单测：

```ts
// !! 主要 API
// computed 泛的这个 T 是 getter 返回值的类型
export function computed<T>(getter: ComputedGetter<T>): ComputedRef<T>
export function computed<T>(
  options: WritableComputedOptions<T>
): WritableComputedRef<T>
export function computed<T>(
  getterOrOptions: ComputedGetter<T> | WritableComputedOptions<T>
) {
  let getter: ComputedGetter<T>
  let setter: ComputedSetter<T>

  // getterOrOptions 可能有两种情况：
  // 1. 就只是一个平常的 计算函数 返回结果值
  // 2. 一个含有 get 和 set 的对象
  if (isFunction(getterOrOptions)) {
    getter = getterOrOptions
    setter = __DEV__
      ? () => {
          console.warn('Write operation failed: computed value is readonly')
        }
      : NOOP  // NOOP = () => {} 表示无操作
  } else {
    getter = getterOrOptions.get
    setter = getterOrOptions.set
  }

  let dirty = true
  let value: T

  // 一个专门用来跑 计算属性 getter 句柄函数的执行器
  // 将 做计算的函数 传给 effect，并给予一些特殊配置！
  const runner = effect(getter, {
    lazy: true,
    // 标记 effect 为计算属性，以便在触发期间获得优先级
    // 为什么会有优先级？参见 effect.ts 254 至 258 行左右
    computed: true,
    scheduler: () => {
      // 这个行内函数作为 scheduler 通过闭包缓存了 dirty 的引用
      // 每次 数据有变化时 这里的 dirty 先变化
      // ⬇️️ 进入下方的 get 的 if 分支执行一次 runner
      dirty = true
    }
  })

  return {
    // 计算属性值 算作 Ref
    _isRef: true,
    // 暴露 effect 使计算属性停止计算
    effect: runner,
    get value() {
      if (dirty) {
        value = runner()
        dirty = false
      }
      // TCR: 当计算属性的 effect 涉及触发另一个 父级 effect，这个响应因子
      // 应当跟踪这个计算属性的所有因子.
      // 对涉及触发 另一个计算属性 也是同样的。
      trackChildRun(runner)
      return value
    },
    set value(newValue: T) {
      setter(newValue)
    }
  } as any
}
```

「因上面的代码注释比较详尽我就不一一讲述了。」

我们都知道计算属性一般来说是只读的，不过也可以自定义它的 `getter` 和 `setter`，体现在这里的 `getterOrOptions`。

计算属性发生变化，一定是其计算因子发生了变化，所以一定有一个与定义此计算属性同时定义好的 `effect` 来触发这些变化，即我们的 `runner`。

`TCR 即 trackChildRun` 是非常有必要的，我们看到下面这个单测：

```ts
it('should trigger effect when chained', () => {
  const value = reactive({ foo: 0 })
  const getter1 = jest.fn(() => value.foo)
  const getter2 = jest.fn(() => {
    return c1.value + 1
  })
  const c1 = computed(getter1)
  const c2 = computed(getter2)

  let dummy
  effect(() => {
    dummy = c2.value
  })
  expect(dummy).toBe(1)
  expect(getter1).toHaveBeenCalledTimes(1)
  expect(getter2).toHaveBeenCalledTimes(1)
  value.foo++
  expect(dummy).toBe(2)
  // should not result in duplicate calls
  expect(getter1).toHaveBeenCalledTimes(2)
  expect(getter2).toHaveBeenCalledTimes(2)
})
```

`c2` 是依赖于 `c1` 的，`c1` 依赖于 `value`，当 `value` 发生变化，应该链式触发使得 `c2` 也更新。

实现的原理就是将子级 `runner` 所有保存的依赖（即当前 `effect`涉及的引用）拷贝到当前 `effect` 中即可：

```ts
// 见上方 TCR 注释 ...
function trackChildRun(childRunner: ReactiveEffect) {
  if (activeEffect === undefined) {
    return
  }
  for (let i = 0; i < childRunner.deps.length; i++) {
    const dep = childRunner.deps[i]
    if (!dep.has(activeEffect)) {
      dep.add(activeEffect)
      activeEffect.deps.push(dep)
    }
  }
}
```



> **心得体会：**从2019年10月份 `vue-next` 代码公开，到 2020年此刻我整理完本模块的解析，已经过去了近3个月的时间，其实在这段过程中 Vue 3 还在不断地进行着优化和收尾工作。
>
> 总算完成了自己这么久以来欠的文章，感到非常高兴！如果你对本模块的代码解析中任何一处说法有疑问或有纠错，非常欢迎你联系我，我会尽快改正！[不如来 Anx 仓库发一个 Issue 吧！](https://github.com/ShenQingchuan/Anx/issues)
>
> 目前国内对 Vue 3 源代码的解读大多还只是 `reactivity` 模块，希望有更多大神来参加，我其实对有尤大在 Vue Conf 上提到的那个 "Block Tree" 的 Vitrual DOM 优化思路很感兴趣，希望之后会有更多解密！学到就是赚到！

## 参考资料：

- [掘金专栏 - 「vue3响应式源码解析-Reactive篇 - 蚂蚁保险体验技术」](https://juejin.im/post/5da9d7ebf265da5bbb1e52b7)
- [掘金专栏 - 「vue3响应式源码解析-Ref篇 - 蚂蚁保险体验技术」](https://juejin.im/post/5d9eff686fb9a04de04d8367)
- [掘金专栏 - 「vue3响应式源码解析-Effect篇 - 蚂蚁保险体验技术」](https://juejin.im/post/5db1d965f265da4d4a305926)
- [Bilibli -「Vue3响应式系统工作过程」](https://www.bilibili.com/video/av79489505?t=20)