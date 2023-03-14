# 【NO.273】CMake基础 第11节 导入目标 

## 1.介绍

正如前面在第8节中提到的，较新版本的CMake允许你使用导入的别名目标链接第三方库。

本教程中的文件如下：

```objectivec
$ tree
.
├── CMakeLists.txt
├── main.cpp
```

- [CMakeLists.txt] - 包含要运行的CMake命令

  ```cmake
  cmake_minimum_required(VERSION 3.5)
  
  # Set the project name
  project (imported_targets)
  
  
  # find a boost install with the libraries filesystem and system
  find_package(Boost 1.46.1 REQUIRED COMPONENTS filesystem system)
  
  # check if boost was found
  if(Boost_FOUND)
      message ("boost found")
  else()
      message (FATAL_ERROR "Cannot find Boost")
  endif()
  
  # Add an executable
  add_executable(imported_targets main.cpp)
  
  # link against the boost libraries
  target_link_libraries( imported_targets
      PRIVATE
          Boost::filesystem
  )
  ```

- [main.cpp] - 具有main的源文件

  ```cpp
  #include <iostream>
  #include <boost/shared_ptr.hpp>
  #include <boost/filesystem.hpp>
  
  int main(int argc, char *argv[])
  {
      std::cout << "Hello Third Party Include!" << std::endl;
  
      // use a shared ptr
      boost::shared_ptr<int> isp(new int(4));
  
      // trivial use of boost filesystem
      boost::filesystem::path path = "/usr/share/cmake/modules";
      if(path.is_relative())
      {
          std::cout << "Path is relative" << std::endl;
      }
      else
      {
          std::cout << "Path is not relative" << std::endl;
      }
  
     return 0;
  }
  ```

## 2.要求

此示例需要以下条件:

- CMake v3.5+
- 安装在默认系统位置的Boost库

## 3.概念

### 3.1 导入目标

导入目标是由FindXXX模块导出的只读目标（例如Boost::filesystem）。

要包括Boost文件系统，你可以执行以下操作：

```cmake
  target_link_libraries( imported_targets
      PRIVATE
          Boost::filesystem
  )
```

这将自动链接`Boost::FileSystem`和`Boost::System`库，同时还包括`Boost include`目录（即不必手动添加include目录）。

## 4.构建示例

```shell
$ mkdir build

$ cd build/

$ cmake ..
-- The C compiler identification is GNU 5.4.0
-- The CXX compiler identification is GNU 5.4.0
-- Check for working C compiler: /usr/bin/cc
-- Check for working C compiler: /usr/bin/cc -- works
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Detecting C compile features
-- Detecting C compile features - done
-- Check for working CXX compiler: /usr/bin/c++
-- Check for working CXX compiler: /usr/bin/c++ -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Boost version: 1.58.0
-- Found the following Boost libraries:
--   filesystem
--   system
boost found
-- Configuring done
-- Generating done
-- Build files have been written to: /data/code/01-basic/K-imported-targets/build

$ make
Scanning dependencies of target imported_targets
[ 50%] Building CXX object CMakeFiles/imported_targets.dir/main.cpp.o
[100%] Linking CXX executable imported_targets
[100%] Built target imported_targets


$ ./imported_targets
Hello Third Party Include!
Path is not relative
```



原文作者：[橘崽崽啊](https://www.cnblogs.com/juzaizai/)

原文链接：https://www.cnblogs.com/juzaizai/p/15069682.html