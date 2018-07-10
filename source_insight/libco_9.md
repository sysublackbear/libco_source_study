#libco(9)
@(源码)

##6.example_echosvr.cpp

看了5之后，我们看下6：基于协程实现的echo服务器（svr端），代码如下：
```cpp
int main(int argc,char *argv[])
{
	if(argc<5){
		printf("Usage:\n"
               "example_echosvr [IP] [PORT] [TASK_COUNT] [PROCESS_COUNT]\n"
               "example_echosvr [IP] [PORT] [TASK_COUNT] [PROCESS_COUNT] -d   # daemonize mode\n");
		return -1;
	}
	// 监听的ip和port
	const char *ip = argv[1];
	int port = atoi( argv[2] );
	int cnt = atoi( argv[3] );  // 协程数
	int proccnt = atoi( argv[4] );  // 进程数
	bool deamonize = argc >= 6 && strcmp(argv[5], "-d") == 0;

	g_listen_fd = CreateTcpSocket( port,ip,true );
	// 监听
	// 1024是backlog参数，含义是：这个参数涉及到一些网络的细节。在进程正理一个一个连接请求的时候，可能还存在其它的连接请求。因为TCP连接是一个过程，所以可能存在一种半连接的状态，有时由于同时尝试连接的用户过多，使得服务器进程无法快速地完成连接请求。如果这个情况出现了，服务器进程希望内核如何处理呢？内核会在自己的进程空间里维护一个队列以跟踪这些完成的连接但服务器进程还没有接手处理或正在进行的连接，这样的一个队列内核不可能让其任意大，所以必须有一个大小的上限。这个backlog告诉内核使用这个数值作为上限。
	// 毫无疑问，服务器进程不能随便指定一个数值，内核有一个许可的范围。这个范围是实现相关的。很难有某种统一，一般这个值会小30以内。
	listen( g_listen_fd,1024 );
	if(g_listen_fd==-1){
		printf("Port %d is in use\n", port);
		return -1;
	}
	printf("listen %d %s:%d\n",g_listen_fd,ip,port);

	// 把fd设置成非阻塞
	SetNonBlock( g_listen_fd );

	for(int k=0;k<proccnt;k++)
	{
		// 根据proccnt个数来fork进程
		pid_t pid = fork();
		if( pid > 0 )
		{
			// 父进程
			continue;
		}
		else if( pid < 0 )
		{
			// 异常
			break;
		}
		for(int i=0;i<cnt;i++)
		{
			// 子进程逻辑
			// 创建协程，入口函数为readwrite_routine
			task_t * task = (task_t*)calloc( 1,sizeof(task_t) );
			task->fd = -1;

			co_create( &(task->co),NULL,readwrite_routine,task );
			co_resume( task->co );
	
		}
		// 创建accept协程
		stCoRoutine_t *accept_co = NULL;
		co_create( &accept_co,NULL,accept_routine,0 );
		co_resume( accept_co );

		co_eventloop( co_get_epoll_ct(),0,0 );

		exit(0);
	}
	if(!deamonize) wait(NULL);  // 非常驻模式，父进程陷入阻塞，发现子进程异常退出了，变成了僵尸进程（子进程的进程描述符还在系统中）的时候，父进程负责清理该僵尸进程
	// 孤儿进程：父进程先于子进程退出，父进程将子进程过继给init进程
	// 僵尸进程：子进程先于父进程退出，且父进程没有调用wait
	return 0;
}

struct task_t
{
	stCoRoutine_t *co;
	int fd;
};
```

###6.1.CreateTcpSocket函数
位于`example_echosvr.cpp`，代码如下：
```cpp
// g_listen_fd = CreateTcpSocket( port,ip,true );
static int CreateTcpSocket(
	const unsigned short shPort /* = 0 */,
	const char *pszIP /* = "*" */,
	bool bReuse /* = false */)
{
	// 非阻塞建立socket连接
	// socket函数的实现详见5.4.1
	// AF_INET：ipv4
	// SOCK_STREAM：流式
	// IPPROTO_TCP：tcp协议
	int fd = socket(AF_INET,SOCK_STREAM, IPPROTO_TCP);
	if( fd >= 0 )
	{
		if(shPort != 0)
		{
			// 是否复用该长连接
			if(bReuse)
			{
				int nReuseAddr = 1;
// hook为socket设置读写超时(如果有需要的话)			setsockopt(fd,SOL_SOCKET,SO_REUSEADDR,&nReuseAddr,sizeof(nReuseAddr));
			}
			struct sockaddr_in addr ;
			SetAddr(pszIP,shPort,addr);
			// 监听该ip和端口
			int ret = bind(fd,(struct sockaddr*)&addr,sizeof(addr));
			if( ret != 0)
			{
				close(fd);
				return -1;
			}
		}
	}
	return fd;
}
```

####6.1.1.setsockopt函数（hook系统调用）
位于`co_hook_syc_call.cpp`，代码如下：
```cpp
// int nReuseAddr = 1;
// setsockopt(fd,SOL_SOCKET,SO_REUSEADDR,&nReuseAddr,sizeof(nReuseAddr));
// 设置socket属性
int setsockopt(
	int fd, 
	int level, 
	int option_name,
	const void *option_value, 
	socklen_t option_len)
{
	HOOK_SYS_FUNC( setsockopt );

	if( !co_is_enable_sys_hook() )
	{
		return g_sys_setsockopt_func( fd,level,option_name,option_value,option_len );
	}
	rpchook_t *lp = get_by_fd( fd );

	// 是否对socket设置属性
	if( lp && SOL_SOCKET == level )
	{
		// int nReuseAddr = 1;
		// 读写接口超时均为1s
		struct timeval *val = (struct timeval*)option_value;
		// SO_RCVTIMEO:如果设置了它，所有对该套接字的读操作在规定的时间里没完成，就直接返回并设置 errno = EWOULDBLOCK
		// SO_SNDTIMEO:如果设置了它，所有对该套接字的写操作在规定的时间里没完成，就直接返回并设置 errno = EWOULDBLOCK
		if( SO_RCVTIMEO == option_name  ) 
		{
			memcpy( &lp->read_timeout,val,sizeof(*val) );
		}
		else if( SO_SNDTIMEO == option_name )
		{
			memcpy( &lp->write_timeout,val,sizeof(*val) );
		}
	}
	return g_sys_setsockopt_func( fd,level,option_name,option_value,option_len );
}
```

**！！！注意：**`SO_REUSEADDR`的含义：
+ 一般来说，一个端口释放后会等待两分钟之后才能再被使用，`SO_REUSEADDR`是让端口释放后立即就可以被再次使用。
+ `SO_REUSEADDR`用于对TCP套接字处于`TIME_WAIT`状态下的socket，才可以重复绑定使用。server程序总是应该在调用`bind()`之前设置`SO_REUSEADDR`套接字选项。TCP，先调用`close()`的一方会进入`TIME_WAIT`状态。


####6.1.2.bind函数
`int ret = bind(fd,(struct sockaddr*)&addr,sizeof(addr));`
没啥好说的，监听ip和port

###6.2.SetNonBlock函数
位于`example_echosvr.cpp`，代码如下：
```cpp
static int SetNonBlock(int iSock)
{
    int iFlags;

    iFlags = fcntl(iSock, F_GETFL, 0);
    iFlags |= O_NONBLOCK;
    iFlags |= O_NDELAY;
    int ret = fcntl(iSock, F_SETFL, iFlags);
    return ret;
}
```
比较简单，就是给fd设置上`O_NONBLOCK`和`O_NDELAY`。

**注：O_NONBLOCK与O_NDELAY有何不同：**
`O_NONBLOCK`和`O_NDELAY`所产生的结果都是使I/O变成非阻塞模式(non-blocking)，在读取不到数据或是写入缓冲区已满会马上return，而不会阻塞等待。

它们的差别在于：在读操作时，如果读不到数据，`O_NDELAY`会使I/O函数马上返回0，但这又衍生出一个问题，因为读取到文件末尾(EOF)时返回的也是0，这样无法区分是哪种情况。因此，`O_NONBLOCK`就产生出来，它在读取不到数据时会回传-1，并且设置`errno`为`EAGAIN`。

`O_NDELAY`是在System V的早期版本中引入的，在编码时，还是推荐POSIX规定的`O_NONBLOCK`，`O_NONBLOCK`可以在open和fcntl时设置。


###6.3.readwrite_routine函数
位于`example_echosvr.cpp`，代码如下：
```cpp
static void *readwrite_routine( void *arg )
{

	co_enable_hook_sys();

	task_t *co = (task_t*)arg;
	char buf[ 1024 * 16 ];
	for(;;)
	{
		if( -1 == co->fd )
		{
			// fd未就绪
			// static stack<task_t*> g_readwrite;
			// 静态变量
			g_readwrite.push( co );
			// 挂起当前协程
			co_yield_ct();
			continue;
		}

		// fd就绪
		int fd = co->fd;
		co->fd = -1;

		for(;;)
		{
			struct pollfd pf = { 0 };
			pf.fd = fd;
			pf.events = (POLLIN|POLLERR|POLLHUP);
			co_poll( co_get_epoll_ct(),&pf,1,1000);

			int ret = read( fd,buf,sizeof(buf) );
			if( ret > 0 )
			{
				// echo，读多少就写多少
				// 读写皆非阻塞
				ret = write( fd,buf,ret );
			}
			if( ret <= 0 )
			{
				// 在Linux环境下开发经常会碰到很多错误(设置errno)，其中EAGAIN是其中比较常见的一个错误(比如用在非阻塞操作中)。
				// 从字面上来看，是提示再试一次。这个错误经常出现在当应用程序进行一些非阻塞(non-blocking)操作(对文件或socket)的时候。例如，以 O_NONBLOCK的标志打开文件/socket/FIFO，如果你连续做read操作而没有数据可读，此时程序不会阻塞起来等待数据准备就绪返回，read函数会返回一个错误EAGAIN，提示你的应用程序现在没有数据可读请稍后再试。
				// accept_routine->SetNonBlock(fd) cause EAGAIN, we should continue
				if (errno == EAGAIN)
				{
					continue;
				}
				close( fd );
				break;
			}
		}

	}
	return 0;
}
```


###6.4.accept_routine函数
位于`example_echosvr.cpp`，代码如下：
```cpp
int co_accept(int fd, struct sockaddr *addr, socklen_t *len );
static void *accept_routine( void * )
{
	co_enable_hook_sys();
	printf("accept_routine\n");
	fflush(stdout);  // 情况标准输出
	for(;;)
	{
		//printf("pid %ld g_readwrite.size %ld\n",getpid(),g_readwrite.size());
		if( g_readwrite.empty() )
		{
			// 不存在未就绪的fd
			printf("empty\n"); //sleep
			struct pollfd pf = { 0 };
			pf.fd = -1;
			poll( &pf,1,1000);  // sleep 1s等待（非阻塞）

			continue;

		}
		struct sockaddr_in addr; //maybe sockaddr_un;
		memset( &addr,0,sizeof(addr) );
		socklen_t len = sizeof(addr);

		int fd = co_accept(g_listen_fd, (struct sockaddr *)&addr, &len);
		if( fd < 0 )
		{
			// accept失败
			// 调用epoll切出
			struct pollfd pf = { 0 };
			pf.fd = g_listen_fd;
			pf.events = (POLLIN|POLLERR|POLLHUP);
			co_poll( co_get_epoll_ct(),&pf,1,1000 );
			continue;
		}
		// 再次检查协程栈是否已经被其他的协程处理掉了。
		// 有则直接返回
		if( g_readwrite.empty() )
		{
			close( fd );
			continue;
		}
		// 将fd设置成非阻塞
		SetNonBlock( fd );
		// 找回对应的协程，执行echo操作
		task_t *co = g_readwrite.top();
		co->fd = fd;
		g_readwrite.pop();
		// 挂起当前协程
		co_resume( co->co );
	}
	return 0;
}
```

####6.4.1.co_accept函数
位于`co_hook_sys_call.cpp`，代码如下：
```cpp
// int fd = co_accept(g_listen_fd, (struct sockaddr *)&addr, &len);
// fd已经设置好ip和port
int co_accept( int fd, struct sockaddr *addr, socklen_t *len )
{
	// 接收客户端请求
	// 返回svr端和cli端一一关联起来的socket对象
	int cli = accept( fd,addr,len );
	if( cli < 0 )
	{
		return cli;
	}
	// 为每个fd生成一个rpchook_t
	alloc_by_fd( cli );
	return cli;
}
```


###6.5.在tcp和udp中，cli端和svr端的处理差异
**tcp连接模式：**
cli端
```powershell
1.创建socket（套接字）
    socket(int af, int type, int protocol)
    第一个参数用来说明网络协议类型，tcp/ip协议只能用AF_INET
    第二个参数：socket通讯类型（tcp，udp等）
    第三个参数：与选择的通讯类型有关
2.向服务器发起连接
    SOCKADDR_IN 声明套接字类型
        sin_family 协议家族
        sin_port 通信端口
        sin_addr 网络ip

    htons 将整型类型转换成网络字节序
    htonl 将长整型转换成网络字节序

    inet_pton（int af, char * str, pvoid addrbuf） 将点分十进制ip地址转换成网络字节
        第一个参数：协议家族
        第二个参数：点分十进制ip地址，例如："127.0.0.1"
        第三个参数：sin_addr

    inte_ntop (int af, pvoid addrbuf, char *str, size_t len) 将网络字节序转换成点分十进制ip地址
        第一个参数：协议家族
        第二个参数：sin_addr
        第三个参数：存储ip地址的字符串
        第四个参数：字节单位长度

    connect(socket s, const sockaddr *name, int namelen)
        第一个参数：创建的套接字
        第二个参数：要链接的套接字地址
        第三个参数：单位长度

3.client与server通信
    send(socket s, char * str, int len, int flag)
        第一个参数：本机创建的套接字
        第二个参数：要发送的字符串
        第三个参数：发送字符串长度
        第四个参数：会对函数行为产生影响，一般设置为0

    recv(socket s, char * buf, int len,int flag)
        第一个参数：本机创建的套接字
        第二个参数：接受消息的字符串
        第三个参数：允许接收字符串的最大长度
        第四个参数：会对函数行为产生影响，一般设置为0

4.释放套接字
    closesocket 释放套接字
```

svr端：
```powershell
1.创建套接字
2.将套接字与本地的ip和端口绑定
    bind（socket s, const sockaddr *name, int namelen）
        第一个参数：创建的套接字
        第二个参数：要绑定的信息
        第三个参数：套接字类型单位长度
3.设置为监听状态
    listen（socket s, int num）
        第一个参数：要监听的socket（套接字）
        第二个参数：等待连接队列的最大长度
4.等待客户请求到来；当请求到来后，接受连接请求，返回一个新的对应于此次连接的套接字
    accept(int socket, sockaddr *name, int *addrlen)
        第一个参数，是一个已设为监听模式的socket的描述符。
        第二个参数，是一个返回值，它指向一个struct sockaddr类型的结构体的变量，保存了发起连接的客户端得IP地址信息和端口信息。
        第三个参数，也是一个返回值，指向整型的变量，保存了返回的地址信息的长度。
        accept函数返回值是一个客户端和服务器连接的SOCKET类型的描述符，在服务器端标识着这个客户端。
5.相互通信
6.断开连接
```

**udp连接模式**
cli端
```powershell
1.创建套接字
2.绑定服务器ip和port已经协议家族
3.相互通信
    sendto(socket s, const char * buf, int len, int flags, const sockaddr *to,int tolen)
        第一个参数：创建的套接字
        第二个参数：要发送的字符串
        第三个参数：要发送的字符串长度
        第四个参数：影响函数的行为，一般为0
        第五个参数：远端套接字信息
        第六个参数：套接字单位长度

    recvfrom（socket s, char * buf, int len, int flags, const sockaddr *to,int tolen）
        第一个参数：创建的套接字
        第二个参数：存储接收的字符串
        第三个参数：可接受的最大字符串长度
        第四个参数：影响函数的行为，一般为0
        第五个参数：远端套接字信息
        第六个参数：套接字单位长度
4.释放套接字
```

svn端：
```powershell
1.创建套接字
2.绑定本地的端口，ip，协议家族信息
3.通信
4.释放连接
```


###6.6.综述
echosvr端的处理过程比较简单，核心还是tcp协议服务器的处理规则：
socket->bind->listen->accept->read/write
主要有两类协程：
+ accept_routine，负责处理出socket对应client的端的某一特地连接；
+ readwrite_routine，fd就绪后，直接执行echo操作，先read再write。
