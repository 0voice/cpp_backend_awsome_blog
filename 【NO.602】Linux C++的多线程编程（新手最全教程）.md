# 【NO.602】Linux C++的多线程编程（新手最全教程）

## 1. 引言

线程（thread）技术早在60年代就被提出，但真正应用多线程到操作系统中去，是在80年代中期，solaris是这方面的佼佼者。传统的Unix也支持线程的概念，但是在一个进程（process）中只允许有一个线程，这样多线程就意味着多进程。现在，多线程技术已经被许多操作系统所支持，包括Windows/NT，当然，也包括Linux。

　　为什么有了进程的概念后，还要再引入线程呢？使用多线程到底有哪些好处？什么的系统应该选用多线程？我们首先必须回答这些问题。

　　使用多线程的理由之一是和进程相比，它是一种非常"节俭"的多任务操作方式。我们知道，在Linux系统下，启动一个新的进程必须分配给它独立的地址空间，建立众多的数据表来维护它的代码段、堆栈段和数据段，这是一种"昂贵"的多任务工作方式。而运行于一个进程中的多个线程，它们彼此之间使用相同的地址空间，共享大部分数据，启动一个线程所花费的空间远远小于启动一个进程所花费的空间，而且，线程间彼此切换所需的时间也远远小于进程间切换所需要的时间。据统计，总的说来，一个进程的开销大约是一个线程开销的30倍左右，当然，在具体的系统上，这个数据可能会有较大的区别。

　　使用多线程的理由之二是线程间方便的通信机制。对不同进程来说，它们具有独立的数据空间，要进行数据的传递只能通过通信的方式进行，这种方式不仅费时，而且很不方便。线程则不然，由于同一进程下的线程之间共享数据空间，所以一个线程的数据可以直接为其它线程所用，这不仅快捷，而且方便。当然，数据的共享也带来其他一些问题，有的变量不能同时被两个线程所修改，有的子程序中声明为static的数据更有可能给多线程程序带来灾难性的打击，这些正是编写多线程程序时最需要注意的地方。

　　除了以上所说的优点外，不和进程比较，多线程程序作为一种多任务、并发的工作方式，当然有以下的优点：

  　　1) 提高应用程序响应。这对图形界面的程序尤其有意义，当一个操作耗时很长时，整个系统都会等待这个操作，此时程序不会响应键盘、鼠标、菜单的操作，而使用多线程技术，将耗时长的操作（time consuming）置于一个新的线程，可以避免这种尴尬的情况。

    　　2) 使多CPU系统更加有效。操作系统会保证当线程数不大于CPU数目时，不同的线程运行于不同的CPU上。

      　　3) 改善程序结构。一个既长又复杂的进程可以考虑分为多个线程，成为几个独立或半独立的运行部分，这样的程序会利于理解和修改。

　　下面我们先来尝试编写一个简单的多线程程序。

## 2. 简单的多线程编程

　　Linux系统下的多线程遵循POSIX线程接口，称为pthread。编写Linux下的多线程程序，需要使用头文件pthread.h，连接时需要使用库libpthread.a。顺便说一下，Linux下pthread的实现是通过系统调用clone()来实现的。clone()是Linux所特有的系统调用，它的使用方式类似fork，关于clone()的详细情况，有兴趣的读者可以去查看有关文档说明。下面我们展示一个最简单的多线程程序threads.cpp。

> //Threads.cpp
> \#include <iostream>
> \#include <unistd.h>
> \#include <pthread.h>
> using namespace std;
>
> void *thread(void *ptr)
> {
> for(int i = 0;i < 3;i++) {
> sleep(1);
> cout << "This is a pthread." << endl;
> }
> return 0;
> }
>
> int main() {
> pthread_t id;
> int ret = pthread_create(&id, NULL, thread, NULL);
> if(ret) {
> cout << "Create pthread error!" << endl;
> return 1;
> }
> for(int i = 0;i < 3;i++) {
> cout << "This is the main process." << endl;
> sleep(1);
> }
> pthread_join(id, NULL);
> return 0;
> }

　　我们编译并运行此程序，可以得到如下结果：

　　This is the main process.
　　This is a pthread.
　　This is the main process.
　　This is the main process.
　　This is a pthread.
　　This is a pthread.
　　再次运行，我们可能得到如下结果：
　　This is a pthread.
　　This is the main process.
　　This is a pthread.
　　This is the main process.
　　This is a pthread.
　　This is the main process.

　　前后两次结果不一样，这是两个线程争夺CPU资源的结果。上面的示例中，我们使用到了两个函数，pthread_create和pthread_join，并声明了一个pthread_t型的变量。

　　pthread_t在头文件/usr/include/bits/pthreadtypes.h中定义：

typedef unsigned long int pthread_t;

　　它是一个线程的标识符。函数pthread_create用来创建一个线程，它的原型为：

> extern int pthread_create __P ((pthread_t *__thread, __const pthread_attr_t *__attr,
> void *(*__start_routine) (void *), void *__arg));

　　第一个参数为指向线程标识符的指针，第二个参数用来设置线程属性，第三个参数是线程运行函数的起始地址，最后一个参数是运行函数的参数。这里，我们的函数thread不需要参数，所以最后一个参数设为空指针。第二个参数我们也设为空指针，这样将生成默认属性的线程。对线程属性的设定和修改我们将在下一节阐述。当创建线程成功时，函数返回0，若不为0则说明创建线程失败，常见的错误返回代码为EAGAIN和EINVAL。前者表示系统限制创建新的线程，例如线程数目过多了；后者表示第二个参数代表的线程属性值非法。创建线程成功后，新创建的线程则运行参数三和参数四确定的函数，原来的线程则继续运行下一行代码。

　　函数pthread_join用来等待一个线程的结束。函数原型为：

extern int pthread_join __P ((pthread_t __th, void **__thread_return));

　　第一个参数为被等待的线程标识符，第二个参数为一个用户定义的指针，它可以用来存储被等待线程的返回值。这个函数是一个线程阻塞的函数，调用它的函数将一直等待到被等待的线程结束为止，当函数返回时，被等待线程的资源被收回。一个线程的结束有两种途径，一种是象我们上面的例子一样，函数结束了，调用它的线程也就结束了；另一种方式是通过函数pthread_exit来实现。它的函数原型为：

extern void pthread_exit __P ((void *__retval)) __attribute__ ((__noreturn__));

　　唯一的参数是函数的返回代码，只要pthread_join中的第二个参数thread_return不是NULL，这个值将被传递给thread_return。最后要说明的是，一个线程不能被多个线程等待，否则第一个接收到信号的线程成功返回，其余调用pthread_join的线程则返回错误代码ESRCH。

　　在这一节里，我们编写了一个最简单的线程，并掌握了最常用的三个函数pthread_create，pthread_join和pthread_exit。下面，我们来了解线程的一些常用属性以及如何设置这些属性。

## 3. 修改线程的属性

　　在上一节的例子里，我们用pthread_create函数创建了一个线程，在这个线程中，我们使用了默认参数，即将该函数的第二个参数设为NULL。的确，对大多数程序来说，使用默认属性就够了，但我们还是有必要来了解一下线程的有关属性。

　　属性结构为pthread_attr_t，它同样在头文件/usr/include/pthread.h中定义，喜欢追根问底的人可以自己去查看。属性值不能直接设置，须使用相关函数进行操作，初始化的函数为pthread_attr_init，这个函数必须在pthread_create函数之前调用。属性对象主要包括是否绑定、是否分离、堆栈地址、堆栈大小、优先级。默认的属性为非绑定、非分离、缺省1M的堆栈、与父进程同样级别的优先级。

　　关于线程的绑定，牵涉到另外一个概念：轻进程（LWP：Light Weight Process）。轻进程可以理解为内核线程，它位于用户层和系统层之间。系统对线程资源的分配、对线程的控制是通过轻进程来实现的，一个轻进程可以控制一个或多个线程。默认状况下，启动多少轻进程、哪些轻进程来控制哪些线程是由系统来控制的，这种状况即称为非绑定的。绑定状况下，则顾名思义，即某个线程固定的"绑"在一个轻进程之上。被绑定的线程具有较高的响应速度，这是因为CPU时间片的调度是面向轻进程的，绑定的线程可以保证在需要的时候它总有一个轻进程可用。通过设置被绑定的轻进程的优先级和调度级可以使得绑定的线程满足诸如实时反应之类的要求。

　　设置线程绑定状态的函数为pthread_attr_setscope，它有两个参数，第一个是指向属性结构的指针，第二个是绑定类型，它有两个取值：PTHREAD_SCOPE_SYSTEM（绑定的）和PTHREAD_SCOPE_PROCESS（非绑定的）。下面的代码即创建了一个绑定的线程。

> \#include <pthread.h>
> pthread_attr_t attr;
> pthread_t tid;
> /*初始化属性值，均设为默认值*/
> pthread_attr_init(&attr);
> pthread_attr_setscope(&attr, PTHREAD_SCOPE_SYSTEM);
> pthread_create(&tid, &attr, (void *) my_function, NULL);

线程的分离状态决定一个线程以什么样的方式来终止自己。在上面的例子中，我们采用了线程的默认属性，即为非分离状态，这种情况下，原有的线程等待创建的线程结束。只有当pthread_join（）函数返回时，创建的线程才算终止，才能释放自己占用的系统资源。而分离线程不是这样子的，它没有被其他的线程所等待，自己运行结束了，线程也就终止了，马上释放系统资源。程序员应该根据自己的需要，选择适当的分离状态。设置线程分离状态的函数为pthread_attr_setdetachstate（pthread_attr_t *attr, int detachstate）。第二个参数可选为PTHREAD_CREATE_DETACHED（分离线程）和 PTHREAD _CREATE_JOINABLE（非分离线程）。这里要注意的一点是，如果设置一个线程为分离线程，而这个线程运行又非常快，它很可能在pthread_create函数返回之前就终止了，它终止以后就可能将线程号和系统资源移交给其他的线程使用，这样调用pthread_create的线程就得到了错误的线程号。要避免这种情况可以采取一定的同步措施，最简单的方法之一是可以在被创建的线程里调用pthread_cond_timewait函数，让这个线程等待一会儿，留出足够的时间让函数pthread_create返回。设置一段等待时间，是在多线程编程里常用的方法。但是注意不要使用诸如wait（）之类的函数，它们是使整个进程睡眠，并不能解决线程同步的问题。

　　另外一个可能常用的属性是线程的优先级，它存放在结构sched_param中。用函数pthread_attr_getschedparam和函数pthread_attr_setschedparam进行存放，一般说来，我们总是先取优先级，对取得的值修改后再存放回去。下面即是一段简单的例子。

> \#include <pthread.h>
> \#include <sched.h>
> pthread_attr_t attr;
> pthread_t tid;
> sched_param param;
> int newprio=20;
> pthread_attr_init(&amp;attr);
> pthread_attr_getschedparam(&attr, &param);
> param.sched_priority=newprio;
> pthread_attr_setschedparam(&attr, &param);
> pthread_create(&tid, &attr, (void *)myfunction, myarg);

## 4. 线程的数据处理

　　和进程相比，线程的最大优点之一是数据的共享性，各个进程共享父进程处沿袭的数据段，可以方便的获得、修改数据。但这也给多线程编程带来了许多问题。我们必须当心有多个不同的进程访问相同的变量。许多函数是不可重入的，即同时不能运行一个函数的多个拷贝（除非使用不同的数据段）。在函数中声明的静态变量常常带来问题，函数的返回值也会有问题。因为如果返回的是函数内部静态声明的空间的地址，则在一个线程调用该函数得到地址后使用该地址指向的数据时，别的线程可能调用此函数并修改了这一段数据。在进程中共享的变量必须用关键字volatile来定义，这是为了防止编译器在优化时（如gcc中使用-OX参数）改变它们的使用方式。为了保护变量，我们必须使用信号量、互斥等方法来保证我们对变量的正确使用。下面，我们就逐步介绍处理线程数据时的有关知识。

4.1 线程数据

　　在单线程的程序里，有两种基本的数据：全局变量和局部变量。但在多线程程序里，还有第三种数据类型：线程数据（TSD: Thread-Specific Data）。它和全局变量很象，在线程内部，各个函数可以象使用全局变量一样调用它，但它对线程外部的其它线程是不可见的。这种数据的必要性是显而易见的。例如我们常见的变量errno，它返回标准的出错信息。它显然不能是一个局部变量，几乎每个函数都应该可以调用它；但它又不能是一个全局变量，否则在A线程里输出的很可能是B线程的出错信息。要实现诸如此类的变量，我们就必须使用线程数据。我们为每个线程数据创建一个键，它和这个键相关联，在各个线程里，都使用这个键来指代线程数据，但在不同的线程里，这个键代表的数据是不同的，在同一个线程里，它代表同样的数据内容。

　　和线程数据相关的函数主要有4个：创建一个键；为一个键指定线程数据；从一个键读取线程数据；删除键。

　　创建键的函数原型为：

extern int pthread_key_create __P ((pthread_key_t *__key,void (*__destr_function) (void *)));

　　第一个参数为指向一个键值的指针，第二个参数指明了一个destructor函数，如果这个参数不为空，那么当每个线程结束时，系统将调用这个函数来释放绑定在这个键上的内存块。这个函数常和函数pthread_once ((pthread_once_t*once_control, void (*initroutine) (void)))一起使用，为了让这个键只被创建一次。函数pthread_once声明一个初始化函数，第一次调用pthread_once时它执行这个函数，以后的调用将被它忽略。

　　在下面的例子中，我们创建一个键，并将它和某个数据相关联。我们要定义一个函数createWindow，这个函数定义一个图形窗口（数据类型为Fl_Window *，这是图形界面开发工具FLTK中的数据类型）。由于各个线程都会调用这个函数，所以我们使用线程数据。

> /* 声明一个键*/
> pthread_key_t myWinKey;
> /* 函数 createWindow */
> void createWindow ( void ) {
> Fl_Window * win;
> static pthread_once_t once= PTHREAD_ONCE_INIT;
> /* 调用函数createMyKey，创建键*/
> pthread_once ( & once, createMyKey) ;
> /*win指向一个新建立的窗口*/
> win=new Fl_Window( 0, 0, 100, 100, "MyWindow");
> /* 对此窗口作一些可能的设置工作，如大小、位置、名称等*/
> setWindow(win);
> /* 将窗口指针值绑定在键myWinKey上*/
> pthread_setpecific ( myWinKey, win);
> }
>
> /* 函数 createMyKey，创建一个键，并指定了destructor */
> void createMyKey ( void ) {
> pthread_keycreate(&myWinKey, freeWinKey);
> }
>
> /* 函数 freeWinKey，释放空间*/
> void freeWinKey ( Fl_Window * win){
> delete win;
> }

　　这样，在不同的线程中调用函数createMyWin，都可以得到在线程内部均可见的窗口变量，这个变量通过函数pthread_getspecific得到。在上面的例子中，我们已经使用了函数pthread_setspecific来将线程数据和一个键绑定在一起。这两个函数的原型如下：

> extern int pthread_setspecific __P ((pthread_key_t __key,__const void *__pointer)); extern void *pthread_getspecific __P ((pthread_key_t __key));

　　这两个函数的参数意义和使用方法是显而易见的。要注意的是，用pthread_setspecific为一个键指定新的线程数据时，必须自己释放原有的线程数据以回收空间。这个过程函数pthread_key_delete用来删除一个键，这个键占用的内存将被释放，但同样要注意的是，它只释放键占用的内存，并不释放该键关联的线程数据所占用的内存资源，而且它也不会触发函数pthread_key_create中定义的destructor函数。线程数据的释放必须在释放键之前完成。

　　4.2 互斥锁

　　互斥锁用来保证一段时间内只有一个线程在执行一段代码。必要性显而易见：假设各个线程向同一个文件顺序写入数据，最后得到的结果一定是灾难性的。

　　我们先看下面一段代码。这是一个读/写程序，它们公用一个缓冲区，并且我们假定一个缓冲区只能保存一条信息。即缓冲区只有两个状态：有信息或没有信息。

> void reader_function ( void );
> void writer_function ( void );
> char buffer;
> int buffer_has_item=0;
> pthread_mutex_t mutex;
> struct timespec delay;
>
> void main ( void ){
> pthread_t reader;
> /* 定义延迟时间*/
> delay.tv_sec = 2;
> delay.tv_nec = 0;
> /* 用默认属性初始化一个互斥锁对象*/
> pthread_mutex_init (&mutex,NULL);
> pthread_create(&reader, pthread_attr_default, (void *)&reader_function), NULL);
> writer_function( );
> }
>
> void writer_function (void){
> while(1){
> /* 锁定互斥锁*/
> pthread_mutex_lock (&mutex);
> if (buffer_has_item==0){
> buffer=make_new_item( );
> buffer_has_item=1;
> }
> /* 打开互斥锁*/
> pthread_mutex_unlock(&mutex);
> pthread_delay_np(&delay);
> }
> }
>
> void reader_function(void){
> while(1){
> pthread_mutex_lock(&mutex);
> if(buffer_has_item==1){
> consume_item(buffer);
> buffer_has_item=0;
> }
> pthread_mutex_unlock(&mutex);
> pthread_delay_np(&delay);
> }
> }

　　这里声明了互斥锁变量mutex，结构pthread_mutex_t为不公开的数据类型，其中包含一个系统分配的属性对象。函数pthread_mutex_init用来生成一个互斥锁。NULL参数表明使用默认属性。如果需要声明特定属性的互斥锁，须调用函数pthread_mutexattr_init。函数pthread_mutexattr_setpshared和函数pthread_mutexattr_settype用来设置互斥锁属性。前一个函数设置属性pshared，它有两个取值，PTHREAD_PROCESS_PRIVATE和PTHREAD_PROCESS_SHARED。前者用来不同进程中的线程同步，后者用于同步本进程的不同线程。在上面的例子中，我们使用的是默认属性PTHREAD_PROCESS_ PRIVATE。后者用来设置互斥锁类型，可选的类型有PTHREAD_MUTEX_NORMAL、PTHREAD_MUTEX_ERRORCHECK、PTHREAD_MUTEX_RECURSIVE和PTHREAD _MUTEX_DEFAULT。它们分别定义了不同的上锁、解锁机制，一般情况下，选用最后一个默认属性。

　　pthread_mutex_lock声明开始用互斥锁上锁，此后的代码直至调用pthread_mutex_unlock为止，均被上锁，即同一时间只能被一个线程调用执行。当一个线程执行到pthread_mutex_lock处时，如果该锁此时被另一个线程使用，那此线程被阻塞，即程序将等待到另一个线程释放此互斥锁。在上面的例子中，我们使用了pthread_delay_np函数，让线程睡眠一段时间，就是为了防止一个线程始终占据此函数。

　　上面的例子非常简单，就不再介绍了，需要提出的是在使用互斥锁的过程中很有可能会出现死锁：两个线程试图同时占用两个资源，并按不同的次序锁定相应的互斥锁，例如两个线程都需要锁定互斥锁1和互斥锁2，a线程先锁定互斥锁1，b线程先锁定互斥锁2，这时就出现了死锁。此时我们可以使用函数pthread_mutex_trylock，它是函数pthread_mutex_lock的非阻塞版本，当它发现死锁不可避免时，它会返回相应的信息，程序员可以针对死锁做出相应的处理。另外不同的互斥锁类型对死锁的处理不一样，但最主要的还是要程序员自己在程序设计注意这一点。

　　4.3 条件变量

　　前一节中我们讲述了如何使用互斥锁来实现线程间数据的共享和通信，互斥锁一个明显的缺点是它只有两种状态：锁定和非锁定。而条件变量通过允许线程阻塞和等待另一个线程发送信号的方法弥补了互斥锁的不足，它常和互斥锁一起使用。使用时，条件变量被用来阻塞一个线程，当条件不满足时，线程往往解开相应的互斥锁并等待条件发生变化。一旦其它的某个线程改变了条件变量，它将通知相应的条件变量唤醒一个或多个正被此条件变量阻塞的线程。这些线程将重新锁定互斥锁并重新测试条件是否满足。一般说来，条件变量被用来进行线承间的同步。

　　条件变量的结构为pthread_cond_t，函数pthread_cond_init（）被用来初始化一个条件变量。它的原型为：

extern int pthread_cond_init __P ((pthread_cond_t *__cond,__const pthread_condattr_t *__cond_attr));

　　其中cond是一个指向结构pthread_cond_t的指针，cond_attr是一个指向结构pthread_condattr_t的指针。结构pthread_condattr_t是条件变量的属性结构，和互斥锁一样我们可以用它来设置条件变量是进程内可用还是进程间可用，默认值是PTHREAD_ PROCESS_PRIVATE，即此条件变量被同一进程内的各个线程使用。注意初始化条件变量只有未被使用时才能重新初始化或被释放。释放一个条件变量的函数为pthread_cond_ destroy（pthread_cond_t cond）。　

　　函数pthread_cond_wait（）使线程阻塞在一个条件变量上。

　　它的函数原型为：

extern int pthread_cond_wait __P ((pthread_cond_t *__cond, pthread_mutex_t *__mutex));

　　线程解开mutex指向的锁并被条件变量cond阻塞。线程可以被函数pthread_cond_signal和函数pthread_cond_broadcast唤醒，但是要注意的是，条件变量只是起阻塞和唤醒线程的作用，具体的判断条件还需用户给出，例如一个变量是否为0等等，这一点我们从后面的例子中可以看到。线程被唤醒后，它将重新检查判断条件是否满足，如果还不满足，一般说来线程应该仍阻塞在这里，被等待被下一次唤醒。这个过程一般用while语句实现。

　　另一个用来阻塞线程的函数是pthread_cond_timedwait()，它的原型为：

extern int pthread_cond_timedwait __P ((pthread_cond_t *__cond, pthread_mutex_t *__mutex, __const struct timespec *__abstime));

　　它比函数pthread_cond_wait()多了一个时间参数，经历abstime段时间后，即使条件变量不满足，阻塞也被解除。

　　函数pthread_cond_signal()的原型为：

　　extern int pthread_cond_signal __P ((pthread_cond_t *__cond));

　　它用来释放被阻塞在条件变量cond上的一个线程。多个线程阻塞在此条件变量上时，哪一个线程被唤醒是由线程的调度策略所决定的。要注意的是，必须用保护条件变量的互斥锁来保护这个函数，否则条件满足信号又可能在测试条件和调用pthread_cond_wait函数之间被发出，从而造成无限制的等待。下面是使用函数pthread_cond_wait()和函数

> pthread_cond_signal()的一个简单的例子。
> pthread_mutex_t count_lock;
> pthread_cond_t count_nonzero;
> unsigned count;
> decrement_count () {
> pthread_mutex_lock (&count_lock);
> while(count==0)
> pthread_cond_wait( &count_nonzero, &count_lock);
> count=count -1;
> pthread_mutex_unlock (&count_lock);
> }
>
> increment_count(){
> pthread_mutex_lock(&count_lock);
> if(count==0)
> pthread_cond_signal(&count_nonzero);
> count=count+1;
> pthread_mutex_unlock(&count_lock);
> }

　　count值为0时，decrement函数在pthread_cond_wait处被阻塞，并打开互斥锁count_lock。此时，当调用到函数increment_count时，pthread_cond_signal（）函数改变条件变量，告知decrement_count（）停止阻塞。读者可以试着让两个线程分别运行这两个函数，看看会出现什么样的结果。

　　函数pthread_cond_broadcast（pthread_cond_t *cond）用来唤醒所有被阻塞在条件变量cond上的线程。这些线程被唤醒后将再次竞争相应的互斥锁，所以必须小心使用这个函数。

　　4.4 信号量

　　信号量本质上是一个非负的整数计数器，它被用来控制对公共资源的访问。当公共资源增加时，调用函数sem_post（）增加信号量。只有当信号量值大于０时，才能使用公共资源，使用后，函数sem_wait（）减少信号量。函数sem_trywait（）和函数pthread_ mutex_trylock（）起同样的作用，它是函数sem_wait（）的非阻塞版本。下面我们逐个介绍和信号量有关的一些函数，它们都在头文件/usr/include/semaphore.h中定义。

　　信号量的数据类型为结构sem_t，它本质上是一个长整型的数。函数sem_init（）用来初始化一个信号量。它的原型为：

　　extern int sem_init __P ((sem_t *__sem, int __pshared, unsigned int __value));

　　sem为指向信号量结构的一个指针；pshared不为０时此信号量在进程间共享，否则只能为当前进程的所有线程共享；value给出了信号量的初始值。

　　函数sem_post( sem_t *sem )用来增加信号量的值。当有线程阻塞在这个信号量上时，调用这个函数会使其中的一个线程不在阻塞，选择机制同样是由线程的调度策略决定的。

　　函数sem_wait( sem_t *sem )被用来阻塞当前线程直到信号量sem的值大于0，解除阻塞后将sem的值减一，表明公共资源经使用后减少。函数sem_trywait ( sem_t *sem )是函数sem_wait（）的非阻塞版本，它直接将信号量sem的值减一。

　　函数sem_destroy(sem_t *sem)用来释放信号量sem。

　　下面我们来看一个使用信号量的例子。在这个例子中，一共有4个线程，其中两个线程负责从文件读取数据到公共的缓冲区，另两个线程从缓冲区读取数据作不同的处理（加和乘运算）。

> /* File sem.c */
> \#include <stdio.h>
> \#include <pthread.h>
> \#include <semaphore.h>
> \#define MAXSTACK 100
> int stack[MAXSTACK][2];
> int size=0;
> sem_t sem;
>
> /* 从文件1.dat读取数据，每读一次，信号量加一*/
> void ReadData1(void){
> FILE *fp=fopen("1.dat","r");
> while(!feof(fp)){
> fscanf(fp,"%d %d",&stack[size][0],&stack[size][1]);
> sem_post(&sem);
> ++size;
> }
> fclose(fp);
> }
>
> /*从文件2.dat读取数据*/
> void ReadData2(void){
> FILE *fp=fopen("2.dat","r");
> while(!feof(fp)){
> fscanf(fp,"%d %d",&stack[size][0],&stack[size][1]);
> sem_post(&sem);
> ++size;
> }
> fclose(fp);
> }
> /*阻塞等待缓冲区有数据，读取数据后，释放空间，继续等待*/
> void HandleData1(void){
> while(1){
> sem_wait(&sem);
> printf("Plus:%d+%d=%d\n",stack[size][0],stack[size][1],
> stack[size][0]+stack[size][1]);
> --size;
> }
> }
>
> void HandleData2(void){
> while(1){
> sem_wait(&sem);
> printf("Multiply:%d*%d=%d\n",stack[size][0],stack[size][1],
> stack[size][0]*stack[size][1]);
> --size;
> }
> }
>
> int main(void){
> pthread_t t1,t2,t3,t4;
> sem_init(&sem,0,0);
> pthread_create(&t1,NULL,(void *)HandleData1,NULL);
> pthread_create(&t2,NULL,(void *)HandleData2,NULL);
> pthread_create(&t3,NULL,(void *)ReadData1,NULL);
> pthread_create(&t4,NULL,(void *)ReadData2,NULL);
> /* 防止程序过早退出，让它在此无限期等待*/
> pthread_join(t1,NULL);
> }

　　在Linux下，我们用命令gcc -lpthread sem.c -o sem生成可执行文件sem。 我们事先编辑好数据文件1.dat和2.dat，假设它们的内容分别为1 2 3 4 5 6 7 8 9 10和 -1 -2 -3 -4 -5 -6 -7 -8 -9 -10 ，我们运行sem，得到如下的结果：

　　Multiply:-1*-2=2
　　Plus:-1+-2=-3
　　Multiply:9*10=90
　　Plus:-9+-10=-19
　　Multiply:-7*-8=56
　　Plus:-5+-6=-11
　　Multiply:-3*-4=12
　　Plus:9+10=19
　　Plus:7+8=15
　　Plus:5+6=11

　　从中我们可以看出各个线程间的竞争关系。而数值并未按我们原先的顺序显示出来这是由于size这个数值被各个线程任意修改的缘故。这也往往是多线程编程要注意的问题。

## 5. 小结

　　多线程编程是一个很有意思也很有用的技术，使用多线程技术的网络蚂蚁是目前最常用的下载工具之一，使用多线程技术的grep比单线程的grep要快上几倍，类似的例子还有很多。希望大家能用多线程技术写出高效实用的好程序来。

原文地址：https://zhuanlan.zhihu.com/p/517076696

作者：CPP后端技术