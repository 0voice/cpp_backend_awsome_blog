# 【NO.541】Linux服务器开发,stl容器，智能指针，正则表达式(C++STL中的智能指针)

## 0.前言

C++标准库中有四种只能指针，即std::auto_ptr、std::unique_ptr、td::shared_ptr、std::weak_ptr。每一个都有适用的场合。智能指针的作用事帮助程序员管理动态分配的对象的生命周期，更有效的防止内存的泄露。
这四种智能指针中，std::auto_ptr是98年遗留的产物，其他三种是C++11标准推出的。std::auto_ptr已经完全被std::unique_ptr取代，所以这里也不在讨论。
有了智能指针，程序员的妈妈再也不必担心内存释放影响程序员的心情了，即便忘记了delete，系统也能够帮助程序员delete，交给智能指针大家都放心了。

## 1.独占式指针 std::unique_ptr

同一个时间内只有一个指针能够指向该对象，当然，该对象的拥有权也能移交出去。独占式智能指针应该是最优先考虑的，可以理解为专属所有权的概念。换句话说，同一时刻只有一个指针指向这个对象，**指针在对象在，指针亡对象亡。**

![在这里插入图片描述](https://img-blog.csdnimg.cn/fa7914b475f9497dad146f14cf0e676f.png)



```cpp
#include<iostream>



#include<string>



#include<memory>



class Test



{



public:



    Test() {}



    ~Test() {}//自己有析构函数



};



auto Function_10()



{



    return std::unique_ptr<std::string>(new std::string("I love Youzi"));



}



std::unique_ptr<std::string> Function_11()



{



    return std::unique_ptr< std::string>(new std::string("I love Youzi"));



}



void Function_12(std::string * arg)



{



    delete arg;



    arg = nullptr;



}



using fp = void(*)(std::string *);



typedef decltype(Function_12)* fp1;



int main()



{



    //初步认识



    std::unique_ptr<int> p1;



    if (nullptr == p1)



        std::cout << "p_test为空指针" << std::endl;



 



    std::unique_ptr<int> p2(new int(10));



 



    std::unique_ptr<int> p3 = std::make_unique<int>(10);



    auto p4 = std::make_unique<int>(10);



 



    //常用操作



    //1、不允许赋值、赋值等动作，只能移动不能复制的类型。



    std::unique_ptr<std::string> pa1(new std::string("I Love Ovoice"));



    //std::unique_ptr<std::string> pa2(pa1); error



    //std::unique_ptr<std::string> pa3=pa1;  error



    std::unique_ptr<std::string> pa4; 



    //pa4 = pa1; error



 



    //2、移动语义写法



    std::unique_ptr<std::string> pb1(new std::string("I Love Ovoice"));



    std::unique_ptr<std::string> pb2 = std::move(pb1);



 



    //3、放弃控制权 release()



    //放弃控制权后，返回裸指针，智能指针置空。返回的裸指针可以手工delete，也可以初始化新的，也可以去赋值。



    std::unique_ptr<std::string> pc1(new std::string("I Love Ovoice"));



    std::unique_ptr<std::string> pc2(pc1.release());



 



    //4、reset()成员函数



    //reset()不带参数时，释放对象，指针置空；有参数时，释放对象，指针指向新内存。



    std::unique_ptr<std::string> pd1(new std::string("I Love Ovoice"));



    pd1.reset();



    if (nullptr == pd1)



        std::cout << "pd1 == nullptr" << std::endl;



 



    std::unique_ptr<std::string> pd2(new std::string("I Love Ovoice"));



    std::unique_ptr<std::string> pd3(new std::string("I Love Ovoice"));



    pd3.reset(pd2.release());



    pd3.reset(new std::string("I Love Ovoice"));



 



    //5、释放智能指针 =nullptr



    std::unique_ptr<std::string> pe1(new std::string("I Love Ovoice"));



    pe1 = nullptr;                //高级啊,和指针一样的。



 



    //6、关于数组



    std::unique_ptr<int[]> pf1(new int[10]);



    pf1[0] = 11;



    pf1[1] = 22;



    pf1[2] = 33;



    



    //有析构函数的类类型，需要自己写删除器进行释放



    auto mydel = [](Test *p) {



        delete []p;



    };



    std::unique_ptr<Test[],decltype(mydel)> pf2(new Test[10], mydel);



    



    //7、获取裸指针 get()



    std::unique_ptr<std::string> pg1(new std::string("I Love Ovoice"));



    std::string * pg2 = pg1.get();



    const char* pg3 = pg2->c_str();



    *pg2 = "Ovoice is a good school!";



    const char* pg4 = pg2->c_str();//调试发现pg3和pg4内存地址不同，由string内部决定



 



    //8、*解引用



    std::unique_ptr<std::string> ph1(new std::string("I Love Ovoice"));



    const char* ph2 = ph1->c_str();



    *ph1 = "Ovoice is a good school!";



    const char* ph3 = ph1->c_str();



    //数组并没有解引用，注意别丢人啦



    std::unique_ptr<int[]> ph4(new int[10]);



    //*ph4; error



 



    //9、交换指向对象 swap()



    std::unique_ptr<std::string> pi1(new std::string("I Love Ovoice"));



    std::unique_ptr<std::string> pi2(new std::string("I Love Youzi"));



    std::swap(pi1,pi2); //个人推荐这种~



    pi1.swap(pi2);



 



    //10、转成std::share_ptr类型



    //share_ptr有一个右值构造函数可以直接将unique_ptr放入



    std::shared_ptr < std::string> pj1 = Function_10();



 



    std::unique_ptr<std::string> pj2(new std::string("I Love Youzi"));



    std::shared_ptr<std::string> pj3 = std::move(pj2);//pj2为空，pj3时share_ptr引用计数为1



 



    //11、被复制的例外



    std::unique_ptr < std::string> pk1;



    pk1 = Function_11();



    



    //12、删除器



    std::unique_ptr<std::string,fp> pl1(new std::string("I Love Youzi"),Function_12);



    std::unique_ptr<std::string, fp1> pl2(new std::string("I Love Youzi"), Function_12);



    std::unique_ptr<std::string, decltype(Function_12)*> pl3(new std::string("I Love Youzi"), Function_12);



    



    auto mydel = [](std::string* arg)



    {



        delete arg;



        arg = nullptr;



    };



    std::unique_ptr<std::string, decltype(mydel)> pl4(new std::string("I Love Youzi"), mydel);



    



    return 0;



}
```

## 2.强引用共享式指针 std::share_ptr

多个指针指向同一个对象，最后一个指针被销毁，引用计数为0，这个对象就会被销毁。



![img](https://img-community.csdnimg.cn/images/053432fa294542e59b524c72656368ab.png)



```cpp
#include<iostream>



#include<string>



#include<memory>



#include "CShare_ptr.h"



void Function_9(int *p)



{



    std::cout << __FUNCTION__ << "现在删除器被调用" << std::endl;



    delete p;



}



template<typename T>



void Function_10(T* p)



{



    std::cout << __FUNCTION__ << "现在删除器被调用" << std::endl;



    delete []p;



}



template<typename T>



std::shared_ptr<T> make_shared_array(size_t size)



{



    return std::shared_ptr<T>(new T[size],Function_10<T[]>);



}



 



auto lambda1 = [](int* p)



{



    std::cout << __FUNCTION__ << "现在删除器被调用" << std::endl;



    delete p;



};



auto lambda2 = [](int* p)



{



    std::cout << __FUNCTION__ << "现在删除器被调用" << std::endl;



    delete p;



};



 



int main()



{



    int i{};



    std::shared_ptr<int> p=std::make_shared<int>(10);



    //1、引用计数 use_count()



    std::cout << "****** 1、引用计数 use_count()" << std::endl;



    std::shared_ptr<int> pa1(new int (100));



    std::cout << ++i<<" pa1.use_count()=" << pa1.use_count() << std::endl;



    std::shared_ptr<int> pa2(pa1);



    std::cout << ++i<<" pa1.use_count()=" << pa1.use_count() << std::endl;



    std::shared_ptr<int> pa3=pa1;



    std::cout << ++i << " pa1.use_count()=" << pa1.use_count() << std::endl;



    std::cout << ++i << " pa3.use_count()=" << pa3.use_count() << std::endl;



 



    //2、是否独占 unique()



    std::cout << "****** 2、是否独占 unique()" << std::endl;



    std::shared_ptr<int> pb1(new int(100));



    if (pb1.unique())



    {



        std::cout << ++i<< " pb1 unique 返回真" << std::endl;



    }



    else



    {



        std::cout << ++i << "pb1 unique 返回假" << std::endl;



    }



 



    //3、指针重置 reset()



    std::cout << "****** 3、指针重置 reset()" << std::endl;



    //引用计数先减1，如果为0指向的对象就要清空。



    std::shared_ptr<int> pc1(new int(100));



    pc1.reset();



    if (nullptr == pc1)



    {



        std::cout <<++i<< " pc1被置空" << std::endl;



    }



    std::shared_ptr<int> pc2(new int(100));



    //释放原内存，指向新内存



    pc2.reset(new int(100));



    



    std::shared_ptr<int> pc3;



    //利用reset初始化空指针



    pc3.reset(new int(10));



    //4、*解引用



    std::cout << "******4、*解引用" << std::endl;



    std::shared_ptr<int> pd1(new int(100));



    char buf[1024];



    sprintf_s(buf,sizeof(buf),"%d",*pd1);



    //5、获取裸指针 get()



    std::cout << "******5、获取裸指针 get()" << std::endl;



    //如果智能指针释放了对象，裸指针也将失效！这给函数的目的是为没有用智能指针的代码提供可能



    std::shared_ptr<int> pe1(new int(100));



    int* pe2 = pe1.get();



    *pe2 = 200;



    //6、交换指针 swap()



    std::cout << "******6、交换指针 swap()" << std::endl;



    std::shared_ptr<int> pf1(new int(100));



    std::shared_ptr<int> pf2(new int(100));



    std::swap(pf1,pf2);



    pf1.swap(pf2);



    //7、赋空 =nullptr



    std::cout << "7、******赋空 =nullptr" << std::endl;



    std::shared_ptr<std::string> pg1(new std::string("I love Mark!"));



    pg1 = nullptr;



 



    //8、删除器和数组



    std::cout << "8、******8、删除器和数组" << std::endl;



    std::shared_ptr<int> ph1(new int(100),Function_9);



    std::shared_ptr<int> ph2(ph1);



    ph1.reset();



    ph2.reset();//现在删除器会被调用



 



    //std::shared_ptr<int> ph3 = make_shared_array<int>(10); 也可以选用std::default_delete<T[]>()



    



    std::shared_ptr<int> ph4(new int(20),lambda1);



    std::shared_ptr<int> ph5(new int(20),lambda2);



    ph5 = ph4;



}
```

## 3.弱引用共享式指针std::weak_ptr

是std::share_ptr 做辅助工作的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/9a0bd558223d4d1ab0d01be734e51538.png)



```cpp
#include "CWeak_ptr.h"



#include<iostream>



#include<string>



#include<memory>



int main()



{



    int i{};



    //1、引用计数 use_count()



    std::cout << "****** 1、引用计数 use_count()" << std::endl;



    auto pa1 = std::make_shared<int>(10);



    auto pa2(pa1);



    std::weak_ptr<int> pa3(pa1);



    std::cout << ++i << " pa3.use_count()=" << pa3.use_count() << std::endl;



 



    //2.过期 xpired()



    pa1.reset();



    pa2.reset();



    if (pa3.expired())



    {



        std::cout << ++i << " pa3.expired()为真" << std::endl;



    }



 



    //3、重置 reset()



    auto pb1 = std::make_shared<int>(10);



    std::weak_ptr<int> pb2(pb1);



    pb2.reset();//pb1为强引用，弱引用已经被重置



 



 



    //4、是否锁住 lock()



    std::cout << "****** 4、是否锁住 lock()" << std::endl;



    auto pc1 = std::make_shared<int>(20);



    std::weak_ptr<int> pc2;



    pc2 = pc1;



    if (!pc2.expired())



    {



        auto pc3 = pc2.lock();



        if (pc3 != nullptr)



        {



            std::cout << "pc3 != nullptr" << "所指对象不存在" << std::endl;



        }



    }



    else



    {



        std::cout << "pc2已经过期" << std::endl;



    }



    return 0;



}
```

## 4.总结

智能指针主要目的是不光是帮助程序员用来吹牛，主要是帮助释放内存，防止内存泄露。对于严谨的程序员内存泄露是低级错误，对于新手却是家常便饭。这里只是简单的浅谈了三种智能指针日常使用的方法，日常生活中还要注意慎用裸指针、enabl_shared_from_this、循环引用等问题，避免踩坑。笔者作为一个C++初学者，还是希望在项目中多多使用智能指针，年轻人毕竟上手能力比较强嘛！
最后，还是要感谢零声学院的Darren老师，在《Linux服务器开发：stl容器，智能指针，正则表达式(C++STL中的智能指针)》这门课中着重的介绍了智能指针，让我对C++11有了新的认识！正所谓“前人播种，后人乘凉。”希望老师们继续努力，砥砺前行。

原文作者：[屯门山鸡叫我小鸡](https://blog.csdn.net/sinat_28294665)

原文链接：https://bbs.csdn.net/topics/604955515