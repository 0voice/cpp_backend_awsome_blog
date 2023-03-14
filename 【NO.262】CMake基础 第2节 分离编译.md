# 【NO.262】CMake基础 第2节 分离编译

## 1.介绍

展示一个hello world示例，它使用不同的文件夹来存储源文件和头文件。

本教程中的文件包括：

```ruby
B-hello-headers$ tree
.
├── CMakeLists.txt
├── include
│   └── Hello.h
└── src
    ├── Hello.cpp
    └── main.cpp
```

- [CMakeLists.txt] - 包含你希望运行的 CMake 命令。

  ```cmake
  # Set the minimum version of CMake that can be used
  # To find the cmake version run
  # $ cmake --version
  cmake_minimum_required(VERSION 3.5)
  
  # Set the project name
  project (hello_headers)
  
  # Create a sources variable with a link to all cpp files to compile
  set(SOURCES
      src/Hello.cpp
      src/main.cpp
  )
  
  # Add an executable with the above sources
  add_executable(hello_headers ${SOURCES})
  
  # Set the directories that should be included in the build command for this target
  # when running g++ these will be included as -I/directory/path/
  target_include_directories(hello_headers
      PRIVATE 
          ${PROJECT_SOURCE_DIR}/include
  )
  ```

- [include/Hello.h] - 要包含的标题文件。

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

- [src/Hello.cpp] - 要编译的源文件。

  ```cpp
  #include <iostream>
  
  #include "Hello.h"
  
  void Hello::print()
  {
      std::cout << "Hello Headers!" << std::endl;
  }
  ```

- [src/main.cpp] - 主源文件。

  ```cpp
  #include "Hello.h"
  
  int main(int argc, char *argv[])
  {
      Hello hi;
      hi.print();
      return 0;
  }
  ```

## 2.概念

### 2.1 目录路径

CMake 语法指定了许多变量，这些变量可用于帮助在项目或源代码树中找到有用的目录。其中一些包括：

| 变量                     | 信息                                            |
| ------------------------ | ----------------------------------------------- |
| CMAKE_SOURCE_DIR         | 根源目录                                        |
| CMAKE_CURRENT_SOURCE_DIR | 如果使用子项目和目录，则为当前源目录。          |
| PROJECT_SOURCE_DIR       | 当前 cmake 项目的源目录。                       |
| CMAKE_BINARY_DIR         | 根二进制文件生成目录。这是运行cmake命令的目录。 |
| CMAKE_CURRENT_BINARY_DIR | 你当前所处的生成目录。                          |
| PROJECT_BINARY_DIR       | 当前项目的生成目录。                            |

### 2.2 源文件变量

创建包含源码文件的变量可使你更清楚地了解这些文件，并轻松将其添加到多个命令中，例如，add_executable()功能。

```cmake
# Create a sources variable with a link to all cpp files to compile
set(SOURCES
    src/Hello.cpp
    src/main.cpp
)

add_executable(${PROJECT_NAME} ${SOURCES})
```

| Note | 另一种替代方案是使用 GLOB 命令使用通配符模式匹配查找文件。 |
| ---- | ---------------------------------------------------------- |
|      |                                                            |

```cmake
file(GLOB SOURCES "src/*.cpp")
```

| Tip  | 对于现代的CMake，不建议对源代码使用变量。相反，通常直接在add_xxx函数中声明源代码。这对于GLOB命令尤其重要，如果你添加一个新的源代码文件，这些命令可能并不总是显示正确的结果。 |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

### 2.3 头文件目录

当你有不同的头文件目录时，可以使用`target_include_directories()`函数让编译器知道它们编译此目标时，会将这些目录添加到带有`-i`标志的编译指令中，例如`-i/directory/path`

```cmake
target_include_directories(target
    PRIVATE
        ${PROJECT_SOURCE_DIR}/include
)
```

PRIVATE标识符指定include的范围，这对库很重要，这些内容将在下一个示例中进行说明。有关该功能的更多详细信息，请访问[这里](https://cmake.org/cmake/help/v3.0/command/target_include_directories.html)

## 3.构建示例

### 3.1 标准输出

构建示例的标准输出内容如下所示：

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
-- Build files have been written to: /home/matrim/workspace/cmake-examples/01-basic/hello_headers/build

$ make
Scanning dependencies of target hello_headers
[ 50%] Building CXX object CMakeFiles/hello_headers.dir/src/Hello.cpp.o
[100%] Building CXX object CMakeFiles/hello_headers.dir/src/main.cpp.o
Linking CXX executable hello_headers
[100%] Built target hello_headers

$ ./hello_headers
Hello Headers!
```

### 3.2 详细输出

在前面的示例中，当运行make命令时，输出仅显示构建的状态。要查看用于调试目的的完整输出，可以在运行make时添加VERBOSE=1标志。

VERBOSE 输出如下所示。可以看到，include目录已经被添加到了编译命令中。

```shell
$ make clean

$ make VERBOSE=1
/usr/bin/cmake -H/home/matrim/workspace/cmake-examples/01-basic/hello_headers -B/home/matrim/workspace/cmake-examples/01-basic/hello_headers/build --check-build-system CMakeFiles/Makefile.cmake 0
/usr/bin/cmake -E cmake_progress_start /home/matrim/workspace/cmake-examples/01-basic/hello_headers/build/CMakeFiles /home/matrim/workspace/cmake-examples/01-basic/hello_headers/build/CMakeFiles/progress.marks
make -f CMakeFiles/Makefile2 all
make[1]: Entering directory `/home/matrim/workspace/cmake-examples/01-basic/hello_headers/build'
make -f CMakeFiles/hello_headers.dir/build.make CMakeFiles/hello_headers.dir/depend
make[2]: Entering directory `/home/matrim/workspace/cmake-examples/01-basic/hello_headers/build'
cd /home/matrim/workspace/cmake-examples/01-basic/hello_headers/build && /usr/bin/cmake -E cmake_depends "Unix Makefiles" /home/matrim/workspace/cmake-examples/01-basic/hello_headers /home/matrim/workspace/cmake-examples/01-basic/hello_headers /home/matrim/workspace/cmake-examples/01-basic/hello_headers/build /home/matrim/workspace/cmake-examples/01-basic/hello_headers/build /home/matrim/workspace/cmake-examples/01-basic/hello_headers/build/CMakeFiles/hello_headers.dir/DependInfo.cmake --color=
make[2]: Leaving directory `/home/matrim/workspace/cmake-examples/01-basic/hello_headers/build'
make -f CMakeFiles/hello_headers.dir/build.make CMakeFiles/hello_headers.dir/build
make[2]: Entering directory `/home/matrim/workspace/cmake-examples/01-basic/hello_headers/build'
/usr/bin/cmake -E cmake_progress_report /home/matrim/workspace/cmake-examples/01-basic/hello_headers/build/CMakeFiles 1
[ 50%] Building CXX object CMakeFiles/hello_headers.dir/src/Hello.cpp.o
/usr/bin/c++    -I/home/matrim/workspace/cmake-examples/01-basic/hello_headers/include    -o CMakeFiles/hello_headers.dir/src/Hello.cpp.o -c /home/matrim/workspace/cmake-examples/01-basic/hello_headers/src/Hello.cpp
/usr/bin/cmake -E cmake_progress_report /home/matrim/workspace/cmake-examples/01-basic/hello_headers/build/CMakeFiles 2
[100%] Building CXX object CMakeFiles/hello_headers.dir/src/main.cpp.o
/usr/bin/c++    -I/home/matrim/workspace/cmake-examples/01-basic/hello_headers/include    -o CMakeFiles/hello_headers.dir/src/main.cpp.o -c /home/matrim/workspace/cmake-examples/01-basic/hello_headers/src/main.cpp
Linking CXX executable hello_headers
/usr/bin/cmake -E cmake_link_script CMakeFiles/hello_headers.dir/link.txt --verbose=1
/usr/bin/c++       CMakeFiles/hello_headers.dir/src/Hello.cpp.o CMakeFiles/hello_headers.dir/src/main.cpp.o  -o hello_headers -rdynamic
make[2]: Leaving directory `/home/matrim/workspace/cmake-examples/01-basic/hello_headers/build'
/usr/bin/cmake -E cmake_progress_report /home/matrim/workspace/cmake-examples/01-basic/hello_headers/build/CMakeFiles  1 2
[100%] Built target hello_headers
make[1]: Leaving directory `/home/matrim/workspace/cmake-examples/01-basic/hello_headers/build'
/usr/bin/cmake -E cmake_progress_start /home/matrim/workspace/cmake-examples/01-basic/hello_headers/build/CMakeFiles 0
```



原文作者：[橘崽崽啊](https://www.cnblogs.com/juzaizai/)

原文链接：https://www.cnblogs.com/juzaizai/p/15069309.html