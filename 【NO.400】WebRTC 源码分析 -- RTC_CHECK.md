# 【NO.400】WebRTC 源码分析 -- RTC_CHECK

**【1】RTC_DCHECK(1 != 1) << "hello world " << 100 << 3.14；的执行过程**

```text
RTC_DCHECK(1 != 1) << "hello world " << 100 << 3.14；的执行过程
宏展开结果
while (!(1 != 1)) FatalLogCall<false>("main.cc", 7, "1 != 1") 
    & LogStreamer<>() << "hello world " << 100 << 3.14；
注，根据运算符的优先级，<< 运算符比 & 运算符优先级高，所以先算 << 运算符
```

**<< 运算符的运算过程**

- \1. 先计算 LogStreamer<>() << "hello world "，LogStreamer<>() 生成临时对象，临时对象会调用 operator<<() 函数，operator<<() 函数会把 "hello world" 和临时对象作为参数，生成 LogStreamer<T, Ts…> 对象，该对象存储着 "hello world" 和临时对象
- \2. 上一步生成的 LogStreamer<T, Ts…> 对象会继续调用 operator<<() 函数，同时把自己和 100 传入，生成一个新的 LogStreamer<T, Ts…> 对象，从而递归下去，直到所有的 << 运算符处理完毕

![img](https://pic3.zhimg.com/80/v2-4ee47dc77f2d995e55ae78a1c0bb7472_720w.webp)

**& 运算符的运算过程**

- \1. 上一步最后会返回 LogStreamer<T, Ts…> 对象，组成了新的表达式，FatalLogCall<false>(“main.cc”, 7, “1 != 1”) & LogStreamer<T, Ts…>
- FatalLogCall<false>(“main.cc”, 7, “1 != 1”) 会生成临时对象，临时对象会继续调用 operator&() 函数
- \2. 在 operator&() 函数中会使用 LogStreamer<T, Ts…> 对象调用 Call() 函数，会产生递归调用，每次调用都会把本类保存的日志数据往下层传递
- \3. 递归到最后，LogStreamer<>() 生成临时对象会调用 FatalLog() 函数，将所有日志数据打印出来

![img](https://pic1.zhimg.com/80/v2-ea2a1738bb7d9a287be5478d006e5258_720w.webp)

原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/485451406