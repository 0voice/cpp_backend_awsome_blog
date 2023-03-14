# 【NO.264】CMake基础 第4节 动态库

## 1.介绍

继续展示一个hello world示例，它将首先创建并链接一个共享库。

这里还显示了如何创建[别名目标](https://cmake.org/cmake/help/v3.0/manual/cmake-buildsystem.7.html#alias-targets)

本教程中的文件如下：

```shell
$ tree
.
├── CMakeLists.txt
├── include
│   └── shared
│       └── Hello.h
└── src
    ├── Hello.cpp
    └── main.cpp
```

- [CMakeLists.txt] - 包含要运行的 CMake 命令

  ```cmake
  cmake_minimum_required(VERSION 3.5)
  
  project(hello_library)
  
  ############################################################
  # Create a library
  ############################################################
  
  #Generate the shared library from the library sources
  add_library(hello_library SHARED 
      src/Hello.cpp
  )
  add_library(hello::library ALIAS hello_library)
  
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
          hello::library
  )
  ```

- [include/shared/Hello.h] - 要包含的头文件

  ```cpp
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

  ```cpp
  #include <iostream>
  
  #include "shared/Hello.h"
  
  void Hello::print()
  {
      std::cout << "Hello Shared Library!" << std::endl;
  }
  ```

- [src/main.cpp] - 具有main的源文件

  ```cpp
  #include "shared/Hello.h"
  
  int main(int argc, char *argv[])
  {
      Hello hi;
      hi.print();
      return 0;
  }
  ```

## 2.概念

### 2.1 添加共享库

与前面的静态库示例一样，add_library()函数也用于从一些源文件创建共享库。它的用法如下：

```cmake
add_library(hello_library SHARED
    src/Hello.cpp
)
```

这将使用传递给add_library()函数的源码文件创建一个名为libhello_library.so的共享库。

### 2.2 别名目标

顾名思义，别名目标是目标的替代名称，可以在只读上下文中替代真实的目标名称。

```cmake
add_library(hello::library ALIAS hello_library)
```

ALIAS类似于“同义词”。ALIAS目标只是原始目标的另一个名称。因此ALIAS目标的要求是不可修改的——您无法调整其属性、安装它等。

如下所示，这允许你在将目标链接到其他目标时使用别名引用该目标。

### 2.3 链接共享库

链接共享库与链接静态库相同。创建可执行文件时，请使用`target_link_library()`函数指向库。

```cmake
add_executable(hello_binary
    src/main.cpp
)

target_link_libraries(hello_binary
    PRIVATE
        hello::library
)
```

这告诉CMake使用别名目标名称将hello_library链接到hello_binary可执行文件。

链接器调用它的一个示例是：

```shell
/usr/bin/c++ CMakeFiles/hello_binary.dir/src/main.cpp.o -o hello_binary -rdynamic libhello_library.so -Wl,-rpath,/home/matrim/workspace/cmake-examples/01-basic/D-shared-library/build
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
-- Build files have been written to: /home/matrim/workspace/cmake-examples/01-basic/D-shared-library/build

$ make
Scanning dependencies of target hello_library
[ 50%] Building CXX object CMakeFiles/hello_library.dir/src/Hello.cpp.o
Linking CXX shared library libhello_library.so
[ 50%] Built target hello_library
Scanning dependencies of target hello_binary
[100%] Building CXX object CMakeFiles/hello_binary.dir/src/main.cpp.o
Linking CXX executable hello_binary
[100%] Built target hello_binary

$ ls
CMakeCache.txt  CMakeFiles  cmake_install.cmake  hello_binary  libhello_library.so  Makefile

$ ./hello_binary
Hello Shared Library!
```

原文作者：[橘崽崽啊](https://www.cnblogs.com/juzaizai/)

原文链接：https://www.cnblogs.com/juzaizai/p/15069328.html