# libco(4)
@(源码)

## 3.example_cond.cpp

这个例子的`main`函数比较简短：
```cpp
int main()
{
	// 新建stEnv_t对象
	stEnv_t* env = new stEnv_t;
	// 创建一个主协程
	env->cond = co_cond_alloc();

	// 定义消费者协程
	// 在当前主协程创建一个新的协程,并且切换到新的协程。
	stCoRoutine_t* consumer_routine;
	co_create(&consumer_routine, NULL, Consumer, env);
	// 挂起当前协程，切换到target协程
	co_resume(consumer_routine);

	// 定义生产者协程
	stCoRoutine_t* producer_routine;
	co_create(&producer_routine, NULL, Producer, env);
	co_resume(producer_routine);
	
	co_eventloop(co_get_epoll_ct(), NULL, NULL);
	return 0;
}
```

### 3.1.stEnv_t数据结构
位于`example_cond.cpp`，代码如下：
```cpp
struct stEnv_t
{
	stCoCond_t* cond;  // 协程条件变量
	queue<stTask_t*> task_queue;
};
```
我们再看下`stCoCond_t`的数据结构，位于`co_routine.cpp`中：
```cpp
struct stCoCond_t
{
	stCoCondItem_t *head;
	stCoCondItem_t *tail;
};
struct stCoCondItem_t 
{
	stCoCondItem_t *pPrev;
	stCoCondItem_t *pNext;
	stCoCond_t *pLink;

	stTimeoutItem_t timeout;
};

```
这里的`stTimeoutItem_t`就是之前提到过的时间轮的概念。
```cpp
struct stTimeoutItem_t
{

	enum
	{
		eMaxTimeout = 40 * 1000 //40s
	};
	stTimeoutItem_t *pPrev;
	stTimeoutItem_t *pNext;
	stTimeoutItemLink_t *pLink;

	unsigned long long ullExpireTime;

	OnPreparePfn_t pfnPrepare;
	OnProcessPfn_t pfnProcess;

	void *pArg; // routine 
	bool bTimeout;
};
```

### 3.2.co_cond_alloc函数
位于`co_routine.cpp`，代码如下：
```cpp
stCoCond_t *co_cond_alloc()
{
	return (stCoCond_t*)calloc( 1,sizeof(stCoCond_t) );
}
```
调用calloc在堆中分配数据结构。

### 3.3.stCoRoutine_t数据结构
位于`co_routine_inner.h`，代码如下：
```cpp
struct stCoRoutine_t
{
	stCoRoutineEnv_t *env;
	pfn_co_routine_t pfn;
	void *arg;
	coctx_t ctx;

	char cStart;
	char cEnd;
	char cIsMain;
	char cEnableSysHook;
	char cIsShareStack;

	void *pvEnv;

	//char sRunStack[ 1024 * 128 ];
	stStackMem_t* stack_mem;


	//save satck buffer while confilct on same stack_buffer;
	char* stack_sp; 
	unsigned int save_size;
	char* save_buffer;

	stCoSpec_t aSpec[1024];

};
```

### 3.4.co_create创建协程函数
位于`co_routine.cpp`，代码如下：
```cpp
// 传入Consumer函数
// co_create(&consumer_routine, NULL, Consumer, env);
int co_create( 
	stCoRoutine_t **ppco,
	const stCoRoutineAttr_t *attr,
	pfn_co_routine_t pfn,
	void *arg )
{
	if( !co_get_curr_thread_env() ) 
	{
		co_init_curr_thread_env();
	}
	// 再分配一个协程co
	stCoRoutine_t *co = co_create_env( co_get_curr_thread_env(), attr, pfn,arg );  // arg = stEnv_t env;
	*ppco = co;
	return 0;
}
```
其中，`co_get_curr_thread_env`和`co_init_curr_thread_env`和`co_create_env`分别在1.2和1.3和1.3.1均有定义。


### 3.5.co_resume恢复协程函数
位于`co_routine.cpp`，代码如下：
```cpp
// 挂起当前协程，让
void co_resume( stCoRoutine_t *co )
{
	// 找出对应协程的协程环境
	stCoRoutineEnv_t *env = co->env;
	// 调用递归栈-1，代表出栈
	// 当前栈顶协程
	stCoRoutine_t *lpCurrRoutine = env->pCallStack[ env->iCallStackSize - 1 ];
	// 如果协程还没有开始
	if( !co->cStart )
	{
		// 准备新的协程上下文
		coctx_make( &co->ctx,(coctx_pfn_t)CoRoutineFunc,co,0 );
		// 标记协程已经开始
		co->cStart = 1;
	}
	// 协程入栈
	env->pCallStack[ env->iCallStackSize++ ] = co;
	// 开始切换
	co_swap( lpCurrRoutine, co );
}
```

#### 3.5.1.coctx_make函数
位于`coctx.cpp`，代码如下：
```cpp
// 初始化coctx_t
#if defined(__i386__)
int coctx_init( coctx_t *ctx )
{
	memset( ctx,0,sizeof(*ctx));
	return 0;
}
int coctx_make( coctx_t *ctx,coctx_pfn_t pfn,const void *s,const void *s1 )
{
	//make room for coctx_param
	char *sp = ctx->ss_sp + ctx->ss_size - sizeof(coctx_param_t);
	sp = (char*)((unsigned long)sp & -16L);

	
	coctx_param_t* param = (coctx_param_t*)sp ;
	param->s1 = s;
	param->s2 = s1;

	memset(ctx->regs, 0, sizeof(ctx->regs));

	ctx->regs[ kESP ] = (char*)(sp) - sizeof(void*);
	ctx->regs[ kEIP ] = (char*)pfn;

	return 0;
}
#elif defined(__x86_64__)
int coctx_make( coctx_t *ctx,coctx_pfn_t pfn,const void *s,const void *s1 )
{
	// 栈底
	char *sp = ctx->ss_sp + ctx->ss_size;
	// GCC坚持一个x86编程指导方针，也就是一个函数使用的所有栈空间必须是16字节的整数倍。
	sp = (char*) ((unsigned long)sp & -16LL  );

	memset(ctx->regs, 0, sizeof(ctx->regs));

	// sp再往下偏移8个字节记录到regs[kRSP]中(此时sp%16==8)；
	ctx->regs[ kRSP ] = sp - 8;
	// 协程的入口函数地址
	ctx->regs[ kRETAddr] = (char*)pfn;
	// 函数参数
	ctx->regs[ kRDI ] = (char*)s;
	ctx->regs[ kRSI ] = (char*)s1;
	return 0;
}

int coctx_init( coctx_t *ctx )
{
	memset( ctx,0,sizeof(*ctx));
	return 0;
}

#endif
```
我们回忆下协程的结构：
```cpp
struct coctx_t
{
#if defined(__i386__)
	void *regs[ 8 ];
#else
	void *regs[ 14 ];
#endif
	size_t ss_size;
	char *ss_sp;
	
};
```

由此，我们可以看出来：
1. libco用栈(`stCoRoutine_t *pCallStack[128];`一个协程结构体数组)来管理协程。栈顶的协程是当前运行的协程。
2. 第一次创建协程的时候去检查是否有这个全局的协程环境变量，然后创建，随之也创建了主协程。每次可以通过当前进程的`pid`获取这个协程环境变量。
3. 一个进程被剖分为多个协程。然后可以在每个协程之间的切换。
4. 每个协程拥有的东西，无非是自己的调用栈信息和各种上下文状态（当前寄存器信息）。这些信息保存在协程里面的`coctx_t`。

##### 3.5.2.一个简单函数调用的过程
由于上面讲到的涉及协程切换，所以我们先得补一下函数调用的一些基础。看一下这个例子：
```cpp
#include<stdio.h>
int foo(int a, int b)
{
    return a+b;
}
int main(void)
{
    foo(2, 5);//函数调用栈发生在这里
    return 0;
}
```

1. 首先程序进入到`main`函数，分配一段新的栈帧。这个时候gcc会更新`%rbp`和`%rsp`两个寄存器的值。`%rbp`是基址指针寄存器，指向当前栈帧的底部位置。每次调用函数开始都会更新这个寄存器，代表开辟了一个新的栈帧。`%rsp`是栈指针寄存器，更改值，指向当前栈帧的顶部位置。如图：（进入main函数，但没有开始调用foo函数）
**先补一张基础**：
![4-1.png](https://github.com/sysublackbear/libco_source_study/blob/master/libco_pic/4-1.png)
2. 然后到了`foo(2, 5)`这一句话，`%rsi`会保存参数5，`%rdi`会保存参数2。函数调用过程的参数是从右到左放入寄存器，过多的参数会压栈。同时，`%rip`指令寄存器会，会存放下一个要执行的指令地址。如图所示：
![4-2.png](https://github.com/sysublackbear/libco_source_study/blob/master/libco_pic/4-2.png)
3. 然后执行`callq`指令。`callq`只做两件事情：`push %rip`(a.把函数调用的下一个指令地址入栈，而这个地址就是函数的返回地址；b.把%rip指向新调用的函数的入口地址)**注意：将rip寄存器指向最新调用的函数入口地址，意味着函数开始调度执行！！！！**；
4. 然后类似1，不过先把上一个栈帧的底部地址压到当前栈帧。`%rbp`和`%rsp`寄存器会更新，指向新的栈帧和栈顶。如图所示：
![4-3.png](https://github.com/sysublackbear/libco_source_study/blob/master/libco_pic/4-3.png)
5. 函数执行完，先出栈，栈顶元素为上一个栈帧的rbp地址，恢复上一个栈帧环境到寄存器`%rbp`和`%rsp`。然后再遍历过所有的临时，局部变量和多余参数列表，最后再出栈原来的上一个调用函数的下一个执行(`return 0`)。达到恢复上一个栈的效果。
![4-4.png](https://github.com/sysublackbear/libco_source_study/blob/master/libco_pic/4-4.png)

以下是一些附录资料：
**gcc对寄存器的使用规则**
+ **指令寄存器: %rip**
+ **栈指针: %rsp**
+ 函数返回值: %rax
+ 函数参数: %rdi, %rsi, %rdx, %rcx, %r8, %r9(依次对应被调函数的第1-6的参数，当函数参数超过6个时通过压栈方式传参)
+ 数据存储: %rbx, %rbp, %r12, %r13, %14, %15 遵循被调用者使用规则, %r10, %r11 遵循调用者使用规则
+ 浮点寄存器: %mmx(0-7) %xmm(0-15) %mxcsr

被调用者使用规则(Callee-Saved): **简单说就是使用之前需要先备份这些寄存器的值，使用完毕后恢复原本寄存器的值**
调用者使用规则(Caller-Saved): **简单说就是调用子函数之前需要先备份这些寄存器的值，因为被调函数可能需要用到**

**C/C++栈、栈帧与函数调用**
栈一般来说又称为堆栈，由编译器自动分配和释放，行为类似数据结构中的栈(FILO)，堆栈主要有三个用途：

为函数内部定义的非静态局部变量提供存储空间；
记录函数调用过程中的相关信息，包括函数的参数（当函数参数超过6个时通过压栈传参），返回地址和一些寄存器值；
用于暂存长算术表达式部分计算结果或alloca()函数分配的栈内内存。
当发生函数调用时：

1. 调用函数从右到左传参(第1-6个参数通过寄存器传递，超出6个的参数通过压栈传递)；
2. 调用函数使用call指令调用被调函数(call指令相当于pushq %rip + jmpq)；
3. 进入到被调函数，被调函数会先保存调用函数的栈底地址(pushq %rbp)，然后用rbp记录当前函数的栈底地址(movq %rsp, %rbp)；
4. 被调函数使用%ebp下方的位置来存放被调函数中的局部变量和临时变量，当需要再调用函数时继续按照1递归进行；
5. 当被调函数执行完毕时，先使用leave指令恢复调用函数的栈底和栈顶地址(leave指令等价于movq %rbp %rsp + popq %rbp)，最后通过retq指令返回(retq指令相当于popq %rip + jmpq)。

**栈帧**指的是在函数调用过程中每个函数所占用的一段连续栈空间(从栈底%rbp到栈顶%rsp)。

函数调用过程中，调用栈布局大致如下：
```cpp
high ---------->  |上一个栈帧    |
                --------------------------------------------------------
                  |上一个栈帧的ebp|
                  |局部、临时变量 |        调用函数栈帧
                  |参数          |
                  |ret func addr |
                ---------------------------------------------------------
                  |上一个栈帧的ebp|
                  |局部、临时变量 |        被调函数栈帧
                  |参数          |
                  |ret func addr |
                ---------------------------------------------------------
low  ---------->  |下一个栈帧     |
```
函数局部变量总是通过%ebp减去偏移量来访问，而函数参数总是通过%ebp增加偏移量来访问。

##### 3.5.3.CoRoutineFunc函数
位于`co_routine.cpp`，代码如下：
```cpp
coctx_make( &co->ctx,(coctx_pfn_t)CoRoutineFunc,co,0 );
static int CoRoutineFunc( stCoRoutine_t *co,void * )
{
	if( co->pfn )
	{
		// 调用协程的入口函数
		co->pfn( co->arg );
	}
	// 标记协程已经执行完毕
	co->cEnd = 1;

	stCoRoutineEnv_t *env = co->env;

	co_yield_env( env );

	return 0;
}
```
没什么好讲的，逻辑比较简单。

##### 3.5.4.co_swap函数
位于`co_routine.cpp`，代码如下：
```cpp
// co_swap( lpCurrRoutine, co );
// 从curr切换到pending_co
void co_swap(stCoRoutine_t* curr, stCoRoutine_t* pending_co)
{
	// 获取当前的协程环境
 	stCoRoutineEnv_t* env = co_get_curr_thread_env();

	// 这里是大神雷哥说的：
	// 因为后面需要知道当前栈顶来复制栈内容
	// 但是我们不知道当前函数的栈顶地址是多少
	// 这里我们用到了一个取巧（trickly）的实现
	// 就是新建一个局部变量c，然后取c的地址
	// 获取到当前栈顶的地址,记为stack_sp
	//get curr stack sp
	char c;
	curr->stack_sp= &c;

	if (!pending_co->cIsShareStack)
	{
		// 不是共享栈模式，不需要保存占用共享栈的协程 
		env->pending_co = NULL;
		env->occupy_co = NULL;
	}
	else 
	{
		// 待切换协程
		// 保存新协程的地址到线程的协程上下文环境
		env->pending_co = pending_co;
		//get last occupy co on the same stack mem
		// 从新协程的共享栈中获取前一个占用该共享栈的协程地址
		stCoRoutine_t* occupy_co = pending_co->stack_mem->occupy_co;
		//set pending co to occupy thest stack mem;
		// 更新占用共享栈的协程地址为新协程
		pending_co->stack_mem->occupy_co = pending_co;

		// 保存前一个占用新协程共享栈的协程地址到线程上下文occupy_co
		env->occupy_co = occupy_co;
		// 如果前一个协程非空而且不等于新的协程
		if (occupy_co && occupy_co != pending_co)
		{
			// 复制前一个协程到stack_buffer中
			save_stack_buffer(occupy_co);
		}
	}

	//swap context
	// 交换协程的上下文环境。
	coctx_swap(&(curr->ctx),&(pending_co->ctx) );

	//stack buffer may be overwrite, so get again;
	// 共享栈模式下，由于协程切换之后栈空间已经被新的协程覆盖，所以重新获取
	stCoRoutineEnv_t* curr_env = co_get_curr_thread_env();
	stCoRoutine_t* update_occupy_co =  curr_env->occupy_co;
	stCoRoutine_t* update_pending_co = curr_env->pending_co;
	
	if (update_occupy_co && update_pending_co && update_occupy_co != update_pending_co)
	{
		//resume stack buffer
		if (update_pending_co->save_buffer && update_pending_co->save_size > 0)
		{
			// 将save_buffer的内容拷贝到栈顶
			memcpy(update_pending_co->stack_sp, update_pending_co->save_buffer, update_pending_co->save_size);
		}
	}
}
```

##### 3.5.5.save_stack_buffer函数
位于`co_routine.cpp`，代码如下：
```cpp
// save_stack_buffer(occupy_co);
// 复制前一个协程到stack_buffer
void save_stack_buffer(stCoRoutine_t* occupy_co)
{
	// copy out
	// 取出共享栈
	stStackMem_t* stack_mem = occupy_co->stack_mem;
	// 读取出协程栈的栈长度（栈顶-栈底）
	int len = stack_mem->stack_bp - occupy_co->stack_sp;

	if (occupy_co->save_buffer)
	{
		// 清空协程的buffer空间
		free(occupy_co->save_buffer), occupy_co->save_buffer = NULL;
	}

	// 重新分配
	occupy_co->save_buffer = (char*)malloc(len); //malloc buf;
	occupy_co->save_size = len;

	memcpy(occupy_co->save_buffer, occupy_co->stack_sp, len);
	// 这里函数的作用仅仅是分配一段协程栈的空间。
}
```

##### 3.5.6.coctx_swap函数
位于`coctx.cpp`中，代码如下：
```cpp
// coctx_swap(&(curr->ctx),&(pending_co->ctx) );
//64 bit
extern "C"
{
	extern void coctx_swap( coctx_t *,coctx_t* ) asm("coctx_swap");
};

.globl coctx_swap
#if !defined( __APPLE__ ) && !defined( __FreeBSD__ )
.type  coctx_swap, @function
#endif
coctx_swap:

#if defined(__i386__)
	leal 4(%esp), %eax //sp 
	movl 4(%esp), %esp 
	leal 32(%esp), %esp //parm a : &regs[7] + sizeof(void*)

	pushl %eax //esp ->parm a 

	pushl %ebp
	pushl %esi
	pushl %edi
	pushl %edx
	pushl %ecx
	pushl %ebx
	pushl -4(%eax)

	
	movl 4(%eax), %esp //parm b -> &regs[0]

	popl %eax  //ret func addr
	popl %ebx  
	popl %ecx
	popl %edx
	popl %edi
	popl %esi
	popl %ebp
	popl %esp
	pushl %eax //set ret func addr

	xorl %eax, %eax
	ret

#elif defined(__x86_64__)
	leaq 8(%rsp),%rax  // 让rax指向当前协程栈帧的栈顶，rsp为栈指针
	leaq 112(%rdi),%rsp  // rsp指向rdi+112，由于16字节对齐，112/16=7来指向regs[14],超出coctx_t，因为栈是高地址到低地址增长的
	// 每个coctx数组压栈
	pushq %rax  // 将寄存器的变量逐个压栈，想到赋值到regs数组里面
    pushq %rbx
	pushq %rbx
	pushq %rcx
	pushq %rdx

	pushq -8(%rax) //当前协程的返回地址

	pushq %rsi
	pushq %rdi
	pushq %rbp
	pushq %r8
	pushq %r9
	pushq %r12
	pushq %r13
	pushq %r14
	pushq %r15
	
	movq %rsi, %rsp  // %rsp指向切入coctx_t的regs[0]
	// 逐个出栈，放到各个寄存器里面
	popq %r15
	popq %r14
	popq %r13
	popq %r12
	popq %r9
	popq %r8
	popq %rbp
	popq %rdi
	popq %rsi
	popq %rax //ret func addr
	popq %rdx
	popq %rcx
	popq %rbx
	popq %rsp
	pushq %rax  // 将上一个栈帧的栈顶塞到栈里，类似函数调用，方便函数返回
	
	xorl %eax, %eax  // 将eax清0
	ret  // 协程入口函数地址保存到rip寄存器中，注意！！！正如上面说的，相当于待切换协程开始调度了。
#endif
```
调用汇编代码，注意到`coctx_swap`的函数签名包含两个参数的分别是切出和切入的协程指针。根据寄存器使用规则，在进入`coctx_swap`时，`%rdi`保存着切出的协程指针，`%rsi`保存着切入的协程指针。
> 函数参数: %rdi, %rsi, %rdx, %rcx, %r8, %r9(依次对应被调函数的第1-6的参数，当函数参数超过6个时通过压栈方式传参)

协程切换无非需要干两个事情：
1. 保存切出协程的上下文；
2. 恢复切入协程的上下文。

注意：但是程序中不能直接使用`coctx_t`，这是因为`coctx_t`其实只是提供了协程切换的能力。如果直接使用，当协程入口函数执行完毕的时候，程序会发生未定义的行为。

因此，`libco`定义了两个数据结构`stCoRoutine_t`和`stCoRoutineEnv_t`，提供使用。
其中，`stCoRoutineEnv_t`是`stCoRoutine_t`的运行环境，保存了协程调用栈，最多可以递归调用128层，调用栈栈顶指向当前正在运行的协程。

这个函数是汇编实现的：`rsi`指向新协程的`coctx_t`，`rdi`指向当前协程的`coctx_t`。下面主要做了两件事情：把当前协程的关键寄存器存到`rdi`指向的`coctx_t`里面的`regis[14]`，然后把刚刚准备的`rsi`指向`coctx_t`的`regis[14]`信息替换当前关键寄存器。重点是改变来`rsp`和`rip`两个寄存器，从而完成协程切换。


