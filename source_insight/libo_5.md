#libco(5)

@(源码)

#####3.5.7.协程切换的流程图
如图所示：
![Alt text](./1529139557214.png)

###3.6.Consumer函数
位于`example_cond.cpp`，代码如下：
```cpp
// args = env;
void* Consumer(void* args)
{
	// 打开当前协程的hook开关。
	co_enable_hook_sys();
	stEnv_t* env = (stEnv_t*)args;
	// stEnv_t* env为进程级别
	while (true)
	{
		if (env->task_queue.empty())
		{
			// 如果任务队列为空，挂
			co_cond_timedwait(env->cond, -1);
			continue;
		}
		// 任务队列,出队
		stTask_t* task = env->task_queue.front();
		env->task_queue.pop();
		// 代表消费
		printf("%s:%d consume task %d\n", __func__, __LINE__, task->id);
		free(task);
	}
	return NULL;
}
```

####3.6.1.co_enable_hook_sys函数
位于`co_hook_sys_call.cpp`，代码如下：
```cpp
// 打开系统的hook开关
void co_enable_hook_sys()
{
	// 获取当前线程的协程环境的调用栈栈顶协程。
	stCoRoutine_t *co = GetCurrThreadCo();
	if( co )
	{
		co->cEnableSysHook = 1;
	}
}
```

#####3.6.1.1.GetCurrThreadCo函数
位于`co_routine.cpp`，代码如下：
```cpp
// 获取当前线程的协程环境的调用栈栈顶协程。
stCoRoutine_t *GetCurrCo( stCoRoutineEnv_t *env )
{
	return env->pCallStack[ env->iCallStackSize - 1 ];
}
stCoRoutine_t *GetCurrThreadCo( )
{
	stCoRoutineEnv_t *env = co_get_curr_thread_env();
	if( !env ) return 0;
	return GetCurrCo(env);
}
```

####3.6.2.co_cond_timedwait函数
位于`co_routine.cpp`，代码如下：
```cpp
// co_cond_timedwait(env->cond, -1);
// 每一个stCoCond_t都绑定了一个定时器
int co_cond_timedwait( stCoCond_t *link,int ms )
{
	// 分配stCoCondItem_t节点
	stCoCondItem_t* psi = (stCoCondItem_t*)calloc(1, sizeof(stCoCondItem_t));
	psi->timeout.pArg = GetCurrThreadCo();
	// 回调函数
	psi->timeout.pfnProcess = OnSignalProcessEvent;

	// 等待时间大于0
	if( ms > 0 )
	{
		unsigned long long now = GetTickMS();
		psi->timeout.ullExpireTime = now + ms;
		// 加入到Timeout列表
		int ret = AddTimeout( co_get_curr_thread_env()->pEpoll->pTimeout,&psi->timeout,now );
		if( ret != 0 )
		{
			// 加入失败，释放空间
			free(psi);
			return ret;
		}
	}
	// 将psi挂到link，等待调度。
	AddTail( link, psi);

	co_yield_ct();
	// co_yield和co_yield_ct都是挂起协程，其区别也一目了然。虽然两者效果相同，但目前使用较多的是co_yield_ct，因为同一时刻只有一个协程在运行，那么挂起操作当然就是挂当前协程，所以直接调用co_yield_ct就可以了。协程内部可以保证协程被唤醒后，会接着挂起的地方继续执行，但业务需要保证的是，当前被唤醒的协程确实是你想要唤醒的。

	// 协程唤醒后，会在这继续执行
	// 将psi从列表中移除
	RemoveFromLink<stCoCondItem_t,stCoCond_t>( psi );
	// 释放psi
	free(psi);

	return 0;
}
```

#####3.6.2.1.co_yield_ct函数
位于`co_routine.cpp`，代码如下：
```cpp
void co_yield_env( stCoRoutineEnv_t *env )
{
	// 上一层协程
	stCoRoutine_t *last = env->pCallStack[ env->iCallStackSize - 2 ];
	// 当前协程
	stCoRoutine_t *curr = env->pCallStack[ env->iCallStackSize - 1 ];

	env->iCallStackSize--;

	co_swap( curr, last);
}

// 切出当前协程，回溯到上一层协程
void co_yield_ct()
{
	co_yield_env( co_get_curr_thread_env() );
}
```

#####3.6.2.2.OnSignalProcessEvent函数
位于`co_routine.cpp`，代码如下：
```cpp
// cosumer协程的回调函数
static void OnSignalProcessEvent( stTimeoutItem_t * ap )
{
	stCoRoutine_t *co = (stCoRoutine_t*)ap->pArg;
	co_resume( co );
}
```

###3.7.Producer函数
位于`example_cond.cpp`，代码如下：
```cpp
// co_create(&producer_routine, NULL, Producer, env);
void* Producer(void* args)
{
	// 打开当前协程的hook开关
	co_enable_hook_sys();
	stEnv_t* env=  (stEnv_t*)args;
	int id = 0;
	while (true)
	{
		// 生产任务stTask_t对象
		stTask_t* task = (stTask_t*)calloc(1, sizeof(stTask_t));
		task->id = id++;
		env->task_queue.push(task);
		// 生成协程
		printf("%s:%d produce task %d\n", __func__, __LINE__, task->id);
		co_cond_signal(env->cond);
		// 这里使用的是打开了hook开关而实现的poll函数
		poll(NULL, 0, 1000);
	}
	return NULL;
}
```
**！！！注意**：这里`poll(NULL, 0, 1000);`的目的是为了实现一个类似`sleep`函数的功能，即睡眠一段时间等待触发。这里不能使用`sleep`函数，因为`sleep`函数的实现是：
1. 注册一个信号signal(SIGALRM,handler)。接收内核给出的一个信号。
2. 调用alarm()函数。
3. pause()挂起进程。
由此可见，sleep会挂起当前进程，如果调用，所有的其他协程将被挂起，无法进行正常的协程调度。

####3.7.1.co_cond_signal函数
位于`co_routine.cpp`，代码如下：
```cpp
int co_cond_signal( stCoCond_t *si )
{
	// 从stCoCond_t链表中取出头结点
	stCoCondItem_t * sp = co_cond_pop( si );
	if( !sp ) 
	{
		return 0;
	}
	// 将sp->timeout从等待列表中去除
	RemoveFromLink<stTimeoutItem_t,stTimeoutItemLink_t>( &sp->timeout );

	// 添加到就绪列表
	AddTail( co_get_curr_thread_env()->pEpoll->pstActiveList,&sp->timeout );

	return 0;
}
```

#####3.7.1.1.co_cond_pop函数
位于`co_routine.cpp`，代码如下：
```cpp
// stCoCondItem_t * sp = co_cond_pop( si );
stCoCondItem_t *co_cond_pop( stCoCond_t *link )
{
	stCoCondItem_t *p = link->head;
	if( p )
	{
		PopHead<stCoCondItem_t,stCoCond_t>( link );
	}
	return p;
}
```

####3.7.2.poll函数
位于`co_hook_sys_call.cpp`，代码如下：
```cpp
// poll(NULL, 0, 1000);
int poll(struct pollfd fds[], nfds_t nfds, int timeout)
{

	HOOK_SYS_FUNC( poll );

	if( !co_is_enable_sys_hook() )
	{
		// 不在协程中或者协程没有打开hook开关
		// 直接调原来的系统调用。
		return g_sys_poll_func( fds,nfds,timeout );
	}

	return co_poll_inner( co_get_epoll_ct(),fds,nfds,timeout, g_sys_poll_func);
}
```
**注意**：之所以要hook，是因为系统调用级别的poll函数会阻塞整个线程，而我们需要达到的效果是只阻塞当前协程，

####3.7.3.HOOK_SYS_FUNC函数
位于`co_hook_sys_call.cpp`，代码如下：
```cpp
#define HOOK_SYS_FUNC(name) if( !g_sys_##name##_func ) { g_sys_##name##_func = (name##_pfn_t)dlsym(RTLD_NEXT,#name); }
```
比如上面是`poll`，相当于hook掉`poll`调用：
```cpp 
if( !g_sys_poll_func ) 
{
	g_sys_poll_func = (poll_pfn_t)dlsym(RTLD_NEXT,poll); 
}
```
注意：
```cpp
typedef int (*poll_pfn_t)(struct pollfd fds[], nfds_t nfds, int timeout);
```
可见，`g_sys_poll_func`只一个函数指针。
其中，`dlsym`是个系统函数。
> dlsym是一个计算机函数，功能是根据动态链接库操作句柄与符号，返回符号对应的地址，不但可以获取函数地址，也可以获取变量地址。
> 主要用于运行时加载动态库。可执行文件在运行时可以加载不同的动态库，这就为hook系统函数提供了基础。

`RTLD_NEXT`允许从调用方链接映射列表中的下一个关联目标文件获取符号。即动态链接时**跳过**当前库文件的`poll`函数。如3.7.2。

####3.7.4.co_is_enable_sys_hook函数
位于`co_routine.cpp`，代码如下：
```cpp
bool co_is_enable_sys_hook()
{
	// 获取当前线程的协程栈栈顶协程
	stCoRoutine_t *co = GetCurrThreadCo();
	// 判断协程是否为空，且已经打开了hook开关
	return ( co && co->cEnableSysHook );
}
```

####3.7.5.co_poll_inner函数
位于`co_routine.cpp`，代码如下：
hook后重新实现的`poll`函数。
```cpp
// return co_poll_inner( co_get_epoll_ct(),fds,nfds,timeout, g_sys_poll_func);
// co_poll_inner(co_get_epoll_ct(), NULL, 0, 1000);
// fds:NULL
// nfds:0
// timeout:1000
typedef int (*poll_pfn_t)(struct pollfd fds[], nfds_t nfds, int timeout);
// co_get_epoll_ct()：获取协程环境的epoll服务。
int co_poll_inner( stCoEpoll_t *ctx,struct pollfd fds[], nfds_t nfds, int timeout, poll_pfn_t pollfunc)
{
	// timeout == 0表示非阻塞调用，直接执行poll系统调用
    if (timeout == 0)
	{
		// g_sys_poll_func
		return pollfunc(fds, nfds, timeout);
	}
	// timeout为负值，无限阻塞
	if (timeout < 0)
	{
		timeout = INT_MAX;
	}
	// 获取epoll_fd，在AllocEpoll里面的co_epoll_create函数里面初始化
	int epfd = ctx->iEpollFd;
	// 获取当前线程环境的协程栈栈顶协程
	stCoRoutine_t* self = co_self();

	//1.struct change
	// stPoll_t继承自stTimeoutItem_t
	stPoll_t& arg = *((stPoll_t*)malloc(sizeof(stPoll_t)));
	memset( &arg,0,sizeof(arg) );

	arg.iEpollFd = epfd;  // 填充epoll_fd
	arg.fds = (pollfd*)calloc(nfds, sizeof(pollfd));  // 填充poll_fds,生成nfds个pollfd结构
	arg.nfds = nfds;  // 填充nfds

	// 雷哥说的：这里对内存分配做了小优化
	// 对于只有1个fd要监听的情况并且是私有栈模式直接使用栈空间，否则从堆上分配空间
	// 因为栈分配的空间的性能要被在堆上分配空间的性能要高，关于栈分配空间和堆分配空间的差异，可见下面。
	stPollItem_t arr[2];
	if( nfds < sizeof(arr) / sizeof(arr[0]) && !self->cIsShareStack) // 相当于nfds < 2 && !self->cIsShareStack
	{
		// 直接使用栈上空间
		arg.pPollItems = arr;
	}	
	else
	{
		// 从堆中分配空间
		arg.pPollItems = (stPollItem_t*)malloc( nfds * sizeof( stPollItem_t ) );
	}
	memset( arg.pPollItems,0,nfds * sizeof(stPollItem_t) );

	// 回调函数
	arg.pfnProcess = OnPollProcessEvent;
	// 当前栈顶协程
	arg.pArg = GetCurrCo( co_get_curr_thread_env() );
	
	
	//2. add epoll
	for(nfds_t i=0;i<nfds;i++)
	{
		// 当前fd事件
		arg.pPollItems[i].pSelf = arg.fds + i;
		arg.pPollItems[i].pPoll = &arg;
		// 就绪函数
		arg.pPollItems[i].pfnPrepare = OnPollPreparePfn;
		struct epoll_event &ev = arg.pPollItems[i].stEvent;

		if( fds[i].fd > -1 )  // 等于fds+i
		{
			// 给arg.pPollItems[i].stEvent赋值
			ev.data.ptr = arg.pPollItems + i;
			ev.events = PollEvent2Epoll( fds[i].events );
			// 将poll的event（监听事件）转化为epoll
			
			// 底层调用epoll_ctl(epfd,EPOLL_CTL_ADD, fds[i].fd, &ev);
			// 注册新的fd到epfd中
			int ret = co_epoll_ctl( epfd,EPOLL_CTL_ADD, fds[i].fd, &ev );
			// epoll_ctl成功返回0,失败返回-1
			if (ret < 0 && errno == EPERM && nfds == 1 && pollfunc != NULL)
			{
				if( arg.pPollItems != arr )
				{
					// 释放堆中空间
					free( arg.pPollItems );
					arg.pPollItems = NULL;
				}
				// 回收fds
				free(arg.fds);
				// 回收arg
				free(&arg);
				// 调用epoll失败，又再调用了poll系统调用进行兜底
				return pollfunc(fds, nfds, timeout);
			}
		}
		//if fail,the timeout would work
	}

	//3.add timeout

	unsigned long long now = GetTickMS();
	arg.ullExpireTime = now + timeout;  // 建立定时器
	int ret = AddTimeout( ctx->pTimeout,&arg,now );  // 将pollItem_t加入到定时器列表
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
		// 切换协程（因为把pollItem填加到定时器即可，等着让它触发，然后切出协程即可。）
		// 挂起本协程等待事件激活或者超时
		co_yield_env( co_get_curr_thread_env() );
		// 返回iRaiseCnt
		iRaiseCnt = arg.iRaiseCnt;
	}

    {
		//clear epoll status and memory
		// 清除epoll占用的空间
		// 从定时器中取出定时项
		RemoveFromLink<stTimeoutItem_t,stTimeoutItemLink_t>( &arg );
		for(nfds_t i = 0;i < nfds;i++)
		{
			int fd = fds[i].fd;
			if( fd > -1 )
			{
				// 从epollfd中删除fd
				co_epoll_ctl( epfd,EPOLL_CTL_DEL,fd,&arg.pPollItems[i].stEvent );
			}
			// 将已有事件返回
			fds[i].revents = arg.fds[i].revents;
		}


		if( arg.pPollItems != arr )
		{
			// 回收空间
			free( arg.pPollItems );
			arg.pPollItems = NULL;
		}

		free(arg.fds);
		free(&arg);
	}
	// 返回iRaiseCnt
	return iRaiseCnt;
}

stCoRoutine_t *co_self()
{
	// 获取当前线程环境的协程栈栈顶协程
	return GetCurrThreadCo();
}

struct stPoll_t : public stTimeoutItem_t 
{
	struct pollfd *fds;
	nfds_t nfds; // typedef unsigned long int nfds_t;

	stPollItem_t *pPollItems;

	int iAllEventDetach;

	int iEpollFd;

	int iRaiseCnt;


};

struct stPollItem_t : public stTimeoutItem_t
{
	struct pollfd *pSelf;
	stPoll_t *pPoll;

	struct epoll_event stEvent;
};
```

**这里需要知道poll函数的基本用法以及各个参数的含义：**
```cpp
#include <poll.h>
int poll(struct pollfd fd[], nfds_t nfds, int timeout);
```
> 参数：
> 1）第一个参数:一个结构数组,struct pollfd结构如下：
> struct pollfd{
>     int fd;              //文件描述符
>     short events;    //请求的事件
>     short revents;   //返回的事件
>     };
>     events和revents是通过对代表各种事件的标志进行逻辑或运算构建而成的。events包括要监视的事件，poll用已经发生的事件填充revents。
> 2）第二个参数nfds：要监视的描述符的数目。
> 3）最后一个参数timeout：是一个用毫秒表示的时间，是指定poll在返回前没有接收事件时应该等待的时间。如果  它的值为-1，poll就永远都不会超时。如果整数值为32个比特，那么最大的超时周期大约是30分钟。

**栈分配空间和堆分配空间的差异：**
先看一个例子．
char c; //栈上分配
char *p = new char[3]; //堆上分配，将地址赋给了p;

在 编译器遇到第一条指令时，计算其大小，然后去查找当前栈的空间是大于所需分配的空间大小，如果这时栈内空间大于所申请的空间，那么就为其分配内存空间，注 意：在这里，内在空间的分配是连续的，是接着上次分配结束后进行分配的．如果栈内空间小于所申请的空间大小，那么这时系统将揭示栈溢出，并给出相应的异常 信息．

编译器遇到第二条指令时，由于p是在栈上分配的，所以在为p分配内在空间时和上面的方法一样，但当遇到new关 键字，那么编译器都知道，这是用户申请的动态内存空间，所以就会转到堆上去为其寻找空间分配．大家注意：堆上的内存空间不是连续的，它是由相应的链表将其 空间区时的内在区块连接的，所以在接到分配内存空间的指定后，它不会马上为其分配相应的空间，而是先要计算所需空间，然后再到遍列整个堆（即遍列整个链的 节点），将第一次遇到的内存块分配给它．最后再把在堆上分配的字符数组的首地址赋给p.，这个时候，大家已经清楚了，p中现在存放的是在堆中申请的字符数组的首地址，也就是在堆中申请的数组的地址现在被赋给了在栈上申请的指针变量p．为了更加形象的说明问题，请看下图：
![Alt text](./1529512024431.png)




