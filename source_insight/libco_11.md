#libco(11)
@(源码)

##8.example_setenv.cpp

源码篇幅有点小，代码如下：
```cpp
const char* CGI_ENV_HOOK_LIST [] = 
{
	"CGINAME",
};
struct stRoutineArgs_t
{
	int iRoutineID;
};

int main(int argc, char* argv[])
{
	co_set_env_list(CGI_ENV_HOOK_LIST, sizeof(CGI_ENV_HOOK_LIST) / sizeof(char*));
	stRoutineArgs_t  args[3];
	for (int i = 0; i < 3; i++)
	{
		// 创建协程
		stCoRoutine_t* co = NULL;
		args[i].iRoutineID = i;
		co_create(&co, NULL, RoutineFunc, &args[i]);
		co_resume(co);
	}
	co_eventloop(co_get_epoll_ct(), NULL, NULL);
	return 0;
}
```

###8.1.co_set_env_list函数
位于`co_hook_sys_call.cpp`，代码如下：
```cpp
static stCoSysEnvArr_t g_co_sysenv = { 0 };

// co_set_env_list(CGI_ENV_HOOK_LIST, sizeof(CGI_ENV_HOOK_LIST) / sizeof(char*));
void co_set_env_list( const char *name[],size_t cnt)
{
	if( g_co_sysenv.data )
	{
		return ;
	}
	g_co_sysenv.data = (stCoSysEnv_t*)calloc( 1,sizeof(stCoSysEnv_t) * cnt  );

	for(size_t i=0;i<cnt;i++)
	{
		if( name[i] && name[i][0] )
		{
			// 说 明：strdup不是标准的c函数。strdup()在内部调用了malloc()为变量分配内存，不需要使用返回的字符串时，需要用free()释放相应的内存空间，否则会造成内存泄漏。
			g_co_sysenv.data[ g_co_sysenv.cnt++ ].name = strdup( name[i] );
		}
	}
	if( g_co_sysenv.cnt > 1 )
	{
		qsort( g_co_sysenv.data,g_co_sysenv.cnt,sizeof(stCoSysEnv_t),co_sysenv_comp );
		// 快排
		// 环境变量去重
		stCoSysEnv_t *lp = g_co_sysenv.data;
		stCoSysEnv_t *lq = g_co_sysenv.data + 1;
		for(size_t i=1;i<g_co_sysenv.cnt;i++)
		{
			if( strcmp( lp->name,lq->name ) )
			{
				++lp;
				if( lq != lp  )
				{
					*lp = *lq;
				}
			}
			++lq;
		}
		g_co_sysenv.cnt = lp - g_co_sysenv.data + 1;
	}

}
```

####8.1.1.stCoSysEnvArr_t数据结构
位于`co_hook_sys_call.cpp`，代码如下：
```cpp
struct stCoSysEnv_t
{
	char *name;	
	char *value;
};
struct stCoSysEnvArr_t
{
	stCoSysEnv_t *data;
	size_t cnt;
};
```

###8.2.RoutineFunc函数
位于`example_setenv.cpp`，代码如下：
```cpp
// co_create(&co, NULL, RoutineFunc, &args[i]);
void* RoutineFunc(void* args)
{
	co_enable_hook_sys();

	stRoutineArgs_t* g = (stRoutineArgs_t*)args;

	SetAndGetEnv(g->iRoutineID);
	return NULL;
}
```

####8.2.1.SetAndGetEnv函数
位于`example_setenv.cpp`，代码如下：
```cpp
void SetAndGetEnv(int iRoutineID)
{
	printf("routineid %d begin\n", iRoutineID);

	//use poll as sleep
	// sleep 500ms
	poll(NULL, 0, 500);

	char sBuf[128];
	sprintf(sBuf, "cgi_routine_%d", iRoutineID);
	// 记在自己的协程空间里面（协程变量）
	int ret = setenv("CGINAME", sBuf, 1);
	if (ret)
	{
		printf("%s:%d set env err ret %d errno %d %s\n", __func__, __LINE__,
				ret, errno, strerror(errno));
		return;
	}
	printf("routineid %d set env CGINAME %s\n", iRoutineID, sBuf);

	// epoll 500ms可能会被切出去
	poll(NULL, 0, 500);

	char* env = getenv("CGINAME");
	if (!env)
	{
		printf("%s:%d get env err errno %d %s\n", __func__, __LINE__,
				errno, strerror(errno));
		return;
	}
	printf("routineid %d get env CGINAME %s\n", iRoutineID, env);
}
```

#####8.2.1.1.setenv函数
位于`co_hook_sys_call.cpp`，代码如下：
```cpp
// int ret = setenv("CGINAME", sBuf, 1);
// hook的目的：将进程几倍的环境变量存到协程栈中。
int setenv(const char *n, const char *value, int overwrite)
{
	HOOK_SYS_FUNC( setenv )
	if( co_is_enable_sys_hook() && g_co_sysenv.data )
	{
		// 获取当前协程
		stCoRoutine_t *self = co_self();
		if( self )
		{
			if( !self->pvEnv )
			{
				// 生成分配
				// 设置到当前协程的void *pvEnv;
				self->pvEnv = dup_co_sysenv_arr( &g_co_sysenv );
			}
			stCoSysEnvArr_t *arr = (stCoSysEnvArr_t*)(self->pvEnv);

			// n="CGINAME"
			stCoSysEnv_t name = { (char*)n,0 };

			// 二分搜索法
			// 在stCoSysEnvArr_t中查找name(CGINAME)
			stCoSysEnv_t *e = (stCoSysEnv_t*)bsearch( &name,arr->data,arr->cnt,sizeof(name),co_sysenv_comp );

			if( e )
			{
				// overwrite = 1; 需要覆盖写入
				if( overwrite || !e->value  )
				{
					if( e->value ) free( e->value );
					e->value = ( value ? strdup( value ) : 0 );
				}
				return 0;
			}
		}

	}
	return g_sys_setenv_func( n,value,overwrite );
}

static stCoSysEnvArr_t *dup_co_sysenv_arr( stCoSysEnvArr_t * arr )
{
	stCoSysEnvArr_t *lp = (stCoSysEnvArr_t*)calloc( sizeof(stCoSysEnvArr_t),1 );	
	if( arr->cnt )
	{
		lp->data = (stCoSysEnv_t*)calloc( sizeof(stCoSysEnv_t) * arr->cnt,1 );
		lp->cnt = arr->cnt;
		memcpy( lp->data,arr->data,sizeof( stCoSysEnv_t ) * arr->cnt );
	}
	return lp;
}

// 根据stCoSysEnv.name进行排序
static int co_sysenv_comp(const void *a, const void *b)
{
	return strcmp(((stCoSysEnv_t*)a)->name, ((stCoSysEnv_t*)b)->name); 
}
```

#####8.2.1.2.getenv函数
位于`co_hook_sys_call.cpp`，代码如下：
```cpp
char *getenv( const char *n )
{
	HOOK_SYS_FUNC( getenv )
	if( co_is_enable_sys_hook() && g_co_sysenv.data )
	{
		stCoRoutine_t *self = co_self();

		stCoSysEnv_t name = { (char*)n,0 };

		if( !self->pvEnv )
		{
			self->pvEnv = dup_co_sysenv_arr( &g_co_sysenv );
		}
		stCoSysEnvArr_t *arr = (stCoSysEnvArr_t*)(self->pvEnv);

		// 二分查找
		stCoSysEnv_t *e = (stCoSysEnv_t*)bsearch( &name,arr->data,arr->cnt,sizeof(name),co_sysenv_comp );

		if( e )
		{
			return e->value;
		}

	}
	// 非hook，才从进程的环境变量中获取
	return g_sys_getenv_func( n );

}
```

###8.3.综述
该程序主要介绍了如何使用协程变量，或者协程的环境变量的读写（对`setenv`和`getenv`的hook）。输出结果：

![Alt text](./1531073673375.png)

如图所示，主协程在调度完三个协程之后，一直在event
