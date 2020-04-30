---
title: Mongoose 的Type化 Typegoose库的使用
date: 2019-08-27 14:58:02
tags:
 - 前端
 - Typescript
 - Mongodb
---



## 😭 1. 在转 Typescript 之后用 Mongoose 的体验

当我决心把协会的API接口服务（基于Node.js + Koa2）开发转向 Typescript 之后，很多原来在 JavaScript 模式下使用的 npm外部包、库的写法用法都发生了一些小变化。

对于JavaScript来说，要连接 Mongodb **几乎没有比 Mongoose 没有更好的第二选择**。但是在用 Mongoose 官方 **`.d.ts`** 化版本时有许多不尽如人意的地方，罗列如下：

### Σ(⊙▽⊙"a 文档模型要建两遍！而且得分别维护

例如我们给出一个要建立一个用户集合，那么我要有用户的数据结构，**并据此由 Mongoose 来建立 model**，定义如下：

```typescript
/* Example 1 */
interface User {
  name?: string;
  age: number;
}

mongoose.model('User', {
  name: String,
  age: { type: Number, required: true },
});
```

你可以看到这样的代码看上去是非常没有表现力的，因为描述一个数据结构的属性我们竟然写了两遍。另外，**如果我们的文档关系之间有嵌套，比如我们还需要录入该用户的职位信息：**这样的代码就变得更加让人啼笑皆非了...

```typescript
/* Example 2 */
interface User {
  name?: string;
  age: number;
} // 这里同 Example 1
interface Job {
  title?: string;
  position?: string;
} // 这里是新加的
mongoose.model('User', {
  name: String,
  age: { type: Number, required: true },
  job: {
    title: String;
    position: String;  // 这里又把 Job 的内容重写了一遍...
  },
});
```

所以我想着有这么古怪的问题自然要找万能的开源社区啦！在 Github 简单寻找一番我便找到了 **社区维护的 Mongoose Typescript 实现 [Typegoose (戳此链接以查看)](https://github.com/szokodiakos/typegoose)**



## 🤣 2. Typegoose 里那些显而易见的优势

那么我们就刚才的例子，来看看 Typegoose 下是怎么写的吧：

```typescript
class Job {
  @prop()
  title?: string;

  @prop()
  position?: string;
}

class User extends Typegoose {
  @prop()
  name?: string;

  @prop({ required: true })
  age!: number;

  @prop()
  job?: Job;
}
```

代码明显地变得清爽了，数据结构之间的关系也更一目了然了。**它是采用了 [reflect-metadata](https://github.com/rbuckton/reflect-metadata) 来使得数据能够 change once, run everywhere 的。**



而在这里 Typegoose 文档中给出了这样一条提示：**Please note that sub documents do not have to extend Typegoose. You can still give them default value in `prop` decorator, but you can't create static or instance methods on them.**

比如我们这里的 Job 类，它是用户文档下的子文档，所以他不必继承 Typegoose 这个类，你依然可以使用 **`@prop`** 装饰器来描述一个文档字段，但是不能对它创建任何静态方法或是基于它的实例方法了。



## 😄3. 关于使用 Typegoose ...

如何使用其实大部分的 API 在 **[Typegoose 的 API 文档](https://github.com/szokodiakos/typegoose#api-documentation)** 中都有介绍，不过是英文版的有一点难啃，还望大家有点耐心。

#### 首先引入Typegoose中需要用到的函数：

```typescript
import { prop, Typegoose, pre } from 'typegoose'; //27.2k (gzipped: 8.3k)
```

#### 然后创建 “文档” 模型： Collection Class Defined by Document Model

这里你所定义的一些字段对应的是：**Mongodb中每条文档的各个字段**

```typescript
export default class 文档从属的集合名称 extends Typegoose {
  @prop()
  somethingA: 类型;
  
  @prop({required: true})  // 注明是新插入一条文档时必填的字段
  somethingB: 类型;
  
	...
}
```

另外我推荐一个从慕课网七月老师的 Python Flask 高级编程中学来的数据记录项的必备属性：**_createAt_ 和 _status_**，表达该数据记录项的 **创建时间 和 该记录当前的状态（可以用于软删除）**

```typescript
@pre<文档从属的集合名称>('save', function(next) {
  let now = new Date();
  if(!this._createAt){
    this._createAt = now;
  } // 保存时自动记录文档创建时间
  next();
})
export default class 文档从属的集合名称 extends Typegoose {
	...
}
```





#### 导出录入了该文档模型的集合类：

```typescript
// 以下是建议写法：
export const 文档从属的集合名称Model = new 文档从属的集合名称().getModelForClass(文档从属的集合名称);
// 以 RPZ学习内容展讲信息模型类举例：
export const SpeechInfoModel = new SpeechInfo().getModelForClass(SpeechInfo);
```

#### 定义了Model后在Controller中使用：

以 **`Koa2.js 的 Typescript 工程` ** 举例：因为 Koa 依赖 ctx 在洋葱模型中传递上下文，所以可以用一个异步函数来作为路由控制器：

```typescript
// 引入学习内容展讲记录文档模型
import { SpeechInfoModel } from '../mongodb/model/SpeechInfo';    

// 引入 Mongodb 执行结果封装格式化方法
// import { retpack } from '../mongodb/mongoutils';        
function retpack(action_result: Object|Array<any>, meta: Object=null) {
  return {
    "result": action_result,		// 真正返回的数据结果
    "meta": meta								// 服务端特意加上的一些描述信息
  }
}

// 引入 ctx上下文返回封装格式化方法
// import { resbundle } from '../apiutils';  
// 定义一个标准的 JSON 返回数据格式
function resbundle(ctx: any, log_type: string, error_code: number, msg: string, data: object) {
  console_logger(ctx, log_type, "API Result: " + msg);  
  // 什么？console_logger 不知道是啥？ 看下面吧！~
  
  return {
    error_code: error_code,		// 项目自定义的API返回结果的错误描述代码
    msg: msg,				// API返回的文字描述信息
    bundle_data: data		// API 返回的数据
  };
}
// (用的是 log4js 做日志库哟~)
let console_logger = (ctx: any, type: string, log_str:string) => {
    try {
        log_str = `${ctx.url} ${ctx.method} - ` + log_str;
    } catch(err) {
        //TypeError, ctx 可能是 null，因为可能记录的并不是上下文、路由函数内的日志
    }
    console.log(`${type} - ${log_str}`);
    switch (type) {
        case "info":
            logger.info(log_str); break;
        case "debug":
            logger.debug(log_str); break;
        case "error":
            logger.error(log_str); break;
    }
}

```

```typescript
// 以学习内容展讲信息 控制器举例：
export default class SpeechInfoController {
  /** /v1/speech/score
   * GET  取得个人展讲的余额分数
   * @params
   * - openid  所属干事的微信openid，据此来查找分数
   */
  async get_score_byopenid(ctx: any) {
    ...
    const pre_result: any = await SpeechInfoModel.findOne({officer: officer_record.id});
    ...
    const new_record = await new SpeechInfoModel({officer: officer_record.id}).save();
    ...
}
```

具体的业务逻辑我就不详细展示了，主要展示最关键的两行，因为 Typegoose 将查询封装成了一个 Promise，这就 **避免了写地狱式的回调**，用了 **`async / await`** 之后代码看上去更有表现力了。

1. **查询、更新和删除时：**

   是直接调用 Model 的静态方法，查询的几种方式基本和 Mongoose 的 JavaScript 版实现保持一致

   **`find / findById / findByIdAndDelete / findByIdAndUpdate / findByIdAndRemove / findOne / findOneAndDelete / findOneAndUpdate / findOneAndRemove  `**

2. **插入新的文档时：**

   请注意这里是 **`new Model的名字({这里以key-value形式传入所有数据项，注意不要漏掉必填的}).save()`**，而因为与删改查不同，删改首先得查，查不到只会返回空数组，而插入是可能报错的！所以记得用 **`try / catch`** 块包裹住。

   



## 另外再说NoSQL（Mongodb）与传统 SQL 的一些区别小心得

> 我这个人超级奇怪的，用MySQL的时候我想着只通过 id 做外键链接关系（~~其实就是对象间关系没学精所以不能运用自如🤣~~），用表间关系少到怀疑自己是在用NoSQL，而当我转而使用 Mongodb 时，却又会怀念起关系型数据库那一套完整而规范的理论来...

当我打开DBMS的周边管理工具 **DBeaver** 和 Mongodb 的 **Robo 3T** 时，我感觉可以对比着来看两种数据库模式：

![SQL](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2019-08-29-WeChatcc2f164ab47f9da76f1cb5b5ed4a5a78.png)

对SQL型数据库，如果我要在这个图形化界面工具里手动修改一条数据，那么操作的顺序依次是：

1. **打开数据库 rpzwxapp**
2. **打开表**
3. 找到**那一个条目**然后修改它的**某一个字段**



![NoSQL](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2019-08-29-WeChat8bc6ed805579f87155aa02488a49ae74.png)

而对于Mongodb数据库，操作的顺序依次是：

1. **打开数据库 rpzmongo**
2. **打开集合（Collection） rpzofficers**
3. 然后 Mongodb 就会执行 **`db.getCollection('rpzofficers').find({})`** 查询整张表，
4. 而我这里只有一条数据，**就相当于SQL的表中的一条记录**

故而：

- **`SQL Database ≈ Mongodb Database`**
- **`SQL Table ≈ Mongodb Collection`**
- **`SQL Dataitem = Mongodb Document `**

