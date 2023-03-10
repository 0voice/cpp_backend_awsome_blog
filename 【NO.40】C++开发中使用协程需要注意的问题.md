# 【NO.40】C++开发中使用协程需要注意的问题

![img](https://pic2.zhimg.com/80/v2-0690aa296b825c1e84e5580ac6dbb0f1_720w.webp)

在异步操作里，如异步连接、异步读写之类的协程，co_await这些协程时需要注意线程切换的细节。

以asio异步连接协程为例：

```text
class client {
public:
    client() {
        thd_ = std::thread([this]{
            io_ctx_.run();
        });
    }

    async_simple::coro::Lazy<bool> async_connect(auto host, auto port) {
        bool ret = co_await util::async_connect(host, port);  #1
        co_return ret;      #2                                 
    }

    ~client() {
        io_ctx_.stop();
        if(thd_.joinable()) {
            thd_.join();
        }
    }

private:
  asio::io_context io_ctx_;
  std::thread thd_;
};

int main() {
    client c;
    async_simple::coro::syncAwait(c.async_connect());
    std::cout<<"quit\n"; #3
}
```

这个例子很简单，client在连接之后就析构了，看起来没什么问题。但是运行之后就会发生线程join的错误，错误的意思是在线程里join自己了。这是怎么回事？co_await一个异步连接的协程，当连接成功后协程返回，这时候发生了线程切换。异步连接返回的时候是在io_context的线程里，代码中的#1在主线程，#2在io_context线程，之后就co_return 返回到main函数的#3，这时候#3仍然在io_context线程里，接着client就会析构了，这时候仍然在io_context线程里，析构的时候会调用thd_.join(); 然后就导致了在io_context的线程里join自己的错误。

这是使用协程时容易犯错的一个地方，解决方法就是避免co_await回来之后去析构client，或者co_await回来仍然回到主线程。这里可以考虑用协程条件变量，在异步连接的时候发起一个新的协程并传入协程条件变量并在连接返回后set_value，主线程去co_await这个条件变量，这样连接返回后就回到主线程了，就可以解决在io线程里join自己的问题了。

![img](https://pic2.zhimg.com/80/v2-0690aa296b825c1e84e5580ac6dbb0f1_720w.webp)

还是以上面的异步连接为例子，需要对之前的async_connect协程增加一个超时功能，代码稍作修改：

```text
class client {
public:
    client() : socket_(io_ctx_) {
        thd_ = std::thread([this]{
            io_ctx_.run();
        });
    }

    async_simple::coro::Lazy<bool> async_connect(auto host, auto port, auto duration) {
        coro_timer timer(io_ctx_);
        timeout(timer, duration).start([](auto&&){}); // #1 启动一个新协程做超时处理
        bool ret = co_await util::async_connect(host, port, socket_);//假设这里co_await返回后回到主线程
        co_return ret;                                      
    }

    ~client() {
        io_ctx_.stop();
        if(thd_.joinable()) {
            thd_.join();
        }
    }


private:
  async_simple::coro::Lazy<void> timeout(auto &timer, auto duration) {
    bool is_timeout = co_await timer.async_wait(duration);
    if(is_timeout) {
        asio::error_code ignored_ec;
        socket_.shutdown(tcp::socket::shutdown_both, ignored_ec);
        socket_.close(ignored_ec);
    }

    co_return;
  }

  asio::io_context io_ctx_;
  tcp::socket socket_;
  std::thread thd_;
  bool is_timeout_;
};

int main() {
    client c;
    async_simple::coro::syncAwait(c.async_connect("localhost", "9000", 5s));
    std::cout<<"quit\n"; #3
}
```

这个代码增加连接超时处理的协程，注意#1那里为什么需要新启动一个协程，而不能用co_await呢？因为co_await是阻塞语义，co_await会导致永远超时，启动一个新的协程不会阻塞当前协程从而可以去调用async_connect。

当timeout超时发生时就关闭socket，这时候async_connect就会返回错误然后返回到调用者，这看起来似乎可以对异步连接做超时处理了，但是这个代码是有问题的。假如异步连接没有超时会发生什么？没有超时的话就返回到main函数了，然后client就析构了，当timeout协程resume回来的时候client其实已经析构了，这时候再去调用成员变量socket_ close将会导致一个访问已经析构对象的错误。

也许有人会说，那就在co_return之前去取消timer不就好了吗？这个办法也不行，因为取消timer，timeout协程并不会立即返回，仍然会存在访问已经析构对象的问题。

正确的做法应该是对两个协程进行同步，timeout协程和async_connect协程需要同步，在async_connect协程返回之前需要确保timeout协程已经完成，这样就可以避免访问已经析构对象的问题了。

这个问题其实也是异步回调安全返回的一个经典问题，协程也同样会遇到这个问题，上面提到的对两个协程进行同步是解决方法之一，另外一个方法就是使用shared_from_this，就像异步安全回调那样处理。



还是以异步连接为例：

```text
async_simple::coro::Lazy<bool> async_connect(const std::string &host, const std::string& port) {
    co_return co_await util::async_connect(host, port);
}

async_simple::coro::Lazy<void> test_connect() {
    bool ok = co_await async_connect("localhost", "8000");
    if(!ok){
        std::cout<<"connect failed\n";
    }

    std::cout<<"connect ok\n";
}

int main() {
    async_simple::coro::syncAwait(test_connect());
}
```

这个代码简单明了，就是测试一下异步连接是否成功，运行也是正常的。如果稍微改一下test_connect：

```text
async_simple::coro::Lazy<void> test_connect() {
    auto lazy = async_connect("localhost", "8000");
    bool ok = co_await lazy;
    if(!ok){
        std::cout<<"connect failed\n";
    }

    std::cout<<"connect ok\n";
}
```

很遗憾，这个代码会导致连接总是失败，似乎很奇怪，后面发现原因是因为async_connect的两个参数失效了，但是写法和刚开始的写法几乎一样，为啥后面这种写法会导致参数失效呢？

原因是co_await一个协程函数时，其实做了两件事：

- 调用协程函数创建协程，这个步骤会创建协程帧，把参数和局部变量拷贝到协程帧里；
- co_await执行协程函数；

回过头来看auto lazy = async_connect("localhost", "8000"); 这个代码调用协程函数创建了协程，这时候拷贝到协程帧里面的是两个临时变量，在这一行结束的时候临时变量就析构了，在下一行去co_await执行这个协程的时候就会出现参数失效的问题了。

co_await async_connect("localhost", "8000"); 这样为什么没问题呢，因为协程创建和协程调用都在一行完成的，临时变量知道协程执行之后才会失效，因此不会有问题。

问题的本质其实是C++临时变量生命周期的问题。使用协程的时候稍微注意一下就好了，可以把const std::string&改成std::string，这样就不会临时变量生命周期的问题了，如果不想改参数类型就co_await 协程函数就好了，不分成两行去执行协程。

原文地址：https://zhuanlan.zhihu.com/p/575882816    

作者：linux