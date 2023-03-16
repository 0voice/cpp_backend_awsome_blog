# 【NO.424】百行代码实现基于C++11的线程池threadpool , 简洁且可带任意多参数

C++11 加入了线程库，从此告别了标准库不支持并发的历史。然而 c++ 对于多线程的支持还是比较低级，稍微高级一点的用法都需要自己去实现，譬如线程池、信号量等。

线程池(thread pool)这个东西，在面试上多次被问到，一般的回答都是：“**管理一个任务队列，一个线程队列，然后每次取一个任务分配给一个线程去做，循环往复。**” 貌似没有问题吧。但是写起程序来的时候就出问题了。

废话不多说，先上实现，然后再啰嗦。(dont talk, show me ur code !)

## 1.代码实现

```text
#pragma once
#ifndef THREAD_POOL_H
#define THREAD_POOL_H

#include <vector>
#include <queue>
#include <atomic>
#include <future>
//#include <condition_variable>
//#include <thread>
//#include <functional>
#include <stdexcept>

namespace std
{
//线程池最大容量,应尽量设小一点
#define THREADPOOL_MAX_NUM 16
//#define THREADPOOL_AUTO_GROW

//线程池,可以提交变参函数或拉姆达表达式的匿名函数执行,可以获取执行返回值
//不直接支持类成员函数, 支持类静态成员函数或全局函数,Opteron()函数等
class threadpool
{
using Task = function<void()>; //定义类型
vector<thread> _pool; //线程池
queue<Task> _tasks; //任务队列
mutex _lock; //同步
condition_variable _task_cv; //条件阻塞
atomic<bool> _run{ true }; //线程池是否执行
atomic<int> _idlThrNum{ 0 }; //空闲线程数量

public:
inline threadpool(unsigned short size = 4) { addThread(size); }
inline ~threadpool()
{
_run=false;
_task_cv.notify_all(); // 唤醒所有线程执行
for (thread& thread : _pool) {
//thread.detach(); // 让线程“自生自灭”
if(thread.joinable())
thread.join(); // 等待任务结束， 前提：线程一定会执行完
}
}

public:
// 提交一个任务
// 调用.get()获取返回值会等待任务执行完,获取返回值
// 有两种方法可以实现调用类成员，
// 一种是使用 bind：.commit(std::bind(&Dog::sayHello, &dog));
// 一种是用 mem_fn：.commit(std::mem_fn(&Dog::sayHello), this)
template<class F, class... Args>
auto commit(F&& f, Args&&... args) ->future<decltype(f(args...))>
{
if (!_run) // stoped ??
throw runtime_error("commit on ThreadPool is stopped.");

using RetType = decltype(f(args...)); // typename std::result_of<F(Args...)>::type, 函数 f 的返回值类型
auto task = make_shared<packaged_task<RetType()>>(
bind(forward<F>(f), forward<Args>(args)...)
); // 把函数入口及参数,打包(绑定)
future<RetType> future = task->get_future();
{ // 添加任务到队列
lock_guard<mutex> lock{ _lock };//对当前块的语句加锁 lock_guard 是 mutex 的 stack 封装类，构造的时候 lock()，析构的时候 unlock()
_tasks.emplace([task](){ // push(Task{...}) 放到队列后面
(*task)();
});
}
#ifdef THREADPOOL_AUTO_GROW
if (_idlThrNum < 1 && _pool.size() < THREADPOOL_MAX_NUM)
addThread(1);
#endif // !THREADPOOL_AUTO_GROW
_task_cv.notify_one(); // 唤醒一个线程执行

return future;
}

//空闲线程数量
int idlCount() { return _idlThrNum; }
//线程数量
int thrCount() { return _pool.size(); }
#ifndef THREADPOOL_AUTO_GROW
private:
#endif // !THREADPOOL_AUTO_GROW
//添加指定数量的线程
void addThread(unsigned short size)
{
for (; _pool.size() < THREADPOOL_MAX_NUM && size > 0; --size)
{ //增加线程数量,但不超过 预定义数量 THREADPOOL_MAX_NUM
_pool.emplace_back( [this]{ //工作线程函数
while (_run)
{
Task task; // 获取一个待执行的 task
{
// unique_lock 相比 lock_guard 的好处是：可以随时 unlock() 和 lock()
unique_lock<mutex> lock{ _lock };
_task_cv.wait(lock, [this]{
return !_run || !_tasks.empty();
}); // wait 直到有 task
if (!_run && _tasks.empty())
return;
task = move(_tasks.front()); // 按先进先出从队列取一个 task
_tasks.pop();
}
_idlThrNum--;
task();//执行任务
_idlThrNum++;
}
});
_idlThrNum++;
}
}
};

}

#endif 
```

代码不多吧,上百行代码就完成了 线程池, 并且, 看看 commit, 哈, 不是固定参数的, 无参数数量限制! 这得益于可变参数模板.

## 2.怎么使用?

```text
#include "threadpool.h"
#include <iostream>

void fun1(int slp)
{
printf(" hello, fun1 ! %d\n" ,std::this_thread::get_id());
if (slp>0) {
printf(" ======= fun1 sleep %d ========= %d\n",slp, std::this_thread::get_id());
std::this_thread::sleep_for(std::chrono::milliseconds(slp));
}
}

struct gfun {
int operator()(int n) {
printf("%d hello, gfun ! %d\n" ,n, std::this_thread::get_id() );
return 42;
}
};

class A {
public:
static int Afun(int n = 0) { //函数必须是 static 的才能直接使用线程池
std::cout << n << " hello, Afun ! " << std::this_thread::get_id() << std::endl;
return n;
}

static std::string Bfun(int n, std::string str, char c) {
std::cout << n << " hello, Bfun ! "<< str.c_str() <<" " << (int)c <<" " << std::this_thread::get_id() << std::endl;
return str;
}
};

int main()
try {
std::threadpool executor{ 50 };
A a;
std::future<void> ff = executor.commit(fun1,0);
std::future<int> fg = executor.commit(gfun{},0);
std::future<int> gg = executor.commit(a.Afun, 9999); //IDE提示错误,但可以编译运行
std::future<std::string> gh = executor.commit(A::Bfun, 9998,"mult args", 123);
std::future<std::string> fh = executor.commit([]()->std::string { std::cout << "hello, fh ! " << std::this_thread::get_id() << std::endl; return "hello,fh ret !"; });

std::cout << " ======= sleep ========= " << std::this_thread::get_id() << std::endl;
std::this_thread::sleep_for(std::chrono::microseconds(900));

for (int i = 0; i < 50; i++) {
executor.commit(fun1,i*100 );
}
std::cout << " ======= commit all ========= " << std::this_thread::get_id()<< " idlsize="<<executor.idlCount() << std::endl;

std::cout << " ======= sleep ========= " << std::this_thread::get_id() << std::endl;
std::this_thread::sleep_for(std::chrono::seconds(3));

ff.get(); //调用.get()获取返回值会等待线程执行完,获取返回值
std::cout << fg.get() << " " << fh.get().c_str()<< " " << std::this_thread::get_id() << std::endl;

std::cout << " ======= sleep ========= " << std::this_thread::get_id() << std::endl;
std::this_thread::sleep_for(std::chrono::seconds(3));

std::cout << " ======= fun1,55 ========= " << std::this_thread::get_id() << std::endl;
executor.commit(fun1,55).get(); //调用.get()获取返回值会等待线程执行完

std::cout << "end... " << std::this_thread::get_id() << std::endl;


std::threadpool pool(4);
std::vector< std::future<int> > results;

for (int i = 0; i < 8; ++i) {
results.emplace_back(
pool.commit([i] {
std::cout << "hello " << i << std::endl;
std::this_thread::sleep_for(std::chrono::seconds(1));
std::cout << "world " << i << std::endl;
return i*i;
})
);
}
std::cout << " ======= commit all2 ========= " << std::this_thread::get_id() << std::endl;

for (auto && result : results)
std::cout << result.get() << ' ';
std::cout << std::endl;
return 0;
}
catch (std::exception& e) {
std::cout << "some unhappy happened... " << std::this_thread::get_id() << e.what() << std::endl;
}
```

为了避嫌，先进行一下版权说明：代码是 me “写”的，但是思路来自 Internet， 特别是这个线程池实现。

## 3.实现原理

接着前面的废话说。“管理一个任务队列，一个线程队列，然后每次取一个任务分配给一个线程去做，循环往复。” 这个思路有神马问题？

线程池一般要复用线程，所以如果是取一个 task 分配给某一个 thread，执行完之后再重新分配，在语言层面基本都是不支持的：**一般语言的 thread 都是执行一个固定的 task 函数，执行完毕线程也就结束了(至少 c++ 是这样)**。

so 要如何实现 task 和 thread 的分配呢？让每一个 thread 都去执行调度函数：循环获取一个 task，然后执行之。idea 是不是很赞！保证了 thread 函数的唯一性，而且复用线程执行 task 。

即使理解了 idea，代码还是需要详细解释一下的。

- 一个线程 pool，一个任务队列 queue ，应该没有意见；
- **任务队列是典型的生产者-消费者模型**，本模型至少需要两个工具：一个 mutex + 一个条件变量，或是一个 mutex + 一个信号量。mutex 实际上就是锁，保证任务的添加和移除(获取)的互斥性，一个条件变量是保证获取 task 的同步性：一个 empty 的队列，线程应该等待(阻塞)；
- atomic<bool> 本身是原子类型，从名字上就懂：**它们的操作 load()/store() 是原子操作，所以不需要再加 mutex。**

## 4.c++语言细节

即使懂原理也不代表能写出程序，上面用了众多c++11的“奇技淫巧”，下面简单描述之。

- using Task = function<void()> 是类型别名，简化了 typedef 的用法。function<void()> 可以认为是一个函数类型，接受任意原型是 void() 的函数，或是函数对象，或是匿名函数。void() 意思是不带参数，没有返回值。
- pool.emplace_back([this]{...}) 和 pool.push_back([this]{...}) 功能一样，只不过前者性能会更好；
- pool.emplace_back([this]{...}) 是构造了一个线程对象，执行函数是拉姆达匿名函数 ；
- 所有对象的初始化方式均采用了 {}，而不再使用 () 方式，因为风格不够一致且容易出错；
- 匿名函数：[this]{...} 不多说。[] 是捕捉器，this 是引用域外的变量 this指针， 内部使用死循环, 由cv_task.wait(lock,[this]{...}) 来阻塞线程；
- delctype(expr) 用来推断 expr 的类型，和 auto 是类似的，相当于类型占位符，占据一个类型的位置；auto f(A a, B b) -> decltype(a+b) 是一种用法，不能写作 decltype(a+b) f(A a, B b)，为啥？！c++ 就是这么规定的！
- commit 方法是不是略奇葩！可以带任意多的参数，第一个参数是 f，后面依次是函数 f 的参数！(注意:参数要传struct/class的话,建议用pointer,小心变量的作用域) 可变参数模板是 c++11 的一大亮点，够亮！至于为什么是 Arg... 和 arg... ，因为规定就是这么用的！
- commit 直接使用只能调用stdcall函数，但有两种方法可以实现调用类成员，一种是使用 bind：.commit(std::bind(&Dog::sayHello, &dog))；一种是用 mem_fn：.commit(std::mem_fn(&Dog::sayHello), &dog)；
- **make_shared 用来构造 shared_ptr 智能指针。**用法大体是 shared_ptr<int> p = make_shared<int>(4) 然后 *p == 4 。智能指针的好处就是， 自动 delete ！
- bind 函数，接受函数 f 和部分参数，返回currying后的匿名函数，譬如 bind(add, 4) 可以实现类似 add4 的函数！
- forward() 函数，类似于 move() 函数，后者是将参数右值化，前者是... 肿么说呢？大概意思就是：不改变最初传入的类型的引用类型(左值还是左值，右值还是右值)；
- packaged_task 就是任务函数的封装类，通过 get_future 获取 future ， 然后通过 future 可以获取函数的返回值(future.get())；packaged_task 本身可以像函数一样调用 () ；
- queue 是队列类， front() 获取头部元素， pop() 移除头部元素；back() 获取尾部元素，push() 尾部添加元素；
- **lock_guard 是 mutex 的 stack 封装类**，构造的时候 lock()，析构的时候 unlock()，是 c++ RAII 的 idea；
- condition_variable cv; 条件变量， 需要配合 unique_lock 使用；unique_lock 相比 lock_guard 的好处是：可以随时 unlock() 和 lock()。cv.wait() 之前需要持有 mutex，wait 本身会 unlock() mutex，如果条件满足则会重新持有 mutex。
- 最后**线程池析构的时候,join() 可以等待任务都执行完在结束**,很安全!

原文地址：https://zhuanlan.zhihu.com/p/472021998

作者：linux