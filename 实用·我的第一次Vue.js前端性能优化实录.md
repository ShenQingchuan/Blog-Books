---
title: 📒实用·我的第一次Vue.js前端性能优化实录
date: 2019-08-07 02:29:13
tags: 
  - 前端
  - Vue

---

## Vue CLI3 下的文件打包注意事项

### 1. 缘起:

> 在我之前写 Vue.js 的好长一段时间里，见证了 Vue 的项目基础结构和 Vue CLI 3 的慢慢完善。其实对于我这样一个初学者来说，就像是试图在一条湍急的河流中生存下来。
>
> **你需要不断的学新东西、你需要不断地尝试、不断地犯错，因为只有这样你才能成长。**老是 `npm install --save ant-design-vue` 然后用着他们写好的组件，这的确也是工作的一部分，但却没能让人学到什么。
>
> #### 从 WebStorm：
>
> 在 vue cli 还不成熟的那段日子里，我记得 webstorm 默认生成的 vue project 文件夹里的 `package.json` 文件的 `scripts` 脚本中还是 `npm run dev`，实际执行的内容还是基于 webpack 的
>
> #### 到 Vue CLI3：
>
> 这期间我也没有注意过这件事情，毕竟对当下的我来说，完成一个项目的设计，实现需要做到的功能远比看懂一个项目框架背后的原理更重要，所以我很自然地就跟着官方文档，它说什么就是什么😄

#### 项目说明

本次的项目是一个报名SPA，本身用到的组件不多，用脚指头都能数的过来。大概的项目结构是这样的：

```bash
├── README.md
├── babel.config.js
├── package-lock.json
├── package.json
├── postcss.config.js
├── public
│   ├── favicon.ico
│   └── index.html
├── src
│   ├── App.vue
│   ├── assets
│   │   ├── jklogo.png
│   │   ├── project-info.json
│   │   └── sicnulogo.jpg
│   ├── components
│   │   ├── Signin.vue
│   │   └── SigninForm.vue
│   ├── icon.js
│   ├── main.js
│   ├── router.js
│   ├── store.js
│   └── views
│       ├── About.vue
│       ├── Check.vue
│       └── Home.vue
├── vue.config.js
└── yarn.lock

5 directories, 22 files
```

**说实话，以前我干的所有事儿都是在 `src` 文件夹下的。**不错，写代码撸组件的确只需要在 `views` 、`components`里肝就完事儿了。但是对于打包而言，根目录下那一群花花绿绿的配置文件才是关键哦😯~

因为我从来没有自己写过打包的配置，从来都是 `npm run build` 弄一个 dist 文件夹就往服务器上丢。在 vue 2 初期的时候，是手写 `webpack.config.js` 的，**而自从集成到了 vue cli3 之后，webpack  的相关配置就需要写在 `vue.config.js` 里了。**

 

### 2. 压缩与“丑化”：

当我第一次打包完成后，我的代码足足有 `12.5M`，所以我当时满脑子想的都是怎么样把代码压缩、minify...

![Minified Javascript code](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2019-08-06-F37593D6-F6D4-41D1-B9F7-A7DA59245474.png)



**首先你需要在根目录，新建一个 `vue.config.js` 文件，我管它叫 vue 总配置文件，它应当用 ES5 规范书写。**而要压缩代码成这样一副没法阅读的 ugly 模样，需要配置 webpack 的一个插件：`uglifyjs-webpack-plugin`，通过 npm 安装后：

```js
// vue.config.js
const path = require('path');
const UglifyJsPlugin = require('uglifyjs-webpack-plugin');

const isProduction = process.env.NODE_ENV === 'production';

module.exports = {
  // see https://github.com/vuejs/vue-cli/blob/dev/docs/webpack.md
  // chainWebpack: ...
  
  configureWebpack: (config) => {
    if (isProduction) {
      // -->...
      
      // 为生产环境修改配置...
      config.plugins.push(
        // 生产环境自动删除 console, 压缩代码
        new UglifyJsPlugin({
          uglifyOptions: {
            compress: {
              drop_debugger: true,
              drop_console: true,
              pure_funcs: ['console.log'],		// 这三个配置项可以消除console.log
            },
          },
          sourceMap: false,
          parallel: true,
        }),
      );
  }
}
```

- **configureWebpack 这一配置选项请参阅 [VueCLI3 官方文档  Webpack 配置项简介]([https://cli.vuejs.org/zh/guide/webpack.html#%E7%AE%80%E5%8D%95%E7%9A%84%E9%85%8D%E7%BD%AE%E6%96%B9%E5%BC%8F](https://cli.vuejs.org/zh/guide/webpack.html#简单的配置方式))** ，plugins 是插件的列表。

- **Uglifyjs 插件的相关配置：[请参阅 uglifyjs 的官方文档](https://github.com/webpack-contrib/uglifyjs-webpack-plugin)**



### 模块拆卸与CDN：

然而光是 Minify 了代码，我发现打包出来的 app.js 仍然有10多M，看来不得不继续找办法了。

当时我还不会用 webpack-analyer ，在网上看来了 **`npm run build --report`** , 但是没有打开生成的 `report.html` 。**所以小伙伴们记住啦~ 使用 `npm run build --report`  可以生成打包结果的体积分析！**

- 我只是傻乎乎地点开了 app.js ，那密密麻麻的字母数字一下子炸开锅了… 在一卡一卡地向下滑动中，我发现似乎有一些图标的 SVG 被填充了进来，导致文件十分庞大。**这一定是与引入的 Icon 有关，无论你是使用 element-ui 亦或是我这里用到的 ant-design-vue，都有类似的问题。**
- 同时，作为 Vue.js 这样一个提供 CDN 引入的框架以及其附属全家桶，我们没有必要在打包时将它们引入，应该是让用户到访问页面时，让浏览器去请求 CDN 获取 Vue 的 Runtime、Vue-Router、Vuex … **因为访问 CDN 肯定比访问我们的小破服务器快呀 😂~**

所以我们需要继续增加配置：**在刚才的代码块 // … —> 的部分添加如下代码段**

**vue 配置代码演示1：**

```js
// 用cdn方式引入
config.externals = {
  vue: 'Vue',
  vuex: 'Vuex',
  'vue-router': 'VueRouter',
  axios: 'axios',
};
```

这个代码段的含义是，**当打包程序遇到引入这些包的时候，不打包到结果内**。

- 在最开始处声明 cdn 资源的URL：

**vue配置代码演示2：**

```js
const cdn = {
  css: [],
  js: [
    'https://cdn.bootcss.com/vue/2.6.6/vue.min.js',
    'https://cdn.bootcss.com/vue-router/3.0.2/vue-router.min.js',
    'https://cdn.bootcss.com/vuex/3.1.0/vuex.min.js',
    'https://cdn.bootcss.com/axios/0.18.0/axios.min.js',
  ],
};
```

- 在👆**代码演示 1** 中的 **`// chainWebpack ... `** 处加上以下代码段：

**vue 配置代码演示3：**

```js
chainWebpack: (config) => {
    config
      .entry('index')
      .add('babel-polyfill')  // 当然因为这里的需要，得 npm install --save babel-polyfill
      .end();
    
  	// ... 配置别名 ...
  
    if (isProduction) {
      // 生产环境打包
      // 删除预加载
      config.plugins.delete('preload');
      config.plugins.delete('prefetch');
      // 压缩代码
      config.optimization.minimize(true);
      // 分割代码
      config.optimization.splitChunks({
        chunks: 'all',
      });
      // 生产环境注入cdn
      config.plugin('html')
        .tap((args) => {
          args[0].cdn = cdn;
          return args;
        });
    }
  },
```

- 接下来我们还需要到 **`public/index.html`** 中去引入 Vue.js 相关的 cdn资源：

  在 index.html 的 **`<header> </header>`** 里添加如下的代码段：

  ```ejs
  <!-- 使用CDN的CSS文件 -->
  <% for (var i in htmlWebpackPlugin.options.cdn && htmlWebpackPlugin.options.cdn.css) { %>
    <link href="<%= htmlWebpackPlugin.options.cdn.css[i] %>" rel="preload" as="style">
    <link href="<%= htmlWebpackPlugin.options.cdn.css[i] %>" rel="stylesheet">
  <% } %>
  <!-- 使用CDN的JS文件 -->
  <% for (var i in htmlWebpackPlugin.options.cdn && htmlWebpackPlugin.options.cdn.js) { %>
  	<link href="<%= htmlWebpackPlugin.options.cdn.js[i] %>" rel="preload" as="script">
  <% } %>
  ```

  在 **`<div id="app"> </div>`** 的下方添加如下的代码段：

  ```ejs
  <!-- built files will be auto injected -->
  <% for (var i in htmlWebpackPlugin.options.cdn && htmlWebpackPlugin.options.cdn.js) { %>
  	<script src="<%= htmlWebpackPlugin.options.cdn.js[i] %>"></script>
  <% } %>
  ```



- 此时模块已经**被拆分得差不多了**，剩下要做的一件事情就是：**不要整体导入整个 Icon 库，需要的图标按需导入：**

  推荐大家看一看这个由 AntDesignVue 作者团队成员建的 [Icon优化指导仓库](https://github.com/HeskeyBaozi/reduce-antd-icons-bundle-demo) 

  1. 首先我们需要在 **`src/`** 文件夹下新建一个 **`icon.js`** 文件，在此之中你可按需要从 **`node_modules/@ant-design/icons/lib/`** 中导出你想要的 Icon 图标

     ```js
     // 1.你不但需要导出 <a-icon> 所对应的图标
     // 2.你还需要导出其他组件可能用到的 icon
     // 以下是格式：
     export {
       default as SmileOutline
     } from '@ant-design/icons/lib/outline/SmileOutline';
     ```

  2. 在 **代码演示段 3 ** 中的 **…配置别名…** 处的：

     ```js
     // 配置别名
     config.resolve.alias
       .set('@ant-design/icons/lib/dist', resolve('./src/icon.js'));
     ```

     **你认真研究下 node_modules 中的`@ant-design/icons/lib/dist`实质上是个 `dist.js `**，也就是说曾经我们**默认全局引入**的 icons 整个大包就被咱们的 `icon.js` 偷换了。

     
     
| Before | After|
| ------ | ----- |
| ![压缩前](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2019-08-06-181023.png) | ![压缩后](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2019-08-06-181116.png) |
| ![压缩前结果](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2019-08-06-181142.png) | ![压缩后结果](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2019-08-06-181217.png) |



(≧ω≦)ゞ 可以看到咱们这个心头大患总算是解除了！~ (≧∀≦ゞ**

再运行 **`npm run build`** 后就大概会是如下的结果咯 ~

![最终打包](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2019-08-06-67445A80-61E7-48FC-927E-6AE386B4E5B2.png)



## 小结：

😄 其实前端性能优化，也没有想象的那么复杂嘛！~ **世上无难事，只怕有心人！**

Webpack 真的是门大学问，且不说它自己这么臃肿是好是坏，但是对前端开发者而言，总归是一门非常有利的工具！