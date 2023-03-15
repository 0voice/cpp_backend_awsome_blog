# 【NO.303】C语言回调函数到底是什么？如何使用回调函数？

## **1.什么是回调函数？**

回调函数，光听名字就比普通函数要高大上一些，那到底什么是回调函数呢？恕我读得书少，没有在那本书上看到关于回调函数的定义。我在百度上搜了一下，发现众说纷纭，有很大一部分都是使用类似这么一个场景来说明：A君去B君店里买东西，恰好缺货，A君留下号码给B君，有货时通知A君。感觉这个让人更容易想到的是异步操作，而不是回调。

另外还有两句英文让我印象深刻：1) If you call me, I will call you back; 2) Don't call me, I will call you. 看起来好像很有道理，但是仔细一想，普通函数不也可以做到这两点吗？所以，我觉得这样的说法都不是很妥当，因为我觉得这些说法都没有把回调函数的特点表达出来，也就是都看不到和普通函数到底有什么差别。

不过，百度百科的解析我觉得还算不错（虽然经常吐槽百度搜索...）：回调函数就是一个通过函数指针调用的函数。如果你把函数的指针（地址）作为参数传递给另一个函数，当这个指针被用来调用其所指向的函数时，我们就说这是回调函数。

下面先说说我的看法。我们可以先在字面上先做个分解，对于"回调函数"，中文其实可以理解为这么两种意思：1) 被回调的函数；2) 回头执行调用动作的函数。那这个回头调用又是什么鬼？

**先来看看来自维基百科的对回调（Callback）的解析**：In computer programming, a callback is any executable code that is passed as an argument to other code, which is expected to call back (execute) the argument at a given time. This execution may be immediate as in a synchronous callback, or it might happen at a later time as in an asynchronous callback. 也就是说，把一段可执行的代码像参数传递那样传给其他代码，而这段代码会在某个时刻被调用执行，这就叫做回调。如果代码立即被执行就称为同步回调，如果在之后晚点的某个时间再执行，则称之为异步回调。关于同步和异步，这里不作讨论，请查阅相关资料。

**再来看看来自Stack Overflow某位大神简洁明了的表述：**A "callback" is any function that is called by another function which takes the first function as a parameter。也就是说，函数 F1 调用函数 F2 的时候，函数 F1 通过参数给 函数 F2 传递了另外一个函数 F3 的指针，在函数 F2 执行的过程中，函数F2 调用了函数 F3，这个动作就叫做回调（Callback），而先被当做指针传入、后面又被回调的函数 F3 就是回调函数。到此应该明白回调函数的定义了吧？

## **2.为什么要使用回调函数？**

很多朋友可能会想，为什么不像普通函数调用那样，在回调的地方直接写函数的名字呢？这样不也可以吗？为什么非得用回调函数呢？有这个想法很好，因为在网上看到解析回调函数的很多例子，其实完全可以用普通函数调用来实现的。要回答这个问题，我们先来了解一下回到函数的好处和作用，那就是解耦，对，就是这么简单的答案，就是因为这个特点，普通函数代替不了回调函数。所以，在我眼里，这才是回调函数最大的特点。来看看维基百科上面我觉得画得很好的一张图片。

![img](https://pic2.zhimg.com/80/v2-6b541dce8cfd665f1199cedca95287a5_720w.webp)

```text
    #include<stdio.h>
    #include<softwareLib.h> // 包含Library Function所在读得Software library库的头文件

    int Callback() // Callback Function
{
        // TODO
        return 0;
    }
    int main() // Main program
{
        // TODO
        Library(Callback);
        // TODO
        return 0;
    }
```

乍一看，回调似乎只是函数间的调用，和普通函数调用没啥区别，但仔细一看，可以发现两者之间的一个关键的不同：在回调中，主程序把回调函数像参数一样传入库函数。这样一来，只要我们改变传进库函数的参数，就可以实现不同的功能，这样有没有觉得很灵活？并且丝毫不需要修改库函数的实现，这就是解耦。

再仔细看看，主函数和回调函数是在同一层的，而库函数在另外一层，想一想，如果库函数对我们不可见，我们修改不了库函数的实现，也就是说不能通过修改库函数让库函数调用普通函数那样实现，那我们就只能通过传入不同的回调函数了，这也就是在日常工作中常见的情况。现在再把main()、Library()和Callback()函数套回前面 F1、F2和F3函数里面，是不是就更明白了？

明白了回调函数的特点，是不是也可以大概知道它应该在什么情况下使用了？没错，你可以在很多地方使用回调函数来代替普通的函数调用，但是在我看来，如果需要降低耦合度的时候，更应该使用回调函数。

## **3.怎么使用回调函数？**

知道了什么是回调函数，了解了回调函数的特点，那么应该怎么使用回调函数？下面来看一段简单的可以执行的同步回调函数代码。

```text
    #include<stdio.h>

    int Callback_1() // Callback Function 1
{
        printf("Hello, this is Callback_1 \n");
        return 0;
    }

    int Callback_2() // Callback Function 2
{
        printf("Hello, this is Callback_2 \n");
        return 0;
    }

    int Callback_3() // Callback Function 3
{
        printf("Hello, this is Callback_3 \n");
        return 0;
    }

    int Handle(int (*Callback)())
{
        printf("Entering Handle Function.\n ");
        Callback();
        printf("Leaving Handle Function.\n ");
    }

    int main()
{
        printf("Entering Main Function.\n ");
        Handle(Callback_1);
        Handle(Callback_2);
        Handle(Callback_3);
        printf("Leaving Main Function.\n");
        return 0;
    }
```

运行结果：

> Entering Main Function.
> Entering Handle Function.
> Hello, this is Callback_1
> Leaving Handle Function.
> Entering Handle Function.
> Hello, this is Callback_2
> Leaving Handle Function.
> Entering Handle Function.
> Hello, this is Callback_3
> Leaving Handle Function.
> Leaving Main Function.

可以看到，Handle()函数里面的参数是一个指针，在main()函数里调用Handle()函数的时候，给它传入了函数Callback_1()/Callback_2()/Callback_3()的函数名，这时候的函数名就是对应函数的指针，也就是说，回调函数其实就是函数指针的一种用法。现在再读一遍这句话：A "callback" is any function that is called by another function which takes the first function as a parameter，是不是就更明白了呢？

## **4.怎么使用带参数的回调函数？**

眼尖的朋友可能发现了，前面的例子里面回调函数是没有参数的，那么我们能不能回调那些带参数的函数呢？答案是肯定的。那么怎么调用呢？我们稍微修改一下上面的例子就可以了：

```text
      #include<stdio.h>

    int Callback_1(int x) // Callback Function 1
{
        printf("Hello, this is Callback_1: x = %d ", x);
        return 0;
    }

    int Callback_2(int x) // Callback Function 2
{
        printf("Hello, this is Callback_2: x = %d ", x);
        return 0;
    }

    int Callback_3(int x) // Callback Function 3
{
        printf("Hello, this is Callback_3: x = %d ", x);
        return 0;
    }

    int Handle(int y, int (*Callback)(int))
{
        printf("Entering Handle Function. ");
        Callback(y);
        printf("Leaving Handle Function. ");
    }

    int main()
{
        int a = 2;
        int b = 4;
        int c = 6;
        printf("Entering Main Function. ");
        Handle(a, Callback_1);
        Handle(b, Callback_2);
        Handle(c, Callback_3);
        printf("Leaving Main Function. ");
        return 0;
    }
```

运行结果：

> Entering Main Function.Entering Handle Function.Hello, this is Callback_1: x = 2Leaving Handle Function.Entering Handle Function.Hello, this is Callback_2: x = 4Leaving Handle Function.Entering Handle Function.Hello, this is Callback_3: x = 6Leaving Handle Function.Leaving Main Function.

可以看到，并不是直接把int Handle(int (*Callback)()) 改成 int Handle(int (*Callback)(int)) 就可以的，而是通过另外增加一个参数来保存回调函数的参数值，像这里 int Handle(int y, int (*Callback)(int)) 的参数 y。同理，可以使用多个参数的回调函数。

## **5.参考练习**

```text
#include <stdio.h>
typedef  void (*listen)(int);

listen mlisten[3];

void register_observer(listen obs)
{
  for(int i=0;i<3;i++)
  {
    if(mlisten[i] == 0)
    {
      mlisten[i] = obs;
      return ;
    }  
  }
}

void listen0(int i)
{
  printf("listen0 received i=%d\n",i);
}
void listen1(int i)
{
  printf("listen1 received i=%d\n",i);
}
void listen2(int i)
{
  printf("listen2 received i=%d\n",i);
}

void notify_all_observer(int val)
{
  for(int i=0;i<sizeof(mlisten)/sizeof(mlisten[0]);i++)
  {
    if(mlisten[i] != 0)
    {
      mlisten[i](val);
    }  
  }
}
int main()
{
  int i=0;
  printf("lis1:%d\n",listen0);
  register_observer(listen0);
  printf("lis2:%d\n",listen1);
  register_observer(listen1);
  register_observer(listen2);

  while(1)
  {
    scanf("%d\n", &i);
    printf("%d\n",i);
    notify_all_observer(i);
  }
}
```

原文地址：https://zhuanlan.zhihu.com/p/537404427

作者：linux