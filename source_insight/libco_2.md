# libco(2)

@(源码)源码

待续libco(1)。

##### 1.4.1.1.epoll_event数据结构
位于`co_epoll.h`。目前还不清楚这个数据是用来干嘛的。
```cpp
struct epoll_event
{
	uint32_t events;
	epoll_data_t data;
};

// union代表多个成员共用同一块内存
typedef union epoll_data
{
	void *ptr;
	int fd;
	uint32_t u32;
	uint64_t u64;

} epoll_data_t;
```
`epoll_event`的成员`events`是一个bit set。其值可以是下面各值的或。
+ EPOLLINT：监听可读事件。
+ EPOLLOUT：监听可写事件。
+ EPOLLRDHUP ：监听对端断开连接。
+ EPOLLPRI：外带数据。
+ EPOLLERR：Error condition happened on the associated file descriptor.
+ EPOLLHUP：Hang up happened on the associated file descriptor.
+ EPOLLET：边沿触发，epoll监听的fd默认是电平触发。
+ EPOLLONESHOT：对应fd的事件被触发通知后，需要重新调用epoll_ctl对其进行修改(EPOLL_CTL_MOD)，才能恢复事件触发通知

##### 1.4.1.2.kevent数据结构
`kevent`是linux系统API定义的一个数据结构，背景如下：
> poll和select已经是相当不错的复用文件描述符的方法了。但为了使用这两个函数，你需要创建一个描述符的链表，然后把它们发送给内核，在返回的时候又要再次查看这个链表。这看上去有点效率低下。一个更好一些的模型是把描述符链表交给内核，然后就等待。一旦有某个或多个事件发生，内核就把一个只包含有发生了事件的描述符的链表通知给进程，由此避免了每次函数返回的时候都要去遍历整个链表。尽管对于只打开了几个描述符的进程而言这点改进算不得什么，但对于那些打开了几千个文件描述符的程序来说，这种性能改进就相当显著了。这就是kqueue诞生背后的主要目的。同时，设计者还希望进程能够检测更多类型的事件，比如文件修改、文件删除、信号交付或者子进程退出，并提供一个包含了其它任务的灵活的函数调用。处理信号、复用文件描述符、以及等待子进程等操作都可以封装到这个单一的kqueue接口中，因为它们都是在等待某个事件的发生。
> kevent结构体用于和kqueue的通信。FreeBSD上的头文件位于/usr/include/sys/event.h。在这个文件中有对kevent结构体的声明，以及其它的一些选项和标志。和select和poll比起来，kqueue还相当的年轻，所以它一直都在发展和添加新的特性。请查看你的系统头文件以确定任何新的或者特定于系统的选项。

原始的`kevent`结构声明如下：
```cpp
struct kevent {
	uintptr_t ident;  // 文件描述符
	short filter;  // 指定你希望内核用于ident成员的过滤器。
	u_short flags;  // 告诉内核应当对该事件完成哪些操作和处理哪些必要的标志。在返回的时候，flags成员可用于保存错误条件。
	u_int fflags;  // 指定你想让内核使用的特定于过滤器的标志。在返回的时候，fflags成员可用于保存特定于过滤器的返回值。
	intptr_t data;  // data成员用于保存任何特定于过滤器的数据。
	void *udata;  // udata成员并不由kqueue使用，kqueue会把它的值不加修改地透传。这个成员可被进程用来发送信息甚至是一个函数给它自己，用于一些依赖于事件检测的场合。
};
```

#### 1.4.2.co_epoll_res_alloc函数
位于`co_epoll.cpp`，代码如下：
```cpp
// ctx->result =  co_epoll_res_alloc( stCoEpoll_t::_EPOLL_SIZE );
// static const int _EPOLL_SIZE = 1024 * 10;
struct co_epoll_res *co_epoll_res_alloc( int n )
{
	struct co_epoll_res * ptr = 
		(struct co_epoll_res *)malloc( sizeof( struct co_epoll_res ) );

	// 分配n个epoll_events和kevent
	ptr->size = n;
	ptr->events = (struct epoll_event*)calloc( 1,n * sizeof( struct epoll_event ) );
	ptr->eventlist = (struct kevent*)calloc( 1,n * sizeof( struct kevent) );

	return ptr;
}
```

#### 1.4.3.co_epoll_wait函数
位于`co_epoll.cpp`中，代码也很简单：
```cpp
int	co_epoll_wait(
	int epfd,
	struct co_epoll_res *events,
	int maxevents,
	int timeout )
{
	return epoll_wait( epfd,events->events,maxevents,timeout );
}
```
这里调用的是系统函数`epoll_wait`。介绍如下：
> 该函数用于轮询I/O事件的发生；
> 等待事件的产生，类似于select()调用。参数events用来从内核得到事件的集合，maxevents告之内核这个events有多大(数组成员的个数)，这个maxevents的值不能大于创建epoll_create()时的size，参数timeout是超时时间（毫秒，0会立即返回，-1将不确定，也有说法说是永久阻塞）。
> 该函数返回需要处理的事件数目，如返回0表示已超时。
> 返回的事件集合在events数组中，数组中实际存放的成员个数是函数的返回值。返回0表示已经超时。

#### 1.4.4.AddTail函数
位于`co_routine.cpp`中，代码如下：
```cpp
// 就绪函数为空,追加
// stTimeoutItem_t * item;
// stTimeoutItemLink_t * active;
// struct stTimeoutItemLink_t
// {   
//     stTimeoutItem_t *head;
//     stTimeoutItem_t *tail;
// };
// AddTail( active,item );
template <class TNode,class TLink>
void inline AddTail(TLink*apLink,TNode *ap)
{
	// item的stTimeoutItemLink_t不为空,直接返回
	// 证明该时间轮盘不属于该链表,已经放到某个链表下面了
	// TODO:这里是考虑并发访问的情况？
	if( ap->pLink )
	{
		return ;
	}
	// 追加到链表末尾
	if(apLink->tail)
	{
		apLink->tail->pNext = (TNode*)ap;
		ap->pNext = NULL;
		ap->pPrev = apLink->tail;
		apLink->tail = ap;
	}
	else
	{
		//追加到头结点
		apLink->head = apLink->tail = ap;
		ap->pNext = ap->pPrev = NULL;
	}
	ap->pLink = apLink;
}
```

#### 1.4.5.GetTickMS函数
获取当前时间，精确到毫秒级别，位于`co_routine.cpp`，如下：
```cpp
#if defined( __LIBCO_RDTSCP__) 
static unsigned long long counter(void)
{
	register uint32_t lo, hi;
	register unsigned long long o;
	__asm__ __volatile__ (
			"rdtscp" : "=a"(lo), "=d"(hi)::"%rcx"
			);
	o = hi;
	o <<= 32;
	return (o | lo);

}
```
这里通过`rdtscp`这个指令获取一个计数。这个指令的含义是去读一个tsc寄存器。这个寄存器再cpu每个时钟信号到来的时候会加1，比如cpu主频为1MHz的每秒加1000000次。


```cpp
static unsigned long long getCpuKhz()
{
	FILE *fp = fopen("/proc/cpuinfo","r");
	if(!fp) return 1;
	char buf[4096] = {0};
	fread(buf,1,sizeof(buf),fp);
	fclose(fp);

	char *lp = strstr(buf,"cpu MHz");
	if(!lp) return 1;
	lp += strlen("cpu MHz");
	while(*lp == ' ' || *lp == '\t' || *lp == ':')
	{
		++lp;
	}

	double mhz = atof(lp);
	unsigned long long u = (unsigned long long)(mhz * 1000);
	return u;
}
#endif

static unsigned long long GetTickMS()
{
#if defined( __LIBCO_RDTSCP__) 
	static uint32_t khz = getCpuKhz();
	return counter() / khz;
#else
	struct timeval now = { 0 };
	gettimeofday( &now,NULL );
	unsigned long long u = now.tv_sec;
	u *= 1000;
	u += now.tv_usec / 1000;
	return u;
#endif
}
```
这里取出cpu当前主频。用计数去除以主频得到毫秒级的时间。这个时间不是当前的时间，因为我们在计算轮盘超时元素的时候只要算两个值之间的时间差。这里的getCpuKhz是从/proc/cpuinfo 取到的值。

#### 1.4.5.TakeAllTimeout函数
位于`co_routine.cpp`中，代码如下：
```cpp
inline void TakeAllTimeout( 
	stTimeout_t *apTimeout,
	unsigned long long allNow,
	stTimeoutItemLink_t *apResult )
{
	// 最近没有计算过的时刻
	if( apTimeout->ullStart == 0 )
	{
		apTimeout->ullStart = allNow;  // 当前时刻作为start时刻
		apTimeout->llStartIdx = 0;
	}

	// 当前时刻低于时间轮的时间
	// 已经执行过的事件，事件已经加入到列表中了。
	// 无需重复添加
	if( allNow < apTimeout->ullStart )
	{
		return ;
	}
	// 当前时间-剩余时间+1代表走过了多少时间
	int cnt = allNow - apTimeout->ullStart + 1;
	// 不能超过轮盘上的时刻总数
	if( cnt > apTimeout->iItemSize )
	{
		cnt = apTimeout->iItemSize;
	}
	if( cnt < 0 )
	{
		return;
	}
	for( int i = 0;i<cnt;i++)
	{
		// 找出每个时刻在时间轮盘上的item
		int idx = ( apTimeout->llStartIdx + i) % apTimeout->iItemSize;
		Join<stTimeoutItem_t,stTimeoutItemLink_t>( apResult,apTimeout->pItems + idx  );  // 将apTimeout里面的项加入到apResult
	}
	apTimeout->ullStart = allNow;
	apTimeout->llStartIdx += cnt - 1;
}
```

##### 1.4.5.1.Join函数
位于`co_routine.cpp`中，代码如下：
```cpp
template <class TNode,class TLink>
void inline Join( TLink*apLink,TLink *apOther )
{
	//printf("apOther %p\n",apOther);
	// 加入进来的为一个空链表,直接返回
	if( !apOther->head )
	{
		return ;
	}
	// 找出apOther的头结点
	TNode *lp = apOther->head;
	while( lp )
	{
		// 每个时间片挂在时间轮上面
		lp->pLink = apLink;
		lp = lp->pNext;
	}
	lp = apOther->head;
	// 尾部不为空
	if(apLink->tail)
	{
		// 时间片挂在上面
		apLink->tail->pNext = (TNode*)lp;
		lp->pPrev = apLink->tail;
		apLink->tail = apOther->tail;
	}
	else
	{
		// 空链表，直接替代
		apLink->head = apOther->head;
		apLink->tail = apOther->tail;
	}
	// 析构原来的apOther
	apOther->head = apOther->tail = NULL;
}
```

#### 1.4.6.PopHead函数
位于`co_routine.cpp`，代码如下：
```cpp
template <class TNode,class TLink>
void inline PopHead( TLink*apLink )
{
	// apLink的头结点为空,证明是个空链表
	if( !apLink->head ) 
	{
		return ;
	}
	TNode *lp = apLink->head;
	if( apLink->head == apLink->tail )
	{
		// 有且仅有一个节点,置为空
		apLink->head = apLink->tail = NULL;
	}
	else
	{
		// 头结点赋值给下一个节点
		apLink->head = apLink->head->pNext;
	}

	lp->pPrev = lp->pNext = NULL;
	lp->pLink = NULL;

	if( apLink->head )
	{
		apLink->head->pPrev = NULL;
	}
}
```

#### 1.4.7.AddTimeout函数
位于`co_routine.cpp`，代码如下：
```cpp
// AddTimeout(ctx->pTimeout, lp, now);
int AddTimeout( 
	stTimeout_t *apTimeout,
	stTimeoutItem_t *apItem ,
	unsigned long long allNow )
{
	if( apTimeout->ullStart == 0 )
	{
		// timeout列表未开始计数
		apTimeout->ullStart = allNow;
		apTimeout->llStartIdx = 0;
	}
	if( allNow < apTimeout->ullStart )
	{
		// timeout列表还没有开始计时
		co_log_err("CO_ERR: AddTimeout line %d allNow %llu apTimeout->ullStart %llu",
					__LINE__,allNow,apTimeout->ullStart);

		return __LINE__;
	}
	if( apItem->ullExpireTime < allNow )
	{
		// 过期项不加入到timeout列表
		co_log_err("CO_ERR: AddTimeout line %d apItem->ullExpireTime %llu allNow %llu apTimeout->ullStart %llu",
					__LINE__,apItem->ullExpireTime,allNow,apTimeout->ullStart);

		return __LINE__;
	}
	// 计算列表
	unsigned long long diff = apItem->ullExpireTime - apTimeout->ullStart;

	if( diff >= (unsigned long long)apTimeout->iItemSize )
	{
		// diff最多为当前列表长度减1
		diff = apTimeout->iItemSize - 1;
		co_log_err("CO_ERR: AddTimeout line %d diff %d",
					__LINE__,diff);

		//return __LINE__;
	}
	// TODO:暂时未知道什么意思
	AddTail( apTimeout->pItems + ( apTimeout->llStartIdx + diff ) % apTimeout->iItemSize , apItem );

	return 0;
}
```

### 1.5.co_eventloop函数综述
这里函数主要用到了libco实现了时间轮盘的特点。轮盘上把一些网络io的事件挂上去。当发生对应网络事件的时候，切换到对应协程。或是当网络io事件超时的时候也切换到协程。事件的结构体有变量标记当前这个事件是超时了还是被触发了。TODO：后面这里会补上数据结构图。

`co_eventloop`函数的步骤如下:
1. 获取所有的事件列表；`co_epoll_res *result = ctx->result;`
2. `stTimeoutItemLink_t *active = (ctx->pstActiveList);`active列表挂载的是要发生的事件列表；
3. 调用`co_epoll_wait`陷入阻塞，知道内核通知事件就绪；
4. 遍历就绪的事件列表，取出对应的事件；
5. 如果触发函数为非空，则执行触发函数（该函数其实能标记事件由epoll触发）。`item->pfnPrepare( item,result->events[i],active );`该函数顺便也放到active列表中；
6. 就绪函数为空，直接把事件加入到active列表；
7. 计算当前时间`unsigned long long now = GetTickMS();`，获取所有超时事件。
8. 查看时间轮盘，找出上次计算的时间点与当前时间相差的时间项加入到timeout列表中。同时将时间轮盘向前推；
9. 遍历timeout列表，给他们打上超时引起的标记；
10. 然后将超时引起的事件项加入到active列表中。相当于两部分事件列表：active（epoll触发的事件），timeout（时间轮盘统计超时的事件）；
11. 遍历所有的active列表，把还没有超时的函数放到timeout列表（可能存在某些时间必须超时才能触发，不能提前触发），超时时间以`ullExpireTime`为准；
12. 执行每个事件的回调函数`lp->pfnProcess( lp );`；
13. 每次循环事件，执行pfn函数，看是否需要退出循环，终止事件循环机制。`if (-1 == pfn( arg )) { break; }`。

### 1.6.数据结构图
epoll事件数据结构
![2-1.png](https://github.com/sysublackbear/libco_source_study/blob/master/libco_pic/2-1.png)

stTimeout_t时间轮数据结构：
![2-2.png](https://github.com/sysublackbear/libco_source_study/blob/master/libco_pic/2-2.png)

stTimeoutItemLink_t 和 stTimeoutItem_t 数据结构
![2-3.png](https://github.com/sysublackbear/libco_source_study/blob/master/libco_pic/2-3.png)

有关定时器实现：
+ https://blog.csdn.net/anonymalias/article/details/52022787
+ http://novoland.github.io/%E5%B9%B6%E5%8F%91/2014/07/26/%E5%AE%9A%E6%97%B6%E5%99%A8%EF%BC%88Timer%EF%BC%89%E7%9A%84%E5%AE%9E%E7%8E%B0.html
