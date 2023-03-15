# 【NO.321】c++开发工作中常见的几种内存泄漏场景汇总

> 内存泄漏（Memory Leak）是指程序中已动态分配的堆内存由于某种原因程序未释放或无法释放，造成系统内存的浪费，导致程序运行速度减慢甚至系统崩溃等严重后果。

作为C/C++程序员，谁还不写Bug，Bug里面的王者要数内存泄漏，内存泄漏具有其独有的属性，比如说：隐蔽性强、难以排查、占用资源不断累积等特点，更甚者是会让人想要摔键盘……

本文主要是对工作中经常遇到的内存泄漏场景进行总结，与君共勉。

## **1.在构造函数中抛出异常**

```text
class Test
{
public:
    Test(int iFlag)
    {
        m_pBuf = new char[4*1024*1024];
        if(!iFlag)
        {
            throw("Exception:验证构造函数抛出异常");
        }
    }
    ~Test()
    { 
        std::cout<<"释放类中申请的资源"<<std::endl;
        delete [] m_pBuf;
    }
private:
    char *m_pBuf;
};


int main()
{
    Test cTest(0);
    return 0;
}
```

如上面的代码所示，代码在编译器中编译正常，声明cTest类实例后如果传入参数为真，则可以正常运行且类中new的资源可以正常释放。但是当传入参数为0时，运行代码后抛出异常。进程退出，异常信息如下图所示：



![img](https://pic2.zhimg.com/80/v2-5ad841016aad40461c8a6feb1b40a0bd_720w.webp)



从结果可以看出，抛出异常后代码退出，但是类的析构函数没有被调用。这也说明如果在构造函数中抛出异常，类的析构函数是不会被调用的。所以如果要在构造函数中使用抛出异常，那么切记，一定要在抛出异常前对申请的资源进行正确的释放。反之，就像上面的代码一样，产生内存泄漏的风险。如果要将上面的代码修改正确，可以做如下修改：

```text
Test(int iFlag)
    {
        m_pBuf = new char[4*1024*1024];
        if(!iFlag)
        {
            delete [] m_pBuf;
            throw("Exception:验证构造函数抛出异常");
        }
    }
```

## **2.匿名对象产生的内存泄漏**

如下面的代码所示，代码功能定义一个临时的对象，定义好后没有使用指针对其进行指向，在程序退出时，临时对象申请的资源就不会进行释放，使用内存检测工具后，就会提示内存泄漏风险。

```text
int main()
{
    std::cout<< (*new std::string("hello world"))<<std::endl;
    return 0;
}
```

如代码所示，上面这段代码既能通过编译，又能输出正确的结果，但是却存在内存泄漏风险。这是因为在上面的代码中使用了new在堆上申请了资源且成功分配了空间。如果在代码运行结束后没有使用delete进行释放，那么就会产生内存泄漏。如果想逃过内存泄漏工具的检测，只要将*new std::string("hello world")赋值给一个指针，然后在退出前对指针进行释放即可。

## **3.基类中的析构函数引发的内存泄露**

在C++中，如果子类的对象是通过基类的指针进行删除，如果基类的析构函数不是虚拟的，那么子类的析构函数可能不会被调用，从而导致派生类资源没有被释放，进而产生内存泄漏。如下面的代码所示：

```text
class TBase
{
public:
    virtual int DoSomeThing(){std::cout<<"TBase::DoSomeThing"<<std::endl;}
    virtual ~TBase(){}
};


class Test:public TBase
{
public:
    Test()
    {
        m_pBuf = new char[12];
        memset(m_pBuf,0x0,sizeof(m_pBuf));
    }
    ~Test()
    { 
        std::cout<<"Test子类释放类中申请的资源"<<std::endl;
        delete [] m_pBuf;
    }
    
    int DoSomeThing(){
        std::cout<<"Test::DoSomeThing"<<m_pBuf<<std::endl;
        return 0;
    }
private:
    char *m_pBuf;
};


int main()
{
    TBase *pBase = new Test();
    delete pBase;
    return 0;
}
```

如代码所示，在main函数中定义了一个基类的指针，并指向其子类对象，随后对基类指针进行释放，本意是想通过对基类指针释放同时也调用子类的析构函数释放子类资源。但是事与愿违，运行后，并没有调用子类的析构函数。这是因为，在基类中并没有定义析构函数，在这种情况下，编译器会为我们默认生成一个析构函数，但还不够智能，生成的析构函数不是虚拟的，这样在对基类指针进行析构时就不能调用子类析构函数释放资源。如果想要达到我们想要的效果，只需将基类中的析构函数定义成虚析构即可。修改后运行结果如图所示：



![img](https://pic2.zhimg.com/80/v2-2960313f603ebbd6b62d8a1056f5f46d_720w.webp)



可见，子类中的资源得到正常释放。

## **4.void\*指针产生的内存泄漏**

如下面的代码所示，定义一个类对象后，本意想在类外统一定义一个资源释放接口对资源进行释放，但是事情却没有向我们想象的地方进行。依旧使用上面的代码，我们在这里只新增一个资源释放接口并在main函数中修改调用。

```text
void deleteObj(void *pObj)
{
    delete pObj;
    std::cout<<"delete pObj"<<std::endl;
}
int main()
{
    Test *pBase = new Test();
    deleteObj(pBase);
    return 0;
}
```

上面的代码乍一看是没什么问题的，但是仔细考虑下，将pBase传入到deleteObj后，pBase被转换成void*类型，然后再释放。但是这样做就破坏了delete的工作原理，delete删除对象时，先调用对象的析构函数，再delete指针对象，上面的代码在将pBase转换成void*后，delete获取不到析构对象的类型就不能正确调用对象的析构函数，因此导致析构对象可能产生内存泄漏。

## **5.容器元素产生的内存泄漏**

容器元素产生的内存泄漏主要是当容器中的元素为指针时，每次new一个对象都会将指针保存在容器中，清理容器时，容器中的指针对象不会同时被清理。

在编码时，如果容器中保存的是对象，那么容器会自动对对象进行清理，但如果是指针，则需要编码人员手动对容器中保存的指针进行清理，如下面的代码所示,在这里依旧复用前面章节的代码，本次只对main函数进行改造：

```text
int main()
{
    std::vector<Test *> vTest;
    for(int i=0;i<10;i++)
    {
        vTest.push_back(new Test());
    }
    vTest.clear();
    return 0;
}
```

如上所示，程序结束时仅使用clear方法对vector资源进行清理，但是，保存在vector中的指针对象并没有被清理掉，需要我们手动进行处理，要想得到正确的代码，按照如下方式进行修改：

```text
for(int i=0;i<vTest.size();i++)
{
    delete vTest[i];
}
```

代码运行后，从结果可知，资源被正确释放，如下图所示：

![img](https://pic2.zhimg.com/80/v2-e5c1bd03fb70322e1c7a20927eb0aedd_720w.webp)

## **6.写在最后**

开发有bug是正常的，要学会学习bug，不断的总结编程过程中遇到的问题，才能让我们更好的解决问题，最终把问题消灭在写代码之前。

学习地址：https://zhuanlan.zhihu.com/p/519933572

作者：linux