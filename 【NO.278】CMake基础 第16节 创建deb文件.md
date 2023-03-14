# 【NO.278】CMake基础 第16节 创建deb文件 

## 1.介绍

此示例显示如何使用deb格式生成Linux安装程序。

本教程中的文件如下：

```ruby
$ tree
.
├── cmake-examples.conf
├── CMakeLists.txt
├── include
│   └── Hello.h
└── src
    ├── Hello.cpp
    └── main.cpp
```

- [CMakeLists.txt] - 包含要运行的CMake命令。

  ```cmake
  cmake_minimum_required(VERSION 3.5)
  
  project(cmake_examples_deb)
  
  # set a project version
  set (deb_example_VERSION_MAJOR 0)
  set (deb_example_VERSION_MINOR 2)
  set (deb_example_VERSION_PATCH 2)
  set (deb_example_VERSION "${deb_example_VERSION_MAJOR}.${deb_example_VERSION_MINOR}.${deb_example_VERSION_PATCH}")
  
  
  ############################################################
  # Create a library
  ############################################################
  
  #Generate the shared library from the library sources
  add_library(cmake_examples_deb SHARED src/Hello.cpp)
  
  target_include_directories(cmake_examples_deb
      PUBLIC
          ${PROJECT_SOURCE_DIR}/include
  )
  ############################################################
  # Create an executable
  ############################################################
  
  # Add an executable with the above sources
  add_executable(cmake_examples_deb_bin src/main.cpp)
  
  # link the new hello_library target with the hello_binary target
  target_link_libraries( cmake_examples_deb_bin
      PUBLIC
          cmake_examples_deb
  )
  
  ############################################################
  # Install
  ############################################################
  
  # Binaries
  install (TARGETS cmake_examples_deb_bin
      DESTINATION bin)
  
  # Library
  # Note: may not work on windows
  install (TARGETS cmake_examples_deb
      LIBRARY DESTINATION lib)
  
  # Config
  install (FILES cmake-examples.conf
      DESTINATION etc)
  
  ############################################################
  # Create DEB
  ############################################################
  
  # Tell CPack to generate a .deb package
  set(CPACK_GENERATOR "DEB")
  
  # Set a Package Maintainer.
  # This is required
  set(CPACK_DEBIAN_PACKAGE_MAINTAINER "Thom Troy")
  
  # Set a Package Version
  set(CPACK_PACKAGE_VERSION ${deb_example_VERSION})
  
  # Include CPack
  include(CPack)
  ```

- [cmake-examples.conf] - 示例配置文件。

  ```ini
  # Sample configuration file that could be installed
  ```

- [include/Hello.h] - 要包含的头文件。

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
      std::cout << "Hello Install!" << std::endl;
  }
  ```

- [src/main.cpp] - 具有main的源文件。

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

### 2.1 生成器

`make package`目标可以使用CPack生成器来创建安装程序。

对于Debian包，你可以使用以下命令告诉CMake创建一个生成器：

```cmake
set(CPACK_GENERATOR "DEB")
```

在设置了描述软件包的各种设置之后，你必须使用以下命令告诉CMake包含CPack生成器。

```cmake
include(CPack)
```

包含后，通常使用make install目标安装的所有文件现在都可以打包到Debian包中。

### 2.2 Debian包设置

CPack公开了软件包的各种设置。在本例中，我们设置了以下内容：

```cmake
# Set a Package Maintainer.
# This is required
set(CPACK_DEBIAN_PACKAGE_MAINTAINER "Thom Troy")

# Set a Package Version
set(CPACK_PACKAGE_VERSION ${deb_example_VERSION})
```

它设置维护人员和版本。下面指定了更多Debian特定的设置。

| Variable                           | Info                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| CPACK_DEBIAN_PACKAGE_MAINTAINER    | Maintainer information                                       |
| CPACK_PACKAGE_DESCRIPTION_SUMMARY  | Package short description                                    |
| CPACK_PACKAGE_DESCRIPTION          | Package description                                          |
| CPACK_DEBIAN_PACKAGE_DEPENDS       | For advanced users to add custom scripts.                    |
| CPACK_DEBIAN_PACKAGE_CONTROL_EXTRA | The build directory you are currently in.                    |
| CPACK_DEBIAN_PACKAGE_SECTION       | Package section (see [here](http://packages.debian.org/stable/)) |
| CPACK_DEBIAN_PACKAGE_VERSION       | Package version                                              |

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
-- Build files have been written to: /home/matrim/workspace/cmake-examples/06-installer/deb/build

$ make help
The following are some of the valid targets for this Makefile:
... all (the default if no target is provided)
... clean
... depend
... cmake_examples_deb
... cmake_examples_deb_bin
... edit_cache
... install
... install/local
... install/strip
... list_install_components
... package
... package_source
... rebuild_cache
... src/Hello.o
... src/Hello.i
... src/Hello.s
... src/main.o
... src/main.i
... src/main.s

$ make package
Scanning dependencies of target cmake_examples_deb
[ 50%] Building CXX object CMakeFiles/cmake_examples_deb.dir/src/Hello.cpp.o
Linking CXX shared library libcmake_examples_deb.so
[ 50%] Built target cmake_examples_deb
Scanning dependencies of target cmake_examples_deb_bin
[100%] Building CXX object CMakeFiles/cmake_examples_deb_bin.dir/src/main.cpp.o
Linking CXX executable cmake_examples_deb_bin
[100%] Built target cmake_examples_deb_bin
Run CPack packaging tool...
CPack: Create package using DEB
CPack: Install projects
CPack: - Run preinstall target for: cmake_examples_deb
CPack: - Install project: cmake_examples_deb
CPack: Create package
CPack: - package: /home/matrim/workspace/cmake-examples/06-installer/deb/build/cmake_examples_deb-0.2.2-Linux.deb generated.

$ ls
CMakeCache.txt  cmake_examples_deb-0.2.2-Linux.deb  cmake_examples_deb_bin  CMakeFiles  cmake_install.cmake  CPackConfig.cmake  _CPack_Packages  CPackSourceConfig.cmake  install_manifest.txt  libcmake_examples_deb.so  Makefile
```

原文作者：[橘崽崽啊](https://www.cnblogs.com/juzaizai/)

原文链接：https://www.cnblogs.com/juzaizai/p/15069925.html