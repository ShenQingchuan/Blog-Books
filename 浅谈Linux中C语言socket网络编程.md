
# **Linux  C语言socket网络编程** 

注意：**本文是按照 TCP、UDP的工作过程进行总结的**

1. TCP套 **`socket`** 接口编程：

基于TCP的 **客户/服务器**（C/S）模式的工作过程如下：

![image](http://rpzoss.oss-cn-chengdu.aliyuncs.com/tmyBlog/2020-01-04-055200.png)

## 服务器进程中的一些函数：

### 1.**socket()：**

   ```c
   /*  函数所需头文件及其原型 */
   #include <sys/socket.h>
   int socket( int family, int type, int protocol);
   socketfd = soket(AF_INET, SOCK_STREAM, 0);
   /*  socketfd 作为返回值，可以记作描述符。
   	若 socketfd 非负则表示成功，为负则表示失败。
   	参数：
   		family   -> 指明协议族
   		type     -> 字节流类型
   		protocol -> 一般置0.
   	参数 family 的取值范围是：  
   		AF_LOCAL    UNIX 协议族 
   		AF_ROUTE    路由套接口  
           AF_INET    	IPv4 协议  
           AF_INET6    IPv6 协议  
           AF_KEY     	密钥套接口
       参数 type 的取值范围：   
           SOCK_STREAM   TCP 套接口  
           SOCK_DGRAM    UDP 套接口  
           SOCK_PACKET   支持数据链路访问  
           SOCK_RAM      原始套接口 
   */
   ```

生成套接口描述字（套接字）后，要为套接口的地址数据结构进行赋初值。

通用套接口地址的数据结构中，**struct sockaddr_in** 需要掌握：

   ```c
   struct in_addr  {    
       in_addr_t s_addr;       
       /*32 位 IP 地址，网络字节序*/  
   }; 
   
   struct sockaddr_in  {    
       uint8 sin_len;    
       sa_family_t sin_family;    
       in_port_t sin_port;              
       /*16 位端口号，网络字节序*/    
       struct in_addr sin_addr;             
       char sin_zero[8];                
       /*备用的域，未使用*/  
   };
   ```

   **PS：** 需要注意的是，一般在 **socket()** 之后，我们会填写 **`sockaddr`** 的相关内容。

   ```c 
   	/* Fill the local socket address struct */
   	memset (&servaddr,0,sizeof(servaddr));
   	servaddr.sin_family = AF_INET;           			// Protocol Family
   	servaddr.sin_port = htons (PORT);         			// Port number
   	servaddr.sin_addr.s_addr  = htonl (INADDR_ANY);		// AutoFill local address
   	
   ```

### 2. bind()：

   ```c
   	// 函数原型：
   	#include <sys/socket.h>  
   	int bind(int sockfd,const struct sockaddr *myaddr,socklen_t addrlen);
   	/*
   		参数 sockfd ：套接字描述符。 
   		参数 my_addr：指向 sockaddr 结构体的指针（该结构体中保存有端口和 IP 地址 信息）。 
   		参数 addlen：结构体 sockaddr 的长度。
           
           返回：0──成功， -1──失败 
   	*/
   	ret = bind(sockfd,(struct sockaddr *)&my_addr,sizeof(struct sockaddr)); 
   
   	/* 	功能：当调用 socket 函数创建套接字后，该套接字并没有与 本机地址和端口等 信息相连，
   		bind 函数将完成这些工作。
   	*/
   ```

### 3. listen()：

   ```c
   	//	函数原型：
   	#include <sys/socket.h>  
   	#include<sys/types.h>
   	// 	#define BACKLOG 10
   	int listen(int sockfd,int backlog);
   
   	/*
   		参数 sockfd ：套接字描述符。
   		参数 backlog ：规定内核为此套接口排队的最大选择个数。 
   	*/
   	ret = listen(sockfd,BACKLOG);
   	
   	// 通常采用一下的异常处理：
   	if(listen(listenfd,BACKLOG) == -1){  
   		printf("ERROR: Failed to listen Port %d.\n", PORT);
   		return (0);
   	}
   	else{
   		printf("OK: Listening the Port %d sucessfully.\n", PORT);
   	}
   ```

   

**处在监听模式下后，程序就需要一个循环来实现挂起等待客户机请求。所以接下来的一步就是 接受客户机的请求。**

### 4. accept()：

先来了解一下 accept() 这个函数：

```c
   // 	函数原型：
   #include <sys/socket.h>  
   #include<sys/types.h> 
   int accept(int sockfd,struct sockaddr *cliaddr,socklen_t *addrlen);
   
   /*
   	sockfd 	参数：监听的  套接字描述符。 
   	cliaddr 参数：指向结构体 sockaddr 的指针。  
   	addrlen 参数：cliaddr 参数指向的内存空间的长度。 
   */
   
   sin_size = sizeof(struct sockaddr_in);  
   connect_fd = accept(sockfd,( struct sockaddr *)&their_addr,&sin_size); 
   
```

- **accept()** 函数用于面向连接类型的套接字类型。
- **accept()** 函数将从连接请求队列中获得连接信息，创建新的套 接字，并返回该套接字的文件描述符。
- 新创建的套接字用于服务器与客户机的通信，而原来的套接字仍然处于监听状态。
- 它们的区别在于：监听套接口描述字 只有一个，而且一直存在，
- 每一个连接都有一个已连接套接口描述字，当连接断开 时就关闭该描述字。

  **注意：bind 函数和 accept 函数的第三个参数是不一样的。**

### 5. close()：

```c
  //	函数原型：
  #include <unistd.h>  
  int close(int sockfd); 
  // 	成功则返回 0，否则返回-1。 
  // 	功能：关闭套接口 其中参数 sockfd 是关闭的套接口描述字。 
  //	当对一个套接口调用 close（）时， 关闭该套接口描述字，并停止连接。
   
  以后这个套接口不能再使用，也不能再执行 任何读写操作，
  但关闭时已经排队准备发送的数据仍会被发出 使用完一个套接口后，
  一定要记得将它关掉，任何一个文件读写操作完毕之后，都要关闭它的描述字。 
   
```

## **客户机进程中的一些函数：**

### 1. socket()：

这个函数前面提过，这里不必多说。 

创建套接字后，同理，也需要对套接口进行设置： （这是在客户端填充的服务器 端的资料）......

   ```c
   bzero(&server_addr,sizeof(server_addr)); // 初始化,置 0 
   server_addr.sin_family=AF_INET;          // IPV4 
   server_addr.sin_port=htons(portnumber);  
   // (将本机器上的 short 数据转化为网络上的 short 数据)端口号，与服务器端 的端口号相同 
   server_addr.sin_addr=*((struct in_addr *)host->h_addr_list);  // IP 地址
   ```

### 2. connect()：

   ```c
connect(sockfd,(struct sockaddr *)(&server_addr), sizeof(structsockaddr)); 
函数原型：
  #include <sys/types.h>  
  #include <sys/socket.h>  
  int connect(int sockfd,const struct sockaddr *serv_addr,int addrlen); 
   
/*
   返回值：成功：返回 0 错误：返回-1，并将全局变量 errno 设置为相应的错误号。 
   参数 sockfd ：数据发送的套接字，解决从哪里发送的问题，ockfd 是先前 socket 返回的值 
   参数 serv_addr：据发送的目的地，也就是服务器端的地址 
   参数 addrlen：指定 server_addr 结构体的长度 
*/
   
  函数功能：
  创建了一个套接口之后，使客户端和服务器连接。其实就是完成一个 有连接协议 的连接过程，
  对于 TCP 来说就是那个三段握手过程。
   ```

关于三段握手： 《计算机网络》谢希仁编著 第七版中 将其定名为：" **三报文握手** "：

客户端先用 **connect()** 向服务器发出一个要求连接的信号 **`SYN1`**; 

服务器 进程接收到这个信号后，发回应答信号 **`ack1`**，同时这也是一个要求回答的信号 **`SYN2`**; 

客户端收到信号 **`ack1`** 和 **`SYN2`** 后，再次应答 **`ack2`**; 服务器收到应答信号 **`ack2`**,一次连接才算建立完成。  

从上面过程可以看出，服务器会收到两次信 号 **`SYN1`** 和 **`ack2`**，因此服务器进程需要两个队列保存不同状态的连接。

刚接收 到 **`SYN1`** 信号时，连接还未完成，这时的连接放在一个名为“**未完成连接**”的队列中。接收到 **`ack2`** 信号后，三段握手完成，这时的连接放在名为“已完成连接” 的队列中，等待 **accept()** 调用。

## **关于 recv() 、send() 和 recvfrom() 、sendto() ：**

1. **先说前两个：**

**recv()** 和 **send()** 都是基于 TCP 协议。

不论是客户还是服务器应用程序都用send函数来向TCP连接的另一端发送数据。

客户程序一般用send函数向服务器发送请求，而服务器则通常用send函数来向客户程序发送应答。

同样，不论是客户还是服务器应用程序都用recv函数从TCP连接的另一端接收数据。 

```c
 // 函数原型：
int send( SOCKET s, const char *buf, int len, int flags );
int recv( SOCKET s, char *buf, int len, int flags );
```

**（1）`recv`** 先等待 `s` 的发送缓冲中的数据被协议传送完毕，如果协议在传送 **`s`** 的发送缓冲中的数据时出现网络错误，那么recv函数返回 **`SOCKET_ERROR`** ；

**（2）** 如果 **`s`** 的发送缓冲中没有数据或者数据被协议成功发送完毕后，`recv` 先检查套接字 **`s`** 的接收缓冲区，如果 **`s`** 接收缓冲区中没有数据或者协议正在接收数据，那么 `recv` 就一直等待，直到协议把数据接收完毕。

当协议把数据接收完毕，**`recv`** 函数就把 **`s`** 的接收缓冲中的数据 **`copy`** 到 **`buf`** 中（注意协议接收到的数据可能大于 **`buf`** 的长度，所以在这种情况下要调用几次 **`recv`** 函数才能把s的接收缓冲中的数据 **`copy`** 完。**`recv`** 函数仅仅是 **`copy`** 数据，真正的接收数据是协议来完成的）；

其中，**`recv`** 函数返回其实际 **`copy`** 的字节数。如果 **`recv`** 在 **`copy`** 时出错，那么它返回 **`SOCKET_ERROR`**；如果recv函数在等待协议接收数据时网络中断了，那么它返回 0。

   

2. **然后是后两个：**

**`recvfrom()`** 和 **`sendto()`** 都是基于 UDP 协议。

不同于 TCP 协议，UDP 提供的是一种**无连接的、不可靠的**数据包协议。它不对数据进行确认、出错重传、排序等可靠性处理，但是它却具有**代码小、速度快和系统开销小**等优点。对于某些应用程序，使用 UDP 来实现，将带来更大效率。 

与基于 TCP 协议的客户机/服务器模式的工作流程图相比较，它们的主要区别 在于：  

使用 TCP 套接口必须先建立连接（例如客户进程的 **`connect()`** ，服务器进程 的 **`listen()`**和 **`accept()`** ） 。

而 UDP 套接口不需预先连接，它在调用 **`socket()`**生成一个套接口后，

在服务器端调用 **`bind()`** 绑定众所周知的端口后， 服务器阻塞于 **`recvfrom()`** 调用，

客户端调用 **`sendto()`** 发送数据请求，阻塞于 **`recvfrom()`** 调用，

服务器端调用 **`recvfrom()`** 接收数据，服务器端也调用 **`sendto()`** 向客户发送数据作为应答，然后阻塞于 **`recvfrom()`** 调用，

客户端 调用 **`recvfrom()`** 接收数据...... 

当数据传输完成以后，UDP 套接口中的客户端调用 **`close()`** 断开连接，而 TCP 套接口中的客户端不必再发出“断开连接信号”来通知服务器端关闭连接。

一些重要的应用程序，如域名服务系统 DNS、网络文件 系统 NFS 都使用 UDP 套接口。 

```c
 // 	函数原型：
 #include <sys/socket.h>   
 int recvfrom(int sockfd, void *buff, int len,int flags, struct sockaddr *fromaddr, int *addrlen);
/*
   参数 sockfd 为套接口描述字；  
   参数 buff 为指向读缓冲的指针；  
   参数 len 为读的字节数；  
   参数 flags 一般设置为 0；  
   参数 fromaddr 为指向数据接收的套接口地址结构的指针；  
   参数 addrlen 为套接口结构长度。
   	
   函数返回实际读的字节数，可以为 0，如果出错，则返回-1。
*/
   
   int sendto(int sockfd, void *mes,int len, int flags, struct sockaddr *toaddr, int *addrlen); 
   /*
   	参数 mes 为指向写缓冲的指针；  
   	参数 toaddr 为指向数据发送的套接口地址结构的指针； 
   	
   	函数返回实际写的字节数，可以为 0，如果出错，则返回-1。 
   */
```

   


## **拓展阅读 - 其他博文链接：**

[socket 网络编程 - CSDN 博主：奔跑中的小兔子](https://blog.csdn.net/m0_37947204/article/details/80489431)