# 【NO.592】内存泄漏-原因、避免和定位

作为C/C++开发人员，内存泄漏是最容易遇到的问题之一，这是由C/C++语言的特性引起的。C/C++语言与其他语言不同，需要开发者去申请和释放内存，即需要开发者去管理内存，如果内存使用不当，就容易造成段错误(segment fault)或者内存泄漏(memory leak)。

今天，借助此文，分析下项目中经常遇到的导致内存泄漏的原因，以及如何避免和定位内存泄漏。

本文的主要内容如下：

![img](https://pic1.zhimg.com/80/v2-5e58747ebd3203c05e74b30ef120dd34_720w.webp)

## 1.背景

C/C++语言中，内存的分配与回收都是由开发人员在编写代码时主动完成的，好处是内存管理的开销较小，程序拥有更高的执行效率；弊端是依赖于开发者的水平，随着代码规模的扩大，极容易遗漏释放内存的步骤，或者一些不规范的编程可能会使程序具有安全隐患。如果对内存管理不当，可能导致程序中存在内存缺陷，甚至会在运行时产生内存故障错误。

内存泄漏是各类缺陷中十分棘手的一种，对系统的稳定运行威胁较大。当动态分配的内存在程序结束之前没有被回收时，则发生了内存泄漏。由于系统软件，如操作系统、编译器、开发环境等都是由C/C++语言实现的，不可避免地存在内存泄漏缺陷，特别是一些在服务器上长期运行的软件，若存在内存泄漏则会造成严重后果，例如性能下降、程序终止、系统崩溃、无法提供服务等。

所以，本文从原因、避免以及定位几个方面去深入讲解，希望能给大家带来帮助。

## 2.概念

内存泄漏（Memory Leak）是指程序中己动态分配的堆内存由于某种原因程序未释放或无法释放，造成系统内存的浪费，导致程序运行速度减慢甚至系统崩溃等严重后果。

当我们在程序中对原始指针(raw pointer)使用new操作符或者free函数的时候，实际上是在堆上为其分配内存，这个内存指的是RAM，而不是硬盘等永久存储。持续申请而不释放(或者少量释放)内存的应用程序，最终因内存耗尽导致OOM(out of memory)。

![img](https://pic1.zhimg.com/80/v2-d6376878235a44f6999e02f6dd1fae64_720w.webp)

方便大家理解内存泄漏的危害，我们举个简单的例子。有一个宾馆，有100间房间，顾客每次都是在前台进行登记，然后拿到房间钥匙。如果有些顾客不需要该房间了，也不归还钥匙，久而久之，前台处可用房间越来越少，收入也越来越少，濒临倒闭。当程序申请了内存，而不进行归还，久而久之，可用内存越来越少，OS就会进行自我保护，杀掉该进程，这就是我们常说的OOM(out of memory)。

## 3.分类

内存泄漏分为以下两类：

- 堆内存泄漏：我们经常说的内存泄漏就是堆内存泄漏，在堆上申请了资源，在结束使用的时候，没有释放归还给OS，从而导致该块内存永远不会被再次使用
- 资源泄漏：通常指的是系统资源，比如socket，文件描述符等，因为这些在系统中都是有限制的，如果创建了而不归还，久而久之，就会耗尽资源，导致其他程序不可用

本文主要分析堆内存泄漏，所以后面的内存泄漏均指的是堆内存泄漏。

## 4.根源

内存泄漏，主要指的是在堆(heap)上申请的动态内存泄漏，或者说是指针指向的内存块忘了被释放，导致该块内存不能再被申请重新使用。

之前在知乎上看了一句话，指针是C的精髓，也是初学者的一个坎。换句话说，内存管理是C的精髓，C/C++可以直接跟OS打交道，从性能角度出发，开发者可以根据自己的实际使用场景灵活进行内存分配和释放。虽然在C++中自C++11引入了smart pointer，虽然很大程度上能够避免使用裸指针，但仍然不能完全避免，最重要的一个原因是你不能保证组内其他人不适用指针，更不能保证合作部门不使用指针。

那么为什么C/C++中会存在指针呢？

这就得从进程的内存布局说起。

## 5.进程内存布局

![img](https://pic4.zhimg.com/80/v2-70034ef213a987bf827bb3d647d1c883_720w.webp)

上图为32位进程的内存布局，从上图中主要包含以下几个块：

内核空间：供内核使用，存放的是内核代码和数据

stack：这就是我们经常所说的栈，用来存储自动变量(automatic variable)

mmap:也成为内存映射，用来在进程虚拟内存地址空间中分配地址空间，创建和物理内存的映射关系

heap:就是我们常说的堆，动态内存的分配都是在堆上

bss:包含所有未初始化的全局和静态变量，此段中的所有变量都由0或者空指针初始化，程序加载器在加载程序时为BSS段分配内存

ds:初始化的数据块

- 包含显式初始化的全局变量和静态变量
- 此段的大小由程序源代码中值的大小决定，在运行时不会更改
- 它具有读写权限，因此可以在运行时更改此段的变量值
- 该段可进一步分为初始化只读区和初始化读写区

text：也称为文本段

- 该段包含已编译程序的二进制文件。
- 该段是一个只读段，用于防止程序被意外修改
- 该段是可共享的，因此对于文本编辑器等频繁执行的程序，内存中只需要一个副本

由于本文主要讲内存分配相关，所以下面的内容仅涉及到栈(stack)和堆(heap)。

![img](https://pic1.zhimg.com/80/v2-441bb5bd2deb8c11e7ae7a02807507b0_720w.webp)



## 6.栈

栈一块连续的内存块，栈上的内存分配就是在这一块连续内存块上进行操作的。编译器在编译的时候，就已经知道要分配的内存大小，当调用函数时候，其内部的遍历都会在栈上分配内存；当结束函数调用时候，内部变量就会被释放，进而将内存归还给栈。

```text
class Object {
  public:
    Object() = default;
    // ....
};

void fun() {
  Object obj;
  
  // do sth
}
```

在上述代码中，obj就是在栈上进行分配，当出了fun作用域的时候，会自动调用Object的析构函数对其进行释放。

前面有提到，局部变量会在作用域（如函数作用域、块作用域等）结束后析构、释放内存。因为分配和释放的次序是刚好完全相反的，所以可用到堆栈先进后出（first-in-last-out, FILO）的特性，而 C++ 语言的实现一般也会使用到调用堆栈（call stack）来分配局部变量（但非标准的要求）。

因为栈上内存分配和释放，是一个进栈和出栈的过程(对于编译器只是一个移动指针的过程)，所以相比于堆上的内存分配，栈要快的多。

虽然栈的访问速度要快于堆，每个线程都有一个自己的栈，栈上的对象是不能跨线程访问的，这就决定了栈空间大小是有限制的，如果栈空间过大，那么在大型程序中几十乃至上百个线程，光栈空间就消耗了RAM，这就导致heap的可用空间变小，影响程序正常运行。

**设置**

在Linux系统上，可用通过如下命令来查看栈大小：

```text
ulimit -s
10240
```

在笔者的机器上，执行上述命令输出结果是10240(KB)即10m，可以通过shell命令修改栈大小。

```text
ulimit -s 102400
```

通过如上命令，可以将栈空间临时修改为100m，可以通过下面的命令：

```text
/etc/security/limits.conf
```

**分配方式**

**静态分配**

静态分配由编译器完成，假如局部变量以及函数参数等，都在编译期就分配好了。

```text
void fun() {
  int a[10];
}
```

上述代码中，a占10 * sizeof(int)个字节，在编译的时候直接计算好了，运行的时候，直接进栈出栈。

**动态分配**

可能很多人认为只有堆上才会存在动态分配，在栈上只可能是静态分配。其实，这个观点是错的，栈上也支持动态分配，该动态分配由alloca()函数进行分配。栈的动态分配和堆是不同的，通过alloca()函数分配的内存由编译器进行释放，无序手动操作。

**特点**

- 分配速度快：分配大小由编译器在编译器完成
- 不会产生内存碎片：栈内存分配是连续的，以FIFO的方式进栈和出栈
- 大小受限：栈的大小依赖于操作系统
- 访问受限：只能在当前函数或者作用域内进行访问

## 7.堆

堆（heap）是一种内存管理方式。内存管理对操作系统来说是一件非常复杂的事情，因为首先内存容量很大，其次就是内存需求在时间和大小块上没有规律（操作系统上运行着几十甚至几百个进程，这些进程可能随时都会申请或者是释放内存，并且申请和释放的内存块大小是随意的）。

堆这种内存管理方式的特点就是自由（随时申请、随时释放、大小块随意）。堆内存是操作系统划归给堆管理器（操作系统中的一段代码，属于操作系统的内存管理单元）来管理的，堆管理器提供了对应的接口_sbrk、mmap_等，只是该接口往往由运行时库进行调用，即也可以说由运行时库进行堆内存管理，运行时库提供了malloc/free函数由开发人员调用，进而使用堆内存。

**分配方式**

正如我们所理解的那样，由于是在运行期进行内存分配，分配的大小也在运行期才会知道，所以堆只支持动态分配，内存申请和释放的行为由开发者自行操作，这就很容易造成我们说的内存泄漏。

**特点**

- 变量可以在进程范围内访问，即进程内的所有线程都可以访问该变量
- 没有内存大小限制，这个其实是相对的，只是相对于栈大小来说没有限制，其实最终还是受限于RAM
- 相对栈来说访问比较慢
- 内存碎片
- 由开发者管理内存，即内存的申请和释放都由开发人员来操作

## 8.堆与栈区别

理解堆和栈的区别，对我们开发过程中会非常有用，结合上面的内容，总结下二者的区别。

对于栈来讲，是由编译器自动管理，无需我们手工控制；对于堆来说，释放工作由程序员控制，容易产生memory leak

空间大小不同

- 一般来讲在 32 位系统下，堆内存可以达到4G的空间，从这个角度来看堆内存几乎是没有什么限制的。
- 对于栈来讲，一般都是有一定的空间大小的，一般依赖于操作系统(也可以人工设置)

能否产生碎片不同

- 对于堆来讲，频繁的内存分配和释放势必会造成内存空间的不连续，从而造成大量的碎片，使程序效率降低。
- 对于栈来讲，内存都是连续的，申请和释放都是指令移动，类似于数据结构中的进栈和出栈

增长方向不同

- 对于堆来讲，生长方向是向上的，也就是向着内存地址增加的方向
- 对于栈来讲，它的生长方向是向下的，是向着内存地址减小的方向增长

分配方式不同

- 堆都是动态分配的，比如我们常见的malloc/new；而栈则有静态分配和动态分配两种。
- 静态分配是编译器完成的，比如局部变量的分配，而栈的动态分配则通过alloca()函数完成
- 二者动态分配是不同的，栈的动态分配的内存由编译器进行释放，而堆上的动态分配的内存则必须由开发人自行释放

分配效率不同

- 栈有操作系统分配专门的寄存器存放栈的地址，压栈出栈都有专门的指令执行，这就决定了栈的效率比较高
- 堆内存的申请和释放专门有运行时库提供的函数，里面涉及复杂的逻辑，申请和释放效率低于栈

截止到这里，栈和堆的基本特性以及各自的优缺点、使用场景已经分析完成，在这里给开发者一个建议，能使用栈的时候，就尽量使用栈，一方面是因为效率高于堆，另一方面内存的申请和释放由编译器完成，这样就避免了很多问题。

## 9.扩展

终于到了这一小节，其实，上面讲的那么多，都是为这一小节做铺垫。

在前面的内容中，我们对比了栈和堆，虽然栈效率比较高，且不存在内存泄漏、内存碎片等，但是由于其本身的局限性(不能多线程、大小受限)，所以在很多时候，还是需要在堆上进行内存。

我们先看一段代码：

```text
#include <stdio.h>
#include <stdlib.h>

int main() {
  int a;
  int *p;
  p = (int *)malloc(sizeof(int));
  free(p);

  return 0;
}
```

上述代码很简单，有两个变量a和p，类型分别为int和int *，其中，a和p存储在栈上，p的值为在堆上的某块地址(在上述代码中，p的值为0x1c66010)，上述代码布局如下图所示：

![img](https://pic2.zhimg.com/80/v2-4e8fa45166b831272c853ab637c08089_720w.webp)

**产生方式**

以产生的方式来分类，内存泄漏可以分为四类:

- 常发性内存泄漏
- 偶发性内存泄漏
- 一次性内存泄漏
- 隐式内存泄漏

**常发性内存泄漏**

产生内存泄漏的代码或者函数会被多次执行到，在每次执行的时候，都会产生内存泄漏。

**偶发性内存泄漏**

与常发性内存泄漏不同的是，偶发性内存泄漏函数只在特定的场景下才会被执行。

笔者在19年的时候，曾经遇到一个这种内存泄漏。有一个函数专门进行价格加密，每次泄漏3个字节，且只有在竞价成功的时候，才会调用此函数进行价格加密，因此泄漏的非常不明显。当时发现这个问题，是上线后的第二天，帮忙排查线上问题，发现内存较上线前上涨了点(大概几百兆的样子)，了解glibc内存分配原理的都清楚，调用delete后，内存不一定会归还给OS，但是本着宁可信其有，不可信其无的心态，决定来分析是否真的存在内存泄漏。

当时用了个比较傻瓜式的方法，通过top命令，将该进程所占的内存输出到本地文件，大概几个小时后，将这些数据导入Excel中，内存占用基本呈一条斜线，所以基本能够确定代码存在内存泄漏，所以就对新上线的这部分代码进行重新review，定位到泄漏点，然后修复，重新上线。

**一次性内存泄漏**

这种内存泄漏在程序的生命周期内只会泄漏一次，或者说造成泄漏的代码只会被执行一次。

有的时候，这种可能不算内存泄漏，或者说设计如此。就以笔者现在线上的服务来说，类似于如下这种：

```text
int main() {
  auto *service = new Service;
  // do sth
  service->Run();// 服务启动
  service->Loop(); // 可以理解为一个sleep，目的是使得程序不退出
  return 0;
}
```

这种严格意义上，并不算内存泄漏，因为程序是这么设计的，即使程序异常退出，那么整个服务进程也就退出了，当然，在Loop()后面加个delete更好。

**隐式内存泄漏**

程序在运行过程中不停的分配内存，但是直到结束的时候才释放内存。严格的说这里并没有发生内存泄漏，因为最终程序释放了所有申请的内存。但是对于一个服务器程序，需要运行几天，几周甚至几个月，不及时释放内存也可能导致最终耗尽系统的所有内存。所以，我们称这类内存泄漏为隐式内存泄漏。

比较常见的隐式内存泄漏有以下三种：

- 内存碎片：还记得我们之前的那篇文章深入理解glibc内存管理精髓，程序跑了几天之后，进程就因为OOM导致了退出，就是因为内存碎片导致剩下的内存不能被重新分配导致
- 即使我们调用了free/delete，运行时库不一定会将内存归还OS，具体深入理解glibc内存管理精髓
- 用过STL的知道，STL内部有一个自己的allocator，我们可以当做一个memory poll，当调用vector.clear()时候，内存并不会归还OS，而是放回allocator，其内部根据一定的策略，在特定的时候将内存归还OS，是不是跟glibc原理很像

**分类**

**未释放**

这种是很常见的，比如下面的代码：

```text
int fun() {
    char * pBuffer = malloc(sizeof(char));
    
    /* Do some work */
    return 0;
}
```

上面代码是非常常见的内存泄漏场景(也可以使用new来进行分配)，我们申请了一块内存，但是在fun函数结束时候没有调用free函数进行内存释放。

在C++开发中，还有一种内存泄漏，如下：

```text
class Obj {
 public:
   Obj(int size) {
     buffer_ = new char;
   }
   ~Obj(){}
  private:
   char *buffer_;
};

int fun() {
  Object obj;
  // do sth
  return 0;
}
```

上面这段代码中，析构函数没有释放成员变量buffer_指向的内存，所以在编写析构函数的时候，一定要仔细分析成员变量有没有申请动态内存，如果有，则需要手动释放，我们重新编写了析构函数，如下：

```text
~Object() {
  delete buffer_;
}
```

在C/C++中，对于普通函数，如果申请了堆资源，请跟进代码的具体场景调用free/delete进行资源释放；对于class，如果申请了堆资源，则需要在对应的析构函数中调用free/delete进行资源释放。

**未匹配**

在C++中，我们经常使用new操作符来进行内存分配，其内部主要做了两件事：

1. 通过operator new从堆上申请内存(glibc下，operator new底层调用的是malloc)
2. 调用构造函数(如果操作对象是一个class的话)

对应的，使用delete操作符来释放内存，其顺序正好与new相反：

1. 调用对象的析构函数(如果操作对象是一个class的话)
2. 通过operator delete释放内存

```text
void* operator new(std::size_t size) {
    void* p = malloc(size);
    if (p == nullptr) {
        throw("new failed to allocate %zu bytes", size);
    }
    return p;
}
void* operator new[](std::size_t size) {
    void* p = malloc(size);
    if (p == nullptr) {
        throw("new[] failed to allocate %zu bytes", size);
    }
    return p;
}

void  operator delete(void* ptr) throw() {
    free(ptr);
}
void  operator delete[](void* ptr) throw() {
    free(ptr);
}
```

为了加深多这块的理解，我们举个例子：

```text
class Test {
 public:
   Test() {
     std::cout << "in Test" << std::endl;
   }
   // other
   ~Test() {
     std::cout << "in ~Test" << std::endl;
   }
};

int main() {
  Test *t = new Test;
  // do sth
  delete t;
  return 0;
}
```

在上述main函数中，我们使用new 操作符创建一个Test类指针

1. 通过operator new申请内存(底层malloc实现)
2. 通过placement new在上述申请的内存块上调用构造函数
3. 调用ptr->~Test()释放Test对象的成员变量
4. 调用operator delete释放内存

上述过程，可以理解为如下：

```text
// new
void *ptr = malloc(sizeof(Test));
t = new(ptr)Test
  
// delete
ptr->~Test();
free(ptr);
```

好了，上述内容，我们简单的讲解了C++中new和delete操作符的基本实现以及逻辑，那么，我们就简单总结下下产生内存泄漏的几种类型。

**new 和 free**

仍然以上面的Test对象为例，代码如下：

```text
Test *t = new Test;
free(t)
```

此处会产生内存泄漏，在上面，我们已经分析过，new操作符会先通过operator new分配一块内存，然后在该块内存上调用placement new即调用Test的构造函数。而在上述代码中，只是通过free函数释放了内存，但是没有调用Test的析构函数以释放Test的成员变量，从而引起内存泄漏。

**new[] 和 delete**

```text
int main() {
  Test *t = new Test [10];
  // do sth
  delete t;
  return 0;
}
```

在上述代码中，我们通过new创建了一个Test类型的数组，然后通delete操作符删除该数组，编译并执行，输出如下：

```text
in Test
in Test
in Test
in Test
in Test
in Test
in Test
in Test
in Test
in Test
in ~Test
```

从上面输出结果可以看出，调用了10次构造函数，但是只调用了一次析构函数，所以引起了内存泄漏。这是因为调用delete t释放了通过operator new[]申请的内存，即malloc申请的内存块，且只调用了t[0]对象的析构函数，t[1..9]对象的析构函数并没有被调用。

**虚析构**

记得08年面谷歌的时候，有一道题，面试官问，std::string能否被继承，为什么？

当时没回答上来，后来过了没多久，进行面试复盘的时候，偶然看到继承需要父类析构函数为virtual，才恍然大悟，原来考察点在这块。

下面我们看下std::string的析构函数定义：

```text
~basic_string() { 
  _M_rep()->_M_dispose(this->get_allocator()); 
}
```

这块需要特别说明下，std::basic_string是一个模板，而std::string是该模板的一个特化，即std::basic_string

```text
typedef std::basic_string<char> string;
```

现在我们可以给出这个问题的答案：不能，因为std::string的析构函数不为virtual，这样会引起内存泄漏。

仍然以一个例子来进行证明。

```text
class Base {
 public:
  Base(){
    buffer_ = new char[10];
  }

  ~Base() {
    std::cout << "in Base::~Base" << std::endl;
    delete []buffer_;
  }
private:
  char *buffer_;

};

class Derived : public Base {
 public:
  Derived(){}

  ~Derived() {
    std::cout << "int Derived::~Derived" << std::endl;
  }
};

int main() {
  Base *base = new Derived;
  delete base;
  return 0;
}
```

上面代码输出如下：

```text
in Base::~Base
```

可见，上述代码并没有调用派生类Derived的析构函数，如果派生类中在堆上申请了资源，那么就会产生内存泄漏。

为了避免因为继承导致的内存泄漏，我们需要将父类的析构函数声明为virtual，代码如下(只列了部分修改代码，其他不变):

```text
~Base() {
    std::cout << "in Base::~Base" << std::endl;
    delete []buffer_;
  }
```

然后重新执行代码，输出结果如下：

```text
int Derived::~Derived
in Base::~Base
```

借助此文，我们再次总结下存在继承情况下，构造函数和析构函数的调用顺序。

派生类对象在创建时构造函数调用顺序：

1. 调用父类的构造函数
2. 调用父类成员变量的构造函数
3. 调用派生类本身的构造函数

派生类对象在析构时的析构函数调用顺序：

1. 执行派生类自身的析构函数
2. 执行派生类成员变量的析构函数
3. 执行父类的析构函数

为了避免存在继承关系时候的内存泄漏，请遵守一条规则：无论派生类有没有申请堆上的资源，请将父类的析构函数声明为virtual。

**循环引用**

在C++开发中，为了尽可能的避免内存泄漏，自C++11起引入了smart pointer，常见的有shared_ptr、weak_ptr以及unique_ptr等(auto_ptr已经被废弃)，其中weak_ptr是为了解决循环引用而存在，其往往与shared_ptr结合使用。

下面，我们看一段代码：

```text
class Controller {
 public:
  Controller() = default;

  ~Controller() {
    std::cout << "in ~Controller" << std::endl;
  }

  class SubController {
   public:
    SubController() = default;

    ~SubController() {
      std::cout << "in ~SubController" << std::endl;
    }

    std::shared_ptr<Controller> controller_;
  };

  std::shared_ptr<SubController> sub_controller_;
};

int main() {
  auto controller = std::make_shared<Controller>();
  auto sub_controller = std::make_shared<Controller::SubController>();

  controller->sub_controller_ = sub_controller;
  sub_controller->controller_ = controller;
  return 0;
}
```

编译并执行上述代码，发现并没有调用Controller和SubController的析构函数，我们尝试着打印下引用计数，代码如下：

```text
int main() {
  auto controller = std::make_shared<Controller>();
  auto sub_controller = std::make_shared<Controller::SubController>();

  controller->sub_controller_ = sub_controller;
  sub_controller->controller_ = controller;

  std::cout << "controller use_count: " << controller.use_count() << std::endl;
  std::cout << "sub_controller use_count: " << sub_controller.use_count() << std::endl;
  return 0;
}
```

编译并执行之后，输出如下：

```text
controller use_count: 2
sub_controller use_count: 2
```

通过上面输出可以发现，因为引用计数都是2，所以在main函数结束的时候，不会调用controller和sub_controller的析构函数，所以就出现了内存泄漏。

上面产生内存泄漏的原因，就是我们常说的循环引用。

![img](https://pic2.zhimg.com/80/v2-9f040f0db90f89242be50912390c770d_720w.webp)

为了解决std::shared_ptr循环引用导致的内存泄漏，我们可以使用std::weak_ptr来单面去除上图中的循环。

```text
class Controller {
 public:
  Controller() = default;

  ~Controller() {
    std::cout << "in ~Controller" << std::endl;
  }

  class SubController {
   public:
    SubController() = default;

    ~SubController() {
      std::cout << "in ~SubController" << std::endl;
    }

    std::weak_ptr<Controller> controller_;
  };

  std::shared_ptr<SubController> sub_controller_;
};
```

在上述代码中，我们将SubController类中controller_的类型从std::shared_ptr变成std::weak_ptr，重新编译执行，结果如下：

```text
controller use_count: 1
sub_controller use_count: 2
in ~Controller
in ~SubController
```

从上面结果可以看出，controller和sub_controller均以释放，所以循环引用引起的内存泄漏问题，也得以解决。

![img](https://pic3.zhimg.com/80/v2-06dfe5e79265dab1bd0ed9f8fb059592_720w.webp)

可能有人会问，使用std::shared_ptr可以直接访问对应的成员函数，如果是std::weak_ptr的话，怎么访问呢？我们可以使用下面的方式:

```text
std::shared_ptr controller = controller_.lock();
```

即在子类SubController中，如果要使用controller调用其对应的函数，就可以使用上面的方式。

## 10.避免

**避免在堆上分配**

众所周知，大部分的内存泄漏都是因为在堆上分配引起的，如果我们不在堆上进行分配，就不会存在内存泄漏了(这不废话嘛)，我们可以根据具体的使用场景，如果对象可以在栈上进行分配，就在栈上进行分配，一方面栈的效率远高于堆，另一方面，还能避免内存泄漏，我们何乐而不为呢。

**手动释放**

- 对于malloc函数分配的内存，在结束使用的时候，使用free函数进行释放
- 对于new操作符创建的对象，切记使用delete来进行释放
- 对于new []创建的对象，使用delete[]来进行释放(使用free或者delete均会造成内存泄漏)

**避免使用裸指针**

尽可能避免使用裸指针，除非所调用的lib库或者合作部门的接口是裸指针。

```text
int fun(int *ptr) {// fun 是一个接口或lib函数
  // do sth
  
  return 0;
}

int main() {}
  int a = 1000;
  int *ptr = &a;
  // ...
  fun(ptr);
  
  return 0;
}
```

在上面的fun函数中，有一个参数ptr,为int *，我们需要根据上下文来分析这个指针是否需要释放，这是一种很不好的设计

**使用STL中或者自己实现对象**

在C++中，提供了相对完善且可靠的STL供我们使用，所以能用STL的尽可能的避免使用C中的编程方式，比如：

- 使用std::string 替代char *, string类自己会进行内存管理，而且优化的相当不错
- 使用std::vector或者std::array来替代传统的数组
- 其它

## 11.智能指针

自C++11开始，STL中引入了智能指针(smart pointer)来动态管理资源，针对使用场景的不同，提供了以下三种智能指针。

**unique_ptr**

unique_ptr是限制最严格的一种智能指针，用来替代之前的auto_ptr，独享被管理对象指针所有权。当unique_ptr对象被销毁时，会在其析构函数内删除关联的原始指针。

unique_ptr对象分为以下两类：

- unique_ptr 该类型的对象关联了单个Type类型的指针
- std::unique_ptr<Type> p1(new Type); // c++11 auto p1 = std::make_unique<Type>(); // c++14
- unique_ptr<Type[]> 该类型的对象关联了多个Type类型指针，即一个对象数组
- std::unique_ptr<Type[]> p2(new Type[n]()); // c++11 auto p2 = std::make_unique<Type[]>(n); // c++14
- 不可用被复制
- unique_ptr<int> a(new int(0)); unique_ptr<int> b = a; // 编译错误 unique_ptr<int> b = std::move(a); // 可以通过move语义进行所有权转移

根据使用场景，可以使用std::unique_ptr来避免内存泄漏，如下：

```text
void fun() {
  unique_ptr<int> a(new int(0));
  // use a
}
```

在上述fun函数结束的时候，会自动调用a的析构函数，从而释放其关联的指针。

**shared_ptr**

与unique_ptr不同的是，unique_ptr是独占管理权，而shared_ptr则是共享管理权，即多个shared_ptr可以共用同一块关联对象，其内部采用的是引用计数，在拷贝的时候，引用计数+1，而在某个对象退出作用域或者释放的时候，引用计数-1，当引用计数为0的时候，会自动释放其管理的对象。

```text
void fun() {
  std::shared_ptr<Type> a; // a是一个空对象
  {
    std::shared_ptr<Type> b = std::make_shared<Type>(); // 分配资源
    a = b; // 此时引用计数为2
    {
      std::shared_ptr<Type> c = a; // 此时引用计数为3
    } // c退出作用域，此时引用计数为2
  } // b 退出作用域，此时引用计数为1
} // a 退出作用域，引用计数为0，释放对象
```

**weak_ptr**

weak_ptr的出现，主要是为了解决shared_ptr的循环引用，其主要是与shared_ptr一起来私用。和shared_ptr不同的地方在于，其并不会拥有资源，也就是说不能访问对象所提供的成员函数，不过，可以通过weak_ptr.lock()来产生一个拥有访问权限的shared_ptr。

```text
std::weak_ptr<Type> a;
{
  std::shared_ptr<Type> b = std::make_shared<Type>();
  a = b
} // b所对应的资源释放
```

**RAII**

RAII是Resource Acquisition is Initialization(资源获取即初始化)的缩写，是C++语言的一种管理资源，避免泄漏的用法。

利用的就是C++构造的对象最终会被销毁的原则。利用C++对象生命周期的概念来控制程序的资源,比如内存,文件句柄,网络连接等。

RAII的做法是使用一个对象，在其构造时获取对应的资源，在对象生命周期内控制对资源的访问，使之始终保持有效，最后在对象析构的时候，释放构造时获取的资源。

简单地说，就是把资源的使用限制在对象的生命周期之中，自动释放。

举个简单的例子，通常在多线程编程的时候，都会用到std::mutex,以下为例

```text
std::mutex mutex_;

void fun() {
  mutex_.lock();
  
  if (...) {
    mutex_.unlock();
    return;
  }
  
  mutex_.unlock()
}
```

在上述代码中，如果if分支多的话，每个if分支里面都要释放锁，如果一不小心忘记释放，那么就会造成故障，为了解决这个问题，我们使用RAII技术，代码如下：

```text
std::mutex mutex_;

void fun() {
  std::lock_guard<std::mutex> guard(mutex_);

  if (...) {
    return;
  }
}
```

在guard出了fun作用域的时候，会自动调用mutex_.lock()进行释放，避免了很多不必要的问题。

## 12.定位

在发现程序存在内存泄漏后，往往需要定位泄漏点，而定位这一步往往是最困难的，所以经常为了定位泄漏点，采取各种各样的方案，甭管方案优雅与否，毕竟管他白猫黑猫，抓住老鼠才是好猫，所以在本节，简单说下笔者这么多年定位泄漏点的方案，有些比较邪门歪道，您就随便看看就行。

**日志**

这种方案的核心思想，就是在每次分配内存的时候，打印指针地址，在释放内存的时候，打印内存地址，这样在程序结束的时候，通过分配和释放的差，如果分配的条数大于释放的条数，那么基本就能确定程序存在内存泄漏，然后根据日志进行详细分析和定位。

```text
char * fun() {
  char *p = (char*)malloc(20);
  printf("%s, %d, address is: %p", __FILE__, __LINE__, p);
  // do sth
  return p;
}

int main() {
  fun();
  
  return 0;
}
```

**统计**

统计方案可以理解为日志方案的一种特殊实现，其主要原理是在分配的时候，统计分配次数，在释放的时候，则是统计释放的次数，这样在程序结束前判断这俩值是否一致，就能判断出是否存在内存泄漏。

此方法可帮助跟踪已分配内存的状态。为了实现这个方案，需要创建三个自定义函数，一个用于内存分配，第二个用于内存释放，最后一个用于检查内存泄漏。代码如下：

```text
static unsigned int allocated  = 0;
static unsigned int deallocated  = 0;
void *Memory_Allocate (size_t size)
{
    void *ptr = NULL;
    ptr = malloc(size);
    if (NULL != ptr) {
        ++allocated;
    } else {
        //Log error
    }
    return ptr;
}
void Memory_Deallocate (void *ptr) {
    if(pvHandle != NULL) {
        free(ptr);
        ++deallocated;
    }
}
int Check_Memory_Leak(void) {
    int ret = 0;
    if (allocated != deallocated) {
        //Log error
        ret = MEMORY_LEAK;
    } else {
        ret = OK;
    }
    return ret;
}
```

**工具**

在Linux上比较常用的内存泄漏检测工具是valgrind，所以咱们就以valgrind为工具，进行检测。

我们首先看一段代码：

```text
#include <stdlib.h>

void func (void){
    char *buff = (char*)malloc(10);
}

int main (void){
    func(); // 产生内存泄漏
    return 0;
}
```

- 通过gcc -g leak.c -o leak命令进行编译
- 执行valgrind --leak-check=full ./leak

在上述的命令执行后，会输出如下：

```text
==9652== Memcheck, a memory error detector
==9652== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==9652== Using Valgrind-3.15.0 and LibVEX; rerun with -h for copyright info
==9652== Command: ./leak
==9652==
==9652==
==9652== HEAP SUMMARY:
==9652==     in use at exit: 10 bytes in 1 blocks
==9652==   total heap usage: 1 allocs, 0 frees, 10 bytes allocated
==9652==
==9652== 10 bytes in 1 blocks are definitely lost in loss record 1 of 1
==9652==    at 0x4C29F73: malloc (vg_replace_malloc.c:309)
==9652==    by 0x40052E: func (leak.c:4)
==9652==    by 0x40053D: main (leak.c:8)
==9652==
==9652== LEAK SUMMARY:
==9652==    definitely lost: 10 bytes in 1 blocks
==9652==    indirectly lost: 0 bytes in 0 blocks
==9652==      possibly lost: 0 bytes in 0 blocks
==9652==    still reachable: 0 bytes in 0 blocks
==9652==         suppressed: 0 bytes in 0 blocks
==9652==
==9652== For lists of detected and suppressed errors, rerun with: -s
==9652== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```

valgrind的检测信息将内存泄漏分为如下几类：

- definitely lost：确定产生内存泄漏
- indirectly lost：间接产生内存泄漏
- possibly lost：可能存在内存泄漏
- still reachable：即使在程序结束时候，仍然有指针在指向该块内存，常见于全局变量

主要上面输出的下面几句：

```text
==9652==    by 0x40052E: func (leak.c:4)
==9652==    by 0x40053D: main (leak.c:8)
SHELL 复制 全屏
```

提示在main函数(leak.c的第8行)fun函数(leak.c的第四行)产生了内存泄漏，通过分析代码，原因定位，问题解决。

valgrind不仅可以检测内存泄漏，还有其他很强大的功能，由于本文以内存泄漏为主，所以其他的功能就不在此赘述了，有兴趣的可以通过valgrind --help来进行查看

> 对于Windows下的内存泄漏检测工具，笔者推荐一款轻量级功能却非常强大的工具UMDH，笔者在十二年前，曾经在某外企负责内存泄漏，代码量几百万行，光编译就需要两个小时，尝试了各种工具(免费的和收费的)，最终发现了UMDH，如果你在Windows上进行开发，强烈推荐。

## 13.经验之谈

在C/C++开发过程中，内存泄漏是一个非常常见的问题，其影响相对来说远低于coredump等，所以遇到内存泄漏的时候，不用过于着急，大不了重启嘛。

在开发过程中遵守下面的规则，基本能90+%避免内存泄漏：

- 良好的编程习惯，只有有malloc/new，就得有free/delete
- 尽可能的使用智能指针，智能指针就是为了解决内存泄漏而产生
- 使用log进行记录
- 也是最重要的一点，谁申请，谁释放

对于malloc分配内存，分配失败的时候返回值为NULL，此时程序可以直接退出了，而对于new进行内存分配，其分配失败的时候，是抛出std::bad_alloc，所以为了第一时间发现问题，不要对new异常进行catch，毕竟内存都分配失败了，程序也没有运行的必要了。

如果我们上线后，发现程序存在内存泄漏，如果不严重的话，可以先暂时不管线上，同时进行排查定位；如果线上泄漏比较严重，那么第一时间根据实际情况来决定是否回滚。在定位问题点的时候，可以采用缩小范围法，着重分析这次新增的代码，这样能够有效缩短问题解决的时间。

## 14.结语

C/C++之所以复杂、效率高，是因为其灵活性，可用直接访问操作系统API，而正因为其灵活性，就很容易出问题，团队成员必须愿意按照一定的规则来进行开发，有完整的review机制，将问题暴露在上线之前。这样才可以把经历放在业务本身，而不是查找这些问题上，有时候往往一个小问题就能消耗很久的时间去定位解决，所以，一定要有一个良好的开发习惯。

原文地址：https://zhuanlan.zhihu.com/p/523628871

作者：CPP后端技术