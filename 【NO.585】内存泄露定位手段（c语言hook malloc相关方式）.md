# 【NO.585】内存泄露定位手段（c语言hook malloc相关方式）

如何确定有内存泄露问题，如何定位到内存泄露位置，如何写一个内存泄漏检测工具？

## 1.概述

内存泄露本质：其实就是申请调用malloc/new，但是释放调用free/delete有遗漏，或者重复释放的问题。

内存泄露会导致的现象：作为一个服务器，长时间运行，内存泄露会导致进程虚拟内存被占用完，导致进程崩溃吧。（堆上分配的内存）

如何规避或者发现内存泄露呢？

===》1：如何检测有内存泄露？（除了内存监控工具htop，耗时，效果不明显）

===》2：如何定位内存泄露的代码问题（少量代码可以阅读代码排除，线上版本呢？）

=====》引入gc

=====》少量代码可以通过排查代码进行定位

=====》已经确定代码有内存泄露，可以用过valgrind/mtrace等市场上已有的一些工具

=====》本质是malloc和free的次数不一致导致，我们通过hook的方式，对malloc和free次数进行统计

## 2.通过hook的方式检测，定位内存泄露（四种方法）

在生产环境重定位内存泄露的问题，我们可以在产品中增加这些定位手段，通过配置文件开关控制其打开，方便内存泄露定位。

几种不同的方式本质：都是对malloc和free进行hook，增加一些处理进行检测。

### 2.1 测试代码描述内存泄露

如下代码，从代码看，明显可以看到是有内存泄露的，但是如果看不到代码，或者代码量过多，从运行现象上我们就很难发现了。

```text
#include <stdio.h>
#include <stdlib.h>

int main()
{
    void * ptr1 = malloc(10);
    void * ptr2 = malloc(20);
    
    free(ptr1);
    
    void * ptr3 = malloc(30);
    free(ptr3);
    return 0;
}

//代码运行是没有问题，也没有报错的，但是明显可以看到ptr2是没有内存释放的，如果是服务器有这种代码，长时间运行会有严重问题的。
```

### 2.2 通过dlsym库函数对malloc/free进行hook

我在 Linux/unix系统编程手册 这本书中了解相关dlsym函数的使用

要想知道有内存泄露，或者直接定位内存泄露的代码位置，本质还是对调用的malloc/free进行hook, 对调用malloc/free分别增加监控来分析。

使用dlsym库函数，获取malloc/free函数的地址，通过RTLD_NEXT进行比标记（这个标记适用于在其他地方定义的函数同名的包装函数，如在主程序中定义的malloc,代替系统的malloc）,实现用我们主程序中malloc代替系统调用malloc.

**2.2.1：第一版试着**

在调用malloc和free前，使用dlsym函数和RTLD_NEXT标记，获取系统库malloc/free地址，以及用本地定义的malloc/free代替系统调用。

```text
//1:使用void * dlsym(void* handle, char* symbool)函数和handle为RTLD_NEXT标记，对malloc/free进行hook
//2:RTLD_NEXT标记 需要在本地实现 symbool同名函数达到hook功能，即这里要实现malloc/free功能
//3:dlsym()返回的是symbool 参数对应的函数的地址，在同名函数中用该地址实现真正的调用

//RTLD_NEXT 是dlsym() 库中的伪句柄，定义_GNU_SOURCE宏才能识别
//可以通过 man dlsym
//测试发现 ：必须放在最顶部，不然编译报 RTLD_NEXT没有定义 
#define _GNU_SOURCE
#include <dlfcn.h>  //对应的头文件

//第一步功能，确定hook成功，先在我们的hook函数中增加一些打印信息验证
#include <stdio.h>
#include <stdlib.h>

//定义相关全局变量，获取返回的函数地址，进行实际调用
typedef void *(*malloc_t)(size_t size);
malloc_t malloc_f = NULL;

typedef void (*free_t)(void* p);
free_t free_f = NULL;

//要hook的同名函数
void * malloc(size_t size){
    printf("exec malloc \n");
    return malloc_f(size);
}

void free(void * p){
    printf("exec free \n");
    free_f(p);
}
//通过dlsym 对malloc和free使用前进行hook
static void init_malloc_free_hook(){
    //只需要执行一次
    if(malloc_f == NULL){
        malloc_f = dlsym(RTLD_NEXT, "malloc"); //除了RTLD_NEXT 还有一个参数RTLD_DEFAULT
    }
    
    if(free_f == NULL)
    {
        free_f =  dlsym(RTLD_NEXT, "free");
    }
    return ;
}


int main()
{
    init_malloc_free_hook(); //执行一次
    void * ptr1 = malloc(10);
    void * ptr2 = malloc(20);
    
    free(ptr1);
    
    void * ptr3 = malloc(30);
    free(ptr3);
    return 0;
}
```

上述代码是有问题的，现象及定位问题：

```text
hlp@ubuntu:~/mem_test$ gcc dlsym_hook.c -o dlsym_hook -ldl
hlp@ubuntu:~/mem_test$ ./dlsym_hook 
Segmentation fault (core dumped)


#使用gdb对问题进行定位 
hlp@ubuntu:~/mem_test$ gdb ./dlsym_hook 
(gdb) b 54    #加断点
Breakpoint 1 at 0x400729: file dlsym_hook.c, line 54.
(gdb) b 28	  #加断点
Breakpoint 2 at 0x400682: file dlsym_hook.c, line 28.
(gdb) r       #开始运行
Starting program: /home/hlp/mem_test/dlsym_hook 
Breakpoint 1, main () at dlsym_hook.c:54
54	    void * ptr1 = malloc(10);
(gdb) c		#单步执行
Continuing.
Breakpoint 2, malloc (size=10) at dlsym_hook.c:28   #第一个mallocy已经执行
28	    printf("exec malloc \n");	
(gdb) c
Continuing.
Breakpoint 2, malloc (size=1024) at dlsym_hook.c:28  #这里的1024不是我们代码里面的，
28	    printf("exec malloc \n");
(gdb) c
Continuing.

Breakpoint 2, malloc (size=1024) at dlsym_hook.c:28    #发现malloc 1024一直循环执行 怀疑是printf中会调用malloc,
28	    printf("exec malloc \n");
(gdb) c
Continuing.

Breakpoint 2, malloc (size=1024) at dlsym_hook.c:28
28	    printf("exec malloc \n");
(gdb) 

#通过gdb进行定位时，可以确定，我们hook malloc函数内部调用printf,printf底层其实是有调用malloc,从而printf内部成为递归，一直调用了。
#所以我们需要规避这种现象，让hook函数内部其他业务只执行一次，不要因为第三方库内部机制导致类似问题
```

增加特定标识，优化上述代码：

```text
//使用标识，使hook的函数内部只执行一次，不因为第三方库原因导致递归现象
#define _GNU_SOURCE
#include <dlfcn.h>  //对应的头文件
#include <stdio.h>
#include <stdlib.h>

typedef void *(*malloc_t)(size_t size);
malloc_t malloc_f = NULL;
typedef void (*free_t)(void* p);
free_t free_f = NULL;

//定义一个hook函数的标志 使内部逻辑只执行一次
int enable_malloc_hook = 1;
int enable_free_hook = 1;

//要hook的同名函数
void * malloc(size_t size){
    if(enable_malloc_hook) //对第三方调用导致的递归进行规避
    {
        enable_malloc_hook = 0;
        printf("exec malloc \n");
        enable_malloc_hook = 1;
    }
    return malloc_f(size);
}

void free(void * p){
    if(enable_free_hook){
        enable_free_hook = 0;
        printf("exec free \n");
        enable_free_hook = 1;
    }
    free_f(p);
}

//通过dlsym 对malloc和free使用前进行hook
static void init_malloc_free_hook(){
    //只需要执行一次
    if(malloc_f == NULL){
        malloc_f = dlsym(RTLD_NEXT, "malloc"); //除了RTLD_NEXT 还有一个参数RTLD_DEFAULT
    }
    
    if(free_f == NULL)
    {
        free_f =  dlsym(RTLD_NEXT, "free");
    }
    return ;
}
int main()
{
    init_malloc_free_hook(); //执行一次
    void * ptr1 = malloc(10);
    void * ptr2 = malloc(20);
    
    free(ptr1);
    
    void * ptr3 = malloc(30);
    free(ptr3);
    return 0;
}
```

上述代码执行成功，现象如下：

```text
hlp@ubuntu:~/mem_test$ gcc dlsym_hook_ok.c -o dlsym_hook_ok -ldl -g
hlp@ubuntu:~/mem_test$ ./dlsym_hook_ok 
exec malloc 
exec malloc 
exec free 
exec malloc 
exec free
#对比执行的 malloc和free次数 可以确定有内存泄露
```

如何增加行号标识呢？让我们确定到代码位置？

如何确定有内存泄露呢？直接通过代码，识别到malloc/free的对应次数，定位到有代码问题的位置。

**2.2.2：能识别到行号，以及有问题代码位置**

```text
//我们知道，一般可以通过__LINE__ 标识日志当前行号位置，但是这里不适用
//可以通过 __builtin_return_address 获取上级调用的退出的地址，可以设置时1级，也可以设置是2级别...

// 增加打印调用malloc和free位置的信息。 这里打印地址  通过addr2line进行地址和行号转换
#define _GNU_SOURCE
#include <dlfcn.h>  //对应的头文件
#include <stdio.h>
#include <stdlib.h>

typedef void *(*malloc_t)(size_t size);
malloc_t malloc_f = NULL;
typedef void (*free_t)(void* p);
free_t free_f = NULL;

int enable_malloc_hook = 1;
int enable_free_hook = 1;

void * malloc(size_t size){
    if(enable_malloc_hook) //对第三方调用导致的递归进行规避
    {
        enable_malloc_hook = 0;
        //打印上层调用的地址
        
        void *carrer = __builtin_return_address(0);
        printf("exec malloc [%p ]\n", carrer );
        enable_malloc_hook = 1;
    }
    return malloc_f(size);
}

void free(void * p){
    if(enable_free_hook){
        enable_free_hook = 0;
        void *carrer = __builtin_return_address(0);
        printf("exec free [%p]\n", carrer);
        enable_free_hook = 1;
    }
    free_f(p);
}

//通过dlsym 对malloc和free使用前进行hook
static void init_malloc_free_hook(){
    //只需要执行一次
    if(malloc_f == NULL){
        malloc_f = dlsym(RTLD_NEXT, "malloc"); //除了RTLD_NEXT 还有一个参数RTLD_DEFAULT
    }
    
    if(free_f == NULL)
    {
        free_f =  dlsym(RTLD_NEXT, "free");
    }
    return ;
}
int main()
{
    init_malloc_free_hook(); //执行一次
    void * ptr1 = malloc(10);
    void * ptr2 = malloc(20);
    
    free(ptr1);
    
    void * ptr3 = malloc(30);
    free(ptr3);
    return 0;
}
```

执行结果及查找对应行数：

```text
#执行结果如下
hlp@ubuntu:~/mem_test$ gcc dlsym_hook_addr.c -o dlsym_hook_addr -ldl
hlp@ubuntu:~/mem_test$ ./dlsym_hook_addr 
exec malloc [0x400797 ]
exec malloc [0x4007a5 ]
exec free [0x4007b5]
exec malloc [0x4007bf ]
exec free [0x4007cf]
#可以通过addr2line 获取到对应的代码行数 编译的时候要带 -g
hlp@ubuntu:~/mem_test$ addr2line -fe ./dlsym_hook_addr -a 0x400797
0x0000000000400797
main
/home/hlp/mem_test/dlsym_hook_addr.c:57
```

**2.2.3：通过策略，查找有问题的代码**

从上文可以知道，我们通过对malloc和free的hook，可以获得各自malloc和hook的次数。

以及我们可以通过__builtin_return_address 接口获取到实际调用malloc/free的位置。

除此之外，malloc之间有所关联的是申请内存的地址，

汇总：

===》可以思考，通过malloc和free关联的地址作为标识，对malloc和free的次数进行统计即可。

===》可以设计数据结构，对不同地址，malloc的地址和free的地址进行保存，malloc的次数和free的次数进行控制判断

===》这里根据老师的逻辑，用文件的方式进行控制。

测试代码如下：

```text
// malloc和free 之间的关联是申请内存的地址，以该地址作为基准
// malloc时写入一个文件，打印行数等必要信息  free时删除这个文件 通过有剩余文件判断内存泄露
#define _GNU_SOURCE
#include <dlfcn.h>  //对应的头文件
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

typedef void *(*malloc_t)(size_t size);
malloc_t malloc_f = NULL;
typedef void (*free_t)(void* p);
free_t free_f = NULL;

int enable_malloc_hook = 1;
int enable_free_hook = 1;

#define MEM_FILE_LENGTH 40
void * malloc(size_t size){
    if(enable_malloc_hook) //对第三方调用导致的递归进行规避
    {
        enable_malloc_hook = 0;
        //实际的内存申请，根据该地址写文件和free 相互关联
        void *ptr =malloc_f(size);
        //打印上层调用的地址
        void *carrer = __builtin_return_address(0);
        printf("exec malloc [%p ]\n", carrer );
        
        //通过写入文件的方式 对malloc和free进行关联  malloc时写入文件
        char file_buff[MEM_FILE_LENGTH] = {0};
        sprintf(file_buff, "./mem/%p.mem", ptr);
        
        //打开文件写入必要信息 使用前创建目录级别
        FILE *fp = fopen(file_buff, "w");
        fprintf(fp, "[malloc addr : +%p ] ---->mem:%p  size:%lu \n",carrer, ptr, size);
        fflush(fp); //刷新写入文件
        
        enable_malloc_hook = 1;
        return ptr;
    }else
    {
         return malloc_f(size);
    }
}

void free(void * p){
    if(enable_free_hook){
        enable_free_hook = 0;
        void *carrer = __builtin_return_address(0);
        
        //free时删除文件  根据剩余文件判断内存泄露
        char file_buff[MEM_FILE_LENGTH] = {0};
        sprintf(file_buff, "./mem/%p.mem", p);
        //删除文件 根据malloc对应的指针
        if(unlink(file_buff) <0)
        {
            printf("double free: %p, %p \n", p, carrer);
        }
        //这里的打印实际就没意义了
        printf("exec free [%p]\n", carrer);
        free_f(p);
        enable_free_hook = 1;
    }else
    {
        free_f(p);
    }
}

//通过dlsym 对malloc和free使用前进行hook
static void init_malloc_free_hook(){
    //只需要执行一次
    if(malloc_f == NULL){
        malloc_f = dlsym(RTLD_NEXT, "malloc"); //除了RTLD_NEXT 还有一个参数RTLD_DEFAULT
    }
    
    if(free_f == NULL)
    {
        free_f =  dlsym(RTLD_NEXT, "free");
    }
    return ;
}
int main()
{
    init_malloc_free_hook(); //执行一次
    void * ptr1 = malloc(10);
    void * ptr2 = malloc(20);
    
    free(ptr1);
    
    void * ptr3 = malloc(30);
    free(ptr3);
    return 0;
}
```

执行结果：

```text
# 这里的打印只是为了理解  没有多大意义，真正的分析还得依靠文件目录
hlp@ubuntu:~/mem_test$ mkdir mem
hlp@ubuntu:~/mem_test$ gcc dlsym_hook_file.c -o dlsym_hook_file -ldl -g
hlp@ubuntu:~/mem_test$ ./dlsym_hook_file 
exec malloc [0x400ad5 ]
exec malloc [0x400ae3 ]
exec free [0x400af3]
exec malloc [0x400afd ]
exec free [0x400b0d]
hlp@ubuntu:~/mem_test$ cd mem/

# 这里在我们的目标目录下  看到有文件存在，说明存在内存泄露
hlp@ubuntu:~/mem_test/mem$ ls
0xe38680.mem
# 通过文件中的日志信息，对其进行分析，找到问题代码位置
hlp@ubuntu:~/mem_test/mem$ cat 0xe38680.mem 
[malloc addr : +0x400ae3 ] ---->mem:0xe38680  size:20 
hlp@ubuntu:~/mem_test/mem$ cd ../
#通过地址转换 找到我们有问题代码 没有释放的代码位置 
hlp@ubuntu:~/mem_test$ addr2line -fe ./dlsym_hook_file 0x400ae3
main
/home/hlp/mem_test/dlsym_hook_file.c:85
```

### 2.3 通过宏定义的方式对malloc/free进行hook

本质其实就是对系统调用的malloc/free进行替换，调用我们的目标方法，可以通过hook或者重载的方法实现。

使用宏定义的方式，实现malloc/free的替换。

```text
#include <stdio.h>
#include <stdlib.h>

//不能放在这里  放在这里  会对malloc_hook 和 free_hook 内部实际调用的也替换，就形成的递归调用了 并且无法规避
//#define malloc(size) 	malloc_hook(size, __FILE__, __LINE__)
//#define free(p) 		free_hook(p,  __FILE__, __LINE__)

#define MEM_FILE_LENGTH 40
//实现目标函数
void *malloc_hook(size_t size, const char* file, int line)
{
    //这里还是通过文件的方式进行识别
    void *ptr =malloc(size);
    char file_name_buff[MEM_FILE_LENGTH] = {0};
    sprintf(file_name_buff, "./mem/%p.mem", ptr);
        
    //打开文件写入必要信息 使用前创建目录级别
    FILE *fp = fopen(file_name_buff, "w");
    fprintf(fp, "[file:%s  line:%d ] ---->mem:%p  size:%lu \n",file, line, ptr, size);
    fflush(fp); //刷新写入文件
    
    printf("exec malloc [%p:%lu], file: %s, line:%d \n", ptr, size, file, line );
    return ptr;
}

void free_hook(void *p, const char* file, int line)
{
    char file_name_buff[MEM_FILE_LENGTH] = {0};
    sprintf(file_name_buff, "./mem/%p.mem", p);
    
    if(unlink(file_name_buff) <0)
    {
        printf("double free: %p, file: %s. line :%d \n", p, file, line);
    }
    
    //这里的打印实际就没意义了
    printf("exec free [%p], file: %s line:%d \n", p, file, line);
    free(p);
}
//宏定义实现代码中调用malloc/free时调用我们目标函数
#define malloc(size) 	malloc_hook(size, __FILE__, __LINE__)
#define free(p) 		free_hook(p,  __FILE__, __LINE__)

int main()
{
    //init_malloc_free_hook(); //执行一次
    void * ptr1 = malloc(10);
    void * ptr2 = malloc(20);
    
    free(ptr1);
    
    void * ptr3 = malloc(30);
    free(ptr3);
    return 0;
}
```

代码执行如下：

```text
hlp@ubuntu:~/mem_test$ gcc define_hook.c -o define_hook
hlp@ubuntu:~/mem_test$ ./define_hook 
exec malloc [0x1f91010:10], file: define_hook.c, line:47 
exec malloc [0x1f92680:20], file: define_hook.c, line:48 
exec free [0x1f91010], file: define_hook.c line:50 
exec malloc [0x1f938e0:30], file: define_hook.c, line:52 
exec free [0x1f938e0], file: define_hook.c line:53 
#通过目录下文件 以及文件内容  可以分析到代码有问题的行号
#注意  测试前最好清空目录下的文件 
hlp@ubuntu:~/mem_test$ ls ./mem
0x1f92680.mem
hlp@ubuntu:~/mem_test$ cat ./mem/0x1f92680.mem 
[file:define_hook.c  line:48 ] ---->mem:0x1f92680  size:20
```

### 2.4 _libc_malloc

对malloc进行劫持 使用实际的内存申请_libc_malloc 进行申请内存以及其他控制

```text
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
//实际内存申请的函数
extern void *__libc_malloc(size_t size);
int enable_malloc_hook = 1;

extern void __libc_free(void* p);
int enable_free_hook = 1;

// func --> malloc() { __builtin_return_address(0)}
// callback --> func --> malloc() { __builtin_return_address(1)}
// main --> callback --> func --> malloc() { __builtin_return_address(2)}

//calloc, realloc
void *malloc(size_t size) {

	if (enable_malloc_hook) {
		enable_malloc_hook = 0;

		void *p = __libc_malloc(size); //重载达到劫持后 实际内存申请
		void *caller = __builtin_return_address(0); // 0
		
		char buff[128] = {0};
		sprintf(buff, "./mem/%p.mem", p);

		FILE *fp = fopen(buff, "w");
		fprintf(fp, "[+%p] --> addr:%p, size:%ld\n", caller, p, size);
		fflush(fp);

		//fclose(fp); //free
		
		enable_malloc_hook = 1;
		return p;
	} else {
		return __libc_malloc(size);
	}
	
	return NULL;
}


void free(void *p) {
	if (enable_free_hook) {

		enable_free_hook = 0;

		char buff[128] = {0};
		sprintf(buff, "./mem/%p.mem", p);

		if (unlink(buff) < 0) { // no exist
			printf("double free: %p\n", p);
		}
		
		__libc_free(p);

		// rm -rf p.mem
		enable_free_hook = 1;

	} else {
		__libc_free(p);
	}
}

int main()
{
    //init_malloc_free_hook(); //执行一次
    void * ptr1 = malloc(10);
    void * ptr2 = malloc(20);
    
    free(ptr1);
    
    void * ptr3 = malloc(30);
    free(ptr3);
    return 0;
}
```

代码运行及分析如下：

```text
hlp@ubuntu:~/mem_test$ gcc libc_hook.c -o libc_hook -g
hlp@ubuntu:~/mem_test$ ./libc_hook 
#通过查看目录下生成的文件   可以知道有内存泄露 通过日志内部信息  使用addr2line 通过打印地址找到对应的代码位置
hlp@ubuntu:~/mem_test$ cat ./mem/0x733270.mem 
[+0x4009f7] --> addr:0x733270, size:20
hlp@ubuntu:~/mem_test$ addr2line -fe ./libc_hook 0x4009f7
main
/home/hlp/mem_test/libc_hook.c:69   #找到问题代码位置
```

### 2.5 mem_trace

原理：glic提供__malloc_hook, __realloc_hook, __free_hook可以实现hook自定义malloc/free函数

```text
#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>
#include <malloc.h>

	  /* #include <malloc.h>
       void *(*__malloc_hook)(size_t size, const void *caller);
       void *(*__realloc_hook)(void *ptr, size_t size, const void *caller);
       void *(*__memalign_hook)(size_t alignment, size_t size,
                                const void *caller);
       void (*__free_hook)(void *ptr, const void *caller);
       void (*__malloc_initialize_hook)(void);
       void (*__after_morecore_hook)(void);*/
//#include <mcheck.h>

typedef void *(*malloc_hook_t)(size_t size, const void *caller);
typedef void (*free_hook_t)(void *p, const void *caller);

malloc_hook_t 	old_malloc_f = NULL;
free_hook_t 	old_free_f = NULL;
int replaced = 0;

void mem_trace(void);
void mem_untrace(void);

void *malloc_hook_f(size_t size, const void *caller) {

	mem_untrace();
	void *ptr = malloc(size);
	//printf("+%p: addr[%p]\n", caller, ptr);
	char buff[128] = {0};
	sprintf(buff, "./mem/%p.mem", ptr);

	FILE *fp = fopen(buff, "w");
	fprintf(fp, "[+%p] --> addr:%p, size:%ld\n", caller, ptr, size);
	fflush(fp);

	fclose(fp); //free
	mem_trace();
	return ptr;
}

void *free_hook_f(void *p, const void *caller) {

	mem_untrace();
	//printf("-%p: addr[%p]\n", caller, p);

	char buff[128] = {0};
	sprintf(buff, "./mem/%p.mem", p);

	if (unlink(buff) < 0) { // no exist
		printf("double free: %p\n", p);
		return NULL;
	}
	free(p);
	mem_trace();
}
//对__malloc_hook 和__free_hook 重赋值 
void mem_trace(void) { //mtrace
	replaced = 1;      
	old_malloc_f = __malloc_hook; //malloc --> 
	old_free_f = __free_hook;

	__malloc_hook = malloc_hook_f;
	__free_hook = free_hook_f;
}

//还原 __malloc_hook 和__free_hook
void mem_untrace(void) {
	__malloc_hook = old_malloc_f;
	__free_hook = old_free_f;
	replaced = 0;
}

int main()
{
    mem_trace();  //mtrace();    //进行hook劫持
    void * ptr1 = malloc(10);
    void * ptr2 = malloc(20);
    
    free(ptr1);
    
    void * ptr3 = malloc(30);
    free(ptr3);
    mem_untrace();  //muntrace(); //取消劫持
    return 0;
}
```

这里编译的时候有一些警告，但是编译成功了，

除此之外 相关资料可以用 man __malloc_hook 去了解

```text
hlp@ubuntu:~/mem_test$ gcc 3.c -o 3 -g
hlp@ubuntu:~/mem_test$ ./3
hlp@ubuntu:~/mem_test$ cat mem/0x1789030.mem 
[+0x400ab2] --> addr:0x1789030, size:20
hlp@ubuntu:~/mem_test$ addr2line -fe ./3 0x400ab2
main
#可以确定问题代码位置
/home/hlp/mem_test/3.c:79
```

## 3.总结：

内存泄露的本质是，在堆上分配内存，使用malloc/calloc/realloc 以及内存的释放free不匹配导致的。

怎么检测，定位内存泄露？：

===》实际上对内存管理的相关函数进行劫持（hook），增加一些必要信息供我们分析。

===》不同的方案，其实就是劫持(hook的方式不一样)

===》可以使用重载，宏定义，操作系统提供的hook方式（__malloc_hook）等不通的方案

===》在hook的基础上，要进行分析，需要用相关的策略，这里用的多个文件，可以用数据结构管理。

===》除此之外，__builtin_return_address函数可以获取函数调用地址，以及addr2line命令对地址和代码行数进行转换，确定问题代码位置。

===》编译的时候，要加-g,addr2line才可用。

这里只是简单的内存泄露hook的一些方案demo，有关多线程等细节再实际项目中也需要考虑。

原文地址：https://zhuanlan.zhihu.com/p/609107360

作者：linux