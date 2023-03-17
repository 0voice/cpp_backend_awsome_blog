# 【NO.538】Linux服务器开发,原子操作CAS与锁实现

## 1.互斥锁 mutex

锁这个东西之所以会存在是因为资源不能被两个以上的线程同时使用，锁是对临界资源的一种保护。大家知道，”idx++”其中的“++”这个操作，转成汇编为三个指令：

- Mov[idx],%eax(将idx从内存中转到寄存器);
- Inc %eax(通过寄存器进行自增);
- Mov %eax,[idx](从寄存器中读出)。
  在不同线程间切换，先前线程会将idx的值保留至本线程内的寄存器，线程切换后即便值已经发生改变却无法做到更新同步，最终导致竞争资源得到的结果小于理想值。
  \```c
  \#include<stdio.h>
  \#include<unistd.h>
  \#include<pthread.h>

pthread_mutex_t mutex;
//加锁
pthread_mutex_lock(&mutex);
(*pcount) ++;
pthread_mutex_unlock(&mutex);

//锁的初始化
pthread_mutex_init(&mutex,NULL);

~~~javascript
##  二、自旋锁 spinlock



自旋锁的用法和互斥锁是十分相似的。



 



```c



#include<stdio.h>



#include<unistd.h>



#include<pthread.h>



 



pthread_spin_t spinlock;



//加锁



pthread_spin_lock(&spinlock);



    (*pcount) ++;



pthread_spin_unlock(&spinlock);



 



//锁的初始化，共享锁



pthread_spinlock_init(&spinlock,PTHREAD_PROCESS_SHARED);
~~~

虽然使用起来看样子非常相似，但是还是有着细微的差别：

- spinlock在等待时，不会有线程和cpu的切换，mutex会出现线程的切换。

- 临界资源操作简单的，没有发生系统调用，没有耗时操作可以优先选择spinlock。

- 原子<spinlock<mutex。

  ## 三、原子操作

  //xaddl 的意思是将value=value+add；锁住内存总线

```c
int inc(int *value,int add)



{



    int old;



    __asm__ volatile (



        : "lock; xaddl %2 ,%1;“



        :"=a" (old)



        :"m" (*value),"a" (add)



        :"cc"."memory"



    );



    return old;



}



//调用



inc(pcont,1);
```



![在这里插入图片描述](https://img-blog.csdnimg.cn/63783b8335ef44e1ab0b020cb72adb45.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_11,color_FFFFFF,t_70,g_se,x_16)


哈哈，最后同样要是一百万的结果！
原子操作的本质需要cpu指令集的操作支持，如果想把链表用原子锁锁住那是不行的，因为这个动作要需要四个动作。
原子操作知识cpu支持的指令集。当联想到CAS（compare and swap）时，要注意的时，cas只是原子操作的一种。



## 2.线程私有空间 pthread_key

```c
/***************************************************************************************/



#define THREAD_COUNT 4



pthread_key_t key;



pthread_key_create(&key,NULL);



typedef void *（thread_cb)*(void *);



/***************************************************************************************/



void print_thread1_key(void)



{



    int *p=(int *)pthread_getspecific(key);



    printf("thread 1:%d\n",*p);



}



void *thread1_proc(void * arg){



        int i=5;



        pthread_setspecific(key,&i);



        print_thread1_key();



}



/***************************************************************************************/



void print_thread2_key(void)



{



    (char *)pthread_getspecific(key);



    printf("thread 2:%d\n",*p);



}



void *thread2_proc(void * arg){



        cha *p=”thread2_proc“;



        pthread_setspecific(key,p);



        print_thread2_key();



}



/***************************************************************************************/



struct pair



{



    int x;



    int y;



}



void print_thread3_key(void)



{



    istruct pair *p=(int *)pthread_getspecific(key);



    printf("thread 3:x=%d y=%d\n",*p);



}



void *thread3_proc(void * arg){



        struct pair p={1,2};



        pthread_setspecific(key,&p);



        print_thread3_key();



}



/***************************************************************************************/



int main(){



thread_cb callback[THREAD_COUNT ]={



                        thread1_proc,



                        thread2_proc,



                        thread3_proc        



                                  };



for(int i{};i<THREAD_COUNT ;i++)



{



        pthread_create(&thid[i],NULL,callback[i],&count);



}



for(int i{};i<THREAD_COUNT ;i++)



{



        pthread_join(thid[i],NULL);



}



    return 0;



}
```

## 3.共享内存

## 4.cpu亲缘性



![在这里插入图片描述](https://img-blog.csdnimg.cn/827091a423d846f2be6ac237c1851a28.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16)



```c
//让cpu进行粘合的作用。



#include<stdio.h>



#include<pthread.h>



#iclude<setjmp.h>



#include<sched.h>



#include<sys/syscall.h>



#include<stdlib.h>



#include<string.h>



 



#define __USE_GNU



#include<unistd.h>



void process_affinity(int num)



{



    //gttid();和下一行代码同意



    pid_t selfid=syscall(_NR_gettid);



    cpu_set_t mask;



    CPU_ZERO(&MASK);



    CPU_SET(selfid %num,&mask);



    sched_setaffiniity(selfid,sizeof(mask),&mask);



    while(1)



    {}



}



int main()



{



    int num=sysconf(_SC_NPROCESSORS_CONF);



    int i=0;



    pid_t pid=0;



    for(i=0;i<num;i++)



    {



        pid=fork();



            if(pid<=(pid_t)0)



            {



                    break;



            }



    }



    if(pid==0)



    {



        process_affinity(num);



    }



    while(1)



        usleep(1);



}
```

## 5.setjmp/longjmp

```c
#include<stdio.h>



#include<unistd.h>



#include<pthread.h>



#iclude<setjmp.h>



//跨函数的跳跃



#define Try count=setjmp(env);if(count)==0



#define Catch(type) else if(count==type)



#define Throw(type) longjmp(env,type)



#define Finally



 



struct ExceptionFrame{



    jmp_buf env;



    int count=0;



    struct ExceptionFrame *next;



};



void sub_func(int idx)



{



    //longjmp(env,idx);



    Throw(idx);



}



 



int main()



{



#if 0



    count=setjmp(env);



    if(0==count)



    {



        sub_func(++count);



    }else if(1==count)



    {



        sub_func(++count);



    }else if(2==count)



    {



        sub_func(++count);



    }else



    {



        printf("other item");



    }



    



#else



    Try{



    sub_func(++count);



    }catch(1){



    sub_func(++count);



    }catch(2){



    sub_func(++count);



    }catch(3){



    sub_func(++count);



    }Finally{



        printf("other item\n");



    }    



}
```

## 6.try/catch 嵌套怎么解决？

用链表将ExceptionFrame串起来。

## 7.线程安全怎么解决？

线程的私有空间解决这个问题。

## 8.线上参考版本待完成

原文作者：[屯门山鸡叫我小鸡](https://blog.csdn.net/sinat_28294665)

原文链接：https://bbs.csdn.net/topics/604939496