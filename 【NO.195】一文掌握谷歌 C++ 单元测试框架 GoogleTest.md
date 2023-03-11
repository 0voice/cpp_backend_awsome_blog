# 【NO.195】一文掌握谷歌 C++ 单元测试框架 GoogleTest

## 1.简介

![img](https://pica.zhimg.com/v2-fab7c4bac9ffec62280760d444ae6ac0_720w.jpg?source=d16d100b)

GoogleTest（简称 GTest） 是 Google 开源的一个跨平台的（Liunx、Mac OS X、Windows等）的 C++ 单元测试框架，可以帮助程序员测试 C++ 程序的结果预期。不仅如此，它还提供了丰富的断言、致命和非致命判断、参数化、”死亡测试”等等。

GoogleTest 官网：[GoogleTest User’s Guide](http://link.zhihu.com/?target=https%3A//google.github.io/googletest/)

GitHub 仓库：[https://github.com/google/googletest](http://link.zhihu.com/?target=https%3A//github.com/google/googletest)

## 2.单元测试

单元测试（unit testing），是指对软件中的最小可测试单元进行检查和验证。至于单元的大小或范围，并没有一个明确的标准，单元可以是一个函数、方法、类、功能模块或者子系统。

单元测试通常和白盒测试联系到一起，如果单从概念上来讲两者是有区别的，不过我们通常所说的单元测试和白盒测试都认为是和代码有关系的，所以在某些语境下也通常认为这两者是同一个东西。还有一种理解方式，单元测试和白盒测试就是对开发人员所编写的代码进行测试。

## 3.优势

测试是独立的和可重复的。GoogleTest 使每个测试用例运行在不同的对象上，从而使测试之间相互独立。当测试失败时，GoogleTest 允许单独运行它以进行快速调试。

测试有良好的组织，可以反映被测试代码的结构。 GoogleTest 将相关测试划分到一个测试组内，组内的测试能共享数据，使测试易于维护。

测试是可移植的和可重复使用的。 与平台无关的代码，其测试代码也应该和平台无关，GoogleTest 能在不同的操作系统下工作，并且支持不同的编译器。

当测试用例执行失败时，提供尽可能多的有效信息，以便定位问题。 GoogleTest 不会在第一次测试失败时停止。相反，它只会停止当前测试并继续下一个测试。还可以设置报告非致命故障的测试，然后继续当前测试。因此，您可以在单个运行-编辑-编译周期中检测和修复多个错误。

测试框架应该将测试编写者从琐事中解放出来，让他们专注于测试内容。 GoogleTest 自动跟踪所有定义的测试，不需要用户为了运行它们而进行枚举。

测试高效、快速。GoogleTest 能在测试用例之间复用测试资源，只需支付一次设置/拆分成本，并且不会使测试相互依赖，这样的机制使单元测试更加高效。

## 4.环境搭建

安装 GoogleTest

### 4.1Bazel

首先创建一个工作区目录：

```text
$ mkdir my_workspace && cd my_workspace
```

接着在工作区的根目录中创建一个 WORKSPACE 文件，并在其中引入外部依赖 GoogleTest，此时 Bazel 会自动去 Github 上拉取文件：

```text
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
  name = "com_google_googletest",
  urls = ["https://github.com/google/googletest/archive/609281088cfefc76f9d0ce82e1ff6c30cc3591e5.zip"],
  strip_prefix = "googletest-609281088cfefc76f9d0ce82e1ff6c30cc3591e5",
)
```

接着选取一个目录作为包目录，在该目录下进行代码的编写，例如一个 [http://hello_test.cc](http://link.zhihu.com/?target=http%3A//hello_test.cc) 文件：

```text
#include <gtest/gtest.h>

// Demonstrate some basic assertions.
TEST(HelloTest, BasicAssertions) {
  // Expect two strings not to be equal.
  EXPECT_STRNE("hello", "world");
  // Expect equality.
  EXPECT_EQ(7 * 6, 42);
}
```

在同目录下的 BUILD 文件中说明目标编译的规则：

```text
cc_test(
  name = "hello_test",
  size = "small",
  srcs = ["hello_test.cc"],
  deps = ["@com_google_googletest//:gtest_main"],
)
```

此时执行以下命令即可构建并运行测试程序：

```text
bazel test --test_output=all //:hello_test 
```

### 4.2Cmake

首先创建一个目录：

```text
$ mkdir my_project && cd my_project
```

接着创建 CMakeLists.txt 文件，并声明对 GoogleTest 的依赖，此时 Cmake 会自动去下载对应的库：

```text
cmake_minimum_required(VERSION 3.14)
project(my_project)

# GoogleTest requires at least C++11
set(CMAKE_CXX_STANDARD 11)

include(FetchContent)
FetchContent_Declare(
  googletest
  URL https://github.com/google/googletest/archive/609281088cfefc76f9d0ce82e1ff6c30cc3591e5.zip
)
# For Windows: Prevent overriding the parent project's compiler/linker settings
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)
```

此时我们就可以在代码中使用 GoogleTest，我们创建一个 [http://hello_test.cc](http://link.zhihu.com/?target=http%3A//hello_test.cc) 文件：

```text
#include <gtest/gtest.h>

// Demonstrate some basic assertions.
TEST(HelloTest, BasicAssertions) {
  // Expect two strings not to be equal.
  EXPECT_STRNE("hello", "world");
  // Expect equality.
  EXPECT_EQ(7 * 6, 42);
}
```

然后在 CMakeLists.txt 的末尾加上 [http://hello_test.cc](http://link.zhihu.com/?target=http%3A//hello_test.cc) 的构建规则：

```text
enable_testing()

add_executable(
  hello_test
  hello_test.cc
)
target_link_libraries(
  hello_test
  gtest_main
)

include(GoogleTest)
gtest_discover_tests(hello_test)
```

运行下面的命令构建并允许测试程序：

```text
cmake -S . -B build

cmake --build build

cd build && ctest
```



### 4.3安装示例项目

从 GoogleTest 的 GitHub 中下载官方提供的示例项目：

```text
git clone https://github.com/google/googletest.git
```

示例项目位于`googletest/googletest/samples/`目录，示例内容可参考官网的说明：[samples](http://link.zhihu.com/?target=https%3A//google.github.io/googletest/samples.html)

## 5.GoogleTest 实战

**断言**

**断言的概念**

GoogleTest 中，是通过断言（Assertion）来判断代码实现的功能是否符合预期。断言的结果分为成功（success）、非致命错误（non-fatal failture）和致命错误（fatal failture）。

- 成功：断言成功，程序的行为符合预期，程序允许继续向下运行。可以在代码中调用 SUCCEED() 来表示这个结果。
- 非致命错误：断言失败，但是程序没有直接中止，而是继续向下运行。可以在代码中调用 FAIL() 来表示这个结果。
- 致命错误：中止当前函数或者程序崩溃。可以在代码中调用 ADD_FAILURE() 来表示这个结果。

GoogleTest 的断言是类似于函数调用的宏。通过对其行为进行断言来测试一个类或函数。当断言失败时，GoogleTest 会打印断言的源文件和行号位置以及失败消息。还可以提供自定义失败消息，该消息将附加到 GoogleTest 的消息中。

**EXPECT 与 ASSERT**

宏的格式有两种：

- `EXPECT_*`：在失败时会产生非致命故障，不会中止当前功能。
- `ASSERT_*`：在失败时会产生致命错误，并中止当前函数，同一用例后面的语句将不再执行。

通常 `EXPECT_*` 是首选，因为它们允许在测试中报告多个。但是如果在某条件不成立，程序就无法运行时，就应该使用 `ASSERT_*`。

另一方面，因为 `ASSERT_*` 是直接从当前函数返回的，可能会导致一些内存、文件资源没有释放，因此存在内存泄漏的问题。

GoogleTest 提供了一组断言，用于以各种方式验证代码的行为。包括检查布尔条件、基于关系运算符比较值、验证字符串值、浮点值等等。甚至还有一些断言可以通过提供自定义谓词来验证更复杂的状态。有关 GoogleTest 提供的断言的完整列表，可以参考官方文档：Assertions。

**自定义失败信息**

如果想要提供自定义的失败信息，只需要使用流操作符 << 将这些信息流到断言宏中，例如：

```text
ASSERT_EQ(x.size(), y.size()) << "Vectors x and y are of unequal length";

for (int i = 0; i < x.size(); ++i) {
  EXPECT_EQ(x[i], y[i]) << "Vectors x and y differ at index " << i;
}
```

任何可以流向 std::ostream 的东西都可以流向断言宏，特别是 C 语言的字符串（char*）和 std::string 对象。如果一个宽字符串（wchar_t*，Windows 上 UNICODE 模式下的 TCHAR*，或 std::wstring）被流向一个断言，它将在打印时被转换成 UTF-8。

## 6.功能测试

### 6.1 TEST

如果想要创建测试，可以使用宏函数 `TEST`，它具有以下特点：

- `TEST` 是一个没有返回值的宏函数。
- 我们可以在该函数中使用断言来检测代码是否有效，测试的结果由断言决定。如果测试中的任何断言失败（致命或非致命），或者如果测试崩溃，则整个测试失败。否则，它会成功。

函数定义如下：

```text
TEST(TestSuiteName, TestName) {
  ... test body ...
}
```

TestSuiteName 对应测试用例集名称，TestName 是归属的测试用例名称。这两个名称都必须是有效的 C++ 标识符，并且它们不应包含任何下划线。测试的全名由其包含的测试用例集及其测试名称组成。来自不同测试用例集的测试可以具有相同的名称。

这里以官方提供的 Sample1 中的阶乘函数为例：

http://sample1.cc 中的阶乘函数定义如下：

```text
int Factorial(int n) {
  int result = 1;
  for (int i = 1; i <= n; i++) {
    result *= i;
  }

  return result;
}
```

[http://sample1_unittest.cc](http://link.zhihu.com/?target=http%3A//sample1_unittest.cc) 中即为测试代码，这里为了能够更好的组织测试用例，将数据根据正负数划分为三个测试用例集，每一个用例集中都是相同类型的测试用例。

```text
// 测试负数
TEST(FactorialTest, Negative) {
  EXPECT_EQ(1, Factorial(-5));
  EXPECT_EQ(1, Factorial(-1));
  EXPECT_GT(Factorial(-10), 0);
}

// 测试0
TEST(FactorialTest, Zero) { EXPECT_EQ(1, Factorial(0)); }

// 测试正数
TEST(FactorialTest, Positive) {
  EXPECT_EQ(1, Factorial(1));
  EXPECT_EQ(2, Factorial(2));
  EXPECT_EQ(6, Factorial(3));
  EXPECT_EQ(40320, Factorial(8));
}
```

当我们执行测试时，输出如下：

```text
[==========] Running 6 tests from 2 test cases.
[----------] Global test environment set-up.
[----------] 3 tests from FactorialTest
[ RUN      ] FactorialTest.Negative
[       OK ] FactorialTest.Negative (0 ms)
[ RUN      ] FactorialTest.Zero
[       OK ] FactorialTest.Zero (0 ms)
[ RUN      ] FactorialTest.Positive
[       OK ] FactorialTest.Positive (0 ms)
[----------] 3 tests from FactorialTest (2 ms total)
```

这就表示我们的代码通过了所有的测试用例。

### 6.2 TEST_F

如果发现自己编写了两个或多个对相似数据进行操作的测试，可以使用 test fixture 来为多个测试重用这些相同的配置。

> fixture，其语义是固定的设施，而 test fixture 在 GoogleTest 中的作用就是为每个 TEST 都执行一些同样的操作。

其对应的宏函数是`TEST_F`，函数定义如下：

```text
TEST_F(TestFixtureName, TestName) {
  ... test body ...
}
```

TestFixtureName 对应一个 test fixture 类的名称。因此我们需要自己去定义一个这样的类，并让它继承`testing::Test`类，然后根据我们的需要实现下面这两个虚函数：

- `virtual void SetUp()`：类似于构造函数，在 `TEST_F` 之前运行；
- `virtual void TearDown()`：类似于析构函数，在 `TEST_F` 之后运行。

此外`testing::Test`还提供了两个 static 函数：

- `static void SetUpTestSuite()`：在第一个 `TEST` 之前运行
- `static void TearDownTestSuite()`：在最后一个 `TEST` 之后运行

除了这两种，还有一种全局事件，即继承`testing::Environment`，并实现下面两个虚函数：

- `virtual void SetUp()`：在所有用例之前运行；
- `virtual void TearDown()`：在所有用例之后运行。

这里以官方提供的 Sample3 中实现的 Queue 为例，其实现如下：

```text
#ifndef GOOGLETEST_SAMPLES_SAMPLE3_INL_H_
#define GOOGLETEST_SAMPLES_SAMPLE3_INL_H_

#include <stddef.h>

template <typename E>  // E is the element type
class Queue;

template <typename E>  // E is the element type
class QueueNode {
  friend class Queue<E>;

 public:
  const E& element() const { return element_; }

  QueueNode* next() { return next_; }
  const QueueNode* next() const { return next_; }

 private:
  explicit QueueNode(const E& an_element)
      : element_(an_element), next_(nullptr) {}
    
  const QueueNode& operator=(const QueueNode&);
  QueueNode(const QueueNode&);

  E element_;
  QueueNode* next_;
};

template <typename E>  // E is the element type.
class Queue {
 public:
  Queue() : head_(nullptr), last_(nullptr), size_(0) {}

  ~Queue() { Clear(); }

  void Clear() {
    if (size_ > 0) {
      // 1. Deletes every node.
      QueueNode<E>* node = head_;
      QueueNode<E>* next = node->next();
      for (;;) {
        delete node;
        node = next;
        if (node == nullptr) break;
        next = node->next();
      }

      head_ = last_ = nullptr;
      size_ = 0;
    }
  }

  size_t Size() const { return size_; }

  QueueNode<E>* Head() { return head_; }
  const QueueNode<E>* Head() const { return head_; }

  QueueNode<E>* Last() { return last_; }
  const QueueNode<E>* Last() const { return last_; }

  void Enqueue(const E& element) {
    QueueNode<E>* new_node = new QueueNode<E>(element);

    if (size_ == 0) {
      head_ = last_ = new_node;
      size_ = 1;
    } else {
      last_->next_ = new_node;
      last_ = new_node;
      size_++;
    }
  }

  E* Dequeue() {
    if (size_ == 0) {
      return nullptr;
    }

    const QueueNode<E>* const old_head = head_;
    head_ = head_->next_;
    size_--;
    if (size_ == 0) {
      last_ = nullptr;
    }

    E* element = new E(old_head->element());
    delete old_head;

    return element;
  }

  template <typename F>
  Queue* Map(F function) const {
    Queue* new_queue = new Queue();
    for (const QueueNode<E>* node = head_; node != nullptr;
         node = node->next_) {
      new_queue->Enqueue(function(node->element()));
    }

    return new_queue;
  }

 private:
  QueueNode<E>* head_;  // The first node of the queue.
  QueueNode<E>* last_;  // The last node of the queue.
  size_t size_;         // The number of elements in the queue.

  Queue(const Queue&);
  const Queue& operator=(const Queue&);
};
#endif  // GOOGLETEST_SAMPLES_SAMPLE3_INL_H_
```

接着看看测试用例是如何编写的，首先声明了一个 test fixture 类，在这个类中实现了一些测试时用到的辅助函数，以及使用`SetUp`预置了一些测试数据。（除了有特殊需求，则不需要实现`TearDown`，因为析构函数已经帮我们释放了资源）

```text
class QueueTestSmpl3 : public testing::Test {
 protected: 
    
  void SetUp() override {
    q1_.Enqueue(1);
    q2_.Enqueue(2);
    q2_.Enqueue(3);
  }

  // 一个辅助函数
  static int Double(int n) { return 2 * n; }

  // 测试 Queue::Map() 时用到的辅助函数
  void MapTester(const Queue<int>* q) {

    const Queue<int>* const new_q = q->Map(Double);

    ASSERT_EQ(q->Size(), new_q->Size());

    for (const QueueNode<int>*n1 = q->Head(), *n2 = new_q->Head();
         n1 != nullptr; n1 = n1->next(), n2 = n2->next()) {
      EXPECT_EQ(2 * n1->element(), n2->element());
    }
    delete new_q;
  }

  Queue<int> q0_;
  Queue<int> q1_;
  Queue<int> q2_;
};
```

接着看看它的`TEST_F`：

```text
/ 测试默认构造函数
TEST_F(QueueTestSmpl3, DefaultConstructor) {
  EXPECT_EQ(0u, q0_.Size());
}

// 测试出队
TEST_F(QueueTestSmpl3, Dequeue) {
  int* n = q0_.Dequeue();
  EXPECT_TRUE(n == nullptr);

  n = q1_.Dequeue();
  ASSERT_TRUE(n != nullptr);
  EXPECT_EQ(1, *n);
  EXPECT_EQ(0u, q1_.Size());
  delete n;

  n = q2_.Dequeue();
  ASSERT_TRUE(n != nullptr);
  EXPECT_EQ(2, *n);
  EXPECT_EQ(1u, q2_.Size());
  delete n;
}

// 测试Map函数
TEST_F(QueueTestSmpl3, Map) {
  MapTester(&q0_);
  MapTester(&q1_);
  MapTester(&q2_);
}
}  // namespace
```

这里以 DefaultConstructor 为例，来分析一下它的执行流程：

1. QueueTestSmpl3 调用构造函数，构造对象。
2. QueueTestSmpl3 对象调用 SetUp 函数初始化测试配置。
3. DefaultConstructor 开始执行并结束测试。
4. QueueTestSmpl3 对象调用隐式生成的 TearDown 进行清理。
5. QueueTestSmpl3 调用析构函数，析构对象。

### 6.3运行测试

#### 6.3.1 调用测试

与其他 C++ 框架不同，TEST 和 TEST_F 会隐式向 GoogleTest 注册这些测试函数，因此我们不需要为了运行这些它们而进行枚举。

定义完测试后，你可以用 RUN_ALL_TESTS 来运行它们，此时会运行所有的测试，如果全部成功则返回 0，反之则返回 1。其执行流程如下：

1. 保存所有 GoogleTest 标志的状态。
2. 为第一个测试创建一个 test fixture 对象。
3. 通过 SetUp 初始化 test fixture 对象。
4. 在 test fixture 对象上进行测试。
5. 通过 TearDown 清理 test fixture。
6. 删除 test fixture。
7. 恢复所有 GoogleTest 标志的状态。
8. 对下一个测试重复上述步骤，直到所有测试都运行完毕。

如果发生致命故障，则将跳过后续步骤。

#### 6.3.2编写 main 函数

用户不需要编写自己的 `main` 函数，编译器会自动将它链接到 `gtest_main`。如果用户有特殊需求，需要编写一个 `main` 函数，则需要在返回时调用 `RUN_ALL_TESTS()` 来运行所有测试，例如：

```text
int main(int argc, char **argv) {
  printf("Running main() from %s\n", __FILE__);
  testing::InitGoogleTest(&argc, argv);
  return RUN_ALL_TESTS(); 
}
```

`testing::InitGoogleTest`函数会解析 GoogleTest 标志的命令行，并删除所有识别的标志。这允许用户通过各种标志控制测试程序的行为。您必须在调用`RUN_ALL_TESTS`之前调用此函数 ，否则标志将无法正确初始化。

原文地址：https://zhuanlan.zhihu.com/p/544491071

作者：linux