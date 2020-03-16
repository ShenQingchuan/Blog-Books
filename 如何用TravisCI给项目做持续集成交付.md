# 如何用 TravisCI 给项目做持续集成交付 - NodeJS 版

## 持续集成交付是什么？

**持续集成： ** 强调开发人员提交了新代码之后，立刻自动的进行构建、测试。根据测试结果，我们可以确定新代码和原有代码能否正确地集成在一起。

![持续集成](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-03-16-100815.jpg)

**持续交付：** 在持续集成的基础上，将集成后的代码部署到更贴近真实运行环境的「类生产环境」中。交付给质量团队或者用户，以供评审。如果评审通过，代码就进入生产阶段。

![持续交付](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-03-16-100904.jpg)



## 给项目添加 TravisCI 配置

如果你想让你的项目能够在每次向 Github Push 后，由 TravisCI 平台帮你自动自动执行这一套「测试、打包、发布部署」的流程，那么你需要在项目中添加一个名字叫（**注意！必须叫这个名字**）: `.travis.yml`

> 众所周知，YAML 格式的文件由于历史问题，有 `.yaml` 和 `.yml` 两种后缀名。但是请注意，经过我曾经的失败经验教训， TravisCI 只认 `.yml` 这个后缀名哦！



当然，由于项目使用的技术栈不尽相同，我很难给大家演示出所有语言的配置情况，不过 TravisCI 网站的文档中有较为翔实的教程（但是可惜是完全英文），网上也可以找到不少的对应语言的配置模板。

但总之配置文件的书写都遵循以下的流程：

- 项目基于的程序设计语言：对应 YAML 字段 `language`，给出运行的主要环境

- `install`：安装依赖。

  大多数的现代软件工程项目中，都引入了大量的第三方包作为依赖，比如做 Java 开发的同学经常用 Maven、Gradle 等工具，导入了诸如 FastJSON 这样的库、做 Node.js 的同学们可能也都经常使用 npm 或者 yarn 来向 `node_modules` 中添加依赖 ...

  由于这些第三方依赖库数量繁多，虽然我们在本地开发时需要它们，但一般在远端仓库当中并不会存储它们，而是用一个清单来罗列它们的名字，等到部署时才统一再次拉取下来。

- `script`：该字段指定要运行的脚本，`script: true`表示不执行任何脚本，状态直接设为成功。

  

  关于整个运行过程的过程，TravisCI 提供了一整套所谓的「**生命周期**」，也就是一套先后顺序，其中的每一种都是一个 YAML 字段：
  
   1. `before_install`：依赖安装之前
   2. `install`：依赖安装的指令配置
   3. `before_script`：命令脚本执行之前
   4. `script`：运行脚本的命令
   5. `aftersuccess ` 或者  `afterfailure`：脚本运行阶段 成功 或 失败 后
   6. [ 可选的 ] `before_deploy` ：如果你有部署的安排，那么这一步在部署执行前
   7. [ 可选的 ]` deploy`：部署的具体命令
   8. [ 可选的 ]` after_deploy`：部署之后
   9. `after_script`：命令脚本执行之后



## 光说不练难理解

以我的开源项目 Kahra-UI 为例，这是一个 Vue.js 的组件库项目，所以它需要的很明显是一个 Node.js 环境。

配置必须书写的有以下内容：

```yaml
language: node_js  # 指定运行环境的语言
node_js: 
  - lts/*          # lts -> Long Time Supported 长期支持版本
install:
  - yarn install   
  # 如果你不知道、不了解 Yarn, 那么我可以给你解释一下效果等同：npm install
```

而后我们就要书写一些「打包、部署」等过程中需要的指令了，也就是那些需要在命令行（或称终端）下输入的运行命令，不同的语言都有不同的启动方式：

Node.js 基于包管理工具，可以将一串常常的命令写在核心包管理配置文件 `package.json` 中，之后再使用简单的 `npm run xxx` 或者 `yarn xxx` 运行那个命令，比如在我的项目 `package.json` 中：

![package.json 的 scripts 字段](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-03-16-113344.png)

那么反映到 TravisCI 的配置中：

```yaml
language: node_js
node_js:
  - lts/*
install:
  - yarn install 			# 或者 npm install
script:
# - yarn test:unit    # 而我的项目暂时还没有写单元测试，所以并不需要这一步
  - yarn docs:build 	# 或者 npm run docs:build
  # 这一步 docs:build 实际上是在打包构建项目的文档，显然它也是一个网页项目
```



> TravisCI 的原理实际上就是利用了 Docker 这样的现代化 DevOps 运维工具，构建了一个沙箱环境。
>
> 一般一个软件工程项目打包后都会是一个独立的文件夹📁️，一般是 `dist` ，另外如果你输出的是一个二进制的可执行文件可能放在 `bin` 目录下，或直接是 `xxx.exe` 或者 Linux 下就是 `xxx` ，终端下到该目录 `./xxx` 就能运行。

所以运行了 `yarn docs:build` 命令之后，我们打包出了这个 `dist` 文件夹，那有什么方便的办法来部署呢？

如果你也像我一样需要部署一个由 `HTML + CSS + JS + 另一些静态资源` 构建的前端项目，那么 Github Pages 无疑是一个很好的选择，它是由 Github 为你的某个仓库提供的服务。

不用自己手动申请，完全可以让 TravisCI 帮你代工这些琐事，只需要在 `.travis.yml` 中添加如下配置：

```yaml
deploy:
  provider: pages             # 指定是 Github Pages 服务
  skip_cleanup: true					# 「建议为 true」也就是下一次构建时，不会完全清理掉上一次的
  local_dir: dist							# 指定打包输出的目录
  github_token: $GITHUB_TOKEN # 下面我们说到的「需要申请的令牌」
  keep_history: true					# 「建议为 true」保持历史
  on:
    branch: master						# 指定发布到仓库的哪一个分支
```



## 开始部署

### 第一步「申请之前所需的令牌」：

虽然你的 TravisCI 就是由你的 GitHub 登录的，但是对于这样一个访问仓库的操作，还是需要一些更谨慎、安全级别更高的措施，那就是 **Github Access Token（访问令牌）**

当你登录好 Github 后，访问 [你的个人设置页](https://github.com/settings/profile) 点击左边 👈🏻 菜单栏的 「 Developer Settings 」：

![开发者设置](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-03-16-113351.png)

之后是再点击跳转后页面左边 👈🏻 的 「 Personal access tokens 」：

![Personal access tokens](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-03-16-113410.png)

然后点击右边的 「 Generate new token 」:

![Generate new token](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-03-16-113357.png)

可能会叫你确认密码，然后就是对这个令牌的一些配置：

对于下面的勾选框 ✅️ ，请根据你自己的项目需要，**谨慎地** 选择开放允许的权限：

![配置 token](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-03-16-113401.png)

点击最下方 「Generate Token」确认后，你就得到了一个新的 Token，而这个 Token 的真实字符串值只会在这时给你显示出来，**注意！仅此一次！**点击右边的复制按钮，记得复制到你的备忘录上保存好！之后我们会用到。

![注意复制 Token](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-03-16-113417.png)



### 第二步「登录 TravisCI」：

说了这么久，终于到了登录 TravisCI 的时刻啦！

> 请注意！由于众所周知的 the Great Wall，访问该网站的速度奇慢无比，即使是梯子的 PAC 模式。
>
> **请务必使用「全局模式」**!!

![登录 TravisCI1](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-03-16-114023.png)

![登录 TravisCI2](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-03-16-114044.png)



接下来，要为你的仓库开启 TravisCI 的 Build ，需要在左边 👈🏻 操作栏中点击 + 加号：

![添加仓库](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-03-16-114402.png)

然后找到你的仓库，**把右边的开关点开：**

![开启 build](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-03-16-114555.png)

然后回到刚才的工作台，你就可以看到你的仓库已经准备就绪了，但是由于还需要配置申请到的 Token，所以准备去设置页面填写它：

![去设置页面](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-03-16-115454.png)

在设置页面下方的「 Environment Variables 」即环境变量：

> **Q：一般一些很机密的字符串、密钥🔐️都存储为 Linux 系统的环境变量，这是为什么？**
>
> 这些作为密钥的字符串，无论你写在代码里还是配置文件里，只要是明文，那么有心人就一定有办法能拿到。
>
> - 要么你选择加密它，成为一个哈希串难以被破解。
>
>   TravisCI 也提供了方法加密，详情可以参见 [ 阮一峰的此篇博客介绍的 「5.2 加密信息」](www.ruanyifeng.com/blog/2017/12/travis_ci_tutorial.htmlsd)
>
> - 要么你就把它配置成 **环境变量**，因为除非黑客完全攻破了你的服务器，拿到了你的 Shell，否则任何人都看不见它，只能知道它被存储在了某个名字的环境变量里。
>
>   这是相对最安全的办法。

![设置Token为环境变量](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-03-16-115839.png)

如上所见，我已经存好了 `GITHUB_TOKEN`，这个名字正是之前我们在 `.travis.yml` 中配置的那个名字！一定要一致哦，不然服务器就不认识啦。

之后就可以发动部署，开启你的持续集成交付之旅啦！



## 「推荐扩展阅读」

- [「阮一峰的网络日志 - 使用 TravisCI 进行持续集成交付」](www.ruanyifeng.com/blog/2017/12/travis_ci_tutorial.htmlsd)

- [「TravisCI 部署时遇到的错误」](https://segmentfault.com/a/1190000018254440)
- ... ... 还有更多等待你自己去发现 ~ 

