# 【NO.250】C++一行代码实现任意系统函数 Hook

作者：joliphzhu，腾讯 IEG 客户端开发工程师

> 一句话实现系统 API 的 Hook，参数记录以及数据过滤与修改，关注敏感数据本身而不是哪个 API 的哪个参数可能有敏感的需要处理的信息，写工具的时候想到上述能力可以借助模板实现，赶紧尝试了下，顺便做个笔记分享。

## 0.**AnyCall**

## 1.**背景**

一般来说所有 ApiHook 库都会需要提供一个与被 HookApi 相似/相同的 Myxxx 函数以实现参数访问，这里以 BlackBone 的 LocalHook 举例，其需要的是被 HookApi 的引用参数形式，如下所示

```text
bool TestFunc1(char a, int b)
{
    return a + b;
}

bool MyTestFunc1(char& a, int& b)
{
    return a + b;
}

blackbone::Detour<decltype(&TestFunc1)> hook;
hook.Hook(&TestFunc1, &MyTestFunc1, blackbone::HookType::Inline);
```

上述使用方式需要为每个被挂钩的函数都写一个符合参数要求的 Myxxx 函数并将其所有的参数加上引用符号，多写些 API 就产生了大量重复性的代码

## 2.**通用化处理逻辑的优势**

既然在这里已经知道被钩挂的函数类型，那么是否可以利用 C++模板为我们自动生成一个通用函数，以实现一行代码完成任意 API 的 Hook 呢？进一步来说，这样的处理方式是否可以分离 API 和参数的对应关系，使我们不再关注需要修改哪个 API 的哪个参数的内容，而是只关注什么数据是敏感数据，对所有参数只要出现敏感数据的参数就进行修改呢，下面是尝试实现上述逻辑的代码笔记

## 3.**类型萃取生成函数**

函数的参数类型萃取需要借助 struct 辅助实现，先看下如果不使用 struct 辅助直接定义模板函数的困难在哪，代码如下

```text
template<typename RET, typename... ARGS>
RET FunctionCreater(ARGS&... args)
{
 //do something...
}

blackbone::Detour<decltype(&TestFunc1)> hook;
hook.Hook(&TestFunc1, &FunctionCreater<bool,char,int>, blackbone::HookType::Inline);
```

这里的模板参数需要用 testFuncHooker<bool,char,int>形式传递，即先是返回值类型再是各个参数类型，如果需要进一步自动化处理的话则需要实现自动提取参数类型并将其逐个依次在此展开的能力，使用 struct 可以避免实现上述复杂的逻辑，代码如下

```text
template<typename RET, typename... ARGS>
struct AnyCall;

template<typename RET, typename... ARGS>
struct AnyCall<RET(ARGS...)>
{
    static RET FunctionCreater(ARGS&... args)
    {
  //do something...
    }
};
```

这里利用变参模板 + 类型萃取，struct 先申明返回值和可变参数包类型的名称，并在特化匹配阶段将 decltype(&TestFunc1) 整体拆分出其中的返回值类型和各个参数类型,再通过叠加使用宏定义即可在代码层面实现一行钩挂指定 API 的能力，如下

```text
template<typename RET, typename... ARGS>
struct AnyCall;

template<typename RET, typename... ARGS>
struct AnyCall<RET(ARGS...)>
{
    static RET FunctionCreater(ARGS&... args)
    {
  //do something...
    }
};

#define HookApi(FuncName)   static blackbone::Detour<decltype(&FuncName)> hook##FuncName; \
                            hook##FuncName.Hook(&FuncName, &AnyCall<decltype(FuncName)>::FunctionCreater, blackbone::HookType::Inline);

HookApi(ReadFile);
HookApi(WriteFile);
HookApi(CreateFile);
......
```

## 4.**任意函数调用参数监控**

### 4.1 **函数名称获取**

Hook 的一大目标就是需要辅助分析关键 API 调用信息，用上述 AnyCall 可以很好地解决参数打印需求，但首先需要解决的就是函数名获取的问题，不然日志会很难读，Anycall 的模板参数中只传递了函数的类型，是感知不到函数名的，因此函数名的信息只有在宏定义的阶段才能访问到，好在从 c++ 17 起静态局部字符串变量可以作为模板参数传递，这使得我们可以较为轻松的把他纳入我们的宏定义中去实现，如下

```text
#define HookApi(FuncName) static const wchar_t name##FuncName[] = L#FuncName; \
       static blackbone::Detour<decltype(&FuncName)> hook##FuncName; \
       hook##FuncName.Hook(&FuncName, &AnyCall<name##FuncName,decltype(FuncName)>::FunctionCreater, blackbone::HookType::Inline);

HookApi(ReadFile);
//宏展开后代码如下
static const wchar_t nameReadFile[] = L"ReadFile";
static blackbone::Detour<decltype(&ReadFile)> hookReadFile;
hookReadFile.Hook(&ReadFile, &AnyCall<nameReadFile, decltype(ReadFile)>::FunctionCreater, blackbone::HookType::Inline);
```

### 4.2 **展开可变参数包打印**

对变参模板使用递归的方式进行展开 + 任意日志库即可实现参数信息的打印，这里以打印到控制台为例

```text
template<typename RET, typename... ARGS>
struct AnyCall;

template<typename RET, typename... ARGS>
struct AnyCall<RET(ARGS...)>
{
    static RET FunctionCreater(ARGS&... args)
    {
  std::initializer_list<int> expandLog{ (std::wcout << args << "|", 0)... };
    }
};
```

LogArgs 使用初始化列表 + 逗号表达式的方式逐个展开可变参数包，并在每个参数间添加"|"符号分割，但这么写会有些问题，比如遇到为空的字符串指针会崩溃以及遇到特殊的不能被 wstringstream 处理的类型就会报错，前者为运行时的问题可以通过运行时判断处理，后者作为类型问题可以通过模板参数匹配解决

### 4.3 **适配特殊参数的处理逻辑**

由于需要额外的判断逻辑，因此在初始化列表内不能再使用 operator<<默认处理了，需要调用自定义的包装函数，修改如下

```text
template<typename ArgType>
void LogArgs(std::wstringstream& logInfo, ArgType&& arg)
{
    logInfo << typeid(ArgType).name() << "|";
}

template<typename RET, typename... ARGS>
struct AnyCall;

template<typename RET, typename... ARGS>
struct AnyCall<RET(ARGS...)>
{
    static RET FunctionCreater(ARGS&... args)
    {
  std::wstringstream logInfo;
  std::initializer_list<int> expandLog{ (LogArgs(logInfo,args),0)... };
  std::wcout << logInfo.str() << std::endl;
    }
};
```

后续解决上面提到的两个问题：

1. 首先是空字符串指针的崩溃问题：对指针类型且值为 nullptr 的情况特殊处理
2. 其次是没被 wstringstream 的 operator<<重载的参数类型的打印问题：使用 requires 定义一个 concept 让编译器帮助判断参数是否可被打印，然后特化处理可以被打印的部分逻辑，在不能处理的类型部分将其类型名称打印出来

```text
template<typename T>    concept CANLOG_TYPE = requires(std::wstringstream & logInfo, T x) { logInfo << x; };

template<CANLOG_TYPE ArgType>
void LogArgs(std::wstringstream& logInfo, ArgType&& arg)
{
 if (std::is_pointer_v<std::decay_t<ArgType>> && !arg)
 {
  logInfo << "nullptr|";
  return;
 }

 logInfo << arg << "|";
}

template<typename ArgType>
void LogArgs(std::wstringstream& logInfo, ArgType&& arg)
{
    logInfo << typeid(ArgType).name() << "|";
}
```

### 4.4 **任意函数调用参数过滤**

Hook 的第二大目的一般是需要对指定数据进行过滤/欺骗，数据获取可以用上述方案通用化解决，但是参数的过滤方面用 AnyCall 会有一些挑战，尤其是如果希望做到完全通用化的敏感数据过滤的目标的话，后面会提，先看下如何进行相关逻辑处理，类似参数日志打印的处理方式，将参数逐个展开传递给 ArgHandler，在 ArgHandler 内即可实现基于参数类型的数据过滤策略，AnyCall 实现如下

```text
template<const wchar_t* funcName, typename RET, typename... ARGS>
struct AnyCall;

template<const wchar_t* funcName, typename RET, typename... ARGS>
struct AnyCall<funcName, RET(ARGS...)>
{
    static RET FunctionCreater(ARGS&... args)
    {
  std::wstringstream logInfo;
  std::initializer_list<int> expandLog{ (LogArgs(logInfo,args),0)... };
  LOG() << funcName << L": " << logInfo.str();

  std::initializer_list<int> expandScan{ (ArgHandler(args),0)... };

  if constexpr (!std::is_same_v<RET, void>)
  {
   return RET{};
  }
 }
```

ArgHandle 需要对不同类型的参数做处理

### 4.5 **特殊修饰符的参数类型**

首先是对具有 const 修饰符的参数不做处理(系统函数具有 const 修饰符的参数一般也不会有需要被修改的内容)

```text
template<typename ARG>
void ArgHandler(const ARG& x) {}
```

### 4.6 **指针参数类型**

然后即是对指针类型的参数的处理，结构体指针类型的参数需要解引用才能获取到其结构体的大小

```text
template<typename T>    concept POINTER_TYPE = std::is_pointer_v<T> && !std::is_same_v<T, void*>;

template<POINTER_TYPE ARG>
void ArgHandler(ARG x)
{
 if (!x) return;
 //...
 if constexpr (!std::is_pointer_v<std::remove_pointer_t<ARG>>)
 {
  ArgHandler((byte*)x, sizeof(std::remove_pointer_t<ARG>));
 }
 else
 {
  ArgHandler(*x);
 }
}
```

这里的处理方式是将指针移除拿到其结构体的大小，拥有地址和大小后对其数据进行处理(两个参数的 ArgHandler 函数在此省略实现)，多级指针使用递归的方式解决，此处递归过程在编译后可以全部优化掉，另外在//...处省略了对字符串指针类型的处理过程

### 4.7 **链表形结构体的处理**

上述的参数通用处理逻辑在处理非内存连续性结构体时会出现遗漏，比如链表形结构体这样内部有类似 next 指针变量就会导致只能扫描到头结点，这种结构体内部的特殊字段导致结构体的实际范围扩展的情况，由于很难有更通用的处理方式，只能使用特化解决，但可以使用 if constexpr 替代特化简化相关代码，让用不到此逻辑的函数优化掉该分支

```text
template<typename T>    concept POINTER_TYPE = std::is_pointer_v<T> && !std::is_same_v<T, void*>;

template<POINTER_TYPE ARG>
void ArgHandler(ARG x)
{
 if (!x) return;

 if constexpr (std::is_same_v<ARG, PNodeList>)
 {
        while (x)
        {
   //dosometing
            x = x->Next;
        }
        return;
 }

 if constexpr (!std::is_pointer_v<std::remove_pointer_t<ARG>>)
 {
  ArgHandler((BYTE*)x, sizeof(std::remove_pointer_t<ARG>));
 }
 else
 {
  ArgHandler(*x);
 }
}
```

### 4.8 **最难以处理的 void\*类型**

上述指针逻辑在概念阶段就排除掉了 void*类型的指针，因为此类指针通常的使用方式是强制类型转换成其他的结构体指针类型，这里的转换逻辑 API 文档定义的，编译期不可能推导出来，以一个驱动通信函数 DeviceIoControl 为例，其实现形式如下：

```text
BOOL DeviceIoControl(
  [in]                HANDLE       hDevice,
  [in]                DWORD        dwIoControlCode,
  [in, optional]      LPVOID       lpInBuffer,
  [in]                DWORD        nInBufferSize,
  [out, optional]     LPVOID       lpOutBuffer,
  [in]                DWORD        nOutBufferSize,
  [out, optional]     LPDWORD      lpBytesReturned,
  [in, out, optional] LPOVERLAPPED lpOverlapped
);
```

其中的 lpOutBuffer 是我们关注的内容，但他是 LPVOID 类型，实际在使用的时候，外部会强制转换成当前需要的结构体指针进行访问，这里外部对 lpOutBuffer 的大小的感知是通过随 lpOutBuffer 一同返回的 nOutBufferSize 确定的。但问题就在这里，一是 ArgHandler 参数扫描每次只能接受一个参数，二是对于编译器来说 AnyCall 的内部是无法感知这里参数间人为定义的关系，所以这种问题也只能通过特化去解决，那么可以使用字符串编译期比较解决特化问题吗？

```text
template<const wchar_t* funcName, typename RET, typename... ARGS>
struct AnyCall;

template<const wchar_t* funcName, typename RET, typename... ARGS>
struct AnyCall<funcName, RET(ARGS...)>
{
    static RET FunctionCreater(ARGS&... args)
    {
  if constexpr(wcscmp_compiletime(funName,L"DeviceIoControl"))
  {
   //指定第4与第5号参数的关联性
   //......
  }

  std::initializer_list<int> expandScan{ (ArgHandler(args),0)... };

  if constexpr (!std::is_same_v<RET, void>)
  {
   return RET{};
  }
    }
};
```

这里即使 wcscmp_compiletime 函数可以实现编译期的字符串比较也不能实现编译期的结果计算，测试是这样原因，应该是编译器还是将 funcName 当做一个外部符号有关？这里暂不清楚原因，引入 std::integer_sequence 拆分字符串的话可能能解决这里的问题，当然还有次一级的解决方案，即类型比较的特化，代码如下

```text
template<const wchar_t* funcName, typename RET, typename... ARGS>
struct AnyCall;

template<const wchar_t* funcName, typename RET, typename... ARGS>
struct AnyCall<funcName, RET(ARGS...)>
{
    static RET FunctionCreater(ARGS&... args)
    {
        if constexpr (std::is_same_v<RET(ARGS...), decltype(DeviceIoControl)>)
        {
            //指定第4与第5号参数的关联性
   //......
        }

        std::initializer_list<int> expandScan{ (ArgHandler(args),0)... };

        if constexpr (!std::is_same_v<RET, void>)
        {
            return RET{};
        }
    }
};
```

相对字符比较来说可能会存在不同的函数同一个类型的问题，但是同类型大概率关联性指定也是一样的，这里参数关联使用 tuple 去访问即可

```text
template<unsigned INDEX1, unsigned INDEX2, typename... ARGS>
void RelationArgHandler(ARGS&... args)
{
    std::tuple<ARGS...> argsTuple(args...);
    auto buffer = std::get<INDEX1>(argsTuple);
    auto size = std::get<INDEX2>(argsTuple);
    //do someting...
}

template<const wchar_t* funcName, typename RET, typename... ARGS>
struct AnyCall;

template<const wchar_t* funcName, typename RET, typename... ARGS>
struct AnyCall<funcName, RET(ARGS...)>
{
    static RET FunctionCreater(ARGS&... args)
    {
        if constexpr (std::is_same_v<RET(ARGS...), decltype(DeviceIoControl)>)
        {
            //指定第4与第5号参数的关联性
   RelationArgHandler<4, 5, ARGS...>(args...);
        }

        std::initializer_list<int> expandScan{ (ArgHandler(args),0)... };

        if constexpr (!std::is_same_v<RET, void>)
        {
            return RET{};
        }
    }
};
```

### 4.9 **从汇编角度看生成的一个 API 案例**

简化后的测试代码如下：

```text
bool TestFunc1(int a, int* b,int** c)
{
    return a + b;
}

template<typename T>    concept POINTER_TYPE = std::is_pointer_v<T>;

template<typename ARG>
__forceinline void ArgHandler(ARG x)
{
    printf("%s|", typeid(ARG).name());
}

template<POINTER_TYPE ARG>
__forceinline void ArgHandler(ARG x)
{
    ArgHandler(*x);
}

template<typename RET, typename... ARGS>
struct AnyCall;

template<typename RET, typename... ARGS>
struct AnyCall<RET(ARGS...)>
{
    static RET FunctionCreater(ARGS&... args)
    {
        std::initializer_list<int> expandLog{ (std::wcout << args << "|", 0)... };
        std::initializer_list<int> expandScan{ (ArgHandler(args),0)... };
        if constexpr (!std::is_void_v<RET>) return RET{};
    }
};

auto func = AnyCall<decltype(TestFunc1)>::FunctionCreater;

int a = 1;
int* b = nullptr;
int** c = nullptr;
func(a, b, c);
```

逻辑为先打印参数值再打印参数类型，测试输出为: 1|0000000000000000|0000000000000000|int|int|int|，符合预期(记得开启优化) 在 Compiler Explorer ([http://godbolt.org](https://link.zhihu.com/?target=http%3A//godbolt.org))上查看对应的汇编代码，可以看到生成的逻辑很简单就是依次将参数输出以及依次将参数调用 ArgHandler 函数。

![img](https://pic1.zhimg.com/80/v2-4b7ae682bd650dc963813c9c2773613c_720w.webp)

## 5.**总结**

1. 一句话实现系统 API 的 HOOK
2. 参数记录以及数据过滤与修改
3. 关注敏感数据本身而不是哪个 API 的哪个参数可能有敏感的需要处理的信息
4. 完全通用化的参数处理逻辑

起初的 3 点设想能够比较好的实现，但是在第 4 点完全通用化的处理逻辑层面尚且存在些难题，尤其是特殊参数的处理上还是得少量的使用特化逻辑去解决。

原文作者：[腾讯技术工程](https://www.zhihu.com/org/teng-xun-ji-zhu-gong-cheng)

原文链接：https://zhuanlan.zhihu.com/p/545872317