# 【NO.536】Linux系统中的文件操作

## 1.文件的作用

```javascript
 linux中，一切皆文件（网络设备除外）



硬件设备也“是”文件，通过文件来使用设备



目录（文件夹）也是一种文件
```

## 2.Linux的文件结构



![在这里插入图片描述](https://img-blog.csdnimg.cn/58bb261d0ad14997a8c19264dd03a5d0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_17,color_FFFFFF,t_70,g_se,x_16)


root：该目录为系统管理员(也称作超级管理员)的用户主目录。



bin：bin是Binary的缩写, 这个目录存放着最经常使用的命令。

boot：这里存放的是启动Linux时使用的一些核心文件，包括一些连接文件以及镜像文件。

dev：dev是Device(设备)的缩写, 该目录下存放的是Linux的外部设备，在Linux中访问设备的方式和访问文件的方式是相同的。

etc：所有的配置文件, 所有的系统管理所需要的配置文件和子目录都存放在这里。

home：用户的主目录，在Linux中，每个用户都有一个自己的目录，一般该目录名是以用 户的账号命名的。

var：存放着在不断变化的文件数据，我们习惯将那些经常被修改的目录放在这个目录下。 包括各种日志文件。

lib：这个目录里存放着系统最基本的动态连接共享库，其作用类似于Windows里的DLL文 件。几乎所有的应用程序都需要用到这些共享库。

usr：系统用户工具和程序
-- bin：用户命令
-- sbin：超级用户使用的比较高级的管理程序和系统守护程序。
-- include：标准头文件
-- lib：库文件
-- src：内核源代码

tmp：用来存放一些临时文件。

media：linux 系统会自动识别一些设备，例如U盘、光驱等等，当识别后，linux会把识别 的设备挂载到这个目录下。

mnt：临时挂载其他文件。
proc: 包含了进程的相关信息。

## 3.文件操作方式

1. 文件描述符 fd

   ```javascript
     是一个>=0的整数
   
   
   
     每打开一个文件，就创建一个文件描述符，通过文件描述符来操作文件
   
   
   
     
   
   
   
     预定义的文件描述符：
   
   
   
     0：标准输入，对应于已打开的标准输入设备（键盘）
   
   
   
     1：标准输出，对应于已打开的标准输出设备（控制台）
   
   
   
     2 : 标准错误， 对应于已打开的标准错误输出设备（控制台）
   
   
   
     
   
   
   
     多次打开同一个文件，可得到多个不同的文件描述符。
   ```

1） 使用底层文件操作（系统调用）
比如：read
可使用man 2 查看
2） 使用I/O库函数
比如：fread
可使用man 3 查看

## 4.底层文件操作(关于文件的系统调用）

> 1. write
>
>    ```javascript
>      (1) 用法
>    ```
>
> 
>
>            man 2 write
>
> 
>
>      (2) 返回值
>
> 
>
>            成功：返回实际写入的字节数
>
> 
>
>            失败：返回 -1， 错误编号设置 errno  可用( strerror(errno) ) 查看
>
> 
>
>       注意：是从文件的当前指针位置写入！
>
> 
>
>                文件刚打开时，文件的位置指针指向文件头
>
>    ```
> 
>    ```

```cpp
// main1.c



#include <errno.h>



#include <string.h>



 



int main(void)



{



    int len = 0;



 



    char buff[] = "hello world\n";



 



    len = write(1, buff, sizeof(buff));



    if (len < 0)



    {



        printf("write to stdout failed. reason: %s\n", strerror(errno));



    }



    else



    {



        printf("write %d bytes.\n", len);



    }



 



    len = write(2, buff, sizeof(buff));



 



    if (len < 0)



    {



        printf("write to stderr failed. reason: %s\n", strerror(errno));



    }



 



    return 0;



}
```

> 1. read
>
>    ```javascript
>        (1)用法
>    ```
>
> 
>
>             man 2 read
>
> 
>
>        (2)返回值
>
> 
>
>              大于0 : 实际读取的字节数
>
> 
>
>              0  : 已读到文件尾
>
> 
>
>              -1 ：出错
>
> 
>
>        注意：参数3表示最多能接受的字节数，而不是指一定要输入的字节数
>
> 
>
>        实例：main2.c
>
> 
>
>        运行：# ./a.out          /* 用户输入回车符时结束输入 */
>
> 
>
>              # ./a.out < main.c      /* 利用重定向， 使用文件main.c作为输入 */
>
>    ```
> 
>    ```

```cpp
// main2.c



#include <stdio.h>



#include <stdlib.h>



 



int main(void)



{



 



    char buffer[1024];



    int cnt = 0;



 



    cnt = read(0, buffer, sizeof(buffer));



 



    write(1, buffer, cnt);



 



    return 0;



}
```

> 1. open
>
>    ```javascript
>        (1) 用法
>    ```
>
> 
>
>             main 2 open
>
> 
>
>        (2) 返回值
>
> 
>
>              成功：文件描述符
>
> 
>
>              失败：-1
>
> 
>
>        (3) 打开方式
>
> 
>
>              O_RDONLY        只读
>
> 
>
>              O_WRONLY        只写
>
> 
>
>              O_RDWR           读写
>
> 
>
>                O_CREAT           如果文件不存在，则创建该文件，并使用第3个 
>
>    ```
>    参数设置权限,如果文件存在 ，则只打开文件
> 
>    ```javascript
>              O_EXCL            如果同时使用O_CREAT而且该文件又已经存在
>    ```
>
>    时，则返回错误, 用途：以防止多个进程同时创建
>
>    同一个文件
>
>    ```javascript
>              O_APPEND         尾部追加方式（打开后，文件指针指向文件的末尾）
> 
> 
> 
>              O_TRUNC          若文件存在，则长度被截为0，属性不变
> 
> 
> 
>     example:  open("/dev/hello", O_RDONLY|O_CREAT|O_EXCL, 0777)              
> 
> 
> 
>        (4) 参数3 (设置权限）
> 
> 
> 
>              当使用O_CREAT时，使用参数3             
> 
> 
> 
>              S_I(R/W/X)(USR/GRP/OTH)
> 
> 
> 
>              例：
> 
> 
> 
>                S_IRUSR | S_IWUSR    文件的所有者对该文件可读可写
> 
> 
> 
>                (八进制表示法)0600     文件的所有者对该文件可读可写    
> 
> 
> 
>        注意：
> 
> 
> 
>              返回的文件描述符是该进程未打开的最小的文件描述符
>    ```

```cpp
// open_demo.c



#include <stdlib.h>



#include <sys/types.h>



#include <sys/stat.h>



#include <fcntl.h>



#include <errno.h>



#include <string.h>



 



#define FILE_RW_LEN 1024



 



int main(void)



{



    int fd = 0;



    int count = 0;



    char buffer[FILE_RW_LEN] = "I 'm Martin.";



 



    fd = open("./martin.txt", O_CREAT | O_RDWR | O_APPEND | O_TRUNC, S_IRWXU | S_IRGRP | S _IXGRP | S_IROTH);



    // fd = open("./martin.txt", O_CREAT|O_EXCL|O_RDWR, S_IRWXU|S_IRGRP|S_IXGRP|S                                 _IROTH);



 



    if (fd < 0)



    {



        printf("open file martin.txt failed. reason: %s\n", strerror(errno));



        exit(-1);



    }



 



    count = write(fd, buffer, strlen(buffer));



 



    printf("written: %d bytes.\n", count);



 



    close(fd);



}
```

> 1. close
>
>    ```javascript
>       (1) 用法
>    ```
>
> 
>
>             man 2 close                
>
> 
>
>             终止指定文件描述符与对应文件之间的关联，
>
> 
>
>             并释放该文件描述符，即该文件描述符可被重新使用              
>
> 
>
>       (2）返回值
>
> 
>
>            成功： 0
>
> 
>
>            失败： -1              
>
> 
>
>       实例：
>
> 
>
>          使用read/write实现文件复制 
>
> 
>
>          close_demo.c 
>
>    ```
> 
>    ```

```cpp
// close_demo.c



#include <stdlib.h>



#include <stdio.h>



 



#define FILE1_NAME "file1.txt"



#define FILE2_NAME "file2.txt"



 



int main(void)



{



 



    int file1, file2;



    char buffer[4096];



    int len = 0;



 



    file1 = open(FILE1_NAME, O_RDONLY);



    if (file1 < 0)



    {



        printf("open file %s failed\n, reason: %s\n", FILE1_NAME, strerror(errno));



        exit(-1);



    }



 



    file2 = open(FILE2_NAME, O_CREAT | O_WRONLY, S_IRUSR | S_IWUSR);



 



    if (file2 < 0)



    {



        printf("open file %s failed\n, reason: %s\n", FILE2_NAME, strerror(errno));



        exit(-1);



    }



 



    while ((len = read(file1, buffer, sizeof(buffer))) > 0)



    {



        write(file2, buffer, len);



    }



 



    close(file2);



    close(file1);



    return 0;



}
```

观察耗时
./a.out
time ./a.out

```javascript
         补充：time命令



          time命令分别输出:



              real - 程序总的执行时间、



              usr - 该程序本身所消耗的时间、



              sys - 系统调用所消耗的时间
```

> 1. lseek
>
>    ```javascript
>      (1) 用法
>    ```
>
> 
>
>           man 2 lseek
>
> 
>
>      (2) 返回值
>
> 
>
>           成功：返回新的文件位置与文件头之间偏移
>
> 
>
>           失败： -1
>
> 
>
>      实例：从文件偏移量100的位置拷贝100个字节到另一个文件
>
> 
>
>      lseek_demo.c
>
>    ```
> 
>    ```

```cpp
// lseek_demo.c



#include <sys/types.h>



#include <sys/stat.h>



#include <fcntl.h>



#include <errno.h>



#include <stdlib.h>



#include <stdio.h>



#include <string.h>



 



#define FILE1_NAME "lseek_demo.c"



#define FILE2_NAME "lseek_demo_2.c"



 



#define SIZE 100



 



int main(void)



{



    int file1, file2;



    char buffer[1024];



    int ret;



 



    file1 = open(FILE1_NAME, O_RDONLY);



    if (file1 < 0)



    {



        printf("open file %s failed\n", FILE1_NAME);



        exit(-1);



    }



 



    file2 = open(FILE2_NAME, O_WRONLY | O_CREAT, S_IRUSR | S_IWUSR);



    if (file2 < 0)



    {



        printf("open file %s failed\n", FILE2_NAME);



        exit(-1);



    }



 



    // file size



    ret = lseek(file1, 0, SEEK_END);



    printf("file size: %d\n", ret);



 



    ret = lseek(file1, 100, SEEK_SET);



    printf("lseek ret: %d\n", ret);



 



    ret = read(file1, buffer, SIZE);



    if (ret > 0)



    {



        buffer[ret] = '\0';



        printf("read[%d]: %s\n", ret, buffer);



        write(file2, buffer, SIZE);



    }



 



    close(file1);



    close(file2);



    return 0;



}
```

> 1. ioctl
>
>    ```javascript
>      ioctl是设备驱动程序中对设备的I/O通道进行管理的函数。所谓对I/O通道进行管理，就是对设备的一些特性进行控制，例如串口的传输波特率、马达的转速等等。是设备驱动程序中设备控制接口函数,用来控制设备.
>    ```
>
>    函数名: ioctl
>
>    功 能: 控制I/O设备
>
>    用 法: int ioctl(int fd, int cmd,[int *argdx, int argcx]);
>
>    参数：fd是用户程序打开设备时使用open函数返回的文件标示符，cmd是用户程序对设备的控制命令，后面是一些补充参数，一般最多一个，这个参数的有无和cmd的意义相关。

## 5.系统调用

### 5.1 标准I/O库

直接使用系统调用的缺点
(1) 影响系统性能
系统调用比普通函数调用开销大
因为，频繁的系统调用要进行用户空间和内核空间的切换
(2) 系统调用一次所能读写的数据量大小，受硬件的限制

```javascript
       解决方案: 使用带缓冲功能的标准I/O库（以减少系统调用次数）



       



/* C语言中的文件操作中已描述 */



1) fwrite



2) fread



3) fopen



4) fclose



5) fseek    



6) fflush
```

## 6.proc文件系统

/proc是一个特殊的文件系统，
该目录下文件用来表示与启动、内核相关的特殊信息

```javascript
1) /proc/cpuinfo



   CPU详细信息



   



2) /proc/meminfo



   内存相关信息



 



3) /proc/version



   版本信息



 



4) /proc/sys/fs/file-max



   系统中能同时打开的文件总数



   可修改该文件



 



5) 进程的相关信息



   /proc/32689/ 表示指定进程（进程号为32689)的相关信息



   



6) /proc/devices



    已分配的字符设备、块设备的设备号
```

## 7.文件锁

1. 并发对文件I/O操作的影响

   ```javascript
    解决办法？
   ```

   2)文件锁

   ```javascript
    用法：man 2 fcntl
   
   
   
    
   
   
   
    头文件：#include <unistd.h>
   
   
   
            #include <fcntl.h>
   
   
   
    
   
   
   
   函数定义：int fcntl(int fd, int cmd, ... /* arg */ );
   
   
   
       参数： cmd 取值  F_GETLK,   F_SETLK 和 F_SETLKW ,分别表示获取锁、设置锁和同步设置锁.
   
   
   
    
   
   
   
    文件锁的表示
   
   
   
          struct flock
   ```

> // struct flock 结构体说明
> struct flock {
> short l_type; /*F_RDLCK, F_WRLCK, or F_UNLCK */
> off_t l_start; /*offset in bytes, relative to l_whence */
> short l_whence; /*SEEK_SET, SEEK_CUR, or SEEK_END */
> off_t l_len; /*length, in bytes; 0 means lock to EOF */
> pid_t l_pid; /*returned with F_GETLK */
> };
> l_type: 第一个成员是加锁的类型：只读锁，读写锁，或是解锁。
> l_start和l_whence: 用来指明加锁部分的开始位置。
> l_len: 是加锁的长度。
> l_pid: 是加锁进程的进程id。
> 举例：
> 我们现在需要把一个文件的前三个字节加读锁，则该结构体的l_type=F_RDLCK, l_start=0,
> l_whence=SEEK_SET, l_len=3, l_pid不需要指定，然后调用fcntl函数时，
> cmd参数使F_SETLK.

```cpp
//



#include <unistd.h>



#include <fcntl.h>



#include <stdio.h>



#include <stdlib.h>



 



#define FILE_NAME "test.txt"



 



int flock_set(int fd, int type)



{



    printf("pid=%d into...\n", getpid());



 



    struct flock flock;



    flock.l_type = type;



    flock.l_whence = SEEK_SET;



    flock.l_start = 0;



    flock.l_len = 0;



    flock.l_pid = -1;



 



    fcntl(fd, F_GETLK, &flock);



 



    if (flock.l_type != F_UNLCK)



    {



        if (flock.l_type == F_RDLCK)



        {



            printf("flock has been set to read lock by %d\n", flock.l_pid);



        }



        else if (flock.l_type == F_WRLCK)



        {



            printf("flock has been set to write lock by %d\n", flock.l_pid);



        }



    }



 



    flock.l_type = type;



    if (fcntl(fd, F_SETLKW, &flock) < 0)



    {



        printf("set lock failed!\n");



        return -1;



    }



 



    switch (flock.l_type)



    {



    case F_RDLCK:



        printf("read lock is set by %d\n", getpid());



        break;



    case F_WRLCK:



        printf("write lock is set by %d\n", getpid());



        break;



    case F_UNLCK:



        printf("lock is released by %d\n", getpid());



        break;



    default:



        break;



    }



 



    printf("pid=%d out.\n", getpid());



    return 0;



}



 



int main(void)



{



    int fd;



 



    fd = open(FILE_NAME, O_RDWR | O_CREAT, 0666);



    if (fd < 0)



    {



        printf("open file %s failed!\n", FILE_NAME);



    }



 



    // flock_set(fd, F_WRLCK);



    flock_set(fd, F_RDLCK);



    getchar();



    flock_set(fd, F_UNLCK);



    getchar();



 



    close(fd);



    return 0;



}
```

原文作者：[我也要当昏君](https://blog.csdn.net/qq_46118239)

原文链接：https://bbs.csdn.net/topics/605725262