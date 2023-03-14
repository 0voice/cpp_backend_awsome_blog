# 【NO.274】CMake基础 第12节 设置C++标准

## 1.介绍

自从C++11和C++14发布以来，一个常见的用例是调用编译器来使用这些标准。随着CMake的发展，它添加了一些功能来使这一点变得更容易，而CMake的新版本已经改变了实现这一点的方式。下面的示例显示了设置C++标准的三种不同方法，以及可以使用哪些版本的CMake。

这些例子包括：

- [common-method]. 一个简单的版本，应该可以与大多数版本的CMake一起使用
- [cxx-standard]. 使用CMake v3.1中引入的`CMAKE_CXX_STANDARD`变量
- [compile-features]. 使用CMakev3.1中引入的`target_compile_features`函数

## 2.一个简单的版本

此示例显示了设置C++标准的通用方法。这可以与大多数版本的CMake一起使用。但是，如果你使用CMake的最新版本，则可以用更方便的方法。

本教程中的文件如下：

```objectivec
A-hello-cmake$ tree
.
├── CMakeLists.txt
├── main.cpp
```

- [CMakeLists.txt] - 包含要运行的CMake命令。

  ```cmake
  # Set the minimum version of CMake that can be used
  # To find the cmake version run
  # $ cmake --version
  cmake_minimum_required(VERSION 2.8)
  
  # Set the project name
  project (hello_cpp11)
  
  # try conditional compilation
  
  #  Check whether the CXX compiler supports a given flag.
  ## CHECK_CXX_COMPILER_FLAG(<flag> <var>)
  #  <flag> - the compiler flag
  #  <var>  - variable to store the result
  #  This internally calls the check_cxx_source_compiles macro and sets CMAKE_REQUIRED_DEFINITIONS to <flag>
  include(CheckCXXCompilerFlag)
  CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
  CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
  
  # check results and add flag
  if(COMPILER_SUPPORTS_CXX11)#
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
  elseif(COMPILER_SUPPORTS_CXX0X)#
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
  else()
      message(STATUS "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
  endif()
  
  # Add an executable
  add_executable(hello_cpp11 main.cpp)
  ```

- [main.cpp] - 一个针对C++11的简单的“Hello World”CPP文件。

  ```cpp
  #include <iostream>
  
  int main(int argc, char *argv[])
  {
      auto message = "Hello C++11";
      std::cout << message << std::endl;
      return 0;
  }
  ```

### 2.1 概念

#### 2.1.1 检查编译标志

CMake支持尝试使用传递给函数CMAKE_CXX_COMPILER_FLAG的任何标志编译程序。然后将结果存储在你传入的变量中。

例如:

```cmake
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
```

此示例将尝试使用标志`-std=c++11`编译程序，并将结果存储在变量`COMPILER_SUPPORTS_CXX11`中。

`include(CheckCXXCompilerFlag)`这一行告诉CMake包含此函数以使其可用。

#### 2.1.2 添加标志

一旦确定编译是否支持标志，就可以使用标准的cmake方法将该标志添加到目标。在本例中，我们使用CMAKE_CXX_FLAGS将该标志传递到所有目标。

```cmake
if(COMPILER_SUPPORTS_CXX11)#
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
elseif(COMPILER_SUPPORTS_CXX0X)#
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
else()
    message(STATUS "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
endif()
```

上面的示例只检查编译标志的GCC版本，并支持从C++11回退到标准化前的C++0x标志。在实际使用中，你可能希望检查C14，或者添加对不同编译设置方法的支持，例如`-std=gnu11`。

### 2.2 构建示例

下面是构建此示例的示例输出。

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
-- Performing Test COMPILER_SUPPORTS_CXX11
-- Performing Test COMPILER_SUPPORTS_CXX11 - Success
-- Performing Test COMPILER_SUPPORTS_CXX0X
-- Performing Test COMPILER_SUPPORTS_CXX0X - Success
-- Configuring done
-- Generating done
-- Build files have been written to: /data/code/01-basic/L-cpp-standard/i-common-method/build

$ make VERBOSE=1
/usr/bin/cmake -H/data/code/01-basic/L-cpp-standard/i-common-method -B/data/code/01-basic/L-cpp-standard/i-common-method/build --check-build-system CMakeFiles/Makefile.cmake 0
/usr/bin/cmake -E cmake_progress_start /data/code/01-basic/L-cpp-standard/i-common-method/build/CMakeFiles /data/code/01-basic/L-cpp-standard/i-common-method/build/CMakeFiles/progress.marks
make -f CMakeFiles/Makefile2 all
make[1]: Entering directory `/data/code/01-basic/L-cpp-standard/i-common-method/build'
make -f CMakeFiles/hello_cpp11.dir/build.make CMakeFiles/hello_cpp11.dir/depend
make[2]: Entering directory `/data/code/01-basic/L-cpp-standard/i-common-method/build'
cd /data/code/01-basic/L-cpp-standard/i-common-method/build && /usr/bin/cmake -E cmake_depends "Unix Makefiles" /data/code/01-basic/L-cpp-standard/i-common-method /data/code/01-basic/L-cpp-standard/i-common-method /data/code/01-basic/L-cpp-standard/i-common-method/build /data/code/01-basic/L-cpp-standard/i-common-method/build /data/code/01-basic/L-cpp-standard/i-common-method/build/CMakeFiles/hello_cpp11.dir/DependInfo.cmake --color=
Dependee "/data/code/01-basic/L-cpp-standard/i-common-method/build/CMakeFiles/hello_cpp11.dir/DependInfo.cmake" is newer than depender "/data/code/01-basic/L-cpp-standard/i-common-method/build/CMakeFiles/hello_cpp11.dir/depend.internal".
Dependee "/data/code/01-basic/L-cpp-standard/i-common-method/build/CMakeFiles/CMakeDirectoryInformation.cmake" is newer than depender "/data/code/01-basic/L-cpp-standard/i-common-method/build/CMakeFiles/hello_cpp11.dir/depend.internal".
Scanning dependencies of target hello_cpp11
make[2]: Leaving directory `/data/code/01-basic/L-cpp-standard/i-common-method/build'
make -f CMakeFiles/hello_cpp11.dir/build.make CMakeFiles/hello_cpp11.dir/build
make[2]: Entering directory `/data/code/01-basic/L-cpp-standard/i-common-method/build'
/usr/bin/cmake -E cmake_progress_report /data/code/01-basic/L-cpp-standard/i-common-method/build/CMakeFiles 1
[100%] Building CXX object CMakeFiles/hello_cpp11.dir/main.cpp.o
/usr/bin/c++    -std=c++11   -o CMakeFiles/hello_cpp11.dir/main.cpp.o -c /data/code/01-basic/L-cpp-standard/i-common-method/main.cpp
Linking CXX executable hello_cpp11
/usr/bin/cmake -E cmake_link_script CMakeFiles/hello_cpp11.dir/link.txt --verbose=1
/usr/bin/c++    -std=c++11    CMakeFiles/hello_cpp11.dir/main.cpp.o  -o hello_cpp11 -rdynamic
make[2]: Leaving directory `/data/code/01-basic/L-cpp-standard/i-common-method/build'
/usr/bin/cmake -E cmake_progress_report /data/code/01-basic/L-cpp-standard/i-common-method/build/CMakeFiles  1
[100%] Built target hello_cpp11
make[1]: Leaving directory `/data/code/01-basic/L-cpp-standard/i-common-method/build'
/usr/bin/cmake -E cmake_progress_start /data/code/01-basic/L-cpp-standard/i-common-method/build/CMakeFiles 0
```

## 3.使用CMAKE_CXX_STANDARD变量

### 3.1 介绍

此示例说明如何使用CMAKE_CXX_STANDARD变量设置C++标准。这是从CMake v3.1开始提供的。

本教程中的文件如下：

```objectivec
A-hello-cmake$ tree
.
├── CMakeLists.txt
├── main.cpp
```

- [CMakeLists.txt] - 包含要运行的CMake命令

  ```cmake
  # Set the minimum version of CMake that can be used
  # To find the cmake version run
  # $ cmake --version
  cmake_minimum_required(VERSION 3.1)
  
  # Set the project name
  project (hello_cpp11)
  
  # set the C++ standard to C++ 11
  set(CMAKE_CXX_STANDARD 11)
  
  # Add an executable
  add_executable(hello_cpp11 main.cpp)
  ```

- [main.cpp] - 一个针对C++11的简单的“Hello World”CPP文件

  ```cpp
  #include <iostream>
  
  int main(int argc, char *argv[])
  {
      auto message = "Hello C++11";
      std::cout << message << std::endl;
      return 0;
  }
  ```

### 3.2 概念

#### 3.2.1 使用CXX_STANDARD属性

设置CMAKE_CXX_STANDARD变量会导致所有目标上的CXX_STANDARD属性改变。这会影响CMake在编译时设置适当的标志。

| Note | `CMAKE_CXX_STANDARD`变量会回退到最接近的不会失败的适当标准。例如，如果请求`-std=gnu11`，最后可能变成`-std=gnu0x`。这可能会在编译时导致意外故障。 |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

### 3.3 构建示例

下面是构建此示例的示例输出：

```shell
$ mkdir build
$ cd build


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
-- Configuring done
-- Generating done
-- Build files have been written to: /data/code/01-basic/L-cpp-standard/ii-cxx-standard/build

$ make VERBOSE=1
/usr/bin/cmake -H/data/code/01-basic/L-cpp-standard/ii-cxx-standard -B/data/code/01-basic/L-cpp-standard/ii-cxx-standard/build --check-build-system CMakeFiles/Makefile.cmake 0
/usr/bin/cmake -E cmake_progress_start /data/code/01-basic/L-cpp-standard/ii-cxx-standard/build/CMakeFiles /data/code/01-basic/L-cpp-standard/ii-cxx-standard/build/CMakeFiles/progress.marks
make -f CMakeFiles/Makefile2 all
make[1]: Entering directory '/data/code/01-basic/L-cpp-standard/ii-cxx-standard/build'
make -f CMakeFiles/hello_cpp11.dir/build.make CMakeFiles/hello_cpp11.dir/depend
make[2]: Entering directory '/data/code/01-basic/L-cpp-standard/ii-cxx-standard/build'
cd /data/code/01-basic/L-cpp-standard/ii-cxx-standard/build && /usr/bin/cmake -E cmake_depends "Unix Makefiles" /data/code/01-basic/L-cpp-standard/ii-cxx-standard /data/code/01-basic/L-cpp-standard/ii-cxx-standard /data/code/01-basic/L-cpp-standard/ii-cxx-standard/build /data/code/01-basic/L-cpp-standard/ii-cxx-standard/build /data/code/01-basic/L-cpp-standard/ii-cxx-standard/build/CMakeFiles/hello_cpp11.dir/DependInfo.cmake --color=
Dependee "/data/code/01-basic/L-cpp-standard/ii-cxx-standard/build/CMakeFiles/hello_cpp11.dir/DependInfo.cmake" is newer than depender "/data/code/01-basic/L-cpp-standard/ii-cxx-standard/build/CMakeFiles/hello_cpp11.dir/depend.internal".
Dependee "/data/code/01-basic/L-cpp-standard/ii-cxx-standard/build/CMakeFiles/CMakeDirectoryInformation.cmake" is newer than depender "/data/code/01-basic/L-cpp-standard/ii-cxx-standard/build/CMakeFiles/hello_cpp11.dir/depend.internal".
Scanning dependencies of target hello_cpp11
make[2]: Leaving directory '/data/code/01-basic/L-cpp-standard/ii-cxx-standard/build'
make -f CMakeFiles/hello_cpp11.dir/build.make CMakeFiles/hello_cpp11.dir/build
make[2]: Entering directory '/data/code/01-basic/L-cpp-standard/ii-cxx-standard/build'
[ 50%] Building CXX object CMakeFiles/hello_cpp11.dir/main.cpp.o
/usr/bin/c++     -std=gnu++11 -o CMakeFiles/hello_cpp11.dir/main.cpp.o -c /data/code/01-basic/L-cpp-standard/ii-cxx-standard/main.cpp
[100%] Linking CXX executable hello_cpp11
/usr/bin/cmake -E cmake_link_script CMakeFiles/hello_cpp11.dir/link.txt --verbose=1
/usr/bin/c++      CMakeFiles/hello_cpp11.dir/main.cpp.o  -o hello_cpp11 -rdynamic
make[2]: Leaving directory '/data/code/01-basic/L-cpp-standard/ii-cxx-standard/build'
[100%] Built target hello_cpp11
make[1]: Leaving directory '/data/code/01-basic/L-cpp-standard/ii-cxx-standard/build'
```

## 4.使用target_compile_features函数

### 4.1 介绍

此示例说明如何使用`target_compile_features`函数设置C++标准。

本教程中的文件如下：

```objectivec
A-hello-cmake$ tree
.
├── CMakeLists.txt
├── main.cpp
```

- [CMakeLists.txt] - 包含要运行的CMake命令

  ```cmake
  # Set the minimum version of CMake that can be used
  # To find the cmake version run
  # $ cmake --version
  cmake_minimum_required(VERSION 3.1)
  
  # Set the project name
  project (hello_cpp11)
  
  # Add an executable
  add_executable(hello_cpp11 main.cpp)
  
  # set the C++ standard to the appropriate standard for using auto
  target_compile_features(hello_cpp11 PUBLIC cxx_auto_type)
  
  # Print the list of known compile features for this version of CMake
  message("List of compile features: ${CMAKE_CXX_COMPILE_FEATURES}")
  ```

- [main.cpp] - 一个针对C++11的简单的“Hello World”C++文件

  ```cpp
  #include <iostream>
  
  int main(int argc, char *argv[])
  {
      auto message = "Hello C++11";
      std::cout << message << std::endl;
      return 0;
  }
  ```

### 4.2 概念

#### 4.2.1 使用target_compile_features

在目标上调用target_compile_features函数将检查传入的功能，并由CMake确定正确的用于目标的编译器标志。

```cmake
target_compile_features(hello_cpp11 PUBLIC cxx_auto_type)
```

与其他`target_*`函数一样，你可以指定所选目标的功能范围。这将填充目标的`INTERFACE_COMPILE_FEATURES`属性。可用功能列表可从`CMAKE_CXX_COMPILE_FEATURES`变量中找到。你可以使用以下代码获取可用功能的列表：

```cmake
message("List of compile features: ${CMAKE_CXX_COMPILE_FEATURES}")
```

### 4.3 构建示例

下面是构建此示例的输出。

```shell
$ mkdir build
$ cd build

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
List of compile features: cxx_template_template_parameters;cxx_alias_templates;cxx_alignas;cxx_alignof;cxx_attributes;cxx_auto_type;cxx_constexpr;cxx_decltype;cxx_decltype_incomplete_return_types;cxx_default_function_template_args;cxx_defaulted_functions;cxx_defaulted_move_initializers;cxx_delegating_constructors;cxx_deleted_functions;cxx_enum_forward_declarations;cxx_explicit_conversions;cxx_extended_friend_declarations;cxx_extern_templates;cxx_final;cxx_func_identifier;cxx_generalized_initializers;cxx_inheriting_constructors;cxx_inline_namespaces;cxx_lambdas;cxx_local_type_template_args;cxx_long_long_type;cxx_noexcept;cxx_nonstatic_member_init;cxx_nullptr;cxx_override;cxx_range_for;cxx_raw_string_literals;cxx_reference_qualified_functions;cxx_right_angle_brackets;cxx_rvalue_references;cxx_sizeof_member;cxx_static_assert;cxx_strong_enums;cxx_thread_local;cxx_trailing_return_types;cxx_unicode_literals;cxx_uniform_initialization;cxx_unrestricted_unions;cxx_user_literals;cxx_variadic_macros;cxx_variadic_templates;cxx_aggregate_default_initializers;cxx_attribute_deprecated;cxx_binary_literals;cxx_contextual_conversions;cxx_decltype_auto;cxx_digit_separators;cxx_generic_lambdas;cxx_lambda_init_captures;cxx_relaxed_constexpr;cxx_return_type_deduction;cxx_variable_templates
-- Configuring done
-- Generating done
-- Build files have been written to: /data/code/01-basic/L-cpp-standard/iii-compile-features/build


$ make VERBOSE=1
/usr/bin/cmake -H/data/code/01-basic/L-cpp-standard/iii-compile-features -B/data/code/01-basic/L-cpp-standard/iii-compile-features/build --check-build-system CMakeFiles/Makefile.cmake 0
/usr/bin/cmake -E cmake_progress_start /data/code/01-basic/L-cpp-standard/iii-compile-features/build/CMakeFiles /data/code/01-basic/L-cpp-standard/iii-compile-features/build/CMakeFiles/progress.marks
make -f CMakeFiles/Makefile2 all
make[1]: Entering directory '/data/code/01-basic/L-cpp-standard/iii-compile-features/build'
make -f CMakeFiles/hello_cpp11.dir/build.make CMakeFiles/hello_cpp11.dir/depend
make[2]: Entering directory '/data/code/01-basic/L-cpp-standard/iii-compile-features/build'
cd /data/code/01-basic/L-cpp-standard/iii-compile-features/build && /usr/bin/cmake -E cmake_depends "Unix Makefiles" /data/code/01-basic/L-cpp-standard/iii-compile-features /data/code/01-basic/L-cpp-standard/iii-compile-features /data/code/01-basic/L-cpp-standard/iii-compile-features/build /data/code/01-basic/L-cpp-standard/iii-compile-features/build /data/code/01-basic/L-cpp-standard/iii-compile-features/build/CMakeFiles/hello_cpp11.dir/DependInfo.cmake --color=
Dependee "/data/code/01-basic/L-cpp-standard/iii-compile-features/build/CMakeFiles/hello_cpp11.dir/DependInfo.cmake" is newer than depender "/data/code/01-basic/L-cpp-standard/iii-compile-features/build/CMakeFiles/hello_cpp11.dir/depend.internal".
Dependee "/data/code/01-basic/L-cpp-standard/iii-compile-features/build/CMakeFiles/CMakeDirectoryInformation.cmake" is newer than depender "/data/code/01-basic/L-cpp-standard/iii-compile-features/build/CMakeFiles/hello_cpp11.dir/depend.internal".
Scanning dependencies of target hello_cpp11
make[2]: Leaving directory '/data/code/01-basic/L-cpp-standard/iii-compile-features/build'
make -f CMakeFiles/hello_cpp11.dir/build.make CMakeFiles/hello_cpp11.dir/build
make[2]: Entering directory '/data/code/01-basic/L-cpp-standard/iii-compile-features/build'
[ 50%] Building CXX object CMakeFiles/hello_cpp11.dir/main.cpp.o
/usr/bin/c++     -std=gnu++11 -o CMakeFiles/hello_cpp11.dir/main.cpp.o -c /data/code/01-basic/L-cpp-standard/iii-compile-features/main.cpp
[100%] Linking CXX executable hello_cpp11
/usr/bin/cmake -E cmake_link_script CMakeFiles/hello_cpp11.dir/link.txt --verbose=1
/usr/bin/c++      CMakeFiles/hello_cpp11.dir/main.cpp.o  -o hello_cpp11 -rdynamic
make[2]: Leaving directory '/data/code/01-basic/L-cpp-standard/iii-compile-features/build'
[100%] Built target hello_cpp11
make[1]: Leaving directory '/data/code/01-basic/L-cpp-standard/iii-compile-features/build'
/usr/bin/cmake -E cmake_progress_start /data/code/01-basic/L-cpp-standard/
```



原文作者：[橘崽崽啊](https://www.cnblogs.com/juzaizai/)

原文链接：https://www.cnblogs.com/juzaizai/p/15069685.html