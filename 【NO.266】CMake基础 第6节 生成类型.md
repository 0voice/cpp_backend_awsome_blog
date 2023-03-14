# 【NO.266】CMake基础 第6节 生成类型

## 1.介绍

CMake有许多内置的构建配置，可用于编译你的项目。它们指定优化级别以及调试信息是否包含在二进制文件中。

提供的级别包括：

- Release - 将标志`-O3 -DNDEBUG`添加到编译器
- Debug - 添加标志`-g`
- MinSizeRel - 添加标志`-Os -DNDEBUG`
- RelWithDebInfo - 添加标志`-O2 -g -DNDEBUG`

本教程中的文件如下：

```objectivec
$ tree
.
├── CMakeLists.txt
├── main.cpp
```

- [CMakeLists.txt] - 包含你希望运行的 CMake 命令

  ```cmake
  # Set the minimum version of CMake that can be used
  # To find the cmake version run
  # $ cmake --version
  cmake_minimum_required(VERSION 3.5)
  
  # Set a default build type if none was specified
  if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    message("Setting build type to 'RelWithDebInfo' as none was specified.")
    set(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING "Choose the type of build." FORCE)
    # Set the possible values of build type for cmake-gui
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release"
      "MinSizeRel" "RelWithDebInfo")
  endif()
  
  # Set the project name
  project (build_type)
  
  # Add an executable
  add_executable(cmake_examples_build_type main.cpp)
  ```

- [main.cpp] - 主源文件

  ```cpp
  #include <iostream>
  
  int main(int argc, char *argv[])
  {
     std::cout << "Hello Build Type!" << std::endl;
     return 0;
  }
  ```

## 2.概念

### 2.1 设置生成类型

可以使用以下方法设置生成类型。

- 使用GUI工具，如ccmake/cmake-gui
- 通过命令行传递到cmake：

```shell
cmake .. -DCMAKE_BUILD_TYPE=Release
```

### 2.2 设置默认生成类型

CMake提供的默认构建类型是不包含用于优化的编译器标志。对于某些项目，你可能希望设置默认生成类型，以便不必记住设置它。为此，你可以将以下代码添加到顶级CMakeLists.txt中。

```cmake
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message("Setting build type to 'RelWithDebInfo' as none was specified.")
  set(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release"
    "MinSizeRel" "RelWithDebInfo")
endif()
```

### 2.3 set_property

在指定域中设置一个命名属性

```cmake
set_property(  <GLOBAL                            |
                DIRECTORY [dir]                   |
                TARGET    [target1 [target2 ...]] |
                SOURCE    [src1 [src2 ...]]       |
                TEST      [test1 [test2 ...]]     |
                CACHE     [entry1 [entry2 ...]]>
               [APPEND][APPEND_STRING]
               PROPERTY <name>[value1 [value2 ...]])
```

在某个域中对零个或多个对象设置一个属性。第一个参数决定该属性设置所在的域。它必须为下面中的其中之一：

GLOBAL域是唯一的，并且不接特殊的任何名字。

DIRECTORY域默认为当前目录，但也可以用全路径或相对路径指定其他的目录（前提是该目录已经被CMake处理）。

TARGET域可命名零或多个已经存在的目标。

SOURCE域可命名零或多个源文件。注意：源文件属性只对在相同目录下的目标是可见的(CMakeLists.txt)。

TEST域可命名零或多个已存在的测试。

CACHE域必须命名零或多个已存在条目的cache.

必选项PROPERTY后面紧跟着要设置的属性的名字。其他的参数用于构建以分号隔开的列表形式的属性值。如果指定了APPEND选项，则指定的列表将会追加到任何已存在的属性值当中。如果指定了APPEND_STRING选项，则会将值作为字符串追加到任何已存在的属性值。

### 2.4 get_property

从作用域中的一个对象获取一个属性。第一个参数指定存储结果的变量。第二个参数确定从中获取属性的范围。它必须是以下之一：

```cmake
get_property(  <variable>
               <GLOBAL             |
                DIRECTORY [dir]    |
                TARGET    <target> |
                SOURCE    <source> |
                TEST      <test>   |
                CACHE     <entry>  |
                VARIABLE>
               PROPERTY <name>
               [SET | DEFINED |BRIEF_DOCS | FULL_DOCS])
```

相关域的说明与set_property意义相同。

必选项PROPERTY后面紧跟着要获取的属性的名字。如果指定了SET选项，则变量会被设置为一个布尔值，表明该属性是否已设置。如果指定了DEFINED选项，则变量也会被设置为一个布尔值，表明该属性是否已定义（如通过define_property）。如果定义了BRIEF_DOCS或FULL_DOCS选项，则该变量被设置为一个字符串，包含了对请求的属性的文档。如果该属性没有相关文档，则会返回NOTFOUND。

## 3.构建示例

```shell
$ mkdir build

$ cd build/

$ cmake ..
Setting build type to 'RelWithDebInfo' as none was specified.
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
-- Build files have been written to: /home/matrim/workspace/cmake-examples/01-basic/F-build-type/build

$ make VERBOSE=1
/usr/bin/cmake -H/home/matrim/workspace/cmake-examples/01-basic/F-build-type -B/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build --check-build-system CMakeFiles/Makefile.cmake 0
/usr/bin/cmake -E cmake_progress_start /home/matrim/workspace/cmake-examples/01-basic/F-build-type/build/CMakeFiles /home/matrim/workspace/cmake-examples/01-basic/F-build-type/build/CMakeFiles/progress.marks
make -f CMakeFiles/Makefile2 all
make[1]: Entering directory `/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build'
make -f CMakeFiles/cmake_examples_build_type.dir/build.make CMakeFiles/cmake_examples_build_type.dir/depend
make[2]: Entering directory `/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build'
cd /home/matrim/workspace/cmake-examples/01-basic/F-build-type/build && /usr/bin/cmake -E cmake_depends "Unix Makefiles" /home/matrim/workspace/cmake-examples/01-basic/F-build-type /home/matrim/workspace/cmake-examples/01-basic/F-build-type /home/matrim/workspace/cmake-examples/01-basic/F-build-type/build /home/matrim/workspace/cmake-examples/01-basic/F-build-type/build /home/matrim/workspace/cmake-examples/01-basic/F-build-type/build/CMakeFiles/cmake_examples_build_type.dir/DependInfo.cmake --color=
Dependee "/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build/CMakeFiles/cmake_examples_build_type.dir/DependInfo.cmake" is newer than depender "/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build/CMakeFiles/cmake_examples_build_type.dir/depend.internal".
Dependee "/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build/CMakeFiles/CMakeDirectoryInformation.cmake" is newer than depender "/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build/CMakeFiles/cmake_examples_build_type.dir/depend.internal".
Scanning dependencies of target cmake_examples_build_type
make[2]: Leaving directory `/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build'
make -f CMakeFiles/cmake_examples_build_type.dir/build.make CMakeFiles/cmake_examples_build_type.dir/build
make[2]: Entering directory `/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build'
/usr/bin/cmake -E cmake_progress_report /home/matrim/workspace/cmake-examples/01-basic/F-build-type/build/CMakeFiles 1
[100%] Building CXX object CMakeFiles/cmake_examples_build_type.dir/main.cpp.o
/usr/bin/c++    -O2 -g -DNDEBUG   -o CMakeFiles/cmake_examples_build_type.dir/main.cpp.o -c /home/matrim/workspace/cmake-examples/01-basic/F-build-type/main.cpp
Linking CXX executable cmake_examples_build_type
/usr/bin/cmake -E cmake_link_script CMakeFiles/cmake_examples_build_type.dir/link.txt --verbose=1
/usr/bin/c++   -O2 -g -DNDEBUG    CMakeFiles/cmake_examples_build_type.dir/main.cpp.o  -o cmake_examples_build_type -rdynamic
make[2]: Leaving directory `/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build'
/usr/bin/cmake -E cmake_progress_report /home/matrim/workspace/cmake-examples/01-basic/F-build-type/build/CMakeFiles  1
[100%] Built target cmake_examples_build_type
make[1]: Leaving directory `/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build'
/usr/bin/cmake -E cmake_progress_start /home/matrim/workspace/cmake-examples/01-basic/F-build-type/build/CMakeFiles 0$ mkdir build
$ cd build/
/build$ cmake ..
Setting build type to 'RelWithDebInfo' as none was specified.
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
-- Build files have been written to: /home/matrim/workspace/cmake-examples/01-basic/F-build-type/build
/build$ make VERBOSE=1
/usr/bin/cmake -H/home/matrim/workspace/cmake-examples/01-basic/F-build-type -B/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build --check-build-system CMakeFiles/Makefile.cmake 0
/usr/bin/cmake -E cmake_progress_start /home/matrim/workspace/cmake-examples/01-basic/F-build-type/build/CMakeFiles /home/matrim/workspace/cmake-examples/01-basic/F-build-type/build/CMakeFiles/progress.marks
make -f CMakeFiles/Makefile2 all
make[1]: Entering directory `/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build'
make -f CMakeFiles/cmake_examples_build_type.dir/build.make CMakeFiles/cmake_examples_build_type.dir/depend
make[2]: Entering directory `/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build'
cd /home/matrim/workspace/cmake-examples/01-basic/F-build-type/build && /usr/bin/cmake -E cmake_depends "Unix Makefiles" /home/matrim/workspace/cmake-examples/01-basic/F-build-type /home/matrim/workspace/cmake-examples/01-basic/F-build-type /home/matrim/workspace/cmake-examples/01-basic/F-build-type/build /home/matrim/workspace/cmake-examples/01-basic/F-build-type/build /home/matrim/workspace/cmake-examples/01-basic/F-build-type/build/CMakeFiles/cmake_examples_build_type.dir/DependInfo.cmake --color=
Dependee "/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build/CMakeFiles/cmake_examples_build_type.dir/DependInfo.cmake" is newer than depender "/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build/CMakeFiles/cmake_examples_build_type.dir/depend.internal".
Dependee "/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build/CMakeFiles/CMakeDirectoryInformation.cmake" is newer than depender "/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build/CMakeFiles/cmake_examples_build_type.dir/depend.internal".
Scanning dependencies of target cmake_examples_build_type
make[2]: Leaving directory `/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build'
make -f CMakeFiles/cmake_examples_build_type.dir/build.make CMakeFiles/cmake_examples_build_type.dir/build
make[2]: Entering directory `/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build'
/usr/bin/cmake -E cmake_progress_report /home/matrim/workspace/cmake-examples/01-basic/F-build-type/build/CMakeFiles 1
[100%] Building CXX object CMakeFiles/cmake_examples_build_type.dir/main.cpp.o
/usr/bin/c++    -O2 -g -DNDEBUG   -o CMakeFiles/cmake_examples_build_type.dir/main.cpp.o -c /home/matrim/workspace/cmake-examples/01-basic/F-build-type/main.cpp
Linking CXX executable cmake_examples_build_type
/usr/bin/cmake -E cmake_link_script CMakeFiles/cmake_examples_build_type.dir/link.txt --verbose=1
/usr/bin/c++   -O2 -g -DNDEBUG    CMakeFiles/cmake_examples_build_type.dir/main.cpp.o  -o cmake_examples_build_type -rdynamic
make[2]: Leaving directory `/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build'
/usr/bin/cmake -E cmake_progress_report /home/matrim/workspace/cmake-examples/01-basic/F-build-type/build/CMakeFiles  1
[100%] Built target cmake_examples_build_type
make[1]: Leaving directory `/home/matrim/workspace/cmake-examples/01-basic/F-build-type/build'
/usr/bin/cmake -E cmake_progress_start /home/matrim/workspace/cmake-examples/01-basic/F-build-type/build/CMakeFiles 0
```

原文作者：[橘崽崽啊](https://www.cnblogs.com/juzaizai/)

原文链接：https://www.cnblogs.com/juzaizai/p/15069657.html