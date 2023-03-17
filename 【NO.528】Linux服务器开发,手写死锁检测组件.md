# 【NO.528】Linux服务器开发,手写死锁检测组件

前言：
当项目收尾阶段，毫无疑问都要进行死锁的检测。如果线程数量少，可以利用查看日志、GDB调试或者干脆看cpu占用率检测出死锁问题。现在问题来了，如果线程数量多，我们该如何操作呢？今天我们就来研究学习一下！





![在这里插入图片描述](https://img-blog.csdnimg.cn/9fb1fa360f324018b1fac17073ab30aa.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_17,color_FFFFFF,t_70,g_se,x_16#pic_center)



# 1.死锁的构建

## 1.1 死锁cpu展示



![看不出来，换成spin_lock就清晰了！](https://img-blog.csdnimg.cn/43e8446442de45e48036d85339626ab8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16)



## 1.2. 基础代码实现

```c
#include<stdio.h>



#include<stdlib.h>



#include<stdint.h>



#include<pthread.h>



#include<unistd.h>



#include<iostream>



pthread_mutex_t mtx1 = PTHREAD_MUTEX_INITIALIZER;



pthread_mutex_t mtx2 = PTHREAD_MUTEX_INITIALIZER;



pthread_mutex_t mtx3 = PTHREAD_MUTEX_INITIALIZER;



pthread_mutex_t mtx4 = PTHREAD_MUTEX_INITIALIZER;



void* thread_routine_1(void *)



{



    std::cout << __FUNCTION__ <<" Start..." << std::endl;



    pthread_mutex_lock(&mtx1);



    sleep(1);



    pthread_mutex_lock(&mtx2);



    pthread_mutex_unlock(&mtx2);



    pthread_mutex_unlock(&mtx1);



    std::cout << __FUNCTION__ << " End..." << std::endl;



}



void* thread_routine_2(void*)



{



    std::cout << __FUNCTION__ << " Start..." << std::endl;;



    pthread_mutex_lock(&mtx2);



    sleep(1);



    pthread_mutex_lock(&mtx3);



    pthread_mutex_unlock(&mtx3);



    pthread_mutex_unlock(&mtx2);



    std::cout << __FUNCTION__ << " End..." << std::endl;



}



void* thread_routine_3(void*)



{



    std::cout << __FUNCTION__ << " Start..." << std::endl;



    pthread_mutex_lock(&mtx3);



    sleep(1);



    pthread_mutex_lock(&mtx4);



    pthread_mutex_unlock(&mtx4);



    pthread_mutex_unlock(&mtx3);



    std::cout << __FUNCTION__ << " End..." << std::endl;



}



void* thread_routine_4(void*)



{



    std::cout << __FUNCTION__ << " Start..." << std::endl;



    pthread_mutex_lock(&mtx4);



    sleep(1);



    pthread_mutex_lock(&mtx1);



    pthread_mutex_unlock(&mtx1);



    pthread_mutex_unlock(&mtx4);



    std::cout << __FUNCTION__ << " End..." << std::endl;



}



int main()



{



    pthread_t th1, th2,th3,th4;



    pthread_create(&th1,NULL,thread_routine_1,NULL);



    pthread_create(&th2,NULL,thread_routine_2,NULL);



    pthread_create(&th2, NULL, thread_routine_3, NULL);



    pthread_create(&th2, NULL, thread_routine_4, NULL);



 



    pthread_join(th1,NULL);



    pthread_join(th2,NULL);



    pthread_join(th3,NULL);



    pthread_join(th4,NULL);



    return 0;



}
```

## 1.3 简单理解记录

刚开始King老师演示了两个线程两个锁的情况，不过死锁的情况只是偶现，偶现的问题往往最难排查，后来在线程函数中加上sleep(1)，踏踏实实说一秒，线程抢占之间存在稳态就变为了必现，可以看出King老师对死锁这件事有一个超乎常人的看法。
代码为四个线程抢占四个锁的额资源，线程虽然只是增加了两个，但是代码的复杂程度直线飙升，大量重复冗余的代码也随之出现。在这里如果能运用函数指针进行赋值，感觉也许会更好些。
如果屏蔽掉线程四的内容，锁之间没有闭环，死锁瞬间解决，这会不会也是一个新的思路呢？当遇到复杂棘手的困难问题，也许先尽可能的简化，这才是更快的解决问题的方法吧？

## 1.4 死锁终端显示



![在这里插入图片描述](https://img-blog.csdnimg.cn/0ef5bde3578240169eb5d76e44692e76.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16)



# 2.pthread的hook

如何知道线程和锁的对应关系，哪把线程占用哪把锁？hook就是解决这个问题。

## 2.1 代码增改

### 2.1.1 pthread_mutex_lock_f、

```c
typedef int (*pthread_mutex_lock_t)(pthread_mutex_t * mutex);



pthread_mutex_lock_t pthread_mutex_lock_f;



typedef int (*pthread_mutex_unlock_t)(pthread_mutex_t * mutex);



pthread_mutex_unlock_t pthread_mutex_unlock_f;
```

- 这里笔者觉得完全可以定义成一个函数指针，因为笔者是个懒汉。

### 2.1.2 pthread_mutex_lock()

```cpp
int pthread_mutex_lock(pthread_mutex_t* mutex)



{



    std::cout << __FUNCTION__ << " Start..." << std::endl;



    std::cout <<"pthread="<<pthread_self()<< " mutex=" <<mutex<< std::endl;



    pthread_mutex_lock_f(mutex);



    std::cout << __FUNCTION__ << " End..." << std::endl;



}
```

### 2.1.3 pthread_mutex_unlock()

```cpp
int pthread_mutex_unlock(pthread_mutex_t* mutex)



{



    std::cout << __FUNCTION__ << " Start..." << std::endl;



    pthread_mutex_unlock_f(mutex);



    std::cout << __FUNCTION__ << " End..." << std::endl;



}
```

### 2.1.4 init_hook()

```c
static int init_hook()



{



    pthread_mutex_lock_f =dlsym(RTLD_NEXT,"pthread_mutex_lock");



    pthread_mutex_lock_f =dlsym(RTLD_NEXT,"pthread_unmutex_lock");



}
```

- 参数1：RTLD_NEXT 表示fd，从这个系统库中获取
- 参数2：pthread_mutex_lock: 第二个参数表示标签，是函数的位置
- 返回值：phtread_mutex_lock_f 表示函数指针，记录一个和目标函数同类型的地址

### 2.1.5 终端输入man dlsym，查看外挂专用函数dlsym()



![在这里插入图片描述](https://img-blog.csdnimg.cn/c085411425cd483b98b0aed715cb9aff.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16)



### 2.1.6 宏和头文件

- _GNU_SOUCE宏会在预编译阶段打开<dlfc.h>中的部分功能参与编译

```c
#define _GNU_SOUCE



#include <dlfc.h>



//int main()函数中记得执行初始化函数！



init_hook（）;
```

## 2.2 编译命令

- gcc -o deadlock deadlock.c -lpthread -ldl
- 记得加上库一起编译

## 2.3 死锁状态



![在这里插入图片描述](https://img-blog.csdnimg.cn/7fcf89b650254f6397b038fee1de5fb2.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16)



# 3.图的构建

dfs深度优先算法，依次访问每一个节点，标记置1，如果下一个节点已经访问了那说明有环产生。

## 3.1 原理

线程占用的锁存入locklist当中，其他线程再想占用时先前去申请。

## 3.2 结果



![在这里插入图片描述](https://img-blog.csdnimg.cn/ad77320173f241cd9f8f609bbc24fb03.png)



# 4.三个源语的构建

## 4.1 示意图



![在这里插入图片描述](https://img-blog.csdnimg.cn/9507a5a8b7b14ae7aa6b193df7f211df.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_18,color_FFFFFF,t_70,g_se,x_16)



## 4.2 分析图



![在这里插入图片描述](https://img-blog.csdnimg.cn/ce5545d40e404f6c96b37059cf7f3119.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16)



## 4.3 逻辑思考

追求女孩之前，要先去获得目标的情报，对方有没有男朋友？有没有谈恋爱的意愿，这样才能事半功倍。
当心仪对象发出自己的意愿，都应该有对应的逻辑考虑。当她愿意跟咱搞，咱应该先进行朋友圈官宣，然后让其抹去前任的记忆，再去拉手等更多肢体上的接触，要有条不紊的进行。
分手的时候，应该互相抹去共同的记忆，释放各自的资源，把她的昵称改为“渣女”，也许有一种爱真的叫放手吧！

## 4.4 代码增改

```c
#define MAX 100



enum Type{PROCESS,RESOURCE};



struct source_type{



        unint64 id;



        enum Type type;



        unint64 lock_id;



        int degrssl



}



struct vertex{



        struct source_typ s;



        struct vertex *next;



};



struct task_graph{



        struct vertex list[MAX];



        int num;



        struct source_type locklist[MAX];



        int lockidx;



    



        pthread_mutex_t mutex;



};



struct task_graph *tg=NULL;



int path[MAX+1];



int visiited[MAX];



int k=0;



int deadlock = 0;



struct vertex *create_vertex(){



    struct vertex *tex=(struct vertex *)malloc(sizeof(struct vertex));



    tx->s =type;



    tex->next=NULL;



    return tex;



};



int serach_vertex(){



    for(int i{};i<tg->num;i++){



            if(tg->list[i].s.type==type.type && tg->list[i].s.id == typpe.id){



                            return i;



            }



    }



    return -1;



};



void add_vertex(){



        if(serch_vertex(type)==-1){



        tg->list[tg->num].s=type;



        tg->list[tf->num].next=NULL;



        tg->num++;



    }



};



int add_edg(struct sourec_type from,struct source_type to){



        add_vrtex(from);



        add_vertex(to);



        struct vertex *v = &(tg->list[idx]);



        while(v !=NULL){



        if(v->s.id==j.id) return 1;



        v=v->nex;



        }



};



    return 0;



int remove_edge(struct sourece_type from,struct source_type to){



        int idxi=search_vertex(from);



        int idxj=search_vertex(to);



        if(idxi !=-1 && idxj !=-1){



            struct vertex *v=&tg->list[idx];



            struct vertex 



        }



    }  



int DFS(int idx) {



    struct vertex *ver = &tg->list[idx];



    if (visited[idx] == 1) {



 



        path[k++] = idx;



        print_deadlock();



        deadlock = 1;



        return 0;



    }



    visited[idx] = 1;



    path[k++] = idx;



    while (ver->next != NULL)



     {



        DFS(search_vertex(ver->next->s));



        k --;



        ver = ver->next;



    }



 



    return 1;



 



}    
```

# 5.启动线程检测

命令：gcc -o deadlock_succeess deadlock_success.c -lpthread -ldl

# 6.调试运行



![在这里插入图片描述](https://img-blog.csdnimg.cn/c14ba707f2d6427895786eb8250061c3.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16)



# 7.总结

通过本节King老师的讲解，小生对检测死锁组件这部分知识有了初步的认识。老师不仅传授了知识，在对人生伴侣方面也给了我一定得启发。尽管代码自己感觉还是没有吃透，图的构件这一部分知识并没有很好的掌握，但是我会继续努力的研究出个所以然，自己的心态过于急于求成，应该戒骄戒躁，踏踏实实把这块知识重新掌握。如果不能解决，就不要前进。
接下来还会利用C++11中的std::thread线程新特性写出一个不一样风格的检测死锁模块，各位敬请期待吧！

原文作者：[屯门山鸡叫我小鸡](https://blog.csdn.net/sinat_28294665)

原文链接：https://bbs.csdn.net/topics/604966093