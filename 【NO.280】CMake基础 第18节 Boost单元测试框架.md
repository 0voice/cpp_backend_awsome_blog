# 【NO.280】CMake基础 第18节 Boost单元测试框架

## 1.介绍

使用CTest，你可以生成`make test`目标来运行自动化单元测试。这个例子展示了如何找到Boost单元测试框架，创建测试并运行它们。

本教程中的文件如下：

```shell
$ tree
.
├── CMakeLists.txt
├── Reverse.h
├── Reverse.cpp
├── Palindrome.h
├── Palindrome.cpp
├── unit_tests.cpp
```

- [CMakeLists.txt] - 包含要运行的CMake命令。

  ```cmake
  cmake_minimum_required(VERSION 3.5)
  
  # Set the project name
  project (boost_unit_test)
  
  
  # find a boost install with the libraries unit_test_framework
  find_package(Boost 1.46.1 REQUIRED COMPONENTS unit_test_framework)
  
  # Add an library for the example classes
  add_library(example_boost_unit_test
      Reverse.cpp
      Palindrome.cpp
  )
  
  target_include_directories(example_boost_unit_test
      PUBLIC
          ${CMAKE_CURRENT_SOURCE_DIR}
  )
  
  target_link_libraries(example_boost_unit_test
      PUBLIC
          Boost::boost
  )
  
  #############################################
  # Unit tests
  
  # enable CTest testing
  enable_testing()
  
  # Add a testing executable
  add_executable(unit_tests unit_tests.cpp)
  
  target_link_libraries(unit_tests
      example_boost_unit_test
      Boost::unit_test_framework
  )
  
  target_compile_definitions(unit_tests
      PRIVATE
          BOOST_TEST_DYN_LINK
  )
  
  add_test(test_all unit_tests)
  ```

- [Reverse.h] / [.cpp] - 反转字符串的类。

  ```cpp
  #ifndef __REVERSE_H__
  #define __REVERSE_H__
  
  #include <string>
  
  /**
   * Trivial class whose only function is to reverse a string.
   * Should use std::reverse instead but want to have example with own class
   */
  class Reverse
  {
  public:
      std::string reverse(std::string& toReverse);
  };
  #endif
  ```

  ```cpp
  #include "Reverse.h"
  
  std::string Reverse::reverse(std::string& toReverse)
  {
      std::string ret;
  
      for(std::string::reverse_iterator rit=toReverse.rbegin(); rit!=toReverse.rend(); ++rit)
      {
          ret.insert(ret.end(), *rit);
      }
      return ret;
  }
  ```

- [Palindrome.h] / [.cpp] - 测试字符串是否为回文的类。

  ```cpp
  #ifndef __PALINDROME_H__
  #define __PALINDROME_H__
  
  #include <string>
  
  /**
   * Trivial class to check if a string is a palindrome.
   */
  class Palindrome
  {
  public:
      bool isPalindrome(const std::string& toCheck);
  };
  
  #endif
  ```

  ```cpp
  #include "Palindrome.h"
  
  bool Palindrome::isPalindrome(const std::string& toCheck)
  {
  
      if (toCheck == std::string(toCheck.rbegin(), toCheck.rend())) {
          return true;
      }
  
      return false;
  }
  ```

- [unit_tests.cpp] - 使用Boost单元测试框架的单元测试。

  ```cpp
  #include <string>
  #include "Reverse.h"
  #include "Palindrome.h"
  
  #define BOOST_TEST_MODULE VsidCommonTest
  #include <boost/test/unit_test.hpp>
  
  BOOST_AUTO_TEST_SUITE( reverse_tests )
  
  BOOST_AUTO_TEST_CASE( simple )
  {
      std::string toRev = "Hello";
  
      Reverse rev;
      std::string res = rev.reverse(toRev);
  
      BOOST_CHECK_EQUAL( res, "olleH" );
  
  }
  
  
  BOOST_AUTO_TEST_CASE( empty )
  {
      std::string toRev;
  
      Reverse rev;
      std::string res = rev.reverse(toRev);
  
      BOOST_CHECK_EQUAL( res, "" );
  }
  
  BOOST_AUTO_TEST_SUITE_END()
  
  
  BOOST_AUTO_TEST_SUITE( palindrome_tests )
  
  BOOST_AUTO_TEST_CASE( is_palindrome )
  {
      std::string pal = "mom";
      Palindrome pally;
  
      BOOST_CHECK_EQUAL( pally.isPalindrome(pal), true );
  
  }
  
  BOOST_AUTO_TEST_SUITE_END()
  ```

## 2.要求

此示例要求将Boost库安装在默认系统位置。

使用的库是Boost单元测试框架。

## 3.概念

### 3.1 启用测试

要启用测试，你必须在顶级CMakeLists.txt中包含以下行。

```cmake
enable_testing()
```

这将启用对当前文件夹及其下面所有文件夹的测试。

### 3.2 添加测试可执行文件

此步骤的要求将取决于你的单元测试框架。在Boost的示例中，你可以创建包含要运行的所有单元测试的二进制文件。

```cmake
add_executable(unit_tests unit_tests.cpp)

target_link_libraries(unit_tests
    example_boost_unit_test
    Boost::unit_test_framework
)

target_compile_definitions(unit_tests
    PRIVATE
        BOOST_TEST_DYN_LINK
)
```

在上面的代码中，添加了一个单元测试二进制文件，它链接到Boost单元测试框架，并包含一个定义来告诉它我们正在使用动态链接。

### 3.3 添加测试

要添加测试，可以调用`add_test()`函数。这将创建一个命名测试，该测试将运行提供的命令。

```cmake
add_test(test_all unit_tests)
```

在本例中，我们创建了一个名为test_all的测试，该测试将运行由调用`add_executable`创建的unit_test可执行文件创建的可执行文件。

## 4.构建示例

```shell
$ mkdir build

$ cd build/

$ cmake ..
-- The C compiler identification is GNU 4.8.4
-- The CXX compiler identification is GNU 4.8.4
-- Check for working C compiler: /usr/bin/cc
-- Check for working C compiler: /usr/bin/cc -- works
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working CXX compiler: /usr/bin/c++
-- Check for working CXX compiler: /usr/bin/c++ -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Boost version: 1.54.0
-- Found the following Boost libraries:
--   unit_test_framework
-- Configuring done
-- Generating done
-- Build files have been written to: /home/matrim/workspace/cmake-examples/05-unit-testing/boost/build

$ make
Scanning dependencies of target example_boost_unit_test
[ 33%] Building CXX object CMakeFiles/example_boost_unit_test.dir/Reverse.cpp.o
[ 66%] Building CXX object CMakeFiles/example_boost_unit_test.dir/Palindrome.cpp.o
Linking CXX static library libexample_boost_unit_test.a
[ 66%] Built target example_boost_unit_test
Scanning dependencies of target unit_tests
[100%] Building CXX object CMakeFiles/unit_tests.dir/unit_tests.cpp.o
Linking CXX executable unit_tests
[100%] Built target unit_tests

$ make test
Running tests...
Test project /home/matrim/workspace/cmake-examples/05-unit-testing/boost/build
    Start 1: test_all
1/1 Test #1: test_all .........................   Passed    0.00 sec

100% tests passed, 0 tests failed out of 1

Total Test time (real) =   0.01 sec
```

如果更改代码并导致单元测试产生错误。然后，在运行测试时，你将看到以下输出。

```shell
$ make test
Running tests...
Test project /home/matrim/workspace/cmake-examples/05-unit-testing/boost/build
    Start 1: test_all
1/1 Test #1: test_all .........................***Failed    0.00 sec

0% tests passed, 1 tests failed out of 1

Total Test time (real) =   0.00 sec

The following tests FAILED:
          1 - test_all (Failed)
Errors while running CTest
make: *** [test] Error 8
```

原文作者：[橘崽崽啊](https://www.cnblogs.com/juzaizai/)

原文链接：https://www.cnblogs.com/juzaizai/p/15072101.html