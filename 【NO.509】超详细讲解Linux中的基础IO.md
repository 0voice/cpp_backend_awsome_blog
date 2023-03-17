# 【NO.509】超详细讲解Linux中的基础IO

## 1.系统文件IO

操作文件除了使用C语言、C++以及其他的语言的接口之外，还可以使用操作系统提供的接口

### 1.1 open

![img](https://pic2.zhimg.com/80/v2-5cc02b24372d541d1c31bb940e8efd41_720w.webp)

#### 1.1.1 open的第一个参数

open函数的第一个参数是pathname，表示要打开或创建的目标文件

- 若pathname以**路径**的方式给出，则需要创建该文件时，在pathname路径下进行创建
- 若pathname以**文件名**的方式给出，则需要创建该文件时，默认在当前路径下进行创建

#### 1.1.2 open的第二个参数

open函数的第二个参数是flags，表示打开文件的方式

![img](https://pic4.zhimg.com/80/v2-b73b60dec6d0784c2732b415b460049f_720w.webp)

上面提供的参数选项仅仅是较为常用的，具体可以man 2 open命令进行查看

打开文件时可以传入多个参数选项，当有多个选项传入时，将这些选项用"按位或"隔开

**传参原理:**

系统接口open的第二个参数flags为整型类型，在32位平台上有32个bit位。若将一个bit位作为一个标志位，则理论上flags可以传递32种不同的标志位。而实际上flags的参数选项都是宏

![img](https://pic3.zhimg.com/80/v2-d0fca16077f4dbbcf406e5790b200866_720w.webp)

在oen函数内部就可以通过使用一系列的位运算来判断是否设置了某一选项。

#### 1.1.3 open的第三个参数

open函数的第三个参数是mode，表示创建文件的默认权限

传入0666参数进行文件创建，按理应得到-rw-rw-rw-权限的文件，但却得到了如下图的文件

![img](https://pic3.zhimg.com/80/v2-2418212ab7397e9fd89bc6ff15a749f6_720w.webp)

这是因为权限掩码的存在(默认为0002)，可以在代码中设置权限掩码来避免上述情况发生。

```text
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
int main()
{
    umask(0000);//设置文件掩码                                                                     
    int fd = open("./log.txt",O_RDWR | O_CREAT | O_TRUNC,0666);
    close(fd);
    return 0;
}
```

![img](https://pic3.zhimg.com/80/v2-1af93c9f6743f9ca9c778338c162394a_720w.webp)

**注意:** 当不需创建文件时，可以不用设置第三个参数

#### 1.1.4 open的返回值

open函数成功调用后返回打开文件的文件描述符，若调用失败则返回-1。文件描述符具体在后面进行讲解。

### 1.2 close

使用close函数时传入需要关闭文件的**文件描述符**（即调用open函数的返回值）即可。若关闭文件成功则返回0；若关闭文件失败则返回-1。

![img](https://pic2.zhimg.com/80/v2-cc9dd1b05665d898b51c10f3c71ee6dd_720w.webp)

### 1.3 write

Linux系统接口中使用write函数向文件写入信息

![img](https://pic4.zhimg.com/80/v2-ab9ddb2f6d5a17491faa0c27950ef4cb_720w.webp)

将buf位置开始向后count字节的数据写入文件描述符为fd的文件当中。

- 若数据写入成功，则返回实际写入数据的字节数。
- 若数据写入失败，则返回-1。

```text
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
int main()
{
    int fd = open("./log.txt",O_RDWR | O_CREAT | O_TRUNC,0666);
    if(fd < 0){
        perror("open:");
        exit(1);
    }
 
    const char* buf = "hello Linux!\n";
    write(fd, buf, strlen(buf));                                                     
    close(fd);
    return 0;
}
```

![img](https://pic4.zhimg.com/80/v2-269e28789497b0e6f930f29028306e17_720w.webp)

Linux系统接口中使用write函数向文件写入信息

![img](https://pic4.zhimg.com/80/v2-a43f26ab65567a7c3481b988850d56ef_720w.webp)

从文件描述符为fd的文件读取count字节的数据到buf位置当中

- 若数据读取成功，则返回实际读取数据的字节数
- 若数据读取失败，则返回-1

```text
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
int main()
{
    int fd = open("./log.txt",O_RDONLY);
    if(fd < 0){
        perror("open:");
        exit(1);
    }
 
    char buf[64] = {0};
    while(read(fd, buf, 63)){
        printf("%s",buf);                                                                                                                                                        
        memset(buf,0x00,64);
    }
 
    close(fd);
    return 0;
}
```

![img](https://pic1.zhimg.com/80/v2-5b5856f82fa7eef26dc493130bf48cc4_720w.webp)

## 2.文件描述符

### 2.1 进程与文件描述符

文件被访问的前提是被加载到内存中打开，一个进程可以打开多个文件，而系统当中又存在大量进程。即在系统中任何时刻都可能存在大量已经打开的文件，所以操作系统内部提供了struct file结构体用于描述文件(先描述，再组织)，再使用双链表进行管理。

而为了区分已经打开的文件哪些属于特定的某一个进程，我们就还需要建立进程和文件之间的对应关系。

**进程与文件之间的对应关系是如何确定的呢？**

task_struct（PCB）中有一个指针，该指针指向一个名为files_struct的结构体。在该结构体中就有一个名为fd_array的指针数组，该数组的下标就是我们所谓的文件描述符。

当一个进程打开其第一个文件时，先将该文件从磁盘当中加载到内存，形成对应的struct file变量，将该struct file变量链入文件双链表中，并将该结构体的首地址填入到fd_array数组中下标为3的位置，最后返回该文件的文件描述符给调用进程。

![img](https://pic4.zhimg.com/80/v2-c3b5a32da1f0fb730adeea3c0aeb4cab_720w.webp)

因此只需要有某一文件的文件描述符，就可以找到与该文件相关的文件信息，进而对文件进行一系列输入输出操作。

**内存文件 VS 磁盘文件**

当文件存储在磁盘当中时，被称为磁盘文件；当磁盘文件被加载到内存中后，此时的文件被称为内存文件。磁盘文件和内存文件之间的关系就像程序和进程的关系一样，当程序加载到内存运行起来后便成了进程，而当磁盘文件加载到内存后便成了内存文件。

磁盘文件由两部分构成，分别是文件内容和文件属性。文件内容就是文件当中存储的数据，文件属性就是文件的一些基本信息，例如文件名、文件大小以及文件创建时间等信息都是文件属性，文件属性又被称为元信息。

文件加载到内存时，一般先加载文件的属性信息，当需要对文件内容进行读取、输入或输出等操作时，再延后式的加载文件数据。

### 2.2 文件描述符的分配规则

Linux进程会默认打开3个文件描述符，分别是标准输入0、标准输出1、标准错误2

分别对应的物理设备一般为：键盘、显示器、显示器

键盘和显示器都属于硬件，且操作系统能够识别。当某一进程创建时，操作系统就会根据键盘、显示器、显示器形成各自的struct file变量，将这3个struct file变量连入文件双链表当中，并将这3个struct file变量的地址分别填入fd_array数组下标为0、1、2的位置，至此就默认打开了标准输入流、标准输出流和标准错误流。

```text
#include <stdio.h>                                                                                                                                                               
#include <string.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
int main()
{
    int fd1 = open("./log1.txt",O_RDWR | O_CREAT | O_TRUNC,0666);
    int fd2 = open("./log2.txt",O_RDWR | O_CREAT | O_TRUNC,0666);
    int fd3 = open("./log3.txt",O_RDWR | O_CREAT | O_TRUNC,0666);
    int fd4 = open("./log4.txt",O_RDWR | O_CREAT | O_TRUNC,0666);
    int fd5 = open("./log5.txt",O_RDWR | O_CREAT | O_TRUNC,0666);
    printf("fd1: %d\n",fd1);
    printf("fd2: %d\n",fd2);
    printf("fd3: %d\n",fd3);
    printf("fd4: %d\n",fd4);
    printf("fd5: %d\n",fd5);
    close(fd1);
    close(fd2);
    close(fd3);
    close(fd4);
    close(fd5);
    return 0;
}
```

![img](https://pic2.zhimg.com/80/v2-600354a4c318311996b0439f9d07ccd5_720w.webp)

则后续打开的文件的文件描述符，从3开始逐个增加1。

在打开文件之前将文件描述符为0和2的文件关闭又会发生什么呢？

```text
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
int main()
{
    close(0);
    close(2);                                                                                                                                                                    
    int fd1 = open("./log1.txt",O_RDWR | O_CREAT | O_TRUNC,0666);
    int fd2 = open("./log2.txt",O_RDWR | O_CREAT | O_TRUNC,0666);
    int fd3 = open("./log3.txt",O_RDWR | O_CREAT | O_TRUNC,0666);
    int fd4 = open("./log4.txt",O_RDWR | O_CREAT | O_TRUNC,0666);
    int fd5 = open("./log5.txt",O_RDWR | O_CREAT | O_TRUNC,0666);
    printf("fd1: %d\n",fd1);
    printf("fd2: %d\n",fd2);
    printf("fd3: %d\n",fd3);
    printf("fd4: %d\n",fd4);
    printf("fd5: %d\n",fd5);
    close(fd1);
    close(fd2);
    close(fd3);
    close(fd4);
    close(fd5);
    return 0;
}
```

![img](https://pic4.zhimg.com/80/v2-b023236ad3d07e7db7b591c571039033_720w.webp)

**结论：** 文件描述符是从最小但是没有被使用的fd_array数组下标开始进行分配的

## 3.重定向

### 3.1 自实现重定向原理

#### 3.1.1 输出重定向

输出重定向就是，将本应该输出到一个文件的数据重定向输出到另一个文件中

![img](https://pic2.zhimg.com/80/v2-01d4eb5d8782d641ff0ae14c3deed671_720w.webp)

若想让本应该输出到"显示器文件"的数据输出到log.txt文件当中，可以在打开log.txt文件之前将文件描述符为1的文件关闭（即将“显示器文件”关闭）。当我们后续打开log.txt文件时所分配到的文件描述符就是1。

```text
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
int main()
{
	close(1);
	int fd = open("./log.txt",O_RDWR | O_CREAT | O_TRUNC, 0666);
    if(fd < 0){
        perror("opern error:");
        return 1;
    }
    printf("hello world\n");
	fflush(stdout);//为什么需要刷新？阅读后续章节《缓冲区》
	close(fd);
	return 0;
}
```

![img](https://pic2.zhimg.com/80/v2-042054fbdc8ef28fd3a9278f873b0d79_720w.webp)

#### 3.1.2 追加重定向

输出重定向是覆盖式输出数据，而追加重定向是追加式输出数据，并无其他区别

```text
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
int main()
{
    close(1);
    int fd = open("./log.txt",O_WRONLY | O_APPEND);
    if(fd < 0){
        perror("opern error:");
        return 1;
    }
    printf("hello world\n");                                                         
    fflush(stdout);
    return 0;
}
```

![img](https://pic4.zhimg.com/80/v2-4734750c4a2e9417ad5e2e5263381593_720w.webp)

#### 3.1.3 输入重定向

输入重定向，将本应该从一个文件读取数据，现在重定向为从另一个文件读取数据

![img](https://pic3.zhimg.com/80/v2-bfd71feb3c84b87c4c9479a9c5de9d46_720w.webp)

```text
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
int main()
{
    close(0);
    int fd = open("./log.txt",O_RDONLY);
    if(fd < 0){
        perror("opern error:");
        return 1;
    }
    char buf[64] = {0};
    while(scanf("%s",buf) != EOF){
        printf("%s\n",buf);                                                          
    }
    return 0;
}
```

![img](https://pic4.zhimg.com/80/v2-c25b407eb80f017bbf332e40093534c7_720w.webp)

### 3.2 dup2函数

在Linux环境下还可以使用dup2()系统调用来实现重定向。dup2()本质上是通过fd_array数组中地址元素的拷贝完成重定向的。

![img](https://pic1.zhimg.com/80/v2-53977ff6cc642357ef07ba3d5b984594_720w.webp)

```text
#include <unistd.h>
int dup2(int oldfd, int newfd);
```

函数功能： 将fd_array[oldfd]的内容拷贝到fd_array[newfd]当中

函数返回值： 若调用成功则返回newfd，否则返回-1

- 若oldfd不是有效的文件描述符，则dup2调用失败，并且此时文件描述符为newfd的文件没有被关闭。
- 若oldfd是一个有效的文件描述符，但是newfd和oldfd具有相同的值，则dup2不做任何操作并直接返回newfd。

```text
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>
#include <fcntl.h>
int main()
{
	int fd = open("log.txt", O_WRONLY | O_CREAT, 0666);
	if (fd < 0){
		perror("open");
		return 1;
	}
	close(1);
	dup2(fd, 1);
	printf("hello printf\n");
	fprintf(stdout, "hello fprintf\n");
	return 0;
}
```

![img](https://pic2.zhimg.com/80/v2-7392e31b82c2002297bc2c9e5499a65d_720w.webp)

## 4.C语言IO深入底层

相较于C库函数或其他语言的库函数而言，系统调用接口更为底层，语言层的库函数实际上都是对系统接口进行的封装。

在Linux环境下的C库底层使用的就是Linux操作系统提供的接口，在Windows环境下的C库底层使用的便是Windows操作系统给提供的接口，再通过条件编译即可实现跨平台

### 4.1 C语言中默认打开的三个流

前面提到任何进程在运行的时候都会默认打开三个流，即标准输入流、标准输出流以及标准错误流，对应到C语言当中就是stdin、stdout以及stderr。

```text
extern FILE *stdin;
extern FILE *stdout;
extern FILE *stderr;
```

stdin、stdout以及stderr都是属于FILE*类型的，与使用fopen()打开文件时获得的文件指针相同

注意： 不止是C语言当中stdin、stdout和stderr，C++当中也有对应的cin、cout和cerr，其他语言中也有类似的概念。这种特性并不是某种语言所特有的，而是由操作系统所支持的。

**标准输出流和标准错误流对应的都是显示器，它们有什么区别？**

```text
#include <stdio.h>
int main()
{
	printf("hello printf\n"); //stdout
	perror("perror"); //stderr
 
	fprintf(stdout, "stdout:hello fprintf\n"); //stdout
	fprintf(stderr, "stderr:hello fprintf\n"); //stderr
	return 0;
}
```

![img](https://pic3.zhimg.com/80/v2-f4b9daafb77052a1b3603860dff2d206_720w.webp)

从上面的结果来看，好像并没有什么区别，但是将程序运行结果重定向输出到文件log.txt当中后，可以发现log.txt文件当中只有向标准输出流输出的两行字符串，而向标准错误流输出的两行数据并没有重定向到文件当中，而是仍然输出到了显示器上。

![img](https://pic2.zhimg.com/80/v2-e8d1fdb55fd1dae7b793234aef28f4bd_720w.webp)

使用命令进行重定向时，默认重定向的是文件描述符为1的标准输出流，而并不会对文件描述符为2的标准错误流进行重定向。当然也可以显示指定重定向标准错误流。

![img](https://pic2.zhimg.com/80/v2-523b537f56cdcc2710799993b0805f61_720w.webp)

### 4.2 FILE

因为库函数是对系统调用接口的封装，本质上访问文件都是通过文件描述符fd进行访问的，所以C库当中的FILE结构体内部必定封装了文件描述符fd。

在/usr/include/stdio.h头文件中可以看到下面这句代码，也就是说FILE实际上就是struct _IO_FILE结构体的一个别名。

```text
typedef struct _IO_FILE FILE;
```

而在`/usr/include/libio.h`头文件中可以找到`struct _IO_FILE`结构体的定义，在该结构体的众多成员当中，我们可以看到一个名为`_fileno`的成员，这个成员实际上就是封装的文件描述符。

```text
struct _IO_FILE {
	int _flags;       /* High-order word is _IO_MAGIC; rest is flags. */
#define _IO_file_flags _flags
 
	//缓冲区相关
	/* The following pointers correspond to the C++ streambuf protocol. */
	/* Note:  Tk uses the _IO_read_ptr and _IO_read_end fields directly. */
	char* _IO_read_ptr;   /* Current read pointer */
	char* _IO_read_end;   /* End of get area. */
	char* _IO_read_base;  /* Start of putback+get area. */
	char* _IO_write_base; /* Start of put area. */
	char* _IO_write_ptr;  /* Current put pointer. */
	char* _IO_write_end;  /* End of put area. */
	char* _IO_buf_base;   /* Start of reserve area. */
	char* _IO_buf_end;    /* End of reserve area. */
	/* The following fields are used to support backing up and undo. */
	char *_IO_save_base; /* Pointer to start of non-current get area. */
	char *_IO_backup_base;  /* Pointer to first valid character of backup area */
	char *_IO_save_end; /* Pointer to end of non-current get area. */
 
	struct _IO_marker *_markers;
 
	struct _IO_FILE *_chain;
 
	int _fileno; //封装的文件描述符
#if 0
	int _blksize;
#else
	int _flags2;
#endif
	_IO_off_t _old_offset; /* This used to be _offset but it's too small.  */
 
#define __HAVE_COLUMN /* temporary */
	/* 1+column number of pbase(); 0 is unknown. */
	unsigned short _cur_column;
	signed char _vtable_offset;
	char _shortbuf[1];
 
	/*  char* _save_gptr;  char* _save_egptr; */
 
	_IO_lock_t *_lock;
#ifdef _IO_USE_OLD_IO_FILE
};
```

**C库文件IO函数的具体流程理解**

例：fopen函数在上层为用户申请FILE结构体变量，并返回该结构体的地址(FILE*)。在底层通过系统接口open打开对应的文件，得到文件描述符fd，并把fd填充到FILE结构体当中的_fileno变量中，至此便完成了文件的打开操作。

而C语言当中的其他文件操作函数，比如fread、fwrite、fputs、fgets等，都是先根据我们传入的文件指针找到对应的FILE结构体，然后在FILE结构体当中找到文件描述符，最后通过文件描述符对文件进行的一系列操作。

### 4.3 用户层缓冲区

```text
#include <stdio.h>
#include <unistd.h>
int main()
{
	printf("hello printf\n");
	fputs("hello fputs\n", stdout);
	write(1, "hello write\n", 12);
	fork();
	return 0;
}
```

![img](https://pic3.zhimg.com/80/v2-7a7e3bf34c8ea9ff1f0c8ce9bbe3f09a_720w.webp)

查看上述代码，貌似并没有什么奇特的地方，甚至这个fork()十分的多余。但是将程序执行结果重定向到log.txt之后会得到这种情况。

![img](https://pic4.zhimg.com/80/v2-b19eb5180d1ede77f8cb80175199527f_720w.webp)

用户层的缓冲区，通常有以下三种缓冲策略:

- 无缓冲
- 行缓冲。（常见于对显示器进行刷新数据）
- 全缓冲。（常见于对磁盘文件写入数据）

当直接执行可执行程序，将数据打印到显示器时所采用的就是行缓冲，又因为代码当中每个字符串后都有'\n'，所以当执行完对应代码后就立即将数据刷新到了显示器上。

但是将运行结果重定向到log.txt文件后，用户层缓冲区的刷新策略就变为了全缓冲，此时使用printf和fputs函数打印的数据都进入了C语言的缓冲区中，之后当我们使用fork函数创建子进程时，由于进程间具有独立性，而之后当父进程或是子进程对要刷新缓冲区内容时，本质就是对父子进程共享的数据进行了修改，此时就需要对数据进行写时拷贝，至此缓冲区当中的数据就变成了两份，一份父进程的，一份子进程的，所以重定向到log.txt文件当中printf和puts函数打印的数据就有两份。但由于write函数是系统接口，不存在用户层缓冲区，因此write函数打印的数据就只有一份。

**用户层缓冲区由谁提供？在哪？**

由C语言库实现提供。用户层的输出缓冲区就存在stdout中，stdout是一个FILE*的指针。在FILE结构体当中还有一大部分成员是用于记录缓冲区相关的信息的。

```text
//缓冲区相关
/* The following pointers correspond to the C++ streambuf protocol. */
/* Note:  Tk uses the _IO_read_ptr and _IO_read_end fields directly. */
char* _IO_read_ptr;   /* Current read pointer */
char* _IO_read_end;   /* End of get area. */
char* _IO_read_base;  /* Start of putback+get area. */
char* _IO_write_base; /* Start of put area. */
char* _IO_write_ptr;  /* Current put pointer. */
char* _IO_write_end;  /* End of put area. */
char* _IO_buf_base;   /* Start of reserve area. */
char* _IO_buf_end;    /* End of reserve area. */
```

**操作系统是否存在缓冲区？**

操作系统实际上也是存在缓冲区的，当刷新用户缓冲区的数据时，并不是直接将用户缓冲区的数据刷新到磁盘或是显示器上，而是先将数据刷新到操作系统缓冲区，然后再由操作系统将数据刷新到磁盘或是显示器上。（操作系统有自己的刷新机制，我们不必关系操作系统缓冲区的刷新规则）

## 5.理解"一切皆文件"思想

"一切皆文件"是Linux的设计哲学，体现在os的软件设计层面。

Linux是C语言编写的，但是如何使用C语言实现面向对象？

面向对象语言中的类大致由成员属性和成员方法构成，C语言中可以包含成员属性但是并不可以包含成员方法，但是可以使用函数指针指向函数来代替成员方法。

![img](https://pic3.zhimg.com/80/v2-85e21378b5830bb8d411bf78d6a96756_720w.webp)

通过这个设计方式，消除了上层处理硬件的差异，统一了看待文件的方式，一切皆文件。

## 6.文件系统

文件可以分为内存文件和磁盘文件，之前谈论的都是内存文件，接下开始讲解磁盘文件

### 6.1 认识inode

磁盘文件由两部分构成，分别是文件内容和文件属性（都存储在磁盘中）。文件内容就是文件当中存储的数据，文件属性就是文件的一些基本信息，例如文件名、文件大小以及文件创建时间等信息都是文件属性，文件属性又被称为元信息。

![img](https://pic1.zhimg.com/80/v2-99d0577506b6bc4259fc5e64891e33ac_720w.webp)

在Linux操作系统中，文件的元信息和内容是分离存储的，其中保存元信息的结构被称之为inode。因为系统当中存在大量的文件，所以需要给每个文件的属性集起一个唯一的编号，即inode号。即inode是一个文件的属性集合，Linux中每个文件都对应一个inode，为了区分系统中大量inode，我们为每个inode设置了inode编号。

![img](https://pic3.zhimg.com/80/v2-ac113825fcd80ff72dd73723cf046be6_720w.webp)

**ls**命令添加 **-i** 选项即可查看文件的inode编号

### 6.2 磁盘概念

内存是掉电易失存储介质，而磁盘是一种永久性存储介质。在计算机中，磁盘几乎是唯一的机械设备。磁盘在冯诺依曼体系结构当中既可以充当输入设备，又可以充当输出设备。

![img](https://pic2.zhimg.com/80/v2-53e8cd228ca9d0f242a7a5c4edb7caf5_720w.webp)

**磁盘寻址方案(CHS寻址)**

1. 确定读写信息在磁盘的哪个盘面
2. 确定读写信息在磁盘的哪个柱面(磁道)
3. 确定读写信息在磁盘的哪个扇区

### 6.3 磁盘分区与格式化

若要理解文件系统，先将磁盘想象成一个线性的存储介质。譬如磁带，当磁带被卷起来时就像磁盘一样是圆形的，但当将磁带拉直后就是线性的。

#### **6.3.1 磁盘分区**

磁盘通常被称为块设备，一般以扇区为单位，一个扇区的大小通常为512字节。计算机为了更好的管理磁盘，对磁盘进行了分区。磁盘分区就是使用分区编辑器在磁盘上划分几个逻辑部分，盘片一旦划分成数个分区，不同的目录与文件就可以存储进不同的分区。分区越多，就可以将文件的性质区分得越细，按照更为细分的性质，存储在不同的地方以管理文件。譬如在Windows操作系统下磁盘一般被分为C盘和D盘两个区域。

![img](https://pic1.zhimg.com/80/v2-8d83839aa4735187c0ba142af893b0cc_720w.webp)

#### 6.3.2 磁盘格式化

当磁盘完成分区后，还需对磁盘进行格式化。磁盘格式化就是对磁盘中的分区进行初始化的一种操作，这种操作通常会导致现有的磁盘或分区中所有的文件被清除。其实磁盘格式化就是对分区后的各个区域写入对应的管理信息。

写入的管理信息是什么是由文件系统决定的，不同的文件系统格式化时写入的管理信息是不同的，常见的文件系统有EXT2、EXT3、XFS、NTFS等。

### 6.4 EXT2文件系统的存储方案

对于每一个分区来说，分区的头部会包括一个启动块(Boot Block)，对于该分区的其余区域，EXT2文件系统会根据分区的大小将其划分为一个个的块组(Block Group)

注意: 启动块的大小是确定的，而块组的大小是由格式化的时候确定的，且不可随意更改

![img](https://pic4.zhimg.com/80/v2-27fa159d83e14d2007c0aed09af12c97_720w.webp)

而每个组块都有着相同的组成结构，每个组块都由超级块(Super Block)、块组描述符表(Group Descriptor Table)、块位图(Block Bitmap)、inode位图(inode Bitmap)、inode表(inode Table)以及数据表(Data Block)组成

![img](https://pic1.zhimg.com/80/v2-0434039f59fa7664f0c09fc3fee473b0_720w.webp)

1. Super Block： 存放文件系统本身的结构信息。记录的信息主要有：Data Block和inode的总量、未使用的Data Block和inode的数量、一个Data Block和inode的大小、最近一次挂载的时间、最近一次写入数据的时间、最近一次检验磁盘的时间等其他文件系统的相关信息。Super Block的信息被破坏，可以说整个文件系统结构就被破坏了。
2. Group Descriptor Table： 块组描述符表，描述该分区当中块组的属性信息
3. Block Bitmap： 块位图当中记录着Data Block中数据块是否被占用
4. inode Bitmap： inode位图当中记录着每个inode是否空闲可用
5. inode Table： 存放文件属性，即每个文件的inode
6. Data Blocks： 存放文件内容

**注意：**

1. 组块中会存在冗余的Super Block，当某个Super Block被破坏后可以通过该块组中的其他Super Block进行恢复
2. 磁盘分区并格式化后，每个分区的inode个数就确定了
3. 一个文件使用的数据块和inode结构的对应关系，通过一个数组进行维护。该数组一般可以存储15个元素，其中前12个元素分别对应该文件使用的12个数据块，剩余的三个元素分别是一级索引、二级索引和三级索引。当该文件使用数据块的个数超过12个时，可以用这三个索引进行数据块扩充

**如何创建一个文件？**

1. 通过遍历inode位图的方式，找到一个空闲的inode
2. 在inode表当中找到对应的inode，并将文件的属性信息填充进inode结构中
3. 将该文件的文件名和inode指针添加到目录文件的数据块中

**如何对文件写入信息？**

1. 通过文件的inode编号找到对应的inode结构
2. 通过inode结构找到存储该文件内容的数据块，并将数据写入数据块
3. 若不存在数据块或申请的数据块已被写满，则通过遍历块位图的方式找到一个空闲的块号，并在数据区当中找到对应的空闲块，再将数据写入数据块，最后还需要建立数据块和inode结构的映射关系

**如何删除文件？**

1. 将该文件对应的inode在inode位图当中置为无效。
2. 将该文件申请过的数据块在块位图当中置为无效。

此操作并不会真正将文件对应的信息删除，而只是将其inode号和数据块号置为了无效，所以当删除文件后短时间内是可以恢复的。但后续创建其他文件或是对其他文件进行写入操作申请数据块号时，可能会将该置为无效了的inode号和数据块号分配出去，此时删除文件的数据就会被覆盖，就无法恢复文件了。

为什么拷贝文件较慢，而删除文件较快呢？

因为拷贝文件需要先创建文件，然后再对该文件进行写入操作，该过程需要先申请inode号并填入文件的属性信息，之后还需要再申请数据块号，最后才能进行文件内容的数据拷贝，而删除文件只需将对应文件的inode号和数据块号置为无效即可，无需真正的删除文件。

对目录的认识

1. Linux中一切皆文件，目录也被看作是文件
2. 目录有自己的属性信息，目录的inode结构当中存储的就是目录的属性信息
3. 目录的数据块当中存储的就是该目录下的文件名以及对应文件的inode指针

### 6.5 软硬链接

#### 6.5.1 软链接

Linux环境下的软链接就类似与Windows环境下的快捷方式

![img](https://pic3.zhimg.com/80/v2-bf0d7acacbc52d78b472356a7b848cf6_720w.webp)

软链接又称符号链接，软链接文件相对于源文件来说是一个独立的文件，该文件有自己的inode号，但是该文件包含源文件的路径名。

但软链接文件只是其源文件的一个标记，当删除了源文件后，链接文件不能独立存在，虽然仍保留文件名，但却不能执行或是查看软链接的内容了。

#### 6.5.2 硬链接

硬链接文件的inode号与源文件的inode号是相同的，并且硬链接文件的大小与源文件的大小也是相同的。且当创建了一个硬链接文件后，该硬链接文件和源文件的硬链接数都变成了2。

![img](https://pic1.zhimg.com/80/v2-795aa1858ae3e334172813af9e6cd258_720w.webp)

硬链接文件就是源文件的一个别名，一个文件有几个文件名，该文件的硬链接数就是几。且当硬链接的源文件被删除后，硬链接文件仍能正常执行，只是文件的链接数减少了一个，因为此时该文件的文件名少了一个。

**为什么新建目录的的硬链接数是2？**

![img](https://pic4.zhimg.com/80/v2-49e81007a20825a6beee1b308d380537_720w.webp)

因为每个目录创建后，该目录下默认会有两个隐含文件.和..，分别代表当前目录和上级目录。因此这里创建的目录有两个名字，一个是dir另一个就是该目录下的.，所以刚创建的目录硬链接数是2。

![img](https://pic1.zhimg.com/80/v2-417c418f50fca1216d6935fc60720130_720w.webp)

通过上图也可以看出dir和该目录下的`.`的inode号相同，也可以说明它们代表的实际是同一个文件

原文地址：https://zhuanlan.zhihu.com/p/602871471

作者：linux