# 【NO.389】最全的MSVC编译参数，收藏备用，MinGW与MSVC编译的区别

## 1.**msvc的命令行编译链接命令**

### 1.1 **cl命令格式**


CL [option…] file… [option | file]… [lib…] [@command-file] [/link link-opt…]

选项→用途
option→参数可以使用/或者-，具体含义可以使用/HELP option看到解释。
file→一个或者多个源文件，.obj文件或者。lib文件，CL编译源文件传递.obj和.lib给linker
lib→一个或多个库文件，cl将传送给linker
command-file→一个保存多个选项的文件
link-opt→一个或多个链接操作，cl将传递给linkercl用到的环境变量

变量INCLUDE→指定vc的头文件位置，windows sdk的头文件位置。等等，中间用；分割。
LIB→指定vc的库，windows sdk的库路径。中间用;分割。

##  2.**优化参数**


选项→用途
/O1→目标尺寸最小
/O2→目标速度最快
/Ob→控制inline扩展，/Ob{0/1/2}
/Od→关闭优化
/Og→使用全局优化
/Oi→产生固定函数
/Os→偏向尺寸优化
/Ot→偏向速度优化
/Ox→最佳优化
/Oy→忽略帧指针（仅x86)
/favor→对特定架构优化。/favor:{blend/ATOM/AMD64/INTEL64}

##  3.**产生代码**


选项→用途
/arch→产生代码时使用SSE或者SSE2指令（仅x86）
/clr→产生运行在the common language runtime上的输出文件
/EH→指定异常处理模型/EH{s/a}[c][r][-]
/fp→指定浮点指针行为/fp:[precise/except[-]/fast/strict]
/GA→对windows应用进行优化
/Gd→使用_cdecl调用转换(仅x86)
/Ge→激活堆栈探测
/GF→打开字符串池
/Gh→调用钩子函数_penter
/GH→调用钩子函数_pexit
/GL→打开整个程序优化
/Gm→打开最小重建造
/GR→打开运行时类型信息(RTII)
/Gr→使用_fastcall调用转换（仅x86)
/GS→检查缓冲区安全
/Gs→控制堆栈探测
/GT→使用静态线程本地存储时保证数据分配安全
/guard:cf→加入控制流安全检查
/Gv→使用_vectorcall调用转换(仅x86)
/Gw→打开整个程序的全局数据优化
/GX→打开同步异常处理
/Gy→打开函数级链接
/GZ→打开快速检查，等同于/RTC1
/Gz→使用_stdcall调用转换(仅x86)
/homeparams→强制参数通过寄存器传递，仅x64编译
/hotpatch→创建一个补丁镜像
/Qfast_transcendentals→Generates fast transcendentals.
/QIfist→禁止从浮点转换为整数是调用函数_ftol(仅x86)
/Qimprecise_fwaits→在try块内部移除fwait命令
/Qpar→打开自动并行循环
/Gpar-report→打开一个并行循环的报告级
/Gsafe_fp_loads→对浮点值使用整数移动指令，禁止某些浮点指令的装入优化
/RTC)→打开运行时错误检查
/volatile)→解释执行选择怎样的volatile关键字

##  4.**输出文件**


选项→用途
/doc→处理注释文档到一个XML文件
/FA→配置汇编列表文件
/Fa→创建汇编列表文件
/Fd→删除程序数据库文件
/Fe→重命名执行文件
/Fi→指定预处理输出文件名
/Fm→创建mapfile
/Fo→创建object文件
/Fp→指定预编译头文件名
/FR /Fr→参数浏览文件


**预处理**


选项→用途
/AI→Specifies a directory to search to resolve file references passed to the #using directive.
/C→Preserves comments during preprocessing.
/D→定预处理宏
/E→复制预处理到标准输出
/EP→复制预处理到标准输出
/FI→预处理指定的include文件
/FU→Forces the use of a file name, as if it had been passed to the #using directive.
/Fx→合并注入代码和源代码
/I→指定include文件搜索路径
/P→写预处理到一个输出文件
/U→删除预定义宏
/u→和/U相同
/X→忽略标准include路径

##  5.**语言**


选项→用途
/openmp→打开#pragma omp在源代码中
/vd→禁止或者打开隐藏vtordisp类的成员
/vmb→Uses best base for pointers to members.
/vmg→Uses full generality for pointers to members.
/vmm→申明多继承
/vms→申明单继承
/vmv→申明虚拟继承
/Z7→产生和C 7.0兼容的调试信息
/Za→禁用语音扩展
/Zc→指定一个标准行为在/Ze下
/Ze→打开语音扩展
/Zg→产生函数原型
/ZI→在程序数据库中包括调试信息（仅x86)
/Zi→产生完整的调试信息
/Zl→从.obj文件中删除默认的库名
/Zpn→打包结构成员
/Zs→仅做语法检查
/ZW→产生一个输出文件能运行在windows运行环境

## 6.**链接**


选项→用途
/F→设置堆栈尺寸
/LD→创建动态链接库
/LDd→创建一个调试动态链接库
/link→传输指定的参数给link
/LN→创建一个MSIL模型
/MD→编译创建一个[多线程](https://link.zhihu.com/?target=https%3A//so.csdn.net/so/search%3Ffrom%3Dpc_blog_highlight%26q%3D%E5%A4%9A%E7%BA%BF%E7%A8%8B) DLL,使用msvcrt.lib
/MDd→编译创建一个调试多线程 DLL，使用msvcrtd.lib
/MT→编译创建一个多线程执行程序，使用libcmt.lib
/MTd→编译场景一个调试多线程执行程序，使用libcmtd.lib

## 7.**预编译头**


选项→用途
/Y-→在当前建造中忽略其他全部预处理头编译选项
/Yc→创建一个预编译头文件
/Yd→在全部的object文件中放置完整的调试信息
/Yu→在编译期间使用预编译头文件

##  8.**杂项**


选项→用途
/?→列出编译选项
@→指定一个响应文件
/analyze→打开代码分析
/bigobj→Increases the number of addressable sections in an .obj file.
/c→编译但不链接
/cgthreads→给cl.exe指定一个线程数用来优化在建造过程中的性能
/errorReport→打开在vc++终端中提供内部编译错误信息(ICE)
/FC→显示传递给cl.exe的源代码的完整路径到一个文件中
/FS→强制写入一个程序数据库文件(PDB)
/H→现在扩展名的长度
/HELP→列出编译选项
/J→改变默认char类型
/kernel→编译器和链接器将创建一个可以在windows内核中执行的执行程序
/MP→同时建造多源代码文件
/nologo→禁止显示启动版权标志
/sdl→打开一些附加的安全功能和警告
/showIncludes→在编译期间显示全部include文件的列表
/Tc/TC→指定C源代码
/Tp/TP→指定C++源代码
/V→版本信息
/Wall→打开全部警告，包括默认关闭的警告
/W→警告级别
/w→关闭全部警告
/WL→打开在用命令行编译C++源代码时使用一行显示错误和警告信息
/Wp64→侦测可能的64-bit问题
/Yd→在对象文件中放置完整的调试信息
/Yl→当创建一个调试库时植入PCH引用
/Zm→指定一个预编译头分配限制

## 9.**MinGW与MSVC编译的区别**

本人使用的是QT5.6，当时我们选择下载的是第一个VS2015版本，也就是通过MSVC方式编译，我们来对比一下这两个编译器的区别：

1. MSVC是指微软的VC编译器
2. MinGW是指是Minimalist GNU on Windows的缩写。它是一个可自由使用和自由发布的Windows特定头文件和使用GNU工具集导入库的集合，允许你在GNU/Linux和Windows平台生成本地的Windows程序而不需要第三方C运行时库。

原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/434454915