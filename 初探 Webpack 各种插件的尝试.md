# 初探 Webpack - 各种插件的尝试

![](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-01-30-131015.png)

> 想要入门前端工程化，那么入门 Webpack 自然是必不可少的。Webpack 拥有将一切打包成为方便浏览器解析的静态资源，当然其配置文件也是其学习难点所在。
>
> Webpack 也是高度开放的，把大部分的活儿都交给了社区去做，所以要搭建起一个开发环境，我们需要给 Webpack 配置很多的插件。



## 基础概念

Webpack 基于一个名为 `webpack.config.js` 的 JavaScript 脚本文件来配置运行，核心就是导出一个对象，一般的开发环境的配置至少应该基于以下结构：

```js
module.exports = {
  entry: 'src/index.js',
  module: {
    rules: [
      // ...
    ]
  },
  output: {
    path: path.resolve(process.cwd(), "dist"),
    filename: 'js/[name].[chunkHash:8].js'
  },
  plugins: [
    // ...
  ],
}
```

### 入口点：`entry`

前端项目一般都会有一个所有 JS 脚本的总领文件 `index.js`，由它导出所有操作、数据解构等等... 相当于是所有脚本文件的根节点。

### 模块与插件 ：`module & plugins`

模块当中一般是根据后缀名来判断文件类型，然后使用相应的 **`loader`**，`loader` 是 Webpack 的重要概念，它发挥的作用也挺像是一个插件，但跟 `plugins` 却并不相同，它主要负责某一类文件的加载、打包，而插件的功能则更加灵活一些，还会做一些文件重整理、甚至是内容处理的工作。

### 输出位置：`output`

一般地，前端开发者都会选择将写好的 Web 项目输出到一个 `dist` 文件夹 📁️，里面一般有如下几种子文件和文件夹：

1. `css/` 样式文件夹，还可能名为 `styles/`

2. `js/` 脚本文件夹

3. `static/` 静态资源文件夹，还可能名为 `assets/`

   其中又细分为：

   - `images/` 图片文件夹
   - `fonts/` 字体文件夹
   - `svgs/` SVG 矢量图文件夹
   - `icons/` 图标文件夹

4. `index.html` 如果是 SPA（单页面应用）一般都是只有一个 HTML 文件，如果是用 Nuxt.js 等服务端渲染框架开发，最后生成多页面，则可能所有 HTML 文件都存放在名为 `views/`  的文件夹中

这样的结构部署十分方便，Nginx 等工具书写这类部署的配置甚至不超过 `5` 行 ... 😂️



## 打包命令

在 `package.json` 中，我们在 `scripts` 中新建一个命令：`build`:

```json
{
  "scripts": {
    "build": "webpack --mode production --config scripts/webpack.config.js"
  }
}
```

之后我们还会讲到，在开发环境 `mode `为 `development`时，可以使用 Webpack Dev Server 做开发时的热重载。

## 插件篇

### 处理视图

总的来说，浏览器要想呈现给你页面的效果，还是依托与 HTML 的「**骨架**」，辅以「**样式**」与「**数据**」，毕竟这是 Web 前端世界的三元老，雷打不动。

哪怕一个 Web 应用前端极其复杂（比如我之前 F12 看过 Bilibili 的前端代码，一个页面的 HTML 的 `<header/>` 里起码有上百个 `<style/>`），仍然不过是「一群`<link/>` ，一群`<script/>`」

Webpack 打包 HTML 的插件，比较流行的是 `html-webpack-plugin`

```bash
yarn add -D html-webpack-plugin
```

**这是我们讲到的第一个插件，你需要知道插件的配置书写在什么位置。**

`plugins` 是一个数组，若要使用插件，就直接 `new` 此插件构造器，并传入配置对象：

```js
{
  // 省略其他...
  plugins: [
  	new HtmlWebpackPlugin({
      title: 'Hello World!',
      template: 'public/index.html'
    }),
	]
}
```

`title`指的是要操作的 HTML 源文件中的 `<title>` 中的内容，`template` 的值应是源文件的路径，当然你还可以配置一个 `filename` 属性，作为最终生成的文件名。如果你面临多 HTML 文件的情况，可能需要考虑遍历项目内的所有 HTML 文件，依次配置 `new HtmlWebpackPlugin()` 



### 处理样式

样式的情况比较复杂，有很多预处理器可以使用，我自己比较偏爱 Less + PostCSS，那让我们看看应该怎么写吧：

```js
{
  // 省略其他...
  module: {
    rules: [
      {
        test: /\.less$/,
        use: [
          MiniCssExtractPlugin.loader,
          'css-loader',
          'less-loader',
          'postcss-loader'
        ]
      },
    ]
  }
}
```

Webpack 已经进入 4.0 时代，`mini-css-extract-plugin` 的前身 `extract-text-webpack-plugin` 仅支持到 3.x，它的作用是将样式内容提取成 `.css` 文件，然后让 HTML 文件引用之。

同时它还有一个配套插件：

```js
plugins: [
  new MiniCssExtractPlugin({
    filename: 'css/[name].[chunkHash:8].css'
  }),
]
```

之所以要在文件名前加 `css/` 这样表示文件夹的前缀，是因为当我们运行 `build` 命令时：

![](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-01-30-WeChat1135e03ba8a13e446c4a855628bd12fe.png)

打包上面这个结构的项目就会使 `index.html` 和 打包生成出来的 CSS 文件 在同一层级，这自然是不好的。样式文件在一个大项目中会非常多，规范是希望我们将所有样式文件都放在一个统一的文件夹里。

![](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-01-30-WeChat59289ed3ba2b698727d80ecae8562bda.png)

可以看到打包出来的 css 文件带上了 Hash 值，这是为了避免可能存在的 **同命名冲突** 等情况，

而关于 PostCSS，**PostCSS插件有很多优秀的预处理器功能**。而这些插件和 Less 这样的预处理器结合在一起工作会变得更好，一个典型的例子就是 PostCSS 的优秀插件 `autoprefixer` 可以帮助我们补充一些 CSS 属性在各个不同浏览器下的前缀，如 `--webkit-flex`这样的 ...

使用方法也十分简单，只要你把 `postcss.config.js` 写正确，再安装 `postcss-loader` 再添加到 `.less` 配置`use`中就完成啦！



### 打包清理

为了保证每次打包都会生成完全新的 `dist` ，我们需要把之前可能存在于 `dist` 文件夹中的文件清理掉，这时我们需要的插件是：`clean-webpack-plugin`

不过引入的时候有一个坑，需要用解构语法取出 `CleanWebpackPlugin`这个构造器 ，在这里写出来以作备忘：

```js
const { CleanWebpackPlugin } = require('clean-webpack-plugin');
```

之后直接插入进 `plugins`之后即可！

```js
{
  // 省略其他...
  plugins: [
  	new CleanWebpackPlugin(),
	]
}
```

