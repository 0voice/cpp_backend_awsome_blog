# 【NO.263】CMake基础 第3节 静态库

## 1.介绍

继续展示一个hello world示例，它首先创建并链接一个静态库。这是一个简化示例，这里的库和二进制文件在同一个文件夹中。通常，这些将会被放到子项目中，这些内容将会在以后描述。

本教程中的文件如下：

```objectivec
$ tree
.
├── CMakeLists.txt
├── include
│   └── static
│       └── Hello.h
└── src
    ├── Hello.cpp
    └── main.cpp
```

- [CMakeLists.txt] - 包含你希望运行的 CMake 命令

  ```cmake
  cmake_minimum_required(VERSION 3.5)
  
  project(hello_library)
  
  ############################################################
  # Create a library
  ############################################################
  
  #Generate the static library from the library sources
  add_library(hello_library STATIC 
      src/Hello.cpp
  )
  
  target_include_directories(hello_library
      PUBLIC 
          ${PROJECT_SOURCE_DIR}/include
  )
  
  
  ############################################################
  # Create an executable
  ############################################################
  
  # Add an executable with the above sources
  add_executable(hello_binary 
      src/main.cpp
  )
  
  # link the new hello_library target with the hello_binary target
  target_link_libraries( hello_binary
      PRIVATE 
          hello_library
  )
  ```

- [include/static/Hello.h] - 要包含的头文件

  ```c++
  #ifndef __HELLO_H__
  #define __HELLO_H__
  
  class Hello
  {
  public:
      void print();
  };
  
  #endif
  ```

- [src/Hello.cpp] - 要编译的源文件

  ```c++
  #include <iostream>
  
  #include "static/Hello.h"
  
  void Hello::print()
  {
      std::cout << "Hello Static Library!" << std::endl;
  }
  ```

- [src/main.cpp] - 主源文件

  ```c++
  #include "static/Hello.h"
  
  int main(int argc, char *argv[])
  {
      Hello hi;
      hi.print();
      return 0;
  }
  ```

## 2.概念

### 2.1 添加静态库

`add_library()`功能用于从某些源文件创建库。调用方式如下：

```cmake
add_library(hello_library STATIC
    src/Hello.cpp
)
```

此命令将使用add_library()调用中的源代码创建一个名为`libhello_library.a`的静态库

| 注意 | 如上一节所述，我们按照现代 CMake 的建议，将源文件直接传递给add_library调用。 |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

### 2.2 添加头文件目录

在本例中，我们使用`target_include_directory()`函数将include目录包含在库中，并将范围设置为PUBLIC。

```cmake
target_include_directories(hello_library
    PUBLIC
        ${PROJECT_SOURCE_DIR}/include
)
```

这将导致包含的目录在以下位置使用：

- 在编译该库时
- 在编译链接至该库的任何其他目标时。

作用域的含义是：

- PRIVATE - 将目录添加到此目标的include目录中
- INTERFACE - 将该目录添加到任何链接到此库的目标的include目录中（不包括自己）。
- PUBLIC - 它包含在此库中，也包含在链接此库的任何目标中。

| 提示 | 对于公共头文件，让你的include文件夹使用子目录“命名空间”通常是个好主意。传递给target_include_directories的目录将是你的Include目录树的根目录，并且你的C++文件应该包含从那里到你要使用的头文件的路径。在本例中，你可以看到我们按如下方式执行操作。使用此方法意味着，在项目中使用多个库时，头文件名冲突的可能性较小。比如： |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

```c++
#include "static/Hello.h"
```

### 2.3 链接库

在创建使用库的可执行文件时，你必须告诉编译器有关库的信息。这可以使用target_link_library()函数来完成。

```cmake
add_executable(hello_binary
    src/main.cpp
)

target_link_libraries( hello_binary
    PRIVATE
        hello_library
)
```

这告诉CMake在链接时将hello_library与hello_binary可执行文件链接起来。它还将从hello_library传递任何具有PUBLIC或INTERFACE作用范围的include目录到hello_binary。

编译器调用它的一个示例是:

```shell
/usr/bin/c++ CMakeFiles/hello_binary.dir/src/main.cpp.o -o hello_binary -rdynamic libhello_library.a
```

## 3.构建示例

```shell
$ mkdir build

$ cd build

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
-- Configuring done
-- Generating done
-- Build files have been written to: /home/matrim/workspace/cmake-examples/01-basic/C-static-library/build

$ make
Scanning dependencies of target hello_library
[ 50%] Building CXX object CMakeFiles/hello_library.dir/src/Hello.cpp.o
Linking CXX static library libhello_library.a
[ 50%] Built target hello_library
Scanning dependencies of target hello_binary
[100%] Building CXX object CMakeFiles/hello_binary.dir/src/main.cpp.o
Linking CXX executable hello_binary
[100%] Built target hello_binary

$ ls
CMakeCache.txt  CMakeFiles  cmake_install.cmake  hello_binary  libhello_library.a  Makefile

$ ./hello_binary
Hello Static Library!
```



原文作者：[橘崽崽啊](https://www.cnblogs.com/juzaizai/)

原文链接：https://www.cnblogs.com/juzaizai/p/15069323.html