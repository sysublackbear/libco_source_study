#libco(1)

@(源码)


看了下libco的源码介绍，如下：
> libco是微信后台大规模使用的c/c++协程库，2013年至今稳定运行在微信后台的数万台机器上。  
> libco通过仅有的几个函数接口 co_create/co_resume/co_yield 再配合 co_poll，可以支持同步或者异步的写法，如线程库一样轻松。同时库里面提供了socket族函数的hook，使得后台逻辑服务几乎不用修改逻辑代码就可以完成异步化改造。
> libco的特性
> 无需侵入业务逻辑，把多进程、多线程服务改造成协程服务，并发能力得到百倍提升;
> 支持CGI框架，轻松构建web服务(New);
> 支持gethostbyname、mysqlclient、ssl等常用第三库(New);
> 可选的共享栈模式，单机轻松接入千万连接(New);
> 完善简洁的协程编程接口
> 类pthread接口设计，通过co_create、co_resume等简单清晰接口即可完成协程的创建与恢复；
>  __thread的协程私有变量、协程间通信的协程信号量co_signal (New);
>  语言级别的lambda实现，结合协程原地编写并执行后台异步任务 (New);
>  基于epoll/kqueue实现的小而轻的网络框架，基于时间轮盘实现的高性能定时器;

决定先从几个example的源码开始看。后面有可能会穿插介绍libco里面，协程是怎么实现的。

##1.example_thread.cpp
main函数非常简单，创建若干个线程，每个线程的入口运行函数为`routine_func`。
```cpp
int main(int argc,char *argv[])
{
	int cnt = atoi( argv[1] );

	pthread_t tid[ cnt ];
	for(int i=0;i<cnt;i++)
	{
		pthread_create( tid + i,NULL,routine_func,0);
	}
	for(;;)
	{
		sleep(1);
	}
	
	return 0;
}
```
然后我们看一下`routine_func`这个函数到底做了啥？
```cpp
int loop(void *)
{
	return 0;
}
static void *routine_func( void * )
{
	stCoEpoll_t * ev = co_get_epoll_ct(); //ct = current thread
	co_eventloop( ev,loop,0 );
	return 0;
}
```

###1.1.co_get_epoll_ct()函数
其中，ct = current thread。这个函数也有简单，但是需要理解清楚其中的函数怎么实现的。
看函数的含义，像是为每一个线程获取对应的epoll事件。
```cpp
stCoEpoll_t *co_get_epoll_ct()
{ 
    if( !co_get_curr_thread_env() )  // 根据当前线程,获取对应的协程环境,一个线程对应一个协程环境
    {           
        co_init_curr_thread_env();  // 初始化协程环境
    }           
    return co_get_curr_thread_env()->pEpoll;  // 返回管理协程的epoll
} 
```

####1.1.1.stCoEpoll_t 事件驱动的数据结构
数据结构如下：
```cpp
struct stCoEpoll_t
{                                              
    int iEpollFd;  // 一个epoll事件监听的fd
    static const int _EPOLL_SIZE = 1024 * 10;
    struct stTimeout_t *pTimeout;  // 时间轮，具体是什么待会看。
    struct stTimeoutItemLink_t *pstTimeoutList;  // 临时保存到期的定时器，不知道具体是什么
    struct stTimeoutItemLink_t *pstActiveList;  // 即将要发生的事件列表
    co_epoll_res *result;  // TODO:这个数据结构具体也不知道是什么
};
```

什么是fd？
> fd 是(file descriptor)，这种一般是BSD Socket的用法，用在Unix/Linux系统上。在Unix/Linux系统下，一个socket句柄，可以看做是一个文件，在socket上收发数据，相当于对一个文件进行读写，所以一个socket句柄，通常也用表示文件句柄的fd来表示。

####1.1.2.stTimeout_t 时间轮的数据结构
时间轮盘的数据结构如下：
```cpp
struct stTimeout_t
{   
    stTimeoutItemLink_t *pItems;  // 指向上面的时间轮盘
    int iItemSize;  // 轮盘上面的时刻总数
    unsigned long long ullStart;  // 最近计算过的时间，毫秒级别
    long long llStartIdx;  // 相当于轮盘的指针
};
struct stTimeoutItemLink_t
{   
    stTimeoutItem_t *head;
    stTimeoutItem_t *tail;
};
// 挂在轮盘上面的每个item的具体数据结构
struct stTimeoutItem_t
{
 
    enum
    {  
        eMaxTimeout = 40 * 1000 //40s
    }; 
    stTimeoutItem_t *pPrev;  // 双向链表
    stTimeoutItem_t *pNext;
    stTimeoutItemLink_t *pLink;
	// 过期时间
    unsigned long long ullExpireTime;
 
	// 就绪的函数
    OnPreparePfn_t pfnPrepare;  // 注册一个函数,函数成员stTimeoutItem,epoll_event,stTimeoutItemLink_t
    // 事件触发或超时引起后所调用的回调函数
    OnProcessPfn_t pfnProcess;  // 注册一个函数,函数成员stTimeoutItem
    // 指向被挂起的协程,具体怎么回事待会看下
    void *pArg; // routine 
    bool bTimeout;  // 这个元素用来标记是否由于超时引起的
};     
typedef void (*OnPreparePfn_t)( stTimeoutItem_t *,struct epoll_event &ev, stTimeoutItemLink_t *active );
typedef void (*OnProcessPfn_t)( stTimeoutItem_t *);
```

###1.2.co_get_curr_thread_env()函数
这个函数的实现逻辑也很简单：
```cpp
stCoRoutineEnv_t *co_get_curr_thread_env()
{                 
    return g_arrCoEnvPerThread[ GetPid() ];
}
static stCoRoutineEnv_t* g_arrCoEnvPerThread[ 204800 ] = { 0 };
```
g_arrCoEnvPerThread：全局的，记录每个线程含有的所有协程环境的数组空间，一个线程可以开204800个协程环境，每个线程通过自身的tid作为下标在进程空间的数组访问。

####1.2.1.GetPid()
不同linux的系统函数`getpid()`，这里考虑跨平台因素重新实现了`GetPid()`：
```cpp
static pid_t GetPid()
{
	// 注意: pid,tid为static，static只会初始化一次，变量存放在堆中，再次重入该函数时，会略过该行。（static __thread为线程级别）                   
    static __thread pid_t pid = 0;  // 线程局部存储(tls)
    static __thread pid_t tid = 0;  // __thread是GCC内置的线程局部存储设施，存取效率可以和全局变量相比。
    if( !pid || !tid || pid != getpid() )
    {               
        pid = getpid();  // 获取进程id
        // 根据不同的操作系统内核,获取不同的线程pid，记为tid          
#if defined( __APPLE__ )         
        tid = syscall( SYS_gettid );    
        if( -1 == (long)tid )    
        {           
            tid = pid;  // 只有一个线程,线程id等于进程id？           
        }           
#elif defined( __FreeBSD__ )     
        syscall(SYS_thr_self, &tid);    
        if( tid < 0 )            
        {           
            tid = pid;           
        }           
#else               
        tid = syscall( __NR_gettid );   
#endif              
                    
    }               
    return tid;                         
} 
```
关于tls：
>  __thread是GCC内置的线程局部存储设施，存取效率可以和全局变量相比。
>  __thread变量每一个线程有一份独立实体，各个线程的值互不干扰。可以用来修饰那些带有全局性且值可能变，但是又不值得用全局变量保护的变量。

####1.2.2.stCoRoutineEnv_t 数据结构
在协程被创建前，需要有个某进程/线程下的协程环境结构体变量。这个变量用于管理当前进程/线程下的协程。
```cpp
struct stCoRoutineEnv_t
{                                         
    stCoRoutine_t *pCallStack[ 128 ];  // 协程栈,最多递归调用128层
    int iCallStackSize;  // 实际占用的栈空间大小
    stCoEpoll_t *pEpoll;  // 管理协程的网络调用时的epoll，以及定时器
   
    //for copy stack log lastco and nextco
    stCoRoutine_t* pending_co;  // 将要运行的，等待中的协程
    stCoRoutine_t* occupy_co;  // 运行中的，要被调度的协程
};
```
其中,`stCoEpoll_t`的具体数据结构在上面1.1.1有所介绍。

####1.2.3.stCoRoutine_t——协程的数据结构
位于co_routine_inner.h中。数据结构如下：
```cpp
struct stCoRoutine_t
{            
    stCoRoutineEnv_t *env; //协程关联的协程环境结构体变量
    pfn_co_routine_t pfn;  // 协程的入口函数
    // typedef void *(*pfn_co_routine_t)( void *);
    void *arg;  // 传入入口函数的参数
    coctx_t ctx;  // 协程上下文，下面会有数据结构进行介绍
             
    char cStart;  // 协程开始运行标志
    char cEnd;  // 协程结束运行标志
    char cIsMain;  // 主协程名称?TODO：后面补充
    char cEnableSysHook;  // 是否开启系统调用hook，TODO：后面补充
    char cIsShareStack;
             
    // 下面这些变量目前不清楚有什么用?TODO:后面补充
    void *pvEnv;
             
    //char sRunStack[ 1024 * 128 ];
    stStackMem_t* stack_mem;  // 每个协程的协程栈空间
             
             
    //save satck buffer while confilct on same stack_buffer;
    char* stack_sp; 
    unsigned int save_size;
    char* save_buffer;
             
    stCoSpec_t aSpec[1024];     
};           
```


####1.2.4.coctx_t的数据结构
位于`coctx.h`中。协程上下文结构，用于保存协程栈的地址和切换时保存寄存器的值。
```cpp
struct coctx_t      
{                   
#if defined(__i386__)  // 32位
    void *regs[ 8 ];
#else               
    void *regs[ 14 ];  // 64位，存放寄存器的信息，TODO：详细信息需要补充
#endif
	// stStackMem_t.stack_size
    size_t ss_size;  // 栈空间大小
    // stStackMem_t.stack_bp?
    char *ss_sp;  // 栈空间最低地址。之所以这么说，是因为栈的分配是由高地址到低地址。
                    
};                  
```

####1.2.5.stStackMem_t的数据结构
协程栈的数据结构位于`co_routine_inner.h`，如下：
```cpp
struct stStackMem_t
{        
    stCoRoutine_t* occupy_co;  // 共享栈上压着的上一个协程
    int stack_size;  // 栈的大小
    char* stack_bp; //stack_buffer + stack_size
    char* stack_buffer;  // 栈的buffer
         
};       
```
`stCoRoutine_t`的介绍在1.2.3中。

####1.2.6.stCoSpec_t的数据结构
位于`co_routine_inner.h`中。如下：
```c++
struct stCoSpec_t
{                
    void *value;
};       
```
目前还不太清楚这个结构时用来干嘛的，TODO：待补充。


###1.3.co_init_curr_thread_env()函数
初始化所在线程对应的协程环境函数。代码如下：
```cpp
void co_init_curr_thread_env()
{
	pid_t pid = GetPid();	// 获取当前线程id(tid)
	g_arrCoEnvPerThread[ pid ] = (stCoRoutineEnv_t*)calloc( 1,sizeof(stCoRoutineEnv_t) ); // 分配单个协程环境空间,也可以使用malloc
	stCoRoutineEnv_t *env = g_arrCoEnvPerThread[ pid ];

	env->iCallStackSize = 0;  // 初始化调用栈长度(0)
	struct stCoRoutine_t *self = co_create_env( env, NULL, NULL,NULL );  // 初始化协程(创造出协程)
	self->cIsMain = 1;  // 是否为主协程

	env->pending_co = NULL;  // TODO:这两个变量有什么作用
	env->occupy_co = NULL;

	coctx_init( &self->ctx );  // 初始化协程上下文环境

	env->pCallStack[ env->iCallStackSize++ ] = self;  // 将协程压栈

	stCoEpoll_t *ev = AllocEpoll();  //注册事件驱动的模块
	SetEpoll( env,ev );  // 设到协程环境中
}
```

####1.3.1.co_create_env()创造协程的函数
创造协程的函数，代码如下：
```cpp
// 将协程（入口函数为pfn的协程注册到）
struct stCoRoutine_t *co_create_env(
	stCoRoutineEnv_t * env, 
	const stCoRoutineAttr_t* attr, 
	pfn_co_routine_t pfn,
	void *arg )
{

	stCoRoutineAttr_t at;  // 栈的特征
	if( attr )
	{
		memcpy( &at,attr,sizeof(at) );  // 拷贝对象
	}
	if( at.stack_size <= 0 )
	{
		at.stack_size = 128 * 1024;  // 分配默认的栈大小（128kb）
	}
	else if( at.stack_size > 1024 * 1024 * 8 )
	{
		at.stack_size = 1024 * 1024 * 8;  // 最大为1024*1024*8（8mb）
	}

	// 少于1024，至少给它分配一个1024空间：1024 = 0x1000
	if( at.stack_size & 0xFFF ) 
	{
		at.stack_size &= ~0xFFF;
		at.stack_size += 0x1000;
	}

	stCoRoutine_t *lp = (stCoRoutine_t*)malloc( sizeof(stCoRoutine_t) );  // 分配协程结构空间
	
	memset( lp,0,(long)(sizeof(stCoRoutine_t))); 


	lp->env = env;
	lp->pfn = pfn;
	lp->arg = arg;

	stStackMem_t* stack_mem = NULL;
	if( at.share_stack )  // 共享栈模式
	{
		// 获得协程栈空间
		stack_mem = co_get_stackmem( at.share_stack);
		at.stack_size = at.share_stack->stack_size;
	}
	else  // 默认非共享栈模式
	{
		// 分配协程栈空间
		stack_mem = co_alloc_stackmem(at.stack_size);
	}
	lp->stack_mem = stack_mem;

	lp->ctx.ss_sp = stack_mem->stack_buffer;
	lp->ctx.ss_size = at.stack_size;

	lp->cStart = 0;
	lp->cEnd = 0;
	lp->cIsMain = 0;
	lp->cEnableSysHook = 0;
	lp->cIsShareStack = at.share_stack != NULL;

	lp->save_size = 0;
	lp->save_buffer = NULL;

	return lp;
}
```

#####1.3.1.1.stCoRoutineAttr_t数据结构
位于`co_routine.h`，如下：
```cpp
struct stCoRoutineAttr_t
{
	int stack_size;  // 栈空间
	stShareStack_t*  share_stack;  // 共享栈
	stCoRoutineAttr_t()
	{
		stack_size = 128 * 1024;  // 默认128*1024
		share_stack = NULL;
	}
}__attribute__ ((packed));
```
其中，`__attribute__((packed))`的含义是：
>`__attribute__` ((packed)) 的作用就是告诉编译器取消结构在编译过程中的优化对齐,按照实际占用字节数进行对齐，是GCC特有的语法。这个功能是跟操作系统没关系，跟编译器有关，gcc编译器不是紧凑模式的，我在windows下，用vc的编译器也不是紧凑的，用tc的编译器就是紧凑的。

#####1.3.1.2.co_get_stackmem函数
位于`co_routine.cpp`中，如下：
```cpp
static stStackMem_t* co_get_stackmem(
	stShareStack_t* share_stack)
{
	if (!share_stack)
	{
		return NULL;
	}
	int idx = share_stack->alloc_idx % share_stack->count;
	share_stack->alloc_idx++;

	return share_stack->stack_array[idx];
}
```
从共享栈中分配一个空间

#####1.3.1.3.co_alloc_stackmem函数
位于`co_routine.cpp`中，如下：
```cpp
/////////////////for copy stack //////////////////////////
stStackMem_t* co_alloc_stackmem(unsigned int stack_size)
{
	stStackMem_t* stack_mem = (stStackMem_t*)malloc(sizeof(stStackMem_t));
	stack_mem->occupy_co= NULL;
	stack_mem->stack_size = stack_size;
	stack_mem->stack_buffer = (char*)malloc(stack_size);
	stack_mem->stack_bp = stack_mem->stack_buffer + stack_size;  // 栈顶
	return stack_mem;
} 
```
没什么好说的，在堆栈中分配一个栈空间。共享栈的数据结构如下：
```cpp
struct stStackMem_t
{
	stCoRoutine_t* occupy_co;
	int stack_size;
	char* stack_bp; //stack_buffer + stack_size
	char* stack_buffer;

};

struct stShareStack_t
{
	unsigned int alloc_idx;
	int stack_size;
	int count;
	stStackMem_t** stack_array;
};
```

####1.3.2.coctx_init函数
位于`coctx.cpp`中，初始化协程上下文环境。如下：
```cpp
int coctx_init( coctx_t *ctx )
{
	memset( ctx,0,sizeof(*ctx));
	return 0;
}
```
也很简单，清0初始化。

####1.3.3.AllocEpoll函数
位于`co_routine.cpp`，如下：
```cpp
stCoEpoll_t *AllocEpoll()
{
	stCoEpoll_t *ctx = (stCoEpoll_t*)calloc( 1,sizeof(stCoEpoll_t) );  // 分配事件的数据结构

	ctx->iEpollFd = co_epoll_create( stCoEpoll_t::_EPOLL_SIZE );
	ctx->pTimeout = AllocTimeout( 60 * 1000 );
	
	ctx->pstActiveList = (stTimeoutItemLink_t*)calloc( 1,sizeof(stTimeoutItemLink_t) );
	ctx->pstTimeoutList = (stTimeoutItemLink_t*)calloc( 1,sizeof(stTimeoutItemLink_t) );
	return ctx;
}
```
也是分配空间的操作。

#####1.3.3.1.co_epoll_create函数
位于`co_epoll.cpp`中，代码如下：
```cpp
int co_epoll_create( int size )
{      
    return epoll_create( size );
}      
```
调用的是系统函数`epoll_create`。含义如下：
>  创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大。这个参数不同于select()中的第一个参数，给出最大监听的fd+1的值。需要注意的是，当创建好epoll句柄后，它就是会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。
>  该函数生成一个epoll专用的文件描述符。它其实是在内核申请一空间，用来存放你想关注的socket fd上是否发生以及发生了什么事件。size就是你在这个epoll fd上能关注的最大socket fd数。随你定好了。只要你有空间。

这里`stCoEpoll_t::_EPOLL_SIZE`为1024 * 10

####1.3.4.SetEpoll函数
位于`co_routine.cpp`，逻辑也很简单：
```cpp
void SetEpoll( stCoRoutineEnv_t *env,stCoEpoll_t *ev )
{
	env->pEpoll = ev;
}
```
把初始化的epoll事件注册到对应的协程环境中。

###1.4.co_eventloop函数
驱动事件循环的函数。调用的地方：
```cpp
int loop(void *)
{
	return 0;
}
co_eventloop(ev, loop, 0);
```
代码有点复杂：
```cpp
void co_eventloop( 
	stCoEpoll_t *ctx,
	pfn_co_eventloop_t pfn,
	void *arg )
{
	// co_epoll_res *result;
	// result未被初始化
	if( !ctx->result )
	{
		ctx->result =  co_epoll_res_alloc( stCoEpoll_t::_EPOLL_SIZE );
	}
	// 获取所有的事件列表
	co_epoll_res *result = ctx->result;


	// 死循环
	for(;;)
	{
		// ctx->iEpollFd由epoll_create创建
		// 等待就绪的事件列表
		// 因为定时器的最小精度是1ms,所以最多等待1ms
		int ret = co_epoll_wait( ctx->iEpollFd,result,stCoEpoll_t::_EPOLL_SIZE, 1 );

		stTimeoutItemLink_t *active = (ctx->pstActiveList);
		stTimeoutItemLink_t *timeout = (ctx->pstTimeoutList);

		// 初始化timeout
		memset( timeout,0,sizeof(stTimeoutItemLink_t) );
		// 遍历实际需要处理的事件数目
		for(int i=0;i<ret;i++)
		{
			// 时间轮盘
			stTimeoutItem_t *item = (stTimeoutItem_t*)result->events[i].data.ptr;
			// 就绪函数非空
			if( item->pfnPrepare )
			{
				// 执行就绪函数
				item->pfnPrepare( item,result->events[i],active );
			}
			else
			{
				// 就绪函数为空,追加
				// stTimeoutItem_t * item;
				// stTimeoutItemLink_t * active;
				// struct stTimeoutItemLink_t
				// {   
				//     stTimeoutItem_t *head;
				//     stTimeoutItem_t *tail;
				// };
				AddTail( active,item );
			}
		}

		// 获取当前时间(精确到毫秒)
		unsigned long long now = GetTickMS();

		// 获取所有到期的定时器
		TakeAllTimeout( ctx->pTimeout,now,timeout );

		stTimeoutItem_t *lp = timeout->head;
		while( lp )
		{
			//printf("raise timeout %p\n",lp);
			lp->bTimeout = true;
			lp = lp->pNext;
		}

		Join<stTimeoutItem_t,stTimeoutItemLink_t>( active,timeout );

		lp = active->head;
		while( lp )
		{

			// 去掉头结点
			PopHead<stTimeoutItem_t,stTimeoutItemLink_t>( active );
			// 时间轮由于超时引起的，但未过期
            if (lp->bTimeout && now < lp->ullExpireTime) 
			{
				int ret = AddTimeout(ctx->pTimeout, lp, now);
				if (!ret) 
				{
					lp->bTimeout = false;
					lp = active->head;
					continue;
				}
			}
			// 处理回调函数
			// 事件被激活或是超时的时候就调用。
			if( lp->pfnProcess )
			{
				lp->pfnProcess( lp );
			}

			lp = active->head;
		}
		// pfn——执行触发终止返回的函数
		if( pfn )
		{
			if( -1 == pfn( arg ) )
			{
				break;
			}
		}

	}
}
```

####1.4.1.co_epoll_res数据结构
位于`co_epoll.h`中，如下：
```cpp
struct co_epoll_res
{
	int size;  // todo:目前不清楚
	struct epoll_event *events; 
	struct kevent *eventlist;
};
```
未完待续
