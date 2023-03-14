# 【NO.241】协程及c++ 20原生协程研究报告

> 以下内容来自于腾讯后台研发工程师johnyao

## 0.**引言**

最近对C++20协程的进行了预研, 作为对比，同时研究了下市面上已经存在的其他协程实现方案。

虽然工作重点是C++20协程的预研，但作为一篇完整的文章, 不可避免的要从协程的基础开始讲起。

本文第一部分介绍现有一些协程的实现方案。如果你已经对协程非常熟悉，尤其是知道栈(stack)，帧(frame)在协程知识体系中意义，可以跳过第一部分。

本文第二部分介绍C++20协程的实现。如果你已经对C++20协程有所了解，知道C++20协程的基础实现，可以跳过第二部分。

本书第三部分是实验数据，以及C++20协程在项目内落地的设想，欢迎各位同学参与讨论。

## **1. 协程现状**

### **1.1 协程概述**

关于协程的定义和实现，并没有像进程和线程那样有统一的标准。而且由于都有一个“程”字，初学者很容易去和线程和进程做类比，往往走进理解的误区。

所以我们一开始只是引用维基的定义，简单说明下协程是什么：“协程是允许执行被挂起与被恢复子例程（也称为子程序 subroutine）”。

这一章节，我从函数切换的寄存器操作入手，继而通过协程的实现，和不同协程分类标准的介绍，帮助读者理解协程的本质。

### **1.2 函数切换 & 栈 & 帧**

**背景知识**

我们都知道进程内存空间被划分为下面的各内存段（引自维基百科Code segment）：

1. text 代码段
2. data 已经初始化的全局数据
3. bss 未初始化的全局数据
4. heap 堆区
5. stack 栈区

对于函数切换以及后面介绍的协程切换，我们最关心的是栈区的内存的管理。

当函数调用发生后，会扩展栈区内存，用于存放函数体内声明的局部变量。当被调函数返回时，则释放这部分内存。每次函数调用扩展的这部分内存我们称之为栈帧。**函数调用栈(call stack)就是由栈帧组成的，函数调用和返回就是栈帧（stack frame）入栈出栈的过程。**

我们以如下代码为例，看下这个过程中详细步骤。

```
#include <cstdint>

int64_t f(int p1, int p2, int p3, int p4, int p5, int p6, int p7, int p8)

{

int x = 0;

int y = 0;

int64_t z = 0;

int64_t l = 0;

x++;

p1++;

p7++;

p8++;

return 0;

}



int main()

{

int64_t i = 0;

i++;

i = f(1, 2, 3, 4, 5, 6, 7, 8);

return 0;

}
```

这里需要一些汇编的基础知识，但由于这部分不是此篇文章的重点，感兴趣的同学可以自行学习。 对于本文要解释的函数切换过程，我们只需要了解如下内容：维持栈帧需要两个寄存器的支持，对于x86_64架构，即%rbp，%rsp。在本文中, 我们把他们称为

1. rbp **栈帧基地址寄存**： 当前栈帧的起始内存地址。
2. rsp **栈顶地址寄存器**：整个调用栈的栈顶地址。

此外还需要了解：函数需要执行的下一条指令存放在%rip中，我们称之为指令地址寄存器。

有了如上的背景知识，我们看下在main函数中调用函数f，具体发生了什么。以此来加深栈帧的概念。

### 1.3 **参数传递**

根据上一节的内容，在函数f调用发生之前，%rbp指向main函数对应栈帧的基地址。%rsp指向整个栈区的顶地址。

函数调用的第一步是参数传递。上面例子参数传递部分，在gcc编译器下，产生的汇编如下：

pushq $8

pushq $7

movl $6, %r9d

movl $5, %r8d

movl $4, %ecx

movl $3, %edx

movl $2, %esi

movl $1, %edi

参数个数小于等于6时，采用寄存器传递，对于gcc使用的是%rdi, %rsi, %rdx, %rcx, %r8, %r9这6个寄存器。当大于6个参数时，需要使用栈区内存进行传递。本例中的参数7和8，都被执行了压栈操作pushq。 pushq，可以认为是两步操作

1. 扩展栈区地址空间，也就是将%rsp寄存器内容增加8。
2. 将目的操作数，拷贝到这个新扩展的内存空间。

可以看到参数传递是在调用方完成的。

### 1.4 **函数调用**

函数调用产生的汇编代码如下:

call f(int, int, int, int, int, int, int, int)

call指令也相当于三步操作

1. 扩展栈区地址空间，也就是将%rsp寄存器内容增加8（x86_64架构）。
2. 将%rip的内容（一下条指令，函数返回后需要执行的指令），拷贝到这个地址空间。
3. jmp到f函数标号。

### 1.5 **被调函数的处理**

pushq %rbp

movq %rsp, %rbp

subq $56, %rsp

被调用函数生成的汇编，需要执行三步初始化操作：

1. 保存调用者（main函数）的栈帧基地址寄存器的地址。
2. 将当前的栈顶指针，作为新的栈帧的基地址。
3. 根据当前栈帧（f函数）内申请的局部变量的总大小，直接扩展栈的大小。这里之所以用sub, 是因为栈的地址空间是从高地址向低地址部分扩展的。

### 1.6 **切换总结**

以上部分，可以总结为下图。

![img](https://pic3.zhimg.com/80/v2-5c8f22be47bcf67f27e9440c6e761fa2_720w.webp)

需要说明下，虽然为了画图方便，%rsp指向了一个未使用的地址，**但实际上栈顶指针指向实际使用的地址: 比如参数传递部分，指向的是参数7的地址; 函数调用部分指向的是返回地址对应的地址。**

在被调函数完成相关初始化处理后, **我们称绿色部分为函数f对应的栈帧。也可以把 (%rbp %rip] 部分当做函数f的栈帧。**对栈帧的理解，可能会有些许差别，但在此处并不关键。

理解了上述函数调用的过程后,函数返回的过程可以简单的理解为以上操作的反向操作。

movl $0, %eax

leave

ret

也是三步:

1. 函数返回值保存在%rax中。
2. leave指令会把当前%rbp的值恢复到%rsp（恢复栈顶）；会将保存在f函数栈帧中的main函数的栈帧基地址弹出, 恢复到%rbp寄存器中
3. ret指令会将保存在f函数栈帧中的返回地址(main函数函数调用后的下一条指令地址)弹出, 恢复到%rip寄存器中

## **2.有栈协程的实现**

### 2.1**基于栈帧切换的协程**

如果我们理解了上述函数调用的实现细节, 如果我们允许函数f 在执行某些等待异步操作的时机, 将它的执行上下文, 主要是各寄存器(通用寄存器, %rbp, %rsp, %rip等)的值, 保存在某个协程上下文对象(内存)中。

在异步操作完成，需要切换回来时, 我们再将这些保存的值恢复到各寄存器, 那么函数f 就成为了一个协程。当然这个过程还涉及到栈帧这块内存本身的处理, 这点在后面的小节马上介绍。

```
int64_t f(int p1, int p2, int p3, int p4, int p5, int p6, int p7, int p8)

{

int x = 0;

int y = 0;

int64_t z = 0;

int64_t l = 0;

<等待异步操作-yield>

x++;

p1++;

p7++;

p8++;

return 0;

}
```

posix ucontex, boost fcontext和tencent libco都是这种类型的协程。 **对于有栈协程, 时刻要记住一点: 栈帧中使用的指针型变量, 如果不是指向该栈帧中的局部变量, 在协程恢复后其意义可能已经发生改变。**

### 2.2 **有栈协程定义**

有栈协程是指协程本身有自己独立的调用栈。基于栈帧切换的协程, 除了寄存器上下文,一般需要我们给协程指定其使用的栈空间。在协程切换后, 你会发现调用方的函数调用栈替换为了, 被调方协程的调用栈。 以libco为例, 协程的寄存器上下文保存在regs[ 14 ], 而协程使用的栈由ss_sp指定。

```
struct coctx_t

{

\#if defined(__i386__)

void *regs[ 8 ];

\#else

void *regs[ 14 ];

\#endif

size_t ss_size;

char *ss_sp;

};
```

### 2.3 **有栈协程切换**

我们以libco为例看下有栈协程的切换。 libco的协程入口函数遵循如下原型：

typedef void* (*coctx_pfn_t)( void* s, void* s2 );

在函数切换前，我们要完成对上下文的初始化，主要是完成如下三点：

1. 将RSP（此时还不是寄存器，而是保存该寄存器的内存）设置为之前指定的ss_sp对应的地址空间的最大值-8（可以想下为什么设置为栈空间的最大值，前面已经提过）。
2. 将返回地址设置为协程函数pfn的起始地址，这样协程上下文切换后，就可以从指定的函数执行。
3. 将函数的参数保存在RDI， RSI（此时还不是寄存器，而是保存该寄存器的内存）

```
int coctx_make(coctx_t* ctx, coctx_pfn_t pfn, const void* s, const void* s1) {

char* sp = ctx->ss_sp + ctx->ss_size - sizeof(void*);

sp = (char*)((unsigned long)sp &amp; -16LL);



memset(ctx->regs, 0, sizeof(ctx->regs));

void** ret_addr = (void**)(sp);

*ret_addr = (void*)pfn;



ctx->regs[kRSP] = sp;



ctx->regs[kRETAddr] = (char*)pfn;



ctx->regs[kRDI] = (char*)s;

ctx->regs[kRSI] = (char*)s1;

return 0;

}
```

协程的切换过程相较于函数切换的call指令，使用的是coctx_swap函数。这是汇编实现的一个函数，函数原型如下：

extern void coctx_swap(coctx_t*, coctx_t*) asm("coctx_swap");

想下上面的“参数传递”小节，coctx_swap作为一个函数， 在进入该函数前。函数第一个参数coctx_t*会设置到%rdi，这个结构体用于保存切换前的协程上下文。第二个参数会设置到%rsi，这个结构体就是切换后的协程上下文。如果第二个参数对应的上下文结构刚通过上面coctx_make初始化完成。那么通过下述恢复操作，

movq 48(%rsi), %rbp

movq 104(%rsi), %rsp

<other-code>

movq 56(%rsi), %rdi

<other-code>

leaq 8(%rsp), %rsp

pushq 72(%rsi)

movq 64(%rsi), %rsi

ret

关键的寄存器会被设置：

1. 设置新协程的栈帧基地址寄存器，此处是0。
2. 设置新协程的栈顶地址寄存器（前面已经介绍过，coctx_t结构中指定的ss_sp对应的地址空间的最大值-8）。
3. 通过"movq 56(%rsi), %rdi" 把coctx_make的第三个参数 void *s放置在参数传递寄存器%rdi。*
4. *通过"leaq 8(%rsp), %rsp" 将栈顶降低8（用于保存下一步的返回地址）*
5. *通过"pushq 72(%rsi)" 把coctx_swap返回后执行的指令设置为协程如果为函数pfn（coctx_make的第二个参数） 进行压栈。*
6. *通过"movq 64(%rsi), %rsi" 把coctx_make的第四个参数 void* s1放置在参数传递寄存器%rsi。
7. 通过ret指令将第5步的压栈的地址弹出到%rip，开始了新协程函数的执行。

### 2.4 **切换总结**

在执行完被调函数初始化后，会开始新的栈的执行，后续该协程栈上的函数调用和普通函数调用没有区别。 完整的流程如下图：

![img](https://pic3.zhimg.com/80/v2-313cd5a6e396b2162d0f6ec00c231cee_720w.webp)

再次说明下，虽然为了画图方便，%rsp指向了一个未使用的地址，**但实际上栈顶指针指向实际使用的地址: 比如第二步中，指向返回地址对应的栈空间。**

其他的有栈协程切换方式和libco类似，不一一赘述。

### 2.5 **私有栈 vs 共享栈**

libco在协程切换之上，还提供了私有栈和共享栈的封装。 私有栈是针对每个新开的协程都指定独立的栈空间，栈空间不能太大，有越界风险。 共享栈则是定义一个默认线程栈空间大小的栈，多个协程共享同一空间，使用者不用担心越界风险。但在协程切换时，涉及到栈空间的保存。

## **3.无栈协程**

### 3.1 **基于状态机的协程**

这种方法, 本人还没有仔细研究。简单说下和其他同事讨论的相关结论: 这种方式并不会执行寄存器级的上下文保存和恢复, 只是将函数执行到的行号记录在协程对象的成员变量中, 协程函数通过switch case和跳转宏, 在恢复执行时跳转到指定的行号继续执行。 这种实现方式就可以认为是一种无栈协程。无栈并不是没有stack。而是在现有的stack上创建协程栈帧。不会为协程指定独立的stack空间。 C++20的原生协程就是此种实现。这里可以提前透露下，相较于其他无栈协程，**C++20的原生协程创建的栈帧存在于堆上，我们可称之为堆帧，并不会随函数的挂起而销毁。**

## **4.对称协程 vs 非对称协程**

关于协程还有一种分类方法，对称，非对称。对称协程只提供了一种协程间的控制转移的语义即pass control, 而非对称提供了两种, invoke和suspend。 利用libco可以实现对称协程，也可以实现非对称协程。但我们一般倾向于实现非对称协程，实现如下程序架构。

![img](https://pic2.zhimg.com/80/v2-cc9695e939eee3d2ade8c8b6b2b127dd_720w.webp)

c++20的原生协程也是非对称式的。在协程挂起时会返回到它的调用方。但我们还是可以实现它的对称转移，其中原因下篇文章会讲到。 对称协程的控制转移示意图如下：

![img](https://pic4.zhimg.com/80/v2-7d3d29745247921928e5ac474e21f88f_720w.webp)

## **5.C++ 20协程**

这一章节我们会给出，C++20协程的定义，并列举协程需要的所有接口。这一章节会一下涌现很多术语和概念，可能你会感到有些困扰，但不用担心，后续章节会逐一解释各个接口的具体使用。

### **5.1 C++20协程总览**

我们先看下C++20协程的定义。C++20协程标准引入了3个新的关键字, co_await, co_yield, co_return。如果一个函数包含了如上3个关键字之一，则该函数就是一个协程。

除了这3个关键字，实现一个C++20协程还需要实现两个鸭子类型，分别是promise type和awaiter type。

举个例子：对于如下函数some_coroutine，由于在函数体内使用了co_await, 所以在C++20标准下，它就成为一个协程。

```
T some_coroutine(P param)

{

<declare x>

co_await x;

}
```

按照编译器的约定，该函数的返回值类型T，必须包含名为promise_type的子类型，且该子类型必须拥有约定的接口。

```
class T

{

public:

class promise_type

{

public:

<method of convention>

};



};
```

对于co_await操作数x，可能是如下类型：

1. 鸭子类型awaiter type。
2. 可以通过 T::promise_type::await_transform 接口转换为awaiter type的类型。
3. 第三种鸭子类型，awaitable type（和awaiter关系紧密，但有所区别）。

### 5.2 **接口清单**

关于上文中提到的三种鸭子类型，我们将相关接口约定列举如下，后续章节会介绍基础接口的使用。

- awaiter type需要实现如下名字的函数:

1. await_ready
2. await_suspend
3. await_resume



- awaitable type需要实现如下的操作符重载:

1. operator co_await()



- promise type需要实现如下名字的函数：

1. get_return_object
2. initial_suspend
3. final_suspend
4. unhandled_exception
5. return_void



- promise type可选实现如下名字的函数：

1. return_value
2. operater new
3. operater delete
4. get_return_object_on_allocation_failure
5. yield_value
6. await_transform



## **6.C++20协程实现原理**

### 6.1 **awaiter type**

我们先从co_await的语义实现说起。

co_await x;

假设x是我们之前说的awaiter type的变量。我们知道awaiter type有三个必须实现的接口，await_ready, await_suspend, await_resume。

那么co_await的执行过程相当于如下伪代码：(引自参考文献1)

```
{

if (!awaiter.await_ready())

{

using handle_t = std::experimental::coroutine_handle<P>;



using await_suspend_result_t =

decltype(awaiter.await_suspend(handle_t::from_promise(p)));



<suspend-coroutine>



if constexpr (std::is_void_v<await_suspend_result_t>)

{

awaiter.await_suspend(handle_t::from_promise(p));

<return-to-caller-or-resumer>

}

else

{

static_assert(

std::is_same_v<await_suspend_result_t, bool>,

"await_suspend() must return 'void' or 'bool'.");



if (awaiter.await_suspend(handle_t::from_promise(p)))

{

<return-to-caller-or-resumer>

}

}



<resume-point>

}



return awaiter.await_resume();

}
```

简单的讲，就是如下步骤

1. 首先调用await_ready判断是否需要执行挂起（异步操作是否已经完成）
2. 然后调用await_suspend

- 如果返回值是void版本的实现，则直接挂起。
- 如果返回值是bool版本的实现，则根据返回值决定是否挂起。
- 如果返回值是coroutine_handle<>版本的实现，挂起并返回到该返回值对应的协程。



1. 当协程唤醒后，会执行await_resume()。其返回值作为(co_await x)表达式的值。

coroutine_handle<>是新出现的一个类型。从名字我们就可以知道，它是协程的句柄。后续在介绍promise type类型时会继续介绍它。

总览部分也提到了co_await操作数x，除了awaiter type，还可能是如下其他类型：

所以对于非awaiter type的x变量，可能经历如下转换步骤(引自参考文献1)。

```
template<typename P, typename T>

decltype(auto) get_awaitable(P&amp; promise, T&amp;&amp; expr)

{

if constexpr (has_any_await_transform_member_v<P>)

return promise.await_transform(static_cast<T&amp;&amp;>(expr));

else

return static_cast<T&amp;&amp;>(expr);

}



template<typename Awaitable>

decltype(auto) get_awaiter(Awaitable&amp;&amp; awaitable)

{

if constexpr (has_member_operator_co_await_v<Awaitable>)

return static_cast<Awaitable&amp;&amp;>(awaitable).operator co_await();

else if constexpr (has_non_member_operator_co_await_v<Awaitable&amp;&amp;>)

return operator co_await(static_cast<Awaitable&amp;&amp;>(awaitable));

else

return static_cast<Awaitable&amp;&amp;>(awaitable);

}
```

简单的讲就是：

1. 调用T::promise_type::await_transform，将x作为参数传入，返回新的对象y（如果没有定义该函数，y就是x本身）。
2. 如果上一步得到的y对象的类重载了co_await()运算符，或者有全局的co_await()运算符，则调用该运算符，返回一个awaiter。

### 6.2 **promise type**

上一小节，我们已经介绍了promise type的其中一个接口await_transform。下面我们继续看下promise type其他接口，借此了解协程函数本身的实现细节。

针对上面的协程some_coroutine，以及它的返回值类型T，调用协程的语句可以理解为如下过程 (引自参考文献1)

```
// Pretend there's a compiler-generated structure called 'coroutine_frame'

// that holds all of the state needed for the coroutine. It's constructor

// takes a copy of parameters and default-constructs a promise object.

struct coroutine_frame { ... };



T some_coroutine(P param)

{

auto* f = new coroutine_frame(std::forward<P>(param));



auto returnObject = f->promise.get_return_object();



// Start execution of the coroutine body by resuming it.

// This call will return when the coroutine gets to the first

// suspend-point or when the coroutine runs to completion.

coroutine_handle<decltype(f->promise)>::from_promise(f->promise).resume();



// Then the return object is returned to the caller.

return returnObject;

}
```

如果你对之前文章中提到的函数切换，协程切换还有印象的话，作为一个被调用的函数，他需要保存其局部变量的栈帧空间。 对于C++20的原生协程，可以看到，编译器首先会为协程在堆上分配这块空间，我称之为**堆帧**。堆栈的大小可以认为是，T::promise_type的大小，协程局部变量以及参数的大小累计相加得到的。

在协程的**堆帧**上，会同时创建协程对应的T::promise_type的变量。 然后调用其get_return_object()接口。这个接口负责返回一个T类型的变量。 这里有一点我个人的理解：这里的伪代码只是演示方便，**执行过程并不会封装为一个函数，并不会启动新的栈帧，而是在原有栈帧上执行此逻辑**。所以协程函数返回的T类型的变量，只是一个临时变量。

这里我们再次看到coroutine_handle。在介绍了堆帧后，我们现在可以说，这个句柄维持了指向协程堆帧的指针。我们可以调用该句柄的resume函数恢复挂起状态协程的执行。

协程本身的执行遵循如下伪代码的流程(引自参考文献1)

```
{

co_await promise.initial_suspend();

try

{

<body-statements>

}

catch (...)

{

promise.unhandled_exception();

}

FinalSuspend:

co_await promise.final_suspend();

}
```



1. 首先会调用协程对应的promise变量的initial_suspend函数，该函数返回值应可以作为co_await的操作数（参见上一小节的内容）。这里主要是允许C++20协程的使用者，可以在协程执行前就挂起。
2. 然后开始执行我们编写的协程代码。 执行代码过程中，如果遇到了挂起，则会返回到调用者。
3. 最后，无论是否中间经历了挂起，在协程完全结束后，还会调用协程对应的promise变量的final_suspend函数，该函数返回值应可以作为co_await的操作数。这里主要是允许C++20协程的使用者，可以在退出前做适当的处理。
4. 这里还需要实现unhandled_exception()，用于处理协程本身未处理的异常。

除此外，promise type还有一个必须实现的接口，return_void() 或者 return value() 二选一。 在使用co_return时， 会调用你实现的函数，并跳转到FinalSuspend。

### 6.3 **co_yield**

至此，我们还剩一个关键字没有解释。在协程内调用co_yield

co_yield <expr>



相当于调用



co_await promise.yield_value(<expr>).

也就是说，对于要支持co_yield的协程，promise_type需要实现yield_value函数，同样的，该函数返回值应可以作为co_await的操作数。

## **7.一个简单的实现**

有了以上的理解，那么我们及可以实现一个简单的demo了。

```
std::coroutine_handle<> g_handle;

struct BaseSwapTestCoro

{

struct awaiter

{

bool await_ready() { return false; }

void await_suspend(std::coroutine_handle<> h) { g_handle = h; }

void await_resume() {}

};



struct promise_type

{

BaseSwapTestCoro get_return_object() { return {}; }



std::suspend_never initial_suspend() { return {}; }

std::suspend_never final_suspend() noexcept { return {}; }

void unhandled_exception() {}

void return_void() {}

};

};
```

需要说明的是std::suspend_never是预定义的变量，表明是nerver suspend的awaiter。

测试代码如下

```
BaseSwapTestCoro SomeFunc()

{

LOG(0, "in coroutine: before await");

co_await BaseSwapTestCoro::awaiter();

LOG(0, "in coroutine: after await");

}



TEST(base, swap)

{

SomeFunc();

LOG(0, "in main: before resume");

g_handle.resume();

LOG(0, "in main: after resume");

}
```

测试输出如下

```
[ RUN ] base.swap

base_test.cpp:29|in coroutine: before await

base_test.cpp:37|in main: before resume

base_test.cpp:31|in coroutine: after await

base_test.cpp:39|in main: after resume

[ OK ] base.swap (0 ms)
```

关于C++20协程实现的基本原理，先介绍到这么多。如果想进一步了解其他可选接口的使用，可以阅读参考资料1。这里需要说明一点，协程的语义并没有改变C++的基本语法规则，比如：

1. co_await BaseSwapTestCoro::awaiter(); 这里会创建awaiter的一个临时变量，那么这个临时变量在该语句执行完成后就会释放。
2. 协程退出后，栈帧就会销毁。g_handle就会指向一块已经释放的内存，再次resume就会的导致crash。 所以对于上面的例子，可以在await_resume, 将g_handle置空，以防野指针问题。

## **8.实验及落地设想**

### **8.1 基础性能测试**

在了解了C++20的实现原理后，我做了协程的基础创建和切换的试验

```
std::coroutine_handle<> g_handle;

struct BaseSwapTestCoro

{

struct awaiter

{

bool await_ready() { return false; }

void await_suspend(std::coroutine_handle<> h) { g_handle = h; }

int await_resume() { return 1; }

};



struct promise_type

{

BaseSwapTestCoro get_return_object() { return {}; }



awaiter initial_suspend() { return awaiter{}; }

std::suspend_never final_suspend() noexcept { return {}; }

void unhandled_exception() {}



void return_void() {}



auto await_transform() = delete; // no use co_wait

auto yield_value(int) { return awaiter{}; } // how to use void

};

};



TEST(base, swap)

{

base_swap::SomeFunc();

// test

int MAX_LOOP_COUNT = 1000000;

auto begin = CALC_CLOCK_NOW();

for (int i = 0; i < MAX_LOOP_COUNT; i++)

{

base_swap::g_handle.resume();

}

auto end = CALC_CLOCK_NOW(); // 340ns

LOG(0, "cost %lld ps", CALC_PS_AVG_CLOCK(end - begin, MAX_LOOP_COUNT) / 2);

// EXPECT_EQ(g_counter, MAX_LOOP_COUNT);

}
```

对比libco的方案，有如下数据

| 方案                   | 耗时（单位：皮秒=0.001纳秒） |
| ---------------------- | ---------------------------- |
| libco原生实现          | 17,000 ps                    |
| libco opt by walkerdu2 | 4,243 ps                     |
| c20上下文切换          | 1,660 ps                     |

此外， 还得到了c20协程创建开销 1,400 ps。

看到这个数据还是很令人振奋的。

### **8.2 落地设想**

考虑项目内的使用情况，我们往往会将某些协程函数进行封装，这样就会出现某个协程函数等待另一个协程函数的请求。

举个例子，某个RPC请求的响应函数，由于需要请求其他的服务，所以被实现为一个协程A。某些常用的其他服务请求被封装为协程B。A使用B完成部分功能。

假设协程调用过程如下

```
T B()

{

<co_await service b>

}



T A()

{

B();

<其他同步操作>

<co_await service c>

};
```

之前有提过，C++20协程是非对称的。如果这样实现的话， 在B函数挂起时， 会返回到A协程的下一条语句继续执行。 且B协程后续唤醒后，执行完成相关逻辑，并不会回到A。而是回到他的唤醒者。如下图所示

![img](https://pic4.zhimg.com/80/v2-4719610b739680af51e4b6b795c75107_720w.webp)

而我们想得到的效果是某种对称转移的语义（如果对协程的对称性不了解，可以参见前面的章节）。

![img](https://pic1.zhimg.com/80/v2-17714de9fedd0afa6c490eaf11a92900_720w.webp)

上面对称转移到语义就要求我们在协程A中可以 co_await B协程, 等待其执行完成。

T A()

{

co_await B();

<其他同步操作>

<co_await service c>

};

### 8.3 **实现对称转移语义**

参考Lewiss Baker的第四篇文章3，我试着实现了这种对称转移的语义。思路如下 ，针对 co_await B(); 这个语句执行如下步骤：

1. B协程启动后通过initial_suspend立即挂起，并返回对应的T类型对象，此T类型对象保存了B协程句柄。
2. 通过await_transform将T类型对象转换为一个awaiter type，并在其await_suspend函数，通过保存的B协程句柄，在其对应的promise对象中记录他的调用者A。
3. 在B协程被唤醒，执行完后，利用final_suspend，恢复A的执行。

代码地址 [https://git.woa.com/johnyao/c20_coroutine/blob/master/include/coro.h](https://link.zhihu.com/?target=https%3A//git.woa.com/johnyao/c20_coroutine/blob/master/include/coro.h)

## **9.实验对比**

在完成初版的封装后，我们对比了三个方案：项目内已经落地的基于libco的协程池方案，Asio Library内的协程方案，以及本人实验封装的版本。

创建协程测试用例（其中项目内的测试数据由组内的seanxchen提供）：

| 方案                       | 耗时（单位：纳秒） |
| -------------------------- | ------------------ |
| 项目内libco协程池 首次创建 | 6436 ns            |
| 项目内libco协程池 预创建   | 140 ns             |
| C++20实验封装              | 42~368 ns          |
| C++20 asio co_spawn        | 450 ns             |

可以看到，有栈协程由于初始化的时候需要额外申请栈区空间等操作，首次使用的时候效率是最低的。达到了微秒级，6.5微秒左右。但由于协程池的使用，在初始化后再次使用，效率是最高的，140ns。Asio Library中使用的co_spawn，则要450 ns。 这里需要说明的是，本人实验封装的版本不包含协程管理的开销，由于没有提供统一的spawn接口，所以协程运行时间和协程的嵌套层级有关系。

C++20的无栈协程，会随着协程调用层级的增加，增加执行耗时。而有栈协程不会有这部分的开销，一旦协程创建成功，则协程内的函数执行就是普通函数调用。所以我只对比了自己的实验封装和Asio Library针对不同嵌套层级的开销。

| 嵌套层级  | C++20实验封装 | C++20 asio封装 |
| --------- | ------------- | -------------- |
| 1层(同步) | 42 ns         | 34 ns          |
| 3层       | 221 ns        | 124 ns         |
| 5层       | 368 ns        | 241 ns         |

测试用例git地址： [https://git.woa.com/johnyao/c20_coroutine/tree/master/tests](https://link.zhihu.com/?target=https%3A//git.woa.com/johnyao/c20_coroutine/tree/master/tests)

以上结果只是初步结论，后续还会持续校验和跟进。

## 10.**结语**

虽然对比项目内现有协程实现，性能有所降低，但个人认为相较于有栈协程，C++20的协程还是有他的优势。

1. 基础性能确实优越。
2. 语言原生支持， 后续可能有高效的对称转移语义的标准库。
3. 堆帧空间可认为不受限制，不用担心爆栈。

作为初步的预研，C++20协程可以总结为，在语言层面实现了一种非对称的无栈协程。作为语言原生支持的协程，基础的效率表现很亮眼。在项目中实际落地，还需要进一步的探索。后续有空闲时间，会继续关注如下三点

1. 如何提高协程的对称转移的效率。
2. 如何提高协程管理的效率。
3. 针对特定框架定制更高效的协程封装。

最后也欢迎感兴趣的同学，一起讨论C++20协程实际落地过程中的最佳实践。

## 11.**参考**

1. [https://lewissbaker.github.io/2018/09/05/understanding-the-promise-type](https://link.zhihu.com/?target=https%3A//lewissbaker.github.io/2018/09/05/understanding-the-promise-type)
2. [https://github.com/walkerdu/libco](https://link.zhihu.com/?target=https%3A//github.com/walkerdu/libco)
3. [https://lewissbaker.github.io/2020/05/11/understanding_symmetric_transfer](https://link.zhihu.com/?target=https%3A//lewissbaker.github.io/2020/05/11/understanding_symmetric_transfer)
4. [https://km.woa.com/articles/show/355152?kmref=search&from_page=1&no=1](https://link.zhihu.com/?target=https%3A//km.woa.com/articles/show/355152%3Fkmref%3Dsearch%26from_page%3D1%26no%3D1)

**欢迎点赞分享，关注[@鹅厂架构师](https://www.zhihu.com/org/e-han-jia-gou-shi)，一起探索更多业界领先产品技术。**

原文作者：鹅厂架构师

原文链接：https://zhuanlan.zhihu.com/p/509698893