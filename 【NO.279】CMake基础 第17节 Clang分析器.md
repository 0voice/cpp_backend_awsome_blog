# 【NO.279】CMake基础 第17节 Clang分析器

## 1.介绍

此示例说明如何调用Clang Static Analyzer以使用scan-build工具执行静态分析。

此示例中包含的文件包括：

```shell
$ tree
.
├── CMakeLists.txt
├── subproject1
│   ├── CMakeLists.txt
│   └── main1.cpp
└── subproject2
    ├── CMakeLists.txt
    └── main2.cpp
```

- [CMakeLists.txt] - 顶级CMakeLists.txt。

  ```cmake
  cmake_minimum_required (VERSION 3.5)
  
  project(cppcheck_analysis)
  
  # Use debug build as recommended
  set(CMAKE_BUILD_TYPE Debug)
  
  # Have cmake create a compile database
  set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
  
  # Add sub directories
  add_subdirectory(subproject1)
  add_subdirectory(subproject2)
  ```

- [subproject1/CMakeLists.txt] - C子项目1的Make命令。

  ```cmake
  # Set the project name
  project (subproject1)
  
  # Add an executable with the above sources
  add_executable(${PROJECT_NAME} main1.cpp)
  ```

- [subproject1/main.cpp] - 子项目的源代码，没有错误。

  ```cpp
  #include <iostream>
  
  int main(int argc, char *argv[])
  {
     std::cout << "Hello Main1!" << std::endl;
     return 0;
  }
  ```

- [subproject2/CMakeLists.txt] - C子项目2的Make命令。

  ```cmake
  # Set the project name
  project (subproject2)
  
  # Add an executable with the above sources
  add_executable(${PROJECT_NAME} main2.cpp)
  ```

- [subproject2/main2.cpp] - 包含错误的子项目的源代码。

  ```cpp
  #include <iostream>
  
  int main(int argc, char *argv[])
  {
     std::cout << "Hello Main2!" << std::endl;
     int* x = NULL;
     std::cout << *x << std::endl;
     return 0;
  }
  ```

## 2.要求

要运行此示例，必须安装CLANG分析器和scan-build工具。可以使用以下命令将其安装在Ubuntu上。

```shell
$ sudo apt-get install clang
$ sudo apt-get install clang-tools
```

该工具以下方式可用：

```shell
$ scan-build-3.6
```

## 3.概念

### 3.1 scan-build

要运行clang静态分析器，你可以在运行编译器的同时使用工具Scan-Build来运行分析器。 这会覆盖CC和CXX环境变量，并用它自己的工具替换它们。要运行它，你可以执行以下操作：

```shell
$ scan-build-3.6 cmake ..
$ scan-build-3.6 make
```

默认情况下，这将运行你的平台的标准编译器，即LINUX上的GCC。但是，如果要覆盖此设置，可以将命令更改为：

```shell
$ scan-build-3.6 --use-cc=clang-3.6 --use-c++=clang++-3.6 -o ./scanbuildout/ make
```

### 3.2 scan-build输出

scan-build仅在编译时输出警告，还将生成包含错误详细分析的HTML文件列表。

```shell
$ cd scanbuildout/
$ tree
.
└── 2017-07-03-213514-3653-1
    ├── index.html
    ├── report-42eba1.html
    ├── scanview.css
    └── sorttable.js

1 directory, 4 files
```

默认情况下，这些文件输出到`/tmp/scanbuildout/{run folder}`。你可以使用 `scan-build -o /output/folder`文件夹更改此设置。

## 4.构建示例

```shell
$ mkdir build

$ cd build/

$ scan-build-3.6 -o ./scanbuildout make
scan-build: Using '/usr/lib/llvm-3.6/bin/clang' for static analysis
make: *** No targets specified and no makefile found.  Stop.
scan-build: Removing directory '/data/code/clang-analyzer/build/scanbuildout/2017-07-03-211632-937-1' because it contains no reports.
scan-build: No bugs found.
devuser@91457fbfa423:/data/code/clang-analyzer/build$ scan-build-3.6 -o ./scanbuildout cmake ..
scan-build: Using '/usr/lib/llvm-3.6/bin/clang' for static analysis
-- The C compiler identification is GNU 5.4.0
-- The CXX compiler identification is GNU 5.4.0
-- Check for working C compiler: /usr/share/clang/scan-build-3.6/ccc-analyzer
-- Check for working C compiler: /usr/share/clang/scan-build-3.6/ccc-analyzer -- works
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Detecting C compile features
-- Detecting C compile features - done
-- Check for working CXX compiler: /usr/share/clang/scan-build-3.6/c++-analyzer
-- Check for working CXX compiler: /usr/share/clang/scan-build-3.6/c++-analyzer -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Found CPPCHECK: /usr/local/bin/cppcheck
cppcheck found. Use cppccheck-analysis targets to run it
-- Configuring done
-- Generating done
-- Build files have been written to: /data/code/clang-analyzer/build
scan-build: Removing directory '/data/code/clang-analyzer/build/scanbuildout/2017-07-03-211641-941-1' because it contains no reports.
scan-build: No bugs found.

$ $ scan-build-3.6 -o ./scanbuildout make
scan-build: Using '/usr/lib/llvm-3.6/bin/clang' for static analysis
Scanning dependencies of target subproject1
[ 25%] Building CXX object subproject1/CMakeFiles/subproject1.dir/main1.cpp.o
[ 50%] Linking CXX executable subproject1
[ 50%] Built target subproject1
Scanning dependencies of target subproject2
[ 75%] Building CXX object subproject2/CMakeFiles/subproject2.dir/main2.cpp.o
/data/code/clang-analyzer/subproject2/main2.cpp:7:17: warning: Dereference of null pointer (loaded from variable 'x')
   std::cout << *x << std::endl;
                ^~
1 warning generated.
[100%] Linking CXX executable subproject2
[100%] Built target subproject2
scan-build: 1 bug found.
scan-build: Run 'scan-view /data/code/clang-analyzer/build/scanbuildout/2017-07-03-211647-1172-1' to examine bug reports.

$ cd scanbuildout/
$ tree
.
└── 2017-07-03-213514-3653-1
    ├── index.html
    ├── report-42eba1.html
    ├── scanview.css
    └── sorttable.js

1 directory, 4 files
```

原文作者：[橘崽崽啊](https://www.cnblogs.com/juzaizai/)

原文链接：https://www.cnblogs.com/juzaizai/p/15072093.html