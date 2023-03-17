# 【NO.605】C++之内存管理：申请与释放

## 1.C/C++内存分布

### **1.1虚拟内存分段**

一所学校，在设计的时候是有其规划的。要有宿舍楼，要有办公楼，要有教学楼，要有图书馆，要有体育馆，要有食堂，要有礼堂……，不同的建筑发挥不同的作用，使校园整体功能齐全，设计合理。

内存也是如此，我们称之为虚拟内存分段。

![img](https://pic4.zhimg.com/80/v2-2673b2b190081bee5bd42f4ec0acf367_720w.webp)

注意：

1. 栈又叫堆栈，非静态局部变量/函数参数/返回值等等，栈是向下增长的。
2. 内存映射段是高效的I/O映射方式，用于装载一个共享的动态内存库。用户可使用系统接口创建共享共享内存，做进程间通信。
3. 堆用于程序运行时动态内存分配，堆是可以上增长的。
4. 数据段–存储全局数据和静态数据。
5. 代码段–可执行的代码/只读常量。

### **1.2理解一些概念**

1.2.1栈帧向下增长

> 栈又叫堆栈，非静态局部变量/函数参数/返回值等等，栈是向下增长的。

那么，什么叫做向下增长呢？我们来看一个例子：

```text
#include<iostream>
using namespace std;

void f2()
{
	int b = 0;
	cout <<"b:"<< &b << endl;
	cout << endl;
	cout << endl;
}

void f1()
{
	int a = 0;
	cout<<"a:" << &a << endl;
	cout << endl;
	cout << endl;
	f2();
}

int main()
{
	f1();
	return 0;
}
```

在这段代码中，函数调用的时候会在内存上建立栈帧。程序由main函数进入，接着调用f1，接着再调用f2.

![img](https://pic4.zhimg.com/80/v2-f6a616d9002142da77881afc09ab7ebb_720w.webp)

我们来看一下输出的，a和b的地址的结果：

![img](https://pic1.zhimg.com/80/v2-6eeff770f633cd63619104182c7b2504_720w.webp)

我们发现，a的地址是比b的地址高的，也就是说在栈帧生长的过程中，地址是从高到低的，所以我们说，栈是向下生长的。

然后通过这个例子我们对局部变量的生命周期也可以有一个新的理解，出栈的时候栈会销毁，局部变量也随之消失，这就是局部变量的生命周期。

1.2.2堆向上生长

堆和栈类似，但堆是向上生长的。

![img](https://pic2.zhimg.com/80/v2-e5b80d53f68788bd1bd9878d1ba39e1d_720w.webp)

但是，后申请的空间的地址一定会比先申请的大吗？

当我们多申请几次，来看一下：

![img](https://pic4.zhimg.com/80/v2-6320d3ed14c505076f0db28013043663_720w.webp)

这又是为什么呢？因为内存不断申请和释放，所以我们有可能申请到前面释放过的空间，这就可能会导致这种情况的产生。所以堆是可以向上增长的，但不一定。

1.2.3栈和堆会碰撞吗？

我们上面已经知道，栈和堆一个是向下生长，一个是向上生长，这么双向生长难道不会碰撞吗？

![img](https://pic1.zhimg.com/80/v2-9bc6eba00a3fd7432bd562d6ede5be7c_720w.webp)

答案是不会的。

栈并不是一直可以向下走的，栈是有规定大小的。堆很大，但也是有上限的。所以会存在失败的栈溢出和malloc失败的情况。

1.2.4关于const的说明

我们说常量区是是储存常量的，那么const定义的常量是存在常量区吗？

答案是不是的，注意，const定义的是常变量，本质还是变量哦，记住，本质是变量，所以不在常量区。那么如果const定义在函数里面，就在栈区，如果定义成static就在静态区。

## 2.C语言中动态内存管理方式

### **2.1malloc/calloc/realloc和free**

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

### **2.2 malloc/calloc/realloc的区别**

![img](https://pic3.zhimg.com/80/v2-169738f8fd9b0830e16867221af7e39a_720w.webp)

## 3.C++内存管理方式

C语言内存管理方式在C++中可以继续使用，但有些地方就无能为力而且使用起来比较麻烦，因此C++又提出了自己的内存管理方式：通过new和delete操作符进行动态内存管理

### **3.1 new/delete操作内置类型**

![img](https://pic1.zhimg.com/80/v2-813ee9feb3c7552754abe73345a4e42c_720w.webp)

3.1.1malloc和freeVSnew和delete

我们以申请10个int的数组为例

![img](https://pic2.zhimg.com/80/v2-3ed5b6414a7bd4cf6a7be5c80275d47d_720w.webp)

申请和释放单个元素的空间，使用new和delete操作符，申请和释放连续的空间，使用new[]和delete[]

3.1.2 使用new动态申请的实例

```text
void Test()
{
// 动态申请一个int类型的空间
int* ptr4 = new int;
// 动态申请一个int类型的空间并初始化为10
int* ptr5 = new int(10);
// 动态申请10个int类型的空间
int* ptr6 = new int[3];
delete ptr4;
delete ptr5;
delete[] ptr6;
}
```

### 3.2 new和delete操作自定义类型

```text
#include<iostream>
using namespace std;

struct ListNode
{
	ListNode *_next;
	ListNode *_prev;
	int _val;
	ListNode(int val = 0)
		:_next(nullptr), _prev(nullptr), _val(val)
	{}
};
int main()
{
	//C
	struct ListNode *n1 = (struct ListNode *)malloc(sizeof(struct ListNode));

	//C++
	ListNode *n2 = new ListNode;

	return 0;
}
```

我们来看看上面的代码，我们分别开辟了n1和n2，他们会有什么不同吗？我们来看调试结果。

![img](https://pic3.zhimg.com/80/v2-2e295e455ef45bdab5c0213e4bd0cde6_720w.webp)

所以我们可以知道，malloc只是开空间，而new针对自定义类型时，是开空间加构造函数初始化。

如果我们再使用free和delete，再来看看。当我们屏蔽掉new。

![img](https://pic4.zhimg.com/80/v2-8dfb092c5addc0f1173d5155851c96e3_720w.webp)

而当我们屏蔽掉malloc

![img](https://pic1.zhimg.com/80/v2-4d5e6c83758ff443e57167b098b113e0_720w.webp)

我们发现，在delete的同时，也调用了析构函数。而free并不能调用析构函数。

于是，我们可以得出结论：申请自定义类型的空间时，new会调用构造函数，delete会调用析构函数，而malloc与free不会。

![img](https://pic1.zhimg.com/80/v2-f9f032a7c49c0cd44096a58e68964dfc_720w.webp)

## 4.operator new与operator delete函数

### **4.1 operator new与operator delete函数**

new和delete是用户进行动态内存申请和释放的操作符，operator new 和operator delete是系统提供的全局函数，new在底层调用operator new全局函数来申请空间，delete在底层通过operator delete全局函数来释放空间。

4.1.1对比申请失败处理方式

他们的用法和malloc和free一样，都是在堆上申请释放空间，但是失败了的处理方式不同，malloc是返回空指针，而operator new是抛异常

```text
#include<iostream>
using namespace std;

struct ListNode
{
	ListNode *_next;
	ListNode *_prev;
	int _val;
	ListNode(int val = 0)
		:_next(nullptr), _prev(nullptr), _val(val)
	{}
	~ListNode()
	{
		cout << "ListNode 析构" << endl;
	}
};
int main()
{
	struct ListNode *n1 = (struct ListNode *)malloc(sizeof(struct ListNode));
	free(n1);
	struct ListNode *n2 = (struct ListNode *)operator new(sizeof(struct ListNode));
	operator delete (n2);
	void *p3 = malloc(0x7fffffff);
	if (p3 == NULL)
	{
		cout << "malloc fail" << endl;
	}
	try
	{
		void *p4 = operator new(0x7fffffff);
	}
	catch (exception &e)
	{
		cout << e.what() << endl;
	}
	return 0;
}
```

![img](https://pic1.zhimg.com/80/v2-ec0379f3a365b22b19bbcd70198c13d8_720w.webp)

4.1.2 operator new和operator delete原理探寻

4.1.2.1 operator new实现原理

operator new：该函数实际通过malloc来申请空间，当malloc申请空间成功时直接返回；申请空间失败，尝试执行空 间不足应对措施，如果改应对措施用户设置了，则继续申请，否则抛异常。

```text
void *__CRTDECL operator new(size_t size) _THROW1(_STD bad_alloc)
{
// try to allocate size bytes
void *p;
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

4.1.2.2 operator delete 实现原理

operator delete: 该函数最终是通过free来释放空间的

```text
void operator delete(void *pUserData)
{
_CrtMemBlockHeader * pHead;
RTCCALLBACK(_RTC_Free_hook, (pUserData, 0));
if (pUserData == NULL)
return;
_mlock(_HEAP_LOCK); /* block other threads */
__TRY
/* get a pointer to memory block header */
pHead = pHdr(pUserData);
/* verify block type */
_ASSERTE(_BLOCK_TYPE_IS_VALID(pHead->nBlockUse));
_free_dbg( pUserData, pHead->nBlockUse );
__FINALLY
_munlock(_HEAP_LOCK); /* release other threads */
__END_TRY_FINALLY
return;
}
```

### **4.2 operator new与operator delete的类专属重载**

4.2.1理解为什么要专属重载

专属重载，听起来很高端的样子。那么重载他们有什么好处呢？为了弄清楚这个问题，我们首先要先了解一个概念，池化技术。

什么叫池化技术，举一个例子，嗯，作为一名贫穷的大学生，生活费的来源是家长。如果我们的生活费是随花随给的，早上你上食堂吃了两根油条喝了一碗豆浆，要付款的时候你大手一挥和食堂大妈说等等，我要找我妈要钱。于是你给你妈妈打了一个电话，妈妈给你转了5块钱，你付了早饭钱（这一切建立在你早上能起床吃早饭的前提下，像我这种，笑死，早饭根本来不及，只能吃午饭了）。嗯，然后午饭你从食堂买了一碗面，再给妈妈打电话要了10元，然后你付了午饭的钱……就这样，你每花一次钱都需要找家长要一次，是不是很麻烦？还特别浪费你和妈妈之间的感情？

但是如果，哎，每个月月初，妈妈给你打1500元，对你说，孩儿啊，你这一个月生活费在这了哈，多退少不补哈。哎，这样你就爽了，油条豆浆随便刷啊，想买啥买啥，等到月底，呵呵，你懂。

池化技术就类似于给你一个月的生活费使你可以在需要花钱的时候自己就可以支付，而不再需要向妈妈要。在池化技术中，分为很多种类的池：

![img](https://pic4.zhimg.com/80/v2-92b7de916de16e4e541d5d5ac005aa33_720w.webp)

池中预存了我们需要的“生活费”，需要时直接自行支付，这样就提高了程序的效率。专属重载operator new和operator delete就是使用内存池进行申请和释放空间，可以提高效率。

![img](https://pic3.zhimg.com/80/v2-785663accc4a6b9fda4ec11ffff4fa8a_720w.webp)

4.2.2 operator new和operator delete 专属重载的实现

针对链表的节点ListNode通过重载类专属 operator new/ operator delete，实现链表节点使用内存池申请和释放内存，提高效率。

```text
#include<iostream>
using namespace std;

struct ListNode
{
	ListNode *_next;
	ListNode *_prev;
	int _val;
	//类中专属的重载
	void* operator new(size_t n)
	{
		void* p = nullptr;
		p = allocator<ListNode>().allocate(1);
		cout << "memory pool allocate" << endl;
		return p;
	}
	void operator delete(void* p)
	{
		allocator<ListNode>().deallocate((ListNode*)p, 1);
		cout << "memory pool deallocate" << endl;
	}
	ListNode(int val = 0)
		:_next(nullptr), _prev(nullptr), _val(val)
	{}
	~ListNode()
	{
		cout << "ListNode 析构" << endl;
	}
};

int main()
{
	ListNode* p = new ListNode(1);
	delete p;

}
```

我们来对比一下使用重载和不使用重载的反汇编代码

![img](https://pic1.zhimg.com/80/v2-4bbc6039438023fadb69d6724421194c_720w.webp)

专属重载还是很香的，毕竟谁不想要一笔可以自由支配的生活费呢？

## 5.new和delete的实现原理

### **5.1内置类型**

如果申请的是内置类型的空间，new和malloc，delete和free基本类似，不同的地方是：new/delete申请和释放的是单个元素的空间，new[]和delete[]申请的是连续空间，而且new在申请空间失败时会抛异常，malloc会返回NULL。

### **5.2自定义类型**

5.2.1 new的原理

调用operator new函数申请空间

在申请的空间上执行构造函数，完成对象的构造

5.2.2 delete的原理

在空间上执行析构函数，完成对象中资源的清理工作

调用operator delete函数释放对象的空间

5.2.3 new T[N]的原理

调用operator new[]函数，在operator new[]中实际调用operator new函数完成N个对象空间的申请。

在申请的空间上执行N次构造函数。

5.2.4 delete[]的原理

在释放的对象空间上执行N次析构函数，完成N个对象中资源的清理

调用operator delete[]释放空间，实际在operator delete[]中调用operator delete来释放空间

## 6.定位new表达式(placement-new)

定位new表达式是在已分配的原始内存空间中调用构造函数初始化一个对象。

### **6.1一个使用场景**

```text
#include<iostream>
using namespace std;

class A
{
public:
	A(int a = 0)
		:_a(a)
	{
		cout << "A" << this << endl;
	}
	~A()
	{
		cout << "~A" << this << endl;
	}
private:
	int _a;
};

int main()
{
	A*p = (A*)malloc(sizeof(A));
	return 0;
}
```

我们看上面的代码，我们申请了一块和A大小相同的空间，那么我们如何来初始化它呢？

这种直接初始化的方法肯定行不通

![img](https://pic3.zhimg.com/80/v2-4019c4140971eae8a14143d0b0f2867e_720w.webp)

我们说，调用构造函数初始化啊，这样可以吗？

![img](https://pic3.zhimg.com/80/v2-deff1c2ecd578b83b330b4610cf7246a_720w.webp)

还是报错的。为什么我们无法调用构造函数呢？

我们来想一下，我们是malloc了一块空间，空间的大小等于A而不是声明了一个A的对象，这块空间不是对象，然后将它强转为A*，所以我们是无法调用构造函数的。

那么我们如何初始化它呢？于是定位new横空出世。

### **6.2 使用格式**

> new (place_address) type或者new (place_address) type(initializer-list)
> place_address必须是一个指针，initializer-list 是类型的初始化列表

### **6.3 定位new的使用**

定位new表达式在实际中一般是配合内存池使用。因为内存池分配出的内存没有初始化，所以如果是自定义类型的对象，需要使用new的定义表达式进行显示调构造函数进行初始化。

```text
#include<iostream>
using namespace std;

class A
{
public:
	A(int a = 0)
		:_a(a)
	{
		cout << "A:" << _a<< endl;
	}
	~A()
	{
		cout << "~A:析构"  << endl;
	}
private:
	int _a;
};

int main()
{
	A*p = (A*)malloc(sizeof(A));
	new(p)A;//显示调用构造函数
	cout << endl;
	new(p)A(3);//显示调用构造函数并赋予初始化的值
	return 0;
}
```

![img](https://pic1.zhimg.com/80/v2-e8faafa1ee1f38bdc51262674f7c0814_720w.webp)

## 7.关于内存的常见问题

### **7.1 malloc和realloc的区别**

malloc/free和new/delete的共同点是：都是从堆上申请空间，并且需要用户手动释放。

不同的地方是：

1. malloc和free是函数，new和delete是操作符
2. malloc申请的空间不会初始化，new可以初始化
3. malloc申请空间时，需要手动计算空间大小并传递，new只需在其后跟上空间的类型即可
4. malloc的返回值为void*, 在使用时必须强转，new不需要，因为new后跟的是空间的类型
5. malloc申请空间失败时，返回的是NULL，因此使用时必须判空,new不需要，但是new需要捕获异常
6. 申请自定义类型对象时，malloc/free只会开辟空间，不会调用构造函数与析构函数，而new在申请空间后会调用构造函数完成对象的初始化，delete在释放空间前会调用析构函数完成空间中资源的清理

### **7.2 内存泄露**

7.2.1 什么是内存泄露，内存泄露的危害是什么？

什么是内存泄漏：内存泄漏指因为疏忽或错误造成程序未能释放已经不再使用的内存的情况。内存泄漏并不是指内存在物理上的消失，而是应用程序分配某段内存后，因为设计错误，失去了对该段内存的控制，因而造成了内存的浪费。

内存泄漏的危害：长期运行的程序出现内存泄漏，影响很大，如操作系统、后台服务等等，出现内存泄漏会导致响应越来越慢，最终卡死。

```text
void MemoryLeaks()
{
// 1.内存申请了忘记释放
int* p1 = (int*)malloc(sizeof(int));
int* p2 = new int;
// 2.异常安全问题
int* p3 = new int[10];
Func(); // 这里Func函数抛异常导致 delete[] p3未执行，p3没被释放.
delete[] p3;
}
```

7.2.1.1思考：内存泄露是指针丢了还是内存丢了？

答案是指针丢了。内存是不会丢的。我们在堆上申请了一块空间，我们拿着这块空间的指针可以访问这块空间。由于我们的疏忽，我们弄丢了指针，导致无法知道指针，无法通过指针来释放这块空间，导致内存泄露。

就像你钥匙丢了，进不去家门，你之所以进不去家门不是因为家丢了，而是钥匙丢了。这里指针就相当于钥匙，内存相当于家。内存是不会丢的，家永远在那里。

所以内存泄露丢的是指针哦。

7.2.1.2 关于申请和释放内存本质的思考

通过上面的问题，我们可以更加清晰的认识到:malloc和new申请一块内存的本质是向系统索要一块空间的使用权，free和delete的本质是当不使用这块空间时将这块空间归还给系统，系统可以再分配给别人。

就像租房，你向主人租了这间房（堆上的一块空间），房主把钥匙（指针）交给你，你可以通过钥匙访问这间房，你不再租这间房了。要把钥匙还给房东（释放指针），房东收回房子，他可以再把房子租给别人（再分配）。

7.2.2 内存泄露的分类

C/C++程序中一般我们关心两种方面的内存泄漏：

7.2.2.1 堆内存泄漏(Heap leak)

堆内存指的是程序执行中依据须要分配通过malloc / calloc / realloc / new等从堆中分配的一块内存，用完后必须通过调用相应的 free或者delete 删掉。假设程序的设计错误导致这部分内存没有被释放，那

么以后这部分空间将无法再被使用，就会产生Heap Leak。

7.2.2.2 系统资源泄漏

指程序使用系统分配的资源，比方套接字、文件描述符、管道等没有使用对应的函数释放掉，导致系统资源的浪费，严重可导致系统效能减少，系统执行不稳定。

### **7.3 如何申请4G的空间**

代码如下：

```text
#include <iostream>
using namespace std;
int main()
{
void* p = new char[0xfffffffful];
cout << "new:" << p << endl;
return 0;
}
```

## 8.后记

好的，这篇万字博客就先肝到这里了，博主在这里对内存的一些知识做了比较全面的介绍，希望对大家有所帮助。其实关于内存管理啊，我觉得重点就在于合理申请，及时释放，将有限的内存空间发挥出最大的价值。

原文地址：https://zhuanlan.zhihu.com/p/426417276

作者：CPP后端技术