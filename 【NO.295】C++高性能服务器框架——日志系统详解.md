# 【NO.295】C++高性能服务器框架——日志系统详解

## 1.日志文件系统

对文件系统进行修改时，需要进行很多操作。这些操作可能中途被打断，也就是说，这些操作不是“不可中断”(atomic)的。如果操作被打断，就可能造成文件系统出现不一致的状态。

例如：删除文件时，先要从目录树中移除文件的标示，然后收回文件占用的空间。如果在这两步之间操作被打断，文件占用的空间就无法收回。文件系统认为它是被占用的，但实际上目录树中已经找不到使用它的文件了。

在非日志文件系统中，要检查并修复类似的错误必须对整个文件系统的数据结构进行检查。这个操作可能会花费很长的时间。

为了避免这样的错误，日志文件系统分配了一个称为日志的区域来提前记录要对文件系统做的更改。在崩溃后，只要读取日志重新执行未完成的操作，文件系统就可以恢复一致。这种恢复是原子的，因为只存在几种情况：

- 不需要重新执行：这个事务被标记为已经完成
- 成功重新执行：根据日志，这个事务被重新执行
- 无法重新执行：这个事务会被撤销，就如同这个事务没有发生过一样
- 日志本身不完整：事务还没有被完全写入日志，他会被简单忽略

## 2.日志系统的设计

### 2.1.为什么需要日志

对于一些高频操作（如心跳包、定时器、界面绘制下的某些高频重复的行为），可能在少量次数下无法触发我们想要的行为，而通过断点的暂停方式，我们不得不重复操作几十次、上百次甚至更多，这样排查问题效率是非常低下的。对于这类操作，我们可以通过打印日志，将当时的程序行为上下文现场记录下来，然后从日志系统中找出某次不正常行为的上下文信息。

### 2.2.日志系统的技术上的实现

**同步写日志**

所谓同步写日志，指的是在输出日志的地方，将日志即时写入到文件中去。采用这种方式，势必造成CPU等待，进而导致主线程“卡”在写文件处，进而造成界面卡帧。

但是，很多时候我们不用担心这种问题，主要有两个原因：其一，是用户感觉不到这种同步写文件造成的延迟；其二，是客户端除了UI线程外，还有其他与界面无关的工作线程，在这些线程中写文件一般不会对用户的体验产生什么影响。

**多线程同步写日志出现的问题一——不同线程的日志事件时间序列错乱**

产生这种问题的主要原因是由于多个线程同时写日志到同一个文件时，产生日志的时间和实际写入磁盘的时间不是一个原子操作。我们可以这样来理解，一个线程T1在t1时刻产生了日志，另一个线程T2在t2时刻也产生了一个日志（t2 > t1）。但是由于一些原因导致线程T1发生阻塞，并没有把日志即刻写入到文件中，而线程T2并没有发生阻塞，即刻就把日志信息写入到了文件中。这种情况的存在就会导致不同线程的日志事件时间序列错乱。

**多线程同步写日志出现的问题二——不同线程的日志输出错乱拼接**

假设线程A的日志信息为AAAAAA，线程B的日志信息为BBBBBB，线程C的日志信息为CCCCCC。那么会不会产生一种情况使得日志文件中的输出结果为AABBCCAABBCCAABBCC。

实际上，在类Unix系统上（包括Linux），同一个进程内针对同一个FIEL*的操作是线程安全的，也就是说在Linux系统上不会产生上述的情况发生。

在Windows系统上，由于对FILE*的操作并不是线程安全的，可能会发生上述情况。

这种同步日志的实现方式，一般用于低频写日志的软件系统中，所以可以认为这种多线程同时写日志到同一个文件中是可行的。

**异步写日志**

所谓异步写日志，就是通过一些线程同步技术将日志先暂存下来，然后再通过一个或多个专门的日志写入线程将这些缓存的日志写入到磁盘中。

实现的时候可以使用一个队列来存储其他线程产生的日志，日志线程从该队列中取出日志，然后将日志内容写入文件。

## 3.日志系统的实现

设计一个日志系统需要考虑哪些问题？

既然是日志系统肯定需要记录日志（LogEvent），那么我们就需要一个类来表达日志的概念，这个类至少应该包含两个属性，一个是时间戳，另一个是消息本身。

其次是日志输出（LogAppender），可以输出到不同的地方，控制台、文件。

然后是将日志信息进行格式化输出（LogFormatter），LogAppender 可以引用 LogFormatter 这样就可以将 LogEvent 事件中的日志消息经过 LogFormatter 进行格式化，然后再由 LogAppender 输出。

如果想要获取日志，必须得先获取一个什么东西，这个东西可以成为 Logger，此外，还可以使用 LoggerManager 对这些不同的 Logger 进行管理。

LogLevel 类定义了日志的级别。

LogEventWarp 对 LogEvent 进行了封装，当 LogEventWarp 析构的时候，能够触发 LogEvent 将日志信息写入到指定位置。

### 3.1.类图

![img](https://pic3.zhimg.com/80/v2-3d0b04abd65c2142a1b5bbd1db46ee12_720w.webp)

### 3.2.日志写入流程图

![img](https://pic3.zhimg.com/80/v2-bab7eccf97c1cd34de90211fd47232ce_720w.webp)

### 3.3.LogEvent

LogEvent 是日志事件，所有的日志信息都是由 LogEvent 来管理，同时 LogEvent 也提供了格式化写入的功能。

```
// 日志事件
class LogEvent {
   public:
    // 指向日志事件的指针
    typedef std::shared_ptr<LogEvent> ptr;
    /*
     * 构造函数
     * 传入 Logger 指针，可以将该日志事件写入到对应的 Logger 中
     */
    LogEvent(std::shared_ptr<Logger> logger, LogLevel::Level level, const char* file, 
    ~LogEvent() {}
    // 获取该日志事件所对应的 Logger 类
    std::shared_ptr<Logger> getLogger() const {  return m_logger; }
    // 获取日志级别
    LogLevel::Level getLevel() const { return m_level; }
    // 获取文件名
    const char* getFileName() const { return m_file; }
    // 获取行号
    int32_t getLine() const { return m_line; }
    // 获取运行的时间
    uint32_t getElapse() const { return m_elapse; }
    // 获取线程 id
    uint32_t getThreadId() const { return m_threadId; }
    // 获取协程 id
    uint32_t getFiberId() const { return m_fiberId; }
    // 获取当前时间
    uint32_t getTime() const { return m_time; }
    // 获取日志内容
    std::string getContent() const { return m_ss.str(); }
    // 以 stringstream 的形式获取日志内容
    std::stringstream& getSS() { return m_ss; }
    // 将日志内容进行格式化
    void format(const char* fmt, ...);
    void format(const char* fmt, va_list al);
   private:
    const char* m_file = nullptr;  // 文件名
    int32_t m_line = 0;            // 行号
    uint32_t m_elapse = 0;         // 程序启动开始到现在的毫秒数
    uint32_t m_threadId = 0;       // 线程id
    uint32_t m_fiberId = 0;        // 协程id
    uint64_t m_time;               // 时间戳
    std::stringstream m_ss;             // 日志流
    std::shared_ptr<Logger> m_logger;   // 指向 Logger 类的指针
    LogLevel::Level m_level;            // 该日志事件的级别
};
```

下面的这两个函数提供了一种获取变长参数的方法，具体的原理可以查看这篇博客，大致的意思就是通过传入的第一个参数 fmt 来确定每一个变长参数在内存中的位置，进而获取参数的取值。

(gdb) p fmt
$3 = 0x414228 “test macro fmt error %s”

从上面调试的结果可以看出该函数获取变长参数的具体做法是在给定字符串的最后加上变长参数的格式化输出。

vasprintf 函数将变长参数的内容输出到 buf 中，若成功则返回输出内容的长度，若失败则返回 -1.

```
/**
 * 获取省略号指定的参数
 */
void LogEvent::format(const char* fmt, ...) {
    va_list al; 
    va_start(al, fmt);
    format(fmt, al);
    va_end(al);
}
/**
 * 将参数输出到m_ss中（格式化写入）
 */
void LogEvent::format(const char* fmt, va_list al) {
    char* buf = nullptr;
    int len = vasprintf(&buf, fmt, al);
    if (len != -1) {
        m_ss << std::string(buf, len);
        free(buf);
    }   
}
```

上面的这种格式化输出主要用在了下面的这个宏定义里面

```
#define RAINBOW_LOG_FMT_LEVEL(logger, level, fmt, ...) \
    if (logger->getLevel() <= level) \
        rainbow::LogEventWrap(rainbow::LogEvent::ptr(new rainbow::LogEvent(logger, level, \
                        __FILE__, __LINE__, 0, rainbow::GetThreadId(), \
                        rainbow::GetFiberId(), time(0)))).getEvent()->format(fmt, __VA_ARGS__)
```

### 3.4.LogAppender

通过该类派生出的不同子类可以将日志信息输出到不同的位置。一个 Logger 可以有多个 Appender，LogAppender 有单独的日志级别，因此可以通过设置不同级别的 Appender 从而将日志输出到不同的位置。此外每一个 LogAppender 也都会有自己单独的日志格式，从而方便进行管理。

还可以通过 scoket 套接字，将日志输出到服务器上。

```
// 日志输出到目的地（控制台、文件）
class LogAppender {
   public:
    typedef std::shared_ptr<LogAppender> ptr;
    LogAppender();
    virtual ~LogAppender() {}
    // 纯虚函数，子类必须实现该方法
    virtual void log(std::shared_ptr<Logger> logger, LogLevel::Level level,
                     LogEvent::ptr event) = 0;
    // 按照给定的格式序列化输出
    void setFormatter(LogFormatter::ptr val) { m_formatter = val; }
    // 获取日志格式
    LogFormatter::ptr getFormatter() const { return m_formatter; }
    LogLevel::Level getLevel() { return m_level; }
    void setLevel(const LogLevel::Level& level);
   protected:
    LogLevel::Level m_level;
    LogFormatter::ptr m_formatter;
};
// 输出到控制台的Appender
class StdoutLogAppender : public LogAppender {
   public:
    typedef std::shared_ptr<StdoutLogAppender> ptr;
    virtual void log(Logger::ptr logger, LogLevel::Level level,
                     LogEvent::ptr event) override;
};
// 定义输出到文件的Appender
class FileLogAppender : public LogAppender {
   public:
    typedef std::shared_ptr<FileLogAppender> ptr;
    virtual void log(Logger::ptr logger, LogLevel::Level level,
                     LogEvent::ptr event) override;
    FileLogAppender(const std::string& filename);
    // 重新打开文件，如果文件打开成功则返回true
    bool reopen();
   private:
    std::string m_filename;
    std::ofstream m_filestream;
};
```

### 3.5.LogFormatter

日志格式器（LogFormatter）可以将传入的日志格式进行解析，并可以和 LogEvent 指针结合将特定格式的日志信息输出到 stringstream 中，等待 LogEventWrap 析构的时候将日志信息写入到指定的 Appender 中。

```
// 日志格式器
class LogFormatter {
   public:
    // 指向该类的指针
    typedef std::shared_ptr<LogFormatter> ptr;
    // 日志格式
    LogFormatter(const std::string& pattern);
　　// 对日志进行解析，并返回格式化之后的string
    std::string format(std::shared_ptr<Logger> ptr, LogLevel::Level level,
                       LogEvent::ptr event);
    std::ostream& format(std::ostream& ofs, std::shared_ptr<Logger> ptr, LogLevel::Level level, LogEvent::ptr event);
   public:
    class FormatItem {
       public:
        FormatItem(const std::string& fmt = ""){};
        virtual ~FormatItem() {}
        virtual void format(std::ostream& os, std::shared_ptr<Logger> logger,
                            LogLevel::Level level, LogEvent::ptr event) = 0;
        // 注意这里的指针类型是FormatItem类型的指针
        typedef std::shared_ptr<FormatItem> ptr;
    };  
    void init();
   private:
　　// 日志格式
    std::string m_pattern;
　　// 根据日志格式解析出的日志格式单元类的指针存放至数组中
    std::vector<FormatItem::ptr> m_items;
};
```

### 3.6.Logger

日志可以用 Logger 类来进行表示，每一个 Logger 类含有多个 LoggerAppender，可以通过指针把 Logger 类传递给 LogEvent 从而使日志事件能够获取 Logger 类的一些信息。

std::enable_shared_from_this 能让一个对象（假设其名为 t ，且已被一个 std::shared_ptr 对象 pt 管理）安全地生成其他额外的 std::shared_ptr 实例（假设名为 pt1, pt2, … ） ，它们与 pt 共享对象 t 的所有权。

```
// 日志器
class Logger : public std::enable_shared_from_this<Logger> {
   public:
    typedef std::shared_ptr<Logger> ptr;
    Logger(const std::string& name = "root");
    // 只有满足日直级别的日志才会被输出
    void log(LogLevel::Level level, LogEvent::ptr event);
    void debug(LogEvent::ptr event);
    void info(LogEvent::ptr event);
    void warn(LogEvent::ptr event);
    void error(LogEvent::ptr event);
    void fatal(LogEvent::ptr event);
    void addAppender(LogAppender::ptr appender);
    void delAppender(LogAppender::ptr appender);
    LogLevel::Level getLevel() const { return m_level; }
    void setAppender(LogLevel::Level val) { m_level = val; }
    const std::string getName() const { return this->m_name; }
   private:
    std::string m_name = "root";       // 日志名称
    LogLevel::Level m_level;  // 日志器的级别
    std::list<LogAppender::ptr> m_appenders;  // Appender集合
    LogFormatter::ptr m_formatter;            // 日志格式
};
```

### 3.7.LoggerManager

管理所有的日志器，并且支持通过日志器的名字获取日志器。

```
// 管理所有的日志器
class LoggerManager {
public:
    LoggerManager();
    Logger::ptr getLogger(const std::string& name);
    void init();
    Logger::ptr getRoot() const { return m_root; }
private:
    // 通过日志器的名字获取日志器
    std::map<std::string, Logger::ptr> m_loggers;
    Logger::ptr m_root;
};
/**
 * 日志器管理类，单例模式
 */
typedef rainbow::Singleton<LoggerManager> LoggerMgr;
}  // namespace rainbow
```

原文链接：https://linuxcpp.0voice.com/?id=569

作者：[HG](https://linuxcpp.0voice.com/?auth=10)