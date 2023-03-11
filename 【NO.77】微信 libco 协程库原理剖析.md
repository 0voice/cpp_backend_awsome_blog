# 【NO.77】微信 libco 协程库原理剖析

> 同 Go 语言一样，libco 也是提供了同步风格编程模式，同时还能保证系统的高并发能力，本文主要剖析 libco 中的协程原理。

## **0.简介**

- libco 是微信后台大规模使用的 c/c++协程库，2013 年至今稳定运行在微信后台的数万台机器上。
- libco 通过仅有的几个函数接口 co_create/co_resume/co_yield 再配合 co_poll，可以支持同步或者异步的写法，如线程库一样轻松。同时库里面提供了 socket 族函数的 hook，使得后台逻辑服务几乎不用修改逻辑代码就可以完成异步化改造。
- 开源地址：https://github.com/Tencent/libco

## 1.**准备知识**

### 1.1 协程是什么

- 协程本质上就是用户态线程，又名纤程，将调度的代码在用户态重新实现。有极高的执行效率，因为子程序切换不是线程切换而是由程序自身控制，没有线程切换的开销。协程通常是纯软件实现的多任务，与 CPU 和操作系统通常没有关系，跨平台，跨体系架构。
- 协程在执行过程中，可以调用别的协程自己则中途退出执行，之后又从调用别的协程的地方恢复执行。这有点像操作系统的线程，执行过程中可能被挂起，让位于别的线程执行，稍后又从挂起的地方恢复执行。
- 对于线程而言，其上下文切换流程如下，需要两次权限等级切换和三次栈切换。上下文存储在内核栈上。线程的上下文切换必须先进入内核态并切换上下文, 这就造成了严重的调度开销。线程的结构体存在于内核中，在 pthread_create 时需要进入内核态，频繁创建开销大。

### 1.2 Linux 程序内存布局

Linux 使用虚拟地址空间，大大增加了进程的寻址空间，由低地址到高地址分别为：

- 只读段/代码段：只能读，不可写；可执行代码、字符串字面值、只读变量
- 数据段：已初始化且初值非 0 全局变量、静态变量的空间
- BSS 段：未初始化或初值为 0 的全局变量和静态局部变量
- 堆 ：就是平时所说的动态内存， malloc/new 大部分都来源于此。
- 文件映射区域 ：如动态库、共享内存等映射物理空间的内存，一般是 mmap 函数所分配的虚拟地址空间。
- 栈：用于维护函数调用的上下文空间；局部变量、函数参数、返回地址等
- 内核虚拟空间：用户代码不可见的内存区域，由内核管理(页表就存放在内核虚拟空间)。

其中需要注意的是：栈和堆的这两种不同的地址增长方向，栈从高到低地址增长。堆从低到高增长，后面协程切换中就涉及到该布局的不同。

### 1.3 栈帧

栈帧是从栈上分配的一段内存，每次函数调用时，用于存储自动变量。从物理介质角度看，栈帧是位于 esp（栈指针）及 ebp（基指针）之间的一块区域。每个栈帧对应着一个未运行完的函数。栈帧中保存了该函数的函数参数、返回地址和局部变量等数据。局部变量等分配均在栈帧上分配，函数结束自动释放。

- ESP：栈指针寄存器，指向当前栈帧的栈顶。
- EBP：基址指针寄存器，指向当前栈帧的底部。

C 函数调用，调用者将一些参数放在栈上，调用函数，然后弹出栈上存放的参数。这里涉及调用约定，调用约定涉及参数的入栈顺序（从左到右还是从右到左）、参数入栈和清理的是调用者(caller)还是被调用者(callee)，函数名的处理。

- 采用**cdecl 调用约定的调用者会将参数从右到左的入栈，最后将返回地址入栈。这个返回地址是指，函数调用结束后的下一行执行的代码地址。（**cdecl is the default calling convention for C and C++ programs. Because the stack is cleaned up by the caller, it can do vararg functions. The __cdecl calling convention creates larger executables than __stdcall, because it requires each function call to include stack cleanup code. The following list shows the implementation of this calling convention. The __cdecl modifier is Microsoft-specific.）

## **2.关键数据结构**

libco 的协程控制块 stCoRoutine_t：

```
struct stCoRoutine_t{ stCoRoutineEnv_t *env; pfn_co_routine_t pfn; void *arg; coctx_t ctx; char cStart; char cEnd; char cIsMain; char cEnableSysHook; char cIsShareStack; void *pvEnv; //char sRunStack[ 1024 * 128 ]; stStackMem_t* stack_mem; //save stack buffer while confilct on same stack_buffer; char* stack_sp; unsigned int save_size; char* save_buffer; stCoSpec_t aSpec[1024];};
```

- env：即协程执行的环境，libco 协程一旦创建便跟对应线程绑定了，不支持在不同线程间迁移，这里 env 即同属于一个线程所有协程的执行环境，包括了当前运行协程、嵌套调用的协程栈，和一个 epoll 的封装结构。这个结构是跟运行的线程绑定了的，运行在同一个线程上的各协程是共享该结构的，是个全局性的资源。

```
struct stCoRoutineEnv_t{ stCoRoutine_t *pCallStack[ 128 ]; int iCallStackSize; stCoEpoll_t *pEpoll; //for copy stack log lastco and nextco stCoRoutine_t* pending_co; stCoRoutine_t* occupy_co;};
```

- pfn：实际等待执行的协程函数
- arg：上面协程函数的参数
- ctx：上下文，即 ESP、EBP、EIP 和其他通用寄存器的值

```
struct coctx_t{#if defined(__i386__) void *regs[ 8 ];#else void *regs[ 14 ];#endif size_t ss_size; char *ss_sp;};
```

- cStart、cEnd、cIsMain、cEnableSysHook、cIsShareStack：一些状态和标志变量，后面会细说
- pvEnv：保存程序系统环境变量的指针
- stack_mem：协程运行时的栈内存，这个栈内存是固定的 128KB 的大小。

```
struct stStackMem_t{ stCoRoutine_t* occupy_co; int stack_size; char* stack_bp; //stack_buffer + stack_size char* stack_buffer;};
```

stack_sp、save_size、save_buffer：这里要提到实现 stackful 协程（与之相对的还有一种 stackless 协程）的两种技术：Separate coroutine stacks 和 Copying the stack（又叫共享栈）。这三个变量就是用来实现这两种技术的。

实现细节上，前者为每一个协程分配一个单独的、固定大小的栈；而后者则仅为正在运行的协程分配栈内存，当协程被调度切换出去时，就把它实际占用的栈内存 copy 保存到一个单独分配的缓冲区；当被切出去的协程再次调度执行时，再一次 copy 将原来保存的栈内存恢复到那个共享的、固定大小的栈内存空间。

如果是独享栈模式，分配在堆中的一块作为当前协程栈帧的内存 stack_mem，这块内存的默认大小为 128K。

如果是共享栈模式，协程切换的时候，用来拷贝存储当前共享栈内容的 save_buffer，长度为实际的共享栈使用长度。

通常情况下，一个协程实际占用的（从 esp 到栈底）栈空间，相比预分配的这个栈大小（比如 libco 的 128KB）会小得多；这样一来， copying stack 的实现方案所占用的内存便会少很多。当然，协程切换时拷贝内存的开销有些场景下也是很大的。因此两种方案各有利弊，而 libco 则同时实现了两种方案，默认使用前者，也允许用户在创建协程时指定使用共享栈。

## **3.生命周期**

### 3.1 创建协程 Create coroutine

调用 co_create 将协程创建出来后，这时候它还没有启动，也即是说我们传递的 routine 函数还没有被调用。实质上，这个函数内部仅仅是分配并初始化 stCoRoutine_t 结构体、设置任务函数指针、分配一段“栈”内存，以及分配和初始化 coctx_t。

- ppco：输出参数，co_create 内部为新协程分配一个协程控制块，ppco 将指向这个分配的协程控制块。
- attr：指定要创建协程的属性（栈大小、指向共享栈的指针（使用共享栈模式））
- pfn：协程的任务（业务逻辑）函数
- arg：传递给任务函数的参数

```
int co_create( stCoRoutine_t **ppco,const stCoRoutineAttr_t *attr,pfn_co_routine_t pfn,void *arg ){ if( !co_get_curr_thread_env() ) {  co_init_curr_thread_env(); } stCoRoutine_t *co = co_create_env( co_get_curr_thread_env(), attr, pfn,arg ); *ppco = co; return 0;}
```

### 3.2 启动协程 Resume coroutine

在调用 co_create 创建协程返回成功后，便可以调用 co_resume 函数将它启动了。

取当前协程控制块指针，将待启动的协程压入 pCallStack 栈，然后 co_swap 切换到指向的新协程上取执行，co_swap 不会就此返回，而是要等当前执行的协程主动让出 cpu 时才会让新的协程切换上下文来执行自己的内容。

```
void co_resume( stCoRoutine_t *co ){ stCoRoutineEnv_t *env = co->env; stCoRoutine_t *lpCurrRoutine = env->pCallStack[ env->iCallStackSize - 1 ]; if( !co->cStart ) {  coctx_make( &co->ctx,(coctx_pfn_t)CoRoutineFunc,co,0 );  co->cStart = 1; } env->pCallStack[ env->iCallStackSize++ ] = co; co_swap( lpCurrRoutine, co );}
```

### 3.3 挂起协程 Yield coroutine

在非对称协程理论，yield 与 resume 是个相对的操作。A 协程 resume 启动了 B 协程，那么只有当 B 协程执行 yield 操作时才会返回到 A 协程。在上一节剖析协程启动函数 co_resume() 时，也提到了该函数内部 co_swap() 会执行被调协程的代码。只有被调协程 yield 让出 CPU，调用者协程的 co_swap() 函数才能返回到原点，即返回到原来 co_resume() 内的位置。

在被调协程要让出 CPU 时，会将它的 stCoRoutine_t 从 pCallStack 弹出，“栈指针” iCallStackSize 减 1，然后 co_swap() 切换 CPU 上下文到原来被挂起的调用者协程恢复执行。这里“被挂起的调用者协程”，即是调用者 co_resume() 中切换 CPU 上下文被挂起的那个协程。

```
void co_yield_env( stCoRoutineEnv_t *env ){ stCoRoutine_t *last = env->pCallStack[ env->iCallStackSize - 2 ]; stCoRoutine_t *curr = env->pCallStack[ env->iCallStackSize - 1 ]; env->iCallStackSize--; co_swap( curr, last);}void co_yield_ct(){ co_yield_env( co_get_curr_thread_env() );}void co_yield( stCoRoutine_t *co ){ co_yield_env( co->env );}
```

- 同一个线程上所有协程是共享一个 stCoRoutineEnv_t 结构的，因此任意协程的 co->env 指向的结构都相同。

### 3.4 切换协程 Switch coroutine

上面的启动协程和挂起协程都设计协程的切换，本质是上下文的切换，发生在 co_swap()中。

- 如果是独享栈模式：将当前协程的上下文存好，读取下一协程的上下文。
- 如果是共享栈模式：libco 对共享栈做了个优化，可以申请多个共享栈循环使用，当目标协程所记录的共享栈没有被其它协程占用的时候，整个切换过程和独享栈模式一致。否则就是：将协程的栈空间内容从共享栈拷贝到自己的 save_buffer 中，将下一协程的 save_buffer 中的栈内容拷贝到共享栈中，将当前协程的上下文存好，读取下一协程上下文。

协程的本质是，使用 ContextSwap，来代替汇编中函数 call 调用，在保存寄存器上下文后，把需要执行的协程入口 push 到栈上。

```
void co_swap(stCoRoutine_t* curr, stCoRoutine_t* pending_co){  stCoRoutineEnv_t* env = co_get_curr_thread_env(); //get curr stack sp char c; curr->stack_sp= &c; if (!pending_co->cIsShareStack) {  env->pending_co = NULL;  env->occupy_co = NULL; } else {  env->pending_co = pending_co;  //get last occupy co on the same stack mem  stCoRoutine_t* occupy_co = pending_co->stack_mem->occupy_co;  //set pending co to occupy thest stack mem;  pending_co->stack_mem->occupy_co = pending_co;  env->occupy_co = occupy_co;  if (occupy_co && occupy_co != pending_co)  {   save_stack_buffer(occupy_co);  } } //swap context coctx_swap(&(curr->ctx),&(pending_co->ctx) ); //stack buffer may be overwrite, so get again; stCoRoutineEnv_t* curr_env = co_get_curr_thread_env(); stCoRoutine_t* update_occupy_co =  curr_env->occupy_co; stCoRoutine_t* update_pending_co = curr_env->pending_co; if (update_occupy_co && update_pending_co && update_occupy_co != update_pending_co) {  //resume stack buffer  if (update_pending_co->save_buffer && update_pending_co->save_size > 0)  {   memcpy(update_pending_co->stack_sp, update_pending_co->save_buffer, update_pending_co->save_size);  } }}
```

这里起寄存器拷贝切换作用的 coctx_swap 函数，是用汇编来实现的。

coctx_swap 接受两个参数，第一个是当前协程的 coctx_t 指针，第二个参数是待切入的协程的 coctx_t 指针。该函数调用前还处于第一个协程的环境，调用之后就变成另一个协程的环境了。

```
extern "C"{ extern void coctx_swap( coctx_t *,coctx_t* ) asm("coctx_swap");};.globl coctx_swap#if !defined( __APPLE__ ).type  coctx_swap, @function#endifcoctx_swap:#if defined(__i386__)    movl 4(%esp), %eax    movl %esp,  28(%eax)    movl %ebp, 24(%eax)    movl %esi, 20(%eax)    movl %edi, 16(%eax)    movl %edx, 12(%eax)    movl %ecx, 8(%eax)    movl %ebx, 4(%eax)    movl 8(%esp), %eax    movl 4(%eax), %ebx    movl 8(%eax), %ecx    movl 12(%eax), %edx    movl 16(%eax), %edi    movl 20(%eax), %esi    movl 24(%eax), %ebp    movl 28(%eax), %esp ret#elif defined(__x86_64__) leaq (%rsp),%rax    movq %rax, 104(%rdi)    movq %rbx, 96(%rdi)    movq %rcx, 88(%rdi)    movq %rdx, 80(%rdi) movq 0(%rax), %rax movq %rax, 72(%rdi)    movq %rsi, 64(%rdi) movq %rdi, 56(%rdi)    movq %rbp, 48(%rdi)    movq %r8, 40(%rdi)    movq %r9, 32(%rdi)    movq %r12, 24(%rdi)    movq %r13, 16(%rdi)    movq %r14, 8(%rdi)    movq %r15, (%rdi) xorq %rax, %rax    movq 48(%rsi), %rbp    movq 104(%rsi), %rsp    movq (%rsi), %r15    movq 8(%rsi), %r14    movq 16(%rsi), %r13    movq 24(%rsi), %r12    movq 32(%rsi), %r9    movq 40(%rsi), %r8    movq 56(%rsi), %rdi    movq 80(%rsi), %rdx    movq 88(%rsi), %rcx    movq 96(%rsi), %rbx leaq 8(%rsp), %rsp pushq 72(%rsi)    movq 64(%rsi), %rsi ret#endif
```

### 3.5 退出协程

同协程挂起一样，协程退出时也应将 CPU 控制权交给它的调用者，这也是调用 co_yield_env() 函数来完成的。

我们调用 co_create()、co_resume() 启动协程执行一次性任务，当任务结束后要记得调用 co_free()或 co_release() 销毁这个临时性的协程，否则将引起内存泄漏。

```
void co_free( stCoRoutine_t *co ){    if (!co->cIsShareStack)    {        free(co->stack_mem->stack_buffer);        free(co->stack_mem);    }    //walkerdu fix at 2018-01-20    //存在内存泄漏    else    {        if(co->save_buffer)            free(co->save_buffer);        if(co->stack_mem->occupy_co == co)            co->stack_mem->occupy_co = NULL;    }    free( co );}void co_release( stCoRoutine_t *co ){    co_free( co );}
```

## 4.**补充**

### 4.1 协程的调度

co_eventloop() 即“调度器”的核心所在。这里讲的“调度器”，严格意义上算不上真正的调度器，只是为了表述的方便。libco 的协程机制是非对称的，没有什么调度算法。在执行 yield 时，当前协程只能将控制权交给调用者协程，没有任何可调度的余地。Resume 灵活性稍强一点，不过也还算不得调度。如果非要说有什么“调度算法”的话，那就只能说是“基于 epoll/kqueue 事件驱动”的调度算法。“调度器”就是 epoll/kqueue 的事件循环。

```
void co_eventloop( stCoEpoll_t *ctx,pfn_co_eventloop_t pfn,void *arg ){ if( !ctx->result ) {  ctx->result =  co_epoll_res_alloc( stCoEpoll_t::_EPOLL_SIZE ); } co_epoll_res *result = ctx->result; for(;;) {  int ret = co_epoll_wait( ctx->iEpollFd,result,stCoEpoll_t::_EPOLL_SIZE, 1 );  stTimeoutItemLink_t *active = (ctx->pstActiveList);  stTimeoutItemLink_t *timeout = (ctx->pstTimeoutList);  memset( timeout,0,sizeof(stTimeoutItemLink_t) );  for(int i=0;i<ret;i++)  {   stTimeoutItem_t *item = (stTimeoutItem_t*)result->events[i].data.ptr;   if( item->pfnPrepare )   {    item->pfnPrepare( item,result->events[i],active );   }   else   {    AddTail( active,item );   }  }  unsigned long long now = GetTickMS();  TakeAllTimeout( ctx->pTimeout,now,timeout );  stTimeoutItem_t *lp = timeout->head;  while( lp )  {   //printf("raise timeout %p\n",lp);   lp->bTimeout = true;   lp = lp->pNext;  }  Join<stTimeoutItem_t,stTimeoutItemLink_t>( active,timeout );  lp = active->head;  while( lp )  {   PopHead<stTimeoutItem_t,stTimeoutItemLink_t>( active );            if (lp->bTimeout && now < lp->ullExpireTime)   {    int ret = AddTimeout(ctx->pTimeout, lp, now);    if (!ret)    {     lp->bTimeout = false;     lp = active->head;     continue;    }   }   if( lp->pfnProcess )   {    lp->pfnProcess( lp );   }   lp = active->head;  }  if( pfn )  {   if( -1 == pfn( arg ) )   {    break;   }  } }}
```

在关键数据结构 stCoRoutineEnv_t 中，有一个变量 stCoEpoll_t 类型的指针，即与 epoll 事件循环相关。

- iEpollFd：epoll 实例的文件描述符
- *EPOLL*SIZE：一次 epoll_wait 最多返回的就绪事件个数
- pTimeout：时间轮定时器
- pstTimeoutList：存放超时事件
- pstActiveList：存放就绪事件/超时事件
- result：epoll_wait 得到的结果集

```
struct stCoEpoll_t{ int iEpollFd; static const int _EPOLL_SIZE = 1024 * 10; struct stTimeout_t *pTimeout; struct stTimeoutItemLink_t *pstTimeoutList; struct stTimeoutItemLink_t *pstActiveList; co_epoll_res *result;};
```

一般而言，使用定时功能时，我们首先向定时器中注册一个定时事件（Timer Event），在注册定时事件时需要指定这个事件在未来的触发时间。在到了触发时间点后，我们会收到定时器的通知。

网络框架里的定时器可以看做由两部分组成：

- 第一部分是保存已注册 timer events 的数据结构，第二部分则是定时通知机制。保存已注册的 timer events ，一般选用红黑树，比如 nginx；另外一种常见的数据结构便是时间轮，libco 就使用了这种结构。
- 第二部分是高精度的定时（精确到微秒级）通知机制，一般使用 getitimer/setitimer 这类接口，用 epoll/kqueue 这样的系统调用来完成定时通知。

### 4.2 何时挂起何时恢复

libco 中有 3 种调用 yield 的场景：

- 用户程序中主动调用 co_yield_ct()；
- 程序调用了 epoll() 或 co_cond_timedwait() 陷入“阻塞”等待；
- 程序调用了 connect(), read(), write(), recv(), send() 等系统调用陷入“阻塞”等待。

resume 启动一个协程有 3 种情况：

- 对应用户程序主动 yield 的情况，这种情况也有赖于用户程序主动将协程 co_resume() 起来；
- epoll() 的目标文件描述符事件就绪或超时，co_cond_timedwait() 等到了其他协程的 co_cond_signal() 通知信号或等待超时；
- read(), write() 等 I/O 接口成功读到或写入数据，或者读写超时。

原文作者：alexzmzheng

原文链接：https://mp.weixin.qq.com/s/sy26w9XVvQsQoVhbQoN3vQ