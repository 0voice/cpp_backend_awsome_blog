# 【NO.591】深入了解epoll模型（特别详细）

上网一搜epoll，基本是这样的结果出来：《多路转接I/O – epoll模型》，万变不离这个标题。

但是呢，不变的事物，我们就更应该抓出其中的重点了。

多路、转接、I/O、模型。

别急，先记住这几个词，我比较喜欢你们看我文章的时候带着问题。

## 1.什么是epoll？或者说，它和select有什么判别？

### 1.1 什么是select

有的朋友可能对select也不是很了解啊，我这里稍微科普一下：网络连接，服务器也是通过文件描述符来管理这些连接上来的客户端，既然是供连接的服务器，那就免不了要接收来自客户端的消息。那么多台客户端，消息那么的多，要是漏了一条两条重要消息，那也不要用TCP了，那怎么办？

前辈们就是有办法，轮询，轮询每个客户端文件描述符，查看他们是否带着消息，如果带着，那就处理一下；如果没带着，那就一边等着去。这就是select，轮询，颇有点领导下基层的那种感觉哈。

但是这个select的轮询呐，会有个问题，明眼人一下就能想到，那即是耗费资源啊，耗费什么资源，时间呐，慢呐（其实也挺快了，不过相对epoll来说就是慢）。

再认真想一下，还浪费什么资源，系统资源。有的客户端呐，占着那啥玩意儿不干那啥事儿，这种客户端呐，还不少。这也怪不得人家，哪儿有客户端时时刻刻在发消息，要是有，那就要小心是不是恶意攻击了。那把这么一堆偶尔动一下的客户端的文件描述符一直攥手里，累不累？能一次攥多少个？就像一个老板，一直想着下去巡视，那他可以去当车间组长了哈哈哈。

所以，select的默认上限一般是1024（FD_SETSIZE），当然我们可以手动去改，但是人家给个1024自然有人家的道理，改太大的话系统在这一块的负载就大了。

那句话怎么说的来着，你每次对系统的索取，其实都早已明码标价！哈哈哈。。。

所以，我们选用epoll模型。

讲到这里啊，如果对 select 不了解的朋友可以打开百度搜一下了，因为我们接下来要讲的内容需要你对 select 有点了解的。

### 1.2 为什么select最大只允许1024？

以前从没想过select和poll，一出来关注的就是epoll，后来明白了，对于select，也是有很多细节在里面的。源码之前，了无秘密！！！

```text
#define FD_SETSIZE 1024
#define NFDBITS (8 * sizeof(unsigned long))
#define __FDSET_LONGS (FD_SETSIZE/NFDBITS)

typedef struct {
  unsigned long fds_bits[__FDSET_LONGS];
} fd_set;
```

为什么源码要写成1024？

我觉得是历史遗留原因吧，不信你看一下你服务器在不改动的情况下能允许的最大文件描述符数。

如果有人再问：那 select 就只能监听1024个？那我就要说道说道了，谁说有1024？select 性能就这样了，再往上开意义不大，真要开，编源码去嘛。

### 1.3 什么是epoll

epoll接口是为解决Linux内核处理大量文件描述符而提出的方案。该接口属于Linux下多路I/O复用接口中select/poll的增强。其经常应用于Linux下高并发服务型程序，特别是在大量并发连接中只有少部分连接处于活跃下的情况 (通常是这种情况)，在该情况下能显著的提高程序的CPU利用率。

前面说，select就像亲自下基层视察的老板，那么epoll这个老板就要显得精明的多了。他可不亲自下基层，他找了个美女秘书，他只要盯着他的秘书看就行了，呸，他只需要听取他的秘书的汇报就行了。汇报啥呢？基层有任何消息，跟秘书说，秘书汇总之后一次性交给老板来处理。这样老板的时间不就大大的提高了嘛。

**epoll设计思路简介**

（1）epoll在Linux内核中构建了一个文件系统，该文件系统采用红黑树来构建，红黑树在增加和删除上面的效率极高，因此是epoll高效的原因之一。有兴趣可以百度红黑树了解，但在这里你只需知道其算法效率超高即可。

（2）epoll提供了两种触发模式，水平触发(LT)和边沿触发(ET)。当然，涉及到I/O操作也必然会有阻塞和非阻塞两种方案。目前效率相对较高的是 epoll+ET+非阻塞I/O 模型，在具体情况下应该合理选用当前情形中最优的搭配方案。

（3）epoll所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于1024,举个例子，在1GB内存的机器上大约是10万左右，具体数目可以下面语句查看，一般来说这个数目和系统内存关系很大。

```text
系统最大打开文件描述符数
cat /proc/sys/fs/file-max
```



```text
进程最大打开文件描述符数
ulimit -n
```

修改这个配置：

```text
sudo vi /etc/security/limits.conf
写入以下配置,soft软限制，hard硬限制

*                soft    nofile          65536
*                hard    nofile          100000
```

### 1.4 select、epoll原理图

![img](https://pic3.zhimg.com/80/v2-1b04c79ace11aa275bb76ca0f6d16eba_720w.webp)

缺点：

> （1）单进程可以打开fd有限制；
> （2）对socket进行扫描时是线性扫描，即采用轮询的方法，效率较低；
> （3）用户空间和内核空间的复制非常消耗资源；

poll与select不同的地方：采用链表的方式替换原有fd_set数据结构,而使其没有连接数的限制。

![img](https://pic2.zhimg.com/80/v2-7ac8b45ead05cd4a73f11fe9d27ab4f9_720w.webp)

epoll支持的最大链接数是进程最大可打开的文件的数目。

**epoll一定优于select吗？什么时候不是？**

尽管如此，epoll 的性能并不必然比 select 高，对于 fd 数量较少并且 fd IO 都非常繁忙的情况 select 在性能上反而有优势。

## 2.epoll的工作流程？

**epol_create**

在epoll文件系统建立了个file节点，并开辟epoll自己的内核高速cache区，建立红黑树，建立一个双向链表，用于存储准备就绪的事件。

epoll_create传入一个size参数，size参数只要>0即可，没有任何意义。epoll_create调用函数sys_epoll_create1实现eventpoll的初始化。sys_epoll_create1通过ep_alloc生成一个eventpoll对象，并初始化eventpoll的三个等待队列，wait，poll_wait以及rdlist （ready的fd list）。同时还会初始化被监视fs的rbtree 根节点。

1、首先创建一个struct eventpoll对象；

2、然后分配一个未使用的文件描述符；

3、然后创建一个struct file对象，将file中的struct file_operations *f_op设置为全局变量eventpoll_fops，将void *private指向刚创建的eventpoll对象ep；

4、然后设置eventpoll中的file指针；

5、最后将文件描述符添加到当前进程的文件描述符表中，并返回给用户

![img](https://pic1.zhimg.com/80/v2-f47a8e75f33e160b4e48a77668ae8604_720w.webp)

**epoll_create返回的fd是什么？**

这个模块在内核初始化时（操作系统启动）注册了一个新的文件系统，叫"eventpollfs"（在eventpoll_fs_type结构里），然后挂载此文件系统。另外还创建两个内核cache（在内核编程中，如果需要频繁分配小块内存，应该创建kmem_cahe来做“内存池”）,分别用于存放struct epitem和eppoll_entry。这个内核高速cache区，就是建立连续的物理内存页，就是物理上分配好你想要的size的内存对象，每次使用时都是使用空闲的已分配好的内存。

现在想想epoll_create为什么会返回一个新的fd？

因为它就是在这个叫做"eventpollfs"的文件系统里创建了一个新文件！返回的就是这个文件的fd索引。完美地遵行了Linux一切皆文件的特色。

**epoll_ctl**

1、epoll_ctl()首先判断op是不是删除操作，如果不是则将event参数从用户空间拷贝到内核中。

2、接下来判断用户是否设置了EPOLLEXCLUSIVE标志，这个标志是4.5版本内核才有的，主要是为了解决同一个文件描述符同时被添加到多个epoll实例中造成的“惊群”问题，详细描述可以看这里。 这个标志的设置有一些限制条件，比如只能是在EPOLL_CTL_ADD操作中设置，而且对应的文件描述符本身不能是一个epoll实例。

3、接下来从传入的文件描述符开始，一步步获得struct file对象，再从struct file中的private_data字段获得struct eventpoll对象。（那这个头那个头的都在这里面了）

4、如果要添加的文件描述符本身也代表一个epoll实例，那么有可能会造成死循环，内核对此情况做了检查，如果存在死循环则返回错误。

5、接下来会从epoll实例的红黑树里寻找和被监控文件对应的epollitem对象，如果不存在，也就是之前没有添加过该文件，返回的会是NULL。

6、ep_find()函数本质是一个红黑树查找过程，红黑树查找和插入使用的比较函数是ep_cmp_ffd()，先比较struct file对象的地址大小，相同的话再比较文件描述符大小。struct file对象地址相同的一种情况是通过dup()系统调用将不同的文件描述符指向同一个struct file对象。

7、接下来会根据操作符op的不同做不同的处理，这里我们只看op等于EPOLL_CTL_ADD时的添加操作。首先会判断上一步操作中返回的epollitem对象地址是否为NULL，不是NULL说明该文件已经添加过了，返回错误，否则调用ep_insert()函数进行真正的添加操作。在添加文件之前内核会自动为该文件增加POLLERR和POLLHUP事件。

8、ep_insert()函数中，首先判断epoll实例中监视的文件数量是否已超过限制，没问题则为待添加的文件创建一个epollitem对象。

9、接下来是比较重要的操作：将epollitem对象添加到被监视文件的等待队列上去。等待队列实际上就是一个回调函数链表，定义在/include/linux/wait.h文件中。因为不同文件系统的实现不同，无法直接通过struct file对象获取等待队列，因此这里通过struct file的poll操作，以回调的方式返回对象的等待队列，这里设置的回调函数是ep_ptable_queue_proc。

10、结构体ep_queue的作用是能够在poll的回调函数中取得对应的epollitem对象，这种做法在Linux内核里非常常见。

11、在回调函数ep_ptable_queue_proc中，内核会创建一个struct eppoll_entry对象，然后将等待队列中的回调函数设置为ep_poll_callback()。也就是说，当被监控文件有事件到来时，比如socker收到数据时，ep_poll_callback()会被回调。

12、ep_item_poll()调用完成之后，会将epitem中的fllink字段添加到struct file中的f_ep_links链表中，这样就可以通过struct file找到所有对应的struct epollitem对象。

13、然后就是将epollitem插入到红黑树中。

14、最后再更新下状态就返回了，插入操作也就完成了。

（在这个实现时，将用户空间epoll_event拷贝到内核中，后续可以将其转化为epitem作为节点存入红黑树中，从eventpoll的红黑树中查找fd所对应的epitem实例（二分搜索），根据传入的op参数行为进行switch判断，对红黑树进行不同的操作。对于ep_insert，首先设置了对应的回调函数，然后调用被监控文件的poll方法（每个支持poll的设备驱动程序都要调用），其实就是在poll里调用了回调函数，这个回调函数实际上不是真正的回调函数，真正的回调函数(ep_poll_callback)在该函数内调用，这个回调函数只是创建了struct eppoll_entry，将真正回调函数和epitem关联起来，之后将其加入设备等待队列。当设备就绪，唤醒等待队列上的等待者，调用对应的真正的回调函数，这个回调函数实际上就是将红黑树上收到event的epitem插入到它的就绪队列中并唤醒调用epoll_wait进程。在ep_insert中还将epitem插入到eventpoll中的红黑树上，然后还会去判断当前插入的event是否是刚好发生，如果是直接将其加入就绪队列，然后唤醒epoll_wait。）

**epoll_wait**

观察就绪列表里面有没有数据，并进行提取和清空就绪列表，非常高效。

ep_poll_callback函数主要的功能是将被监视文件的等待事件就绪时，将文件对应的epitem实例添加到就绪队列中，当用户调用epoll_wait()时，内核会将就绪队列中的事件报告给用户。

在epoll_wait主要是调用了ep_poll，在ep_poll里直接判断就绪链表有无数据，有数据就返回，没有数据就sleep，等到timeout时间到后即使链表没数据也返回。当有数据时，还需要将内核就绪事件拷贝到传入参数的events中的用户空间，就绪链表中的数据一旦拷贝就没有了，所以这里要区分LT和ET，如果是LT有可能会将后续的重新放入就绪链表。

ps：我们在调用ep_send_events_proc()将就绪队列中的事件拷贝给用户的期间，新就绪的events被挂载到eventpoll.ovflist所以我们需要遍历eventpoll.ovflist将所有已就绪的epitem 重新挂载到就绪队列中，等待下一次epoll_wait()进行交付…

![img](https://pic4.zhimg.com/80/v2-041903f671cd72b4c1278090cf8c946f_720w.webp)

## 3.LT和ET实现的区别？

由epoll_wait进行实现，如果是LT模式，它发现socket上还有未处理的事件，则在清理就绪列表后，重新把句柄放回刚刚清空的就绪列表。

LT(level triggered) 是 缺省 的工作方式 ，并且同时支持 block 和 no-block socket. 在这种做法中，内核告诉你一个文件描述符是否就绪了，然后你可以对这个就绪的 fd 进行 IO 操作。如果你不作任何操作，内核还是会继续通知你的，所以，这种模式编程出错误可能性要小一点。传统的 select/poll 都是这种模型的代表．

ET(edge-triggered) 是高速工作方式 ，只支持 no-block socket 。在这种模式下，当描述符从未就绪变为就绪时，内核通过 epoll 告诉你。然后它会假设你知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知，直到你做了某些操作导致那个文件描述符不再为就绪状态了 ( 比如，你在发送，接收或者接收请求，或者发送接收的数据少于一定量时导致了一个 EWOULDBLOCK 错误）。但是请注意，如果一直不对这个 fd 作 IO 操作 ( 从而导致它再次变成未就绪 ) ，内核不会发送更多的通知 (only once), 不过在 TCP 协议中， ET 模式的加速效用仍需要更多的 benchmark 确认。

epoll 工作在 ET 模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读 / 阻塞写操作把处理多个文件描述符的任务饿死。最好以下面的方式调用 ET 模式的 epoll 接口，在后面会介绍避免可能的缺陷。

- 基于非阻塞文件句柄
- 只有当 read(2) 或者 write(2) 返回 EAGAIN 时才需要挂起，等待。但这并不是说每次 read() 时都需要循环读，直到读到产生一个 EAGAIN 才认为此次事件处理完成，当 read() 返回的读到的数据长度小于请求的数据长度时，就可以确定此时缓冲中已没有数据了，也就可以认为此事读事件已处理完成。

![img](https://pic4.zhimg.com/80/v2-1cdda3420065a89dbb5758788f0d60b7_720w.webp)

红黑树中每个成员由描述符值和所要监控的文件描述符指向的文件表项的引用等组成。

## 4.关键数据结构

```text
// epoll的核心实现,对应于一个epoll描述符  
struct eventpoll {  
    spinlock_t lock;  
    struct mutex mtx;  
    wait_queue_head_t wq; // sys_epoll_wait() 等待在这里  
    // f_op->poll()  使用的, 被其他事件通知机制利用的wait_address  
    wait_queue_head_t poll_wait;  
    //已就绪的需要检查的epitem 列表 
    struct list_head rdllist;  
    //保存所有加入到当前epoll的文件对应的epitem  
    struct rb_root rbr;  
    // 当正在向用户空间复制数据时, 产生的可用文件  
    struct epitem *ovflist;  
    /* The user that created the eventpoll descriptor */  
    struct user_struct *user;  
    struct file *file;  
    //优化循环检查，避免循环检查中重复的遍历
    int visited;  
    struct list_head visited_list_link;  
}  
```



```text
// 对应于一个加入到epoll的文件  
struct epitem {  
    // 挂载到eventpoll 的红黑树节点  
    struct rb_node rbn;  
    // 挂载到eventpoll.rdllist 的节点  
    struct list_head rdllink;  
    // 连接到ovflist 的指针  
    struct epitem *next;  
    /* 文件描述符信息fd + file, 红黑树的key */  
    struct epoll_filefd ffd;  
    /* Number of active wait queue attached to poll operations */  
    int nwait;  
    // 当前文件的等待队列(eppoll_entry)列表  
    // 同一个文件上可能会监视多种事件,  
    // 这些事件可能属于不同的wait_queue中  
    // (取决于对应文件类型的实现),  
    // 所以需要使用链表  
    struct list_head pwqlist;  
    // 当前epitem 的所有者  
    struct eventpoll *ep;  
    /* List header used to link this item to the &quot;struct file&quot; items list */  
    struct list_head fllink;  
    /* epoll_ctl 传入的用户数据 */  
    struct epoll_event event;  
};  
```



```text
// 与一个文件上的一个wait_queue_head 相关联，因为同一文件可能有多个等待的事件，
//这些事件可能使用不同的等待队列  
struct eppoll_entry {  
    // List struct epitem.pwqlist  
    struct list_head llink;  
    // 所有者  
    struct epitem *base;  
    // 添加到wait_queue 中的节点  
    wait_queue_t wait;  
    // 文件wait_queue 头  
    wait_queue_head_t *whead;  
}; 
```

## 5.红黑树在epoll中的作用

![img](https://pic3.zhimg.com/80/v2-c1030e59657ec2413bf526f043ba1b76_720w.webp)

哈希表. 空间因素,可伸缩性.

(1)频繁增删. 哈希表需要预估空间大小, 这个场景下无法做到.

间接影响响应时间,假如要resize,原来的数据还得移动.即使用了一致性哈希算法,

也难以满足非阻塞的timeout时间限制.(时间不稳定)

(2) 百万级连接,哈希表有镂空的空间,太浪费内存.

跳表. 慢于红黑树. 空间也高.

红黑树. 经验判断,内核的其他地方如防火墙也使用红黑树,实践上看性能最优.

AVL树.平衡二叉树能够保证在最坏的情况下也能达到lgN，要实现这一目标，我们就要保证在插入完成后始终保持平衡状态。在一棵具有N个节点的树中，我们希望该树的高度能够维持在lgN左右，这样我们就能保证只需要lgN次比较操作就可以查找到想要的值。不幸的是，每次插入元素之后维持树的平衡状态太昂贵。所以就出现一些新的数据结构来保证在最坏的情况下插入和查找效率都能保证在对数的时间复杂度内完成。

**文件描述符是以什么方式挂载在红黑树的节点中？是int吗？**

```text
/* 文件描述符信息fd + file, 红黑树的key */  
struct epoll_filefd ffd;
```

## 6.epoll惊群了解多少？

和虚假唤醒有点像。

考虑如下场景：

主进程创建socket, bind, listen之后，fork出多个子进程，每个子进程都开始循环处理（accept)这个socket。每个进程都阻塞在accpet上，当一个新的连接到来时，所有的进程都会被唤醒，但其中只有一个进程会accept成功，其余皆失败，重新休眠。这就是accept惊群。

那么这个问题真的存在吗？

事实上，历史上，Linux 的 accpet 确实存在惊群问题，但现在的内核都解决该问题了。即，当多个进程/线程都阻塞在对同一个 socket 的 accept 调用上时，当有一个新的连接到来，内核只会唤醒一个进程，其他进程保持休眠，压根就不会被唤醒。

如上所述，accept 已经不存在惊群问题，但 epoll 上还是存在惊群问题。即，如果多个进程/线程阻塞在监听同一个 listening socket fd 的 epoll_wait 上，当有一个新的连接到来时，所有的进程都会被唤醒。

accept 确实应该只能被一个进程调用成功，内核很清楚这一点。但 epoll 不一样，他监听的文件描述符，除了可能后续被 accept 调用外，还有可能是其他网络 IO 事件的，而其他 IO 事件是否只能由一个进程处理，是不一定的，内核不能保证这一点，这是一个由用户决定的事情，例如可能一个文件会由多个进程来读写。所以，对 epoll 的惊群，内核则不予处理。

## 7.epoll是线程安全的吗？

简要结论就是epoll是通过锁来保证线程安全的, epoll中粒度最小的自旋锁ep->lock(spinlock)用来保护就绪的队列, 互斥锁ep->mtx用来保护epoll的重要数据结构红黑树。

先来看一下epoll的核心数据结构

```text
struct eventpoll {
	...
	/* 一个自旋锁 */
	spinlock_t lock;
		
	/* 一个互斥锁 */
	struct mutex mtx;

	/* List of ready file descriptors */
	/* 就绪fd队列 */
	struct list_head rdllist;

	/* RB tree root used to store monitored fd structs */
	/* 红黑树 */
	struct rb_root_cached rbr;
	...
};
```

**epoll_ctl()**

以下代码即为epoll_ctl()接口的实现, 当需要根据不同的operation通过ep_insert() 或者ep_remove()等接口对epoll自身的数据结构进行操作时都提前获得了ep->mex锁.

```text
/ * epoll_ctl() 接口 */
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
	struct epoll_event __user *, event)
{
	...
	/* 获得 mtx 锁 */
	mutex_lock_nested(&ep->mtx, 0);
	...

	epi = ep_find(ep, tf.file, fd);

	error = -EINVAL;
	switch (op) {
	case EPOLL_CTL_ADD:
		/* 
		 * 通过ep_insert()接口来完成EPOLL_CTL_ADD的操作
		 * /
		if (!epi) {
			epds.events |= POLLERR | POLLHUP;
			error = ep_insert(ep, &epds, tf.file, fd, full_check);
		} else
			error = -EEXIST;
		if (full_check)
			clear_tfile_check_list();
		break;
	case EPOLL_CTL_DEL:
		/* 
		 * 通过ep_insert()接口来完成EPOLL_CTL_ADD的操作
		 * /
		if (epi)
			error = ep_remove(ep, epi);
		else
			error = -ENOENT;
		break;
	case EPOLL_CTL_MOD:
		/* 
		 * 通过ep_insert()接口来完成EPOLL_CTL_ADD的操作
		 * /
		if (epi) {
			if (!(epi->event.events & EPOLLEXCLUSIVE)) {
				epds.events |= POLLERR | POLLHUP;
				error = ep_modify(ep, epi, &epds);
			}
		} else
			error = -ENOENT;
		break;
	}
	if (tep != NULL)
		mutex_unlock(&tep->mtx);
	mutex_unlock(&ep->mtx);		/* 释放mtx锁 */
	...
}
```

**epoll_wait()**

```text
SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events,
		int, maxevents, int, timeout)
{
	...
	/* Time to fish for events ... */
	/* 
	 * 可以看到epoll_wait()是通过ep_poll()来等待就绪事件的.
	 * /
	error = ep_poll(ep, events, maxevents, timeout);
	...
}
```



```text
static int ep_poll_callback(wait_queue_entry_t *wait, unsigned mode, int sync, void *key)
{
	int pwake = 0;
	unsigned long flags;
	struct epitem *epi = ep_item_from_wait(wait);
	struct eventpoll *ep = epi->ep;
	int ewake = 0;
	
	/* 获得自旋锁 ep->lock来保护就绪队列
	 * 自旋锁ep->lock在ep_poll()里被释放
	 * /
	spin_lock_irqsave(&ep->lock, flags);

	/* If this file is already in the ready list we exit soon */
	/* 在这里将就绪事件添加到rdllist */
	if (!ep_is_linked(&epi->rdllink)) {
		list_add_tail(&epi->rdllink, &ep->rdllist);
		ep_pm_stay_awake_rcu(epi);
	}
	...
}
```

## 8.epoll API

回头来看看API吧。

头文件

```text
#include<sys/epoll.h>
```

创建句柄

```text
int epoll_create(int size);
```

创建一个epoll句柄，参数size用于告诉内核监听的文件描述符个数，跟内存大小有关。

返回epoll 文件描述符

**epoll控制函数**

```text
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event ); // 成功，返0，失败返-1
```

控制某个epoll监控的文件描述符上的事件：注册，修改，删除

参数释义：

epfd：为epoll的句柄

op:表示动作，用3个宏来表示

··· EPOLL_CTL_ADD（注册新的 fd 到epfd）

··· EPOLL_CTL_DEL（从 epfd 中删除一个 fd）

··· EPOLL_CTL_MOD（修改已经注册的 fd 监听事件）

event：告诉内核需要监听的事件

```text
typedef union epoll_data
{
    void* ptr;
    int fd;
    __uint32_t u32;
    __uint64_t u64;
} epoll_data_t;  /* 保存触发事件的某个文件描述符相关的数据 */
 
struct epoll_event
{
    __uint32_t events;  /* epoll event */
    epoll_data_t data;  /* User data variable */
};
/* epoll_event.events:
  EPOLLIN  表示对应的文件描述符可以读
  EPOLLOUT 表示对应的文件描述符可以写
  EPOLLPRI 表示对应的文件描述符有紧急的数据可读
  EPOLLERR 表示对应的文件描述符发生错误
  EPOLLHUP 表示对应的文件描述符被挂断
  EPOLLET  设置ET模式
*/
```

**epoll消息读取**

```text
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

等待所监控文件描述符上有事件的产生

参数释义：

events：用来从内核得到事件的集合

maxevent：用于告诉内核这个event有多大，这个maxevent不能大于创建句柄时的size

timeout：超时时间

··· -1：阻塞

··· 0：立即返回

···>0：指定微秒

成功返回有多少个文件描述符准备就绪，时间到返回0，出错返回-1.

代码示例

```text
/* 实现功能：通过epoll, 处理多个socket
 * 监听一个端口,监听到有链接时,添加到epoll_event
 * xs
 */
 
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <poll.h>
#include <sys/epoll.h>
#include <sys/time.h>
#include <netinet/in.h>
#include <unistd.h> 

#define MYPORT 12345
 
//最多处理的connect
#define MAX_EVENTS 500
 
//当前的连接数
int currentClient = 0; 
 
//数据接受 buf
#define REVLEN 10
char recvBuf[REVLEN];
 
 
//epoll描述符
int epollfd;
//事件数组
struct epoll_event eventList[MAX_EVENTS];
 
void AcceptConn(int srvfd);
void RecvData(int fd);
 
int main()
{
    int i, ret, sinSize;
    int recvLen = 0;
    fd_set readfds, writefds;
    int sockListen, sockSvr, sockMax;
    int timeout;
    struct sockaddr_in server_addr;
    struct sockaddr_in client_addr;
    
    //socket
    if((sockListen=socket(AF_INET, SOCK_STREAM, 0)) < 0)
    {
        printf("socket error\n");
        return -1;
    }
    
    bzero(&server_addr, sizeof(server_addr));
    server_addr.sin_family  =  AF_INET;
    server_addr.sin_port = htons(MYPORT);
    server_addr.sin_addr.s_addr  =  htonl(INADDR_ANY); 
    
    //bind
    if(bind(sockListen, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0)
    {
        printf("bind error\n");
        return -1;
    }
    
    //listen
    if(listen(sockListen, 5) < 0)
    {
        printf("listen error\n");
        return -1;
    }
    
    // epoll 初始化
    epollfd = epoll_create(MAX_EVENTS);
    struct epoll_event event;
    event.events = EPOLLIN|EPOLLET;
    event.data.fd = sockListen;
    
    //add Event
    if(epoll_ctl(epollfd, EPOLL_CTL_ADD, sockListen, &event) < 0)
    {
        printf("epoll add fail : fd = %d\n", sockListen);
        return -1;
    }
    
    //epoll
    while(1)
    {
        timeout=3000;                
        //epoll_wait
        int ret = epoll_wait(epollfd, eventList, MAX_EVENTS, timeout);
        
        if(ret < 0)
        {
            printf("epoll error\n");
            break;
        }
        else if(ret == 0)
        {
            printf("timeout ...\n");
            continue;
        }
        
        //直接获取了事件数量,给出了活动的流,这里是和poll区别的关键
        int i = 0;
        for(i=0; i<ret; i++)
        {
            //错误退出
            if ((eventList[i].events & EPOLLERR) ||
                (eventList[i].events & EPOLLHUP) ||
                !(eventList[i].events & EPOLLIN))
            {
              printf ( "epoll error\n");
              close (eventList[i].data.fd);
              return -1;
            }
            
            if (eventList[i].data.fd == sockListen)
            {
                AcceptConn(sockListen);
        
            }else{
                RecvData(eventList[i].data.fd);
            }
        }
    }
    
    close(epollfd);
    close(sockListen);
    
 
    return 0;
}
 
/**************************************************
函数名：AcceptConn
功能：接受客户端的链接
参数：srvfd：监听SOCKET
***************************************************/
void AcceptConn(int srvfd)
{
    struct sockaddr_in sin;
    socklen_t len = sizeof(struct sockaddr_in);
    bzero(&sin, len);
 
    int confd = accept(srvfd, (struct sockaddr*)&sin, &len);
 
    if (confd < 0)
    {
       printf("bad accept\n");
       return;
    }else
    {
        printf("Accept Connection: %d", confd);
    }
    //将新建立的连接添加到EPOLL的监听中
    struct epoll_event event;
    event.data.fd = confd;
    event.events =  EPOLLIN|EPOLLET;
    epoll_ctl(epollfd, EPOLL_CTL_ADD, confd, &event);
}
 
//读取数据
void RecvData(int fd)
{
    int ret;
    int recvLen = 0;
    
    memset(recvBuf, 0, REVLEN);
    printf("RecvData function\n");
    
    if(recvLen != REVLEN)
    {
        while(1)
        {
            //recv数据
            ret = recv(fd, (char *)recvBuf+recvLen, REVLEN-recvLen, 0);
            if(ret == 0)
            {
                recvLen = 0;
                break;
            }
            else if(ret < 0)
            {
                recvLen = 0;
                break;
            }
            //数据接受正常
            recvLen = recvLen+ret;
            if(recvLen<REVLEN)
            {
                continue;
            }
            else
            {
                //数据接受完毕
                printf("buf = %s\n",  recvBuf);
                recvLen = 0;
                break;
            }
        }
    }
 
    printf("data is %s", recvBuf);
}
```

## 9.番外：高效的并发方式

并发编程的目的是让程序”同时”执行多个任务。如果程序是计算密集型的，并发编程并没有什么优势，反而由于任务的切换使效率降低。但如果程序是I/O密集型的，那就不同了。

并发模式是指I/O处理单元和多个逻辑单元之间协调完成任务的方法，服务器主要有两种并发编程模式:半同步/半异步(half-sync/half-async)模式和领导者/追随者(Leader/Followers)模式。

这里讲一个“半同步/半异步”。

下面的内容需要有一定的基础了，小白可以收藏一下以后变强了再看。

**半同步/半异步模式**

在半同步/半异步模式中，同步线程用于处理客户逻辑，异步线程用于处理I/O事件。异步线程监听到客户请求之后就将其封装成请求对象并插入到请求队列中。请求队列将通知某个工作在同步模式的工作线程来读取并处理该请求对象。

![img](https://pic2.zhimg.com/80/v2-2c7d71211a356909e592e6b090ce7521_720w.webp)

**半同步/半反应堆模式(half-sync/half-reactive模式)**

半同步/半反应堆模式是半同步/半异步模式的一种变体。

其结构如下图:

![img](https://pic2.zhimg.com/80/v2-48c896f892fbbca0a88211339369688d_720w.webp)

在上图中，异步线程只有一个，由主线程充当，负责监听socket上的事件。如果监听socket上有新的连接请求到来，主线程就接受新的连接socket，然后往epoll内核事件表中注册该socket上的读写事件。如果连接socket上有读写事件发生，即有新的客户请求到来或有数据要发送至客户端，主线程就将该连接socket插入到请求队列中，所有工作线程都睡眠在请求队列上，当有任务到来时，他们通过竞争来获取任务的接管权。

由于主线程插入请求队列中的任务是就绪的连接socket，所以该半同步/半反应堆模式所采用的事件处理模式是Reactor模式，即工作线程要自己从socket上读写数据。当然，半同步/半反应堆模式也可以用模拟的Proactor事件处理模式，即由主线程来完成数据的读写操作，此时主线程将应用程序数据、任务类型等信息封装为一个任务对象，然后将其插入到请求队列。

半同步/半反应堆模式的缺点:

主线程和工作线程共享请求队列，因而请求队列是临界资源，所以对请求队列操作的时候需要加锁保护。

每个工作线程在同一时间只能处理一个客户请求。如果客户数量增多，则请求队列中堆积任务太多，客户端的响应会越来越慢。如果增多工作线程的话，则线程的切花也将消耗大量的CPU时间。

**高效的半同步/半异步模式**

在半同步/半反应堆模式中，每个工作线程同时只能处理一个客户请求，如果并发量大的话，客户端响应会很慢。如果每个工作线程都能同时处理多个客户链接，则就能改善这种情况，所以就有了高效的半同步/半异步模式。

其结构如图:

![img](https://pic1.zhimg.com/80/v2-ace9af7e4a576a1d70b07446d184b520_720w.webp)

主线程只管监听socket,当有新的连接socket到来时，主线程就接受连接并返回新的连接socket给某个工作线程。此后该新连接socket上的任何I/O操作都由被选中的工作线程来处理，直到客户端关闭连接。当工作线程检测到有新的连接socket到来时，就把该新的连接socket的读写事件注册到自己的epoll内核事件表中。

主线程和工作线程都维持自己的事件循环，他们各自独立的监听不同事件。因此在这种高效的半同步/半异步模式中，每个线程都工作在异步模式中，所以它并非严格意义上的半同步/半异步模式。

原文地址：https://zhuanlan.zhihu.com/p/427512269

作者：CPP后端技术