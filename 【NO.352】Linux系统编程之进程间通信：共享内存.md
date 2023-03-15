# 【NO.352】Linux系统编程之进程间通信：共享内存

共享内存是进程间通信中最简单的方式之一。共享内存允许两个或更多进程访问同一块内存，就如同 malloc() 函数向不同进程返回了指向同一个物理内存区域的指针。当一个进程改变了这块地址中的内容的时候，其它进程都会察觉到这个更改。



![img](https://pic3.zhimg.com/80/v2-a280de08b184b5fd2a5c35c3330deee2_720w.webp)

共享内存的特点：

1）共享内存是进程间共享数据的一种最快的方法。

一个进程向共享的内存区域写入了数据，共享这个内存区域的所有进程就可以立刻看到其中的内容。

2）使用共享内存要注意的是多个进程之间对一个给定存储区访问的互斥。

若一个进程正在向共享内存区写数据，则在它做完这一步操作前，别的进程不应当去读、写这些数据。

常用函数

1）创建共享内存

所需头文件：

```text
#include <sys/ipc.h>
#include <sys/shm.h>
```

int shmget(key_t key, size_t size,int shmflg);

功能：

创建或打开一块共享内存区。

参数：

key：进程间通信键值，ftok() 的返回值。

size：该共享存储段的长度(字节)。

shmflg：标识函数的行为及共享内存的权限，其取值如下：

IPC_CREAT：如果不存在就创建

IPC_EXCL： 如果已经存在则返回失败

位或权限位：共享内存位或权限位后可以设置共享内存的访问权限，格式和 open() 函数的 mode_t 一样（open() 的使用请点此链接），但可执行权限未使用。

返回值：

成功：共享内存标识符。

失败：-1。

示例代码如下：

```text
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/shm.h>
 
#define BUFSZ 1024
 
int main(int argc, char *argv[])
{
	int shmid;
	key_t key;
	
	key = ftok("./", 2015); 
	if(key == -1)
	{
		perror("ftok");
	}
	
	//创建共享内存
	shmid = shmget(key, BUFSZ, IPC_CREAT|0666);	
	if(shmid < 0) 
	{ 
		perror("shmget"); 
		exit(-1); 
	} 
 
	return 0;
}
```

运行结果如下：

![img](https://pic2.zhimg.com/80/v2-8108c55cd015c8282af81fc37f467249_720w.webp)



2）共享内存映射

所需头文件：

```text
#include <sys/types.h>
#include <sys/shm.h>
```

void *shmat(int shmid, const void *shmaddr, int shmflg);

功能：

将一个共享内存段映射到调用进程的数据段中。简单来理解，让进程和共享内存建立一种联系，让进程某个指针指向此共享内存。

参数：

shmid：共享内存标识符，shmget() 的返回值。

shmaddr：共享内存映射地址(若为 NULL 则由系统自动指定)，推荐使用 NULL。

shmflg：共享内存段的访问权限和映射条件( 通常为 0 )，具体取值如下：

0：共享内存具有可读可写权限。

SHM_RDONLY：只读。

SHM_RND：（shmaddr 非空时才有效）

返回值：

成功：共享内存段映射地址( 相当于这个指针就指向此共享内存 )

失败：-1

3）解除共享内存映射

所需头文件：

```text
#include <sys/types.h>
#include <sys/shm.h>
```

int shmdt(const void *shmaddr);

功能：

将共享内存和当前进程分离( 仅仅是断开联系并不删除共享内存，相当于让之前的指向此共享内存的指针，不再指向)。

参数：

shmaddr：共享内存映射地址。

返回值：

成功：0

失败：-1

4）共享内存控制

所需的头文件：

\#include <sys/ipc.h>

\#include <sys/shm.h>

int shmctl(int shmid, int cmd, struct shmid_ds *buf);

功能：

共享内存属性的控制。

参数：

shmid：共享内存标识符。

cmd：函数功能的控制，其取值如下：

IPC_RMID：删除。(常用 )

IPC_SET：设置 shmid_ds 参数，相当于把共享内存原来的属性值替换为 buf 里的属性值。

IPC_STAT：保存 shmid_ds 参数，把共享内存原来的属性值备份到 buf 里。

SHM_LOCK：锁定共享内存段( 超级用户 )。

SHM_UNLOCK：解锁共享内存段。

SHM_LOCK 用于锁定内存，禁止内存交换。并不代表共享内存被锁定后禁止其它进程访问。其真正的意义是：被锁定的内存不允许被交换到虚拟内存中。这样做的优势在于让共享内存一直处于内存中，从而提高程序性能。

buf：shmid_ds 数据类型的地址(具体类型请点此链接 )，用来存放或修改共享内存的属性。

返回值：

成功：0

失败：-1

实战示例

接下来我们做这么一个例子：创建两个进程，在 A 进程中创建一个共享内存，并向其写入数据，通过 B 进程从共享内存中读取数据。

写端代码如下：

```text
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/shm.h>
 
#define BUFSZ 512
 
int main(int argc, char *argv[])
{
	int shmid;
	int ret;
	key_t key;
	char *shmadd;
	
	//创建key值
	key = ftok("../", 2015); 
	if(key == -1)
	{
		perror("ftok");
	}
	
	//创建共享内存
	shmid = shmget(key, BUFSZ, IPC_CREAT|0666);	
	if(shmid < 0) 
	{ 
		perror("shmget"); 
		exit(-1); 
	}
	
	//映射
	shmadd = shmat(shmid, NULL, 0);
	if(shmadd < 0)
	{
		perror("shmat");
		_exit(-1);
	}
	
	//拷贝数据至共享内存区
	printf("copy data to shared-memory\n");
	bzero(shmadd, BUFSZ); // 共享内存清空
	strcpy(shmadd, "how are you, mike\n");
	
	return 0;
}
```

读端代码如下：

```text
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/shm.h>
 
#define BUFSZ 512
 
int main(int argc, char *argv[])
{
	int shmid;
	int ret;
	key_t key;
	char *shmadd;
	
	//创建key值
	key = ftok("../", 2015); 
	if(key == -1)
	{
		perror("ftok");
	}
	
	system("ipcs -m"); //查看共享内存
	
	//打开共享内存
	shmid = shmget(key, BUFSZ, IPC_CREAT|0666);
	if(shmid < 0) 
	{ 
		perror("shmget"); 
		exit(-1); 
	} 
	
	//映射
	shmadd = shmat(shmid, NULL, 0);
	if(shmadd < 0)
	{
		perror("shmat");
		exit(-1);
	}
	
	//读共享内存区数据
	printf("data = [%s]\n", shmadd);
	
	//分离共享内存和当前进程
	ret = shmdt(shmadd);
	if(ret < 0)
	{
		perror("shmdt");
		exit(1);
	}
	else
	{
		printf("deleted shared-memory\n");
	}
	
	//删除共享内存
	shmctl(shmid, IPC_RMID, NULL);
	
	system("ipcs -m"); //查看共享内存
	
	return 0;
}
```

运行结果如下：

![img](https://pic3.zhimg.com/80/v2-ea961b44c86d0b293bee95b83d730d06_720w.webp)

原文地址：https://zhuanlan.zhihu.com/p/147826545

作者：linux