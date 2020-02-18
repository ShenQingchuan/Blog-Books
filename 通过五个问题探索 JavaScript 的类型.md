# 通过五个问题探索 JavaScript 的类型

> **关于 JavaScript 的类型，还有哪些知识点是需要掌握的呢？**
>
> 来尝试回答这么几个问题：
>
> 1. 为什么有的编程规范要求用 `void 0` 替代 `undefined` ？
> 2. 字符串有最大长度么？
> 3. 为什么 JavaScript 中 `0.1 + 0.2` 不等于 `0.3` ？
> 4. ES6 新加入的 `Symbol` 类型是什么东西？
> 5. 为什么给对象添加的方法能用在基本类型上？

JavaScript 的语言类型是语言方案规范中的定义，一共有七种。



「 1 」对于 `undefined` ，算是一个设计失误，**它并不是一个关键字！而是全局对象的一个属性！**，而 `undefined` 类型和 `Null` 类型一样，只有这么一个值 ...

> **什么又是全局对象？**
>
> - 在浏览器是 `window` 对象，在浏览器控制台你输入 `globalThis`，会出现：
>
>   ![image-20200215113537684](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-02-15-033538.png)
>   
> - 在 Node.js REPL 中输入 `globalThis`，会出现 `Object [global]`：
>
>   ![image-20200215113726825](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-02-15-033726.png)
>

在曾经一段时间，可以通过对其重新赋值进行篡改，引发安全性问题。

根据 MDN 文档，自 ECMAScript 5 标准以来，`undefined` 就是不能被配置、不能被重写的属性了。不过还是可以通过将 `undefined` 作为标识符（即变量名）来造成覆盖！

```js
// 你的项目里面，可不要这样做！

// 打印出 'foo string' PS：说明 undefined 的值和类型都已经改变
(function() {
var undefined = 'foo';
console.log(undefined, typeof undefined)
})()

// 打印出 'foo string' PS：说明 undefined 的值和类型都已经改变
(function(undefined) {
console.log(undefined, typeof undefined)
})('foo')
```

而 `void 0 ` 是一种比较好的替代方案：

 ![image-20200215113837333](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-02-15-033837.png)





「 2 」对于字符串 `String` 的最大长度限制，我们脑门一拍，想到的可能是「 **字符个数的最大长度** 」，但其实在 JavaScript 中，字符串的实现是由 UTF16 编码完成的，字符串的诸多操作：`charAt`，`charCodeAt` 和 `length` 等皆是针对 UTF16，故而字符串的最大长度制约于编码的长度。

> 现行的字符集国际标准，字符是以 Unicode  的方式表示的，每一个 Unicode 的码点表示一个字符，理论上Unicode 的范围是无限的。UTF 是 Unicode  的编码方式，规定了码点在计算机中的表示方法。
>
> `0-65536（U+0000 - U+FFFF）` 的码点被称为基本字符区域，这个区域 UTF16 可以定长表示。

JavaScript 字符串把每一个 UTF16 单元当作一个字符处理，所以超出基本字符区域的时候要格外小心。

这个设计继承自 Java，为求得「 **性能更佳且实现方便** 」



「 3 」对于 `Number` 类型，就是我们通常意义上的 **“数字”** ，大致对应数学中的有理数，不过天下编程语言浮点都差不多，皆有精度限制。

对于臭名昭著的问题 `0.1 + 0.2 !== 0.3`，浮点数的运算精度问题导致等式左右并不完全相等。

正确度比较方法是：

```js
console.log( Math.abs(0.1 + 0.2 - 0.3) <= Number.EPSILON);
```



「 4 」对于 `Symbol` 类型，是 ES6 提出的新类型，在 ES6 中整个对象系统利用 `Symbol` 重塑了，也构成了语言的一类接口形式。它们允许编写与语言结合更紧密的 API。

`Symbol` 对象没法通过 `new Symbol()` 的方法构造，这和以往的其他类型的装箱类型相比，有些反常识 ... 只能使用 `全局的 Symbol 函数`：

```js
var mySymbol = Symbol('Kahra')
```

在我给 `@vuejs/vue-next` 的 PR `#709` 中就利用了相关知识：

为了在 `effect` 中给那些通过 `for ... of` 遍历的集合对象挂上依赖，重写了这些集合类型的生成器函数，在其中加入了 `track(target, ITERATE_KEY)`，在这个「迭代键」上开启了追踪。

对于 `for ... of` 循环，会使用到提案标准中的一个 Symbol 属性 `Symbol.iterator`，比如下面这个 0 到 9 的循环：

```js

var o = new Object()

o[Symbol.iterator] = function() {
  var v = 0
  return {
    next: function() {
      return { value: v++, done: v > 10 }
    }
  }        
};

for(const v of o) {
	console.log(v);  
}
// 0 1 2 3 ... 9
```

这里还利用了一些 Generator 的知识，我们按照迭代器要求，利用一个闭包函数，依次返回这九个数。



「 5 」JavaScript 对象是核心机制，同样也是是一个最复杂的议题，`Object` 表示一切有形无形的东西。

在 JavaScript 中，对象的定义是 **“属性的集合”** 。属性分为数据属性和访问器属性，二者都是 `key-value` 结构，`key` 可以是字符串或者 `Symbol` 类型。

几个基本类型在对象类型中都有对应的 “亲戚”：

- Number
- String
- Boolean
- Symbol

`Number`、`String` 和 `Boolean`，三个构造器是两用的，当跟 `new `搭配时，它们产生对象，当直接调用时，它们表示强制类型转换。

在调用类型中的方法时，对于基础值类型的值，它们会自动生成一个对应亲戚类型的对象，这种行为被称为「 **装箱转换** 」。由于会频繁产生临时对象，**在一些对性能要求较高的场景下，我们应该尽量避免对基本类型做装箱转换。**

可以看出，JavaScript 语言设计上试图模糊对象和基本类型之间的关系 ...



**事实上，“类型” 在 JavaScript  中是一个有争议的概念。**

一方面，标准中规定了运行时数据类型； 另一方面，JavaScript 语言中提供了 `typeof`  这样的运算，用来返回操作数的类型。但 `typeof` 的运算结果，与运行时类型的规定有很多不一致的地方。我们可以看下表来对照一下：

![](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-02-15-093202.jpg)