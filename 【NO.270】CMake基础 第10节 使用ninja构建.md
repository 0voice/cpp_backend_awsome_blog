# 【NO.270】CMake基础 第10节 使用ninja构建

## 1.介绍

如前所述，CMake是一个元（meta）构建系统，可用于为许多其他构建工具创建构建文件。这个例子展示了如何让CMake使用ninja构建工具。

本教程中的文件如下：

```shell
$ tree
.
├── CMakeLists.txt
├── main.cpp
```

- [CMakeLists.txt] - 包含要运行的CMake命令

  ```cmake
  # Set the minimum version of CMake that can be used
  # To find the cmake version run
  # $ cmake --version
  cmake_minimum_required(VERSION 3.5)
  
  # Set the project name
  project (hello_cmake)
  
  # Add an executable
  add_executable(hello_cmake main.cpp)
  ```

- [main.cpp] - 一个简单的“Hello World”CPP文件

  ```cpp
  #include <iostream>
  
  int main(int argc, char *argv[])
  {
     std::cout << "Hello CMake!" << std::endl;
     return 0;
  }
  ```

## 2.概念

### 2.1 生成器

CMake生成器负责为底层构建系统编写输入文件(例如Makefile)。运行`cmake--help`将显示可用的生成器。对于cmake v2.8.12.2，我的系统支持的生成器包括：

```shell
Generators

The following generators are available on this platform:
  Unix Makefiles              = Generates standard UNIX makefiles.
  Ninja                       = Generates build.ninja files (experimental).
  CodeBlocks - Ninja          = Generates CodeBlocks project files.
  CodeBlocks - Unix Makefiles = Generates CodeBlocks project files.
  Eclipse CDT4 - Ninja        = Generates Eclipse CDT 4.0 project files.
  Eclipse CDT4 - Unix Makefiles
                              = Generates Eclipse CDT 4.0 project files.
  KDevelop3                   = Generates KDevelop 3 project files.
  KDevelop3 - Unix Makefiles  = Generates KDevelop 3 project files.
  Sublime Text 2 - Ninja      = Generates Sublime Text 2 project files.
  Sublime Text 2 - Unix Makefiles
                              = Generates Sublime Text 2 project files.Generators
```

正如本文所述，CMake包括不同类型的生成器，如命令行生成器、IDE生成器和其他生成器。

#### 2.1.1 命令行生成工具生成器

这些生成器用于命令行构建工具，如Make和Ninja。在使用CMake生成构建系统之前，必须配置所选的工具链。

支持的生成器包括：

- Borland Makefiles
- MSYS Makefiles
- MinGW Makefiles
- NMake Makefiles
- NMake Makefiles JOM
- Ninja
- Unix Makefiles
- Watcom WMake

#### 2.1.2 IDE构建工具生成器

这些生成器用于集成开发环境，其中包括它们自己的编译器。例如Visual Studio和Xcode，它们本身就包含一个编译器。

支持的生成器包括：

- Visual Studio 6
- Visual Studio 7
- Visual Studio 7 .NET 2003
- Visual Studio 8 2005
- Visual Studio 9 2008
- Visual Studio 10 2010
- Visual Studio 11 2012
- Visual Studio 12 2013
- Xcode

#### 2.1.3 其他生成器

这些生成器创建配置并与其他IDE工具共同工作，并且必须包含在IDE或命令行生成器中。

支持的生成器包括：

- CodeBlocks
- CodeLite
- Eclipse CDT4
- KDevelop3
- Kate
- Sublime Text 2

| Note | 在本例中，ninja是通过命令`sudo apt-get install ninja-build`安装的。 |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

### 2.2 调用生成器

要调用CMake生成器，可以使用-G命令行开关，例如：

```undefined
cmake .. -G Ninja
```

完成上述操作后，CMake将生成所需的Ninja构建文件，这些文件可以通过使用Ninja命令运行。

```shell
$ cmake .. -G Ninja

$ ls
build.ninja  CMakeCache.txt  CMakeFiles  cmake_install.cmake  rules.ninja
```

## 3.构建示例

下面是构建此示例的示例输出。

```shell
$ mkdir build.ninja

$ cd build.ninja/

$ cmake .. -G Ninja
-- The C compiler identification is GNU 4.8.4
-- The CXX compiler identification is GNU 4.8.4
-- Check for working C compiler using: Ninja
-- Check for working C compiler using: Ninja -- works
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working CXX compiler using: Ninja
-- Check for working CXX compiler using: Ninja -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Configuring done
-- Generating done
-- Build files have been written to: /home/matrim/workspace/cmake-examples/01-basic/J-building-with-ninja/build.ninja

$ ninja -v
[1/2] /usr/bin/c++     -MMD -MT CMakeFiles/hello_cmake.dir/main.cpp.o -MF "CMakeFiles/hello_cmake.dir/main.cpp.o.d" -o CMakeFiles/hello_cmake.dir/main.cpp.o -c ../main.cpp
[2/2] : && /usr/bin/c++      CMakeFiles/hello_cmake.dir/main.cpp.o  -o hello_cmake  -rdynamic && :

$ ls
build.ninja  CMakeCache.txt  CMakeFiles  cmake_install.cmake  hello_cmake  rules.ninja

$ ./hello_cmake
Hello CMake!
```

原文作者：[橘崽崽啊](https://www.cnblogs.com/juzaizai/)

原文链接：https://www.cnblogs.com/juzaizai/p/15069678.html