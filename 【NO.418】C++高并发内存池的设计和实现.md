# 【NO.418】C++高并发内存池的设计和实现

## 1.整体设计

### **1.1 需求分析**

池化技术是计算机中的一种设计模式，内存池是常见的池化技术之一，它能够有效的提高内存的申请和释放效率以及内存碎片等问题，但是传统的内存池也存在一定的缺陷，高并发内存池相对于普通的内存池它有自己的独特之处，解决了传统内存池存在的一些问题。

1）直接使用new/delete、malloc/free存在的问题

new/delete用于c++中动态内存管理而malloc/free在c++和c中都可以使用，本质上new/delete底层封装了malloc/free。无论是上面的那种内存管理方式，都存在以下两个问题：

效率问题：频繁的在堆上申请和释放内存必然需要大量时间，降低了程序的运行效率。对于一个需要频繁申请和释放内存的程序来说，频繁调用new/malloc申请内存,delete/free释放内存都需要花费系统时间，频繁的调用必然会降低程序的运行效率。

内存碎片：经常申请小块内存，会将物理内存“切”得很碎，导致内存碎片。申请内存的顺序并不是释放内存的顺序，因此频繁申请小块内存必然会导致内存碎片，造成“有内存但是申请不到大块内存”的现象。

2）普通内存池的优点和缺点

针对直接使用new/delete、malloc/free存在的问题，普通内存池的设计思路是：预先开辟一块大内存，程序需要内存时直接从该大块内存中“拿”一块，提高申请和释放内存的效率，同时直接分配大块内存还减少了内存碎片问题。

优点：申请和释放内存的效率有所提高；一定程度上解决了内存碎片问题。

缺点：多线程并发场景下申请和释放内存存在锁竞争问题造成申请和释放内存的效率降低。

3）高并发内存池要解决的问题

基于以上原因，设计高并发内存池需要解决以下三个问题：

- 效率问题
- 内存碎片问题
- 多线程并发场景下的内存释放和申请的锁竞争问题。

### **1.2 总体设计思路**

高并发内存池整体框架由以下三部分组成，各部分的功能如下：

- 线程缓存（thread cache）：每个线程独有线程缓存，主要解决多线程下高并发运行场景线程之间的锁竞争问题。线程缓存模块可以为线程提供小于64k内存的分配，并且多个线程并发运行不需要加锁。
- 中心控制缓存（central control cache）：中心控制缓存顾名思义，是高并发内存池的中心结构主要用来控制内存的调度问题。负责大块内存切割分配给线程缓存以及回收线程缓存中多余的内存进行合并归还给页缓存，达到内存分配在多个线程中更均衡的按需调度的目的，它在整个项目中起着承上启下的作用。（注意：这里需要加锁，当多个线程同时向中心控制缓存申请或归还内存时就存在线程安全问题，但是这种情况是极少发生的，并不会对程序的效率产生较大的影响，总体来说利大于弊）
- 页缓存（page cache）：以页为单位申请内存，为中心控制缓存提供大块内存。当中心控制缓存中没有内存对象时，可以从page cache中以页为单位按需获取大块内存，同时page cache还会回收central control cache的内存进行合并缓解内存碎片问题。

![img](https://pic1.zhimg.com/80/v2-628fc6858e8a2a1b89e64da8b66dec04_720w.webp)

### **1.3 申请内存流程图**

![img](https://pic2.zhimg.com/80/v2-c3900354c553e3f7c274128920e5f5e1_720w.webp)

## 2.详细设计

### **2.1 各个模块内部结构详细剖析**

1）thread cache

逻辑结构设计

thread cache的主要功能就是为每一个线程提供64K以下大小内存的申请。为了方便管理，需要提供一种特定的管理模式，来保存未分配的内存以及被释放回来的内存，以方便内存的二次利用。这里的管理通常采用将不同大小的内存映射在哈希表中，链接起来。而内存分配的最小单位是字节，64k = 1024*64Byte如果按照一个字节一个字节的管理方式进行管理，至少也得需要1024*64大小的哈希表对不同大小的内存进行映射。为了减少哈希表长度，这里采用按一定数字对齐的方式进行内存分配，将浪费率保持在1%~12%之间。具体结构如下：

![img](https://pic2.zhimg.com/80/v2-2043a11ccafd756ab3c578ef87f6ce91_720w.webp)

具体说明如下：

- 使用数组进行哈希映射，每一个位置存放的是一个链表freelists，该链表的作用是将相同大小的内存对象链接起来方便管理。
- 每个数组元素链接的都是不同大小的内存对象。
- 第一个元素表示对齐数是8，第2个是16....依次类推。对齐数表示在上一个对齐数和这个对齐数之间的大小的内存都映射在这个位置，即要申请1字节或者7字节的内存都在索引位置为0出找8字节大小的内存，要申请9~16字节大小的内存都在索引为1的位置找16字节的内存对象。
- 通过上面的分析，可以看出如果进行8字节对齐，最多会浪费7字节的内存（实际申请1字节内存，返回的是8字节大小的内存对象）,将这种现象称为内存碎片浪费。
- 为了将内存碎片浪费保持在12%以下，也就是说最多容忍有12%的内存浪费，这里使用不同的对齐数进行对齐。
- 0~128采用8字节对齐，129~1024采用16字节对齐，1025~8*1024采用128字节对齐，8*1024~64*1024采用1024字节对齐；内存碎片浪费率分别为：1/8，129/136，1025/1032，8102/8199均在12%左右。同时，8字节对齐时需要[0,15]共16个哈希映射；16字节对齐需要[16,71]共56个哈希映射；128字节对齐需要[72,127]共56个哈希映射;1024字节对齐需要[128,184]共56个哈希映射。
- 哈希映射的结构如下：

![img](https://pic4.zhimg.com/80/v2-d404d653a18a6cb2dc20687fc052d627_720w.webp)

如何保证每个线程独有？

大于64k的内存如何申请？

当thread cache中申请的内存大于64K时，直接向page cache申请。但是page cache中最大也只能申请128页的内存，所以当thread cache申请的内存大于128页时page cache中会自动给thread cache在系统内存中申请。

2）central control cache

central control cache作为thread cache和page cache的沟通桥梁，起到承上启下的作用。它需要向thread cache提供切割好的小块内存，同时他还需要回收thread cache中的多余内存进行合并，在分配给其他其他thread cache使用，起到资源调度的作用。它的结构如下：

![img](https://pic2.zhimg.com/80/v2-29c6886f3dcf57c1acd2143d1bb29709_720w.webp)

具体说明如下：

- central control cache的结构依然是一个数组，他保存的是span类型的对象。
- span是用来管理一块内存的，它里边包含了一个freelist链表，用于将大块内存切割成指定大小的小块内存链接到freelist中，当thread cache需要内存时直接将切割好的内存给thread cache。
- 开始时，每个数组索引位置都是空的，当thread cache申请内存时，spanList数组会向page cache申请一大块内存进行切割后挂在list中。当该快内存使用完，会继续申请新的内存，因此就存在多个span链接的情况。前边span存在对象是因为有可能后边已经申请好内存了前边的内存也释放回来了。
- 当某一个span的全部内存都还回来时，central control cache会再次将这块内存合并，在归还到page cache中。
- 当central control cache为空时，向page cache申请内存，每次至少申请一页，并且必须以页为单位进行申请（这里的页大小由我们自己决定，这里采用4K）。

这里需要注意的是，thread cache可能会有多个，但是central control cache只有一个，要让多个thread cache对象访问一个central control cache对象，这里的central control cache需要设计成单例模式。

3）page cache

page cache是以页为单位进行内存管理的，它是将不同页数的内存利用哈希进行映射，最多映射128页内存，具体结构如下：

![img](https://pic2.zhimg.com/80/v2-ef6ef4f51a86de058ece30b01247c7f5_720w.webp)

page Cache申请和释放内存流程：

- 当central control cache向page cache申请内存时，比如要申请8页的内存，它会先在span大小为8的位置找，如果没有就继续找9 10...128，那个有就从那个中切割8页。
- 例如，走到54时才有内存，就从54处切8页返回给central control cache，将剩余的54-846页挂在46页处。
- 当page cache中没有内存时，它直接申请一个128页的内存挂在128位置。当central control cache申请内存时再从128页切。

### **2.2 设计细节**

1）thread cache

根据申请内存大小计算对应的_freelists索引

- 1~8都映射在索引为0处，9~16都在索引为2处......
- 因此以8字节对齐时，可以表示为：((size + (2^3 - 1)) >> 3) - 1;
- 如果申请的内存为129，索引如何计算？
- 首先前128字节是按照8字节对齐的，因此：((129-128）+（2^4-1))>>4)-1 + 16
- 上式中16表示索引为0~15的16个位置以8字节对齐。

代码实现：

```text
//根据内存大小和对齐数计算对应下标
static inline size_t _Intex(size_t size, size_t alignmentShift)
{
	//alignmentShift表示对齐数的位数，例如对齐数为8 = 2^3时，aligmentShift = 3
	//这样可以将除法转化成>>运算，提高运算效率
	return ((size + (1 << alignmentShift) - 1) >> alignmentShift) - 1;
}
//根据内存大小，计算对应的下标
static inline size_t Index(size_t size)
{
	assert(size <= THREAD_MAX_SIZE);
 
	//每个对齐数对应的索引个数，分别表示8 16 128 1024字节对齐
	int groupArray[4] = {16,56,56,56};
 
	if (size <= 128)
	{
		//8字节对齐
		return _Intex(size, 3) + groupArray[0];
	}
	else if (size <= 1024)
	{
		//16字节对齐
		return _Intex(size, 4) + groupArray[1];
	}
	else if (size <= 8192)
	{
		//128字节对齐
		return _Intex(size, 7) + groupArray[2];
	}
	else if (size <= 65536)
	{
		//1024字节对齐
		return _Intex(size, 10) + groupArray[3];
	}
 
	assert(false);
	return -1;
}
```

freelist向中心缓存申请内存时需要对申请的内存大小进行对齐

首先，需要申请的内存大小不够对齐数时都需要进行向上对齐。即，要申请的内存大小为1字节时需要对齐到8字节。如何对齐？不进行对齐可以吗？

首先，不进行对齐也可以计算出freelist索引，当第一次申请内存时，freelist的索引位置切割后的内存大小就是实际申请的内存大小，并没有进行对齐，造成内存管理混乱。对齐方式如下：

- 对齐数分别为8 = 2^3; 16 = 2^4 ; 128 = 2^7 ; 1024 = 2^10,转化成二进制后只有1个1.
- 在对齐区间内，所有数+对齐数-1后一定是大于等于当前区间的最大值且小于下一个相邻区间的最大值。
- 因此，size + 对齐数 - 1如果是8字节对齐只需将低3位变为0，如果是16字节对齐将低3位变为0......
- 例如：size = 2时，对齐数为8；则size + 8 - 1 = 9，转为而进制位1001，将低三位变为0后为1000，转为十进制就是对齐数8.

> 代码表示如下：alignment表示对齐数
> (size + alignment - 1) & ~(alignment - 1);

注意：向这些小函数，定义成inline可以减少压栈开销。 ‘

如何将小块内存对象“挂在”freelist链表中

哈哈，前边已经为这里做好铺垫了。前边规定单个对象大小最小为8字节，32位系统下一个指针的大小为4字节，64位机器下一个指针的大小为8字节。前边我们规定单个对象最小大小为8字节就是为了无论是在32位系统下还是在64位系统下，都可以保存一个指针将小块对象链接起来。那么，如何使用一小块内存保存指针？

直接在内存的前4/8个字节将下一块内存的地址保存，取内存时直接对该内存解引用就可以取出地址。

> 访问：*(void**)(mem)

每次从freelist中取内存或者归还内存时，直接进行头插或头删即可。

从central control cache中申请内存，一次申请多少合适呢？

这里的思路是采用“慢启动”的方式申请，即第一次申请申请一个，第二次申请2个....当达到一定大小（512个）时不再增加。这样做的好处是，第一次申请给的数量少可以防止某些线程只需要一个多给造成浪费，后边给的多可以减少从central control cache的次数从而提高效率。

当使用慢启动得到的期望内存对象个数大于当前central control cache中内存对象的个数时，有多少给多少。因为，实际上目前只需要一个，我们多申请了不够，那就有多少给多少。当一个都没有的时候才会去page cache申请。

什么时候thread cache将内存还给central controlcache？

当一个线程将内存还给thread cache时，会去判断对应的_freelist的对应位置是否有太多的内存还回来（thread cache中内存对象的大小大于等于最个数的时候，就向central control cache还）。

2）Central Control Cache

SpanList结构

SpanList在central control cache中最重要的作用就是对大块内存管理，它存储的是一个个span类的对象，使用链表进行管理。结构如下：

![img](https://pic1.zhimg.com/80/v2-4c4a4faff188ba0443cba763223943dc_720w.webp)

也就是说，SpanList本质上就是一个span链表。这里考虑到后边归还内存需要找到对应页归还，方便插入，这里将spanlist设置成双向带头循环链表。

Span结构

Span存储的是大块内存的信息，陪SpanList共同管理大块内存，它的内存单位是页（4K）。它的结构实际上就是一个个size大小的对象链接起来的链表。它同时也作为SpanList的节点，spanList是双向循环链表，因此span中还有next和prev指针。

![img](https://pic2.zhimg.com/80/v2-4bfb9c366c319885a613a490adff6831_720w.webp)

```text
struct Span
{
    PageID _pageId = 0;   // 页号
    size_t _n = 0;        // 页的数量
    Span* _next = nullptr;
    Span* _prev = nullptr;
    void* _list = nullptr;  // 大块内存切小链接起来，这样回收回来的内存也方便链接
    size_t _usecount = 0;    // 使用计数，==0 说明所有对象都回来了
    size_t _objsize = 0;    // 切出来的单个对象的大小
};
```

当spanList中没有内存时需要向PageCache申请内存，一次申请多少合适呢？

根据申请的对象的大小分配内存，也就是说单个对象大小越小分配的页数越少，单个对象的大小越大分配到的内存越多。如何衡量多少？

这里我们是通过thread cache中从central control cache中获取的内存对象的个数的上限来确定。也就是说，个数的上限*内存对象的大小就是我们要申请的内存的大小。在右移12位（1页）就是需要申请的页数。

```text
//计算申请多少页内存
static inline size_t NumMovePage(size_t memSize)
{
	//计算thread cache最多申请多少个对象，这里就给多少个对象
	size_t num = NumMoveSize(memSize);
	//此时的nPage表示的是获取的内存大小
	size_t nPage = num*memSize;
	//当npage右移是PAGE_SHIFT时表示除2的PAGE_SHIFT次方，表示的就是页数
	nPage >>= PAGE_SHIFT;
 
	//最少给一页（体现了按页申请的原则）
	if (nPage == 0)
		nPage = 1;
 
	return nPage;
}
```

向central control cache申请一块内存，切割时如果最后产生一个碎片（不够一个对象大小的内存）如何处理？

一旦产生这种情况，最后的碎片内存只能丢弃不使用。但是对于我们的程序来说是不会产生的，因为我们每次申请至少一页，4096可以整除我们所对应的任何一个大小的对象。

central control cache何时将内存还给page cache？

thread cache将多余的内存会还给central control cache中的spanlist对应的span，span中有一个usecount用来统计该span中有多少个对象被申请走了，当usecount为0时，表示所有对象都还回来了，则将该span还给page cache，合并成更大的span。

3）Page Cache

当从一个大页切出一个小页内存时，剩余的内存如何挂在对应位置？

在Page cache中的span它是没有切割的，都是一个整页，也就是说这里的Span的list并没有使用到。这里计算内存的地址都是按照页号计算的，当一个Span中有多页内存时保存的是第一页的内存，那么就可以计算出剩余内存和切走内存的页号，设置相应的页号进行映射即可。

从一个大的Span中切时，采用头切还是尾切？

Span中如何通过页号计算地址？

每一页大小都是固定的，当我们从系统申请一块内存会返回该内存的首地址，申请内存时返回的都是一块连续的内存，所以我们可以使用内存首地址/页大小的方式计算出页号，通过这种方式计算出来的一大块内存的多个页的页号都是连续的。

![img](https://pic2.zhimg.com/80/v2-74916881fbcff5ed1f31d1701fcffe75_720w.webp)

Page Cache向系统申请内存

Page Cache向系统申请内存时，前边我们说过每次直接申请128页的内存。这里需要说明的是，我们的项目中不能出现任和STL中的数据结构和库函数，因此这里申请内存直接采用系统调用VirtualAlloc。下面对VirtualAlloc详细解释：

VirtualAlloc是一个Windows API函数，该函数的功能是在调用进程的虚地址空间,预定或者提交一部分页。简单点的意思就是申请内存空间。

函数声明如下：

```text
LPVOID VirtualAlloc{
LPVOID lpAddress, // 要分配的内存区域的地址
DWORD dwSize, // 分配的大小
DWORD flAllocationType, // 分配的类型
DWORD flProtect // 该内存的初始保护属性
};
```

参数说明：

- LPVOID lpAddress, 分配内存区域的地址。当你使用VirtualAlloc来提交一块以前保留的内存块的时候，lpAddress参数可以用来识别以前保留的内存块。如果这个参数是NULL，系统将会决定分配内存区域的位置，并且按64-KB向上取整(roundup)。
- SIZE_T dwSize, 要分配或者保留的区域的大小。这个参数以字节为单位，而不是页，系统会根据这个大小一直分配到下页的边界DWORD
- flAllocationType, 分配类型 ,你可以指定或者合并以下标志：MEM_COMMIT，MEM_RESERVE和MEM_TOP_DOWN。
- DWORD flProtect 指定了被分配区域的访问保护方式

注：PageCache中有一个map用来存储pageId和Span的映射。在释放内存时，通过memSize计算出pageId，在通过PageId在map中查找对应的Span从而就可以获得单个对象的大小，在根据单个对象的大小确定是要将内存还给page cache还是还给central control cache。

central control cache释放回来的内存如何合并成大内存？

通过span中的页号查找前一页和后一页，判断前一页和后一页是否空闲（没有被申请的内存），如果空闲就进行和并，合并完后重新在map中进行映射。

注意：将PageCache和CentralControlCache设置成单例模式，因为多个线程对同时使用一个page cache和central control cache进行内存管理。

单例模式简单介绍

- 单例模式，顾名思义只能创建一个实例。
- 有两种实现方式：懒汉实现和饿汉实现
- 做法：将构造函数和拷贝构造函数定义成私有且不能默认生成，防止在类外构造对象；定义一个本身类型的成员，在类中构造一个对象，提供接口供外部调用。

4）加锁问题

- 在central control cache和page cache中都存在多个线程访问同一临界资源的情况，因此需要加锁。
- 在central control cache中，不同线程只要访问的不是同一个大小的内存对象，则就不需要加锁，可以提高程序的运行效率（加锁后就有可能导致线程挂起等待），也就是说central control cache中是“桶锁”。需要改freelist那个位置的内存，就对那个加锁。
- page cache中，需要对申请和合并内存进行加锁。
- 这里我们统一使用互斥锁。

注意：使用map进行映射，虽然说我们对pagecache进行了加锁，不会早成写数据的冲突，但是我们还向外提供了查找的接口，就有可能导致一个线程在向map中写而另一个线程又查找，出现线程安全问题，但是如果给查找位置加锁，这个接口会被频繁的调用，造成性能的损失。而在tcmalloc中采用基数树来存储pageId和span的映射关系，从而提高效率。

## 3.测试

### 3.1 单元测试

```text
void func1()
{
	for (size_t i = 0; i < 10; ++i)
	{
		hcAlloc(17);
	}
}
 
void func2()
{
	for (size_t i = 0; i < 20; ++i)
	{
		hcAlloc(5);
	}
}
 
//测试多线程
void TestThreads()
{
	std::thread t1(func1);
	std::thread t2(func2);
 
 
	t1.join();
	t2.join();
}
 
//计算索引
void TestSizeClass()
{
	cout << SizeClass::Index(1035) << endl;
	cout << SizeClass::Index(1025) << endl;
	cout << SizeClass::Index(1024) << endl;
}
 
//申请内存
void TestConcurrentAlloc()
{
	void* ptr0 = hcAlloc(5);
	void* ptr1 = hcAlloc(8);
	void* ptr2 = hcAlloc(8);
	void* ptr3 = hcAlloc(8);
 
	hcFree(ptr1);
	hcFree(ptr2);
	hcFree(ptr3);
}
 
//大块内存的申请
void TestBigMemory()
{
	void* ptr1 = hcAlloc(65 * 1024);
	hcFree(ptr1);
 
	void* ptr2 = hcAlloc(129 * 4 * 1024);
	hcFree(ptr2);
}
 
//int main()
//{
//	//TestBigMemory();
//
//	//TestObjectPool();
//	//TestThreads();
//	//TestSizeClass();
//	//TestConcurrentAlloc();
//
//	return 0;
//}
```

### 3.2 性能测试

```text
void BenchmarkMalloc(size_t ntimes, size_t nworks, size_t rounds)
{
	//创建nworks个线程
	std::vector<std::thread> vthread(nworks);
	size_t malloc_costtime = 0;
	size_t free_costtime = 0;
 
	//每个线程循环依次
	for (size_t k = 0; k < nworks; ++k)
	{
		//铺货k
		vthread[k] = std::thread([&, k]() {
			std::vector<void*> v;
			v.reserve(ntimes);
 
			//执行rounds轮次
			for (size_t j = 0; j < rounds; ++j)
			{
				size_t begin1 = clock();
				//每轮次执行ntimes次
				for (size_t i = 0; i < ntimes; i++)
				{
					v.push_back(malloc(16));
				}
				size_t end1 = clock();
 
				size_t begin2 = clock();
				for (size_t i = 0; i < ntimes; i++)
				{
					free(v[i]);
				}
				size_t end2 = clock();
				v.clear();
 
				malloc_costtime += end1 - begin1;
				free_costtime += end2 - begin2;
			}
		});
	}
 
	for (auto& t : vthread)
	{
		t.join();
	}
 
	printf("%u个线程并发执行%u轮次，每轮次malloc %u次: 花费：%u ms\n",
		nworks, rounds, ntimes, malloc_costtime);
 
	printf("%u个线程并发执行%u轮次，每轮次free %u次: 花费：%u ms\n",
		nworks, rounds, ntimes, free_costtime);
 
	printf("%u个线程并发malloc&free %u次，总计花费：%u ms\n",
		nworks, nworks*rounds*ntimes, malloc_costtime + free_costtime);
}
 
 
// 单轮次申请释放次数 线程数 轮次
void BenchmarkConcurrentMalloc(size_t ntimes, size_t nworks, size_t rounds)
{
	std::vector<std::thread> vthread(nworks);
	size_t malloc_costtime = 0;
	size_t free_costtime = 0;
 
	for (size_t k = 0; k < nworks; ++k)
	{
		vthread[k] = std::thread([&]() {
			std::vector<void*> v;
			v.reserve(ntimes);
 
			for (size_t j = 0; j < rounds; ++j)
			{
				size_t begin1 = clock();
				for (size_t i = 0; i < ntimes; i++)
				{
					v.push_back(hcAlloc(16));
				}
				size_t end1 = clock();
 
				size_t begin2 = clock();
				for (size_t i = 0; i < ntimes; i++)
				{
					hcFree(v[i]);
				}
				size_t end2 = clock();
				v.clear();
 
				malloc_costtime += end1 - begin1;
				free_costtime += end2 - begin2;
			}
		});
	}
 
	for (auto& t : vthread)
	{
		t.join();
	}
 
	printf("%u个线程并发执行%u轮次，每轮次concurrent alloc %u次: 花费：%u ms\n",
		nworks, rounds, ntimes, malloc_costtime);
 
	printf("%u个线程并发执行%u轮次，每轮次concurrent dealloc %u次: 花费：%u ms\n",
		nworks, rounds, ntimes, free_costtime);
 
	printf("%u个线程并发concurrent alloc&dealloc %u次，总计花费：%u ms\n",
		nworks, nworks*rounds*ntimes, malloc_costtime + free_costtime);
}
 
int main()
{
	cout << "==========================================================" << endl;
	BenchmarkMalloc(100000, 4, 10);
	cout << endl << endl;
 
	BenchmarkConcurrentMalloc(100000, 4, 10);
	cout << "==========================================================" << endl;
 
	return 0;
}
```

结果比较

![img](https://pic2.zhimg.com/80/v2-bc38cb0b9978c69b6136767090a6182d_720w.webp)

附：[完整代码](https://link.zhihu.com/?target=https%3A//github.com/MouCoder/cpp_Code/commit/a4d2cd3676b0a866a14e3cf17a90998503208d8a)

原文地址：https://zhuanlan.zhihu.com/p/389658892

作者：linux