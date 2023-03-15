# 【NO.310】linux后端开发-定时器设计详解

定时器模块是后端服务常用的功能之一，用于需要周期性的执行某些任务的场景。设计定时器模块的设计方法非常多，但关键是定时器的效率问题。让我们先从最简单的开始吧。

## **1.最简单的定时器**

一个最简单的定时器功能可以按如下思路实现：

```text
void WebSocketServer::doCheckHeartbeat()
{
    while (m_bRunning)
    {
        //休眠3秒
        sleep(3000)
        
        //检测所有会话的心跳
        checkSessionHeartbeat();
    }
}
```

上述代码在一个新的线程中每隔 3 秒对所有连接会话做心跳检测。这是一个非常简单的实现逻辑，读者可能会觉得有点不可思议，直接使用 sleep 函数休眠 3 秒作为时间间隔，这未免有点“简单粗暴”了吧？这段代码来源于我们之前一个商业项目，并且它至今工作的很好。所以凡事都没有那么绝对，在一些特殊场景下我们确实可以按这种思路来实现定时器，只不过 sleep 函数可能换成一些可以指定超时或等待时间的、让线程挂起或等待的函数（如 select、poll 等）。

但是上述实现定时器的方法毕竟适用场景太少，也不能与 one thread one loop 结构相结合，one thread one loop 结构中的定时器才是我们本文的重点。你可以认为前面介绍的定时器实现只是个热身，现在让我们正式开始。

## **2.定时器设计的基本思路**

根据实际的场景需求，我们的定时器对象一般需要一个唯一标识、过期时间、重复次数、定时器到期时触发的动作，因此一个定时器对象可以设计成如下结构：

```text
typedef std::function<void()> TimerCallback;

//定时器对象
class Timer
{
    Timer() ;
    ~Timer();

    void run()
    {
        callback();
    }
    
    //其他实现下文会逐步完善...

private:
    //定时器的id，唯一标识一个定时器
    int64_t         m_id;
    //定时器的到期时间
    time_t          m_expiredTime;
    //定时器重复触发的次数
    int32_t         m_repeatedTimes;
    //定时器触发后的回调函数
    TimerCallback   m_callback;
};
```

在 one loop one thread 结构中，我们提到使用用到定时器的程序结构：

```text
while (!m_bQuitFlag)
{
	check_and_handle_timers();
	
	epoll_or_select_func();

	handle_io_events();

	handle_other_things();
}
```

我们在函数 **check_and_handle_timers()** 中对各个定时器对象进行处理（检测是否到期，如果到期调用相应的定时器函数），我们先从最简单的情形开始讨论，将定时器对象放在一个 std::list 对象中：

```text
//m_listTimers可以是EventLoop的成员变量
std::list<Timer*> m_listTimers;

void EventLoop::check_and_handle_timers()
{
    for (auto& timer : m_listTimers)
    {
        //判断定时器是否到期
        if (timer->isExpired())
        {
            timer->run();
        }
    }
}
```

为了方便管理所有定时器对象，我们可以专门新建一个 **TimerManager** 类去对定时器对象进行管理，该对象提供了增加、移除和判断定时器是否到期等接口：

```text
class TimerManager
{
public:
    TimerManager() = default;
    ~TimerManager() = default;

    /** 添加定时器
     * @param repeatedCount 重复次数
     * @param 触发间隔
     * @
     * @return 返回创建成功的定时器id
     */
    int64_t addTimer(int32_t repeatedCount, int64_t interval, const TimerCallback& timerCallback);

    /** 移除指定id的定时器
     * @param timerId 待移除的定时器id
     * @return 成功移除定时器返回true，反之返回false
     */
    bool removeTimer(int64_t timerId);

    /** 检测定时器是否到期，如果到期则触发定时器函数
     */
    void checkAndHandleTimers();


private:
    std::list<Timer*> m_listTimers;

};
```

这样 **check_and_handle_timers()** 调用实现变成如下形式：

```text
void EventLoop::check_and_handle_timers()
{
    //m_timerManager可以是EventLoop的成员变量
    m_timerManager.checkAndHandleTimers();
}
```

**addTimer**、**removeTimer**、**checkAndHandleTimers** 的实现如下：

```text
int64_t TimerManager::addTimer(int32_t repeatedCount, int64_t interval, const TimerCallback& timerCallback)
{
    Timer* pTimer = new Timer(repeatedCount, interval, timerCallback);

    m_listTimers.push_back(pTimer);

    return pTimer->getId();
}

bool TimerManager::removeTimer(int64_t timerId)
{
    for (auto iter = m_listTimers.begin(); iter != m_listTimers.end(); ++iter)
    {
        if ((*iter)->getId() == timerId)
        {
            m_listTimers.erase(iter);
            return true;
        }
    }

    return false;
}

void TimerManager::checkAndHandleTimers()
{
    Timer* deletedTimer;
    for (auto iter = m_listTimers.begin(); iter != m_listTimers.end(); )
    {
        if ((*iter)->isExpired())
        {
            (*iter)->run();
            
            if ((*iter)->getRepeatedTimes() == 0)
            {
                //定时器不需要重复，从集合中移除该对象
                deletedTimer = *iter;
                iter = m_listTimers.erase(iter);
                delete deletedTimer;
                continue;
            }
            else 
            {
                ++iter;
            }
        }
    }
}
```

给 **addTimer** 函数传递必要的参数后创建一个 Timer 对象，并返回唯一标识该定时器对象的 id，后续步骤就可以通过定时器 id 来操作这个定时器对象。

我这里的定时器 id 使用了一个单调递增的 int64_t 的整数，你也可以使用其他类型，如 uid，只要能唯一区分每个定时器对象即可。当然，在我这里的设计逻辑中，可能多个线程多个 EventLoop，每一个 EventLoop 含有一个 m_timerManager 对象，但我希望所有的定时器的 id 能够全局唯一，所以我这里每次生成定时器 id 时使用了一个整型原子变量的 id 基数，我将它设置为 Timer 对象的静态成员变量，每次需要生成新的定时器 id 时将其递增 1 即可，这里我利用 C++ 11 的 std::mutex 对 **s_initialId** 进行保护。

```text
//Timer.h
class Timer
{
public:
	Timer::Timer(int32_t repeatedTimes, int64_t interval, const TimerCallback& timerCallback);
	~Timer() {}
	
	bool isExpired();

    void run()
    {
        callback();
    }
    
	//其他无关代码省略...
	
public:
	//生成一个唯一的id
	static int64_t generateId();
	
private:
    //定时器id基准值，初始值为 0
    static int64_t     s_initialId{0};
    //保护s_initialId的互斥体对象
    static std::mutex  s_mutex{};	   
};
```



```text
//Timer.cpp
int64_t Timer::generateId()
{
	int64_t tmpId;
	s_mutex.lock();
	++s_initialId;
	tmpId = s_initialId;
	s_mutex.unlock();
	
	return tmpId;	
}

Timer::Timer(int32_t repeatedTimes, int64_t interval, const TimerCallback& timerCallback)
{
    m_repeatedTimes = repeatedTimes;
    m_interval = interval;

    //当前时间加上触发间隔得到下一次的过期时间
    m_expiredTime = (int64_t)time(nullptr) + interval;

    m_callback = timerCallback;

    m_id = Timer::generateId();
}
```

定时器的下一次过期时间 **m_expiredTime** 是添加定时器的时间点加上触发间隔 **interval**，即上述代码第 **9** 行，也就是说我这里使用绝对时间点作为定时器的过期时间，读者在自己的实现时也可以使用相对时间间隔。

在我的实现中，定时器还有个表示触发次数的变量：**m_repeatedCount**，m_repeatedCount 为 -1 时表示不限制触发次数（即一直触发次数），m_repeatedCount 大于 0 时，每触发一次，m_repeatedCount 递减 1，一直到 m_repeatedCount 等于 0 从定时器集合中移除。

```text
void TimerManager::checkAndHandleTimers()
{
    Timer* deletedTimer;
    for (auto iter = m_listTimers.begin(); iter != m_listTimers.end(); )
    {
        if ((*iter)->isExpired())
        {
            //执行定时器事件
            (*iter)->run();
            
            if ((*iter)->getRepeatedTimes() == 0)
            {
                //定时器不需要重复触发从集合中移除该对象
                deletedTimer = *iter;
                iter = m_listTimers.erase(iter);
                delete deletedTimer;
                continue;
            }
            else 
            {
                ++iter;
            }
        }
    }
}
```

上述代码中我们先遍历定时器对象集合，然后调用 **Timer::isExpired()** 函数判断当前定时器对象是否到期，该函数的实现如下：

```text
bool Timer::isExpired()
{
    int64_t now = time(nullptr);
    return now >= m_expiredTime;
}
```

实现很简单，即用定时器的到期时间与当前系统时间做比较。

如果一个定时器已经到期了，则执行定时器 **Timer::run()**，该函数不仅调用定时器回调函数，还更新定时器对象的状态信息（如触发的次数和下一次触发的时间点）：

```text
void Timer::run()
{
    m_callback();

    if (m_repeatedTimes >= 1)
    {
        --m_repeatedTimes;
    }
		
		//计算下一次的触发时间
    m_
```

除了定时器触发次数变为 0 时会从定时器列表中移除，也可以调用**removeTimer()** 函数主动从定时器列表中移除一个定时器对象：

```text
bool TimerManager::removeTimer(int64_t timerId)
{
    for (auto iter = m_listTimers.begin(); iter != m_listTimers.end(); ++iter)
    {
        if ((*iter)->getId() == timerId)
        {
            m_listTimers.erase(iter);
            return true;
        }
    }

    return false;
}
```

**removeTimer()** 函数成功通过一个定时器 id 成功移除一个定时器对象时会返回 true，反之返回 false。

我们再贴下完整的代码：

**Timer.h**

```text
#ifndef __TIMER_H__
#define __TIMER_H__

#include <functional>

typedef std::function<void()> TimerCallback;

class Timer
{
public:
    /**
     * @param repeatedTimes 定时器重复次数，设置为-1表示一直重复下去
     * @param interval      下一次触发的时间间隔
     * @param timerCallback 定时器触发后的回调函数
     */
    Timer(int32_t repeatedTimes, int64_t interval, const TimerCallback& timerCallback);
    ~Timer();

    int64_t getId()
    {
        return m_id;
    }

    bool isExpired();

    int32_t getRepeatedTimes()
    {
        return m_repeatedTimes;
    }

    void run();

    //其他实现暂且省略
    
public:
	//生成一个唯一的id
	static int64_t generateId();

private:
    //定时器的id，唯一标识一个定时器
    int64_t                     m_id;
    //定时器的到期时间
    time_t                      m_expiredTime;
    //定时器重复触发的次数
    int32_t                     m_repeatedTimes;
    //定时器触发后的回调函数
    TimerCallback               m_callback;
    //触发时间间隔                
    int64_t                     m_interval;

    //定时器id基准值，初始值为 0
    static int64_t     			s_initialId{0};
    //保护s_initialId的互斥体对象
    static std::mutex  			s_mutex{};	
};

#endif //!__TIMER_H__
```

**Timer.cpp**

```text
#include "Timer.h"
#include <time.h>

int64_t Timer::generateId()
{
	int64_t tmpId;
	s_mutex.lock();
	++s_initialId;
	tmpId = s_initialId;
	s_mutex.unlock();
	
	return tmpId;	
}

Timer::Timer(int32_t repeatedTimes, int64_t interval, const TimerCallback& timerCallback)
{
    m_repeatedTimes = repeatedTimes;
    m_interval = interval;

    //当前时间加上触发间隔得到下一次的过期时间
    m_expiredTime = (int64_t)time(nullptr) + interval;

    m_callback = timerCallback;

    //生成一个唯一的id
    m_id = Timer::generateId();
}

bool Timer::isExpired() const
{
    int64_t now = time(nullptr);
    return now >= m_expiredTime;
}

void Timer::run()
{
    m_callback();

    if (m_repeatedTimes >= 1)
    {
        --m_repeatedTimes;
    }

    m_expiredTime += m_interval;
}
```

**TimerManager.h**

```text
#ifndef __TIMER_MANAGER_H__
#define __TIMER_MANAGER_H__

#include <stdint.h>
#include <list>

#include "Timer.h"

void defaultTimerCallback()
{

}

class TimerManager
{
public:
    TimerManager() = default;
    ~TimerManager() = default;

    /** 添加定时器
     * @param repeatedCount 重复次数
     * @param interval      触发间隔
     * @param timerCallback 定时器回调函数
     * @return              返回创建成功的定时器id
     */
    int64_t addTimer(int32_t repeatedCount, int64_t interval, const TimerCallback& timerCallback);

    /** 移除指定id的定时器
     * @param timerId 待移除的定时器id
     * @return 成功移除定时器返回true，反之返回false
     */
    bool removeTimer(int64_t timerId);

    /** 检测定时器是否到期，如果到期则触发定时器函数
     */
    void checkAndHandleTimers();


private:
    std::list<Timer*> m_listTimers;
};

#endif //!__TIMER_MANAGER_H__
```

**TimerManager.cpp**

```text
#include "TimerManager.h"
    
int64_t TimerManager::addTimer(int32_t repeatedCount, int64_t interval, const TimerCallback& timerCallback)
{
    Timer* pTimer = new Timer(repeatedCount, interval, timerCallback);

    m_listTimers.push_back(pTimer);

    return pTimer->getId();
}

bool TimerManager::removeTimer(int64_t timerId)
{
    for (auto iter = m_listTimers.begin(); iter != m_listTimers.end(); ++iter)
    {
        if ((*iter)->getId() == timerId)
        {
            m_listTimers.erase(iter);
            return true;
        }
    }

    return false;
}

void TimerManager::checkAndHandleTimers()
{
    Timer* deletedTimer;
    for (auto iter = m_listTimers.begin(); iter != m_listTimers.end(); )
    {
        if ((*iter)->isExpired())
        {
            (*iter)->run();
            
            if ((*iter)->getRepeatedTimes() == 0)
            {
                //定时器不需要触发从集合中移除该对象
                deletedTimer = *iter;
                iter = m_listTimers.erase(iter);
                delete deletedTimer;
                continue;
            }
            else 
            {
                ++iter;
            }
        }
    }
}
```

以上就是定时器的设计的基本思路，你一定要明白在在这个流程中一个定时器对象具有哪些属性，以及定时器对象该如何管理。当然，这里自顶向下一共三层结构，分别是 EventLoop、TimerManager、Timer，其中 TimerManager 对象不是必需的，在一些设计中直接用 EventLoop 封装相应方法对 Timer 对象进行管理。

理解了 one thread one loop 中定时器的设计之后，我们来看下上述定时器实现中的性能问题。

## **3.定时器效率优化**

上述定时器实现中存在严重的性能问题，即每次我们检测定时器对象是否触发都要遍历定时器集合，移除定时器对象时也需要遍历定时器集合，其实我们可以将定时器按过期时间从小到大排序，这样我们检测定时器对象时，只要从最小的过期时间开始检测，一旦找到过期时间大于当前时间的定时器对象，后面的定时器对象就不需要再判断了。

### **3.1 定时器对象集合的数据结构优化一**

我们可以在每次将定时器对象添加到集合时自动进行排序，如果我们仍然使用 std::list 作为定时器集合，我们可以给 std::list 自定义一个排序函数（从小到大排序）。实现如下：

```text
//Timer.h
class Timer
{
public:
		//无关代码省略...

    int64_t getExpiredTime() const
    {
        return m_expiredTime;
    }
};
```



```text
//TimerManager.h
struct TimerCompare  
{  
    bool operator() (const Timer* lhs, const Timer* rhs)  
    {  
        return lhs->getExpiredTime() <  rhs->getExpiredTime();
    }
}
```

每次添加定时器时调用下自定义排序函数对象 **TimerCompare** （代码第 8 行）：

```text
int64_t TimerManager::addTimer(int32_t repeatedCount, int64_t interval, const TimerCallback& timerCallback)
{
    Timer* pTimer = new Timer(repeatedCount, interval, timerCallback);

    m_listTimers.push_back(pTimer);

    //对定时器对象按过期时间从小到大排序
    m_listTimers.sort(TimerCompare());

    return pTimer->getId();
}
```

现在我们检测排序定时器是否触发就可以按过期时间从最小的一直找到大于当前系统时间的定时器对象就可以结束了：

```text
void TimerManager::checkAndHandleTimers()
{
    //遍历过程中是否调整了部分定时器的过期时间
    bool adjusted = false;
    Timer* deletedTimer;
    
    for (auto iter = m_listTimers.begin(); iter != m_listTimers.end(); )
    {
        if ((*iter)->isExpired())
        {
            (*iter)->run();
            
            if ((*iter)->getRepeatedTimes() == 0)
            {
                //定时器不需要再触发时从集合中移除该对象
                deletedTimer = *iter;
                iter = m_listTimers.erase(iter);
                delete deletedTimer;
                continue;
            }
            else 
            {
                ++iter;
                //标记下集合中有定时器调整了过期时间
                adjusted = true;
            }
        }
        else
        {
            //找到大于当前系统时间的定时器对象就不需要继续往下检查了，退出循环
            break;
        }// end if      
    }// end for-loop

    //由于调整了部分定时器的过期时间，需要重新排序一下
    if (adjusted)
    {
        m_listTimers.sort(TimerCompare());
    }
}
```

上述代码中有个细节需要注意：假设现在系统时刻是 now，定时器集合中定时器过期时间从小到大依次为 t1、t2、t3、t4、t5 ... tn，假设 now < t4 且 now > t5，即 t1、t2、t3、t4 对应的定时器会触发，触发后，会从 t1、t2、t3、t4 减去对应的时间间隔，减去之后，新的 t1’、t2‘、t3‘、t4’ 就不一定小于 t5 ~ tn 了，因此需要再次对定时器集合进行排序。但是，存在一种情形：如果 t1~t5 正好触发后其对应的触发次数变为 0，因此需要从定时器列表中移除它们，这种情形下又不需要对定时器列表进行排序。因此上述代码使用了一个 adjusted 变量以记录是否有过期时间被更新且未从列表中移除的定时器对象，如果有则之后再次对定时器集合进行排序。

上述设计虽然解决了定时器遍历效率低下的问题，但是没法解决移除一个定时器仍然需要遍历的问题， 使用链表结构的 std::list 插入非常方便但定位某个具体的元素效率就比较低了。我们将 std::list 换成 std::map 再试试，当然我们仍然需要对 std::map 中的定时器对象按过期时间进行自定义排序。

### **3.2 定时器对象集合的数据结构优化二**

Timer.h 和 Timer.cpp 文件保持不变，修改后的 TimerManager.h 与 TimerManager.cpp 文件如下：

**TimerManager.h**

```text
#ifndef __TIMER_MANAGER_H__
#define __TIMER_MANAGER_H__

#include <stdint.h>
#include <map>

#include "Timer.h"

struct TimerCompare  
{  
    bool operator () (const Timer* lhs, const Timer* rhs)  
    {  
        return lhs->getExpiredTime() <  rhs->getExpiredTime();
    }
}; 

void defaultTimerCallback()
{

}

class TimerManager
{
public:
    TimerManager() = default;
    ~TimerManager() = default;

    /** 添加定时器
     * @param repeatedCount 重复次数
     * @param interval      触发间隔
     * @param timerCallback 定时器回调函数
     * @return              返回创建成功的定时器id
     */
    int64_t addTimer(int32_t repeatedCount, int64_t interval, const TimerCallback& timerCallback);

    /** 移除指定id的定时器
     * @param timerId 待移除的定时器id
     * @return 成功移除定时器返回true，反之返回false
     */
    bool removeTimer(int64_t timerId);

    /** 检测定时器是否到期，如果到期则触发定时器函数
     */
    void checkAndHandleTimers();


private:
    //key是定时器id，value是定时器对象，注意模板的第三个参数是自定义排序对象TimerCompare
    std::map<int64_t, Timer*, TimerCompare>   m_mapTimers;
};

#endif //!__TIMER_MANAGER_H__
```

**TimerManager.cpp**

```text
#include "TimerManager.h"
    
int64_t TimerManager::addTimer(int32_t repeatedCount, int64_t interval, const TimerCallback& timerCallback)
{
    Timer* pTimer = new Timer(repeatedCount, interval, timerCallback);
    int64_t timerId = pTimer->getId();

    //插入时会自动按TimerCompare对象进行排序
    m_mapTimers[timerId] = pTimer;

    return timerId;
}

bool TimerManager::removeTimer(int64_t timerId)
{
    auto iter = m_mapTimers.find(timerId);
    if (iter != m_mapTimers.end())
    {
        m_mapTimers.erase(iter);
        return true;
    }

    return false;
}

void TimerManager::checkAndHandleTimers()
{
    //遍历过程中是否调整了部分定时器的过期时间
    bool adjusted = false;
    for (auto iter = m_mapTimers.begin(); iter != m_mapTimers.end(); )
    {
        if (iter->second->isExpired())
        {
            iter->second->run();
            
            if (iter->second->getRepeatedTimes() == 0)
            {
                //定时器不需要触发从集合中移除该对象
                iter = m_mapTimers.erase(iter);
                delete (iter->second);
                continue;
            }
            else 
            {
                ++iter;
                //标记下集合中有定时器调整了过期时间
                adjusted = true;
            }
        }
        else
        {
            //找到大于当前系统时间的定时器对象就不需要继续往下检查了，退出循环
            break;
        }
        
    }

    //由于调整了部分定时器的过期时间，需要重新排序一下
    if (adjusted)
    {
        std::map<int64_t, Timer*, TimerCompare> localMapTimers;    
        for (const auto& iter : m_mapTimers)
        {
            localMapTimers[iter.first] = iter.second;
        }

        m_mapTimers.clear();
        m_mapTimers.swap(localMapTimers);
    }
}
```

上述实现中，无论使用 std::list 还是使用 std::map 后，当某个定时器对象需要调整过期时间后仍然要对整体进行排序，这样效率非常低。

### **3.3 定时器对象集合的数据结构优化三**

实际上，为了追求定时器的效率，我们一般有两种常用的方法，**时间轮**和**时间堆**。

**时间轮**

我们先来介绍时间轮的如何实现。

**时间轮**的基本思想是将从现在时刻 t 加上一个时间间隔 interval，以 interval 为步长，将各个定时器对象的过期时间按步长分布在不同的时间槽（time slot）中，当一个时间槽中出现多个定时器对象时，这些定时器对象按加入槽中的顺序串成链表，时间轮的示意图如下：

![img](https://pic1.zhimg.com/80/v2-ab5726c14e6c621ce1933db19b4719b8_720w.webp)

因为每个时间槽的时间间隔是一定的，因此对时间轮中的定时器对象的检测会有两种方法：

第一种方法在每次检测时判断当前系统时间处于哪个时间槽中，比该槽序号小的槽中的所有定时器都已到期，执行对应的定时器函数之后，移除不需要重新触发的定时器，或重新计算需要下一次触发的定时器对象的时间并重新计算将其移到新的时间槽中。这个适用于我们上文说的 one loop one thread 结构。

第二种方法，即每次检测时假设当前的时间与之前相比跳动了一个时间轮的间隔，这种方法适用场景比较小，也不适用于我们这里介绍的 one loop one thread 结构。

时间轮的本质实际上将一个链表按时间分组，虽然提高了一些效率，但是效率上还是存在一个的问题，尤其是当某个时间槽对应的链表较长时。

**时间堆**

再来说**时间堆**，所谓时间堆其实就是利用数据结构中的小根堆（Min Heap）来组织定时器对象，根据到期时间的大小来组织。小根堆示意图如下：

![img](https://pic2.zhimg.com/80/v2-e4534d8f6d68a8e226615bd6ff540a05_720w.webp)

如图所示，图中小根堆的各个节点代表一个定时器对象，它们按过期时间从小到大排列。使用小根堆在管理定时器对象和执行效率上都要优于前面方案中 std::list 和 std::map，这是目前一些主流网络库中涉及到定时器部分的实现，如 Libevent。我在实际项目中会使用 stl 提供的优先队列，即 std::priority_queue，作为定时器的实现，使用 std::priority_queue 的排序方式是从小到大，这是因为 std::priority 从小到大排序时其内部实现数据结构也是小根堆。

## **4.对时间的缓存**

在使用定时器功能时，我们免不了要使用获取系统时间的函数，而在大多数操作系统上获取系统时间的函数属于系统调用，一次系统调用相对于 one thread one loop 结构中的其他逻辑来说可能耗时更多，因此为了提高效率，在一些对时间要求精度不是特别高的情况，我们可能会缓存一些时间，在较近的下次如果需要系统时间，可以使用上次缓存的时间，而不是再次调用获取系统时间的函数，目前不少网络库和商业服务在定时器逻辑这一块都使用这一策略。

上述逻辑的伪码如下：

```text
while (!m_bQuitFlag)
{
	//在这里第一次获取系统时间，并缓存之
	get_system_time_and_cache();
	
	//利用上一步获取的系统时间做一些耗时短的事情
	do_something_fast_with_system_timer();
	
	//这里可以不用再次获取系统时间，而是利用第一步缓存的时间作为当前系统时间
	use_cached_time_to_check_and_handle_timers();
	
	epoll_or_select_func();

	handle_io_events();

	handle_other_things();
}
```

## **5.小结**

定时器的基本实现原理和逻辑并不复杂，核心关键点是如何设计出高效的定时器对象集合数据结构，使每次的从定时器集合中增加、删除、修改和遍历定时器对象更高效。另外，为了进一步提高定时器逻辑的执行效率，在某些场景下可能会利用上次缓存的时间去代替再一次的获取系统时间的系统调用。

定时器的设计还有其他一些需要考虑的问题，例如如果服务器机器时间被认为调提前或者延后了怎么解决，以及定时器事件的时间精度等问题。

原文地址：https://zhuanlan.zhihu.com/p/530162197

作者：linux