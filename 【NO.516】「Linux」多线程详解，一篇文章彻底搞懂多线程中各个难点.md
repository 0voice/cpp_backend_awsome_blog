# 【NO.516】「Linux」多线程详解，一篇文章彻底搞懂多线程中各个难点

## 1.什么是线程？

linux内核中是没有线程这个概念的，而是轻量级进程的概念：LWP。一般我们所说的线程概念是C库当中的概念。

### 1.1线程是怎样描述的？

线程实际上也是一个task_struct，工作线程拷贝主线程的task_struct，然后共用主线程的mm_struct。线程ID是在用task_struct中pid描述的，而task_struct中tgid是线程组ID，表示线程属于该线程组，对于主线程而言，其pid和tgid是相同的，我们一般看到的进程ID就是tgid。

即:

![img](https://pic4.zhimg.com/80/v2-42239c9b5aca18ac66aa9a78bb8921eb_720w.webp)

获取线程ID和主线程ID的值：

![img](https://pic3.zhimg.com/80/v2-1d7656a1fea373626911b29e60e28afe_720w.webp)

但是获取该gettid系统调用接口并没有被封装起来，如果确实需要获取线程ID，可使用：

```text
#include <sys/syscall.h>
int TID = syscall(SYS_gettid);
```

则对线程组而言，所有的tgid一定是一样的，所有的pid一定是不一样的。主线程pid和tgid一样，工作线程pid和tgid一定不一样。

### 1.2如何查看一个线程的ID

命令：ps -eLf

![img](https://pic1.zhimg.com/80/v2-ce6286777e5867197ad60f6cbf6f6cc4_720w.webp)

上述polkitd进程是多线程的，进程ID为731，进程内有6个线程，线程ID为731，764，765，768，781，791。

![img](https://pic1.zhimg.com/80/v2-5ac5511da2bad62578cfbf3d29d146f8_720w.webp)

### 1.3多线程如何避免调用栈混乱的问题？

工作线程和主线程共用一个mm_struct，如果都向栈中压栈，必然会导致调用栈出错。

实际上工作线程压栈是压了共享区，该共享区包含了许多线程独有的资源。如图：

![img](https://pic1.zhimg.com/80/v2-6ad430eaa460730b01d8c833f1530e48_720w.webp)

每一个线程，默认在共享区中占有的空间为8M，可以使用ulimit -s修改。

进程是资源分配的基本单位，线程是调度的基本单位。

**1.3.1线程独有资源**

- 线程ID
- 一组寄存器
- errno
- 信号屏蔽字
- 调度优先级

**1.3.2线程共享资源和环境**

- 文件描述符表
- 信号的处理方式
- 当前工作目录
- 用户id和组id

### 1.4为什么要有多线程？

举个生活中的例子， 这就好比去银行办理业务。 到达银行后， 首先取一个号码， 然后坐下来安心等待。 这时候你一定希望， 办理业务的窗口越多越好。 如果把整个营业大厅当成一个进程的话， 那么每一个窗口就是一个工作线程。

**1.4.1线程带来的优势**

1、线程会共享内存地址空间。

2、创建线程花费的时间要少于创建进程花费的时间。

3、终止线程花费的时间要少于终止进程花费的时间。

4、线程之间上下文切换的开销， 要小于进程之间的上下文切换。

5、线程之间数据的共享比进程之间的共享要简单。

6、充分利用多处理器的可并行数量。（线程会提高运行效率，但当线程多到一定程度后，可能会导致效率下降，因为会有线程调度切换。）

**1.4.2线程带来的缺点**

健壮性降低：多个线程之中， 只要有一个线程不够健壮存在bug（如访问了非法地址引发的段错误） ， 就会导致进程内的所有线程一起完蛋。

线程模型作为一种并发的编程模型， 效率并没有想象的那么高， 会出现复杂度高、 易出错、 难以测试和定位的问题。

### 1.5注意

1、并不是只有主线程才能创建线程， 被创建出来的线程同样可以创建线程。

2、不存在类似于fork函数那样的父子关系， 大家都归属于同一个线程组， 进程ID都相等， group_leader都指向主线程， 而且各有各的线程ID。

> 通过group_leader指针， 每个线程都能找到主线程。 主线程存在一个链表头，后面创建的每一个线程都会链入到该双向链表中。

3、并非只有主线程才能调用pthread_join连接其他线程， 同一线程组内的任意线程都可以对某线程执行pthread_join函数。

4、并非只有主线程才能调用pthread_detach函数， 其实任意线程都可以对同一线程组内的线程执行分离操作。

线程的对等关系：

![img](https://pic3.zhimg.com/80/v2-730890e52179ce98154ba36b39f50f6e_720w.webp)

## 2.线程创建

> 接口：int pthread_create(pthread_t *thread, const pthread_attr_t *attr,void *(*start_routine) (void *), void *arg);
> 参数解释
> 1、thread：线程标识符，是一个出参
> 2、attr：线程属性
> 3、star_routine：函数指针，保存线程入口函数的地址
> 4、arg：给线程入口函数传参
> 返回值：成功返回0，失败返回error number

详解：

第一个参数是pthread_t类型的指针， 线程创建成功的话，会将分配的线程ID填入该指针指向的地址。 线程的后续操作将使用该值作为线程的唯一标识。

第二个参数是pthread_attr_t类型， 通过该参数可以定制线程的属性， 比如可以指定新建线程栈的大小、 调度策略等。 如果创建线程无特殊的要求， 该值也可以是NULL， 表示采用默认属性。

第三个参数是线程需要执行的函数。 创建线程， 是为了让线程执行一定的任务。 线程创建成功之后， 该线程就会执行start_routine函数， 该函数之于线程， 就如同main函数之于主线程。

第四个参数是新建线程执行的start_routine函数的入参。

pthread_create错误码及描述：

![img](https://pic3.zhimg.com/80/v2-12fb1014e55a6338a270e154644393ee_720w.webp)

### 2.1传入参数arg的选择

![img](https://pic2.zhimg.com/80/v2-b2448de22126e5c43e8aa732739368d9_720w.webp)

不要使用临时变量传参，使用堆上开辟的变量可以。

例：

```text
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>
void *ThreadWork(void *arg)
{
  int *p = (int*)arg;
  printf("i am work thread:%p,   data:%d\n",pthread_self(),*p);
  pthread_exit(NULL);
}
int main()
{
  int i = 1;
  pthread_t tid;
  int ret = pthread_create(&tid,NULL,ThreadWork,(void*)&i);//不要传临时变量，这里是示范
  if(ret != 0)
  {
    perror("pthread_create");
    return -1;
  }
  while(1)
  {
    printf("i am main work thread\n");
    sleep(1);
  }
  return 0;
}
```

### 2.2线程ID以及进程地址空间

线程获取自身的ID：

```text
#include <pthread.h>
pthread_t pthread_self(void);
```

判断两个线程ID是否对应着同一个线程：

```text
#include <pthread.h>
int pthread_equal(pthread_t t1, pthread_t t2);
```

返回为0时，则表示两个线程为同一个线程，非0时，表示不是同一个线程。

用户调用pthread_create函数时， 首先要为线程分配线程栈， 而线程栈的位置就落在共享区。 调用mmap函数为线程分配栈空间。 pthread_create函数分配的pthread_t类型的线程ID， 不过是分配出来的空间里的一个地址， 更确切地说是一个结构体的指针。

![img](https://pic4.zhimg.com/80/v2-45e85ef60e7c502b2aaff4c61062f4ab_720w.webp)

即：

![img](https://pic4.zhimg.com/80/v2-f359fdf490a2ac0e09f3ea969674c5d7_720w.webp)

### 2.3线程注意点

1、线程ID是进程地址空间内的一个地址， 要在同一个线程组内进行线程之间的比较才有意义。 不同线程组内的两个线程， 哪怕两者的pthread_t值是一样的， 也不是同一个线程。

2、线程ID就有可能会被复用：

> 1、线程退出。
> 2、线程组的其他线程对该线程执行了pthread_join， 或者线程退出前将分离状态设置为已分离。
> 3、再次调用pthread_create创建线程。

### 2.4线程创建出来的默认值

线程创建的第二个参数是pthread_attr_t类型的指针， pthread_attr_init函数会将线程的属性重置成默认值。

线程属性及默认值：

![img](https://pic3.zhimg.com/80/v2-929b34ff679edc7ab04c6a4805f1f2c2_720w.webp)

如果确实需要很多的线程， 可以调用接口来调整线程栈的大小：

```text
#include <pthread.h>
int pthread_attr_setstacksize(pthread_attr_t *attr,size_t stacksize);
int pthread_attr_getstacksize(pthread_attr_t *attr,size_t *stacksize);
```

## 3.线程终止

线程终止，但进程不会终止的方法：

1、入口函数的return返回，线程就退出了

2、线程调用pthread_exit(NULL)，谁调用谁退出

> \#include <pthread.h>
> void pthread_exit(void *retval);
> 参数：retval是返回信息，”临终遗言“，可以给可以不给
> 该变量不能使用临时变量。
> 可使用：全局变量、堆上开辟的空间、字符串常量。

pthread_exit和线程启动函数（start_routine） 执行return是有区别的。 在start_routine中调用的任何层级的函数执行pthread_exit（） 都会引发线程退出， 而return， 只能是在start_routine函数内执行才能导致线程退出。

3、其它线程调用了pthread_cancel函数取消了该线程

> int pthread_cancel(pthread_t thread);
> thread：线程标识符
> 调用该函数的执行流可以取消其它线程，但是需要知道其它线程的线程标识符，也可以执行流自己取消自己，传入自己的线程标识符。

如果线程组中的任何一个线程调用了exit函数， 或者主线程在main函数中执行了return语句， 那么整个线程组内的所有线程都会终止。

## 4.线程等待

### 4.1线程等待接口

```text
#include <pthread.h>
int pthread_join(pthread_t thread, void **retval);
```

![img](https://pic2.zhimg.com/80/v2-af6806899576fa7e7dc672830659aa25_720w.webp)

调用该函数，该执行流在等待线程退出的时候，该执行流是阻塞在pthread_joind当中的。

### 4.2线程等待和进程等待的不同

第一点不同之处是进程之间的等待只能是父进程等待子进程， 而线程则不然。线程组内的成员是对等的关系， 只要是在一个线程组内， 就可以对另外一个线程执行连接（join） 操作。

第二点不同之处是进程可以等待任一子进程的退出 ， 但是线程的连接操作没有类似的接口， 即不能连接线程组内的任一线程， 必须明确指明要连接的线程的线程ID。

pthread_join()错误码：

![img](https://pic3.zhimg.com/80/v2-305dba3a6243881444f5e677940b07d6_720w.webp)

### 4.3为什么要等待退出的线程？

如果不连接已经退出的线程， 会导致资源无法释放。 所谓资源指的又是什么呢？

1、已经退出的线程， 其空间没有被释放， 仍然在进程的地址空间之内。

2、新创建的线程， 没有复用刚才退出的线程的地址空间。

如果不执行连接操作， 线程的资源就不能被释放， 也不能被复用， 这就造成了资源的泄漏。

纵然调用了pthread_join， 也并没有立即调用munmap来释放掉退出线程的栈， 它们是被后建的线程复用了。 释放线程资源的时候， 若进程可能再次创建线程， 而频繁地munmap和mmap会影响性能， 所以将该栈缓存起来， 放到一个链表之中， 如果有新的创建线程的请求， 会首先在栈缓存链表中寻找空间合适的栈， 有的话， 直接将该栈分配给新创建的线程。

例：

```text
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>
#include <sys/syscall.h>


void *ThreadWork(void *arg)
{
  int *p = (int*)arg;
  printf("pid :  %d\n",syscall(SYS_gettid));
  printf("i am work thread:%p,   data:%d\n",pthread_self(),*p);
  sleep(3);
  pthread_exit(NULL);
}

int main()
{
  int i = 1;
  pthread_t tid;
  int ret = pthread_create(&tid,NULL,ThreadWork,(void*)&i);//不要传临时变量，这里是示范
  if(ret != 0)
  {
    perror("pthread_create");
    return -1;
  }
  pthread_join(tid,NULL);//线程等待
  while(1)
  {
    printf("i am main work thread\n");
    sleep(1);
  }
  return 0;
}
```

## 5.线程分离

```text
接口：#include <pthread.h>
int pthread_detach(pthread_t thread);
```

默认情况下， 新创建的线程处于可连接（Joinable） 的状态， 可连接状态的线程退出后， 需要对其执行连接操作， 否则线程资源无法释放， 从而造成资源泄漏。

如果其他线程并不关心线程的返回值， 那么连接操作就会变成一种负担： 你不需要它， 但是你不去执行连接操作又会造成资源泄漏。 这时候你需要的东西只是：线程退出时， 系统自动将线程相关的资源释放掉， 无须等待连接。

可以是线程组内其他线程对目标线程进行分离， 也可以是线程自己执行pthread_detach函数。

线程的状态之中， 可连接状态和已分离状态是冲突的， 一个线程不能既是可连接的， 又是已分离的。 因此， 如果线程处于已分离的状态， 其他线程尝试连接线程时， 会返回EINVAL错误。

pthread_detach错误码：

![img](https://pic1.zhimg.com/80/v2-0b138e4679b2e1a102f4aa30a393c2dc_720w.webp)

注意：这里的已分离不是指线程失去控制，不归线程组管，而是指线程退出后，系统会自动释放线程资源。若是线程组内的任意线程执行了exit函数，即使是已分离的线程，也仍会收到影响，一并退出。

## 6.线程安全

线程安全中涉及到的概念：

```text
临界资源：多线程中都能访问到的资源
临界区：每个线程内部，访问临界资源的代码，就叫临界区
```

### 6.1什么是线程不安全？

多个线程访问同一块临界资源，导致资源产生二义性的现象。

**6.1.1举一个例子**

- 假设现在有两个线程A和B，单核CPU的情况下，此时有一个int类型的全局变量为100，A和B的入口函数都要对这个全局变量进行–操作。
- 线程A先拿到CPU资源后，对全局变量进行–操作并不是原子性操作，也就是意味着，A在执行–的过程中有可能会被打断。假设A刚刚将全局变量的值读到寄存器当中，就被切换出去了，此时程序计数器保存了下一条执行的指令，上下文信息保存寄存器中的值，这两个东西是用来线程A再次拿到CPU资源后，恢复现场使用的。
- 此时，线程B拿到了CPU资源，对全局变量进行了–操作，并且将100减为了99，回写到了内存中。
- A再次拥有了CPU资源后，恢复现场，继续往下执行，从寄存器中读到的值仍为100，减完之后为99，回写到内存中为99。

上述例子中，线程A和B都对全局变量进行了–操作，全局变量的值应该变为98，但程序现在实际的结果为99，所以这就导致了线程不安全。

### 6.2如何解决线程不安全现象?

解决方案只需做到下述三点即可：

1、代码必须要有互斥的行为： 当一个线程正在临界区中执行时， 不允许其他线程进入该临界区中。

2、如果多个线程同时要求执行临界区的代码， 并且当前临界区并没有线程在执行， 那么只能允许一个线程进入该临界区。

3、如果线程不在临界区中执行， 那么该线程不能阻止其他线程进入临界区。

则本质上，我们需要对该临界区加一把锁：

![img](https://pic4.zhimg.com/80/v2-e270fae0677754a8f7c902b3e1b5d6f3_720w.webp)

锁是一个很普遍的需求， 当然用户可以自行实现锁来保护临界区。 但是实现一个正确并且高效的锁非常困难。 纵然抛下高效不谈， 让用户从零开始实现一个正确的锁也并不容易。 正是因为这种需求具有普遍性， 所以Linux提供了互斥量。

### 6.3互斥量接口

**6.3.1互斥量的初始化**

1、静态分配：

```text
#include <pthread.h>
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
```

2、动态分配：

```text
int pthread_mutex_init(pthread_mutex_t *restrict mutex,const pthread_mutexattr_t *restrict attr);
```

![img](https://pic3.zhimg.com/80/v2-224580e4d8cec26b47aa880ef556a516_720w.webp)

调用int pthread_mutex_init（）函数后，互斥量是处于没有加锁的状态。

**6.3.2互斥量的销毁**

```text
int pthread_mutex_destroy(pthread_mutex_t *mutex);
```

注意：

1、使用PTHREAD_MUTEX_INITIALIZER初始化的互斥量无须销毁。

2、不要销毁一个已加锁的互斥量， 或者是真正配合条件变量使用的互斥量。

3、已经销毁的互斥量， 要确保后面不会有线程再尝试加锁。

当互斥量处于已加锁的状态， 或者正在和条件变量配合使用， 调用pthread_mutex_destroy函数会返回EBUSY错误码。

**6.3.3互斥量的加锁**

```text
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_timedlock(pthread_mutex_t *restrict mutex, const struct timespec *restrict abs_timeout);
```

第一个接口：int pthread_mutex_lock(pthread_mutex_t *mutex);

1、该接口是阻塞加锁接口。

2、mutex为传入互斥锁变量的地址

3、如果mutex当中的计数器为1，pthread_mutex_lock接口就返回了，表示加锁成功，同时计数器当中的值会被更改为0.

4、如果mutex当中的计数器为0，pthread_mutex_lock接口就阻塞了，pthread_mutex_lock接口没有返回了，阻塞在函数内部，直到加锁成功

第二个接口：int pthread_mutex_trylock(pthread_mutex_t *mutex);

1、该接口为非阻塞接口

2、mutex中计数器为1时，加锁成功，计数器置为0，然后返回

3、mutex中计数器为0时，加锁失败，但也会返回，此时加锁是失败状态，一定不要去访问临界资源

4、非阻塞接口一般都需要搭配循环来使用。

第三个接口：int pthread_mutex_timedlock(pthread_mutex_t *restrict mutex, const struct timespec *restrict abs_timeout);

1、带有超时时间的加锁接口

2、不能直接获取互斥锁的时候，会等待abs_timeout时间

3、如果在这个时间内加锁成功了，直接返回，不需要再继续等待剩余的时间，并且表示加锁成功

4、如果超出了该时间，也返回了，但是加锁失败了，需要循环加锁

上述三个加锁接口，第一个接口用的最多。

**6.3.4互斥量的解锁**

```text
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```

1. 对上述所有的加锁接口，都可使用该函数解锁
2. 解锁的时候，会将互斥锁当中计数器的值从0变为1，表示其它线程可以获取互斥量

### 6.4互斥锁的本质

1、在互斥锁内部有一个计数器，其实就是互斥量，计数器的值只能为0或者为1

2、当线程获取互斥锁的时候，如果计数器当前值为0，表示当前线程不能获取到互斥锁，也就是没有获取到互斥锁，就不要去访问临界资源

3、当前线程获取互斥锁的时候，如果计数器当前值为1，表示当前线程可以获取到互斥锁，也就是意味着可以访问临界资源

### 6.5互斥锁中的计数器如何保证了原子性？

获取锁资源的时候（加锁）：

1、寄存器当中值直接赋值为0

2、将寄存器当中的值和计数器当中的值进行交换

3、判断寄存器当中的值，得出加锁结果

两种情况：

![img](https://pic1.zhimg.com/80/v2-2954e6bfb9307a54931c667b7e8faa34_720w.webp)

例：4个线程，对同一个全局变量进行减减操作

```text
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>
#include <sys/syscall.h>
#define NUMBER 4
int g_val = 100;

pthread_mutex_t mutex;//定义互斥锁


void *ThreadWork(void *arg)
{
  int *p = (int*)arg;
  pthread_detach(pthread_self());//自己分离自己，不用主线程回收它的资源了
  while(1)
  {
    pthread_mutex_lock(&mutex);//加锁
    if(g_val > 0)
    {
      printf("i am pid : %d,i get g_val : %d\n",(int)syscall(SYS_gettid),g_val);
      --g_val;
      usleep(2);
    }
    else{
      pthread_mutex_unlock(&mutex);//在所有可能退出的地方，进行解锁
      break;
    }
    pthread_mutex_unlock(&mutex);//解锁

  }
  pthread_exit(NULL);
}

int main()
{
  pthread_t tid[NUMBER];
  pthread_mutex_init(&mutex,NULL);//互斥锁初始化
  int i = 0;
  for(;i < NUMBER;++i)
  {
    int ret = pthread_create(&tid[i],NULL,ThreadWork,(void*)&g_val);//不要传临时变量，这里是示范
    if(ret != 0)
    {
      perror("pthread_create");
      return -1;
     }
  }
  //pthread_join(tid,NULL);//线程等待
  //pthread_detach(tid);//线程分离
  pthread_mutex_destroy(&mutex);//销毁互斥锁
  while(1)
  {
    printf("i am main work thread\n");
    sleep(1);
  }
  return 0;
}
```

### 6.6互斥锁公平嘛？

互斥锁是不公平的。

内核维护等待队列， 互斥量实现了大体上的公平； 由于等待线程被唤醒后， 并不自动持有互斥量， 需要和刚进入临界区的线程竞争（抢锁）， 所以互斥量并没有做到先来先服务。

### 6.7互斥锁的类型

1、PTHREAD_MUTEX_NORMAL： 最普通的一种互斥锁。 它不具备死锁检测功能， 如线程对自己锁定的互斥量再次加锁， 则会发生死锁。

2、
PTHREAD_MUTEX_RECURSIVE_NP： 支持递归的一种互斥锁， 该互斥量的内部维护有互斥锁的所有者和一个锁计数器。 当线程第一次取到互斥锁时， 会将锁计数器置1， 后续同一个线程再次执行加锁操作时， 会递增该锁计数器的值。 解锁则递减该锁计数器的值， 直到降至0， 才会真正释放该互斥量， 此时其他线程才能获取到该互斥量。 解锁时， 如果互斥量的所有者不是调用解锁的线程， 则会返回EPERM。

3、
PTHREAD_MUTEX_ERRORCHECK_NP： 支持死锁检测的互斥锁。 互斥量的内部会记录互斥锁的当前所有者的线程ID（调度域的线程ID） 。 如果互斥量的持有线程再次调用加锁操作， 则会返回EDEADLK。 解锁时， 如果发现调用解锁操作的线程并不是互斥锁的持有者， 则会返回EPERM。

4、自旋锁，自旋锁采用了和互斥量完全不同的策略， 自旋锁加锁失败， 并不会让出CPU， 而是不停地尝试加锁， 直到成功为止。 这种机制在临界区非常小且对临界区的争夺并不激烈的场景下， 效果非常好。自旋锁的效果好， 但是副作用也大， 如果使用不当， 自旋锁的持有者迟迟无法释放锁， 那么， 自旋接近于死循环， 会消耗大量的CPU资源， 造成CPU使用率飙高。 因此， 使用自旋锁时， 一定要确保临界区尽可能地小， 不要有系统调用， 不要调用sleep。 使用strcpy/memcpy等函数也需要谨慎判断操作内存的大小， 以及是否会引起缺页中断。

5、PTHREAD_MUTEX_ADAPTIVE_NP：自适应锁，首先与自旋锁一样， 持续尝试获取， 但过了一定时间仍然不能申请到锁， 就放弃尝试， 让出CPU并等待。 PTHREAD_MUTEX_ADAPTIVE_NP类型的互斥量， 采用的就是这种机制。

### 6.8死锁和活锁

![img](https://pic2.zhimg.com/80/v2-aff262e422876ccf53db6bd423b0be29_720w.webp)

线程1已经成功拿到了互斥量1， 正在申请互斥量2， 而同时在另一个CPU上，线程2已经拿到了互斥量2， 正在申请互斥量1。 彼此占有对方正在申请的互斥量，结局就是谁也没办法拿到想要的互斥量， 于是死锁就发生了。

**6.8.1死锁概念**

死锁是指在一组进程中的各个进程均占有不会释放的资源，但因互相申请被其它进程所占有不会释放的资源而处于一种永久等待的状态。

**6.8.2死锁的四个必要条件**

1、互斥条件：一个资源只能被一个执行流使用

2、请求与保持条件：一个执行流因请求资源而阻塞时，对已获得的资源不会释放

3、不剥夺条件：一个执行流已获得的资源，在未使用完之前，不能强行剥夺

4、循环等待条件：若干执行流之间形成一种头尾相接的循环等待资源的关系

**6.8.3避免死锁**

1、破坏死锁的四个必要条件（实际上只能破坏条件2和4）

2、加锁顺序一致（按照先后顺序申请互斥锁）

3、避免未释放锁的情况

4、资源一次性分配

**6.8.4活锁**

避免死锁的另一种方式是尝试一下，如果取不到锁就返回。

```text
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_timedlock(pthread_mutex_t *restrict mutex,const struct timespec *restrict abs_timeout);
```

这两个函数反映了一种，不行就算了的思想。

trylock不行就回退的思想有可能会引发活锁（live lock） 。 生活中也经常遇到两个人迎面走来， 双方都想给对方让路， 但是让的方向却不协调， 反而互相堵住的情况 。 活锁现象与这种场景有点类似。

![img](https://pic4.zhimg.com/80/v2-2c2ce9846311e996ae6f7da8b0ec64c3_720w.webp)

线程1首先申请锁mutex_a后， 之后尝试申请mutex_b， 失败以后， 释放mutex_a进入下一轮循环， 同时线程2会因为尝试申请mutex_a失败，而释放mutex_b， 如果两个线程恰好一直保持这种节奏， 就可能在很长的时间内两者都一次次地擦肩而过。 当然这毕竟不是死锁， 终究会有一个线程同时持有两把锁而结束这种情况。 尽管如此， 活锁的确会降低性能。

**6.8.5死锁调试**

```text
查看多个线程堆栈：thread apply all bt
跳转到线程中：t 线程号
查看具体的调用堆栈：f 堆栈号
直接从pid号用gdb调试：gdb attach pid

#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>
#include <sys/syscall.h>
#define NUMBER 2

pthread_mutex_t mutex1;//定义互斥锁
pthread_mutex_t mutex2;


void *ThreadWork1(void *arg)
{
  int *p = (int*)arg;
  pthread_mutex_lock(&mutex1);
  
  sleep(2);
  
  pthread_mutex_lock(&mutex2);
  pthread_mutex_unlock(&mutex2);
  pthread_mutex_unlock(&mutex1);
  return NULL;
}

void *ThreadWork2(void *arg)
{
  int *p = (int*)arg;
  pthread_mutex_lock(&mutex2);
  
  sleep(2);
  
  pthread_mutex_lock(&mutex1);
  pthread_mutex_unlock(&mutex1);
  pthread_mutex_unlock(&mutex2);
  return NULL;
}
int main()
{
  pthread_t tid[NUMBER];
  pthread_mutex_init(&mutex1,NULL);//互斥锁初始化
  pthread_mutex_init(&mutex2,NULL);//互斥锁初始化
  int i = 0;
  int ret = pthread_create(&tid[0],NULL,ThreadWork1,(void*)&i);
  if(ret != 0)
  {
    perror("pthread_create");
    return -1;
  }
  ret = pthread_create(&tid[1],NULL,ThreadWork2,(void*)&i);
  if(ret != 0)
  {
    perror("pthread_create");
    return -1;
  }
  //pthread_join(tid,NULL);//线程等待
  //pthread_join(tid,NULL);//线程等待
  //pthread_detach(tid);//线程分离
  pthread_join(tid[0],NULL);
  pthread_join(tid[1],NULL);
  pthread_mutex_destroy(&mutex1);//销毁互斥锁
  pthread_mutex_destroy(&mutex2);//销毁互斥锁
  while(1)
  {
    printf("i am main work thread\n");
    sleep(1);
  }
  return 0;
}
```

在上述代码中，一定会出现死锁，线程1拿到了互斥锁1，又再去申请线程2的互斥锁2，线程2拿到了互斥锁2又再去申请线程1的互斥锁1。

开始调试：

1、找到进程号

![img](https://pic2.zhimg.com/80/v2-8e1eb2ea447981df2212fa2622282591_720w.webp)

2、开始调试

![img](https://pic3.zhimg.com/80/v2-78fdb318eb2acf7c4915c5c586cdbcd6_720w.webp)

3、查看多个线程堆栈

![img](https://pic4.zhimg.com/80/v2-e03c95c8739e8139d23850d381c692e7_720w.webp)

4、跳转到线程中

![img](https://pic3.zhimg.com/80/v2-8c20d0e5574b37001d9e52e48bdbaf3e_720w.webp)

5、查看具体调用堆栈

![img](https://pic3.zhimg.com/80/v2-79c77f49cd2151503280b64586c699c2_720w.webp)

6、查看互斥锁1和互斥锁2，分别被谁拿着

![img](https://pic2.zhimg.com/80/v2-a94db56ae96a52658911c94b2fd7c4b9_720w.webp)

### 6.9读写锁

**6.9.1什么是读写锁？**

大部分情况下，对于共享变量的访问特点：只是读取共享变量的值，而不是修改，只有在少数情况下，才会真正的修改共享变量的值。

在这种情况下，读请求之间是同步的，它们之间的并发访问是安全的。然而写请求必须锁住读请求和其它写请求。

即读线程可多个同时读，而写线程只允许同一时间内一个线程去写。

**6.9.2读写锁接口**

```text
#include <pthread.h>
//销毁
int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);

//初始化
int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock,
              const pthread_rwlockattr_t *restrict attr);
```

![img](https://pic3.zhimg.com/80/v2-890207d7d114d64e436ec1c0284aa92a_720w.webp)

读写锁的默认属性：

![img](https://pic4.zhimg.com/80/v2-62306a2d0fb1d844d61f69b9be243c47_720w.webp)

对于调用pthread_rwlock_init初始化的读写锁，在不需要读写锁的时候，需要调用pthread_rwlock_destroy销毁。

**6.9.3读者加锁**

```text
#include <pthread.h>

int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock); //阻塞类型的读加锁接口
int pthread_rwlock_tryrdlock(pthread_rwlock_t *rwlock); //非阻塞类型的读加锁接口
```

最大的好处就是，允许多个线程以只读加锁的方式获取到读写锁；

本质上，读写锁的内部维护了一个引用计数，每当线程以读方式获取读写锁时，该引用计数+1；

当释放以读加锁的方式的读写锁时，会先对引用计数进行-1，直到引用计数的值为0的时候，才真正释放了这把读写锁。

**6.9.4写者加锁**

```text
#include <pthread.h>
int pthread_rwlock_trywrlock(pthread_rwlock_t *rwlock);// 非阻塞写
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);//阻塞写
```

写锁用的是独占模式，如果当前读写锁被某写线程占用着，则不允许任何读锁通过请求，也不允许任何写锁请求通过，读锁请求和写锁请求都要陷入阻塞，直到线程释放写锁。

**6.9.5 解锁**

```text
#include <pthread.h>
int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);
```

不论是读者加锁还是写者加锁，都采用该接口进行解释。

读者解锁，只有当引用计数为0的时候，才真正释放了读写锁。

**6.9.6读写锁的竞争策略**

对于读写锁而言，目前有两种策略，读者优先和携着优先；

读写锁的类型有如下几种：

```text
PTHREAD_RWLOCK_PREFER_READER_NP, //读者优先
PTHREAD_RWLOCK_PREFER_WRITER_NP, //很唬人， 但是也是读者优先
PTHREAD_RWLOCK_PREFER_WRITER_NONRECURSIVE_NP, //写者优先
PTHREAD_RWLOCK_DEFAULT_NP = PTHREAD_RWLOCK_PREFER_READER_NP
```

读者优先：读锁来请求可以立即响应，只要有一个读锁没完成，那么写锁就无法写。这种策略是不公平的，极端情况下，写现场很可能被饿死，即线程总是拿不到锁资源。

写者优先：只要线程申请了写锁，那么在写锁后面到来的读锁请求就会统统被阻塞，不能先于写锁拿到锁。

读写锁实现中的变量及含义

![img](https://pic1.zhimg.com/80/v2-d2af269cba4c6ddeb8e63811fa945d58_720w.webp)

对于读请求而言：如果

> \1. 无线程持有写锁，即_writer = 0.
> \2. 采用读者优先策略或者当前没有写锁申请请求，即 _nr_writers_queue = 0
> \3. 当满足这两个条件时，读锁请求立即获得读锁，返回之前执行_nr_readers++，表示多了一个线程正在读
> \4. 不满足这两个条件时，执行_nr_readers_queued++，表示增加了一个读锁等待者，然后调用futex，陷入阻塞。醒来之后，执行_nr_readers_queued- -，再次判断是否满足条件1，2

对于写请求而言：如果

> \1. 无线程持有写锁，即_writer = 0.
> \2. 没有线程持有读锁，即_nr_readers = 0.
> \3. 如果上述条件满足，就会立即拿到锁，将_writer 置为当前线程的ID
> \4. 如果不满足，则执行_nr_writers_queue++， 表示增加了一个写锁等待者线程，然后执行futex陷入等待。醒来后，先执行_nr_writers_queue- -，再继续判断条件1，2

对于解锁，如果当前是写锁：

> \1. 执行_writer = 0.，表示释放写锁。
> \2. 根据_nr_writers_queue判断有没有写锁，如果有则唤醒一个写锁，如果没有写锁等待者，则唤醒所有的读锁等待者。

对于解锁，如果当前是读锁：

> \1. 执行_nr_readers- -，表示读锁占有者少了一个。
> \2. 判断_nr_readers是否等于0，是的话则表示当前线程是最后一个读锁占有者，需要唤醒写锁等待者或读锁等待者
> \3. 根据_nr_writers_queue判断是否存在写锁等待者，若有，则唤醒一个写锁等待线程
> \4. 如果没有写锁等待者，判断是否存在读锁等待者，若有，则唤醒全部的读锁等待者

读写锁很容易造成，读者饿死或者写者饿死。

也可以设计公平的读写锁。

代码：

```text
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <sys/syscall.h>
#include <unistd.h>
#include <fcntl.h>

#define THREADCOUNT 100

static int count = 0;
static pthread_rwlock_t lock;

void* Read(void* i)
{
  while(1)
  {
    pthread_rwlock_rdlock(&lock);
    printf("i am 读线程 : %d, 现在的count是%d\n", (int)syscall(SYS_gettid), count);
    pthread_rwlock_unlock(&lock);
    //sleep(1);
  }
}

void* Write(void* i)
{
  while(1)
  {
    pthread_rwlock_wrlock(&lock);
    ++count;
    printf("i am 写线程 : %d, 现在的count是: %d\n", (int)syscall(SYS_gettid), count);
    pthread_rwlock_unlock(&lock);
    sleep(1);
  }
}


int main()
{
  //close(1);
  //int fd = open("./dup2_result.txt", O_CREAT | O_RDWR);
  //dup2(fd, 1);
  pthread_t tid[THREADCOUNT];
  pthread_rwlock_init(&lock, NULL);
  for(int i = 0; i < THREADCOUNT; ++i)
  {
    if(i % 2 == 0)
    {
      pthread_create(&tid[i], NULL, Read, (void*)&i);
    }
    else
    {
      pthread_create(&tid[i], NULL, Write, (void*)&i);
    }
  }

  for(int i = 0; i < THREADCOUNT; ++i)
  {
    pthread_join(tid[i], NULL);
  }

  pthread_rwlock_destroy(&lock);
  return 0;
}
```

上述代码很容易触发线程饿死。
读饿死或者写饿死。

## 7.线程间同步

### 7.1为什么需要线程同步？

线程同步是为了对临界资源访问的合理性。

例如：

> 就像工厂里生产车间没有原料了， 所有生产车间都停工了， 工人们都在车间睡觉。 突然进来一批原料， 如果原料充足， 你会发广播给所有车间， 原料来了， 快来开工吧。 如果进来的原料很少， 只够一个车间开工的， 你可能只会通知一个车间开工。

### 7.2如何做到线程间同步？

条件等待是线程间同步的另一种方法。

如果条件不满足， 它能做的事情就是等待， 等到条件满足为止。 通常条件的达成， 很可能取决于另一个线程， 比如生产者-消费者模型。 当另外一个线程发现条件符合的时候， 它会选择一个时机去通知等待在这个条件上的线程。 有两种可能性， 一种是唤醒一个线程， 一种是广播， 唤醒其他线程。

则在这个情况下，需要做到：

1、线程在条件不满足的情况下， 主动让出互斥量， 让其他线程去折腾， 线程在此处等待， 等待条件的满足；

2、一旦条件满足， 线程就可以立刻被唤醒。

3、线程之所以可以安心等待， 依赖的是其他线程的协作， 它确信会有一个线程在发现条件满足以后， 将向它发送信号， 并且让出互斥量。

### 7.3条件变量

本质上是PCB等待队列 + 等待接口 + 唤醒接口。

**7.3.1条件变量的初始化**

静态初始化

```text
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
```

动态初始化

```text
pthread_cond_init(pthread_cond_t *cond,const pthread_condattr_t *attr);
```

![img](https://pic1.zhimg.com/80/v2-c898a8dc634598dd27f7303ce080aa10_720w.webp)

**7.3.2条件变量的等待**

```text
int pthread_cond_wait(pthread_cond_t *restrict cond,pthread_mutex_t *restrict mutex);
int pthread_cond_timedwait(pthread_cond_t *restrict conpthread_mutex_t *restrict mutex,const struct timespec *restrict abstime);
```

为什么这两个接口中有互斥锁？

条件不会无缘无故地突然变得满足了， 必然会牵扯到共享数据的变化。 所以一定要有互斥锁来保护。 没有互斥锁， 就无法安全地获取和修改共享数据。

同步并没有保证互斥，而保证互斥是使用到了互斥锁。

```text
pthread_mutex_lock(&m)
while(condition_is_false)
{
	pthread_mutex_unlock(&m);
	//解锁之后， 等待之前， 可能条件已经满足， 信号已经发出， 但是该信号可能会被错过
	cond_wait(&cv);
	pthread_mutex_lock(&m);
}
```

上面的解锁和等待不是原子操作。 解锁以后， 调用cond_wait之前，如果已经有其他线程获取到了互斥量， 并且满足了条件， 同时发出了通知信号， 那么cond_wait将错过这个信号， 可能会导致线程永远处于阻塞状态。 所以解锁加等待必须是一个原子性的操作， 以确保已经注册到事件的等待队列之前， 不会有其他线程可以获得互斥量。

那先注册等待事件， 后释放锁不行吗？ 注意， 条件等待是个阻塞型的接口， 不单单是注册在事件的等待队列上， 线程也会因此阻塞于此， 从而导致互斥量无法释放， 其他线程获取不到互斥量， 也就无法通过改变共享数据使等待的条件得到满足， 因此这就造成了死锁。

```text
pthread_mutex_lock(&m);
while(condition_is_false)
	pthread_cond_wait(&v,&m);//此处会阻塞
/*如果代码运行到此处， 则表示我们等待的条件已经满足了，
*并且在此持有了互斥量
*/
/*在满足条件的情况下， 做你想做的事情。
*/
pthread_mutex_unlock(&m);
```

pthread_cond_wait函数只能由拥有互斥量的线程来调用， 当该函数返回的时候， 系统会确保该线程再次持有互斥量， 所以这个接口容易给人一种误解， 就是该线程一直在持有互斥量。 事实上并不是这样的。 这个接口向系统声明了我在PCB等待序列中之后， 就把互斥量给释放了。 这样其他线程就有机会持有互斥量，操作共享数据， 触发变化， 使线程等待的条件得到满足。

```text
pthread_cond_wait内部会进行解锁逻辑，则一定要先放到PCB等待序列中，再进行解锁。
while(condition_is_false)
	pthread_cond_wait(&v,&m);//此处会阻塞
if(condition_is_false)
	pthread_cond_wait(&v,&m);//此处会阻塞
```

唤醒以后， 再次检查条件是否满足， 是不是多此一举？

因为唤醒中存在虚假唤醒（spurious wakeup） ， 换言之，条件尚未满足， pthread_cond_wait就返了。 在一些实现中， 即使没有其他线程向条件变量发送信号， 等待此条件变量的线程也有可能会醒来。

条件满足了发送信号， 但等到调用pthread_cond_wait的线程得到CPU资源时， 条件又再次不满足了。 好在无论是哪种情况， 醒来之后再次测试条件是否满足就可以解决虚假等待的问题。

pthread_cond_wait内部实现逻辑：

1. 将调用pthread_cond_wait函数的执行流放入到PCB等待队列当中
2. 解锁
3. 等待被唤醒
4. 被唤醒之后：

> 1、从PCB等待队列中移除出来
> 2、抢占互斥锁
> 情况1：拿到互斥锁，pthread_cond_wait就返回了
> 情况2：没有拿到互斥锁，阻塞在pthread_cond_wait内部抢锁的逻辑中
> 当阻塞在pthread_cond_wait函数抢锁逻辑中时，一旦执行流时间耗尽，意味着线程就被切换出来了，程序计数器就保存的是抢锁的指令，上下文信息保存的就是寄存器的值
> 当再次拥有CPU资源后，恢复抢锁逻辑
> 直到抢锁成功，pthread_cond_wait函数才会返回

**7.3.3条件变量的唤醒**

```text
int pthread_cond_signal(pthread_cond_t *cond);
int pthread_cond_broadcast(pthread_cond_t *cond);
```

pthread_cond_signal负责唤醒等待在条件变量上的一个线程。

pthread_cond_broadcast，就是广播唤醒等待在条件变量上的所有线程。

先发送信号，然后解锁互斥量，这个顺序是必须的嘛？

先通知条件变量、 后解锁互斥量， 效率会比先解锁、 后通知条件变量低。 因为先通知后解锁， 执行pthread_cond_wait的线程可能在互斥量已然处于加锁状态的时候醒来， 发现互斥量仍然没有解锁， 就会再次休眠， 从而导致了多余的上下文切换。

**7.3.4条件变量的销毁**

```text
int pthread_cond_destroy(pthread_cond_t *cond);
```

注意：

1、永远不要用一个条件变量对另一个条件变量赋值， 即pthread_cond_t cond_b = cond_a不合法， 这种行为是未定义的。

2、使用PTHREAD_COND_INITIALIZE静态初始化的条件变量， 不需要被销毁。

3、要调用pthread_cond_destroy销毁的条件变量可以调用pthread_cond_init重新进行初始化。

4、不要引用已经销毁的条件变量， 这种行为是未定义的。

例：

```text
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>
#include <sys/syscall.h>
#define NUMBER 2

int g_bowl = 0;
pthread_mutex_t mutex;//定义互斥锁
pthread_cond_t cond1;//条件变量
pthread_cond_t cond2;//条件变量


void *WorkProduct(void *arg)
{
  int *p = (int*)arg;
  while(1)
  {
    pthread_mutex_lock(&mutex);
    while(*p > 0)
    {
      pthread_cond_wait(&cond2,&mutex);//条件等待，条件不满足，陷入阻塞
    }
    ++(*p);
    printf("i am workproduct :%d,i product %d\n",(int)syscall(SYS_gettid),*p);
    pthread_cond_signal(&cond1);//通知消费者
    pthread_mutex_unlock(&mutex);//释放锁
  }
  return NULL;
}

void *WorkConsume(void *arg)
{
  int *p = (int*)arg;
  while(1)
  {
    pthread_mutex_lock(&mutex);
    while(*p <= 0)
    {
      pthread_cond_wait(&cond1,&mutex);//条件等待，条件不满足，陷入阻塞
    }
    printf("i am workconsume :%d,i consume %d\n",(int)syscall(SYS_gettid),*p);
    --(*p);
    pthread_cond_signal(&cond2);//通知生产者
    pthread_mutex_unlock(&mutex);//释放锁
  }
  return NULL;
}
int main()
{
  pthread_t cons[NUMBER],prod[NUMBER];
  pthread_mutex_init(&mutex,NULL);//互斥锁初始化
  pthread_cond_init(&cond1,NULL);//条件变量初始化
  pthread_cond_init(&cond2,NULL);//条件变量初始化
  int i = 0;
  for(;i < NUMBER;++i)
  {
    int ret = pthread_create(&prod[i],NULL,WorkProduct,(void*)&g_bowl);
    if(ret != 0)
    {
      perror("pthread_create");
      return -1;
    }
    ret = pthread_create(&cons[i],NULL,WorkConsume,(void*)&g_bowl);
    if(ret != 0)
    {
      perror("pthread_create");
      return -1;
    }
  }
  for(i = 0;i < NUMBER;++i)
  {
    pthread_join(cons[i],NULL);//线程等待
    pthread_join(prod[i],NULL);
  }
  pthread_mutex_destroy(&mutex);//销毁互斥锁
  pthread_cond_destroy(&cond1);
  pthread_cond_destroy(&cond2);
  while(1)
  {
    printf("i am main work thread\n");
    sleep(1);
  }
  return 0;
}
```

在这里为什么有两个条件变量呢？
若所有的线程只使用一个条件变量，会导致所有线程最后都进入PCB等待队列。

thread apply all bt查看：

![img](https://pic3.zhimg.com/80/v2-4e79bc47098cfaea872545f74686d0c2_720w.webp)

**7.3.5情况分析：两个生产者，两个消费者，一个PCB等待队列**

1、最开始的情况，两个消费者抢到了锁，此时生产者未生产，则都放入PCB等待队列中

![img](https://pic3.zhimg.com/80/v2-40e5d68635788d8b8b93badd576a1dae_720w.webp)

2、一个生产者抢到了锁，生产了一份材料，唤醒一个消费者，此时三者抢锁，若两个生产者分别先后抢到了锁，则都进入PCB等待队列中

![img](https://pic3.zhimg.com/80/v2-fd2d33e7cd5015d17d6d6ccf0149c69e_720w.webp)

3、只有一个消费者，则必会抢到锁，消费材料，唤醒PCB等待队列，若此时唤醒的是，消费者，则现在是这样一个情况：

![img](https://pic1.zhimg.com/80/v2-d80be0c043fbdd85c226ae3f33caec8c_720w.webp)

4、两个消费者在外边抢锁，一定都会进入PCB等待队列中

![img](https://pic4.zhimg.com/80/v2-ee9eb3e45aac17a3e894307f136c56cf_720w.webp)

解决上述问题可采用两种方法：

1、使用int pthread_cond_broadcast(pthread_cond_t *cond);，唤醒PCB等待队列中所有的线程。此时所有线程都会同时执行抢锁逻辑，太消费资源了。此方法不妥

2、采用两个PCB等待序列，一个放生产者，一个放消费者，生产者唤醒消费者，消费者唤醒生产者。

## 8.线程取消

### 8.1线程取消函数接口

```text
int pthread_cancel(pthread_t thread);
```

一个线程可以通过调用该函数向另一个线程发送取消请求。 这不是个阻塞型接口， 发出请求后， 函数就立刻返回了， 而不会等待目标线程退出之后才返回。

调用pthread_cancel时， 会向目标线程发送一个SIGCANCEL的信号， 该信号就是kill -l中消失的32号信号。

线程的默认取消状态是PTHREAD_CANCEL_ENABLE。即是可被取消的。

![img](https://pic2.zhimg.com/80/v2-32d5fbd4479866807f26a3e4a9f826a5_720w.webp)

什么是取消点？ 可通过man pthreads查看取消点
就是对于某些函数， 如果线程允许取消且取消类型是延迟取消， 并且线程也收到了取消请求， 那么当执行到这些函数的时候， 线程就可以退出了。

### 8.2线程取消带来的弊端

目标线程可能会持有互斥量、 信号量或其他类型的锁， 这时候如果收到取消请求， 并且取消类型是异步取消， 那么可能目标线程掌握的资源还没有来得及释放就被迫退出了， 这可能会给其他线程带来不可恢复的后果， 比如死锁（其他线程再也无法获得资源） 。

注意：

轻易不要调用pthread_cancel函数， 在外部杀死线程是很糟糕的做法，毕竟如果想通知目标线程退出， 还可以采取其他方法。

如果不得不允许线程取消， 那么在某些非常关键不容有失的代码区域， 暂时将线程设置成不可取消状态， 退出关键区域之后， 再恢复成可以取消的状态。

在非关键的区域， 也要将线程设置成延迟取消， 永远不要设置成异步取消。

### 8.3线程清理函数

假设遇到取消请求， 线程执行到了取消点， 却没有来得及做清理动作（如动态申请的内存没有释放， 申请的互斥量没有解锁等） ， 可能会导致错误的产生， 比如死锁， 甚至是进程崩溃。

为了避免这种情况， 线程可以设置一个或多个清理函数， 线程取消或退出时，会自动执行这些清理函数， 以确保资源处于一致的状态。

如果线程被取消， 清理函数则会负责解锁操作。

```text
void pthread_cleanup_push(void (*routine)(void *),void *arg);
void pthread_cleanup_pop(int execute);
```

这两个函数必须同时出现， 并且属于同一个语法块。

何时会触发注册的清理函数：?

1、当线程的主函数是调用pthread_exit返回的， 清理函数总是会被执行。

2、当线程是被其他线程调用pthread_cancel取消的， 清理函数总是会被执行。

3、当线程的主函数是通过return返回的， 并且pthread_cleanup_pop的唯一参数execute是0时， 清理函数不会被执行.

4、线程的主函数是通过return返回的， 并且pthread_cleanup_pop的唯一参数execute是非零值时， 清理函数会执行一次。

代码：

```text
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <pthread.h>
#include <unistd.h>
#include <sys/syscall.h>
#define NUMBER 2
int g_bowl = 0;

pthread_mutex_t mutex;//定义互斥锁

void clean(void *arg)
{
  printf("Clean up:%s\n",(char*)arg);
  pthread_mutex_unlock(&mutex);//释放锁
}

void *WorkCancel(void *arg)
{
  pthread_mutex_lock(&mutex);
  pthread_cleanup_push(clean,"clean up handler");//清除函数的push
  struct timespec t = {3,0};//取消点
  nanosleep(&t,0);
  pthread_cleanup_pop(0);//清除
  pthread_mutex_unlock(&mutex);
}

void *WorkWhile(void *arg)
{
  sleep(5);
  pthread_mutex_lock(&mutex);
  printf("i get the mutex\n");//若能拿到资源，则表示取消清理函数成功！
  pthread_mutex_unlock(&mutex);
  return NULL;
}
int main()
{
  pthread_t cons,prod;
  pthread_mutex_init(&mutex,NULL);//互斥锁初始化
  int ret = pthread_create(&prod,NULL,WorkCancel,(void*)&g_bowl);//该线程拿到锁，然后挂掉
  if(ret != 0)
  {
    perror("pthread_create");
    return -1;
  }
  int ret1 = pthread_create(&cons,NULL,WorkWhile,(void*)&ret);//测试该线程是否可以拿到锁
  if(ret1 != 0)
  {
    perror("pthread_create");
    return -1;
  }

  pthread_cancel(prod);//取消该线程
  pthread_join(prod,NULL);//线程等待
  pthread_join(cons,NULL);//线程等待
  pthread_mutex_destroy(&mutex);//销毁互斥锁
  while(1)
  {
    sleep(1);
  }
  return 0;
}
```

结果：只要拿到锁，就表明线程清理函数成功了。

![img](https://pic4.zhimg.com/80/v2-75bd5eacc35c2d6a73e1e6abbda4e29b_720w.webp)

## 9.多线程与fork()

永远不要在多线程程序里面调用fork。

Linux的fork函数， 会复制一个进程， 对于多线程程序而言， fork函数复制的是用fork的那个线程， 而并不复制其他的线程。 fork之后其他线程都不见了。 Linux存在forkall语义的系统调用， 无法做到将多线程全部复制。

多线程程序在fork之前， 其他线程可能正持有互斥量处理临界区的代码。 fork之后， 其他线程都不见了， 那么互斥量的值可能处于不可用的状态， 也不会有其他线程来将互斥量解锁。

## 10.生产者与消费者模型

### 10.1生产者与消费者模型的本质

本质上是一个线程安全的队列，和两种角色的线程（生产者和消费者）

存在三种关系：

1、生产者与生产者互斥

2、消费者与消费者互斥

3、生产者与消费者同步+互斥

### 10.2为什么需要生产者与消费者模型？

生产者和消费者彼此之间不直接通讯，而通过阻塞队列来进行通讯，所以生产者生成完数据之后不用等待消费者处理，直接扔给阻塞队列，消费者不找生产者要数据，而是直接从阻塞队列中取，阻塞队列就相当于一个缓冲区，平衡了生产者和消费者的处理能力。这个阻塞队列就是用来给生产者和消费解耦的。

### 10.3优点

1、解耦

2、支持高并发

3、支持忙闲不均

### 10.4实现两个消费者线程，两个生产者线程的生产者消费者模型

生产者生成时用的同一个全局变量，故对该全局变量进行了加锁。

```text
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <pthread.h>
#include <queue>
#include <sys/syscall.h>

#define PTHREAD_COUNT 2
int data = 0;//全局变量作为插入数据
pthread_mutex_t mutex1;
class ModelOfConProd{
  public:
    ModelOfConProd()//构造
    {
      _capacity = 10;
      pthread_mutex_init(&_mutex,NULL);
      pthread_cond_init(&_cons,NULL);
      pthread_cond_init(&_prod,NULL);
    }
    ~ModelOfConProd()//析构
    {
      _capacity = 0;
      pthread_mutex_destroy(&_mutex);
      pthread_cond_destroy(&_cons);
      pthread_cond_destroy(&_prod);
    }

    void Push(int data)//push数据，生产者线程使用的
    {
      pthread_mutex_lock(&_mutex);
      while((int)_queue.size() >= _capacity)
      {
        pthread_cond_wait(&_prod,&_mutex);
      }
      _queue.push(data);
      pthread_mutex_unlock(&_mutex);
      pthread_cond_signal(&_cons);
    }


    void Pop(int& data)//pop数据，消费者线程使用的
    {
      pthread_mutex_lock(&_mutex);
      while(_queue.empty())
      {
        pthread_cond_wait(&_cons,&_mutex);
      }
      data = _queue.front();
      _queue.pop();
      pthread_mutex_unlock(&_mutex);
      pthread_cond_signal(&_prod);
    }

  private:
    int _capacity;//容量大小，限制容量大小
    std::queue<int> _queue;//队列
    pthread_mutex_t _mutex;//互斥锁
    pthread_cond_t _cons;//消费者条件变量
    pthread_cond_t _prod;//生产者条件变量
};

void *ConsumerStart(void *arg)//消费者入口函数
{
  ModelOfConProd *cp = (ModelOfConProd *)arg;
  while(1)
  {
    cp->Push(data);
    printf("i am pid : %d,i push :%d\n",(int)syscall(SYS_gettid),data);
    pthread_mutex_lock(&mutex1);//++的时候，给该全局变量加锁
    ++data;
    pthread_mutex_unlock(&mutex1);
  }
}


void *ProductsStart(void *arg)//生产者入口函数
{
  ModelOfConProd *cp = (ModelOfConProd *)arg;
  int data = 0;
  while(1)
  {
    cp->Pop(data);
    printf("i am pid : %d,i pop :%d\n",(int)syscall(SYS_gettid),data);
  }
}


int main()
{
  ModelOfConProd *cp = new ModelOfConProd;
  pthread_mutex_init(&mutex1,NULL);
  pthread_t cons[PTHREAD_COUNT],prod[PTHREAD_COUNT];
  for(int i = 0;i < PTHREAD_COUNT; ++i)
  {
    int ret = pthread_create(&cons[i],NULL,ConsumerStart,(void*)cp);
    if(ret < 0)
    {
      perror("pthread_create");
      return -1;
    }
    ret = pthread_create(&prod[i],NULL,ProductsStart,(void*)cp);
    if(ret < 0)
    {
      perror("pthread_create");
      return -1;
    }
  }


  for(int i = 0;i < PTHREAD_COUNT;++i)
  {
    pthread_join(cons[i],NULL);
    pthread_join(prod[i],NULL);
  }


  pthread_mutex_destroy(&mutex1);

  return 0;
}
```

## 11.写多线程时应注意

1. 先考虑代码的核心逻辑（先实现）
2. 考虑核心逻辑中是否访问临界资源或者说执行临界区代码，如果有就需要保持互斥
3. 考虑线程之间是否需要同步

原文地址：https://zhuanlan.zhihu.com/p/381191748

作者：linux