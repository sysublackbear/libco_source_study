# libco(12)
@(源码)

## 9.example_specific.cpp

这个例子篇幅还是比较小的，代码如下：
```cpp
int main()
{
	stRoutineArgs_t args[10];
	for (int i = 0; i < 10; i++)
	{
		// 起10个协程
		args[i].routine_id = i;
		co_create(&args[i].co, NULL, RoutineFunc, (void*)&args[i]);
		co_resume(args[i].co);
	}
	co_eventloop(co_get_epoll_ct(), NULL, NULL);
	return 0;
}
```
所以看所有的核心都在`RoutineFunc`里面了。


### 9.1.RoutineFunc函数
位于`example_specific.cpp`，代码如下：
```cpp
CO_ROUTINE_SPECIFIC(stRoutineSpecificData_t, __routine);

void* RoutineFunc(void* args)
{
	co_enable_hook_sys();
	// 协程入口函数参数
	stRoutineArgs_t* routine_args = (stRoutineArgs_t*)args;
	// __routine的idx设置成协程id[0-9]
	__routine->idx = routine_args->routine_id;
	while (true)
	{
		printf("%s:%d routine specific data idx %d\n", __func__, __LINE__, __routine->idx);
		// sleep 1s
		poll(NULL, 0, 1000);
	}
	return NULL;
}
```

### 9.2.CO_ROUTINE_SPECIFIC函数
这个宏比较复杂，位于`co_routine_specific.h`。
```cpp
// CO_ROUTINE_SPECIFIC(stRoutineSpecificData_t, __routine);
#define CO_ROUTINE_SPECIFIC( name,y ) \
\
static pthread_once_t _routine_once_##name = PTHREAD_ONCE_INIT;  \
static pthread_key_t _routine_key_##name;\
static int _routine_init_##name = 0;\
static void _routine_make_key_##name() \
{\
 	(void) pthread_key_create(&_routine_key_##name, NULL); \
}\
template <class T>\
class clsRoutineData_routine_##name\
{\
public:\
	inline T *operator->()\
	{\
		if( !_routine_init_##name ) \
		{\
			pthread_once( &_routine_once_##name,_routine_make_key_##name );\
			_routine_init_##name = 1;\
		}\
		T* p = (T*)co_getspecific( _routine_key_##name );\
		if( !p )\
		{\
			p = (T*)calloc(1,sizeof( T ));\
			int ret = co_setspecific( _routine_key_##name,p) ;\
            if ( ret )\
            {\
                if ( p )\
                {\
                    free(p);\
                    p = NULL;\
                }\
            }\
		}\
		return p;\
	}\
};\
\
static clsRoutineData_routine_##name<name> y;
```
对应的翻译代码为：
```cpp
static pthread_once_t _routine_once_stRoutineSpecificData_t = PTHREAD_ONCE_INIT;  
// 本函数使用初值为PTHREAD_ONCE_INIT的once_control变量保证init_routine()函数在本进程执行序列中仅执行一次。
static pthread_key_t _routine_key_stRoutineSpecificData_t;  
// 线程自身的全局变量
static int _routine_init_stRoutineSpecificData_t = 0;
static void _routine_make_key_stRoutineSpecificData_t()
{
	// 创建线程私有数据
 	(void) pthread_key_create(&_routine_key_stRoutineSpecificData_t, NULL); 
}
template <class T>
class clsRoutineData_routine_stRoutineSpecificData_t
{
public:
	inline T *operator->()
	{
		if( !_routine_init_stRoutineSpecificData_t ) 
		{
			// 类未初始化
			pthread_once( 
		&_routine_once_stRoutineSpecificData_t,
		_routine_make_key_stRoutineSpecificData_t );
			// pthread_once：仅初始化一次
			// _routine_once_stRoutineSpecificData_t:one control变量
			// _routine_make_key_stRoutineSpecificData_t：初始化函数
			// 功能：本函数使用初值为PTHREAD_ONCE_INIT的once_control变量保证init_routine()函数在本进程执行序列中仅执行一次。
			// 在多线程编程环境下，尽管pthread_once()调用会出现在多个线程中，init_routine()函数仅执行一次，究竟在哪个线程中执行是不定的，是由内核调度来决定。
			_routine_init_stRoutineSpecificData_t = 1;  // 打上初始化标记
		}
		T* p = (T*)co_getspecific( _routine_key_stRoutineSpecificData_t );
		if( !p )
		{
			// 读不到
			p = (T*)calloc(1,sizeof( T ));
			// 创建协程/线程私有数据
			int ret = co_setspecific( 
				_routine_key_stRoutineSpecificData_t,
				p) ;
            if ( ret )
            {
                if ( p )
                {
                    free(p);
                    p = NULL;
                }
            }
		}
		return p;
	}
};
static clsRoutineData_routine_stRoutineSpecificData_t<stRoutineSpecificData_t> __routine;  // 创建这样的一个静态类

struct stRoutineSpecificData_t
{
	int idx;
};
```


### 9.2.1.pthread_once_t：一次性初始化控制变量
> 有时候我们需要对一些posix变量只进行一次初始化，如线程键（我下面会讲到）。如果我们进行多次初始化程序就会出现错误。
> 在传统的顺序编程中，一次性初始化经常通过使用布尔变量来管理。控制变量被静态初始化为0，而任何依赖于初始化的代码都能测试该变量。如果变量值仍然为0，则它能实行初始化，然后将变量置为1。以后检查的代码将跳过初始化。
> 但是在多线程程序设计中，事情就变的复杂的多。如果多个线程并发地执行初始化序列代码，可能有2个线程发现控制变量为0，并且都实行初始化，而该过程本该仅仅执行一次。
> 如果我们需要对一个posix变量静态的初始化，可使用的方法是用一个互斥量对该变量的初始话进行控制。但有时候我们需要对该变量进行动态初始化，pthread_once就会方便的多。 

函数原形：
```cpp
pthread_once_t once_control=PTHREAD_ONCE_INIT;
int pthread_once(pthread_once_t *once_control,void(*init_routine)(void));
```
参数：
+ once_control         控制变量
+ init_routine         初始化函数

返回值：
+ 若成功返回0，若失败返回错误编号
+ 类型为`pthread_once_t`的变量是一个控制变量。控制变量必须使用PTHREAD_ONCE_INIT宏静态地初始化。

`pthread_once`函数首先检查控制变量，判断是否已经完成初始化，如果完成就简单地返回；否则，pthread_once调用初始化函数，并且记录下初始化被完成。如果在一个线程初始时，另外的线程调用`pthread_once`，则调用线程等待，直到那个现成完成初始话返回。


### 9.2.2.pthread_key_t介绍
`pthread_key_t`无论是哪一个线程创建，其他所有的线程都是可见的，即一个进程中只需`phread_key_create()`一次。看似是全局变量，然而全局的只是key值，对于不同的线程对应的value值是不同的(通过`pthread_setspcific()`和`pthread_getspecific()`设置)。 

### 9.2.3.pthread_key_create介绍
函数 `pthread_key_create()` 用来创建线程私有数据。该函数从 TSD 池中分配一项，将其地址值赋给 key 供以后访问使用。第 2 个参数是一个销毁函数，它是可选的，可以为 NULL，为 NULL 时，则系统调用默认的销毁函数进行相关的数据注销。如果不为空，则在线程退出时(调用 `pthread_exit()` 函数)时将以 key 锁关联的数据作为参数调用它，以释放分配的缓冲区，或是关闭文件流等。

不论哪个线程调用了 `pthread_key_create()`，所创建的 key 都是所有线程可以访问的，但各个线程可以根据自己的需要往 key 中填入不同的值，相当于提供了一个同名而不同值的全局变量(这个全局变量相对于拥有这个变量的线程来说)。

### 9.2.4.co_getspecific函数
位于`co_routine.cpp`，代码如下：
```cpp
void *co_getspecific(pthread_key_t key)
{
	// 获取当前协程
	stCoRoutine_t *co = GetCurrThreadCo();
	if( !co || co->cIsMain )
	{
		// 协程未初始化或者是主协程
		// 读取线程的私有数据
		return pthread_getspecific( key );
	}
	return co->aSpec[ key ].value;
}
```

`pthread_getspecific`函数：获取线程私有变量的函数

### 9.2.5.co_setspecific函数
位于`co_routine.cpp`，代码如下：
```cpp
int co_setspecific(pthread_key_t key, const void *value)
{
	stCoRoutine_t *co = GetCurrThreadCo();
	if( !co || co->cIsMain )
	{
		return pthread_setspecific( key,value );
	}
	co->aSpec[ key ].value = (void*)value;
	return 0;
}
```

### 9.3.综述
这个例子跟`example_setenv.cpp`类似，类似于写环境变量，这里写的线程/协程私有空间（TLS）。读起来还是比较通俗。输出结果如下：
![12-1.png](https://github.com/sysublackbear/libco_source_study/blob/master/libco_pic/12-1.png)
