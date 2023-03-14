# 【NO.275】CMake基础 第13节 构建子项目



## 1.介绍

此示例说明如何设置包含子项目的CMake项目。顶层CMakeLists.txt调用子目录中的CMakeLists.txt以创建以下内容：

- sublibrary1 - 静态库
- sublibrary2 - 头文件库
- subbinary - 可执行文件

此示例中包含的文件包括：

```mipsasm
$ tree
.
├── CMakeLists.txt
├── subbinary
│   ├── CMakeLists.txt
│   └── main.cpp
├── sublibrary1
│   ├── CMakeLists.txt
│   ├── include
│   │   └── sublib1
│   │       └── sublib1.h
│   └── src
│       └── sublib1.cpp
└── sublibrary2
    ├── CMakeLists.txt
    └── include
        └── sublib2
            └── sublib2.h
```

- [CMakeLists.txt] - 顶级CMakeLists.txt

  ```cmake
  cmake_minimum_required (VERSION 3.5)
  
  project(subprojects)
  
  # Add sub directories
  add_subdirectory(sublibrary1)
  add_subdirectory(sublibrary2)
  add_subdirectory(subbinary)
  ```

- [subbinary/CMakeLists.txt] - 生成可执行文件

  ```cmake
  project(subbinary)
  
  # Create the executable
  add_executable(${PROJECT_NAME} main.cpp)
  
  # Link the static library from subproject1 using it's alias sub::lib1
  # Link the header only library from subproject2 using it's alias sub::lib2
  # This will cause the include directories for that target to be added to this project
  target_link_libraries(${PROJECT_NAME}
      sub::lib1
      sub::lib2
  )
  ```

- [subbinary/main.cpp] - 可执行文件的源代码

  ```cpp
  #include "sublib1/sublib1.h"
  #include "sublib2/sublib2.h"
  
  int main(int argc, char *argv[])
  {
      sublib1 hi;
      hi.print();
  
      sublib2 howdy;
      howdy.print();
      
      return 0;
  }
  ```

- [sublibrary1/CMakeLists.txt] - 创建静态库

  ```cmake
  # Set the project name
  project (sublibrary1)
  
  # Add a library with the above sources
  add_library(${PROJECT_NAME} src/sublib1.cpp)
  add_library(sub::lib1 ALIAS ${PROJECT_NAME})
  
  target_include_directories( ${PROJECT_NAME}
      PUBLIC ${PROJECT_SOURCE_DIR}/include
  )
  ```

- [sublibrary1/include/sublib1/sublib1.h]

  ```cpp
  #ifndef __SUBLIB_1_H__
  #define __SUBLIB_1_H__
  
  class sublib1
  {
  public:
      void print();
  };
  
  #endif
  ```

- [sublibrary1/src/sublib1.cpp]

  ```cpp
  #include <iostream>
  
  #include "sublib1/sublib1.h"
  
  void sublib1::print()
  {
      std::cout << "Hello sub-library 1!" << std::endl;
  }
  ```

- [sublibrary2/CMakeLists.txt] - 设置仅含头文件的库

  ```cmake
  # Set the project name
  project (sublibrary2)
  
  add_library(${PROJECT_NAME} INTERFACE)
  add_library(sub::lib2 ALIAS ${PROJECT_NAME})
  
  target_include_directories(${PROJECT_NAME}
      INTERFACE
          ${PROJECT_SOURCE_DIR}/include
  )
  ```

- [sublibrary2/include/sublib2/sublib2.h]

  ```cpp
  #ifndef __SUBLIB_2_H__
  #define __SUBLIB_2_H__
  
  #include <iostream>
  
  class sublib2
  {
  public:
      void print()
      {
          std::cout << "Hello header only sub-library 2!" << std::endl;
      }
  };
  
  #endif
  ```

| Tip  | 在本例中，我将头文件移动到每个项目include目录下的一个子文件夹中，同时将目标include保留为根include文件夹。这是防止文件名冲突的好主意，因为你必须包含如下所示的文件：`#include“subib1/subib1.h”`。这也意味着，如果你为其他用户安装库，默认安装位置将是`/usr/local/include/subib1/subib1.h`。 |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

## 2.概念

### 2.1 添加子目录

CMakeLists.txt文件可以包含和调用含CMakeLists.txt的子目录

```cmake
add_subdirectory(sublibrary1)
add_subdirectory(sublibrary2)
add_subdirectory(subbinary)
```

### 2.2 引用子项目目录

当使用project()命令创建项目时，CMake将自动创建许多变量，这些变量可用于引用有关项目的详细信息。然后，其他子项目或主项目可以使用这些变量。例如，引用你可以使用的不同项目的源目录。

```cmake
    ${sublibrary1_SOURCE_DIR}
    ${sublibrary2_SOURCE_DIR}
```

CMake创建的变量包括：

| Variable           | Info                                                         |
| ------------------ | ------------------------------------------------------------ |
| PROJECT_NAME       | 由当前`project()`设置的项目名称                              |
| CMAKE_PROJECT_NAME | 由project()命令设置的第一个项目的名称，即顶级项目            |
| PROJECT_SOURCE_DIR | 当前项目的源目录                                             |
| PROJECT_BINARY_DIR | 当前项目的生成目录                                           |
| name_SOURCE_DIR    | 名为“name”的项目的源目录。在本例中，创建的源目录将是`sublibrary1_SOURCE_DIR`、`sublibrary2_SOURCE_DIR`和`subbinary_SOURCE_DIR` |
| name_BINARY_DIR    | 名为“name”的项目的二进制目录。在本例中，创建的二进制目录为`sublibrary1_BINARY_DIR` 、`sublibrary2_BINARY_DIR`和`subbinary_BINARY_DIR`。 |

### 2.3 头文件库

如果你有一个被创建为只包含头文件的库，cmake支持接口目标，以允许在没有任何构建输出的情况下创建目标。有关更多详细信息，请单击[此处](https://cmake.org/cmake/help/v3.4/command/add_library.html#interface-libraries)。

```cmake
add_library(${PROJECT_NAME} INTERFACE)
```

在创建目标时，你还可以使用INTERFACE作用域包含该目标的目录。INTERFACE作用域用于制定目标要求，这些要求在链接此目标的任何库中使用，但不用于目标本身的编译。如下示例，链接至此目标的任何目标都将包含一个include目录，但此目标本身并不进行编译（即不产生任何实体内容）：

```cmake
target_include_directories(${PROJECT_NAME}
    INTERFACE
        ${PROJECT_SOURCE_DIR}/include
)
```

### 2.4 从子项目中引用库

如果某个子项目创建库，则其他项目可以通过在`target_link_library()`命令中调用该项目的名称来引用该库。这意味着你不必引用新库的完整路径，它将作为依赖项被添加。

```cmake
target_link_libraries(subbinary
    PUBLIC
        sublibrary1
)
```

或者，你可以创建一个别名目标，使你可以在只读上下文中引用该目标。

要创建别名目标运行，请执行以下操作：

```cmake
add_library(sublibrary2)
add_library(sub::lib2 ALIAS sublibrary2)
```

要引用别名，只需如下所示：

```cmake
target_link_libraries(subbinary
    sub::lib2
)
```

### 2.5 包含来自子项目的头文件目录

当从子项目添加库时，从`cmake v3`开始，不需要使用它们在二进制文件的include目录中添加项目include目录。

这由创建库时`target_include_directory()`命令中的作用域控制。在本例中，因为子二进制可执行文件链接了subibrary1和subibrary2库，所以它将自动包括`${subibrary1_source_DIR}/include`和`${subibrary2_source_DIR}/include`文件夹，因为它们是随库的PUBLIC和INTERFACE范围导出的。

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
-- Build files have been written to: /home/matrim/workspace/cmake-examples/02-sub-projects/A-basic/build

$ make
Scanning dependencies of target sublibrary1
[ 50%] Building CXX object sublibrary1/CMakeFiles/sublibrary1.dir/src/sublib1.cpp.o
Linking CXX static library libsublibrary1.a
[ 50%] Built target sublibrary1
Scanning dependencies of target subbinary
[100%] Building CXX object subbinary/CMakeFiles/subbinary.dir/main.cpp.o
Linking CXX executable subbinary
[100%] Built target subbinary
```

原文作者：[橘崽崽啊](https://www.cnblogs.com/juzaizai/)

原文链接：https://www.cnblogs.com/juzaizai/p/15069693.html