# 【NO.193】[ C++ ] 一篇带你了解C++中动态内存管理

在我们日常写代码的过程中，我们对内存空间的需求有时候在程序运行的时候才能知道，这时候我们就需要使用动态开辟内存的方法。

## 1、C/C++程序的内存开辟

首先我们先了解一下C/C++程序内存分配的几个区域：

```text
int globalVar = 1;
static int staticGlobalVar = 1;
void Test()
{
	static int staticVar = 1;
	int localVar = 1;
	int num1[10] = { 1, 2, 3, 4 };
	char char2[] = "abcd";
	const char* pChar3 = "abcd";
	int* ptr1 = (int*)malloc(sizeof(int) * 4);
	int* ptr2 = (int*)calloc(4, sizeof(int));
	int* ptr3 = (int*)realloc(ptr2, sizeof(int) * 4);
	free(ptr1);
	free(ptr3);
}
```

![img](https://pic2.zhimg.com/80/v2-7a074855730329beb22b517478d294dd_720w.webp)

\1. 栈区（stack）： 在执行函数时，函数内局部变量的存储单元都可以在栈上创建，函数执行结束时这些存储单元自动被释放。栈内存分配运算内置于处理器的指令集中，效率很高，但是分配的内存容量有限。 栈区主要存放运行函数而分配的局部变量、函数参数、返回数据、返回地址等。

\2. 堆区（heap）： 一般由程序员分配释放， 若程序员不释放，程序结束时可能由 OS 回收 。分配方式类似于链表。

\3. 数据段（静态区） （ static ）存放全局变量、静态数据。程序结束后由系统释放。

\4. 代码段： 存放函数体（类成员函数和全局函数）的二进制代码。

这幅图中，我们可以发现普通的局部变量是在栈上分配空间的，在栈区中创建的变量出了作用域去就会自动销毁。但是被static修饰的变量是存放在数据段(静态区)，在数据段上创建的变量直到程序结束才销毁，所以数据段上的数据生命周期变长了。

## 2.C语言中动态内存管理方式：malloc/calloc/realloc/free

在C语言中，我们经常会用到malloc,calloc和realloc来进行动态的开辟内存；同时，C语言还提供了一个函数free，专门用来做动态内存的释放和回收。其中他们三个的区别也是我们需要特别所强调区别的。

### 2.1malloc、calloc、realloc区别？

malloc函数是向内存申请一块连续可用的空间，并返回指向这块空间的指针。

calloc与malloc的区别只在于calloc会在返回地址之前把申请的空间的每个字节初始化为0。

realloc函数可以做到对动态开辟内存大小的调整。

我们通过这三个函数的定义也可以进行功能的区分：

```text
void Test ()
{
int* p1 = (int*) malloc(sizeof(int));
free(p1);
 
int* p2 = (int*)calloc(4, sizeof (int));
int* p3 = (int*)realloc(p2, sizeof(int)*10);
 
free(p3 );
}
```



## 3.C++内存管理方式

我们都知道，C++语言是兼容C语言的，因此C语言中内存管理方式在C++中可以继续使用。但是有些地方就无能为力了，并且使用起来也可能比较麻烦。因此，C++拥有自己的内管管理方式：通过new和delete操作符进行动态内存管理。

### 3.1 new/delete操作内置类型

```text
int main()
{
	// 动态申请一个int类型的空间
	int* ptr1 = new int;
 
	// 动态申请一个int类型的空间并初始化为10
	int* ptr2 = new int(10);
 
	// 动态申请3个int类型的空间(数组)
	int* ptr3 = new int[3];
 
	// 动态申请3个int类型的空间,初始化第一个空间值为1
	int* ptr4 = new int[3]{ 1 };
 
	delete ptr1;
	delete ptr2;
	delete[] ptr3;
	delete[] ptr4;
 
	return 0;
}
```

我们首先通过画图分析进行剖析代码：

![img](https://pic1.zhimg.com/80/v2-689fdaf75639c236668fa9a9ab2d0cac_720w.webp)

我们在监视窗口看看这3个变量

![img](https://pic3.zhimg.com/80/v2-da3a633d1718cc1f356e1116ab5fa162_720w.webp)

![img](https://pic4.zhimg.com/80/v2-27d18dde5aa447e0ba15160cdb674efb_720w.webp)

注意：申请和释放单个元素的空间，使用new和delete操作符，申请和释放连续的空间，使用new[]和delete[]，要匹配起来使用。

### 3.2 new和delete操作自定义类型

```text
class A {
public:
	A(int a = 0)
		: _a(a)
	{
		cout << "A():" << this << endl;
	}
	~A()
	{
		cout << "~A():" << this << endl;
	}
private:
	int _a;
};
int main()
{
	A* p1 = (A*)malloc(sizeof(A));
	A* p2 = new A(1);
	free(p1);
	delete p2;
 
	return 0;
}
```

在这段代码中，p1是我们使用malloc开辟的，p2是通过new来开辟的。我们编译运行这段代码。

![img](https://pic1.zhimg.com/80/v2-20ec18eb90d468ec53c9a1fc55dbfb44_720w.webp)

**注意：在申请自定义类型的空间时，new会自动调用构造函数，delete时会调用析构函数，而malloc和free不会。**

### 3.3new和malloc处理失败

```text
int main()
{
	void* p0 = malloc(1024 * 1024 * 1024);
	cout << p0 << endl;
 
	//malloc失败，返回空指针
	void* p1 = malloc(1024 * 1024 * 1024);
	cout << p1 << endl;
 
	try
	{
		//new失败，抛异常
		void* p2 = new char[1024 * 1024 * 1024];
		cout << p2 << endl;
	}
	catch (const exception& e)
	{
		cout << e.what() << endl;
	}
 
	return 0;
}
```

![img](https://pic2.zhimg.com/80/v2-048a708b4503b8cf4ed70f5459741491_720w.webp)

我们能够发现，malloc失败时会返回空指针，而new失败时，会抛出异常。

## 4.operator new与operator delete函数

### 4.1 operator new与operator delete函数

C++标准库还提供了operator new和operator delete函数，但是这两个函数并不是对new和delete的重载，operator new和operator delete是两个库函数。(这里C++大佬设计时这样取名确实很容易混淆)

**4.1.1 我们看看operator new库里面的源码**

![img](https://pic2.zhimg.com/80/v2-fd3d59c2cae26c43c304ecfecb5ede79_720w.webp)

```text
void* __CRTDECL operator new(size_t size) _THROW1(_STD bad_alloc) {
	// try to allocate size bytes
	void* p;
	while ((p = malloc(size)) == 0)
		if (_callnewh(size) == 0)
		{
			// report no memory
			// 如果申请内存失败了，这里会抛出bad_alloc 类型异常
			static const std::bad_alloc nomem;
			_RAISE(nomem);
		}
	return (p);
}
```

库里面operator new的作用是封装了malloc，如果malloc失败，抛出异常。

**4.1.2 operator delete库里面的源码**

该函数最终是通过free来释放空间的

![img](https://pic3.zhimg.com/80/v2-ab40afe70f6b97692dd72472f7e57b12_720w.webp)

```text
//operator delete 源码
void operator delete(void* pUserData) {
	_CrtMemBlockHeader* pHead;
	RTCCALLBACK(_RTC_Free_hook, (pUserData, 0));
	if (pUserData == NULL)
		return;
	_mlock(_HEAP_LOCK);  /* block other threads */
	__TRY
		        /* get a pointer to memory block header */
		pHead = pHdr(pUserData);
	         /* verify block type */
	_ASSERTE(_BLOCK_TYPE_IS_VALID(pHead->nBlockUse));
	_free_dbg(pUserData, pHead->nBlockUse);
	__FINALLY
		_munlock(_HEAP_LOCK);  /* release other threads */
	__END_TRY_FINALLY
		return;
}
 
/*
free的实现
*/
#define   free(p)               _free_dbg(p, _NORMAL_BLOCK)
```

**4.1.3 operator new和operator delete的价值(重点)**

```text
class A {
public:
	A(int a = 0)
		: _a(a)
	{
		cout << "A():" << this << endl;
	}
	~A()
	{
		cout << "~A():" << this << endl;
	}
private:
	int _a;
};
int main()
{
	//跟malloc功能一样，失败以后抛出异常
	A* ps1 = (A*)operator new(sizeof(A));
	operator delete(ps1);
 
	A* ps2 = (A*)malloc(sizeof(A));
	free(ps2);
 
	A* ps3 = new A;
	delete ps3;
 
	return 0;
}
```

我们使用new的时候，new要开空间，要调用构造函数。new可以转换成call malloc，call 构造函数。但是call malloc 一旦失败，会返回空指针或者错误码。在面向对象的语言中更喜欢使用异常。而operator new相比较malloc的不同就在于如果一旦失败会抛出异常，因此new的底层实现是调用operator new，operator new会调用malloc(如果失败抛出异常)，再调用构造函数。

我们通过汇编看一下ps3

![img](https://pic3.zhimg.com/80/v2-d289598f8aac95c8bf07e415ebd8efca_720w.webp)

operator delete同理。

总结：通过上述两个全局函数的实现知道，operator new 实际也是通过malloc来申请空间，如果malloc申请空间成功就直接返回，否则执行用户提供的空间不足应对措施，如果用户提供该措施就继续申请，否则就抛异常。operator delete 最终是通过free来释放空间的。

### 4.2 重载operator new 与 operator delete（了解）

专属的operator new技术，提高效率。应用：内存池

```text
class A {
public:
	A(int a = 0)
		: _a(a)
	{
		cout << "A():" << this << endl;
	}
 
	// 专属的operator new
	void* operator new(size_t n)
	{
		void* p = nullptr;
		p = allocator<A>().allocate(1);
		cout << "memory pool allocate" << endl;
		return p;
	}
 
	void operator delete(void* p)
	{
		allocator<A>().deallocate((A*)p, 1);
		cout << "memory pool deallocate" << endl;
 
	}
 
	~A()
	{
		cout << "~A():" << this << endl;
	}
private:
	int _a;
};
int main()
{
	int n = 0;
	cin >> n;
	for (int i = 0; i < n; ++i)
	{
		A* ps1 = new A; //operator new + A的构造函数
	}
 
	return 0;
}
```

![img](https://pic4.zhimg.com/80/v2-5b5a98246fa90142f90759d1bc76722f_720w.webp)

注意：一般情况下不需要对operator new和operator delete进行重载，除非在申请和释放空间时候有某些特殊的需求。比如：在使用new和delete申请和释放空间时，打印一些日志信息，可以简单帮助用户来检测是否存在内存泄漏。

## 5.new 和 delete 的实现原理

### 5.1 内置类型

如果申请的是内置类型的空间，new和malloc，delete和free基本类似，不同的地方是：new/delete申请和释放的是单个元素的空间，new[]和delete[]申请的是连续空间，而且new在申请空间失败时会抛异常，malloc会返回NULL。

### 5.2 自定义类型

**5.2.1 new原理**

1、调用operator new函数申请空间

2、再调用构造函数，完成对对象的构造。

**5.2.2 delete原理**

1、先调用析构函数，完成对对象中资源的清理工作。

2、调用operator delete函数释放对象的空间

### 5.2.3 new T[N]原理

1、先调用operator new[]函数，在operator new[]中世纪调用operator new函数完成N个对象空间的申请

2、在申请的空间上执行N次构造函数

**5.2.4 delete[]原理**

1、在释放的对象空间上执行N次析构函数，完成对N个对象中资源的清理

2、调用operator delete[]释放空间，实际在operator delete[]中调用operator delete来释放空间。

## 6.malloc/free和new/delete的异同

**6.1malloc/free和new/delete的共同点**

都是从堆上申请空间，都需要用户手动释放空间。

**6.2malloc/free和new/delete的不同点**

1：malloc 和 free 是函数， new 和 delete 是操作符

2：malloc 申请的空间不会初始化， new 可以初始化

3：malloc 申请空间时，需要手动计算空间大小并传递， new 只需在其后跟上空间的类型即可，如果是多个对象，[] 中指定对象个数即可

4：malloc 的返回值为 void*, 在使用时必须强转， new 不需要，因为 new 后跟的是空间的类型

5：malloc 申请空间失败时，返回的是 NULL ，因此使用时必须判空， new 不需要，但是 new 需要捕获异常

6： 申请自定义类型对象时， malloc/free 只会开辟空间，不会调用构造函数与析构函数，而 new在申请空间后会调用构造函数完成对象的初始化，delete 在释放空间前会调用析构函数完成空间中资源的清理

原文地址：https://zhuanlan.zhihu.com/p/545869650

作者：linux