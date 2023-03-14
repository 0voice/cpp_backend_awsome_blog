# 【NO.268】CMake基础 第8节 包含第三方库

## 1.介绍

几乎所有重要的项目都需要包含第三方库、头文件或程序。CMake支持使用`find_package()`函数查找这些工具的路径。这将从`CMAKE_MODULE_PATH`中的文件夹列表中搜索格式为`FindXXX.cmake`的CMake模块。在Linux上，默认搜索路径将包含`/usr/share/cmake/Modules`。在我的系统上，这包括对大约1420个通用第三方库的支持。

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
  project (third_party_include)
  
  
  # find a boost install with the libraries filesystem and system
  find_package(Boost 1.46.1 REQUIRED COMPONENTS filesystem system)
  
  # check if boost was found
  if(Boost_FOUND)
      message ("boost found")
  else()
      message (FATAL_ERROR "Cannot find Boost")
  endif()
  
  # Add an executable
  add_executable(third_party_include main.cpp)
  
  # link against the boost libraries
  target_link_libraries( third_party_include
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

此示例要求将Boost库安装在默认系统位置。

```shell
sudo apt-get install libboost-all-dev -y
```

## 3.概念

### 3.1 查找一个包

如上所述，`find_package()`函数将从`CMAKE_MODULE_PATH`中的文件夹列表中搜索格式为`FindXXX.cmake`的CMake模块。`find_package`的参数的确切格式将取决于你要查找的模块。这通常记录在文件`FindXXX.cmake`的顶部

下面是查找Boost的基本示例：

```cmake
find_package(Boost 1.46.1 REQUIRED COMPONENTS filesystem system)
```

这些参数是：

- Boost -库的名称。这是用于查找模块文件FindBoost.cmake的一部分。
- 1.46.1 - 要查找的Boost的最低版本。
- REQUIRED - 告诉模块这是必需的，如果失败，则找不到该模块。
- COMPONENTS - 要查找的库列表。

Boost includes可以接受更多参数，还可以利用其他变量。更复杂的设置将在后面的示例中提供。

### 3.2 检查是否找到该包

大多数包含的软件包都会设置一个变量`XXX_FOUND`，该变量可用于检查该软件包在系统上是否可用。

在本例中，变量为`BOOST_FOUND`：

```cmake
if(Boost_FOUND)
    message ("boost found")
    include_directories(${Boost_INCLUDE_DIRS})
else()
    message (FATAL_ERROR "Cannot find Boost")
endif()
```

### 3.3 导出变量

在找到包之后，它通常会导出变量，这些变量可以告诉用户在哪里可以找到库、头文件或可执行文件。与`XXX_FOUND`变量类似，它们是特定于包的，通常记录在`FindXXX.cmake`文件的顶部。

本例中导出的变量包括：

- `Boost_INCLUDE_DIRS` - Boost头文件的路径

在某些情况下，你还可以通过使用ccmake或cmake-gui检查缓存来检查这些变量。

### 3.4 别名/导入目标

大多数现代CMake库在其模块文件中导出别名目标。导入目标的好处在于，它们还可以填充头文件目录和链接库。

例如，从CMake的3.5版开始，Boost模块就支持此功能。

类似于将你自己的别名目标用于库，模块中的别名可以让引用找到的目标变得更容易。

在Boost的例子中，所有目标通过使用标识符`Boost::`加子模块的名字来导出。例如，你可以使用：

- `Boost::boost` 仅适用于库的头文件
- `Boost::system` 对于Boost系统库
- `Boost::filesystem` 对于文件系统库

与你自己的目标一样，这些目标包含它们的依赖项，因此链接到 `Boost::filesystem` 将自动添加 `Boost::boost`和`Boost::system`依赖。

要链接到导入的目标，可以使用以下命令：

```cmake
  target_link_libraries( third_party_include
      PRIVATE
          Boost::filesystem
  )
```

### 3.5 非别名目标

虽然大多数现代库使用导入的目标，但并非所有模块都已更新。在库尚未更新的情况下，你通常会发现以下变量可用：

- xxx_INCLUDE_DIRS - 指向库的include目录的变量
- xxx_LIBRARY - 指向库路径的变量.

然后，可以将这些文件添加到target_include_directory和target_link_library中：

```cmake
# Include the boost headers
target_include_directories( third_party_include
    PRIVATE ${Boost_INCLUDE_DIRS}
)

# link against the boost libraries
target_link_libraries( third_party_include
    PRIVATE
    ${Boost_SYSTEM_LIBRARY}
    ${Boost_FILESYSTEM_LIBRARY}
)
```

## 4.构建示例

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
-- Boost version: 1.54.0
-- Found the following Boost libraries:
--   filesystem
--   system
boost found
-- Configuring done
-- Generating done
-- Build files have been written to: /home/matrim/workspace/cmake-examples/01-basic/H-third-party-library/build

$ make
Scanning dependencies of target third_party_include
[100%] Building CXX object CMakeFiles/third_party_include.dir/main.cpp.o
Linking CXX executable third_party_include
[100%] Built target third_party_include
matrim@freyr:~/workspace/cmake-examples/01-basic/H-third-party-library/build$ ./
CMakeFiles/          third_party_include
matrim@freyr:~/workspace/cmake-examples/01-basic/H-third-party-library/build$ ./third_party_include
Hello Third Party Include!
Path is not relative
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
-- Boost version: 1.54.0
-- Found the following Boost libraries:
--   filesystem
--   system
boost found
-- Configuring done
-- Generating done
-- Build files have been written to: /home/matrim/workspace/cmake-examples/01-basic/H-third-party-library/build

$ make
Scanning dependencies of target third_party_include
[100%] Building CXX object CMakeFiles/third_party_include.dir/main.cpp.o
Linking CXX executable third_party_include
[100%] Built target third_party_include

$ ./third_party_include
Hello Third Party Include!
Path is not relative
```

原文作者：[橘崽崽啊](https://www.cnblogs.com/juzaizai/)

原文链接：https://www.cnblogs.com/juzaizai/p/15069669.html