# 【NO.143】C++面试集锦( 面试被问到的问题）

# **1. C 和 C++ 区别**

  **
**

# **2. const 有什么用途**

  主要有三点：

   1：定义只读变量，即常量 

   2：修饰函数的参数和函数的返回值 

   3： 修饰函数的定义体，这里的函数为类的成员函数，被const修饰的成员函数代表不修改成员变量的值**
**

 

# **3. 指针和引用的区别**

  1：引用是变量的一个别名，内部实现是只读指针

  2：引用只能在初始化时被赋值，其他时候值不能被改变，指针的值可以在任何时候被改变

  3：引用不能为NULL，指针可以为NULL

  4：引用变量内存单元保存的是被引用变量的地址

  5：“sizeof 引用" = 指向变量的大小 ， "sizeof 指针"= 指针本身的大小

  6：引用可以取地址操作，返回的是被引用变量本身所在的内存单元地址

  7：引用使用在源代码级相当于普通的变量一样使用，做函数参数时，内部传递的实际是变量地址

 

# **4. C++中有了malloc / free , 为什么还需要 new / delete**   

```
  1,malloc与free是C++/C语言的标准库函数，new/delete是C++的运算符。它们都可用于申请动态内存和释放内存。
  2,对于非内部数据类型的对象而言，光用maloc/free无法满足动态对象的要求。
     对象在创建的同时要自动执行构造函数，对象在消亡之前要自动执行析构函数。
     由于malloc/free是库函数而不是运算符，不在编译器控制权限之内，不能够把执行构造函数和析构函数的任务强加于malloc/free。
  3,因此C++语言需要一个能完成动态内存分配和初始化工作的运算符new，以一个能完成清理与释放内存工作的运算符delete。注意new/delete不是库函数。
```

 

 

# **5. 编写类String 的构造函数，析构函数，拷贝构造函数和赋值函数**

 

# **6. 多态的实现**

# **7. 单链表的逆置**

 

# **8. 堆和栈的区别**  

```
  一个由c/C++编译的程序占用的内存分为以下几个部分 
  1、栈区（stack）―   由编译器自动分配释放 ，存放函数的参数值，局部变量的值等。其操作方式类似于数据结构中的栈。 
  2、堆区（heap） ―   一般由程序员分配释放， 若程序员不释放，程序结束时可能由OS回收 。
     注意它与数据结构中的堆是两回事，分配方式倒是类似于链表，呵呵。 
  3、全局区（静态区）（static）―，全局变量和静态变量的存储是放在一块的，
     初始化的全局变量和静态变量在一块区域， 未初始化的全局变量和未初始化的静态变量在相邻的另一块区域。 - 程序结束后有系统释放 
  4、文字常量区  ―常量字符串就是放在这里的。 程序结束后由系统释放 
  5、程序代码区―存放函数体的二进制代码。
```

 

 

# 10. 不调用C/C++ 的字符串库函数，编写strcpy

**

```
   char * strcpy(char * strDest,const char * strSrc)
        {
                if ((strDest==NULL)||strSrc==NULL))                     
                   return NULL;    
                char * strDestCopy=strDest; 
                while ((*strDest++=*strSrc++)!='\0'); 
                *strDest = '\0';
                return strDestCopy;
        }
```

 

 

# 11. 关键字static的作用

**

  \1. 函数体内 static 变量的作用范围为该函数体，不同于 auto 变量， 该变量的内存只被分配一次，因此其值在下次调用时仍维持上次的值

  \2. 在模块内的 static 全局变量可以被模块内所有函数访问，但不能被模块外其他函数访问

  \3. 在模块内的static 函数只可被这一模块内的其他函数调用，这个函数的使用范围被限制在声明它的模块内

  \4. 在类的static 成员变量属于整个类所拥有，对类的所以对象只有一份拷贝

  \5. 在类中的 static 成员函数属于整个类所拥有，这个函数不接收 this 指针，因而只能访问类的 static 成员变量

  

   介绍它最重要的一条：隐藏。（static函数，static变量均可） --> 对应上面的2、3项
    当同时编译多个文件时，所有未加static前缀的全局变量和函数都具有全局可见性。
    举例来说明。同时编译两个源文件，一个是a.c，另一个是main.c。

```
   //a.c
    char a = 'A';               // global variable
    void msg()
    {
      printf("Hello\n");
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
  //main.c
   int main()
   {
     extern char a;       // extern variable must be declared before use
     printf("%c ", a);
     (void)msg();
     return 0;
   }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

    程序的运行结果是：

   A Hello

 

 

   为什么在a.c中定义的全局变量a和函数msg能在main.c中使用？

   前面说过，所有未加static前缀的全局变量和函数都具有全局可见性，其它的源文件也能访问。此例中，a是全局变量，msg是函数，并且都没有加static前缀，

    因此对于另外的源文件main.c是可见的。

   如果加了static，就会对其它源文件隐藏。例如在a和msg的定义前加上static，main.c就看不到它们了。

   利用这一特性可以在不同的文件中定义同名函数和同名变量，而不必担心命名冲突。static可以用作函数和变量的前缀，对于函数来讲，static的作用仅限于隐藏

 

# **12. 在c++程序中调用被C编译器编译后的函数，为什么要加extern“C”**

   C++语言支持函数重载，[C语言](http://lib.csdn.net/base/c)不支持函数重载，函数被C++编译器编译后在库中的名字与C语言的不同，

   假设某个函数原型为：

1.      void foo(int x, inty);

  该函数被C编译器编译后在库中的名字为: _foo

  而C++编译器则会产生像: _foo_int_int  之类的名字。
  为了解决此类名字匹配的问题，C++提供了C链接交换指定符号 extern "C"。

 

 

# **13. 头文件种的ifndef/define/endif 是干什么用的**

   防止头文件被重复包含

 

# **14. 线程和进程的联系和区别**

   **http://blog.csdn[.NET](http://lib.csdn.net/base/dotnet)/wolenski/article/details/7969908**

 

# **15. 线程有哪几种状态**

   **http://blog.csdn[.Net](http://lib.csdn.net/base/dotnet)/wolenski/article/details/7969908**

 

# **16. 进程间的通信方式**

   管道、有名管道、信号、共享内存、消息队列、信号量、套接字、文件.

 

# **17. 线程同步和线程互斥的区别**

  **http://blog.csdn.net/wolenski/article/details/7969908**

 

# **18. 线程同步的方式**

   **[Linux](http://lib.csdn.net/base/linux):**  互斥锁、条件变量和信号量

   **http://blog.csdn.net/zsf8701/article/details/7844316

**

 

# **19. 网络七层**

  **
**

# **20. TCP和UDP有什么区别**

   TCP---传输控制协议,提供的是面向连接、可靠的字节流服务。

         当客户和服务器彼此交换数据前，必须先在双方之间建立一个TCP连接，之后才能传输数据。
    
         TCP提供超时重发，丢弃重复数据，检验数据，流量控制等功能，保证数据能从一端传到另一端。

   UDP---用户数据报协议，是一个简单的面向数据报的运输层协议。

         UDP不提供可靠性，它只是把应用程序传给IP层的数据报发送出去，但是并不能保证它们能到达目的地。
    
         由于UDP在传输数据报前不用在客户和服务器之间建立一个连接，且没有超时重发等机制，故而传输速度很快**
**

 

# **21. 编写socket套接字的步骤**

 

# **22. TCP三次握手和四次挥手, 以及各个状态的作用**

   **http://hi.baidu.com/suxinpingtao51/item/be5f71b3a907dbef4ec7fd0e?qq-pf-to=pcqq.c2c**

 

# **23. HTTP协议**

      http（超文本传输协议）是一个基于请求与响应模式的、无状态的、应用层的协议，常基于TCP的连接方式，

   HTTP1.1版本中给出一种持续连接的机制，绝大多数的Web开发，都是构建在HTTP协议之上的Web应用。

  TCP 和 HTTP区别： http://blog.csdn.net/lemonxuexue/article/details/4485877

 

# **24. 使用过的 shell 命令**

    **cp , mv , rm , mkdir , touch , pwd , cd , ls , top , cat , tail , less , df , du , man , find , kill , sudo , cat 

**

 

# **25. 使用过的 vim 命令**

    wq!, dd , dw , yy , p , i , %s/old/new/g , /abc 向后搜索字符串abc ， ？abc向前搜索字符串abc**
**

 

# **26. 使用过的 gdb 命令**

   **http://blog.csdn.net/dadalan/article/details/3758025**

 

# **27. 常见[算法](http://lib.csdn.net/base/datastructure)**

    快速排序、堆排序和归并排序
    
    **堆排序 ： http://blog.csdn.net/xiaoxiaoxuewen/article/details/7570621**
    
    **快速排序、归并排序： http://blog.csdn.net/morewindows/article/details/6684558

**

    稳定性分析 **http://baike.baidu.com/link?url=ueoZ3sNIOvMNPrdCKbd8mhfebC85B4nRc-7hPEJWi-hFo5ROyWH2Pxs9RtvLFRJL

**

 

# **28. C库函数实现**

 

# **29. 静态链表和动态链表的区别**

   http://blog.csdn.net/toonny1985/article/details/4868786

 

# **31. 大并发( epoll )**

    **优点:**
    
       **http://blog.csdn.net/sunyurun/article/details/8194979

**

    **实例：**
    
       **http://www.cnblogs.com/ggjucheng/archive/2012/01/17/2324974.html**
    
    **
**

# **32. 海量数据处理的知识点，（hash表， hash统计）**

  hash表： http://hi.baidu.com/05104106/item/62736054402852c09e26679b

  海量数据处理方法： http://blog.csdn.net/v_july_v/article/details/7382693

 

 

# **33. 什么时候要用虚析构函数**

    通过基类的指针来删除派生类的对象时，基类的析构函数应该是虚的。否则其删除效果将无法实现。
    
    一般情况下，这样的删除只能够删除基类对象，而不能删除子类对象，形成了删除一半形象，从而千万内存泄漏。

   原因：

       在公有继承中，基类对派生类及其对象的操作，只能影响到那些从基类继承下来的成员。
    
       如果想要用基类对非继承成员进行操作，则要把基类的这个操作（函数）定义为虚函数。
       那么，析构函数自然也应该如此：如果它想析构子类中的重新定义或新的成员及对象，当然也应该声明为虚的。

   注意：

   如果不需要基类对派生类及对象进行操作，则不能定义虚函数（包括虚析构函数），因为这样会增加内存开销。

 

# **34. c++怎样让返回对象的函数不调用拷贝构造函数**

  拷贝构造函数前加 “explicit” 关键字

# **35. 孤儿进程和僵尸进程**

  http://www.cnblogs.com/Anker/p/3271773.html

原文作者：Y1
原文链接：https://www.cnblogs.com/Y1Focus/p/6707121.html