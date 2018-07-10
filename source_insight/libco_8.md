#libco(8)
@(源码)

##5.example_echocli.cpp

很明显可以看到，基于协程实现的echo服务器。代码如下：
```cpp
int main(int argc,char *argv[])
{
	// 定义stEndPoint对象
	stEndPoint endpoint;
	endpoint.ip = argv[1];
	endpoint.port = atoi(argv[2]);
	int cnt = atoi( argv[3] );  // 协程个数
	int proccnt = atoi( argv[4] );  // 连接个数
	// proccnt个进程,cnt个协程
	
	struct sigaction sa;
	sa.sa_handler = SIG_IGN;  // 忽略SIGPIPE信号
	sigaction( SIGPIPE, &sa, NULL );
	
	for(int k=0;k<proccnt;k++)
	{
		// fork进程
		pid_t pid = fork();
		if( pid > 0 )
		{
			// 父进程分支，继续fork
			continue;
		}
		else if( pid < 0 )
		{
			// 异常，直接退出
			break;
		}
		// 子进程分支
		for(int i=0;i<cnt;i++)
		{
			// 创建协程
			stCoRoutine_t *co = 0;
			// func = readwrite_routine
			// 创建协程
			co_create( &co,NULL,readwrite_routine, &endpoint);
			// 切换协程
			co_resume( co );
		}
		co_eventloop( co_get_epoll_ct(),0,0 );

		exit(0);
	}
	return 0;
}
/*./example_echosvr 127.0.0.1 10000 100 50*/
```

###5.1.stEndPoint结构
很简单，EndPoint=ip+port
```cpp
struct stEndPoint
{
	char *ip;
	unsigned short int port;
};
```


###5.2.sigaction介绍
sigaction函数可以读取和修改与指定信号相关联的处理动作。调用成功则返回0，出错则返回-1。signo是指定信号的编号。若act指针非空，则根据act修改该信号的处理动作。若oact指针非空，则通过oact传出该信号原来的处理动作。act和oact指向sigaction结构体。函数如下：
```cpp
struct sigaction
{
	void (*sa_handler)(int);
	void (*sa_sigaction)(int, siginfo_t *, void *);
	sigset_t sa_mask;
	int sa_flags;
	void (*sa_restorer)(void);
};
```
将sa_handler赋值为常数SIG_IGN传给sigaction表示忽略信号，赋值为常数SIG_DFL表示执行系统默认动作，赋值为一个函数指针表示用自定义函数捕捉信号，或者说向内核注册了一个信号处理函数，该函数返回值为void，可以带一个int参数，通过参数可以得知当前信号的编号，这样就可以用同一个函数处理多种信号。显然，这也是一个回调函数，不是被main函数调用，而是被系统所调用。
当某个信号的处理函数被调用时，内核自动将当前信号加入进程的信号屏蔽字，当信号处理函数返回时自动恢复原来的信号屏蔽字，这样就保证了在处理某个信号时，如果这种信号再次产生，那么它会被阻塞到当前处理结束为止。如果在调用信号处理函数时，除了当前信号被自动屏蔽之外，还希望自动屏蔽另外一些信号，则用sa_mask字段说明这些需要额外屏蔽的信号，当信号处理函数返回时自动恢复原来的信号屏蔽字。

上面的目的是为了忽略SIGPIPE信号。

###5.3.关于SIGPIPE信号
这是从网上找回来的解答，可以看下，了解多一点。
这里为什么Cli端要忽略掉SIGPIPE。是因为：SIGPIPE的默认信号处理程序会终止整个进程。这里目的是不希望cli进程被SIGPIPE杀死。所以这里选择忽略对该信号的处置。
那么什么情况会产生SIGPIPE信号呢？如果一个 socket 在接收到了 RST packet （连接重置）之后，程序仍然向这个 socket 写入数据，那么就会产生SIGPIPE信号。
这种现象是很常见的，譬如说，当 client 连接到 server 之后，这时候 server 准备向 client 发送多条消息，但在发送消息之前，client 进程意外奔溃了，那么接下来 server 在发送多条消息的过程中，就会出现SIGPIPE信号。

那么，为什么程序还会在收到RST包之后，还会继续往socket写数据呢？这里涉及：TCP是全双工的信道, 可以看作两条单工信道, TCP连接两端的两个端点各负责一条. 当对端调用close时, 虽然本意是关闭整个两条信道, 但本端只是收到FIN包. 按照TCP协议的语义, 表示对端只是关闭了其所负责的那一条单工信道, 仍然可以继续接收数据. （就是说，即使对端close了连接，只是说明对端不再给本端发送数据了，但并不妨碍本端给对端发送数据。）也就是说, 因为TCP协议的限制, 一个端点无法获知对端的socket是调用了close还是shutdown.（不知道是对端关闭了连接还是对端异常退出了。）

所以，常见的过程：
+ client 连接到 server 之后，client 进程意外奔溃，这时它会发送一个 FIN 给 server。
+ 此时 server 并不知道 client 已经奔溃了，所以它会发送第一条消息给 client。但 client 已经退出了，所以 client 的 TCP 协议栈会发送一个 RST 给 server。
+ server 在接收到 RST 之后，继续写入第二条消息。往一个已经收到 RST 的 socket 继续写入数据，将导致SIGPIPE信号，从而杀死 server。

因此，对 server 来说，为了不被SIGPIPE信号杀死，那就需要忽略SIGPIPE信号。

###5.4.readwrite_routine函数
代码如下：
```cpp
// co_create( &co,NULL,readwrite_routine, &endpoint);
static void *readwrite_routine( void *arg )
{

	co_enable_hook_sys();  // 打开hook系统调用开关

	stEndPoint *endpoint = (stEndPoint *)arg;
	char str[8]="sarlmol";
	char buf[ 1024 * 16 ];
	int fd = -1;
	int ret = 0;
	for(;;)
	{
		if ( fd < 0 )  // fd不可用
		{
			// 初始化socket
			// 这里socket函数hook了系统调用了。
			// domain:PF_INET,IPV4
			// type:SOCK_STREAM,Tcp连接，提供序列化的、可靠的、双向连接的字节流。支持带外数据传输
			// protocol:0，代表只有一种特定的协议
			fd = socket(PF_INET, SOCK_STREAM, 0);
			struct sockaddr_in addr;
			SetAddr(endpoint->ip, endpoint->port, addr);
			ret = connect(fd,(struct sockaddr*)&addr,sizeof(addr));
					
			// EALREADY：socket 是非阻塞的而且有一个挂起的连接（参考上面的 EINPROGRESS）。
			// EINPROGRESS：连接已经建立
			// ！！！这段逻辑是由于上面的connect调用时非阻塞的方式
			// 这里检查连接是否正常建立。
			if ( errno == EALREADY || errno == EINPROGRESS )
			{       
				struct pollfd pf = { 0 };
				pf.fd = fd;
				pf.events = (POLLOUT|POLLERR|POLLHUP);
				co_poll( co_get_epoll_ct(),&pf,1,200);
				//check connect
				int error = 0;
				uint32_t socklen = sizeof(error);
				errno = 0;
				ret = getsockopt(fd, SOL_SOCKET, SO_ERROR,(void *)&error,  &socklen);
				if ( ret == -1 ) 
				{       
					//printf("getsockopt ERROR ret %d %d:%s\n", ret, errno, strerror(errno));
					close(fd);
					fd = -1;
					AddFailCnt();  // 统计连接失败个数
					continue;
				}       
				if ( error ) 
				{       
					errno = error;
					//printf("connect ERROR ret %d %d:%s\n", error, errno, strerror(errno));
					close(fd);
					fd = -1;
					AddFailCnt();
					continue;
				}       
			} 
	  			
		}
		
		// 对fd进行写操作（hook，非阻塞）
		ret = write( fd,str, 8);
		if ( ret > 0 )
		{
			// 写成功之后，进行读取buffer操作
			ret = read( fd,buf, sizeof(buf) );
			if ( ret <= 0 )
			{
				//printf("co %p read ret %d errno %d (%s)\n",
				//		co_self(), ret,errno,strerror(errno));
				close(fd);
				fd = -1;
				AddFailCnt();
			}
			else
			{
				//printf("echo %s fd %d\n", buf,fd);
				AddSuccCnt();
			}
		}
		else
		{
			// 写入失败，关闭fd
			//printf("co %p write ret %d errno %d (%s)\n",
			//		co_self(), ret,errno,strerror(errno));
			close(fd);
			fd = -1;
			AddFailCnt();
		}
	}
	return 0;
}

int	co_poll( stCoEpoll_t *ctx,struct pollfd fds[], nfds_t nfds, int timeout_ms )
{
	return co_poll_inner(ctx, fds, nfds, timeout_ms, NULL);
}

typedef int (*poll_pfn_t)(struct pollfd fds[], nfds_t nfds, int timeout);
int co_poll_inner( stCoEpoll_t *ctx,struct pollfd fds[], nfds_t nfds, int timeout, poll_pfn_t pollfunc)
{
    if (timeout == 0)
	{
		return pollfunc(fds, nfds, timeout);
	}
	if (timeout < 0)
	{
		timeout = INT_MAX;
	}
	int epfd = ctx->iEpollFd;
	stCoRoutine_t* self = co_self();

	//1.struct change
	stPoll_t& arg = *((stPoll_t*)malloc(sizeof(stPoll_t)));
	memset( &arg,0,sizeof(arg) );

	arg.iEpollFd = epfd;
	arg.fds = (pollfd*)calloc(nfds, sizeof(pollfd));
	arg.nfds = nfds;

	stPollItem_t arr[2];
	if( nfds < sizeof(arr) / sizeof(arr[0]) && !self->cIsShareStack)
	{
		arg.pPollItems = arr;
	}	
	else
	{
		arg.pPollItems = (stPollItem_t*)malloc( nfds * sizeof( stPollItem_t ) );
	}
	memset( arg.pPollItems,0,nfds * sizeof(stPollItem_t) );

	arg.pfnProcess = OnPollProcessEvent;
	arg.pArg = GetCurrCo( co_get_curr_thread_env() );
	
	
	//2. add epoll
	for(nfds_t i=0;i<nfds;i++)
	{
		arg.pPollItems[i].pSelf = arg.fds + i;
		arg.pPollItems[i].pPoll = &arg;

		arg.pPollItems[i].pfnPrepare = OnPollPreparePfn;
		struct epoll_event &ev = arg.pPollItems[i].stEvent;

		if( fds[i].fd > -1 )
		{
			ev.data.ptr = arg.pPollItems + i;
			ev.events = PollEvent2Epoll( fds[i].events );

			int ret = co_epoll_ctl( epfd,EPOLL_CTL_ADD, fds[i].fd, &ev );
			if (ret < 0 && errno == EPERM && nfds == 1 && pollfunc != NULL)
			{
				if( arg.pPollItems != arr )
				{
					free( arg.pPollItems );
					arg.pPollItems = NULL;
				}
				free(arg.fds);
				free(&arg);
				return pollfunc(fds, nfds, timeout);
			}
		}
		//if fail,the timeout would work
	}

	//3.add timeout

	unsigned long long now = GetTickMS();
	arg.ullExpireTime = now + timeout;
	int ret = AddTimeout( ctx->pTimeout,&arg,now );
	int iRaiseCnt = 0;
	if( ret != 0 )
	{
		co_log_err("CO_ERR: AddTimeout ret %d now %lld timeout %d arg.ullExpireTime %lld",
				ret,now,timeout,arg.ullExpireTime);
		errno = EINVAL;
		iRaiseCnt = -1;

	}
    else
	{
		co_yield_env( co_get_curr_thread_env() );
		iRaiseCnt = arg.iRaiseCnt;
	}

    {
		//clear epoll status and memory
		RemoveFromLink<stTimeoutItem_t,stTimeoutItemLink_t>( &arg );
		for(nfds_t i = 0;i < nfds;i++)
		{
			int fd = fds[i].fd;
			if( fd > -1 )
			{
				co_epoll_ctl( epfd,EPOLL_CTL_DEL,fd,&arg.pPollItems[i].stEvent );
			}
			fds[i].revents = arg.fds[i].revents;
		}


		if( arg.pPollItems != arr )
		{
			free( arg.pPollItems );
			arg.pPollItems = NULL;
		}

		free(arg.fds);
		free(&arg);
	}

	return iRaiseCnt;
}
```


####5.4.1.socket函数（hook系统调用）
位于`co_hook_sys_call.cpp`，代码如下：
```cpp
int socket(int domain, int type, int protocol)
{
	HOOK_SYS_FUNC( socket );  // 注册系统调用函数
	
	// 没有打开hook开关
	if( !co_is_enable_sys_hook() )
	{
		// 走原来的系统调用
		return g_sys_socket_func( domain,type,protocol );
	}
	// 先进行系统调用，获取fd
	// 对fd设置O_NONBLOCK
	int fd = g_sys_socket_func(domain,type,protocol);
	if( fd < 0 )
	{
		return fd;
	}
	// 为fd分配一个rpchook_t对象
	rpchook_t *lp = alloc_by_fd( fd );
	lp->domain = domain;
	// 给fd赋值
	// 这个函数也hook了系统调用
	// fd:fd
	// cmd:F_SETFL,设置文件状态标志
	fcntl( fd, F_SETFL, g_sys_fcntl_func(fd, F_GETFL,0 ) );
	// 上面这一句只是为了让fd添加O_NONBLOCK属性
	// g_sys_fcntl_func函数
	// fd:fd
	// F_GETFL:读取文件状态标志。
	// g_sys_fcntl_func:获取fd的属性状态

	return fd;
}
```

#####5.4.1.1.alloc_by_fd函数
位于`co_hook_sys_call.cpp`，代码如下：
```cpp
struct rpchook_t
{
	int user_flag;
	struct sockaddr_in dest; //maybe sockaddr_un;
	int domain; //AF_LOCAL , AF_INET

	struct timeval read_timeout;
	struct timeval write_timeout;
};

static inline rpchook_t * alloc_by_fd( int fd )
{
	if( fd > -1 && fd < (int)sizeof(g_rpchook_socket_fd) / (int)sizeof(g_rpchook_socket_fd[0]) )
	{
		// 为fd分配一个rpchook_t
		rpchook_t *lp = (rpchook_t*)calloc( 1,sizeof(rpchook_t) );
		// 读写超时均设置为1s
		lp->read_timeout.tv_sec = 1;
		lp->write_timeout.tv_sec = 1;
		g_rpchook_socket_fd[ fd ] = lp;
		return lp;
	}
	return NULL;
}

static rpchook_t *g_rpchook_socket_fd[ 102400 ] = { 0 };
```


#####5.4.1.2.fcntl(hook系统调用)
位于`co_hook_sys_call.cpp`，代码如下：
```cpp
int fcntl(int fildes, int cmd, ...)
{
	HOOK_SYS_FUNC( fcntl );
	// fd不可用
	if( fildes < 0 )
	{
		return __LINE__;
	}

	va_list arg_list;
	// arg_list:剩下的参数塞到arg_list
	// cmd:最后一个确定的参数是cmd
	va_start( arg_list,cmd );

	int ret = -1;
	rpchook_t *lp = get_by_fd( fildes );
	switch( cmd )
	{
		// F_DUPFD:复制fd
		case F_DUPFD:
		{
			int param = va_arg(arg_list,int);
			ret = g_sys_fcntl_func( fildes,cmd,param );
			break;
		}
		// F_GETFD ：读取文件描述词标志。 
		case F_GETFD:
		{
			ret = g_sys_fcntl_func( fildes,cmd );
			break;
		}
		// F_SETFD ：设置文件描述词标志。
		case F_SETFD:
		{
			int param = va_arg(arg_list,int);
			ret = g_sys_fcntl_func( fildes,cmd,param );
			break;
		}
		// F_GETFL ：读取文件状态标志。 
		case F_GETFL:
		{
			ret = g_sys_fcntl_func( fildes,cmd );
			break;
		}
		// F_SETFL ：设置文件状态标志。
		case F_SETFL:
		{
			int param = va_arg(arg_list,int);
			int flag = param;
			if( co_is_enable_sys_hook() && lp )
			{
				// 给fd添加上非阻塞标记
				flag |= O_NONBLOCK;
			}
			ret = g_sys_fcntl_func( fildes,cmd,flag );
			if( 0 == ret && lp )
			{
				// 把原始的属性记下来。
				lp->user_flag = param;
			}
			break;
		}
		// F_GETOWN：获取当前在文件描述词 fd上接收到SIGIO 或 SIGURG事件信号的进程或进程组标识 
		case F_GETOWN:
		{
			ret = g_sys_fcntl_func( fildes,cmd );
			break;
		}
		// F_SETOWN：设置将要在文件描述词fd上接收SIGIO 或 SIGURG事件信号的进程或进程组标识 。
		case F_SETOWN:
		{
			int param = va_arg(arg_list,int);
			ret = g_sys_fcntl_func( fildes,cmd,param );
			break;
		}
		// F_GETLK：获取文件锁信息。
		case F_GETLK:
		{
			struct flock *param = va_arg(arg_list,struct flock *);
			ret = g_sys_fcntl_func( fildes,cmd,param );
			break;
		}
		// F_SETLK：在指定的字节范围获取锁（F_RDLCK, F_WRLCK）或释放锁（F_UNLCK）。如果和另一个进程的锁操作发生冲突，返回 -1并将errno设置为EACCES或EAGAIN。 
		case F_SETLK:
		{
			struct flock *param = va_arg(arg_list,struct flock *);
			ret = g_sys_fcntl_func( fildes,cmd,param );
			break;
		}
		// F_SETLKW：行为如同F_SETLK，除了不能获取锁时会睡眠等待外。如果在等待的过程中接收到信号，会即时返回并将errno置为EINTR。 
		case F_SETLKW:
		{
			struct flock *param = va_arg(arg_list,struct flock *);
			ret = g_sys_fcntl_func( fildes,cmd,param );
			break;
		}
	}

	va_end( arg_list );

	return ret;
}
```


#####5.4.1.3.get_by_fd函数
位于`co_hook_sys_call.cpp`，代码如下：
```cpp
struct rpchook_t
{
	int user_flag;
	struct sockaddr_in dest; //maybe sockaddr_un;
	int domain; //AF_LOCAL , AF_INET

	struct timeval read_timeout;
	struct timeval write_timeout;
};
// 获取fd对应的rpchook_t
static inline rpchook_t * get_by_fd( int fd )
{
	if( fd > -1 && fd < (int)sizeof(g_rpchook_socket_fd) / (int)sizeof(g_rpchook_socket_fd[0]) )
	{
		return g_rpchook_socket_fd[ fd ];
	}
	return NULL;
}
```


####5.4.2.SetAddr函数
位于`example_echocli.cpp`，代码如下：
```cpp
static void SetAddr(const char *pszIP,const unsigned short shPort,struct sockaddr_in &addr)
{
	// bzero:将内存的前n个字节内容清零
	bzero(&addr,sizeof(addr));
	addr.sin_family = AF_INET;  // 地址族协议：AF_INET代表TCP/IP协议
	// Port number(必须要采用网络数据格式,普通数字可以用htons()函数转换成网络数据格式的数字)
	addr.sin_port = htons(shPort);
	int nIP = 0;
	if( !pszIP || '\0' == *pszIP   
			|| 0 == strcmp(pszIP,"0") || 0 == strcmp(pszIP,"0.0.0.0") 
			|| 0 == strcmp(pszIP,"*") 
	  )
	{
		nIP = htonl(INADDR_ANY);  // 设置0.0.0.0地址格式
	}
	else
	{
		nIP = inet_addr(pszIP);  // 将一个点分十进制的IP转换成一个长整数型数
	}
	addr.sin_addr.s_addr = nIP;

}
```

####5.4.3.connect函数（hook系统调用）
位于`co_hook_sys_call.cpp`，代码如下：
```cpp
int connect(int fd, const struct sockaddr *address, socklen_t address_len)
{
	HOOK_SYS_FUNC( connect );

	if( !co_is_enable_sys_hook() )
	{
		return g_sys_connect_func(fd,address,address_len);
	}

	//1.sys call
	// 先直接调用系统调用
	int ret = g_sys_connect_func( fd,address,address_len );
	// 获取fd对应的rpchook_t
	rpchook_t *lp = get_by_fd( fd );
	if( !lp ) return ret;

	if( sizeof(lp->dest) >= address_len )
	{
		 memcpy( &(lp->dest),address,(int)address_len );
	}
	// 用户如果显式设置了O_NONBLOCK，直接返回给上层
	if( O_NONBLOCK & lp->user_flag ) 
	{
		return ret;
	}
	
	if (!(ret < 0 && errno == EINPROGRESS))
	{
		// 连接正在建立，返回给上层
		return ret;
	}

	//2.wait
	// 阻塞逻辑,会阻塞在这里
	// 这里只有在fd没有设置非阻塞才会走到下面的逻辑
	// 检查fd（连接）的状态，我们希望通过非阻塞的方式，使用epoll的方式。
	int pollret = 0;
	struct pollfd pf = { 0 };

	for(int i=0;i<3;i++) //25s * 3 = 75s
	{
		memset( &pf,0,sizeof(pf) );
		pf.fd = fd;
		pf.events = ( POLLOUT | POLLERR | POLLHUP );

		// 调用hook的poll函数
		pollret = poll( &pf,1,25000 );

		if( 1 == pollret  )
		{
			break;
		}
	}
	// POLLOUT:写数据就绪，不会阻塞
	if( pf.revents & POLLOUT ) //connect succ
	{
		errno = 0;
		return 0;
	}

	//3.set errno
	// 设置errno
	int err = 0;
	socklen_t errlen = sizeof(err);
	// 获取套接字的属性
	getsockopt( fd,SOL_SOCKET,SO_ERROR,&err,&errlen);
	if( err ) 
	{
		errno = err;
	}
	else
	{
		errno = ETIMEDOUT;
	} 
	return ret;
}
```

####5.4.4.AddFailCnt函数
代码如下：
```cpp
void AddFailCnt()
{
	int now = time(NULL);
	if (now >iTime)
	{
		printf("time %d Succ Cnt %d Fail Cnt %d\n", iTime, iSuccCnt, iFailCnt);
		iTime = now;  // 全局变量
		iSuccCnt = 0;  // 全局变量
		iFailCnt = 0;  // 全局变量
	}
	else
	{
		iFailCnt++;
	}
}
```

####5.4.5.write函数（hook系统调用）
位于`co_hook_sys_call.cpp`，代码如下：
```cpp
ssize_t write( int fd, const void *buf, size_t nbyte )
{
	// 打开hook系统调用开关
	HOOK_SYS_FUNC( write );
	
	if( !co_is_enable_sys_hook() )
	{
		// 直接调用系统调用
		return g_sys_write_func( fd,buf,nbyte );
	}
	rpchook_t *lp = get_by_fd( fd );

	if( !lp || ( O_NONBLOCK & lp->user_flag ) )
	{
		// fd已经设置了O_NONBLOCK，直接系统调用返回
		ssize_t ret = g_sys_write_func( fd,buf,nbyte );
		return ret;
	}
	size_t wrotelen = 0;
	int timeout = ( lp->write_timeout.tv_sec * 1000 ) 
				+ ( lp->write_timeout.tv_usec / 1000 );

	// 非阻塞write:buffer能写入多少字节就返回多少字节
	ssize_t writeret = g_sys_write_func( fd,(const char*)buf + wrotelen,nbyte - wrotelen );

	if (writeret == 0)
	{
		return writeret;
	}

	if( writeret > 0 )
	{
		wrotelen += writeret;	
	}
	while( wrotelen < nbyte )
	{
		// buffer写满,注册epoll，等待内核可写时回调
		// 然后继续写入。
		struct pollfd pf = { 0 };
		pf.fd = fd;
		pf.events = ( POLLOUT | POLLERR | POLLHUP );
		poll( &pf,1,timeout );  // 非阻塞sleep

		writeret = g_sys_write_func( fd,(const char*)buf + wrotelen,nbyte - wrotelen );
		
		if( writeret <= 0 )
		{
			break;
		}
		wrotelen += writeret ;
	}
	if (writeret <= 0 && wrotelen == 0)
	{
		return writeret;
	}
	return wrotelen;
}
```

注意，关于NON_BLOCKING的read和write：
几个重要的结论：
1. read总是在接收缓冲区有数据时立即返回，而不是等到给定的read buffer填满时返回。
2. 只有当receive buffer为空时，blocking模式才会等待，而nonblock模式下会立即返回-1（errno = EAGAIN或EWOULDBLOCK）;
3. blocking的write只有在缓冲区足以放下整个buffer时才返回（与blocking read并不相同）
4. nonblock write则是返回能够放下的字节数，之后调用则返回-1（errno = EAGAIN或EWOULDBLOCK）
5. 对于blocking的write有个特例：当write正阻塞等待时对面关闭了socket，则write则会立即将剩余缓冲区填满并返回所写的字节数，再次调用则write失败（connection reset by peer）。


####5.4.6.read函数（hook系统调用）
位于`co_hook_sys_call.cpp`，代码如下：
```cpp
ssize_t read( int fd, void *buf, size_t nbyte )
{
	HOOK_SYS_FUNC( read );
	
	if( !co_is_enable_sys_hook() )
	{
		return g_sys_read_func( fd,buf,nbyte );
	}
	rpchook_t *lp = get_by_fd( fd );

	if( !lp || ( O_NONBLOCK & lp->user_flag ) ) 
	{
		ssize_t ret = g_sys_read_func( fd,buf,nbyte );
		return ret;
	}
	int timeout = ( lp->read_timeout.tv_sec * 1000 ) 
				+ ( lp->read_timeout.tv_usec / 1000 );  // 读写超时都默认设置了1s

	struct pollfd pf = { 0 };
	pf.fd = fd;
	pf.events = ( POLLIN | POLLERR | POLLHUP );

	int pollret = poll( &pf,1,timeout );
	// 调用epoll，等待内核回调可读事件
	ssize_t readret = g_sys_read_func( fd,(char*)buf ,nbyte );

	if( readret < 0 )
	{
		co_log_err("CO_ERR: read fd %d ret %ld errno %d poll ret %d timeout %d",
					fd,readret,errno,pollret,timeout);
	}

	return readret;
	
}
```

###5.5.综述
可以看到，cli进程主要就是在开多个协程，每个协程维护一个fd的读写事件。先写成功再读，每次放"samorl"到svr端，非阻塞。
