# 【NO.563】linux下wait/waitpid处理僵死进程详解(SIGCHLD信号)

## 1.僵尸进程处理

客户正常断开但服务器未处理SIGCHLD信号，会使得服务器子进程僵死。

**设置僵尸进程的目的：**

维护子进程信息，以便父进程在以后某时候获取。信息包括子进程ID，终止状态，资源利用信息（CPU时间、内存使用量等）。如果某进程结束，而该进程有子进程处于僵死状态，那么它的所有僵死子进程的父进程ID会变为1（init进程），init进程会清理它们，即init进程会wait它们，去除其僵死状态。

**处理僵死进程方法：**

（1）忽略SIGCHLD信号可以防止僵尸进程，在server端main函数listen之后添加： signal(SIGCHLD, SIG_IGN);

如下是代码：

```text
//使用sigaction函数取代unix早期版本的signal函数的实现
typedef void Sigfunc(int);
Sigfunc *
Signal(int signo, Sigfunc *func)
{
   struct sigaction act, oact;
   act.sa_handler = func;
   sigemptyset(&act.sa_mask);
   act.sa_flags = 0;
   if (signo == SIGALRM) {
#ifdef SA_INTERRUPT
      act.sa_flags |= SA_INTERRUPT;
#endif
   }
   else {
#ifdef SA_RESTART  //如果系统可以重启被中断的系统调用的话，被中断的系统调用将由内核重启
      act.sa_flags |= SA_RESTART;
#endif
   }
   if (sigaction(signo, &act, &oact) < 0)
      return(SIG_ERR);
   return(oact.sa_handler);
}
```

（2）通过wait/waitpid方法。

```text
//使用waitpid的信号处理函数
void onSignalCatch(int signalNumber)
{
    pid_t pid;
    pid = wait(NULL);
    printf("child %d terminated.\n", pid);
    return;
 }
```

在服务器端listen调用后加入：

```text
Signal(SIGCHLD, onSignalCatch); //这个处理信号的函数必须在fork第一个子进程之前完成，且仅做一次
```

再次执行程序，三个客户端正常终止后，服务器端结果如下：

![img](https://pic1.zhimg.com/80/v2-de1c996f30f6d0a093a4b0cef2099314_720w.webp)

上述的方法调用wait的函数onSignalCatch作为信号处理函数，其正常结束的过程如下：

（1）键入EOF终止客户。客户TCP发送FIN给服务器，服务器响应ACK。

（2）收到客户FIN的服务器TCP发送EOF给子进程阻塞的read，子进程结束。

（3）当SIGCHLD信号递交时，父进程阻塞于accept调用。信号处理函数onSignalCatch执行，其wait调用子进程pid并打印、返回。

（4）信号是父进程在**阻塞于慢系统调用accept时捕获**的，所以，内核就会使accept返回EINTR错误（被中断的系统调用）。而**父进程不处理该错误，被中断的系统调用重启（上述自定义的Signal函数中：设置了SA_RESTART标志），所以accept没返回错误**。

## 2.处理被中断的系统调用

accept被称为**慢系统调用**，多数网络支持函数都属于这一类。**适用于慢系统调用的基本规则是**：当阻塞于某个慢系统调用的进程捕获某个信号且相应信号处理函数返回后，该系统调用可能返回一个EINTR错误。

从上图可以看出，三次客户的终止都没有使得阻塞于accept的服务器终止；因为内核会自动重启被中断的系统调用。而有些系统的标准C函数库中的signal函数不会使得内核自动重启被中断的系统调用，也就是SA_RESTART标志在系统函数的signal函数中没有被设置，在这些系统中服务器程序将终止。
那么并非所有的系统都会将被中断的系统调用重启，所以要处理被中断的系统调用，将accept的调用改为：

```text
if ((connfd = accept(listenfd, (struct sockaddr*) &cliaddr, &clilen)) < 0) {
          if (errno == EINTR)
             continue;  //有些系统不会重启被这中断的系统调用，所以要处理
          else
             err_exit("accept error, server.\n");
      }
```

## 3.wait函数和waitpid函数

```text
#include <sys/wait.h>
pid_t wait(int *statloc);
pid_t waitpid(pid_t pid, int *statloc, int options);  //参数pid允许指定想等待的进程ID，为-1表示等待第一个终止的子进程
//均返回：成功返回进程ID，出错返回0或-1
//statloc指针返回子进程终止状态（一个整数）
//options是附加选项，最常用的是WNOHANG，告知内核在有尚未终止的子进程在运行时不阻塞
```



![img](https://pic3.zhimg.com/80/v2-95785cf20abd74c14b8f278e1f24cbb2_720w.webp)



![img](https://pic4.zhimg.com/80/v2-dba55148256ccc6c8e9777b16a838de3_720w.webp)

**问题：**

在上述情况下，建立信号处理函数在其中调用wait仍然会有僵死进程。问题在于：所有5个信号都在信号处理函数执行前产生；而在一台机器上运行客户和服务器端，信号处理函数只执行一次，因为unix信号一般是不排队的。如果在不同机器上运行客户端服务器端，信号处理函数执行次数不确定，对于留下几个僵死进程以及哪几个会僵死都是不确定的。

如下图分别为创建的5个连接、客户端exit后服务端的四个僵死子进程：

![img](https://pic3.zhimg.com/80/v2-8fe7edbd31b9c3812bdb7f588cc564a2_720w.webp)

![img](https://pic4.zhimg.com/80/v2-544b8793eb8828bcafa853aab5a27b67_720w.webp)

**解决方法：**

使用waitpid而不是wait。如下的处理函数管用，因为在一个循环内调用waitpid，以获取进程终止状态。**waitpid函数的参数options指定为WNOHANG**，告知waitpid在尚有未终止的子进程时，不阻塞。而使用wait无法阻止其在还有子进程未结束时阻塞。

```text
//使用waitpid的信号处理函数
void onSignalCatch(int signalNumber)
{
    pid_t pid;
    int stat;
    //pid = wait(&stat);
    //下列函数的第一个参数为－1表示等待第一个终止的子进程
    while ((pid = waitpid(-1, &stat, WNOHANG)) > 0) 
        printf("child %d terminated.\n", pid);
    return;
 }
```

结果如下：

![img](https://pic3.zhimg.com/80/v2-31fed25821890af77f75cc25349374b6_720w.webp)

![img](https://pic3.zhimg.com/80/v2-28ab44b7b1e169a8c2d02047b48314d6_720w.webp)

至此，客户服务器代码就完成了，加入了对僵死进程的处理。

原文地址：https://zhuanlan.zhihu.com/p/272700092

作者：linux