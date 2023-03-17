# 【NO.514】面试必问的epoll技术，从内核源码出发彻底搞懂epoll

## 1.epoll概述

epoll是linux中IO多路复用的一种机制，I/O多路复用就是通过一种机制，一个进程可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。当然linux中IO多路复用不仅仅是epoll，其他多路复用机制还有select、poll，但是接下来介绍epoll的内核实现。

网上关于epoll接口的介绍非常多，这个不是我关注的重点，但是还是有必要了解。该接口非常简单，一共就三个函数，这里我摘抄了网上关于该接口的介绍：

1. int epoll_create(int size);
   创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大。这个参数不同于select()中的第一个参数，给出最大监听的fd+1的值。需要注意的是，当创建好epoll句柄后，它就是会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。
2. int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
   epoll的事件注册函数，它不同与select()是在监听事件时告诉内核要监听什么类型的事件，而是在这里先注册要监听的事件类型。第一个参数是epoll_create()的返回值，第二个参数表示动作，用三个宏来表示：
   EPOLL_CTL_ADD：注册新的fd到epfd中；
   EPOLL_CTL_MOD：修改已经注册的fd的监听事件；
   EPOLL_CTL_DEL：从epfd中删除一个fd；
   第三个参数是需要监听的fd，第四个参数是告诉内核需要监听什么事，struct epoll_event结构如下：

```text
struct epoll_event {
 __uint32_t events;  /* Epoll events */
 epoll_data_t data;  /* User data variable */
};
```

events可以是以下几个宏的集合：

> EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
> EPOLLOUT：表示对应的文件描述符可以写；
> EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
> EPOLLERR：表示对应的文件描述符发生错误；
> EPOLLHUP：表示对应的文件描述符被挂断；
> EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
> EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里

1. int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
   等待事件的产生，类似于select()调用。参数events用来从内核得到事件的集合，**maxevents告之内核这个events有多大，这个maxevents的值不能大于创建epoll_create()时的size**(备注：在4.1.2内核里面，epoll_create的size没有什么用），参数timeout是超时时间（毫秒，0会立即返回，小于0时将是永久阻塞）。该函数返回需要处理的事件数目，如返回0表示已超时

**epoll相比select/poll的优势**：

- select/poll每次调用都要传递所要监控的所有fd给select/poll系统调用（这意味着每次调用都要将fd列表从用户态拷贝到内核态，当fd数目很多时，这会造成低效）。而每次调用epoll_wait时（作用相当于调用select/poll），不需要再传递fd列表给内核，因为已经在epoll_ctl中将需要监控的fd告诉了内核（epoll_ctl不需要每次都拷贝所有的fd，只需要进行增量式操作）。所以，在调用epoll_create之后，内核已经在内核态开始准备数据结构存放要监控的fd了。每次epoll_ctl只是对这个数据结构进行简单的维护。
- select/poll一个致命弱点就是当你拥有一个很大的socket集合，不过由于网络延时，任一时间只有部分的socket是"活跃"的，但是select/poll每次调用都会线性扫描全部的集合，导致效率呈现线性下降。但是epoll不存在这个问题，它只会对"活跃"的socket进行操作---这是因为在内核实现中epoll是根据每个fd上面的callback函数实现的。
- 当我们调用epoll_ctl往里塞入百万个fd时，epoll_wait仍然可以飞快的返回，并有效的将发生事件的fd给我们用户。这是由于我们在调用epoll_create时，内核除了帮我们在epoll文件系统里建了个file结点，在内核cache里建了个红黑树用于存储以后epoll_ctl传来的fd外，还会再建立一个list链表，用于存储准备就绪的事件，当epoll_wait调用时，仅仅观察这个list链表里有没有数据即可。有数据就返回，没有数据就sleep，等到timeout时间到后即使链表没数据也返回。所以，epoll_wait非常高效。而且，通常情况下即使我们要监控百万计的fd，大多一次也只返回很少量的准备就绪fd而已，所以，epoll_wait仅需要从内核态copy少量的fd到用户态而已。那么，这个准备就绪list链表是怎么维护的呢？当我们执行epoll_ctl时，除了把fd放到epoll文件系统里file对象对应的红黑树上之外，还会给内核中断处理程序注册一个回调函数，告诉内核，如果这个fd的中断到了，就把它放到准备就绪list链表里。所以，当一个fd（例如socket）上有数据到了，内核在把设备（例如网卡）上的数据copy到内核中后就来把fd（socket）插入到准备就绪list链表里了。

## 2.源码分析

epoll相关的内核代码在fs/eventpoll.c文件中，下面分别分析epoll_create、epoll_ctl和epoll_wait三个函数在内核中的实现，分析所用linux内核源码为4.1.2版本。

### 2.1 epoll_create

epoll_create用于创建一个epoll的句柄，其在内核的系统实现如下：

sys_epoll_create:

```text
SYSCALL_DEFINE1(epoll_create, int, size)
{
 if (size <= 0)
 return -EINVAL;
 return sys_epoll_create1(0);
}
```

可见，我们在调用epoll_create时，传入的size参数，仅仅是用来判断是否小于等于0，之后再也没有其他用处。
整个函数就3行代码，真正的工作还是放在sys_epoll_create1函数中。

sys_epoll_create -> sys_epoll_create1:

```text
/*
 * Open an eventpoll file descriptor.
 */
SYSCALL_DEFINE1(epoll_create1, int, flags)
{
 int error, fd;
 struct eventpoll *ep = NULL;
 struct file *file;

 /* Check the EPOLL_* constant for consistency.  */
    BUILD_BUG_ON(EPOLL_CLOEXEC != O_CLOEXEC);

 if (flags & ~EPOLL_CLOEXEC)
 return -EINVAL;
 /*
     * Create the internal data structure ("struct eventpoll").
     */
    error = ep_alloc(&ep);
 if (error < 0)
 return error;
 /*
     * Creates all the items needed to setup an eventpoll file. That is,
     * a file structure and a free file descriptor.
     */
    fd = get_unused_fd_flags(O_RDWR | (flags & O_CLOEXEC));
 if (fd < 0) {
        error = fd;
 goto out_free_ep;
    }
    file = anon_inode_getfile("[eventpoll]", &eventpoll_fops, ep,
                 O_RDWR | (flags & O_CLOEXEC));
 if (IS_ERR(file)) {
        error = PTR_ERR(file);
 goto out_free_fd;
    }
    ep->file = file;
    fd_install(fd, file);
 return fd;

out_free_fd:
    put_unused_fd(fd);
out_free_ep:
    ep_free(ep);
 return error;
}
```

sys_epoll_create1 函数流程如下：

- 首先调用ep_alloc函数申请一个eventpoll结构，并且初始化该结构的成员，这里没什么好说的，代码如下：

sys_epoll_create -> sys_epoll_create1 -> ep_alloc:

```text
static int ep_alloc(struct eventpoll **pep)
{
    int error;
 struct user_struct *user;
 struct eventpoll *ep;
    user = get_current_user();
    error = -ENOMEM;
    ep = kzalloc(sizeof(*ep), GFP_KERNEL);
 if (unlikely(!ep))
        goto free_uid; 

    spin_lock_init(&ep->lock);
    mutex_init(&ep->mtx);
    init_waitqueue_head(&ep->wq);
    init_waitqueue_head(&ep->poll_wait);
    INIT_LIST_HEAD(&ep->rdllist);
    ep->rbr = RB_ROOT;
    ep->ovflist = EP_UNACTIVE_PTR;
    ep->user = user;

    *pep = ep;

 return 0;

free_uid:
    free_uid(user);
 return error;
}
```



- 接下来调用get_unused_fd_flags函数，在本进程中申请一个未使用的fd文件描述符。

sys_epoll_create -> sys_epoll_create1 -> ep_alloc -> get_unused_fd_flags:

```text
int get_unused_fd_flags(unsigned flags)
{
 return __alloc_fd(current->files, 0, rlimit(RLIMIT_NOFILE), flags);
}
```

linux内核中，current是个宏，返回的是一个task_struct结构（我们称之为进程描述符）的变量，表示的是当前进程，进程打开的文件资源保存在进程描述符的files成员里面，所以current->files返回的当前进程打开的文件资源。rlimit(RLIMIT_NOFILE) 函数获取的是当前进程可以打开的最大文件描述符数，这个值可以设置，默认是1024。

__alloc_fd的工作是为进程在[start,end)之间(备注：这里start为0， end为进程可以打开的最大文件描述符数)分配一个可用的文件描述符,这里就不继续深入下去了，代码如下：

sys_epoll_create -> sys_epoll_create1 -> ep_alloc -> get_unused_fd_flags -> __alloc_fd:

```text
/*
 * allocate a file descriptor, mark it busy.
 */
int __alloc_fd(struct files_struct *files,
 unsigned start, unsigned end, unsigned flags)
{
 unsigned int fd;
 int error;
 struct fdtable *fdt;

    spin_lock(&files->file_lock);
repeat:
    fdt = files_fdtable(files);
    fd = start;
 if (fd < files->next_fd)
        fd = files->next_fd;

 if (fd < fdt->max_fds)
        fd = find_next_fd(fdt, fd);

 /*
     * N.B. For clone tasks sharing a files structure, this test
     * will limit the total number of files that can be opened.
     */
    error = -EMFILE;
 if (fd >= end)
 goto out;

    error = expand_files(files, fd);
 if (error < 0)
 goto out;

 /*
     * If we needed to expand the fs array we
     * might have blocked - try again.
     */
 if (error)
 goto repeat;

 if (start <= files->next_fd)
        files->next_fd = fd + 1;

    __set_open_fd(fd, fdt);
 if (flags & O_CLOEXEC)
        __set_close_on_exec(fd, fdt);
 else
        __clear_close_on_exec(fd, fdt);
    error = fd;
#if 1
 /* Sanity check */
 if (rcu_access_pointer(fdt->fd[fd]) != NULL) {
        printk(KERN_WARNING "alloc_fd: slot %d not NULL!\n", fd);
        rcu_assign_pointer(fdt->fd[fd], NULL);
    }
#endif

out:
    spin_unlock(&files->file_lock);****
 return error;
}
```

然后，epoll_create1会调用anon_inode_getfile，创建一个file结构，如下：

sys_epoll_create -> sys_epoll_create1 -> anon_inode_getfile:

```text
/**
 * anon_inode_getfile - creates a new file instance by hooking it up to an
 *                      anonymous inode, and a dentry that describe the "class"
 *                      of the file
 *
 * @name:    [in]    name of the "class" of the new file
 * @fops:    [in]    file operations for the new file
 * @priv:    [in]    private data for the new file (will be file's private_data)
 * @flags:   [in]    flags
 *
 * Creates a new file by hooking it on a single inode. This is useful for files
 * that do not need to have a full-fledged inode in order to operate correctly.
 * All the files created with anon_inode_getfile() will share a single inode,
 * hence saving memory and avoiding code duplication for the file/inode/dentry
 * setup.  Returns the newly created file* or an error pointer.
 */
struct file *anon_inode_getfile(const char *name,
 const struct file_operations *fops,
                void *priv, int flags)
{
 struct qstr this;
 struct path path;
 struct file *file;

 if (IS_ERR(anon_inode_inode))
 return ERR_PTR(-ENODEV);

 if (fops->owner && !try_module_get(fops->owner))
 return ERR_PTR(-ENOENT);

 /*
     * Link the inode to a directory entry by creating a unique name
     * using the inode sequence number.
     */
    file = ERR_PTR(-ENOMEM);
    this.name = name;
    this.len = strlen(name);
    this.hash = 0;
    path.dentry = d_alloc_pseudo(anon_inode_mnt->mnt_sb, &this);
 if (!path.dentry)
        goto err_module;
    path.mnt = mntget(anon_inode_mnt);
 /*
     * We know the anon_inode inode count is always greater than zero,
     * so ihold() is safe.
     */
    ihold(anon_inode_inode);

    d_instantiate(path.dentry, anon_inode_inode);

    file = alloc_file(&path, OPEN_FMODE(flags), fops);
 if (IS_ERR(file))
        goto err_dput;
    file->f_mapping = anon_inode_inode->i_mapping;

    file->f_flags = flags & (O_ACCMODE | O_NONBLOCK);
    file->private_data = priv;

 return file;

err_dput:
    path_put(&path);
err_module:
    module_put(fops->owner);
 return file;
}
```

anon_inode_getfile函数中首先会alloc一个file结构和一个dentry结构，然后将该file结构与一个匿名inode节点anon_inode_inode挂钩在一起，这里要注意的是，在调用anon_inode_getfile函数申请file结构时，传入了前面申请的eventpoll结构的ep变量，申请的file->private_data会指向这个ep变量，同时，在anon_inode_getfile函数返回来后，ep->file会指向该函数申请的file结构变量。

简要说一下file/dentry/inode，当进程打开一个文件时，内核就会为该进程分配一个file结构，表示打开的文件在进程的上下文，然后应用程序会通过一个int类型的文件描述符来访问这个结构，实际上内核的进程里面维护一个file结构的数组，而文件描述符就是相应的file结构在数组中的下标。

dentry结构（称之为“目录项”）记录着文件的各种属性，比如文件名、访问权限等，每个文件都只有一个dentry结构，然后一个进程可以多次打开一个文件，多个进程也可以打开同一个文件，这些情况，内核都会申请多个file结构，建立多个文件上下文。但是，对同一个文件来说，无论打开多少次，内核只会为该文件分配一个dentry。所以，file结构与dentry结构的关系是多对一的。

同时，每个文件除了有一个dentry目录项结构外，还有一个索引节点inode结构，里面记录文件在存储介质上的位置和分布等信息，每个文件在内核中只分配一个inode。 dentry与inode描述的目标是不同的，一个文件可能会有好几个文件名（比如链接文件），通过不同文件名访问同一个文件的权限也可能不同。dentry文件所代表的是逻辑意义上的文件，记录的是其逻辑上的属性，而inode结构所代表的是其物理意义上的文件，记录的是其物理上的属性。dentry与inode结构的关系是多对一的关系。

- 最后，epoll_create1调用fd_install函数，将fd与file交给关联在一起，之后，内核可以通过应用传入的fd参数访问file结构,本段代码比较简单，不继续深入下去了。

sys_epoll_create -> sys_epoll_create1 -> fd_install:

```text
/*
 * Install a file pointer in the fd array.
 *
 * The VFS is full of places where we drop the files lock between
 * setting the open_fds bitmap and installing the file in the file
 * array.  At any such point, we are vulnerable to a dup2() race
 * installing a file in the array before us.  We need to detect this and
 * fput() the struct file we are about to overwrite in this case.
 *
 * It should never happen - if we allow dup2() do it, _really_ bad things
 * will follow.
 *
 * NOTE: __fd_install() variant is really, really low-level; don't
 * use it unless you are forced to by truly lousy API shoved down
 * your throat.  'files' *MUST* be either current->files or obtained
 * by get_files_struct(current) done by whoever had given it to you,
 * or really bad things will happen.  Normally you want to use
 * fd_install() instead.
 */

void __fd_install(struct files_struct *files, unsigned int fd,
 struct file *file)
{
 struct fdtable *fdt;

    might_sleep();
    rcu_read_lock_sched();
 while (unlikely(files->resize_in_progress)) {
        rcu_read_unlock_sched();
        wait_event(files->resize_wait, !files->resize_in_progress);
        rcu_read_lock_sched();
    }
 /* coupled with smp_wmb() in expand_fdtable() */
    smp_rmb();
    fdt = rcu_dereference_sched(files->fdt);
    BUG_ON(fdt->fd[fd] != NULL);
    rcu_assign_pointer(fdt->fd[fd], file);
    rcu_read_unlock_sched();
}

void fd_install(unsigned int fd, struct file *file)
{
    __fd_install(current->files, fd, file);
}
```

总结epoll_create函数所做的事：调用epoll_create后，在内核中分配一个eventpoll结构和代表epoll文件的file结构，并且将这两个结构关联在一块，同时，返回一个也与file结构相关联的epoll文件描述符fd。当应用程序操作epoll时，需要传入一个epoll文件描述符fd，内核根据这个fd，找到epoll的file结构，然后通过file，获取之前epoll_create申请eventpoll结构变量，epoll相关的重要信息都存储在这个结构里面。接下来，所有epoll接口函数的操作，都是在eventpoll结构变量上进行的。

所以，epoll_create的作用就是为进程在内核中建立一个从epoll文件描述符到eventpoll结构变量的通道。

### 2.2 epoll_ctl

epoll_ctl接口的作用是添加/修改/删除文件的监听事件，内核代码如下：

sys_epoll_ctl:

```text
/*
 * The following function implements the controller interface for
 * the eventpoll file that enables the insertion/removal/change of
 * file descriptors inside the interest set.
 */
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
        struct epoll_event __user *, event)
{
 int error;
 int full_check = 0;
    struct fd f, tf;
    struct eventpoll *ep;
    struct epitem *epi;
    struct epoll_event epds;
    struct eventpoll *tep = NULL;

 error = -EFAULT;
 if (ep_op_has_event(op) &&
        copy_from_user(&epds, event, sizeof(struct epoll_event)))
 goto error_return;

 error = -EBADF;
    f = fdget(epfd);
 if (!f.file)
 goto error_return;

 /* Get the "struct file *" for the target file */
    tf = fdget(fd);
 if (!tf.file)
 goto error_fput;

 /* The target file descriptor must support poll */
 error = -EPERM;
 if (!tf.file->f_op->poll)
 goto error_tgt_fput;

 /* Check if EPOLLWAKEUP is allowed */
 if (ep_op_has_event(op))
        ep_take_care_of_epollwakeup(&epds);
 

 /*
     * We have to check that the file structure underneath the file descriptor
     * the user passed to us _is_ an eventpoll file. And also we do not permit
     * adding an epoll file descriptor inside itself.
     */
 error = -EINVAL;
 if (f.file == tf.file || !is_file_epoll(f.file))
 goto error_tgt_fput;

 /*
     * epoll adds to the wakeup queue at EPOLL_CTL_ADD time only,
     * so EPOLLEXCLUSIVE is not allowed for a EPOLL_CTL_MOD operation.
     * Also, we do not currently supported nested exclusive wakeups.
     */
 if (ep_op_has_event(op) && (epds.events & EPOLLEXCLUSIVE)) {
 if (op == EPOLL_CTL_MOD)
 goto error_tgt_fput;
 if (op == EPOLL_CTL_ADD && (is_file_epoll(tf.file) ||
                (epds.events & ~EPOLLEXCLUSIVE_OK_BITS)))
 goto error_tgt_fput;
    }
 

 /*
     * At this point it is safe to assume that the "private_data" contains
     * our own data structure.
     */
    ep = f.file->private_data;

 /*
     * When we insert an epoll file descriptor, inside another epoll file
     * descriptor, there is the change of creating closed loops, which are
     * better be handled here, than in more critical paths. While we are
     * checking for loops we also determine the list of files reachable
     * and hang them on the tfile_check_list, so we can check that we
     * haven't created too many possible wakeup paths.
     *
     * We do not need to take the global 'epumutex' on EPOLL_CTL_ADD when
     * the epoll file descriptor is attaching directly to a wakeup source,
     * unless the epoll file descriptor is nested. The purpose of taking the
     * 'epmutex' on add is to prevent complex toplogies such as loops and
     * deep wakeup paths from forming in parallel through multiple
     * EPOLL_CTL_ADD operations.
     */
    mutex_lock_nested(&ep->mtx, 0);
 if (op == EPOLL_CTL_ADD) {
 if (!list_empty(&f.file->f_ep_links) ||
                        is_file_epoll(tf.file)) {
            full_check = 1;
            mutex_unlock(&ep->mtx);
            mutex_lock(&epmutex);
 if (is_file_epoll(tf.file)) {
 error = -ELOOP;
 if (ep_loop_check(ep, tf.file) != 0) {
                    clear_tfile_check_list();
 goto error_tgt_fput;
                }
            } else
                list_add(&tf.file->f_tfile_llink,
                            &tfile_check_list);
            mutex_lock_nested(&ep->mtx, 0);
 if (is_file_epoll(tf.file)) {
                tep = tf.file->private_data;
                mutex_lock_nested(&tep->mtx, 1);
           }
        }
    }

 /*
     * Try to lookup the file inside our RB tree, Since we grabbed "mtx"
     * above, we can be sure to be able to use the item looked up by
     * ep_find() till we release the mutex.
     */
    epi = ep_find(ep, tf.file, fd);

 error = -EINVAL;
 switch (op) {
 case EPOLL_CTL_ADD:
 if (!epi) {
            epds.events |= POLLERR | POLLHUP;
 error = ep_insert(ep, &epds, tf.file, fd, full_check);
        } else
 error = -EEXIST;
 if (full_check)
            clear_tfile_check_list();
 break;
 case EPOLL_CTL_DEL:
 if (epi)
 error = ep_remove(ep, epi);
 else
 error = -ENOENT;
 break;
 case EPOLL_CTL_MOD:
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
    mutex_unlock(&ep->mtx);

error_tgt_fput:
 if (full_check)
        mutex_unlock(&epmutex);

    fdput(tf);
error_fput:
    fdput(f);
error_return:

 return error;
}
```

根据前面对epoll_ctl接口的介绍，op是对epoll操作的动作（添加/修改/删除事件），ep_op_has_event(op)判断是否不是删除操作，如果op != EPOLL_CTL_DEL为true，则需要调用copy_from_user函数将用户空间传过来的event事件拷贝到内核的epds变量中。因为，只有删除操作，内核不需要使用进程传入的event事件。

接着连续调用两次fdget分别获取epoll文件和被监听文件（以下称为目标文件）的file结构变量（备注：该函数返回fd结构变量，fd结构包含file结构）。

接下来就是对参数的一些检查，出现如下情况，就可以认为传入的参数有问题，直接返回出错：

1. 目标文件不支持poll操作(!tf.file->f_op->poll)；
2. 监听的目标文件就是epoll文件本身(f.file == tf.file)；
3. 用户传入的epoll文件(epfd代表的文件）并不是一个真正的epoll的文件(!is_file_epoll(f.file));
4. 如果操作动作是修改操作，并且事件类型为EPOLLEXCLUSIVE，返回出错等等。

当然下面还有一些关于操作动作如果是添加操作的判断，这里不做解释，比较简单，自行阅读。

在ep里面，维护着一个红黑树，每次添加注册事件时，都会申请一个epitem结构的变量表示事件的监听项，然后插入ep的红黑树里面。在epoll_ctl里面，会调用ep_find函数从ep的红黑树里面查找目标文件表示的监听项，返回的监听项可能为空。

接下来switch这块区域的代码就是整个epoll_ctl函数的核心，对op进行switch出来的有添加(EPOLL_CTL_ADD)、删除(EPOLL_CTL_DEL)和修改(EPOLL_CTL_MOD)三种情况，这里我以添加为例讲解，其他两种情况类似，知道了如何添加监听事件，其他删除和修改监听事件都可以举一反三。

为目标文件添加监控事件时，首先要保证当前ep里面还没有对该目标文件进行监听，如果存在(epi不为空)，就返回-EEXIST错误。否则说明参数正常，然后先默认设置对目标文件的POLLERR和POLLHUP监听事件，然后调用ep_insert函数，将对目标文件的监听事件插入到ep维护的红黑树里面：

sys_epoll_ctl -> ep_insert:

```text
/*
 * Must be called with "mtx" held.
 */
static int ep_insert(struct eventpoll *ep, struct epoll_event *event,
 struct file *tfile, int fd, int full_check)
{
    int error, revents, pwake = 0;
    unsigned long flags;
    long user_watches;
 struct epitem *epi;
 struct ep_pqueue epq;

    user_watches = atomic_long_read(&ep->user->epoll_watches);
 if (unlikely(user_watches >= max_user_watches))
 return -ENOSPC;
 if (!(epi = kmem_cache_alloc(epi_cache, GFP_KERNEL)))
 return -ENOMEM;
 

 /* Item initialization follow here ... */
    INIT_LIST_HEAD(&epi->rdllink);
    INIT_LIST_HEAD(&epi->fllink);
    INIT_LIST_HEAD(&epi->pwqlist);
    epi->ep = ep;
    ep_set_ffd(&epi->ffd, tfile, fd);
    epi->event = *event;
    epi->nwait = 0;
    epi->next = EP_UNACTIVE_PTR;
 if (epi->event.events & EPOLLWAKEUP) {
        error = ep_create_wakeup_source(epi);
 if (error)
            goto error_create_wakeup_source;
    } else {
        RCU_INIT_POINTER(epi->ws, NULL);
    }

 /* Initialize the poll table using the queue callback */
    epq.epi = epi;
    init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);

 /*
     * Attach the item to the poll hooks and get current event bits.
     * We can safely use the file* here because its usage count has
     * been increased by the caller of this function. Note that after
     * this operation completes, the poll callback can start hitting
     * the new item.
     */
    revents = ep_item_poll(epi, &epq.pt);

 /*
     * We have to check if something went wrong during the poll wait queue
     * install process. Namely an allocation for a wait queue failed due
     * high memory pressure.
     */
    error = -ENOMEM;
 if (epi->nwait < 0)
        goto error_unregister;

 /* Add the current item to the list of active epoll hook for this file */
    spin_lock(&tfile->f_lock);
    list_add_tail_rcu(&epi->fllink, &tfile->f_ep_links);
    spin_unlock(&tfile->f_lock);

 /*
     * Add the current item to the RB tree. All RB tree operations are
     * protected by "mtx", and ep_insert() is called with "mtx" held.
     */
    ep_rbtree_insert(ep, epi);

 /* now check if we've created too many backpaths */
    error = -EINVAL;
 if (full_check && reverse_path_check())
        goto error_remove_epi;

 /* We have to drop the new item inside our item list to keep track of it */
    spin_lock_irqsave(&ep->lock, flags);

 /* record NAPI ID of new item if present */
    ep_set_busy_poll_napi_id(epi);

 /* If the file is already "ready" we drop it inside the ready list */
 if ((revents & event->events) && !ep_is_linked(&epi->rdllink)) {
        list_add_tail(&epi->rdllink, &ep->rdllist);
        ep_pm_stay_awake(epi);

 /* Notify waiting tasks that events are available */
 if (waitqueue_active(&ep->wq))
            wake_up_locked(&ep->wq);
 if (waitqueue_active(&ep->poll_wait))
            pwake++;
    }

    spin_unlock_irqrestore(&ep->lock, flags);

    atomic_long_inc(&ep->user->epoll_watches);

 /* We have to call this outside the lock */
 if (pwake)
        ep_poll_safewake(&ep->poll_wait);

 return 0;

error_remove_epi:
    spin_lock(&tfile->f_lock);
    list_del_rcu(&epi->fllink);
    spin_unlock(&tfile->f_lock);
    rb_erase(&epi->rbn, &ep->rbr);

error_unregister:
    ep_unregister_pollwait(ep, epi);

 /*
     * We need to do this because an event could have been arrived on some
     * allocated wait queue. Note that we don't care about the ep->ovflist
     * list, since that is used/cleaned only inside a section bound by "mtx".
     * And ep_insert() is called with "mtx" held.
     */
    spin_lock_irqsave(&ep->lock, flags);
 if (ep_is_linked(&epi->rdllink))
        list_del_init(&epi->rdllink);
    spin_unlock_irqrestore(&ep->lock, flags);

    wakeup_source_unregister(ep_wakeup_source(epi));

error_create_wakeup_source:
    kmem_cache_free(epi_cache, epi);

 return error;
}
```

前面说过，对目标文件的监听是由一个epitem结构的监听项变量维护的，所以在ep_insert函数里面，首先调用kmem_cache_alloc函数，从slab分配器里面分配一个epitem结构监听项，然后对该结构进行初始化，这里也没有什么好说的。我们接下来看ep_item_poll这个函数调用：

sys_epoll_ctl -> ep_insert -> ep_item_poll:

```text
static inline unsigned int ep_item_poll(struct epitem *epi, poll_table *pt)
{
    pt->_key = epi->event.events;

 return epi->ffd.file->f_op->poll(epi->ffd.file, pt) & epi->event.events;
}
```

ep_item_poll函数里面，调用目标文件的poll函数，这个函数针对不同的目标文件而指向不同的函数，如果目标文件为套接字的话，这个poll就指向sock_poll，而如果目标文件为tcp套接字来说，这个poll就是tcp_poll函数。虽然poll指向的函数可能会不同，但是其作用都是一样的，就是获取目标文件当前产生的事件位，并且将监听项绑定到目标文件的poll钩子里面（最重要的是注册ep_ptable_queue_proc这个poll callback回调函数），这步操作完成后，以后目标文件产生事件就会调用ep_ptable_queue_proc回调函数。

接下来，调用list_add_tail_rcu将当前监听项添加到目标文件的f_ep_links链表里面，该链表是目标文件的epoll钩子链表，所有对该目标文件进行监听的监听项都会加入到该链表里面。

然后就是调用ep_rbtree_insert，将epi监听项添加到ep维护的红黑树里面,这里不做解释，代码如下：

sys_epoll_ctl -> ep_insert -> ep_rbtree_insert:

```text
static void ep_rbtree_insert(struct eventpoll *ep, struct epitem *epi)
{
    int kcmp;
 struct rb_node **p = &ep->rbr.rb_node, *parent = NULL;
 struct epitem *epic;

 while (*p) {
        parent = *p;
        epic = rb_entry(parent, struct epitem, rbn);
        kcmp = ep_cmp_ffd(&epi->ffd, &epic->ffd);
 if (kcmp > 0)
            p = &parent->rb_right;
 else
            p = &parent->rb_left;
    }
    rb_link_node(&epi->rbn, parent, p);
    rb_insert_color(&epi->rbn, &ep->rbr);
}
```

前面提到，ep_insert有调用ep_item_poll去获取目标文件产生的事件位，在调用epoll_ctl前这段时间，可能会产生相关进程需要监听的事件，如果有监听的事件产生，(revents & event->events 为 true)，并且目标文件相关的监听项没有链接到ep的准备链表rdlist里面的话，就将该监听项添加到ep的rdlist准备链表里面，rdlist链接的是该epoll描述符监听的所有已经就绪的目标文件的监听项。并且，如果有任务在等待产生事件时，就调用wake_up_locked函数唤醒所有正在等待的任务，处理相应的事件。当进程调用epoll_wait时，该进程就出现在ep的wq等待队列里面。接下来讲解epoll_wait函数。

总结epoll_ctl函数：该函数根据监听的事件，为目标文件申请一个监听项，并将该监听项挂人到eventpoll结构的红黑树里面。

### 2.3 epoll_wait

epoll_wait等待事件的产生，内核代码如下：

sys_epoll_wait:

```text
/*
 * Implement the event wait interface for the eventpoll file. It is the kernel
 * part of the user space epoll_wait(2).
 */
SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events,
 int, maxevents, int, timeout)
{
 int error;
 struct fd f;
 struct eventpoll *ep;

 /* The maximum number of event must be greater than zero */
 if (maxevents <= 0 || maxevents > EP_MAX_EVENTS)
 return -EINVAL;

 /* Verify that the area passed by the user is writeable */
 if (!access_ok(VERIFY_WRITE, events, maxevents * sizeof(struct epoll_event)))
 return -EFAULT;

 /* Get the "struct file *" for the eventpoll file */
    f = fdget(epfd);
 if (!f.file)
 return -EBADF;

 /*
     * We have to check that the file structure underneath the fd
     * the user passed to us _is_ an eventpoll file.
     */
    error = -EINVAL;
 if (!is_file_epoll(f.file))
 goto error_fput;
 

 /*
     * At this point it is safe to assume that the "private_data" contains
     * our own data structure.
     */
    ep = f.file->private_data;

 /* Time to fish for events ... */
    error = ep_poll(ep, events, maxevents, timeout);

error_fput:
    fdput(f);
 return error;
}
```

首先是对进程传进来的一些参数的检查：

- maxevents必须大于0并且小于EP_MAX_EVENTS，否则就返回-EINVAL；
- 内核必须有对events变量写文件的权限，否则返回-EFAULT；
- epfd代表的文件必须是个真正的epoll文件，否则返回-EBADF。

参数全部检查合格后，接下来就调用ep_poll函数进行真正的处理：

sys_epoll_wait -> ep_poll:

```text
/**
 * ep_poll - Retrieves ready events, and delivers them to the caller supplied
 *           event buffer.
 *
 * @ep: Pointer to the eventpoll context.
 * @events: Pointer to the userspace buffer where the ready events should be
 *          stored.
 * @maxevents: Size (in terms of number of events) of the caller event buffer.
 * @timeout: Maximum timeout for the ready events fetch operation, in
 *           milliseconds. If the @timeout is zero, the function will not block,
 *           while if the @timeout is less than zero, the function will block
 *           until at least one event has been retrieved (or an error
 *           occurred).
 *
 * Returns: Returns the number of ready events which have been fetched, or an
 *          error code, in case of error.
 */
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
 int maxevents, long timeout)
{
 int res = 0, eavail, timed_out = 0;
    unsigned long flags;
    u64 slack = 0;
    wait_queue_t wait;
    ktime_t expires, *to = NULL;

 if (timeout > 0) {
        struct timespec64 end_time = ep_set_mstimeout(timeout);

        slack = select_estimate_accuracy(&end_time);
        to = &expires;
        *to = timespec64_to_ktime(end_time);
    } else if (timeout == 0) {
 /*
         * Avoid the unnecessary trip to the wait queue loop, if the
         * caller specified a non blocking operation.
         */
        timed_out = 1;
        spin_lock_irqsave(&ep->lock, flags);
 goto check_events;
    }

fetch_events:

 if (!ep_events_available(ep))
        ep_busy_loop(ep, timed_out);

    spin_lock_irqsave(&ep->lock, flags); 

 if (!ep_events_available(ep)) {
 /*
         * Busy poll timed out.  Drop NAPI ID for now, we can add
         * it back in when we have moved a socket with a valid NAPI
         * ID onto the ready list.
         */
        ep_reset_busy_poll_napi_id(ep);

 /*
         * We don't have any available event to return to the caller.
         * We need to sleep here, and we will be wake up by
         * ep_poll_callback() when events will become available.
         */
        init_waitqueue_entry(&wait, current);
        __add_wait_queue_exclusive(&ep->wq, &wait); 

 for (;;) {
 /*
             * We don't want to sleep if the ep_poll_callback() sends us
             * a wakeup in between. That's why we set the task state
             * to TASK_INTERRUPTIBLE before doing the checks.
             */
            set_current_state(TASK_INTERRUPTIBLE);
 if (ep_events_available(ep) || timed_out)
 break;
 if (signal_pending(current)) {
                res = -EINTR;
 break;
            }

            spin_unlock_irqrestore(&ep->lock, flags);
 if (!schedule_hrtimeout_range(to, slack, HRTIMER_MODE_ABS))
                timed_out = 1;

            spin_lock_irqsave(&ep->lock, flags);
        }

        __remove_wait_queue(&ep->wq, &wait);
        __set_current_state(TASK_RUNNING);
    }
check_events:
 /* Is it worth to try to dig for events ? */
    eavail = ep_events_available(ep);

    spin_unlock_irqrestore(&ep->lock, flags);

 /*
     * Try to transfer events to user space. In case we get 0 events and
     * there's still timeout left over, we go trying again in search of
     * more luck.
     */
 if (!res && eavail &&
        !(res = ep_send_events(ep, events, maxevents)) && !timed_out)
 goto fetch_events;

 return res;
}
```

ep_poll中首先是对等待时间的处理，timeout超时时间以ms为单位，timeout大于0，说明等待timeout时间后超时，如果timeout等于0，函数不阻塞，直接返回，小于0的情况，是永久阻塞，直到有事件产生才返回。

当没有事件产生时（(!ep_events_available(ep))为true）,调用__add_wait_queue_exclusive函数将当前进程加入到ep->wq等待队列里面，然后在一个无限for循环里面，首先调用set_current_state(TASK_INTERRUPTIBLE)，将当前进程设置为可中断的睡眠状态，然后当前进程就让出cpu，进入睡眠，直到有其他进程调用wake_up或者有中断信号进来唤醒本进程，它才会去执行接下来的代码。

如果进程被唤醒后，首先检查是否有事件产生，或者是否出现超时还是被其他信号唤醒的。如果出现这些情况，就跳出循环，将当前进程从ep->wp的等待队列里面移除，并且将当前进程设置为TASK_RUNNING就绪状态。

如果真的有事件产生，就调用ep_send_events函数，将events事件转移到用户空间里面。

sys_epoll_wait -> ep_poll -> ep_send_events:

```text
static int ep_send_events(struct eventpoll *ep,
 struct epoll_event __user *events, int maxevents)
{
 struct ep_send_events_data esed; 

    esed.maxevents = maxevents;
    esed.events = events;

 return ep_scan_ready_list(ep, ep_send_events_proc, &esed, 0, false);
}
```

ep_send_events没有什么工作，真正的工作是在ep_scan_ready_list函数里面：

sys_epoll_wait -> ep_poll -> ep_send_events -> ep_scan_ready_list:

```text
/**
 * ep_scan_ready_list - Scans the ready list in a way that makes possible for
 *                      the scan code, to call f_op->poll(). Also allows for
 *                      O(NumReady) performance.
 *
 * @ep: Pointer to the epoll private data structure.
 * @sproc: Pointer to the scan callback.
 * @priv: Private opaque data passed to the @sproc callback.
 * @depth: The current depth of recursive f_op->poll calls.
 * @ep_locked: caller already holds ep->mtx
 *
 * Returns: The same integer error code returned by the @sproc callback.
 */
static int ep_scan_ready_list(struct eventpoll *ep,
 int (*sproc)(struct eventpoll *,
                       struct list_head *, void *),
 void *priv, int depth, bool ep_locked)
{
 int error, pwake = 0;
    unsigned long flags;
    struct epitem *epi, *nepi;
    LIST_HEAD(txlist);

 /*
    * We need to lock this because we could be hit by
     * eventpoll_release_file() and epoll_ctl().
     */

 if (!ep_locked)
        mutex_lock_nested(&ep->mtx, depth);

 /*
     * Steal the ready list, and re-init the original one to the
     * empty list. Also, set ep->ovflist to NULL so that events
     * happening while looping w/out locks, are not lost. We cannot
     * have the poll callback to queue directly on ep->rdllist,
     * because we want the "sproc" callback to be able to do it
     * in a lockless way.
     */
    spin_lock_irqsave(&ep->lock, flags);
    list_splice_init(&ep->rdllist, &txlist);
    ep->ovflist = NULL;
    spin_unlock_irqrestore(&ep->lock, flags);

 /*
     * Now call the callback function.
     */
 error = (*sproc)(ep, &txlist, priv);

    spin_lock_irqsave(&ep->lock, flags);
 /*
     * During the time we spent inside the "sproc" callback, some
     * other events might have been queued by the poll callback.
     * We re-insert them inside the main ready-list here.
     */
 for (nepi = ep->ovflist; (epi = nepi) != NULL;
         nepi = epi->next, epi->next = EP_UNACTIVE_PTR) {
 /*
         * We need to check if the item is already in the list.
         * During the "sproc" callback execution time, items are
         * queued into ->ovflist but the "txlist" might already
         * contain them, and the list_splice() below takes care of them.
         */
 if (!ep_is_linked(&epi->rdllink)) {
            list_add_tail(&epi->rdllink, &ep->rdllist);
            ep_pm_stay_awake(epi);
        }
    }
 /*
     * We need to set back ep->ovflist to EP_UNACTIVE_PTR, so that after
     * releasing the lock, events will be queued in the normal way inside
     * ep->rdllist.
     */
    ep->ovflist = EP_UNACTIVE_PTR;

 /*
     * Quickly re-inject items left on "txlist".
     */
    list_splice(&txlist, &ep->rdllist);
    __pm_relax(ep->ws);

 if (!list_empty(&ep->rdllist)) {
 /*
         * Wake up (if active) both the eventpoll wait list and
         * the ->poll() wait list (delayed after we release the lock).
         */
 if (waitqueue_active(&ep->wq))
            wake_up_locked(&ep->wq);
 if (waitqueue_active(&ep->poll_wait))
            pwake++;
    }
    spin_unlock_irqrestore(&ep->lock, flags);

 if (!ep_locked)
        mutex_unlock(&ep->mtx); 

 /* We have to call this outside the lock */
 if (pwake)
        ep_poll_safewake(&ep->poll_wait);

 return error;
}
```

ep_scan_ready_list首先将ep就绪链表里面的数据链接到一个全局的txlist里面，然后清空ep的就绪链表，同时还将ep的ovflist链表设置为NULL，ovflist是用单链表，是一个接受就绪事件的备份链表，当内核进程将事件从内核拷贝到用户空间时，这段时间目标文件可能会产生新的事件，这个时候，就需要将新的时间链入到ovlist里面。

仅接着，调用sproc回调函数(这里将调用ep_send_events_proc函数)将事件数据从内核拷贝到用户空间。

sys_epoll_wait -> ep_poll -> ep_send_events -> ep_scan_ready_list -> ep_send_events_proc:

```text
static int ep_send_events_proc(struct eventpoll *ep, struct list_head *head,
                   void *priv)
{
 struct ep_send_events_data *esed = priv;
    int eventcnt;
    unsigned int revents;
 struct epitem *epi;
 struct epoll_event __user *uevent;
 struct wakeup_source *ws;
    poll_table pt;

    init_poll_funcptr(&pt, NULL);

 /*
     * We can loop without lock because we are passed a task private list.
     * Items cannot vanish during the loop because ep_scan_ready_list() is
     * holding "mtx" during this call.
     */
 for (eventcnt = 0, uevent = esed->events;
         !list_empty(head) && eventcnt < esed->maxevents;) {
        epi = list_first_entry(head, struct epitem, rdllink);

 /*
         * Activate ep->ws before deactivating epi->ws to prevent
         * triggering auto-suspend here (in case we reactive epi->ws
         * below).
         *
         * This could be rearranged to delay the deactivation of epi->ws
         * instead, but then epi->ws would temporarily be out of sync
         * with ep_is_linked().
         */
        ws = ep_wakeup_source(epi);
 if (ws) {
 if (ws->active)
                __pm_stay_awake(ep->ws);
            __pm_relax(ws);
        }
        list_del_init(&epi->rdllink);

        revents = ep_item_poll(epi, &pt);

 /*
         * If the event mask intersect the caller-requested one,
         * deliver the event to userspace. Again, ep_scan_ready_list()
         * is holding "mtx", so no operations coming from userspace
         * can change the item.
         */
 if (revents) {
 if (__put_user(revents, &uevent->events) ||
                __put_user(epi->event.data, &uevent->data)) {
               list_add(&epi->rdllink, head);
                ep_pm_stay_awake(epi);
 return eventcnt ? eventcnt : -EFAULT;
            }
            eventcnt++;
            uevent++;
 if (epi->event.events & EPOLLONESHOT)
                epi->event.events &= EP_PRIVATE_BITS;
 else if (!(epi->event.events & EPOLLET)) {
 /*
                 * If this file has been added with Level
                 * Trigger mode, we need to insert back inside
                 * the ready list, so that the next call to
                 * epoll_wait() will check again the events
                 * availability. At this point, no one can insert
                 * into ep->rdllist besides us. The epoll_ctl()
                 * callers are locked out by
                 * ep_scan_ready_list() holding "mtx" and the
                 * poll callback will queue them in ep->ovflist.
                 */
                list_add_tail(&epi->rdllink, &ep->rdllist);
                ep_pm_stay_awake(epi);
           }
        }
    }
 return eventcnt;
}
```

ep_send_events_proc回调函数循环获取监听项的事件数据，对每个监听项，调用ep_item_poll获取监听到的目标文件的事件，如果获取到事件，就调用__put_user函数将数据拷贝到用户空间。

回到ep_scan_ready_list函数，上面说到，在sproc回调函数执行期间，目标文件可能会产生新的事件链入ovlist链表里面，所以，在回调结束后，需要重新将ovlist链表里面的事件添加到rdllist就绪事件链表里面。

同时在最后，如果rdlist不为空（表示是否有就绪事件），并且由进程等待该事件，就调用wake_up_locked再一次唤醒内核进程处理事件的到达（流程跟前面一样，也就是将事件拷贝到用户空间）。

到这，epoll_wait的流程是结束了，但是有一个问题，就是前面提到的进程调用epoll_wait后会睡眠，但是这个进程什么时候被唤醒呢？在调用epoll_ctl为目标文件注册监听项时，对目标文件的监听项注册一个ep_ptable_queue_proc回调函数，ep_ptable_queue_proc回调函数将进程添加到目标文件的wakeup链表里面，并且注册ep_poll_callbak回调，当目标文件产生事件时，ep_poll_callbak回调就去唤醒等待队列里面的进程。

总结一下epoll该函数： epoll_wait函数会使调用它的进程进入睡眠（timeout为0时除外），如果有监听的事件产生，该进程就被唤醒，同时将事件从内核里面拷贝到用户空间返回给该进程。

原文地址：https://zhuanlan.zhihu.com/p/451235847

作者：linux