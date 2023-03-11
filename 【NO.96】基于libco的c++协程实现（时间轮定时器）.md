# 【NO.96】基于libco的c++协程实现（时间轮定时器）

## 1.**在后端的开发中，定时器有很广泛的应用。**

比如：

心跳检测

倒计时

游戏开发的技能冷却

redis的键值的有效期等等，都会使用到定时器。

## 2.定时器的实现数据结构选择

红黑树

对于增删查，时间复杂度为O(logn)，对于红黑树最⼩节点为最左侧节点，时间复杂度O(logn)

最小堆

对于增查，时间复杂度为O(logn)，对于删时间复杂度为O(n)，但是可以通过辅助数据结构（ map 或者hashtable来快速索引节点）来加快删除操作；对于最⼩节点为根节点，时间复杂度为O(1)

跳表

对于增删查，时间复杂度为O(logn)，对于跳表最⼩节点为最左侧节点，时间复杂度为O(1)，但是空间复杂度⽐较⾼，为O(1.5n)

时间轮

对于增删查，时间复杂度为O(1)，查找最⼩节点也为O(1)

## 3.libco的使用了时间轮的实现

首先，时间轮有几个结构，必须理清他们的关系。

```text
struct stTimeoutItem_t
{
	enum { eMaxTimeout = 40 * 1000 };	// 40s
	stTimeoutItem_t* pPrev;				// 前
	stTimeoutItem_t* pNext;				// 后
	stTimeoutItemLink_t* pLink;			// 链表，没有用到，写这里有毛用
 
	OnPreparePfn_t pfnPrepare;			// 不是超时的事件的处理函数
	OnProcessPfn_t pfnProcess;			// resume协程回调函数
 
	void* pArg;							// routine 协程对象指针
	bool bTimeout;						// 是否超时
	unsigned long long ullExpireTime;	// 到期时间
};
 
struct stPoll_t;
struct stPollItem_t : public stTimeoutItem_t
{
	struct pollfd* pSelf;			// 对应的poll结构
	stPoll_t* pPoll;				// 所属的stPoll_t
	struct epoll_event stEvent;		// epoll事件，poll转换过来的
};
 
// co_poll_inner 创建，管理这多个stPollItem_t
struct stPoll_t : public stTimeoutItem_t
{
	struct pollfd* fds;				// poll 的fd集合
	nfds_t nfds;					// poll 事件个数
	stPollItem_t* pPollItems;		// 要加入epoll 事件
	int iAllEventDetach;			// 如果处理过该对象的子项目pPollItems，赋值为1
	int iEpollFd;					// epoll fd句柄
	int iRaiseCnt;					// 此次触发的事件数
};
```

我把这几个结构拉一起了，

![img](https://pic3.zhimg.com/80/v2-6b637f3ab664dc6a78d39d72e0b887f2_720w.webp)

其中，能看出，stCoEpool_t管理了这一切

```text
// TimeoutItem的链表
struct stTimeoutItemLink_t
{
	stTimeoutItem_t* head;
	stTimeoutItem_t* tail;
};
 
// TimeOut 
struct stTimeout_t	// 时间伦
{
	stTimeoutItemLink_t* pItems;	// 时间轮链表，开始初始化分配只一圈的长度，后续直接使用
	int iItemSize;					// 超时链表中一圈的tick 60*1000
	unsigned long long ullStart;	// 时间轮开始时间，会一直变化
	long long llStartIdx;			// 时间轮开始的下标，会一直变化
};
 
// epoll 结构
struct stCoEpoll_t
{
	int iEpollFd;
	static const int _EPOLL_SIZE = 1024 * 10;
	struct stTimeout_t* pTimeout;					// epoll 存着时间轮
	struct stTimeoutItemLink_t* pstTimeoutList;		// 超时事件链表
	struct stTimeoutItemLink_t* pstActiveList;		// 用于signal时会插入
	co_epoll_res* result;
};
```

也就是说，一个[协程](https://link.zhihu.com/?target=https%3A//so.csdn.net/so/search%3Fq%3D%E5%8D%8F%E7%A8%8B%26spm%3D1001.2101.3001.7020)，就有一个，在co_init_curr_thread_env 中创建

它管理着超时链表，信号事件链表

其中的pTimeout，就是时间轮，也就是一个数组，这个数组的大小位60*1000

![img](https://pic3.zhimg.com/80/v2-2f7dd50629e3a224c0953a2e5d82ed22_720w.webp)

```text
stTimeout_t *AllocTimeout( int iSize )
{
	stTimeout_t *lp = (stTimeout_t*)calloc( 1,sizeof(stTimeout_t) );
	lp->iItemSize = iSize;
	// 注意这里先把item分配好了，后续直接使用
	lp->pItems = (stTimeoutItemLink_t*)calloc( 1, sizeof(stTimeoutItemLink_t) * lp->iItemSize );
	lp->ullStart = GetTickMS();
	lp->llStartIdx = 0;
	return lp;
}
```

这就是分配的时间轮的方法，首先指定了下标时间等信息，根据结构注释应该不难懂

有了这些后，再来看看时怎么添加超时事件的

```text
// apTimeout：时间轮
// apItem: 某一个定时item
// allNow：当前的时间
// 函数目的，将超时项apItem加入到apTimeout
int AddTimeout( stTimeout_t *apTimeout, stTimeoutItem_t *apItem ,unsigned long long allNow )
{
	// 这个判断有点多余，start正常已经分配了
	if( apTimeout->ullStart == 0 )
	{
		apTimeout->ullStart = allNow;
		apTimeout->llStartIdx = 0;
	}
	// 当前时间也不大可能比前面的时间大
	if( allNow < apTimeout->ullStart )
	{
		co_log_err("CO_ERR: AddTimeout line %d allNow %llu apTimeout->ullStart %llu",
					__LINE__,allNow,apTimeout->ullStart);
 
		return __LINE__;
	}
	if( apItem->ullExpireTime < allNow )
	{
		co_log_err("CO_ERR: AddTimeout line %d apItem->ullExpireTime %llu allNow %llu apTimeout->ullStart %llu",
					__LINE__,apItem->ullExpireTime,allNow,apTimeout->ullStart);
 
		return __LINE__;
	}
	// 到期时间到start的时间差
	unsigned long long diff = apItem->ullExpireTime - apTimeout->ullStart;
	// itemsize 实际上是毫秒数，如果超出了，说明设置的超时时间过长
	if( diff >= (unsigned long long)apTimeout->iItemSize )
	{
		diff = apTimeout->iItemSize - 1;
		co_log_err("CO_ERR: AddTimeout line %d diff %d",
					__LINE__,diff);
 
		//return __LINE__;
	}
	// 将apItem加到末尾
	AddTail( apTimeout->pItems + ( apTimeout->llStartIdx + diff ) % apTimeout->iItemSize , apItem );
 
	return 0;
}
```

其实，这里有个概念，stTimeoutItemLink_t 与stTimeoutItem_t，也就是说，stTimeout_t里面管理的时60*1000个链表，而每个链表有一个或者多个stTimeoutItem_t，下面这个函数，就是把节点Item加入到链表的方法。

```text
template <class TNode,class TLink>
void inline AddTail(TLink*apLink, TNode *ap)
{
	if( ap->pLink )
	{
		return ;
	}
	if(apLink->tail)
	{
		apLink->tail->pNext = (TNode*)ap;
		ap->pNext = NULL;
		ap->pPrev = apLink->tail;
		apLink->tail = ap;
	}
	else
	{
		apLink->head = apLink->tail = ap;
		ap->pNext = ap->pPrev = NULL;
	}
	ap->pLink = apLink;
}
```

![img](https://pic1.zhimg.com/80/v2-2e3b70df0f91eb6f537309ecea28977c_720w.webp)

到这里，基本把一个超时事件添加到时间轮中了，这时就应该切换协程了co_yield_env

```text
	int ret = AddTimeout( ctx->pTimeout, &arg, now );
	int iRaiseCnt = 0;
	if( ret != 0 )
	{
		co_log_err("CO_ERR: AddTimeout ret %d now %lld timeout %d arg.ullExpireTime %lld",
				ret,now,timeout,arg.ullExpireTime);
		errno = EINVAL;
		iRaiseCnt = -1;
	}
    else
	{
		co_yield_env( co_get_curr_thread_env() );
		iRaiseCnt = arg.iRaiseCnt;
	}
```

接下来，看怎么检测超时事件co_eventloop

```text
    for(;;)
	{
		// 等待事件或超时1ms
		int ret = co_epoll_wait( ctx->iEpollFd,result,stCoEpoll_t::_EPOLL_SIZE, 1 );
		
        //  遍历所有ret事件处理
		for(int i=0;i<ret;i++)
		{
			pfnPrepare(xxx)
		}
 
		// 取出所有的超时时间item，设置为超时
		TakeAllTimeout( ctx->pTimeout, now, plsTimeout );
		stTimeoutItem_t *lp = plsTimeout->head;
		while( lp )
		{
			lp->bTimeout = true;
			lp = lp->pNext;
		}
 
		// 将超时链表plsTimeout加入到plsActive
		Join<stTimeoutItem_t, stTimeoutItemLink_t>( plsActive, plsTimeout );
		lp = plsActive->head;
		while( lp )
		{
            // 弹出链表头，处理超时事件
			PopHead<stTimeoutItem_t,stTimeoutItemLink_t>( plsActive );
            if (lp->bTimeout && now < lp->ullExpireTime) 
			{
				int ret = AddTimeout(ctx->pTimeout, lp, now);
				if (!ret) 
				{
					lp->bTimeout = false;
					lp = plsActive->head;
					continue;
				}
			}
            // 只有stPool_t 才需要切协程，要切回去了
			if( lp->pfnProcess )
			{
				lp->pfnProcess( lp );
			}
			lp = plsActive->head;
		}
 
		// 如果传入该函数指针，则可以控制event_loop 退出
		if( pfn )
		{
			if( -1 == pfn( arg ) )
			{
				break;
			}
		}
	}
```

其中包括了定时事件处理，协程切换，主协程退出等操作。如果设置了主协程退出函数，则主协程可以正常的退出。

原文地址：https://zhuanlan.zhihu.com/p/573575861

作者：linux