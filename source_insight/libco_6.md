  
# libco(6)

@(源码)

##### 3.7.4.1.PollEvent2Epoll函数
位于`co_routine.cpp`，代码如下：
```cpp
static uint32_t PollEvent2Epoll( short events )
{
	uint32_t e = 0;	
	if( events & POLLIN ) 	e |= EPOLLIN;
	if( events & POLLOUT )  e |= EPOLLOUT;
	if( events & POLLHUP ) 	e |= EPOLLHUP;
	if( events & POLLERR )	e |= EPOLLERR;
	if( events & POLLRDNORM ) e |= EPOLLRDNORM;
	if( events & POLLWRNORM ) e |= EPOLLWRNORM;
	return e;
}
```
events包括要监视的事件，看代码本质上就是将poll对应的标志位转化为epoll的标志位。

##### 3.7.4.2.co_epoll_ctl函数
位于`co_epoll.cpp`，代码如下：
```cpp
int	co_epoll_ctl( int epfd,int op,int fd,struct epoll_event * ev )
{
	return epoll_ctl( epfd,op,fd,ev );
}
```
epoll的事件注册函数，它不同与select()是在监听事件时告诉内核要监听什么类型的事件epoll的事件注册函数，它不同与select()是在监听事件时告诉内核要监听什么类型的事件，而是在这里先注册要监听的事件类型。第一个参数是epoll_create()的返回值，第二个参数表示动作，用三个宏来表示：
+ EPOLL_CTL_ADD：注册新的fd到epfd中；
+ EPOLL_CTL_MOD：修改已经注册的fd的监听事件；
+ EPOLL_CTL_DEL：从epfd中删除一个fd；
第三个参数是需要监听的fd，第四个参数是告诉内核需要监听什么事，struct epoll_event结构如下：
```cpp
struct epoll_event {
  __uint32_t events;  /* Epoll events */
  epoll_data_t data;  /* User data variable */
};
```

##### 3.7.4.3.OnPollProcessEvent函数
位于`co_routine.cpp`，代码如下：
```cpp
void OnPollProcessEvent( stTimeoutItem_t * ap )
{
	stCoRoutine_t *co = (stCoRoutine_t*)ap->pArg;
	co_resume( co );
}
```

##### 3.7.4.4.OnPollPreparePfn函数
位于`co_routine.cpp`，代码如下：
```cpp
// OnPollPreparePfn( item,result->events[i],active )
void OnPollPreparePfn( stTimeoutItem_t * ap,struct epoll_event &e,stTimeoutItemLink_t *active )
{
	stPollItem_t *lp = (stPollItem_t *)ap;
	lp->pSelf->revents = EpollEvent2Poll( e.events );


	stPoll_t *pPoll = lp->pPoll;
	pPoll->iRaiseCnt++;  // 唤醒协程数+1

	if( !pPoll->iAllEventDetach )
	{
		pPoll->iAllEventDetach = 1;

		RemoveFromLink<stTimeoutItem_t,stTimeoutItemLink_t>( pPoll );

		// 这里相当于无论你有没有PreparePfn函数，你都需要在执行完毕之后放到active列表中
		AddTail( active,pPoll );

	}
}

// 将epoll的信号转化为poll信号
static short EpollEvent2Poll( uint32_t events )
{
	short e = 0;	
	if( events & EPOLLIN ) 	e |= POLLIN;
	if( events & EPOLLOUT ) e |= POLLOUT;
	if( events & EPOLLHUP ) e |= POLLHUP;
	if( events & EPOLLERR ) e |= POLLERR;
	if( events & EPOLLRDNORM ) e |= POLLRDNORM;
	if( events & EPOLLWRNORM ) e |= POLLWRNORM;
	return e;
}
```

### 3.8.co_eventloop函数
位于`co_routine.cpp`，含义解释详见1.4。
调用处：`co_eventloop(co_get_epoll_ct(), NULL, NULL);`
本质就是从时间轮中找出就绪或者超时了需要触发的事件。然后逐个调用回调函数。我们上面可以看到：
+ Consumer的回调函数是：`OnSignalProcessEvent`。目的是挂起当前协程，切回Consumer协程。
+ Producer的回调函数是：`OnPollProcessEvent`。目的也是挂起当前协程，切回Producer协程。

### 3.9.example_cond综述
整个过程如下：
1. 整个进程主要分为三个协程：主协程，Consumer协程，Producer协程；
2. 首先在主协程中创建了Consumer协程，然后调度到Consumer协程；
3. Consumer协程这个时候检测到task_queue为空，随后调用`co_cond_timewait(env->cond, -1)`，（这里如果不是timeout为-1的话，会把对应的对应的时间轮单项放入到env->pEpoll->pTimeout列表上的。）将新建的psi(stCoCondItem_t)加入到env->cond中；然后调用`co_yield_ct()`切回到主协程；
4. 主协程随后创建了Producer协程，然后调度到Producer协程；
5. Producer协程新建task，塞到task_queue队列中，随后调用`co_cond_signal`。`co_cond_signal`里面首先先从env->cond列表中pop出头元素出来。随后从env->pEpoll->pTimeout中去除上面的时间轮单项（如果有），然后把对应的时间单项放到env->pEpoll->pstActiveList中。
6. 随后Producer协程调用`poll(NULL, 0, 1000);`，由于nfds和fds都为空，所以该函数本质上就是协程sleep 1s的时间。
7. poll函数内部逻辑，如果timeout为0，代表不额外阻塞任何时间，直接调用系统的poll函数；否则，调用hook掉的poll函数。如果nfds大于0，逐个将fds到系统epoll服务注册。阻塞时间的功能(timeout)交给用户态时间轮进行管理，即将整个poll对象放入到时间轮上（`int ret = AddTimeout( ctx->pTimeout,&arg,now ); `）
8. 随后Producer协程挂起，调度回主协程；
9. 主协程调用`co_eventloop`函数，通过调用`co_epoll_wait`，等待内核返回可调度的事件。然后针对每个epoll，逐个执行每个fd的PreparePfn函数。打上标记，唤醒poll的事件。然后加入到active列表中。随后扫描出就绪和超时触发的事件列表，逐个调用它们的回调函数。这里分了两种可能性。
+ 触发到Consumer协程，回到`RemoveFromLink<stCoCondItem_t,stCoCond_t>( psi );`(这里Producer协程也会做，但是这里做个兜底)。然后回到死循环，不断去队列拉取元素进行处理。如果队列为空，在stCoCond_t挂载一个变量，切回主协程。然后主协程又继续陷入co_eventloop。
+ 触发到Producer协程，回到`co_poll_inner`函数，回收所有想epoll注册的fds。然后返回死循环。然后再重新新建task，塞入到队列中，然后再调用`co_cond_signal`，接受信号，然后再调用`poll`函数阻塞1s。再重新挂起Producer协程，切回主协程，主协程继续陷入co_eventloop，如此循环。

**如图所示：**
![6-1.png](https://github.com/sysublackbear/libco_source_study/blob/master/libco_pic/6-1.png)
