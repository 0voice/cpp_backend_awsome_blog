# 【NO.20】一文掌握google开源单元测试框架Google Test

我们在开发的过程中，需要做一些验证测试，来保证我们的代码是按照设计要求工作的，这就需要单元测试了。单元测试（Unit Test），我们称为“UT测试”。对于一个复杂的系统来说，需要编写大量的单元测试用例，有人会觉得这么多的测试代码，将会花费大量的时间，影响开发的进度，会得不偿失。真的是这样吗？其实，对于越是复杂的系统就越是需要单元测试来保证我们的代码的开发质量，及时测试出代码的问题，在开发阶段发现问题总比在系统发布之后发现问题能够较少的节省资源或成本。

对于单元测试应该是每个开发工程师必备的技能，尤其是高阶的开发工程师会更加注重UT的重要性。同时，我们在开发功能模块之前会考虑到测试用例的实现，这样自然的就会考虑到功能模块的模块化便于UT的编写，从这一方面来说也能提高开发人员开发的代码质量。另外，单元测试用例还可以作为示例供开发人员参考，从而能够更轻松的掌握模块的使用。

今天就和大家一起学习一个开源的C++的单元测试框架Google test，大家看名字就知道它是由牛逼的Google公司出品。Google Test可以在多种平台上使用，它可以支持：

> Linux、Mac OS X、Windows、Cygwin、MinGW、Windows Mobile、Symbian、PlatformIO等。

## 1.安装和配置

我们可以从github获取Google Test的源码。

> github下载地址: [https://github.com/google/googletest](https://link.zhihu.com/?target=https%3A//github.com/google/googletest)

因为我们下载到的gTest是源代码，还需要将其编译成库文件再进行使用。下面将和大家一起学习如何在windows环境下生成gTest的库文件。在这之前我们需要安装CMake和MinGW。

将下载的gTest的源码进行解压，源码目录如下图所示。

![img](https://pic2.zhimg.com/80/v2-7b5580b3d8dc932c2d76c48b91c6fa8d_720w.webp)

打开命令行工具cmd，进入源码的工程目录，新建一个build目录用来存放构建文件，然后，进入build目录执行cmake命令生成Makefile文件。

```
mkdir build
cd build
cmake -G "MinGW Makefiles" ..
```

![img](https://pic3.zhimg.com/80/v2-e0b2d6fae1f4dfc369f7c5c268b81cba_720w.webp)

![img](https://pic1.zhimg.com/80/v2-16d08ea7934f23af7e679e0604f49cf4_720w.webp)

Makefile文件生成后，再执行下面的命令mingw32-make编译库文件。编译成功后就会发现有libgtest.a 和libgtest_main.a两个静态库生成。这里注意，Windows下mingw安装的make工具名称是mingw32-make而不是make。

```
mingw32-make
```

![img](https://pic3.zhimg.com/80/v2-6ca799fb7be17f7e6d7c433946a23a46_720w.webp)

接下来我们在VS Code写一个测试用例，使用生成的gTest静态库测试下。按下快捷键【Ctrl+Shift+p】，在弹出的搜索框中搜索【C/C++:Edit Configurations】，可以创建c_cpp_properties.json配置文件。

![img](https://pic3.zhimg.com/80/v2-98dccb609639e53d7b43a7eea1d0994e_720w.webp)

在c_cpp_properties.json配置文件添加gTest的头文件目录。

![img](https://pic2.zhimg.com/80/v2-cf8f7d5c0c7c9a16715bcda0dbf480ad_720w.webp)

在task.json配置文件中添加gTest头文件目录和库文件，task.json配置文件可以通过菜单栏中Terminal选项下的【Configure Default Build Task】选项创建。

![img](https://pic2.zhimg.com/80/v2-b8e5517b49abaeea7d07f8bc1922436d_720w.webp)

![img](https://pic3.zhimg.com/80/v2-f1c2ed2f310d64f7e1b3ed82b9593f72_720w.webp)

上面配置好之后，我们写个测试用例跑一下。

```
#include <iostream>
#include <gtest/gtest.h>

int add(int a, int b)
{
    return a + b;
}

int sub(int a, int b)
{
    return a - b;
}

TEST(testcase, test_add)
{
    EXPECT_EQ(add(1,2), 3);
    EXPECT_EQ(sub(1,2), -1);
}

int main(int argc, char **argv)
{  
    std::cout << "run google test --> " << std::endl << std::endl;
    testing::InitGoogleTest(&argc, argv);  
    return RUN_ALL_TESTS(); 
} 
```

运行结果如下图所示，代码中的TEST是一个宏，用来创建测试用例，它有test_case_name和test_name两个参数。分别是测试用例名和测试名，在后面的文章中我们会对其有更深刻的理解，这里就不细说了。RUN_ALL_TESTS也是一个宏，它是测试用例的入口。EXPECT_EQ这个是一个断言相关的宏，用来检测两个数值是否相等。

![img](https://pic4.zhimg.com/80/v2-cd6a9066f65330ba7c79183434cab35f_720w.webp)



## 2.断言

除了上面示例里的EXPECT_EQ，在gTest里有很多断言相关的宏。断言可以检查出某些条件的真假，因此，我们可以通过它来判断被测试的函数的成功与否。这里断言我们主要可以分为两类：

- 以"ASSERT_"开头的断言，致命性断言（Fatal assertion）
- 以"EXPECT_"开头的断言 ，非致命性断言（Nonfatal assertion）

上面的两种断言会在断言条件不满足时会有区别，即当不满足条件时, "ASSERT*"断言会在当前函数终止，而不会继续执行下去；而"EXPECT*"则会继续执行。我们可以通过下面一个例子来理解下他们的区别。

```
#include <iostream>
#include <gtest/gtest.h>

int add(int a, int b)
{
    return a + b;
}

int sub(int a, int b)
{
    return a - b;
}

TEST(testcase, test_expect)
{
    std::cout << "------ test_expect start-----" << std::endl;

    std::cout << "add function start" << std::endl;
    EXPECT_EQ(add(1,2), 2);
    std::cout << "add function end" << std::endl;

    std::cout << "sub function start" << std::endl;
    EXPECT_EQ(sub(1,2), -1);
    std::cout << "sub function end" << std::endl;

    std::cout << "------ test_expect end-----" << std::endl;
}

TEST(testcase, test_assert)
{

    std::cout << "------ test_assert start-----" << std::endl;

    std::cout << "add function start" << std::endl;
    ASSERT_EQ(add(1,2), 2);
    std::cout << "add function end" << std::endl;

    std::cout << "sub function start" << std::endl;
    ASSERT_EQ(sub(1,2), -1);
    std::cout << "sub function end" << std::endl;

    std::cout << "------ test_assert end-----" << std::endl;
}

int main(int argc, char **argv)
{  
    testing::InitGoogleTest(&argc, argv);  
    return RUN_ALL_TESTS(); 
}
```

从下面的运行结果上看，assert断言检查出被测函数add不满足条件，所以程序就没有继续执行下去；而expect虽然检查出被测试函数add不满足条件，但是程序还是继续去测试sub函数。

![img](https://pic2.zhimg.com/80/v2-1bb29da307c1538401cae497c54f03a9_720w.webp)

上面的示例用到的都是判断相等条件的断言，还有其他条件检查的断言。主要可以分为布尔检查，数值比较检查，字符串检查，浮点数检查，异常检查等等。下面我们逐一认识这些断言。

## 3.布尔检查

布尔检查主要用来检查布尔类型数据，检查其条件是真还是假。

![img](https://pic2.zhimg.com/80/v2-10bf9d8ab9ef57a35585b70f9c3431e5_720w.webp)

## 4.数值比较检查

数值比较检查主要用来比较两个数值之间的大小关系，这里有两个参数。

![img](https://pic1.zhimg.com/80/v2-579377a5007471a517c98107c2f101dc_720w.webp)

## 5.字符串检查

字符串检查主要用来比较字符串的内容。

![img](https://pic3.zhimg.com/80/v2-7c9703ad92123714ea41fc8c91e75512_720w.webp)

## 6.浮点数检查

对于浮点数来说，因为其精度原因，我们无法确定其是否完全相等，实际上对于浮点数我比较两个浮点数近似相等。

![img](https://pic1.zhimg.com/80/v2-c18380c78fa634142e3db9d98837f360_720w.webp)

## 7.异常检查

异常检查可以将异常转换成断言的形式。

![img](https://pic2.zhimg.com/80/v2-64d6bd623e2aa8ccafbfc4250ec6499d_720w.webp)

除了上面的一些类型的断言，还有一切其他的常用断言。

## 8.显示成功或失败

这一类断言会在测试运行中标记成功或失败。它主要有三个宏：

- SUCCED()：标记成功。
- FAIL() : 标记失败，类似ASSERT断言标记致命错误；
- ADD_FAILURE()：标记，类似EXPECT断言标记非致命错误。

```
#include <iostream>
#include <gtest/gtest.h>

int divison(int a, int b)
{
    return a / b;
}

TEST(testCaseTest, test0)
{
    std::cout << "start test 0" << std::endl;
    SUCCEED();
    std::cout << "test pass" << std::endl;
}

TEST(testCaseTest, test1)
{
    std::cout << "start test 1" << std::endl;
    FAIL();
    std::cout << "test fail" << std::endl;
}

TEST(testCaseTest, test2)
{
    std::cout << "start test 2" << std::endl;
    ADD_FAILURE();
    std::cout << "test fail" << std::endl;
}


int main(int argc, char **argv)
{  
    testing::InitGoogleTest(&argc, argv);  
    return RUN_ALL_TESTS(); 
} 
```

执行结果如下：

![img](https://pic4.zhimg.com/80/v2-c65dd0527dd2d1bb9b8b9a19456250c3_720w.webp)

## 9.死亡测试

死亡测试是用来检测测试程序是否按照预期的方式崩溃。

![img](https://pic3.zhimg.com/80/v2-767245c40ecd7fd113fcaabed8cc560a_720w.webp)

```
#include <iostream>
#include <gtest/gtest.h>

int divison(int a, int b)
{
    return a / b;
}

TEST(testCaseDeathTest, test_div)
{
    EXPECT_DEATH(divison(1, 0), "");
}
int main(int argc, char **argv)
{  
    testing::InitGoogleTest(&argc, argv);  
    return RUN_ALL_TESTS(); 
} 
```

上面这个例子就是死亡测试，其运行结果如下，这里需要注意的是test_case_name如果使用DeathTest为后缀，gTest会优先运行。

![img](https://pic4.zhimg.com/80/v2-6c91f252fa01982c02664fc7e69f5be7_720w.webp)

## 10.测试事件

在学习测试事件之前，我们先来了解下三个概念，它们分别是测试程序，测试套件，测试用例。

- 测试程序是一个可执行程序，它有一个测试程序的入口main函数。
- 测试用例是用来定义需要验证的内容。
- 测试套件是测试用例的集合，运行测试。

我们回过来看测试事件，在GTest中有了测试事件的这个机制，就能能够在测试之前或之后能够做一些准备/清理的操作。根据事件执行的位置不同，我们可将测试事件分为三种：

- TestCase级别测试事件：这个级别的事件会在TestCase之前与之后执行；
- TestSuite级别测试事件：这个级别的事件会在TestSuite中第一个TestCase之前与最后一个TestCase之后执行；
- 全局测试事件：这是级别的事件会在所有TestCase中第一个执行前，与最后一个之后执行。

这些测试事件都是基于类的，所以需要在类上实现。下面我们依次来学习这三种测试事件。

## 11.TestCase测试事件

TestCase测试事件，需要实现两个函数SetUp()和TearDown()。

- SetUp()函数是在TestCase之前执行。
- TearDown()函数是在TestCase之后执行。

这两个函数是不是有点像类的构造函数和析构函数，但是切记他们并不是构造函数和析构函数，只是打个比方才这么说而已。我们可以借助下面的代码示例来加深对它的理解。这两个函数是testing::Test的成员函数，我们在编写测试类时需要继承testing::Test。

```
#include <iostream>
#include <gtest/gtest.h>

class calcFunction
{
public:
    int add(int a, int b)
    {
        return a + b;
    }

    int sub(int a, int b)
    {
        return a - b;
    }
};

class calcFunctionTest : public testing::Test
{
protected:
    virtual void SetUp()
    {
        std::cout << "--> " << __func__ << " <--" <<std::endl;
    }
    virtual void TearDown()
    {
        std::cout << "--> " << __func__ << " <--" <<std::endl;
    }

    calcFunction calc;

};

TEST_F(calcFunctionTest, test_add)
{
    std::cout << "--> test_add start <--" << std::endl;
    EXPECT_EQ(calc.add(1,2), 3);
    std::cout << "--> test_add end <--" << std::endl;
}

TEST_F(calcFunctionTest, test_sub)
{
    std::cout << "--> test_sub start <--" << std::endl;
    EXPECT_EQ(calc.sub(1,2), -1);
    std::cout << "--> test_sub end <--" << std::endl;
}

int main(int argc, char **argv)
{  
    testing::InitGoogleTest(&argc, argv);  
    return RUN_ALL_TESTS(); 
} 
```

测试结果如下，两个函数都是是在每个TestCase（test_add和test_sub）之前和之后执行。

![img](https://pic3.zhimg.com/80/v2-a50073b96ec0cbfb70f47c477314731a_720w.webp)

## 12.TestSuite测试事件

TestSuite测试事件，同样的也需要实现的两个函数SetUpTestCase()和TearDownTestCase()，而这两个函数是静态函数。这两个静态函数同样也是testing::Test类的成员，我们直接改写下测试类calcFunctionTest，添加两个静态函数SetUpTestCase()和TearDownTestCase()到测试类中即可。

```
class calcFunctionTest : public testing::Test
{
protected:
    static void SetUpTestCase()
    {
        std::cout<< "--> " <<  __func__ << " <--" << std::endl;
    }

    static void TearDownTestCase()
    {
        std::cout<< "--> " << __func__ << " <--" << std::endl;
    }

    virtual void SetUp()
    {
        std::cout << "--> " << __func__ << " <--" <<std::endl;
    }
    virtual void TearDown()
    {
        std::cout << "--> " << __func__ << " <--" <<std::endl;
    }

    calcFunction calc;

};
```

改写好之后，我们再看一下运行结果。这两个函数分别是在本TestSuite中的第一个TestCase之前和最后一个TestCase之后执行。

![img](https://pic2.zhimg.com/80/v2-f9af1935b96f835960a4921fd2d07a2d_720w.webp)

## 13.全局测试事件

全局测试事件，也需要继承一个类，但是它需要继承testing::Environment类实现SetUp()和TearDown()两个函数。还需要在main函数中调用testing::AddGlobalTestEnvironment方法注册全局事件。我们直接上代码吧！

```
#include <iostream>
#include <gtest/gtest.h>

class calcFunction
{
public:
    int add(int a, int b)
    {
        return a + b;
    }

    int sub(int a, int b)
    {
        return a - b;
    }
};

class calcFunctionEnvironment : public testing::Environment
{
    public:
        virtual void SetUp()
        {
            val = 123;
            std::cout << "--> Environment " << __func__ << " <--" << std::endl;
        }
        virtual void TearDown()
        {
            std::cout << "--> Environment " << __func__ << " <--" << std::endl;
        }

        int val;
};

calcFunctionEnvironment* calc_env;

class calcFunctionTest : public testing::Test
{
protected:
    static void SetUpTestCase()
    {
        std::cout<< "--> " <<  __func__ << " <--" << std::endl;
    }

    static void TearDownTestCase()
    {
        std::cout<< "--> " << __func__ << " <--" << std::endl;
    }

    virtual void SetUp()
    {
        std::cout << "--> " << __func__ << " <--" <<std::endl;
    }
    virtual void TearDown()
    {
        std::cout << "--> " << __func__ << " <--" <<std::endl;
    }

    calcFunction calc;

};

TEST_F(calcFunctionTest, test_add)
{
    std::cout << "--> test_add start <--" << std::endl;
    EXPECT_EQ(calc.add(1,2), 3);
    std::cout << "Global Environment val = " << calc_env->val << std::endl;
    std::cout << "--> test_add end <--" << std::endl;
}

TEST_F(calcFunctionTest, test_sub)
{
    std::cout << "--> test_sub start <--" << std::endl;
    EXPECT_EQ(calc.sub(1,2), -1);
    std::cout << "Global Environment val = " << calc_env->val << std::endl;
    std::cout << "--> test_sub end <--" << std::endl;
}

int main(int argc, char **argv)
{  
    calc_env = new calcFunctionEnvironment;
    testing::AddGlobalTestEnvironment(calc_env);

    testing::InitGoogleTest(&argc, argv);  
    return RUN_ALL_TESTS(); 
} 
```

从测试结果上看，全局事件的这两个函数分别是在第一个TestSuite之前和最后一个TestSuite之后执行的。

![img](https://pic3.zhimg.com/80/v2-1bff7a98385fa963b4b01853d2c53766_720w.webp)

以上三种测试事件我们可以根据需要进行灵活使用。另外，细心的同学会发现，这里测试用例我们该用了TEST_F这个宏，这是因为继承了testing::Test,与之对应就需要使用TEST_F宏。

## 14.参数化

在学习gTest参数化之前我们先看一个测试例子。

```
#include <iostream>
#include <gtest/gtest.h>

class calcFunction
{
public:
    int add(int a, int b)
    {
        std::cout << a << " + " << b << " = " << a + b << std::endl;
        return a + b;
    }

    int sub(int a, int b)
    {
        std::cout << a << " - " << b << " = " << a - b << std::endl;
        return a - b;
    }
};

class calcFunctionTest : public testing::Test
{
protected:
    calcFunction calc;
};

TEST_F(calcFunctionTest, test_add0)
{
    EXPECT_EQ(calc.add(1,2), 3);
}

TEST_F(calcFunctionTest, test_add1)
{
    EXPECT_EQ(calc.add(1,3), 4);
}

TEST_F(calcFunctionTest, test_add2)
{
    EXPECT_EQ(calc.add(2,4), 6);
}

TEST_F(calcFunctionTest, test_add3)
{
    EXPECT_EQ(calc.add(-1,-2), -3);
}

int main(int argc, char **argv)
{  
    testing::InitGoogleTest(&argc, argv);  
    return RUN_ALL_TESTS(); 
} 
```

示例执行结果：

![img](https://pic4.zhimg.com/80/v2-4d528b76739ff7ef73f346f794d91103_720w.webp)

上面的测试用例中我们写了多个测试用例，但是其参数都是同样的，有的实际应用场景可能比这个程序写的测试检查还要多。写这么多重复的代码实在是太累了。gTest提供了一个非常友好的工具，将这些测试的值进行参数化，就不用写那么多重复的代码了。

如何对其进行参数化呢？直接上代码，我们再来看下面一个例子。

```
#include <iostream>
#include <gtest/gtest.h>

class calcFunction
{
public:
    int add(int a, int b)
    {
        std::cout << a << " + " << b << " = " << a + b << std::endl;
        return a + b;
    }

    int sub(int a, int b)
    {
        std::cout << a << " - " << b << " = " << a - b << std::endl;
        return a - b;
    }
};

struct TestParam
{
    int a;
    int b;
    int c;
};

class calcFunctionTest : public ::testing::TestWithParam<struct TestParam>
{
protected:
    calcFunction calc;
    TestParam param;

    virtual void SetUp()
    {
        param.a = GetParam().a;
        param.b = GetParam().b;
        param.c = GetParam().c;
    }

};

TEST_P(calcFunctionTest, test_add)
{
    EXPECT_EQ(calc.add(param.a, param.b), param.c);
}

INSTANTIATE_TEST_CASE_P(addTest, calcFunctionTest, ::testing::Values( TestParam{1, 2 , 3}, 
                                                                      TestParam{1, 3 , 4},
                                                                      TestParam{2, 4 , 6},
                                                                      TestParam{-1, -2 , -3}));

int main(int argc, char **argv)
{  
    testing::InitGoogleTest(&argc, argv);  
    return RUN_ALL_TESTS(); 
} 
```

执行结果和前面的例子一样。

![img](https://pic2.zhimg.com/80/v2-0c563bade7adeb8bc1a21c0eb4b347b5_720w.webp)

从这个例子中，我们不难发现和之前的测试程序有一些不同。这里继承了::testing::TestWithParam类，参数T就是需要参数化的数据类型，这个例子里参数化数据类型是TestParam结构体。这里还需要使用另外一个宏TEST_P而不是TEST_F这个宏，它的两个参数和TEST_F和TEST一致。另外，程序中还增加一个宏INSTANTIATE_TEST_CASE_P用来输入测试参数，它有三个参数（第一个参数大家可任意取名，第二个参数是test_case_name和TEST_P宏的名称一致，第三个参数是需要传递的参数）。

以上就是今天的所有内容，感谢大家耐心的阅读，希望大家都有所收获，愿大家代码无bug。

原文链接：https://zhuanlan.zhihu.com/p/582524287