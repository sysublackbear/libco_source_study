#libco(7)
@(源码)

##4.example_copystack.cpp

先看一下这个例子的`main`函数：
```cpp
int main()
{
	stShareStack_t* share_stack= co_alloc_sharestack(1, 1024 * 128);  // 128kb
	// 定义协程的栈特征数据结构
	stCoRoutineAttr_t attr;
	attr.stack_size = 0;
	attr.share_stack = share_stack;

	// 初始化两个协程
	stCoRoutine_t* co[2];
	int routineid[2];
	for (int i = 0; i < 2; i++)
	{
		routineid[i] = i;
		// 创建两个协程
		co_create(&co[i], &attr, RoutineFunc, routineid + i);
		// 切到co[i]这个协程
		co_resume(co[i]);
	}
	co_eventloop(co_get_epoll_ct(), NULL, NULL);
	return 0;
}
```

###4.1.共享栈的数据结构
位于`co_routine_inner.h`，代码如下：
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

###4.2.co_alloc_sharestack函数
分配共享栈的函数，位于`co_routine.cpp`，代码如下：
```cpp
// stShareStack_t* share_stack= co_alloc_sharestack(1, 1024 * 128);
stShareStack_t* co_alloc_sharestack(int count, int stack_size)
{
	stShareStack_t* share_stack = (stShareStack_t*)malloc(sizeof(stShareStack_t));
	share_stack->alloc_idx = 0;
	share_stack->stack_size = stack_size;

	//alloc stack array
	share_stack->count = count;
	stStackMem_t** stack_array = (stStackMem_t**)calloc(count, sizeof(stStackMem_t*));
	for (int i = 0; i < count; i++)
	{
		stack_array[i] = co_alloc_stackmem(stack_size);
	}
	share_stack->stack_array = stack_array;
	return share_stack;
}

/////////////////for copy stack //////////////////////////
stStackMem_t* co_alloc_stackmem(unsigned int stack_size)
{
	stStackMem_t* stack_mem = (stStackMem_t*)malloc(sizeof(stStackMem_t));
	stack_mem->occupy_co= NULL;
	stack_mem->stack_size = stack_size;
	stack_mem->stack_buffer = (char*)malloc(stack_size);
	stack_mem->stack_bp = stack_mem->stack_buffer + stack_size;
	return stack_mem;
}
```


###4.3.RoutineFunc函数
位于`example_copystack.cpp`，代码如下：
```cpp
void* RoutineFunc(void* args)
{
	co_enable_hook_sys();  // 打开hook系统调用开关
	// 获取协程id routineid[i] = i;
	int* routineid = (int*)args;
	while (true)
	{
		char sBuff[128];
		// 在栈中定义一个128字节的数组
		sprintf(sBuff, "from routineid %d stack addr %p\n", *routineid, sBuff);

		printf("%s", sBuff);
		poll(NULL, 0, 1000); //sleep 1s
	}
	return NULL;
}
```

###4.4.综述
这里例子比较简单，纯粹是为了说明，在共享栈模式下，多个协程共享同一份内存空间。这个demo输出的如下：
![Alt text](./1530428558355.png)
我们可以看到多个协程初始化数组的地址空间是一样的。

至于有人会问，为什么会在循环里面，每次定义的数组的地址空间能够保持一致。这里涉及编译器的优化问题，编译器不会因为循环而反复分配变量，数组的空间会一直保持。
