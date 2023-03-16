# 【NO.415】epoll源码剖析：为什么使用红黑树以及如何使用红黑树

![image-20230207170509923](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230207170509923.png)

我们知道epoll的底层使用了红黑树来管理文件描述符，为什么会选择红黑树这种结构呢？

以下是个人理解：

epoll和poll的一个很大的区别在于，poll每次调用时都会存在一个将pollfd结构体数组中的每个结构体元素从用户态向内核态中的一个链表节点拷贝的过程，而内核中的这个链表并不会一直保存，当poll运行一次就会重新执行一次上述的拷贝过程，这说明一个问题：poll并不会在内核中为要监听的文件描述符长久的维护一个数据结构来存放他们，而epoll内核中维护了一个内核事件表，它是将所有的文件描述符全部都存放在内核中，系统去检测有事件发生的时候触发回调，当你要添加新的文件描述符的时候也是调用epoll_ctl函数使用EPOLL_CTL_ADD宏来插入，epoll_wait也不是每次调用时都会重新拷贝一遍所有的文件描述符到内核态。当我现在要在内核中长久的维护一个数据结构来存放文件描述符，并且时常会有插入，查找和删除的操作发生，这对内核的效率会产生不小的影响，因此需要一种插入，查找和删除效率都不错的数据结构来存放这些文件描述符，那么红黑树当然是不二的人选。

接下来我们来看看epoll底层是如何使用红黑树的

我们知道epoll在添加一个文件描述符进行监听或者删除一个文件描述符时使用的是epoll_ctl函数，该函数底层调用的是sys_epoll_ctl函数，下面给出该函数的部分源码

```text
/*
 * The following function implements the controller interface for
 * the eventpoll file that enables the insertion/removal/change of
 * file descriptors inside the interest set.  It represents
 * the kernel part of the user space epoll_ctl(2).
 */
asmlinkage long
sys_epoll_ctl(int epfd, int op, int fd, struct epoll_event __user *event)
{
	int error;
	struct file *file, *tfile;
	struct eventpoll *ep;
	struct epitem *epi;
	struct epoll_event epds;
 
	DNPRINTK(3, (KERN_INFO "[%p] eventpoll: sys_epoll_ctl(%d, %d, %d, %p)\n",
		     current, epfd, op, fd, event));
 
	error = -EFAULT;
	if (EP_OP_HASH_EVENT(op) &&
	    copy_from_user(&epds, event, sizeof(struct epoll_event)))
		goto eexit_1;
 
	/* Get the "struct file *" for the eventpoll file */
	error = -EBADF;
	file = fget(epfd);
	if (!file)
		goto eexit_1;
 
	/* Get the "struct file *" for the target file */
	tfile = fget(fd);
	if (!tfile)
		goto eexit_2;
 
	/* The target file descriptor must support poll */
	error = -EPERM;
	if (!tfile->f_op || !tfile->f_op->poll)
		goto eexit_3;
 
	/*
	 * We have to check that the file structure underneath the file descriptor
	 * the user passed to us _is_ an eventpoll file. And also we do not permit
	 * adding an epoll file descriptor inside itself.
	 */
	error = -EINVAL;
	if (file == tfile || !IS_FILE_EPOLL(file))
		goto eexit_3;
 
	/*
	 * At this point it is safe to assume that the "private_data" contains
	 * our own data structure.
	 */
	ep = file->private_data;
 
	down_write(&ep->sem);
 
	/* Try to lookup the file inside our hash table */
	epi = ep_find(ep, tfile, fd);
```

在sys_epoll_ctl的参数中，op代表要进行的操作，fd表示要被操作的文件描述符。操作类型定义在下面着三个宏中

```text
/* Valid opcodes to issue to sys_epoll_ctl() */
#define EPOLL_CTL_ADD 1
#define EPOLL_CTL_DEL 2
#define EPOLL_CTL_MOD 3
```

首先呢，会调用ep_find函数在内核事件表也就是红黑树中查找该fd是否已经存在，这里的结果会先保存在epi中，ep_find函数做了什么操作呢？这里就是我们第一个用到红黑树的地方：查找

先来看一下ep_find的实现：

```text
/*
 * Search the file inside the eventpoll hash. It add usage count to
 * the returned item, so the caller must call ep_release_epitem()
 * after finished using the "struct epitem".
 */
static struct epitem *ep_find(struct eventpoll *ep, struct file *file, int fd)
{
	int kcmp;
	unsigned long flags;
	struct rb_node *rbp;
	struct epitem *epi, *epir = NULL;
	struct epoll_filefd ffd;
 
	EP_SET_FFD(&ffd, file, fd);
	read_lock_irqsave(&ep->lock, flags);
	for (rbp = ep->rbr.rb_node; rbp; ) {
		epi = rb_entry(rbp, struct epitem, rbn);
		kcmp = EP_CMP_FFD(&ffd, &epi->ffd);
		if (kcmp > 0)
			rbp = rbp->rb_right;
		else if (kcmp < 0)
			rbp = rbp->rb_left;
		else {
			ep_use_epitem(epi);
			epir = epi;
			break;
		}
	}
	read_unlock_irqrestore(&ep->lock, flags);
 
	DNPRINTK(3, (KERN_INFO "[%p] eventpoll: ep_find(%p) -> %p\n",
		     current, file, epir));
 
	return epir;
}
```

这里的for循环就是一个红黑树的查找过程，我们可以看到这里查找的时候用到的一个变量是kcmp，这个kcmp的值就是我们的fd在红黑树中所用来排序的值。而且我们可以看到这个kcmp的值来源于宏函数EP_CMP_FFD我们来看一下这个宏函数的实现

```text
/* Compare rb-tree keys */
#define EP_CMP_FFD(p1, p2) ((p1)->file > (p2)->file ? +1: \
			    ((p1)->file < (p2)->file ? -1: (p1)->fd - (p2)->fd))
```

根据该宏函数的实现我们看到在比较时其实使用的是一个epoll_filefd的结构体中的file成员来比较的，那么我们再进入epoll_filefd中查看一下

![img](https://pic4.zhimg.com/80/v2-8a397a57324ce8c64ff02e87d3d65cff_720w.webp)

我们看到这里的file是一个struct file类型的指针，当我们比较两个file类型的指针时比较的是他们的指针的值，也就是file结构体的地址。

根据源码判断，在红黑树中排序的根据是file的地址大小。至于为什么，目前还并不是很清楚，也存在我理解错误的可能，这里不是很确定。

查找完毕后，就要开始进行具体的操作了，这里会根据宏的值判断应该进行的操作是插入，删除，还是修改。这里给出sys_epoll_ctl的剩余源码（和文章开头给出的前半部分刚好衔接）

```text
error = -EINVAL;
	switch (op) {
	case EPOLL_CTL_ADD:
		if (!epi) {
			epds.events |= POLLERR | POLLHUP;
 
			error = ep_insert(ep, &epds, tfile, fd);
		} else
			error = -EEXIST;
		break;
	case EPOLL_CTL_DEL:
		if (epi)
			error = ep_remove(ep, epi);
		else
			error = -ENOENT;
		break;
	case EPOLL_CTL_MOD:
		if (epi) {
			epds.events |= POLLERR | POLLHUP;
			error = ep_modify(ep, epi, &epds);
		} else
			error = -ENOENT;
		break;
	}
 
	/*
	 * The function ep_find() increments the usage count of the structure
	 * so, if this is not NULL, we need to release it.
	 */
	if (epi)
		ep_release_epitem(epi);
 
	up_write(&ep->sem);
 
eexit_3:
	fput(tfile);
eexit_2:
	fput(file);
eexit_1:
	DNPRINTK(3, (KERN_INFO "[%p] eventpoll: sys_epoll_ctl(%d, %d, %d, %p) = %d\n",
		     current, epfd, op, fd, event, error));
 
	return error;
}
```

我们看到这部分代码里最主要的工作就是进行这个switch，case语句所做的判断工作了，这里sys_epoll_ctl函数根据参数op的不同而调用不同的函数进行处理，我们以EPOLL_CTL_ADD宏举例，该宏要进行的操作是插入一个新的文件描述符。

epoll底层的红黑树插入是调用ep_insert插入的，而ep_insert函数里面调用了ep_rbtree_insert来进行对红黑树中一个节点的插入。这两个函数的声明如下：

```text
static void ep_rbtree_insert(struct eventpoll *ep, struct epitem *epi);
static int ep_insert(struct eventpoll *ep, struct epoll_event *event,
		     struct file *tfile, int fd);
```

我们忽略ep_insert函数其他的实现要点，直接查看它所调用的函数ep_retree_insert的实现

```text
static void ep_rbtree_insert(struct eventpoll *ep, struct epitem *epi)
{
	int kcmp;
	struct rb_node **p = &ep->rbr.rb_node, *parent = NULL;
	struct epitem *epic;
 
	while (*p) {
		parent = *p;
		epic = rb_entry(parent, struct epitem, rbn);
		kcmp = EP_CMP_FFD(&epi->ffd, &epic->ffd);
		if (kcmp > 0)
			p = &parent->rb_right;
		else
			p = &parent->rb_left;
	}
	rb_link_node(&epi->rbn, parent, p);
	rb_insert_color(&epi->rbn, &ep->rbr);
}
```

可以看到这里在插入一个新节点时对于其在红黑树中的位置的选择过程是用一个while循环来实现的，当该while循环退出后，说明我们已经找到了该节点应在的位置，接下来调用rb_link_node函数将该节点插入到红黑树中，该函数的实现很简单，就是往一颗二叉树中插入一个新的节点，实现如下

```text
static inline void rb_link_node(struct rb_node * node, struct rb_node * parent,
				struct rb_node ** rb_link)
{
	node->rb_parent = parent;
	node->rb_color = RB_RED;
	node->rb_left = node->rb_right = NULL;
 
	*rb_link = node;
}
```

然后再调用rb_insert_color函数，这个函数实现的是对插入一个新节点之后的整个红黑树进行调整的过程，这里牵扯到红黑树的旋转，不是我们本文的重点，只把代码贴上，有兴趣的同学可以下去自习。

```text
void rb_insert_color(struct rb_node *node, struct rb_root *root)
{
	struct rb_node *parent, *gparent;
 
	while ((parent = node->rb_parent) && parent->rb_color == RB_RED)
	{
		gparent = parent->rb_parent;
 
		if (parent == gparent->rb_left)
		{
			{
				register struct rb_node *uncle = gparent->rb_right;
				if (uncle && uncle->rb_color == RB_RED)
				{
					uncle->rb_color = RB_BLACK;
					parent->rb_color = RB_BLACK;
					gparent->rb_color = RB_RED;
					node = gparent;
					continue;
				}
			}
 
			if (parent->rb_right == node)
			{
				register struct rb_node *tmp;
				__rb_rotate_left(parent, root);
				tmp = parent;
				parent = node;
				node = tmp;
			}
 
			parent->rb_color = RB_BLACK;
			gparent->rb_color = RB_RED;
			__rb_rotate_right(gparent, root);
		} else {
			{
				register struct rb_node *uncle = gparent->rb_left;
				if (uncle && uncle->rb_color == RB_RED)
				{
					uncle->rb_color = RB_BLACK;
					parent->rb_color = RB_BLACK;
					gparent->rb_color = RB_RED;
					node = gparent;
					continue;
				}
			}
 
			if (parent->rb_left == node)
			{
				register struct rb_node *tmp;
				__rb_rotate_right(parent, root);
				tmp = parent;
				parent = node;
				node = tmp;
			}
 
			parent->rb_color = RB_BLACK;
			gparent->rb_color = RB_RED;
			__rb_rotate_left(gparent, root);
		}
	}
 
	root->rb_node->rb_color = RB_BLACK;
}
```

原文地址：https://zhuanlan.zhihu.com/p/366955699

作者：linux