# 【NO.412】从linux内核出发彻底弄懂socket底层的来龙去脉

## **1.socket与inode**

socket在Linux中对应的文件系统叫Sockfs,每创建一个socket,就在sockfs中创建了一个特殊的文件，同时创建了sockfs文件系统中的inode，该inode唯一标识当前socket的通信。

如下图所示，左侧窗口使用nc工具创建一个TCP连接；右侧找到该进程id(3384)，通过查看该进程下的描述符，可以看到"3 ->socket:[86851]",socket表示这是一个socket类型的fd，[86851]表示这个一个inode号，能够唯一标识当前的这个socket通信连接，进一步在该inode下查看"grep -i "86851" /proc/net/tcp”可以看到该TCP连接的所有信息(连接状态、IP地址等)，只不过是16进制显示。

![img](https://pic1.zhimg.com/80/v2-ff2f1d386cad6573dee7df7df11e46d8_720w.webp)

**在分析socket与inode之前，先通过ext4文件系统举例：**

在VFS层，即抽象层，所有的文件系统都使用struct inode结构体描述indoe，然而分配inode的方式都不同，如ext4文件系统的分配inode函数是ext4_alloc_inode，如下所示：

```text
static struct inode *ext4_alloc_inode(struct super_block *sb)
{
 struct ext4_inode_info *ei;

 ei = kmem_cache_alloc(ext4_inode_cachep, GFP_NOFS);
 if (!ei)
  return NULL;

 ei->vfs_inode.i_version = 1;
 spin_lock_init(&ei->i_raw_lock);
 INIT_LIST_HEAD(&ei->i_prealloc_list);
 spin_lock_init(&ei->i_prealloc_lock);
 ext4_es_init_tree(&ei->i_es_tree);
 rwlock_init(&ei->i_es_lock);
 INIT_LIST_HEAD(&ei->i_es_list);
 ei->i_es_all_nr = 0;
 ei->i_es_shk_nr = 0;
 ei->i_es_shrink_lblk = 0;
 ei->i_reserved_data_blocks = 0;
 ei->i_da_metadata_calc_len = 0;
 ei->i_da_metadata_calc_last_lblock = 0;
 spin_lock_init(&(ei->i_block_reservation_lock));
#ifdef CONFIG_QUOTA
 ei->i_reserved_quota = 0;
 memset(&ei->i_dquot, 0, sizeof(ei->i_dquot));
#endif
 ei->jinode = NULL;
 INIT_LIST_HEAD(&ei->i_rsv_conversion_list);
 spin_lock_init(&ei->i_completed_io_lock);
 ei->i_sync_tid = 0;
 ei->i_datasync_tid = 0;
 atomic_set(&ei->i_unwritten, 0);
 INIT_WORK(&ei->i_rsv_conversion_work, ext4_end_io_rsv_work);
 return &ei->vfs_inode;
}
```

从函数中可以看出来，函数其实是调用kmem_cache_alloc分配了 ext4_inode_info结构体（结构体如下所示），然后进行了一系列的初始化，最后返回的却是struct inode结构体（如上面代码的return &ei->vfs_inode）。如下结构体ext4_inode_info(ei)所示，vfs_inode是其struct inode结构体成员。

```text
struct ext4_inode_info {
 __le32 i_data[15]; /* unconverted */
 __u32 i_dtime;
 ext4_fsblk_t i_file_acl;

 ......
 struct rw_semaphore i_data_sem;
 struct rw_semaphore i_mmap_sem;
 struct inode vfs_inode;
 struct jbd2_inode *jinode;
  ......
};
```

![img](https://pic1.zhimg.com/80/v2-45d7b2caae1170cee18e1c1bf38f6a80_720w.webp)

再看一下：ext4_inode、ext4_inode_info、inode之间的关联，

ext4_inode如下所示，是磁盘上inode的结构

```text
struct ext4_inode {
 __le16 i_mode;  /* File mode */
 __le16 i_uid;  /* Low 16 bits of Owner Uid */
 __le32 i_size_lo; /* Size in bytes */
 __le32 i_atime; /* Access time */
 __le32 i_ctime; /* Inode Change time */
 __le32 i_mtime; /* Modification time */
 __le32 i_dtime; /* Deletion Time */
 __le16 i_gid;  /* Low 16 bits of Group Id */
 __le16 i_links_count; /* Links count */
 __le32 i_blocks_lo; /* Blocks count */
 __le32 i_flags; /* File flags */
 ......
}
```

ext4_inode_info是ext4文件系统的inode在内存中管理结构体：

```text
struct ext4_inode_info {
 __le32 i_data[15]; /* unconverted */
 __u32 i_dtime;
 ext4_fsblk_t i_file_acl;
 ......
};
```

inode是文件系统抽象层：

```text
struct inode {
    umode_t                 i_mode;
    unsigned short          i_opflags;
    kuid_t                  i_uid;
    kgid_t                  i_gid;
    unsigned int            i_flags;
 
    /* 对inode操作的具体方法
     * 不同的文件系统会注册不同的函数方法即可
     */
    const struct inode_operations   *i_op;
    struct super_block      *i_sb;
    struct address_space    *i_mapping;
 
    unsigned long           i_ino;
    
    union {
        const unsigned int i_nlink;
        unsigned int __i_nlink;
    };
    dev_t                   i_rdev;
    /* 文件大小 */
    loff_t                  i_size;
    /* 文件最后访问时间 */
    struct timespec         i_atime;
    /* 文件最后修改时间 */
    struct timespec         i_mtime;
    /* 文件创建时间 */
    struct timespec         i_ctime;
    spinlock_t              i_lock; /* i_blocks, i_bytes, maybe i_size */
    unsigned short          i_bytes;
    unsigned int            i_blkbits;
    enum rw_hint            i_write_hint;
    blkcnt_t                i_blocks;

    /* Misc */
    unsigned long           i_state;
    struct rw_semaphore     i_rwsem;
    unsigned long           dirtied_when;   /* jiffies of first dirtying */
    unsigned long           dirtied_time_when;
 
    /* inode通过以下结构被加入到的各种链表 */
    struct hlist_node       i_hash;
    struct list_head        i_io_list;      /* backing dev IO list */
 
    struct list_head        i_lru;          /* inode LRU list */
    struct list_head        i_sb_list;
    struct list_head        i_wb_list;      /* backing dev writeback list */
    union {
        struct hlist_head       i_dentry;
        struct rcu_head         i_rcu;
    };
    atomic64_t              i_version;
    atomic_t                i_count;
    atomic_t                i_dio_count;
    atomic_t                i_writecount;
 
    /* 对文件操作(如文件读写等)的具体方法
     * 实现虚拟文件系统的核心结构
     * 不同的文件系统只需要注册不同的函数方法即可
     */
    const struct file_operations    *i_fop; /* former ->i_op->default_file_ops */
    struct file_lock_context        *i_flctx;
    struct address_space    i_data;
    struct list_head        i_devices;
    union {
        struct pipe_inode_info  *i_pipe;
        struct block_device     *i_bdev;
        struct cdev             *i_cdev;
        char                    *i_link;
        unsigned                i_dir_seq;
    };
    __u32                   i_generation;

    void                    *i_private; /* fs or device private pointer */
} __randomize_layout;
```

三者的关系如下图，struct inode是VFS抽象层的表示，ext4_inode_info是ext4文件系统inode在内存中的表示，struct ext4_inode是文件系统inode在磁盘中的表示。

![img](https://pic1.zhimg.com/80/v2-1f9934c8b31e190e4b7dfd17f8c3a2b4_720w.webp)

**VFS采用C语言的方式实现了struct inode和struct ext4_inode_info继承关系，inode与ext4_inode_info是父类与子类的关系**，并且Linux内核实现了inode与ext4_inode_info父子类的互相转换，如下EXT4_I所示：

```text
static inline struct ext4_inode_info *EXT4_I(struct inode *inode)
{
 return container_of(inode, struct ext4_inode_info, vfs_inode);
}
```

**以上是以ext4为例进行了分析，下面将开始从socket与inode进行分析：**

sockfs是虚拟文件系统，所以在磁盘上不存在inode的表示，在内核中有struct socket_alloc来表示内存中sockfs文件系统inode的相关结构体：

```text
struct socket_alloc {
 struct socket socket;
 struct inode vfs_inode;
};
```

**struct socket与struct inode的关系如下图，正如ext4文件系统中struct ext4_inode_info与struct inode的关系类似，inode和socket_alloc结构体是父类与子类的关系。**

![img](https://pic1.zhimg.com/80/v2-624c904338f5ee441997bdcac8233458_720w.webp)

从上面分析ext4文件系统分配inode时，是通过ext4_alloc_inode函数分配了ext4_inode_info结构体，并初始化结构体成员，函数最后返回的是ext4_inode_info中的struct inode成员。sockfs文件系统也类似，sockfs文件系统分配inode时，创建的是socket_alloc结构体，在函数最后返回的是struct inode。

如下所示alloc_inode是分配inode结构体的回调函数接口。

```text
static const struct super_operations sockfs_ops = {
 .alloc_inode = sock_alloc_inode,
 .destroy_inode = sock_destroy_inode,
 .statfs  = simple_statfs,
}
```

sockfs文件系统的inode分配函数是sock_alloc_inode，如下所示：

```text
static struct inode *sock_alloc_inode(struct super_block *sb)
{
 struct socket_alloc *ei;
 struct socket_wq *wq;

 ei = kmem_cache_alloc(sock_inode_cachep, GFP_KERNEL);
 if (!ei)
  return NULL;
 wq = kmalloc(sizeof(*wq), GFP_KERNEL);
 if (!wq) {
  kmem_cache_free(sock_inode_cachep, ei);
  return NULL;
 }
 init_waitqueue_head(&wq->wait);
 wq->fasync_list = NULL;
 wq->flags = 0;
 RCU_INIT_POINTER(ei->socket.wq, wq);

 ei->socket.state = SS_UNCONNECTED;
 ei->socket.flags = 0;
 ei->socket.ops = NULL;
 ei->socket.sk = NULL;
 ei->socket.file = NULL;

 return &ei->vfs_inode;
}
```

sock_alloc_inode函数分配了socket_alloc结构体，也就意味着分配了struct socket和struct inode,并最终返回了socket_alloc结构体成员inode。

故struct socket这个字段出生的时候其实就和一个struct inode结构体伴生出来的，它们俩共同封装在struct socket_alloc中，由sockfs的sock_alloc_inode函数分配的,函数返回的是struct inode结构体.和ext4文件系统类型类似。**sockfs文件系统也实现了struct inode与struct socket的转换:**

```text
static inline struct socket *SOCKET_I(struct inode *inode)
{
 return &container_of(inode, struct socket_alloc, vfs_inode)->socket;
}
```

## **2.socket的创建与初始化**

首先看一下struct socket在内核中的定义：

```text
struct socket {
 socket_state  state;//socket状态

 short   type; //socket类型

 unsigned long  flags;//socket的标志位

 struct socket_wq __rcu *wq;

 struct file  *file;//与socket关联的文件指针
 struct sock  *sk;//套接字的核心，面向底层网络具体协议
 const struct proto_ops *ops;//socket函数操作集
};
```

**在内核中还有struct sock结构体，在struct socket中可以看到那么它们的关系是什么？**

1、socket面向上层，sock面向下层的具体协议

2、socket是内核抽象出的一个通用结构体，主要是设置了一些跟fs相关的字段，而真正跟网络通信相关的字段结构体是struct sock

3、struct sock是套接字的核心，是对底层具体协议做的一层抽象封装，比如TCP协议，struct sock结构体中的成员sk_prot会赋值为tcp_prot,UDP协议会赋值为udp_prot。

(关于更多struct sock的分析将在以后的文章中分析)

**创建socket的系统调用：**在用户空间创建了一个socket后，返回值是一个文件描述符。在SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)最后调用sock_map_fd进行关联，其中返回的就是用户空间获取的文件描述符fd，sock就是调用sock_create创建成功的socket.

```text
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
{
 int retval;
 struct socket *sock;
 int flags;

 /* Check the SOCK_* constants for consistency.  */
 BUILD_BUG_ON(SOCK_CLOEXEC != O_CLOEXEC);
 BUILD_BUG_ON((SOCK_MAX | SOCK_TYPE_MASK) != SOCK_TYPE_MASK);
 BUILD_BUG_ON(SOCK_CLOEXEC & SOCK_TYPE_MASK);
 BUILD_BUG_ON(SOCK_NONBLOCK & SOCK_TYPE_MASK);

 flags = type & ~SOCK_TYPE_MASK;
 if (flags & ~(SOCK_CLOEXEC | SOCK_NONBLOCK))
  return -EINVAL;
 type &= SOCK_TYPE_MASK;

 if (SOCK_NONBLOCK != O_NONBLOCK && (flags & SOCK_NONBLOCK))
  flags = (flags & ~SOCK_NONBLOCK) | O_NONBLOCK;

 retval = sock_create(family, type, protocol, &sock);
 if (retval < 0)
  return retval;

 return sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK));
}
```

socket的创建将调用sock_create函数：

```text
int sock_create(int family, int type, int protocol, struct socket **res)
{
 return __sock_create(current->nsproxy->net_ns, family, type, protocol, res, 0);
}
```

__sock_create函数调用sock_alloc函数分配socket结构和文件节点：

```text
int __sock_create(struct net *net, int family, int type, int protocol,
    struct socket **res, int kern)
{
 int err;
 struct socket *sock;
 const struct net_proto_family *pf;
  //检查family的字段范围
 if (family < 0 || family >= NPROTO)
  return -EAFNOSUPPORT;
 if (type < 0 || type >= SOCK_MAX)
  return -EINVAL;

 ......
 sock = sock_alloc();//分配socket和inode，返回sock
 if (!sock) {
  net_warn_ratelimited("socket: no more sockets\n");
  return -ENFILE; /* Not exactly a match, but its the
       closest posix thing */
 }

 sock->type = type;
  ......
 rcu_read_lock();
 pf = rcu_dereference(net_families[family]);//获取协议族family对应的操作表
 err = -EAFNOSUPPORT;
 if (!pf)
  goto out_release;

 if (!try_module_get(pf->owner))
  goto out_release;

 /* Now protected by module ref count */
 rcu_read_unlock();

 err = pf->create(net, sock, protocol, kern);//调用family协议族的socket创建函数
 if (err < 0)
  goto out_module_put;


 if (!try_module_get(sock->ops->owner))
  goto out_module_busy;

 ......
}
```

socket结构体的创建在sock_alloc()函数中：

```text
struct socket *sock_alloc(void)
{
 struct inode *inode;
 struct socket *sock;

 inode = new_inode_pseudo(sock_mnt->mnt_sb);
 if (!inode)
  return NULL;

 sock = SOCKET_I(inode);

 inode->i_ino = get_next_ino();
 inode->i_mode = S_IFSOCK | S_IRWXUGO;
 inode->i_uid = current_fsuid();
 inode->i_gid = current_fsgid();
 inode->i_op = &sockfs_inode_ops;

 this_cpu_add(sockets_in_use, 1);
 return sock;
}
```

new_inode_pseudo中通过继续调用sockfs文件系统中的sock_alloc_inode函数完成struct socket_alloc的创建并返回其结构体成员struct inode。

然后调用SOCKT_I函数返回对应的struct socket。

在_sock_create中：pf->create(net, sock, protocol, kern);

通过相应的协议族，进一步调用不同的socket创建函数。pf是struct net_proto_family结构体，如下所示：

```text
struct net_proto_family {
 int  family;
 int  (*create)(struct net *net, struct socket *sock,
      int protocol, int kern);
 struct module *owner;
};
```

net_families[]数组里存放的是各个协议族的信息，以family字段作为下标，对应的值为net_pro_family结构体。此处我们针对TCP协议分析，因此我们family字段是AF_INET，pf->create将调用inet_create函数继续完成底层struct sock等创建和初始化。

inet_create函数完成struct socket、struct inode、struct sock的创建与初始化后，**调用sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK));完成socket与文件系统的关联，负责分配文件，并与socket进行绑定：**

1、调用sock_alloc_file，分配一个struct file，并将私有数据指针指向socket结构

2、fd_install 对应文件描述符和file

```text
static int sock_map_fd(struct socket *sock, int flags)
{
 struct file *newfile;
 int fd = get_unused_fd_flags(flags);//为socket分配文件号和文件结构
 if (unlikely(fd < 0)) {
  sock_release(sock);
  return fd;
 }

 newfile = sock_alloc_file(sock, flags, NULL);//分配file对象
 if (likely(!IS_ERR(newfile))) {
  fd_install(fd, newfile);//使文件号与文件结构挂钩
  return fd;
 }

 put_unused_fd(fd);
 return PTR_ERR(newfile);
}
```

get_unused_fd_flags(flags)继续调用alloc_fd完成文件描述符的分配。

sock_alloc_file(sock, flags, NULL)分配一个struct file结构体

```text
struct file *sock_alloc_file(struct socket *sock, int flags, const char *dname)
{
 ......
 file = alloc_file(&path, FMODE_READ | FMODE_WRITE,
    &socket_file_ops);//分配struct file结构体
 if (IS_ERR(file)) {
  /* drop dentry, keep inode for a bit */
  ihold(d_inode(path.dentry));
  path_put(&path);
  /* ... and now kill it properly */
  sock_release(sock);
  return file;
 }

 sock->file = file; //socket通过其file字段进行关联
 file->f_flags = O_RDWR | (flags & O_NONBLOCK);
 file->private_data = sock;//file通过private_data与socket关联
 return file; //返回初始化、关联后的file结构体
}
```

其中file = alloc_file(&path, FMODE_READ | FMODE_WRITE,
&socket_file_ops);分配了file结构体并进行初始化：

```text
struct file *alloc_file(const struct path *path, fmode_t mode,
  const struct file_operations *fop)
{
 struct file *file;

 file = get_empty_filp();
 if (IS_ERR(file))
  return file;

 file->f_path = *path;
 file->f_inode = path->dentry->d_inode;
 file->f_mapping = path->dentry->d_inode->i_mapping;
 file->f_wb_err = filemap_sample_wb_err(file->f_mapping);
 if ((mode & FMODE_READ) &&
      likely(fop->read || fop->read_iter))
  mode |= FMODE_CAN_READ;
 if ((mode & FMODE_WRITE) &&
      likely(fop->write || fop->write_iter))
  mode |= FMODE_CAN_WRITE;
 file->f_mode = mode;
 file->f_op = fop;
 if ((mode & (FMODE_READ | FMODE_WRITE)) == FMODE_READ)
  i_readcount_inc(path->dentry->d_inode);
 return file;
}
```

其中file->f_op = fop，将socket_file_ops传递给文件操作表

```text
static const struct file_operations socket_file_ops = {
 .owner = THIS_MODULE,
 .llseek = no_llseek,
 .read_iter = sock_read_iter,
 .write_iter = sock_write_iter,
 .poll =  sock_poll,
 .unlocked_ioctl = sock_ioctl,
#ifdef CONFIG_COMPAT
 .compat_ioctl = compat_sock_ioctl,
#endif
 .mmap =  sock_mmap,
 .release = sock_close,
 .fasync = sock_fasync,
 .sendpage = sock_sendpage,
 .splice_write = generic_splice_sendpage,
 .splice_read = sock_splice_read,
};
```

以上操作完成了struct socket、struct sock、struct file等的创建、初始化、关联，并最终返回socket描述符fd

![img](https://pic2.zhimg.com/80/v2-e2b5ff5271eb567013ba3ab973bf8055_720w.webp)

**socket描述符fd和我们平时操作文件的文件描述符相同，那么会有一个疑问，可以看到struct file_operations socket_file_ops函数表中并没有提供write()和read()接口,只是看到read_iter，write_iter等接口，那么系统是如何处理的呢？**

以write()为例：

sys_write()->__vfs_write()

```text
ssize_t __vfs_write(struct file *file, const char __user *p, size_t count,
      loff_t *pos)
{
 if (file->f_op->write)//如果文件函数表结构体提供了write接口函数
  return file->f_op->write(file, p, count, pos);//调用它的write函数
 else if (file->f_op->write_iter)
  return new_sync_write(file, p, count, pos);//否则调用new_sync_write函数
 else
  return -EINVAL;
}
```

从__vfs_write函数中可以看出来，如果socket函数表中没有提供write接口函数，则调用new_sync_write:

```text
static ssize_t new_sync_write(struct file *filp, const char __user *buf, size_t len, loff_t *ppos)
{
 ......

 ret = call_write_iter(filp, &kiocb, &iter);
 ......
}
```

call_write_iter:

```text
static inline ssize_t call_write_iter(struct file *file, struct kiocb *kio,struct iov_iter *iter)
{
 return file->f_op->write_iter(kio, iter);//调用socket文件函数表的aio_write函数
}
```

从以上__vfs_write()分析，如果文件函数表结构提供了write接口函数则调用write函数，如果文件函数表结构没有提供write接口函数(如socket操作函数表中没有提供write接口)，则调用write_iter接口，即调用socket操作函数表中的sock_write_iter。就这样通过socket fd进行普通文件系统那样通过描述符进行读写等。

用户得到socket fd,可以进行地址绑定、发送以及接收数据等操作，在Linux内核中有相关的函数完成从socket fd到struct socket、struct file的转换：

```text
static struct socket *sockfd_lookup_light(int fd, int *err, int *fput_needed)
{
 struct fd f = fdget(fd);//通过socket fd获取struct fd结构体，struct fd结构体中有struct file结构
 struct socket *sock;

 *err = -EBADF;
 if (f.file) {
  sock = sock_from_file(f.file, err);//通过获取的struct file结构体获取相应的struct socket指针
  if (likely(sock)) {
   *fput_needed = f.flags;
   return sock;
  }
  fdput(f);
 }
 return NULL;
}
```

fdget()函数从当前进程的files_struct结构中找到网络文件系统中的file文件指针，并封装在struct fd结构体中。sock_from函数通过得到的file结构体得到对应的socket结构指针。sock_from函数如下：

```text
struct socket *sock_from_file(struct file *file, int *err)
{
 if (file->f_op == &socket_file_ops)
  return file->private_data; /* set in sock_map_fd */

 *err = -ENOTSOCK;
 return NULL;
}
```

至此，socket底层来龙去脉的大体结构大概就分析到这。

原文地址：https://zhuanlan.zhihu.com/p/456895712

作者：linux