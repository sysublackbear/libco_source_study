# libco(10)
@(源码)

## 7.example_poll.cpp

我们看下实现源码，初步看像是在几个连接的fd里面进行epoll操作：
```cpp
int main(int argc,char *argv[])
{
	vector<task_t> v;
	for(int i=1;i<argc;i+=2)
	{
		// ./example_poll 127.0.0.1 12365 127.0.0.1 12222 192.168.1.1 1000 192.168.1.2 1111
		// 读取多组ip和port，塞到vector中

		task_t task = { 0 };
		SetAddr( argv[i],atoi(argv[i+1]),task.addr );
		v.push_back( task );
	}

//------------------------------------------------------------------------------------
	printf("--------------------- main -------------------\n");
	vector<task_t> v2 = v;  // 拷贝vector
	poll_routine( &v2 );  // 没有做任何操作啊，ip和port都是空的。
	printf("--------------------- routine -------------------\n");

	for(int i=0;i<10;i++)
	{
		stCoRoutine_t *co = 0;
		vector<task_t> *v2 = new vector<task_t>();
		*v2 = v;
		co_create( &co,NULL,poll_routine,v2 );
		printf("routine i %d\n",i);
		co_resume( co );
	}

	co_eventloop( co_get_epoll_ct(),0,0 );

	return 0;
}
```

### 7.1.poll_routine函数
位于`example_poll.cpp`中，代码如下：
```cpp
// poll_routine(&v2);
static void *poll_routine( void *arg )
{
	co_enable_hook_sys();

	vector<task_t> &v = *(vector<task_t>*)arg;
	for(size_t i=0;i<v.size();i++)
	{
		int fd = CreateTcpSocket();
		SetNonBlock( fd );
		v[i].fd = fd;

		int ret = connect(fd,(struct sockaddr*)&v[i].addr,sizeof( v[i].addr )); 
		printf("co %p connect i %ld ret %d errno %d (%s)\n",
			co_self(),i,ret,errno,strerror(errno));
	}
	struct pollfd *pf = (struct pollfd*)calloc( 1,sizeof(struct pollfd) * v.size() );

	for(size_t i=0;i<v.size();i++)
	{
		pf[i].fd = v[i].fd;
		pf[i].events = ( POLLOUT | POLLERR | POLLHUP );
	}
	set<int> setRaiseFds;
	size_t iWaitCnt = v.size();
	for(;;)
	{
		int ret = poll( pf,iWaitCnt,1000 );
		printf("co %p poll wait %ld ret %d\n",
				co_self(),iWaitCnt,ret);
		for(int i=0;i<ret;i++)
		{
			printf("co %p fire fd %d revents 0x%X POLLOUT 0x%X POLLERR 0x%X POLLHUP 0x%X\n",
					co_self(),
					pf[i].fd,
					pf[i].revents,
					POLLOUT,
					POLLERR,
					POLLHUP
					);
			setRaiseFds.insert( pf[i].fd );
		}
		if( setRaiseFds.size() == v.size())
		{
			break;
		}
		if( ret <= 0 )
		{
			break;
		}

		iWaitCnt = 0;
		for(size_t i=0;i<v.size();i++)
		{
			if( setRaiseFds.find( v[i].fd ) == setRaiseFds.end() )
			{
				pf[ iWaitCnt ].fd = v[i].fd;
				pf[ iWaitCnt ].events = ( POLLOUT | POLLERR | POLLHUP );
				++iWaitCnt;
			}
		}
	}
	for(size_t i=0;i<v.size();i++)
	{
		close( v[i].fd );
		v[i].fd = -1;
	}

	printf("co %p task cnt %ld fire %ld\n",
			co_self(),v.size(),setRaiseFds.size() );
	return 0;
}
```

#### 7.1.1.CreateTcpSocket函数
位于`example_poll.cpp`，代码如下：
```cpp
// int fd = CreateTcpSocket();
static int CreateTcpSocket(
	const unsigned short shPort  = 0 ,
	const char *pszIP  = "*" ,
	bool bReuse  = false )
{
	// ipv4
	// 流式
	// tcp
	int fd = socket(AF_INET,SOCK_STREAM, IPPROTO_TCP);
	if( fd >= 0 )
	{
		if(shPort != 0)
		{
			if(bReuse)
			{
				// 是否复用连接
				int nReuseAddr = 1;
				setsockopt(fd,SOL_SOCKET,SO_REUSEADDR,&nReuseAddr,sizeof(nReuseAddr));
			}
			struct sockaddr_in addr ;
			SetAddr(pszIP,shPort,addr);
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

综述，没什么好说，就是主协程开多个协程监听多个ip和port组成的socket
