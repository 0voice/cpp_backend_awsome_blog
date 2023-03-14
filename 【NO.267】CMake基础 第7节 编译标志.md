# 【NO.267】CMake基础 第7节 编译标志

## 1.引言

CMake支持以多种不同方式设置编译标志：

- 使用target_compile_definitions（）函数
- 使用CMAKE_C_FLAGS和CMAKE_CXX_FLAGS变量。

本教程中的文件如下：

```shell
$ tree
.
├── CMakeLists.txt
├── main.cpp
```

- [CMakeLists.txt] - 包含要运行的CMake命令

  ```cmake
  cmake_minimum_required(VERSION 3.5)
  
  # Set a default C++ compile flag
  set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DEX2" CACHE STRING "Set C++ Compiler Flags" FORCE)
  
  # Set the project name
  project (compile_flags)
  
  # Add an executable
  add_executable(cmake_examples_compile_flags main.cpp)
  
  target_compile_definitions(cmake_examples_compile_flags 
      PRIVATE EX3
  )
  ```

- [main.cpp] - 具有main的源文件

  ```cpp
  #include <iostream>
  
  int main(int argc, char *argv[])
  {
     std::cout << "Hello Compile Flags!" << std::endl;
  
     // only print if compile flag set
  #ifdef EX2
    std::cout << "Hello Compile Flag EX2!" << std::endl;
  #endif
  
  #ifdef EX3
    std::cout << "Hello Compile Flag EX3!" << std::endl;
  #endif
  
     return 0;
  }
  ```

## 2.概念

### 2.1 设置每个目标的C++标志

在现代CMake中设置C++标志的推荐方式是使用每个目标的标志，这些标志可以通过`target_compile_definitions()`函数的作用域（或者说接口范围）递到其他目标（INTERFACE或PUBLIC）。这将填充库的`INTERFACE_COMPILE_DEFINITIONS`，并根据作用域将定义传递到链接的目标。

```cmake
target_compile_definitions(cmake_examples_compile_flags
    PRIVATE EX3
)
```

这将导致编译器在编译目标时添加定义 -DEX3。

如果目标是库，并且已经选择了作用域PUBLIC或者INTERFACE，则该定义也将包含在链接该目标的任何可执行文件中。

对于编译器选项，你还可以使用`target_compile_options()`函数。

### 2.2 设置默认C++标志

`CMAKE_CXX_FLAGS`的默认值为空或包含生成类型的相应标志。

要设置其他默认编译标志，可以将以下内容添加到顶级CMakeLists.txt。

```cmake
set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DEX2" CACHE STRING "Set C++ Compiler Flags" FORCE)
```

与CMAKE_CXX_FLAGS类似的其他选项包括：

- 使用CMAKE_C_FLAGS设置 C 编译器标志
- 使用CMAKE_LINKER_FLAGS设置链接器标志

| Note | 上述命令中的值 `CACHE STRING "Set C++ Compiler Flags" FORCE` 用于强制在CMakeCache.txt文件中设置此变量。有关更多详细信息，请参阅[此处](https://cmake.org/cmake/help/v3.0/command/set.html)。 |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

一旦设置，CMAKE_C_FLAGS和CMAKE_CXX_FLAGS将为该目录或任何包含的子目录中的所有目标全局设置编译器标志/定义。现在不建议将此方法用于一般用途，最好使用`target_compile_definitions`函数。

### 2.3 设置CMake标志

与构建类型类似，可以使用以下方法设置全局C++编译器标志。

- 使用GUI工具，如ccmake/cmake-gui
- 传递到cmake

```shell
cmake .. -DCMAKE_CXX_FLAGS="-DEX3"
```

## 3.构建示例

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
-- Configuring done
-- Generating done
-- Build files have been written to: /home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/build

$ make VERBOSE=1
/usr/bin/cmake -H/home/matrim/workspace/cmake-examples/01-basic/G-compile-flags -B/home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/build --check-build-system CMakeFiles/Makefile.cmake 0
/usr/bin/cmake -E cmake_progress_start /home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/build/CMakeFiles /home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/build/CMakeFiles/progress.marks
make -f CMakeFiles/Makefile2 all
make[1]: Entering directory `/home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/build'
make -f CMakeFiles/cmake_examples_compile_flags.dir/build.make CMakeFiles/cmake_examples_compile_flags.dir/depend
make[2]: Entering directory `/home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/build'
cd /home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/build && /usr/bin/cmake -E cmake_depends "Unix Makefiles" /home/matrim/workspace/cmake-examples/01-basic/G-compile-flags /home/matrim/workspace/cmake-examples/01-basic/G-compile-flags /home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/build /home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/build /home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/build/CMakeFiles/cmake_examples_compile_flags.dir/DependInfo.cmake --color=
Dependee "/home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/build/CMakeFiles/cmake_examples_compile_flags.dir/DependInfo.cmake" is newer than depender "/home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/build/CMakeFiles/cmake_examples_compile_flags.dir/depend.internal".
Dependee "/home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/build/CMakeFiles/CMakeDirectoryInformation.cmake" is newer than depender "/home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/build/CMakeFiles/cmake_examples_compile_flags.dir/depend.internal".
Scanning dependencies of target cmake_examples_compile_flags
make[2]: Leaving directory `/home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/build'
make -f CMakeFiles/cmake_examples_compile_flags.dir/build.make CMakeFiles/cmake_examples_compile_flags.dir/build
make[2]: Entering directory `/home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/build'
/usr/bin/cmake -E cmake_progress_report /home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/build/CMakeFiles 1
[100%] Building CXX object CMakeFiles/cmake_examples_compile_flags.dir/main.cpp.o
/usr/bin/c++    -DEX2   -o CMakeFiles/cmake_examples_compile_flags.dir/main.cpp.o -c /home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/main.cpp
Linking CXX executable cmake_examples_compile_flags
/usr/bin/cmake -E cmake_link_script CMakeFiles/cmake_examples_compile_flags.dir/link.txt --verbose=1
/usr/bin/c++    -DEX2    CMakeFiles/cmake_examples_compile_flags.dir/main.cpp.o  -o cmake_examples_compile_flags -rdynamic
make[2]: Leaving directory `/home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/build'
/usr/bin/cmake -E cmake_progress_report /home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/build/CMakeFiles  1
[100%] Built target cmake_examples_compile_flags
make[1]: Leaving directory `/home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/build'
/usr/bin/cmake -E cmake_progress_start /home/matrim/workspace/cmake-examples/01-basic/G-compile-flags/build/CMakeFiles 0
```

原文作者：[橘崽崽啊](https://www.cnblogs.com/juzaizai/)

原文链接：https://www.cnblogs.com/juzaizai/p/15069664.html