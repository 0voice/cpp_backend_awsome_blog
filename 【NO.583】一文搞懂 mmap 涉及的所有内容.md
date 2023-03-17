# 【NO.583】一文搞懂 mmap 涉及的所有内容

内存映射，简而言之就是将内核空间的一段内存区域映射到用户空间。映射成功后，用户对这段内存区域的修改可以直接反映到内核空间，相反，内核空间对这段区域的修改也直接反映用户空间。那么对于内核空间与用户空间两者之间需要大量数据传输等操作的话效率是非常高的。当然，也可以将内核空间的一段内存区域同时映射到多个进程，这样还可以实现进程间的共享内存通信。

系统调用mmap()就是用来实现上面说的内存映射。最长见的操作就是文件（在Linux下设备也被看做文件）的操作，可以将某文件映射至内存(进程空间)，如此可以把对文件的操作转为对内存的操作，以此避免更多的lseek()与read()、write()操作，这点对于大文件或者频繁访问的文件而言尤其受益。

## 1.**概述**

mmap将一个文件或者其它对象映射进内存。文件被映射到多个页上，如果文件的大小不是所有页的大小之和，最后一个页不被使用的空间将会清零。munmap执行相反的操作，删除特定地址区域的对象映射。

当使用mmap映射文件到进程后，就可以直接操作这段虚拟地址进行文件的读写等操作，不必再调用read，write等系统调用。但需注意，直接对该段内存写时不会写入超过当前文件大小的内容。

采用共享内存通信的一个显而易见的好处是效率高，因为进程可以直接读写内存，而不需要任何数据的拷贝。对于像管道和消息队列等通信方式，则需要在内核和用户空间进行四次的数据拷贝，而共享内存则只拷贝两次数据：一次从输入文件到共享内存区，另一次从共享内存区到输出文件。实际上，进程之间在共享内存时，并不总是读写少量数据后就解除映射，有新的通信时，再重新建立共享内存区域。而是保持共享区域，直到通信完毕为止，这样，数据内容一直保存在共享内存中，并没有写回文件。共享内存中的内容往往是在解除映射时才写回文件的。因此，采用共享内存的通信方式效率是非常高的。

通常使用mmap()的三种情况： **提高I/O效率**、**匿名内存映射**、**共享内存进程通信**。

用户空间 mmap()函数 void *mmap(void *start, size_t length, int prot, int flags,int fd, off_t offset)，下面就其参数解释如下：

- start：用户进程中要映射的用户空间的起始地址，通常为NULL（由内核来指定）
- length：要映射的内存区域的大小
- prot：期望的内存保护标志
- flags：指定映射对象的类型
- fd：文件描述符（由open函数返回）
- offset：设置在内核空间中已经分配好的的内存区域中的偏移，例如文件的偏移量，大小为PAGE_SIZE的整数倍
- 返回值：mmap()返回被映射区的指针，该指针就是需要映射的内核空间在用户空间的虚拟地址

![img](https://pic3.zhimg.com/80/v2-10a229c3b2bd64c17c41ffcfa927668a_720w.webp)

## 2.**内存映射的应用**

- X Window服务器
- 众多内存数据库如MongoDB操作数据,就是把文件磁盘内容映射到内存中进行处理,为什么会提高效率? 很多人不解. 下面就深入分析内存文件映射.
- 通过malloc来分配大内存其实调用的是mmap，可见在malloc(10)的时候调用的是brk, malloc(10 * 1024 * 1024)调用的是mmap

**mmap()用于共享内存的两种方式**

- 使用普通文件提供的内存映射：适用于任何进程之间；此时，需要打开或创建一个文件，然后再调用mmap()；典型调用代码如下：

```text
fd=open(name, flag, mode);   
if(fd<0)   
   ...   
ptr=mmap(NULL, len , PROT_READ|PROT_WRITE, MAP_SHARED , fd , 0);  
```

- 使用特殊文件提供匿名内存映射：适用于具有亲缘关系的进程之间；由于父子进程特殊的亲缘关系，在父进程中先调用mmap()，然后调用fork()。那么在调用fork()之后，子进程继承父进程匿名映射后的地址空间，同样也继承mmap()返回的地址，这样，父子进程就可以通过映射区域进行通信了。注意，这里不是一般的继承关系。一般来说，子进程单独维护从父进程继承下来的一些变量。而mmap()返回的地址，却由父子进程共同维护。 对于具有亲缘关系的进程实现共享内存最好的方式应该是采用匿名内存映射的方式。此时，不必指定具体的文件，只要设置相应的标志即可。

## 3.**示例**

**驱动+应用**

首先在驱动程序分配一页大小的内存，然后用户进程通过mmap()将用户空间中大小也为一页的内存映射到内核空间这页内存上。映射完成后，驱动程序往这段内存写10个字节数据，用户进程将这些数据显示出来。

```text
#include <linux/miscdevice.h>  
#include <linux/delay.h>  
#include <linux/kernel.h>  
#include <linux/module.h>  
#include <linux/init.h>  
#include <linux/mm.h>  
#include <linux/fs.h>  
#include <linux/types.h>  
#include <linux/delay.h>  
#include <linux/moduleparam.h>  
#include <linux/slab.h>  
#include <linux/errno.h>  
#include <linux/ioctl.h>  
#include <linux/cdev.h>  
#include <linux/string.h>  
#include <linux/list.h>  
#include <linux/pci.h>  
#include <linux/gpio.h>  
  
#define DEVICE_NAME "mymap"  
  
static unsigned char array[10]={0,1,2,3,4,5,6,7,8,9};  
static unsigned char *buffer;  
  
static int my_open(struct inode *inode, struct file *file)  
{  
    return 0;  
}  
  
static int my_map(struct file *filp, struct vm_area_struct *vma)  
{      
    unsigned long page;  
    unsigned char i;  
    unsigned long start = (unsigned long)vma->vm_start;  
    //unsigned long end =  (unsigned long)vma->vm_end;  
    unsigned long size = (unsigned long)(vma->vm_end - vma->vm_start);  
    //得到物理地址  
    page = virt_to_phys(buffer);      
    //将用户空间的一个vma虚拟内存区映射到以page开始的一段连续物理页面上  
    if(remap_pfn_range(vma,start,page>>PAGE_SHIFT,size,PAGE_SHARED))//第三个参数是页帧号，由物理地址右移PAGE_SHIFT得到  
        return -1;  
    //往该内存写10字节数据  
    for(i=0;i<10;i++)  
        buffer[i] = array[i];  
      
    return 0;  
}  
  
static struct file_operations dev_fops = {  
    .owner    = THIS_MODULE,  
    .open    = my_open,  
    .mmap   = my_map,  
};  
  
static struct miscdevice misc = {  
    .minor = MISC_DYNAMIC_MINOR,  
    .name = DEVICE_NAME,  
    .fops = &dev_fops,  
};  
  
static int __init dev_init(void)  
{  
    int ret;      
    //注册混杂设备  
    ret = misc_register(&misc);  
    //内存分配  
    buffer = (unsigned char *)kmalloc(PAGE_SIZE,GFP_KERNEL);  
    //将该段内存设置为保留  
    SetPageReserved(virt_to_page(buffer));  
  
    return ret;  
}  

static void __exit dev_exit(void)  
{  
    //注销设备  
    misc_deregister(&misc);  
    //清除保留  
    ClearPageReserved(virt_to_page(buffer));  
    //释放内存  
    kfree(buffer);  
}  

module_init(dev_init);  
module_exit(dev_exit);  
MODULE_LICENSE("GPL");  
MODULE_AUTHOR("LKN@SCUT");  

#include <unistd.h>  
#include <stdio.h>  
#include <stdlib.h>  
#include <string.h>  
#include <fcntl.h>  
#include <linux/fb.h>  
#include <sys/mman.h>  
#include <sys/ioctl.h>   
  
#define PAGE_SIZE 4096  
  
int main(int argc , char *argv[])  
{  
    int fd;  
    int i;  
    unsigned char *p_map;  
    //打开设备  
    fd = open("/dev/mymap",O_RDWR);  
    if(fd < 0)  
    {  
        printf("open fail\n");  
        exit(1);  
    }  
    //内存映射  
    p_map = (unsigned char *)mmap(0, PAGE_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED,fd, 0);  
    if(p_map == MAP_FAILED)  
    {  
        printf("mmap fail\n");  
        goto here;  
    }  
    //打印映射后的内存中的前10个字节内容  
    for(i=0;i<10;i++)  
        printf("%d\n",p_map[i]);  
        
here:  
    munmap(p_map, PAGE_SIZE);  
    return 0;  
}  
```

## 4. **进程间共享内存**

UNIX访问文件的传统方法是用open打开它们, 如果有多个进程访问同一个文件, 则每一个进程在自己的地址空间都包含有该文件的副本，这不必要地浪费了存储空间。下图说明了两个进程同时读一个文件的同一页的情形。系统要将该页从磁盘读到高速缓冲区中, 每个进程再执行一个存储器内的复制操作将数据从高速缓冲区读到自己的地址空间。

![img](https://pic2.zhimg.com/80/v2-60bf94d9289ecb0ce6fe2957a9f9f715_720w.webp)

现在考虑另一种处理方法共享存储映射: 进程A和进程B都将该页映射到自己的地址空间, 当进程A第一次访问该页中的数据时, 它生成一个缺页中断。内核此时读入这一页到内存并更新页表使之指向它。以后, 当进程B访问同一页面而出现缺页中断时, 该页已经在内存, 内核只需要将进程B的页表登记项指向次页即可。如下图所示:

![img](https://pic2.zhimg.com/80/v2-60bf94d9289ecb0ce6fe2957a9f9f715_720w.webp)

下面就是进程A和B共享内存的示例。两个程序映射同一个文件到自己的地址空间, 进程A先运行, 每隔两秒读取映射区域, 看是否发生变化。进程B后运行, 它修改映射区域, 然后退出, 此时进程A能够观察到存储映射区的变化。

进程A的代码:

```text
#include <sys/mman.h>    
#include <sys/stat.h>    
#include <fcntl.h>    
#include <stdio.h>    
#include <stdlib.h>    
#include <unistd.h>    
#include <error.h>    
    
#define BUF_SIZE 100    
    
int main(int argc, char **argv)    
{    
    int fd, nread, i;    
    struct stat sb;    
    char *mapped, buf[BUF_SIZE];    
    
    for (i = 0; i < BUF_SIZE; i++) {    
        buf[i] = '#';    
    }    
    /* 打开文件 */    
    if ((fd = open(argv[1], O_RDWR)) < 0) {    
        perror("open");    
    }    
    /* 获取文件的属性 */    
    if ((fstat(fd, &sb)) == -1) {    
        perror("fstat");    
    }   
    /* 将文件映射至进程的地址空间 */    
    if ((mapped = (char *)mmap(NULL, sb.st_size, PROT_READ |     
                    PROT_WRITE, MAP_SHARED, fd, 0)) == (void *)-1) {    
        perror("mmap");    
    }    
    /* 文件已在内存, 关闭文件也可以操纵内存 */    
    close(fd);    
    /* 每隔两秒查看存储映射区是否被修改 */    
    while (1) {    
        printf("%s\n", mapped);    
        sleep(2);    
    }    
    
    return 0;    
}    
```

进程B的代码:

```text
#include <sys/mman.h>    
#include <sys/stat.h>    
#include <fcntl.h>    
#include <stdio.h>    
#include <stdlib.h>    
#include <unistd.h>    
#include <error.h>    
    
#define BUF_SIZE 100    
    
int main(int argc, char **argv)    
{    
    int fd, nread, i;    
    struct stat sb;    
    char *mapped, buf[BUF_SIZE];    
    
    for (i = 0; i < BUF_SIZE; i++) {    
        buf[i] = '#';    
    }    
    /* 打开文件 */    
    if ((fd = open(argv[1], O_RDWR)) < 0) {    
        perror("open");    
    }    
    /* 获取文件的属性 */    
    if ((fstat(fd, &sb)) == -1) {    
        perror("fstat");    
    }    
    /* 私有文件映射将无法修改文件 */    
    if ((mapped = (char *)mmap(NULL, sb.st_size, PROT_READ |     
                    PROT_WRITE, MAP_PRIVATE, fd, 0)) == (void *)-1) {    
        perror("mmap");    
    }    
    /* 映射完后, 关闭文件也可以操纵内存 */    
    close(fd);    
    /* 修改一个字符 */    
    mapped[20] = '9';    
     
    return 0;    
}    
```

**匿名映射实现父子进程通信**

```text
#include <sys/mman.h>    
#include <stdio.h>    
#include <stdlib.h>    
#include <unistd.h>    
    
#define BUF_SIZE 100    
    
int main(int argc, char** argv)    
{    
    char    *p_map;    
    /* 匿名映射,创建一块内存供父子进程通信 */    
    p_map = (char *)mmap(NULL, BUF_SIZE, PROT_READ | PROT_WRITE,    
            MAP_SHARED | MAP_ANONYMOUS, -1, 0);    
    
    if(fork() == 0) {    
        sleep(1);    
        printf("child got a message: %s\n", p_map);    
        sprintf(p_map, "%s", "hi, dad, this is son");    
        munmap(p_map, BUF_SIZE); //实际上，进程终止时，会自动解除映射。    
        exit(0);    
    }    
    
    sprintf(p_map, "%s", "hi, this is father");    
    sleep(2);    
    printf("parent got a message: %s\n", p_map);    
    
    return 0;    
}    
```

## 5.**mmap进行内存映射的原理**

mmap系统调用的最终目的是将设备或文件映射到用户进程的虚拟地址空间，实现用户进程对文件的直接读写，这个任务可以分为以下三步:

1.在用户虚拟地址空间中寻找空闲的满足要求的一段连续的虚拟地址空间,为映射做准备(由内核mmap系统调用完成)

假如vm_area_struct描述的是一个文件映射的虚存空间，成员vm_file便指向被映射的文件的file结构，vm_pgoff是该虚存空间起始地址在vm_file文件里面的文件偏移，单位为物理页面。mmap系统调用所完成的工作就是准备这样一段虚存空间,并建立vm_area_struct结构体,将其传给具体的设备驱动程序.

2.建立虚拟地址空间和文件或设备的物理地址之间的映射(设备驱动完成) 建立文件映射的第二步就是建立虚拟地址和具体的物理地址之间的映射，这是通过修改进程页表来实现的。mmap方法是file_opeartions结构的成员:int (*mmap)(struct file *,struct vm_area_struct *);

linux有2个方法建立页表:

- 使用remap_pfn_range一次建立所有页表。int remap_pfn_range(struct vm_area_struct *vma, unsigned long virt_addr, unsigned long pfn, unsigned long size, pgprot_t prot)。
- 使用nopage VMA方法每次建立一个页表项。 struct page *(*nopage)(struct vm_area_struct *vma, unsigned long address, int *type);

3.当实际访问新映射的页面时的操作(由缺页中断完成)

- page cache及swap cache中页面的区分：一个被访问文件的物理页面都驻留在page cache或swap cache中，一个页面的所有信息由struct page来描述。struct page中有一个域为指针mapping ，它指向一个struct address_space类型结构。page cache或swap cache中的所有页面就是根据address_space结构以及一个偏移量来区分的。
- 文件与 address_space结构的对应：一个具体的文件在打开后，内核会在内存中为之建立一个struct inode结构，其中的i_mapping域指向一个address_space结构。这样，一个文件就对应一个address_space结构，一个 address_space与一个偏移量能够确定一个page cache 或swap cache中的一个页面。因此，当要寻址某个数据时，很容易根据给定的文件及数据在文件内的偏移量而找到相应的页面。
- 进程调用mmap()时，只是在进程空间内新增了一块相应大小的缓冲区，并设置了相应的访问标识，但并没有建立进程空间到物理页面的映射。因此，第一次访问该空间时，会引发一个缺页异常。
- 对于共享内存映射情况，缺页异常处理程序首先在swap cache中寻找目标页（符合address_space以及偏移量的物理页），如果找到，则直接返回地址；如果没有找到，则判断该页是否在交换区 (swap area)，如果在，则执行一个换入操作；如果上述两种情况都不满足，处理程序将分配新的物理页面，并把它插入到page cache中。进程最终将更新进程页表。 注：对于映射普通文件情况（非共享映射），缺页异常处理程序首先会在page cache中根据address_space以及数据偏移量寻找相应的页面。如果没有找到，则说明文件数据还没有读入内存，处理程序会从磁盘读入相应的页面，并返回相应地址，同时，进程页表也会更新.
- 所有进程在映射同一个共享内存区域时，情况都一样，在建立线性地址与物理地址之间的映射之后，不论进程各自的返回地址如何，实际访问的必然是同一个共享内存区域对应的物理页面。

原文地址：https://zhuanlan.zhihu.com/p/608556281

作者：linux