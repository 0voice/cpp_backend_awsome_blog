# 【NO.276】CMake基础 第14节 在文件中进行变量替换

## 1.介绍

在调用cmake期间，可以创建使用CMakeLists.txt和cmake缓存中的变量的文件。在cmake生成期间，文件被复制到新位置，并替换所有cmake变量。

本教程中的文件如下：

```dos
$ tree
.
├── CMakeLists.txt
├── main.cpp
├── path.h.in
├── ver.h.in
```

- [CMakeLists.txt] - 包含要运行的CMake命令

  ```cmake
  cmake_minimum_required(VERSION 3.5)
  
  # Set the project name
  project (cf_example)
  
  # set a project version
  set (cf_example_VERSION_MAJOR 0)
  set (cf_example_VERSION_MINOR 2)
  set (cf_example_VERSION_PATCH 1)
  set (cf_example_VERSION "${cf_example_VERSION_MAJOR}.${cf_example_VERSION_MINOR}.${cf_example_VERSION_PATCH}")
  
  # Call configure files on ver.h.in to set the version.
  # Uses the standard ${VARIABLE} syntax in the file
  configure_file(ver.h.in ${PROJECT_BINARY_DIR}/ver.h)
  
  # configure the path.h.in file.
  # This file can only use the @VARIABLE@ syntax in the file
  configure_file(path.h.in ${PROJECT_BINARY_DIR}/path.h @ONLY)
  
  # Add an executable
  add_executable(cf_example
      main.cpp
  )
  
  # include the directory with the new files
  target_include_directories( cf_example
      PUBLIC
          ${CMAKE_BINARY_DIR}
  )
  ```

- [main.cpp] - 具有main的源文件

  ```cpp
  #include <iostream>
  #include "ver.h"
  #include "path.h"
  
  int main(int argc, char *argv[])
  {
      std::cout << "Hello Version " << ver << "!" << std::endl;
      std::cout << "Path is " << path << std::endl;
     return 0;
  }
  ```

- [path.h.in] - 包含构建目录路径的文件

  ```cpp
  #ifndef __PATH_H__
  #define __PATH_H__
  
  // version variable that will be substituted by cmake
  // This shows an example using the @ variable type
  const char* path = "@CMAKE_SOURCE_DIR@";
  
  #endif
  ```

- [ver.h.in] - 包含项目版本的文件

  ```cpp
  #ifndef __VER_H__
  #define __VER_H__
  
  // version variable that will be substituted by cmake
  // This shows an example using the $ variable type
  const char* ver = "${cf_example_VERSION}";
  
  #endif
  ```

## 2.概念

### 2.1 配置文件

要在文件中进行变量替换，可以使用CMake中的`configure_file()`函数。此函数的核心参数是源文件和目标文件。

```cmake
configure_file(ver.h.in ${PROJECT_BINARY_DIR}/ver.h)

configure_file(path.h.in ${PROJECT_BINARY_DIR}/path.h @ONLY)
```

上面的第一个示例允许使用像CMake变量一样的`${}`语法或ver.h.in文件中的`@@`定义变量。生成后，新文件ver.h将在`PROJECT_BINARY_DIR`中可用。

```cpp
const char* ver = "${cf_example_VERSION}";
```

第二个示例只允许在path.h.in文件中使用`@@`语法定义变量。生成后，将在`PROJECT_BINARY_DIR`中提供新文件path.h。

```cpp
const char* path = "@CMAKE_SOURCE_DIR@";
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
-- Build files have been written to: /home/matrim/workspace/cmake-examples/03-code-generation/configure-files/build

$ ls
CMakeCache.txt  CMakeFiles  cmake_install.cmake  Makefile  path.h  ver.h

$ cat path.h
#ifndef __PATH_H__
#define __PATH_H__

// version variable that will be substituted by cmake
// This shows an example using the @ variable type
const char* path = "/home/matrim/workspace/cmake-examples/03-code-generation/configure-files";

#endif

$ cat ver.h
#ifndef __VER_H__
#define __VER_H__

// version variable that will be substituted by cmake
// This shows an example using the $ variable type
const char* ver = "0.2.1";

#endif

$ make
Scanning dependencies of target cf_example
[100%] Building CXX object CMakeFiles/cf_example.dir/main.cpp.o
Linking CXX executable cf_example
[100%] Built target cf_example

$ ./cf_example
Hello Version 0.2.1!
Path is /home/matrim/workspace/cmake-examples/03-code-generation/configure-files
```

原文作者：[橘崽崽啊](https://www.cnblogs.com/juzaizai/)

原文链接：https://www.cnblogs.com/juzaizai/p/15069698.html