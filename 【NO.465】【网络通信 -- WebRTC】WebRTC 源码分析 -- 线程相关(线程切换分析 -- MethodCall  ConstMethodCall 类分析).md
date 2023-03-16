# 【NO.465】【网络通信 -- WebRTC】WebRTC 源码分析 -- 线程相关(线程切换分析 -- MethodCall / ConstMethodCall 类分析)

## **1.MethodCall 类分析**

```text
// 类模板
// C    : 包含方法的类类型
// R    : 方法的返回值类型
// Args : 方法的参数类型
template <typename C, typename R, typename... Args>
// QueuedTask : 是一个接口类型，提供了 Run() 接口函数
class MethodCall : public QueuedTask {
 public:
  // 声明函数对象类型
  typedef R (C::*Method)(Args...);
  // MethodCall 对象构造函数
  MethodCall(C* c, Method m, Args&&... args)
      : c_(c),
        m_(m),
        args_(std::forward_as_tuple(std::forward<Args>(args)...)) {}
 
  // 方法运行管理
  R Marshal(const rtc::Location& posted_from, rtc::Thread* t) {
    // 判断给定线程是否是当前线程
    if (t->IsCurrent()) {
      // 执行给定的方法
      Invoke(std::index_sequence_for<Args...>());
    } else {
      // 将任务加入任务队列
      t->PostTask(std::unique_ptr<QueuedTask>(this));
      // 等待异步任务处理完毕
      event_.Wait(rtc::Event::kForever);
    }
    // 返回处理结果
    return r_.moved_result();
  }
 
 private:
  // 任务的 Run 方法覆写
  bool Run() override {
    // 在任务中执行给定的方法
    Invoke(std::index_sequence_for<Args...>());
    // 触发 event 激活等待的线程
    event_.Set();
    return false;
  }
 
  // 模板函数，执行给定类型 C 的方法 M
  // 实际调用 ReturnType 模板类中的 Invoke 方法
  template <size_t... Is>
  void Invoke(std::index_sequence<Is...>) {
    r_.Invoke(c_, m_, std::move(std::get<Is>(args_))...);
  }
 
  // 类实例
  C* c_;
  // 类中的方法
  Method m_;
  // 返回值类型
  ReturnType<R> r_;
  // 方法的参数
  std::tuple<Args&&...> args_;
  // 事件实例，用于等待异步处理结果
  rtc::Event event_;
};
```

## **2.ConstMethodCall 类分析**

```text
// 类模板
// C    : 包含方法的类类型
// R    : 方法的返回值类型
// Args : 方法的参数类型
template <typename C, typename R, typename... Args>
// QueuedTask : 是一个接口类型，提供了 Run() 接口函数
// 执行类 C 中的 const 方法，不会改变类 C
class ConstMethodCall : public QueuedTask {
 public:
  // 声明函数对象类型，const 方法
  typedef R (C::*Method)(Args...) const;
  // ConstMethodCall 对象构造函数
  ConstMethodCall(const C* c, Method m, Args&&... args)
      : c_(c),
        m_(m),
        args_(std::forward_as_tuple(std::forward<Args>(args)...)) {}
 
  // 方法运行管理
  R Marshal(const rtc::Location& posted_from, rtc::Thread* t) {
    // 判断给定线程是否是当前线程
    if (t->IsCurrent()) {
      // 执行给定的方法
      Invoke(std::index_sequence_for<Args...>());
    } else {
      // 将任务加入任务队列
      t->PostTask(std::unique_ptr<QueuedTask>(this));
      // 等待异步任务处理完毕
      event_.Wait(rtc::Event::kForever);
    }
    // 返回处理结果
    return r_.moved_result();
  }
 
 private:
 // 任务的 Run 方法覆写
  bool Run() override {
    // 在任务中执行给定的方法
    Invoke(std::index_sequence_for<Args...>());
    // 触发 event 激活等待的线程
    event_.Set();
    return false;
  }
 
  // 模板函数，执行给定类型 C 的方法 M
  // 实际调用 ReturnType 模板类中的 Invoke 方法
  template <size_t... Is>
  void Invoke(std::index_sequence<Is...>) {
    r_.Invoke(c_, m_, std::move(std::get<Is>(args_))...);
  }
 
  // 类实例，该类是常量
  const C* c_;
  // 类中的方法
  Method m_;
  // 返回值类型
  ReturnType<R> r_;
  // 方法的参数
  std::tuple<Args&&...> args_;
  // 事件实例，用于等待异步处理结果
  rtc::Event event_;
};
```

## **3.ReturnType 类分析**

```text
// 类模板 -- 包装方法执行的返回值
template <typename R>
class ReturnType {
 public:
  // 模板函数，执行给定类型的对应的方法
  template <typename C, typename M, typename... Args>
  void Invoke(C* c, M m, Args&&... args) {
    // 实际执行类型 C 中的方法 M
    r_ = (c->*m)(std::forward<Args>(args)...);
  }
 
  // 获取方法运行的返回值
  R moved_result() { return std::move(r_); }
 
 private:
  R r_;
};
 
// 类模板 -- 包装方法执行的返回值
// 特化模板，返回值为 void 的情况
template <>
class ReturnType<void> {
 public:
  // 模板函数，执行给定类型的对应的方法
  template <typename C, typename M, typename... Args>
  void Invoke(C* c, M m, Args&&... args) {
    // 实际执行类型 C 中的方法 M
    (c->*m)(std::forward<Args>(args)...);
  }
 
  void moved_result() {}
};
```

**附录**

**附录一、方法异步处理的实现**

- 说明，方法的异步处理涉及的知识点总结如下

```text
1. 线程发布任务
webrtc/src/rtc_base/thread.cc
Thread::PostTask(std::unique_ptr<webrtc::QueuedTask> task)
 
2. 事件的等待与激活
webrtc/src/rtc_base/event.h
webrtc/src/rtc_base/event.cc
```

原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/450505246