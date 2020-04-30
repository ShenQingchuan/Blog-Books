---
title: CentOS7 - 我的redis安装宝典
date: 2019-07-20 15:49:29
tags: 
  - Linux
  - 服务器运维
---



> **Redis** 是一个非常好用的缓存KV数据库，在很多Web开发的场合我们都需要用到它，所以我想在这里记录下来我安装 Redis 的一套流程作为避雷记录📝~



##  第一步：先跟着官网的指引，一步也别漏！

**如标题所示，朋友。**

👉让我们访问 Redis 的 [官方网站下载页面](https://redis.io/download) , 你会看到有三个版本的 Redis 可供下载：

- Unstable： **不推荐，因为这是开发版，可能会有很多开发时期的变动，更新频繁不利于服务器生产环境** ~
- **Stable**：我们选的就是他！请记住下面那个下载按钮上的版本号 ~
- Docker：如果你是玩Docker的大神你应该知道怎么部署，而我不是所以就不多说啦 ~



**但是为了方便 CentOS 服务器的操作，我们并不会选择下载安装包然后又用XFtp之类的工具再丢一遍**

我们会选择在服务器上使用 **wget / curl** 工具下载

在下载页上的 `Installation` 标题下，有如下的傻瓜式命令教程 ：

```bash
# 下载 Stable 版本的 redis 源码，准备从源码安装
wget http://download.redis.io/releases/redis-5.0.5.tar.gz

# 解压下载好的压缩包
tar xzf redis-5.0.5.tar.gz

# 进入到解压出的文件夹中
cd redis-5.0.5

# 执行 make 之前 请你确保你的服务器上有 GCC 工具
# 如果没有请先  yum install gcc
make
```

官网对于安装就只说了这么多，下面他就将教你怎么测试：

**👇下面你看到的 src 是你解压出的 redis-x.x.x 的 src 文件夹**

```bash
# 确保你自己是在 redis-x.x.x 的根目录下启动 redis 服务器程序
[root@Aliyun  redis-5.0.5]$ src/redis-server
```

启动之后你就会看到一个 **用字符打印出的 redis 的 logo**，并且程序一直在挂起执行，你没法回到命令行下，**除非你 ctrl + c 主动结束 redis 服务器程序** 



```bash
# 这里演示的是启动 redis 客户端
[root@Aliyun  redis-5.0.5]$ src/redis-cli
redis> set name "shenqingchuan"
OK 		# <- 这里是 K V 键值对设置成功后 redis 的提示
redis> get name
"shenqingchuan"			# <- 你可以使用 get 来得到某个键所对应的的值
```



## 第二步：你可是个数据库，你得一直在啊老兄！

**你当然会说，这有何难，开个 tmux 挂起运行不就完了吗，多大点事儿 ~**

可如果服务器重启了呢，你的缓存服务器可就挂了哦 ~ 所以还是将 redis 配置成为 service 服务更稳妥。

具体如何操作呢？请跟我一步步来：



1. 我们刚刚不是执行了 make 吗？可熟悉 make 的同学都知道，一般 make 过后都会有一步 install 

   ```bash
   # 进入 redis-x.x.x 源码文件夹的 src 目录下
   cd redis-5.0.5/src
   
   # 执行安装
   make install
   
   # 然后你可以测试一下 redis-server 是不是可以直接调用执行了
   redis-server
   # 显然应该打印出 redis 的 logo
   ```

2. 做完第一步，只能说明我们将 redis 安装到了 /usr/local/bin 这个目录下（熟悉 Linux 的同学都应该知道这个目录作为 PATH 的意义）

   但我们的目标是做成 service：

   ```bash
   # 首先做下面的操作时请确保你仍然还在 redis-x.x.x 的 src 目录下
   # 创建 redis 的专属目录
   mkdir -p /usr/local/redis
   
   # 拷贝两个编译完成的二进制可执行文件到 刚刚创建的目录中
   cp ./redis-server /usr/local/redis
   cp ./redis-cli /usr/local/redis
   
   # 回到 redis-x.x.x 的根目录，把 redis 的启动配置文件 也拷贝到专属目录中
   cd ..
   cp redis.conf /usr/local/redis/
   
   # 切换到 redis 专属目录, 编辑配置文件
   vi redis.conf
   
   ```

   大约在文件的第 128 行，可以找到一个 `daemonize no`

   我们将它改为：**daemonize yes**

   > **温馨小提示：**
   >
   > 如果你是个 vim 新手，不想一行一行慢慢看，因为这个文件里的注释实在是太多，而你的服务器环境里的 vim 不一定有行号：
   >
   > 那么你可以在命令模式下敲一个 `/`
   >
   > 然后输入 daemonize 就可以找到啦 ~ 
   >
   >  
   >
   > **看，为了学 redis 的配置，你还多学会了一招 vim 的查找 ~**

3. **添加开机启动服务，**其实就是给 systemd 多写个 service 文件 😄

   **`vi /usr/lib/systemd/system/redis-server.service`**

   以下内容复制即可，因为如果你照着我刚才的步骤做，你的 redis server 启动地址就是我们自建的专属文件夹：

   ```bash
   [Unit]
   Description=The redis-server Process Manager
   After=syslog.target network.target
   
   [Service]
   Type=simple
   PIDFile=/var/run/redis.pid
   ExecStart=/usr/local/redis/redis-server       
   ExecReload=/bin/kill -USR2 $MAINPID
   ExecStop=/bin/kill -SIGINT $MAINPID
   
   [Install]
   WantedBy=multi-user.target
   ```

4. **将服务启动并设置为开机自运行：**

   ```bash
   # 重新载入安装的服务
   systemctl daemon-reload
   
   # 启动 redis-server 服务
   systemctl start redis-server.service
   
   systemctl enable redis-server.service
   ```

5. **检查 Redis server 是否在运行：**

   ```bash
   [root@Aliyun ~]$ ps -A|grep redis
   ```

   结果应当显示一个 redis-server 进程正在运行。

   此时再去用 `redis-cli` 测试一下客户端是否可用，一切都没问题的话，恭喜你安装完成！~

