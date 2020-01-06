# Vue 历程纵览

> 其实 Vue3.0 响应式数据模式的代码实验，我从 10 月份开始就做了，但是因为学校的工作和一些其他七七八八的事情，很可惜被搁置了很久，开始写这篇文章的时候正好是圣诞节，正好看到尤大把 [vue-next](https://github.com/vuejs/vue-next) 仓库 README.md 中的 `Major TODO` 减少至只剩一个 `Server-side rendering` 了，我预感 2020 农历春节前后，Vue 3.0 就应该可以正式上线啦！

	从我听完 2018 年尤大在 VueConf 杭州的远程演讲开始，我就为 Vue 未来的新功能、新特性惊喜不已：

- 更快的运行

- 更优良的 Diff 算法

  从一开始的 **只改变有动态内容的组件** 细化到 **只改变有动态内容的最细单元**，新方案也在不断演进、优化

- 更好的 Typescript 支持

  **放弃了 Class Decorator API** 的方案，全面转投 Function-base 的 Composition API，`setup()` 和各项 Hook 函数在 PPT 上第一次被介绍时，让我震撼不已



另外一点，是我自己对 Vue 这个框架的一些看法：

> [尤大在 JSConf 新加坡站的演讲](https://www.bilibili.com/video/av58793314?from=search&seid=8415139419686769511)中说：
>
> **Vue 在极力追求框架设计当中的平衡**。

何以见得呢？我们可以和目前比较知名的 `React.js、Angular、Svelte` 几个框架做下对比：

- React 拥有相当活跃的社区，我个人愿意称之为：**轮子研发创意工坊**

  JSX、Redux、React Hooks、高阶组件、时间切片、CSS-in-JS 亦或是 React Native 等等... 可以说很多前端领域的新奇想法都来源于 React 社区，有的也成熟于此。

  尤大在新加坡演讲的 12:07 的一个吐槽十分真实：**你如果不怎么会 React 周边的附属生态，你很难说自己是一个 React Developer，而你去到 React 的官方网站，React 本身并不会教你这些东西，你需要去做大量搜索，构建、搭配起自己的 React 解决方案。**

  社区并没有提供一个推荐的方案，但还好，React 社区的潮流会。

- Angular 更像是 Google 的 Java 后端工程师搞出来的东西，虽然经过第一代裂变后的它脱胎原生于 Typescript，但厚重的开发体验、MVC 的强耦感使得它更适合构建一个安全稳定、值得信赖的 Web 软件，背靠大公司因而架构成熟，所以解决方案几乎都是框架内置提供了的。

  所以在国内 Angular 被评论为：**文档资料基本都为英文，遇到实际开发问题参考较少，框架体系复杂，学习曲线比较陡峭，需要更多试错** 这倒也不难理解了。

- Svelte 其实我之前一无所知，但在我去上海交大参加 VueConf 2019 时听到尤大的分享中提到：它并未使用 Virtual DOM，追求极致地编译化、优化性能。

  尤大认为：**追求对渲染结构的编译化，把用户对渲染的操控屏蔽在了模板层，用户没有机会接触更底层了。**

  当我自己在尝试写 RPZ UI 组件库，一边自己尝试一边调查市面其他 UI 库的源代码时，看到 Element UI 的组件有许多都不是用模板，而是直接采用了 `render function` ，实现了复杂的组件逻辑，我才领会到什么是：**JavaScript 和 Virtual DOM 的表现力**。

> 那么 Vue 在前端领域象限里，处在什么位置呢？

<img src="http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2019-12-25-072243.png" alt="2019 前端象限" style="zoom:25%;" />

尤大说是追求设计平衡，但其实 Vue 并不甘心做一个表现平庸的框架，根据他最近的微博称：**基于 Virtual DOM 的优化已经超越了 Svelte，有种破了次元壁的感觉，甚至 Svelte 的作者也看了 BenchMark，“不信也不行。”**

而之所以 Vue 于初学者而言学习曲线平缓，根本原因是适用于任何量级的开发，即使是大学里老师布置的周末 Web 设计作业，你也可以使用 `<script src="js/vue.min.js"></script>`轻松上手。