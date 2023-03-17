# 【NO.614】C++多线程编程，线程互斥和同步通信，死锁问题分析解决

## 1.C++11的多线程类thread

C++11之前，C++库中没有提供和线程相关的类或者接口，因此在编写多线程程序时，Windows上需要调用CreateThread创建线程，Linux下需要调用clone或者pthread线程库的接口函数pthread_create来创建线程。但是这样是直接调用了系统相关的API函数，编写的代码，无法做到跨平台编译运行。

C++11之后提供了thread线程类，可以很方便的编写多线程程序（注意：编译器需要支持C++11之后的语法），代码示例如下：

```text
#include <iostream>
#include <thread>
#include <string>
using namespace std;

// 线程1的线程函数
void threadProc1()
{
	cout << "thread-1 run begin!" << endl;
	// 线程1睡眠2秒
	std::this_thread::sleep_for(std::chrono::seconds(2));
	cout << "thread-1 2秒唤醒，run end!" << endl;
}

// 线程2的线程函数
void threadProc2(int val, string info)
{
	cout << "thread-2 run begin!" << endl;
	cout << "thread-2 args[val:" << val << ",info:" << info << "]" << endl;
	// 线程2睡眠4秒
	std::this_thread::sleep_for(std::chrono::seconds(4));
	cout << "thread-2 4秒唤醒，run end!" << endl;
}
int main()
{
	cout << "main thread begin!" << endl;

	// 创建thread线程对象，传入线程函数和参数，线程直接启动运行
	thread t(threadProc1);
	thread t1(threadProc2, 20, "hello world");

	// 等待线程t和t1执行完，main线程再继续运行
	t.join();
	t1.join();

	cout << "main thread end!" << endl;
	return 0;
}
```

代码运行打印如下：

```text
main thread begin!
thread-1 run begin!
thread-2 run begin!
thread-2 args[val:20,info:hello world]
thread-1 2秒唤醒，run end!
thread-2 4秒唤醒，run end!
main thread end!
```

可以看到，在C++语言层面编写多线程程序，用thread线程类非常简单，定义thread对象，只需要传入相应的线程函数和参数就可以了。

上面同样的代码在Linux平台下面用g++编译：

g++ 源文件名字.cpp -lpthread

【注意】：需要链接pthread线程动态库，所以C++的thread类在Linux环境下使用的就是pthread线程库的相关接口。

然后用strace命令跟踪程序的启动过程：

tony@tony-virtual-machine:~/code$ strace ./a.out

有如下打印输出：

![img](https://pic1.zhimg.com/80/v2-a30aeeaa08f2f30e64d116c0a3801ef8_720w.webp)

说明C++ thread线程对象启动线程的调用过程就是 thread->pthread_create->clone，还是Linux pthread线程库使用的那一套，好处就是现在可以跨平台编译运行了，在Windows上当然调用的就是CreateThread系统API创建线程了。

## 2.线程互斥

在多线程环境中运行的代码段，需要考虑是否存在竞态条件，如果存在竞态条件，我们就说该代码段不是线程安全的，不能直接运行在多线程环境当中，对于这样的代码段，我们经常称之为临界区资源，对于临界区资源，多线程环境下需要保证它以原子操作执行，要保证临界区的原子操作，就需要用到线程间的互斥操作-锁机制，thread类库还提供了更轻量级的基于CAS操作的原子操作类。

下面用模拟3个窗口同时卖票的场景，用代码示例一下线程间的互斥操作。

### 2.1 thread线程类库的互斥锁mutex

下面这段代码，启动三个线程模拟三个窗口同时卖票，总票数是100张，由于整数的- -操作不是线程安全的操作，因为多线程环境中，需要通过加互斥锁做到线程安全，代码如下示例：

```text
// 车票总数是100张
volatile int tickets = 100; 
// 全局的互斥锁
std::mutex mtx;

// 线程函数
void sellTicketTask(std::string wndName)
{
	while (tickets > 0)
	{
		// 获取互斥锁资源
		mtx.lock();
		if (tickets > 0)
		{
			std::cout << wndName << " 售卖第" << tickets << "张票" << std::endl;
			tickets--;
		}
		// 释放互斥锁资源
		mtx.unlock();

		// 每卖出一张票，睡眠100ms，让每个窗口都有机会卖票
		std::this_thread::sleep_for(std::chrono::milliseconds(100));
	}
}

// 模拟车站窗口卖票，使用C++11 线程互斥锁mutex
int main()
{
	// 创建三个模拟窗口卖票线程
	std::thread t1(sellTicketTask, "车票窗口一");
	std::thread t2(sellTicketTask, "车票窗口二");
	std::thread t3(sellTicketTask, "车票窗口三");

	// 等待三个线程执行完成
	t1.join();
	t2.join();
	t3.join();

	return 0;
}
```

通过上面的代码可以看到，C++11的mutex和Linux平台下pthread线程库的pthread_mutex_t互斥锁使用几乎是一样的（实际上在Linux平台下mutex就是调用的pthread_mutex_t互斥锁相关的系统函数），mutex也支持trylock活锁机制，可以自己进行测试。

### 2.2 thread线程类库基于CAS的原子类

实际上，上面代码中因为tickets车票数量是整数，因此它的- -操作需要在多线程环境下添加互斥操作，但是mutex互斥锁毕竟比较重，对于系统消耗有些大，C++11的thread类库提供了针对简单类型的原子操作类，如std::atomic_int，atomic_long，atomic_bool等，它们值的增减都是基于CAS操作的，既保证了线程安全，效率还非常高。

下面代码示例开启10个线程，每个线程对整数增加1000次，保证线程安全的情况下，应该加到10000次，这种情况下，可以用atomic_int来实现，代码示例如下：

```text
#include <iostream>
#include <atomic> // C++11线程库提供的原子类
#include <thread> // C++线程类库的头文件
#include <vector>

// 原子整形，CAS操作保证给count自增自减的原子操作
std::atomic_int count = 0;

// 线程函数
void sumTask()
{
	// 每个线程给count加1000次
	for (int i = 0; i < 1000; ++i)
	{
		count++;
	}
}

int main()
{
	// 创建10个线程放在容器当中
	std::vector<std::thread> vec;
	for (int i = 0; i < 10; ++i)
	{
		vec.push_back(std::thread(sumTask));
	}

	// 等待线程执行完成
	for (int i = 0; i < vec.size(); ++i)
	{
		vec[i].join();
	}

	// 所有子线程运行结束，count的结果每次运行应该都是10000
	std::cout << "count : " << count << std::endl;

	return 0;
}
```

实际上，C++11类库的原子操作类，在Linux平台下调用的也是CAS（compare_and_set）相关的系统接口。

### 2.3 线程同步通信

多线程在运行过程中，各个线程都是随着OS的调度算法，占用CPU时间片来执行指令做事情，每个线程的运行完全没有顺序可言。但是在某些应用场景下，一个线程需要等待另外一个线程的运行结果，才能继续往下执行，这就需要涉及线程之间的同步通信机制。

线程间同步通信最典型的例子就是生产者-消费者模型，生产者线程生产出产品以后，会通知消费者线程去消费产品；如果消费者线程去消费产品，发现还没有产品生产出来，它需要通知生产者线程赶快生产产品，等生产者线程生产出产品以后，消费者线程才能继续往下执行。

C++11 线程库提供的条件变量condition_variable，就是Linux平台下的Condition Variable机制，用于解决线程间的同步通信问题，下面通过代码演示一个生产者-消费者线程模型，仔细分析代码：

```text
#include <iostream>           // std::cout
#include <thread>             // std::thread
#include <mutex>              // std::mutex, std::unique_lock
#include <condition_variable> // std::condition_variable
#include <vector>

// 定义互斥锁(条件变量需要和互斥锁一起使用)
std::mutex mtx;
// 定义条件变量(用来做线程间的同步通信)
std::condition_variable cv;
// 定义vector容器，作为生产者和消费者共享的容器
std::vector<int> vec;

// 生产者线程函数
void producer()
{
	// 生产者每生产一个，就通知消费者消费一个
	for (int i = 1; i <= 10; ++i)
	{
		// 获取mtx互斥锁资源
		std::unique_lock<std::mutex> lock(mtx);

		// 如果容器不为空，代表还有产品未消费，等待消费者线程消费完，再生产
		while (!vec.empty())
		{
			// 判断容器不为空，进入等待条件变量的状态，释放mtx锁，
			// 让消费者线程抢到锁能够去消费产品
			cv.wait(lock);
		}
		vec.push_back(i); // 表示生产者生产的产品序号i
		std::cout << "producer生产产品:" << i << std::endl;

		/* 
		生产者线程生产完产品，通知等待在cv条件变量上的消费者线程，
		可以开始消费产品了，然后释放锁mtx
		*/
		cv.notify_all();

		// 生产一个产品，睡眠100ms
		std::this_thread::sleep_for(std::chrono::milliseconds(100));
	}
}
// 消费者线程函数
void consumer()
{
	// 消费者每消费一个，就通知生产者生产一个
	for (int i = 1; i <= 10; ++i)
	{
		// 获取mtx互斥锁资源
		std::unique_lock<std::mutex> lock(mtx);

		// 如果容器为空，代表还有没有产品可消费，等待生产者生产，再消费
		while (vec.empty())
		{
			// 判断容器为空，进入等待条件变量的状态，释放mtx锁，
			// 让生产者线程抢到锁能够去生产产品
			cv.wait(lock);
		}
		int data = vec.back(); // 表示消费者消费的产品序号i
		vec.pop_back();
		std::cout << "consumer消费产品:" << data << std::endl;

		/*
		消费者消费完产品，通知等待在cv条件变量上的生产者线程，
		可以开始生产产品了，然后释放锁mtx
		*/
		cv.notify_all();

		// 消费一个产品，睡眠100ms
		std::this_thread::sleep_for(std::chrono::milliseconds(100));
	}
}
int main()
{
	// 创建生产者和消费者线程
	std::thread t1(producer);
	std::thread t2(consumer);

	// main主线程等待所有子线程执行完
	t1.join();
	t2.join();

	return 0;
}
```

代码运行结果如下，可以看到，生产者和消费者线程交替生产产品和消费产品，两个线程之间进行了完美的通信协调运行。

```text
producer生产产品:1
consumer消费产品:1
producer生产产品:2
consumer消费产品:2
producer生产产品:3
consumer消费产品:3
producer生产产品:4
consumer消费产品:4
producer生产产品:5
consumer消费产品:5
producer生产产品:6
consumer消费产品:6
producer生产产品:7
consumer消费产品:7
producer生产产品:8
consumer消费产品:8
producer生产产品:9
consumer消费产品:9
producer生产产品:10
consumer消费产品:10
```

## 3.死锁问题案例分析解决

死锁的问题经常会考察到，面对哪些情况下会程序会发生死锁的问题，与其想着怎么把书上的理论背出来，不如从实践的角度举例说明，如何对死锁的问题进行分析定位，然后找到问题点进行修改。

当我们的程序运行时，出现假死的现象，有可能是程序死循环了，有可能是程序等待的I/O、网络事件没发生导致程序阻塞了，也有可能是程序死锁了，下面举例说明在Linux系统下如何分许我们程序的死锁问题。

示例：

当一个程序的多个线程获取多个互斥锁资源的时候，就有可能发生死锁问题，比如线程A先获取了锁1，线程B获取了锁2，进而线程A还需要获取锁2才能继续执行，但是由于锁2被线程B持有还没有释放，线程A为了等待锁2资源就阻塞了；线程B这时候需要获取锁1才能往下执行，但是由于锁1被线程A持有，导致A也进入阻塞。

线程A和线程B都在等待对方释放锁资源，但是它们又不肯释放原来的锁资源，导致线程A和B一直互相等待，进程死锁了。下面代码示例演示这个问题：

```text
#include <iostream>           // std::cout
#include <thread>             // std::thread
#include <mutex>              // std::mutex, std::unique_lock
#include <condition_variable> // std::condition_variable
#include <vector>

// 锁资源1
std::mutex mtx1;
// 锁资源2
std::mutex mtx2;

// 线程A的函数
void taskA()
{
	// 保证线程A先获取锁1
	std::lock_guard<std::mutex> lockA(mtx1);
	std::cout << "线程A获取锁1" << std::endl;

	// 线程A睡眠2s再获取锁2，保证锁2先被线程B获取，模拟死锁问题的发生
	std::this_thread::sleep_for(std::chrono::seconds(2));

	// 线程A先获取锁2
	std::lock_guard<std::mutex> lockB(mtx2);
	std::cout << "线程A获取锁2" << std::endl;

	std::cout << "线程A释放所有锁资源，结束运行！" << std::endl;
}

// 线程B的函数
void taskB()
{
	// 线程B先睡眠1s保证线程A先获取锁1
	std::this_thread::sleep_for(std::chrono::seconds(1));
	std::lock_guard<std::mutex> lockB(mtx2);
	std::cout << "线程B获取锁2" << std::endl;

	// 线程B尝试获取锁1
	std::lock_guard<std::mutex> lockA(mtx1);
	std::cout << "线程B获取锁1" << std::endl;

	std::cout << "线程B释放所有锁资源，结束运行！" << std::endl;
}
int main()
{
	// 创建生产者和消费者线程
	std::thread t1(taskA);
	std::thread t2(taskB);

	// main主线程等待所有子线程执行完
	t1.join();
	t2.join();

	return 0;
}
```

运行上面的程序，打印如下：

```text
tony@tony-virtual-machine:~/code$ ./a.out 
线程A获取锁1
线程B获取锁2
```

可以看到，线程A获取锁1、线程B获取锁2以后，进程就不往下继续执行了，一直等待在这里，如果这是我们碰到的一个问题场景，我们如何判断出这是由于线程间死锁引起的呢？

先通过ps命令查看一下进程当前的运行状态和PID，如下：

```text
root@tony-virtual-machine:/home/tony# ps -aux | grep a.out
tony 1953 0.0 0.0 98108 1904 pts/0 Sl+ 10:41 0:00 ./a.out
root 2064 0.0 0.0 21536 1076 pts/1 S+ 10:51 0:00 grep --color=auto a.out
```

从上面的命令可以看出，a.out进程的PID是1953，当前状态是Sl+，相当于是多线程程序，全部进入阻塞状态。

通过top命令再查看一下进程内每个线程具体的运行情况，如下：

```text
root@tony-virtual-machine:/home/tony# top -Hp 1953

进程 USER PR NI VIRT RES SHR CPU %MEM TIME+ COMMAND
1953 tony 20 0 98108 1904 1752 S 0.0 0.1 0:00.00 a.out
1954 tony 20 0 98108 1904 1752 S 0.0 0.1 0:00.00 a.out
1955 tony 20 0 98108 1904 1752 S 0.0 0.1 0:00.00 a.out
```

从top命令的打印信息可以看出，所有线程都进入阻塞状态，CPU占用率都为0.0，可以排除是死循环的问题，因为死循环会造成CPU使用率居高不下，而且线程的状态也不会是S。那么接下来有可能是由于I/O网络事件没有发生使线程阻塞，或者是线程发生死锁问题了。

通过gdb远程调试正在运行的程序，打印进程每一个线程的调用堆栈信息，过程如下：

通过gdb attach pid远程调试上面的a.out进程，命令如下：

```text
root@tony-virtual-machine:/home/tony# gdb attach 1953
```

进入gdb调试命令行以后，打印所有线程的调用栈信息，信息如下：

(gdb) thread apply all bt

```text
Thread 3 (Thread 0x7feb523ec700 (LWP 1955)):
#0 _llllock_wait () at …/sysdeps/unix/sysv/linux/x86_64/lowlevellock.S:135
#1 0x00007feb53928023 in __GI___pthread_mutex_lock (mutex=0x5646aabe7140 ) at …/nptl/pthread_mutex_lock.c:78
#2 0x00005646aa9e40bf in __gthread_mutex_lock(pthread_mutex_t*) ()
#3 0x00005646aa9e4630 in std::mutex::lock() ()
#4 0x00005646aa9e46ac in std::lock_guardstd::mutex::lock_guard(std::mutex&) ()
#5 0x00005646aa9e42c0 in taskB() ()
#6 0x00005646aa9e4bdb in void std::__invoke_impl<void, void ()()>(std::__invoke_other, void (&&)()) ()
#7 0x00005646aa9e49e8 in std::__invoke_result<void ()()>::type std::__invoke<void ()()>(void (&&)()) ()
#8 0x00005646aa9e50b6 in decltype (__invoke((_S_declval<0ul>)())) std::thread::_Invoker<std::tuple<void ()()> >::_M_invoke<0ul>(std::_Index_tuple<0ul>) ()
#9 0x00005646aa9e5072 in std::thread::_Invoker<std::tuple<void ()()> >::operator()() ()
#10 0x00005646aa9e5042 in std::thread::_State_impl<std::thread::_Invoker<std::tuple<void ()()> > >::_M_run() ()
#11 0x00007feb5365257f in ?? () from /usr/lib/x86_64-linux-gnu/libstdc++.so.6
#12 0x00007feb539256db in start_thread (arg=0x7feb523ec700) at pthread_create.c:463
#13 0x00007feb530ad88f in clone () at …/sysdeps/unix/sysv/linux/x86_64/clone.S:95

Thread 2 (Thread 0x7feb52bed700 (LWP 1954)):
#0 _llllock_wait () at …/sysdeps/unix/sysv/linux/x86_64/lowlevellock.S:135
#1 0x00007feb53928023 in __GI___pthread_mutex_lock (mutex=0x5646aabe7180 ) at …/nptl/pthread_mutex_lock.c:78
#2 0x00005646aa9e40bf in __gthread_mutex_lock(pthread_mutex_t*) ()
#3 0x00005646aa9e4630 in std::mutex::lock() ()
#4 0x00005646aa9e46ac in std::lock_guardstd::mutex::lock_guard(std::mutex&) ()
#5 0x00005646aa9e4183 in taskA() ()
#6 0x00005646aa9e4bdb in void std::__invoke_impl<void, void ()()>(std::__invoke_other, void (&&)()) ()
#7 0x00005646aa9e49e8 in std::__invoke_result<void ()()>::type std::__invoke<void ()()>(void (&&)()) ()
#8 0x00005646aa9e50b6 in decltype (__invoke((_S_declval<0ul>)())) std::thread::_Invoker<std::tuple<void ()()> >::_M_invoke<0ul>(std::_Index_tuple<0ul>) ()
#9 0x00005646aa9e5072 in std::thread::_Invoker<std::tuple<void ()()> >::operator()() ()
#10 0x00005646aa9e5042 in std::thread::_State_impl<std::thread::_Invoker<std::tuple<void ()()> > >::_M_run() ()
#11 0x00007feb5365257f in ?? () from /usr/lib/x86_64-linux-gnu/libstdc++.so.6
#12 0x00007feb539256db in start_thread (arg=0x7feb52bed700) at pthread_create.c:463
#13 0x00007feb530ad88f in clone () at …/sysdeps/unix/sysv/linux/x86_64/clone.S:95

Thread 1 (Thread 0x7feb53d4b740 (LWP 1953)):
—Type to continue, or q to quit—
#0 0x00007feb53926d2d in __GI___pthread_timedjoin_ex (threadid=140648682280704, thread_return=0x0, abstime=0x0,
block=) at pthread_join_common.c:89
#1 0x00007feb536527d3 in std::thread::join() () from /usr/lib/x86_64-linux-gnu/libstdc++.so.6
#2 0x00005646aa9e43bb in main ()
(gdb)
```

从上面的线程调用栈信息可以看到，当前进程有三个线程，分别是Thread1是main线程，Thread2是taskA线程，Thread3是taskB线程。

从调用栈信息可以看到，Thread3线程进入S阻塞状态的原因是因为它最后在#0 _llllock_wait () at，也就是它在等待获取一把锁(lock_wait)，而且堆栈信息打印的很清晰，#1 0x00007feb53928023 in __GI___pthread_mutex_lock (mutex=0x5646aabe7140 ) at …/nptl/pthread_mutex_lock.c:78，Thread3在获取而获取不到，因此进入阻塞状态了。这里结合代码分析，Thread3线程（也就是taskB）最后在这里阻塞了：

```text
void taskB()
{
// 线程B先睡眠1s保证线程A先获取锁1
std::this_thread::sleep_for(std::chrono::seconds(1));
std::lock_guardstd::mutex lockB(mtx2);
std::cout << “线程B获取锁2” << std::endl;
// 线程B尝试获取锁1
std::lock_guardstd::mutex lockA(mtx1); ===》 这里阻塞了！如果不知道怎么定位到源代码行上，看下一小节！
std::cout << “线程B获取锁1” << std::endl;
std::cout << “线程B释放所有锁资源，结束运行！” << std::endl;
}
```

依然是从调用栈信息可以看到，Thread2线程进入S阻塞状态的原因是因为它最后在#0 _llllock_wait () at，也就是它在等待获取一把锁(lock_wait)，而且堆栈信息打印的很清晰，#1 0x00007feb53928023 in __GI___pthread_mutex_lock (mutex=0x5646aabe7180 ) at …/nptl/pthread_mutex_lock.c:78，Thread2在获取而获取不到，因此进入阻塞状态了。这里结合代码分析，Thread2线程（也就是taskA）最后在这里阻塞了：

void taskA()

{

// 保证线程A先获取锁1

std::lock_guardstd::mutex lockA(mtx1);

std::cout << “线程A获取锁1” << std::endl;

// 线程A睡眠2s再获取锁2，保证锁2先被线程B获取，模拟死锁问题的发生

std::this_thread::sleep_for(std::chrono::seconds(2));

// 线程A先获取锁2

std::lock_guardstd::mutex lockB(mtx2); ===》 这里阻塞了！如果不知道怎么定位到源代码行上，看下一小节！

std::cout << “线程A获取锁2” << std::endl;

std::cout << “线程A释放所有锁资源，结束运行！” << std::endl;

}

既然定位到taskA和taskB线程阻塞的原因，都是因为锁获取不到，然后再结合源码进行分析定位，最终发现taskA之所以获取不到mtx2，是因为mtx2早被taskB线程获取了；同样taskB之所以获取不到mtx1，是因为mtx1早被taskA线程获取了，导致所有线程进入阻塞状态，等待锁资源的获取，但是又因为没有线程释放锁，最终导致死锁问题。（从各线程调用栈信息能看出来，这里面和I/O网络事件没什么关系）

## 4.怎么在源码上定位到问题代码

实际上，上面的代码运行一般是发布后的release版本，内部没有调试信息，我们如果想把死锁的原因定位到源码的某一行代码上，就需要一个debug版本（g++编译添加-g选项），操作如下：

1.编译命令

```text
tony@tony-virtual-machine:~/code$ g++ 20190316.cpp -g -lpthread
```

\2. 运行代码

```text
tony@tony-virtual-machine:~/code$ ./a.out
```

线程A获取锁1

线程B获取锁2

…(程序到这里不往下运行了)

3.gdb调试该进程

```text
root@tony-virtual-machine:/home/tony/code# ps -ef | grep a.out
tony 2617 1535 0 12:32 pts/0 00:00:00 ./a.out
root@tony-virtual-machine:/home/tony/code# gdb attach 2617
```

4.查看当前所有的线程

(gdb) info threads

```text
  Id   Target Id         Frame 
* 1    Thread 0x7f8c63002740 (LWP 2617) "a.out" 0x00007f8c62bddd2d in __GI___pthread_timedjoin_ex (
    threadid=140240914892544, thread_return=0x0, abstime=0x0, block=<optimized out>) at pthread_join_common.c:89
  2    Thread 0x7f8c61ea4700 (LWP 2618) "a.out" __lll_lock_wait () at ../sysdeps/unix/sysv/linux/x86_64/lowlevellock.S:135
  3    Thread 0x7f8c616a3700 (LWP 2619) "a.out" __lll_lock_wait () at ../sysdeps/unix/sysv/linux/x86_64/lowlevellock.S:135
```

可以看到有三个线程。

5.切换到线程2

(gdb) thread 2

6.查看线程2目前的调用栈信息，where或者bt命令都可以

(gdb) where

```text
(gdb) where
#0  __lll_lock_wait () at ../sysdeps/unix/sysv/linux/x86_64/lowlevellock.S:135
#1  0x00007f8c62bdf023 in __GI___pthread_mutex_lock (mutex=0x55678928e180 <mtx2>) at ../nptl/pthread_mutex_lock.c:78
#2  0x000055678908b0bf in __gthread_mutex_lock (__mutex=0x55678928e180 <mtx2>)
    at /usr/include/x86_64-linux-gnu/c++/7/bits/gthr-default.h:748
#3  0x000055678908b630 in std::mutex::lock (this=0x55678928e180 <mtx2>) at /usr/include/c++/7/bits/std_mutex.h:103
#4  0x000055678908b6ac in std::lock_guard<std::mutex>::lock_guard (this=0x7f8c61ea3dc0, __m=...)
    at /usr/include/c++/7/bits/std_mutex.h:162
#5  0x000055678908b183 in taskA () at 20190316.cpp:23
#6  0x000055678908bbdb in std::__invoke_impl<void, void (*)()> (__f=@0x556789d78e78: 0x55678908b0f7 <taskA()>)
    at /usr/include/c++/7/bits/invoke.h:60
#7  0x000055678908b9e8 in std::__invoke<void (*)()> (__fn=@0x556789d78e78: 0x55678908b0f7 <taskA()>)
    at /usr/include/c++/7/bits/invoke.h:95
#8  0x000055678908c0b6 in std::thread::_Invoker<std::tuple<void (*)()> >::_M_invoke<0ul> (this=0x556789d78e78)
    at /usr/include/c++/7/thread:234
#9  0x000055678908c072 in std::thread::_Invoker<std::tuple<void (*)()> >::operator() (this=0x556789d78e78)
    at /usr/include/c++/7/thread:243
#10 0x000055678908c042 in std::thread::_State_impl<std::thread::_Invoker<std::tuple<void (*)()> > >::_M_run (
    this=0x556789d78e70) at /usr/include/c++/7/thread:186
#11 0x00007f8c6290957f in ?? () from /usr/lib/x86_64-linux-gnu/libstdc++.so.6
#12 0x00007f8c62bdc6db in start_thread (arg=0x7f8c61ea4700) at pthread_create.c:463
#13 0x00007f8c6236488f in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:95
```

7.查看上面线程2的第5帧信息#5 0x000055678908b183 in taskA () at 20190316.cpp:23

(gdb) f 5

\#5 0x000055678908b183 in taskA () at 20190316.cpp:23

23 std::lock_guard< std::mutex > lockB(mtx2);

可以看到，这里就直接定位到代码一直阻塞在了20190316.cpp的第23行，对应的行代码是std::lock_guard< std::mutex > lockB(mtx2);

可以同样的步骤定位查看线程3的问题代码行。

## 5.死锁问题代码修改

既然发现了问题，那么就知道这个问题场景发生死锁，是由于多个线程获取多个锁资源的时候，顺序不一致导致的死锁问题，那么保证它们获取锁的顺序是一致的，问题就可以解决，代码修改如下：

```text
#include <iostream>           // std::cout
#include <thread>             // std::thread
#include <mutex>              // std::mutex, std::unique_lock
#include <condition_variable> // std::condition_variable
#include <vector>

// 锁资源1
std::mutex mtx1;
// 锁资源2
std::mutex mtx2;

// 线程A的函数
void taskA()
{
	// 保证线程A先获取锁1
	std::lock_guard<std::mutex> lockA(mtx1);
	std::cout << "线程A获取锁1" << std::endl;

	// 线程A尝试获取锁2
	std::lock_guard<std::mutex> lockB(mtx2);
	std::cout << "线程A获取锁2" << std::endl;

	std::cout << "线程A释放所有锁资源，结束运行！" << std::endl;
}

// 线程B的函数
void taskB()
{
	// 线程B获取锁1
	std::lock_guard<std::mutex> lockA(mtx1);
	std::cout << "线程B获取锁1" << std::endl;

	// 线程B尝试获取锁2
	std::lock_guard<std::mutex> lockB(mtx2);
	std::cout << "线程B获取锁2" << std::endl;

	std::cout << "线程B释放所有锁资源，结束运行！" << std::endl;
}
int main()
{
	// 创建生产者和消费者线程
	std::thread t1(taskA);
	std::thread t2(taskB);

	// main主线程等待所有子线程执行完
	t1.join();
	t2.join();

	return 0;
}
```

程序运行正常，打印如下：

> 线程A获取锁1
> 线程A获取锁2
> 线程A释放所有锁资源，结束运行！
> 线程B获取锁1
> 线程B获取锁2
> 线程B释放所有锁资源，结束运行！

【注意】：不做要书呆了，任何问题都要从实践的角度去考虑问题如何定位分析解决，理论结合实践！

原文地址：https://zhuanlan.zhihu.com/p/370409392

作者：linux