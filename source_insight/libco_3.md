#libco(3)
@(源码)


##2.example_closure.cpp
有关闭包的例子，什么是闭包？简单理解就是：
> 当匿名函数和non-local变量结合起来，就形成了闭包。

main函数有一定篇幅，位于`example_closure.cpp`中，如下：
```cpp
int main( int argc,char *argv[] )
{
	// 建立了stCoClosure_t的vector数组
	vector< stCoClosure_t* > v;

	// 保证线程同步的互斥锁，初始化线程锁
	pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;

	int total = 100;
	vector<int> v2;
	// 定义了type_ref这个类,把total,v2,m塞进去作为类的成员
	co_ref( ref,total,v2,m);
	for(int i=0;i<10;i++)
	{
		// for循环10次,调用宏co_func
		co_func( f,ref,i )
		{
			// 打印type_ref ref.total这个成员。
			printf("ref.total %d i %d\n",ref.total,i );
			//lock：锁定线程互斥锁
			pthread_mutex_lock(&ref.m);
			// 把i变量push_back到v2里面
			ref.v2.push_back( i );
			// 解除互斥锁
			pthread_mutex_unlock(&ref.m);
			//unlock
		}
		co_func_end;
		// 把闭包类f放入到v里面（10个）
		v.push_back( new f( ref,i ) );
	}
	for(int i=0;i<2;i++)
	{
		// for循环两次
		co_func( f2,i )
		{
			printf("i: %d\n",i);
			for(int j=0;j<2;j++)
			{
				usleep( 1000 );
				printf("i %d j %d\n",i,j);
			}
		}
		co_func_end;
		// 把f2(0)和f2(1)放到v里面
		v.push_back( new f2( i ) );
	}
	// 多线程执行闭包数组v
	batch_exec( v );
	printf("done\n");

	return 0;
}
```

###2.1.stCoClosure数据结构
位于`co_closure.h`中，结构非常简单，只有一个纯虚函数`exec()`。
```cpp
struct stCoClosure_t 
{
public:
	virtual void exec() = 0;
};
```

###2.2.co_ref函数
位于`co_closure.h`中，代码是一段宏：
```cpp
// 调用：co_ref( ref,total,v2,m);
//2.1 reference

#define co_ref( name,... )\
repeat( comac_argc(__VA_ARGS__) ,decl_typeof,__VA_ARGS__ )\
class type_##name\
{\
public:\
	repeat( comac_argc(__VA_ARGS__) ,impl_typeof,__VA_ARGS__ )\
	int _member_cnt;\
	type_##name( \
		repeat( comac_argc(__VA_ARGS__),con_param_typeof,__VA_ARGS__ ) ... ): \
		repeat( comac_argc(__VA_ARGS__),param_init_typeof,__VA_ARGS__ ) _member_cnt(comac_argc(__VA_ARGS__)) \
	{}\
} name( __VA_ARGS__ ) ;

//1.base 
//-- 1.1 comac_argc

#define comac_get_args_cnt( ... ) comac_arg_n( __VA_ARGS__ )
#define comac_arg_n( _0,_1,_2,_3,_4,_5,_6,_7,N,...) N
#define comac_args_seqs() 7,6,5,4,3,2,1,0
#define comac_join_1( x,y ) x##y

#define comac_argc( ... ) comac_get_args_cnt( 0,##__VA_ARGS__,comac_args_seqs() )
#define comac_join( x,y) comac_join_1( x,y )

//-- 1.2 repeat
#define repeat_0( fun,a,... ) 
#define repeat_1( fun,a,... ) fun( 1,a,__VA_ARGS__ ) repeat_0( fun,__VA_ARGS__ )
#define repeat_2( fun,a,... ) fun( 2,a,__VA_ARGS__ ) repeat_1( fun,__VA_ARGS__ )
#define repeat_3( fun,a,... ) fun( 3,a,__VA_ARGS__ ) repeat_2( fun,__VA_ARGS__ )
#define repeat_4( fun,a,... ) fun( 4,a,__VA_ARGS__ ) repeat_3( fun,__VA_ARGS__ )
#define repeat_5( fun,a,... ) fun( 5,a,__VA_ARGS__ ) repeat_4( fun,__VA_ARGS__ )
#define repeat_6( fun,a,... ) fun( 6,a,__VA_ARGS__ ) repeat_5( fun,__VA_ARGS__ )

#define repeat( n,fun,... ) comac_join( repeat_,n )( fun,__VA_ARGS__)

//2.implement
#if __cplusplus <= 199711L
#define decl_typeof( i,a,... ) typedef typeof( a ) typeof_##a;
#else
#define decl_typeof( i,a,... ) typedef decltype( a ) typeof_##a;
#endif
#define impl_typeof( i,a,... ) typeof_##a & a;
#define impl_typeof_cpy( i,a,... ) typeof_##a a;
#define con_param_typeof( i,a,... ) typeof_##a & a##r,
#define param_init_typeof( i,a,... ) a(a##r),
```
这个函数非常复杂，我当时理解起来非常困难，幸得雷哥的指导，读起来就相当舒服了。

展开过程：
1. `comac_argc(__VA_ARGS__)`：`__VA_ARGS__`代表可变参数列表（在这里为了生动解释例子：`__VA_ARGS__`代表实例`a, b, c, d, e`）。变成：`comac_get_args_cnt(0, ##__VA_ARGS__, comac_args_seqs() )`;
2. `comac_get_args_cnt(0, ##__VA_ARGS__, comac_args_seqs() )`：展开了便是：`comac_get_args_cnt(0, a, b, c, d, e, comac_args_seqs() )`。而`comac_args_seqs()`展开便是：`7,6,5,4,3,2,1`;
3. 全展开便是：`comac_get_args_cnt(0, a, b, c, d, e, 7, 6, 5, 4, 3, 2, 1)`。转过来即是：`comac_arg_n(0, a, b, c, d, e, 7, 6, 5, 4, 3, 2, 1)`，又因为：`comac_arg_n( _0, _1, _2, _3, _4, _5, _6, _7,N, ...) N`，所以值即是：5，这个函数的目的就是为了求出`__VA_ARGS__`有多少个参数。这样的写法，也说明了，支持最多7个函数参数；
4. `repeat(  comac_argc(__VA_ARGS__) ,decl_typeof,__VA_ARGS__ )`。我们先看`repeat`这个宏到底做了啥。
5. `repeat(  comac_argc(__VA_ARGS__) ,decl_typeof,__VA_ARGS__ )` 展开即是：`repeat( 5, decl_type, a, b, c, d, e)`。解开即：`comac_join( repeat_, 5)( fun, a, b, c, d, e)`。
6. `#define comac_join( x, y) comac_join_1(x, y)`，展开即是：`comac_join_1( , repeat_, 5)`,即：`repeat_5`
7. 转化成：`repeat_5(fun, __VA_ARGS__)`。即：`repeat_5(fun, a, b, c, d, e)`。
8. 由于`#define repeat_5( fun,a,... ) fun( 5,a,__VA_ARGS__ ) repeat_4( fun,__VA_ARGS__ )`，所以这里做的事情是：调了一次`fun(5, a, b, c, d, e)`，然后再调了一次`repeat_4(fun,__VA_ARGS__)`;
9. 由于`#define repeat_4( fun,a,... ) fun( 4,a,__VA_ARGS__ ) repeat_3( fun,__VA_ARGS__ )`，所以这里做的事情是：调了一次`fun(4, a, b, c, d, e)`，然后再调了一次`repeat_3(fun,a,b,c,d,e)`;
10. ...一直到`#define repeat_0( fun,a,... )`不执行任何事情。
11. 所以这个过程是做了一个for循环：`fun(1, a, b, c, d, e)~fun(5, a, b, c, d, e)`;
12. 再看`decl_typeof`这个做了什么：
```cpp
#if __cplusplus <= 199711L
#define decl_typeof( i,a,... ) typedef typeof( a ) typeof_##a;
#else
#define decl_typeof( i,a,... ) typedef decltype( a ) typeof_##a;
#endif
```
可以看到，这里相当于将函数参数的数据类型改了个别名形如：`typeof_a`，`typeof_b`，`typeof_c`，`typeof_d`，`typeof_e`。
13. 定义类：`type_ref`；
14. 类里面执行`repeat( comac_argc(__VA_ARGS__) ,impl_typeof,__VA_ARGS__ )\`。这里相当于循环执行：`#define impl_typeof( i,a,... ) typeof_##a & a;`；
15. 然后在类中执行：
```cpp
type_##name( \
		repeat( comac_argc(__VA_ARGS__),con_param_typeof,__VA_ARGS__ ) ... ): \
		repeat( comac_argc(__VA_ARGS__),param_init_typeof,__VA_ARGS__ ) _member_cnt(comac_argc(__VA_ARGS__)) \
	{}\
#define con_param_typeof( i,a,... ) typeof_##a & a##r,
#define param_init_typeof( i,a,... ) a(a##r),
```
相当于for循环执行`con_param_typeof`函数和`param_init_typeof`函数。

所以这里：`co_ref( ref,total,v2,m);`做的事情就是：
`repeat( comac_argc(__VA_ARGS__) ,decl_typeof,__VA_ARGS__ )\`
声明函数参数类型：
```cpp
typedef typeof(total) typeof_total;
typedef typeof(v2) typeof_v2;
typedef typeof(m) typeof_m;

class type_ref
{
	public:
		typeof_total & total;
		typeof_v2 & v2;
		typeof_m & m;
		int _member_cnt;
		type_ref(
			typeof_total & totalr,
			typeof_v2 & v2r,
			typeof_m & mr,
			...
		): 
		total(totalr),
		v2(v2r),
		m(mr),
		_member_cnt(3)
		{}
} ref(total, v2, m);
```
定义了type_ref这样的一个类。

###2.3.co_func函数
位于`co_closure.h`。代码如下：
```cpp
//2.2 function
// co_func( f,ref,i )
#define co_func(name,...)\
repeat( comac_argc(__VA_ARGS__) ,decl_typeof,__VA_ARGS__ )\
class name:public stCoClosure_t\
{\
public:\
	repeat( comac_argc(__VA_ARGS__) ,impl_typeof_cpy,__VA_ARGS__ )\
	int _member_cnt;\
public:\
	name( repeat( comac_argc(__VA_ARGS__),con_param_typeof,__VA_ARGS__ ) ... ): \
		repeat( comac_argc(__VA_ARGS__),param_init_typeof,__VA_ARGS__ ) _member_cnt(comac_argc(__VA_ARGS__))\
	{}\
	void exec()

#define co_func_end }
```
有了上面阅读`co_ref`宏的经验，阅读这个`co_func`就简单多了，展开过程：

1. 定义函数参数类型；
2. 定义类继承`stCoClosure_t`；
3. 展开`repeat( comac_argc(__VA_ARGS__) ,impl_typeof_cpy,__VA_ARGS__ )\ int _member_cnt;\`;
4. 展开`con_param_typeof`;


```cpp
typedef typeof(ref) type_ref;
typedef typeof(i) type_i;

class f : public stCoClosure_t 
{
	public:
		type_ref ref;
		type_i i;
		int _member_cnt;
	public:
		f(
			type_ref & refr,
			type_i & i_r,
			...
		):
		ref(refr),
		i(ir),
		_member_cnt(2)
		{} // 构造函数
		void exec() // 这里继承虚基类stCoClosure_t的纯虚函数exec
}；
```
显然，上面的内容并没有实现`exec()`函数。靠代码当前的内容：
```cpp
{
	printf("ref.total %d i %d\n",ref.total,i );
	//lock
	pthread_mutex_lock(&ref.m);
	ref.v2.push_back( i );
	pthread_mutex_unlock(&ref.m);
	//unlock
}
co_func_end;
```

所以这里类展开后内容为：
```cpp
typedef typeof(ref) type_ref;
typedef typeof(i) type_i;

class f : public stCoClosure_t 
{
	public:
		type_ref ref;
		type_i i;
		int _member_cnt;
	public:
		f(
			type_ref & refr,
			type_i & i_r,
			...
		):
		ref(refr),
		i(ir),
		_member_cnt(2)
		{} // 构造函数
		void exec() // 这里继承虚基类stCoClosure_t的纯虚函数exec
		{
			printf("ref.total %d i %d\n",ref.total,i );
			//lock
			pthread_mutex_lock(&ref.m);
			ref.v2.push_back( i );
			pthread_mutex_unlock(&ref.m);
			//unlock
		}
}；
```


而，`co_func( f2,i )`的展开为：
```cpp
typedef typeof(i) type_i;

class f2: public stCoClosure_t
{
	public:
		type_i i;
		int _member_cnt;
	public:
		f2(
			type_i & ir,
			...
		):
		i(ir),
		_member_cnt(1)
		{}  // 构造函数
		void exec() // 这里继承虚基类stCoClosure_t的纯虚函数exec
		{
			printf("i: %d\n", i);
			for (int j = 0; j < 2; j++)
			{
				usleep(1000);
				printf("i %d j %d\n", i ,j);
			}
		}
};

```

###2.4.batch_exec函数
代码如下：
```cpp
static void batch_exec( vector<stCoClosure_t*> &v )
{
	// 线程数组
	vector<pthread_t> ths;
	for( size_t i=0;i<v.size();i++ )
	{
		pthread_t tid;
		// 创建线程
		pthread_create( &tid,0,thread_func,v[i] );
		ths.push_back( tid );
	}
	for( size_t i=0;i<v.size();i++ )
	{
		// 以阻塞的方式等待线程终止
		pthread_join( ths[i],0 );
	}
}
```

由此看到，线程调用的是函数`thread_func`。代码如下：
```cpp
static void *thread_func( void * arg )
{
	stCoClosure_t *p = (stCoClosure_t*) arg;
	p->exec();
	return 0;
}
```
执行闭包的`exec`函数。

总结：这其实只是通过宏的包装，看起来很像一个内部的匿名函数（`co_func`里面的`exec`函数引用了外部的变量`i`，`ref`等。）。不过这里面的实现用到宏的地方，确实值得借鉴，尤其获取参数个数的地方，思路更是非常新奇。
