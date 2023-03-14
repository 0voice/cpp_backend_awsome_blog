# 【NO.277】CMake基础 第15节 使用Protobuf生成源文件

## 1.介绍

这个例子展示了如何使用Protobuf生成源文件。Protocol Buffers是Google提供的一种数据序列化格式。用户提供带有数据描述的.proto文件。然后使用Protobuf编译器，可以将该原始文件翻译成包括C++在内的多种语言的源代码。

本教程中的文件如下：

```objectivec
$ tree
.
├── AddressBook.proto
├── CMakeLists.txt
├── main.cpp
```

- [AddressBook.proto] - 来自main protocol buffer示例的proto文件

  ```protobuf
  package tutorial;
  
  message Person {
    required string name = 1;
    required int32 id = 2;
    optional string email = 3;
  
    enum PhoneType {
      MOBILE = 0;
      HOME = 1;
      WORK = 2;
    }
  
    message PhoneNumber {
      required string number = 1;
      optional PhoneType type = 2 [default = HOME];
    }
  
    repeated PhoneNumber phone = 4;
  }
  
  message AddressBook {
    repeated Person person = 1;
  }
  ```

- [CMakeLists.txt] - 包含要运行的CMake命令

  ```cmake
  cmake_minimum_required(VERSION 3.5)
  
  # Set the project name
  project (protobuf_example)
  
  # find the protobuf compiler and libraries
  find_package(Protobuf REQUIRED)
  
  # check if protobuf was found
  if(PROTOBUF_FOUND)
      message ("protobuf found")
  else()
      message (FATAL_ERROR "Cannot find Protobuf")
  endif()
  
  # Generate the .h and .cxx files
  PROTOBUF_GENERATE_CPP(PROTO_SRCS PROTO_HDRS AddressBook.proto)
  
  # Print path to generated files
  message ("PROTO_SRCS = ${PROTO_SRCS}")
  message ("PROTO_HDRS = ${PROTO_HDRS}")
  
  # Add an executable
  add_executable(protobuf_example
      main.cpp
      ${PROTO_SRCS}
      ${PROTO_HDRS})
  
  target_include_directories(protobuf_example
      PUBLIC
      ${PROTOBUF_INCLUDE_DIRS}
      ${CMAKE_CURRENT_BINARY_DIR}
  )
  
  # link the exe against the libraries
  target_link_libraries(protobuf_example
      PUBLIC
      ${PROTOBUF_LIBRARIES}
  )
  ```

- [main.cpp] - protobuf示例的源文件.

  ```cpp
  #include <iostream>
  #include <fstream>
  #include <string>
  #include "AddressBook.pb.h"
  using namespace std;
  
  // This function fills in a Person message based on user input.
  void PromptForAddress(tutorial::Person* person) {
    cout << "Enter person ID number: ";
    int id;
    cin >> id;
    person->set_id(id);
    cin.ignore(256, '\n');
  
    cout << "Enter name: ";
    getline(cin, *person->mutable_name());
  
    cout << "Enter email address (blank for none): ";
    string email;
    getline(cin, email);
    if (!email.empty()) {
      person->set_email(email);
    }
  
    while (true) {
      cout << "Enter a phone number (or leave blank to finish): ";
      string number;
      getline(cin, number);
      if (number.empty()) {
        break;
      }
  
      tutorial::Person::PhoneNumber* phone_number = person->add_phone();
      phone_number->set_number(number);
  
      cout << "Is this a mobile, home, or work phone? ";
      string type;
      getline(cin, type);
      if (type == "mobile") {
        phone_number->set_type(tutorial::Person::MOBILE);
      } else if (type == "home") {
        phone_number->set_type(tutorial::Person::HOME);
      } else if (type == "work") {
        phone_number->set_type(tutorial::Person::WORK);
      } else {
        cout << "Unknown phone type.  Using default." << endl;
      }
    }
  }
  
  // Main function:  Reads the entire address book from a file,
  //   adds one person based on user input, then writes it back out to the same
  //   file.
  int main(int argc, char* argv[]) {
    // Verify that the version of the library that we linked against is
    // compatible with the version of the headers we compiled against.
    GOOGLE_PROTOBUF_VERIFY_VERSION;
  
    if (argc != 2) {
      cerr << "Usage:  " << argv[0] << " ADDRESS_BOOK_FILE" << endl;
      return -1;
    }
  
    tutorial::AddressBook address_book;
  
    {
      // Read the existing address book.
      fstream input(argv[1], ios::in | ios::binary);
      if (!input) {
        cout << argv[1] << ": File not found.  Creating a new file." << endl;
      } else if (!address_book.ParseFromIstream(&input)) {
        cerr << "Failed to parse address book." << endl;
        return -1;
      }
    }
  
    // Add an address.
    PromptForAddress(address_book.add_person());
  
    {
      // Write the new address book back to disk.
      fstream output(argv[1], ios::out | ios::trunc | ios::binary);
      if (!address_book.SerializeToOstream(&output)) {
        cerr << "Failed to write address book." << endl;
        return -1;
      }
    }
  
    // Optional:  Delete all global objects allocated by libprotobuf.
    google::protobuf::ShutdownProtobufLibrary();
  
    return 0;
  }
  ```

## 2.要求

此示例需要安装protocol buffers二进制文件和库。可以使用以下命令将其安装在Ubuntu上。

```shell
sudo apt-get install protobuf-compiler libprotobuf-dev
```

## 3.概念

### 3.1 导出变量

由CMake Protobuf包导出并在此示例中使用的变量包括:

- `PROTOBUF_FOUND` - 如果安装了Protocol Buffers
- `PROTOBUF_INCLUDE_DIRS` - protobuf的头文件
- `PROTOBUF_LIBRARIES` - protobuf库

此外，还可以通过查看FindProtobuf.cmake文件顶部的内容找到定义的更多变量。

### 3.2 生成源代码

Protobuf CMake包包含许多帮助函数，以简化代码生成。在本例中，我们生成的是C++源代码，使用以下代码：

```cmake
PROTOBUF_GENERATE_CPP(PROTO_SRCS PROTO_HDRS AddressBook.proto)
```

这些参数包括:

- PROTO_SRCS - 存储.pb.cc文件的变量名称
- PROTO_HDRS- 存储.pb.h文件的变量名称
- AddressBook.proto - 从中生成代码的.proto文件

### 3.3 生成文件

在调用`PROTOBUF_GENERATE_CPP`函数之后，你将拥有上面提到的变量。这些将被标记为自定义命令的输出，该命令将调用Protobuf编译器来生成它们。

要生成这些文件，你应该将它们添加到库或可执行文件中。

例如：

```cmake
add_executable(protobuf_example
    main.cpp
    ${PROTO_SRCS}
    ${PROTO_HDRS})
```

这将导致在对该可执行文件目标调用make时调用protocol buf编译器。

对.proto文件进行更改后，将再次自动生成关联的源文件。但是，如果没有对.proto文件进行任何更改，并且你重新运行make，则不会执行任何操作。

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
-- Looking for include file pthread.h
-- Looking for include file pthread.h - found
-- Looking for pthread_create
-- Looking for pthread_create - not found
-- Looking for pthread_create in pthreads
-- Looking for pthread_create in pthreads - not found
-- Looking for pthread_create in pthread
-- Looking for pthread_create in pthread - found
-- Found Threads: TRUE
-- Found PROTOBUF: /usr/lib/x86_64-linux-gnu/libprotobuf.so
protobuf found
PROTO_SRCS = /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/AddressBook.pb.cc
PROTO_HDRS = /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/AddressBook.pb.h
-- Configuring done
-- Generating done
-- Build files have been written to: /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build

$ ls
CMakeCache.txt  CMakeFiles  cmake_install.cmake  Makefile

$ make VERBOSE=1
/usr/bin/cmake -H/home/matrim/workspace/cmake-examples/03-code-generation/protobuf -B/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build --check-build-system CMakeFiles/Makefile.cmake 0
/usr/bin/cmake -E cmake_progress_start /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles/progress.marks
make -f CMakeFiles/Makefile2 all
make[1]: Entering directory `/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build'
make -f CMakeFiles/protobuf_example.dir/build.make CMakeFiles/protobuf_example.dir/depend
make[2]: Entering directory `/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build'
/usr/bin/cmake -E cmake_progress_report /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles 1
[ 33%] Running C++ protocol buffer compiler on AddressBook.proto
/usr/bin/protoc --cpp_out /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build -I /home/matrim/workspace/cmake-examples/03-code-generation/protobuf /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/AddressBook.proto
cd /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build && /usr/bin/cmake -E cmake_depends "Unix Makefiles" /home/matrim/workspace/cmake-examples/03-code-generation/protobuf /home/matrim/workspace/cmake-examples/03-code-generation/protobuf /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles/protobuf_example.dir/DependInfo.cmake --color=
Dependee "/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles/protobuf_example.dir/DependInfo.cmake" is newer than depender "/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles/protobuf_example.dir/depend.internal".
Dependee "/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles/CMakeDirectoryInformation.cmake" is newer than depender "/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles/protobuf_example.dir/depend.internal".
Scanning dependencies of target protobuf_example
make[2]: Leaving directory `/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build'
make -f CMakeFiles/protobuf_example.dir/build.make CMakeFiles/protobuf_example.dir/build
make[2]: Entering directory `/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build'
/usr/bin/cmake -E cmake_progress_report /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles 2
[ 66%] Building CXX object CMakeFiles/protobuf_example.dir/main.cpp.o
/usr/bin/c++    -I/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build    -o CMakeFiles/protobuf_example.dir/main.cpp.o -c /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/main.cpp
/usr/bin/cmake -E cmake_progress_report /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles 3
[100%] Building CXX object CMakeFiles/protobuf_example.dir/AddressBook.pb.cc.o
/usr/bin/c++    -I/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build    -o CMakeFiles/protobuf_example.dir/AddressBook.pb.cc.o -c /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/AddressBook.pb.cc
Linking CXX executable protobuf_example
/usr/bin/cmake -E cmake_link_script CMakeFiles/protobuf_example.dir/link.txt --verbose=1
/usr/bin/c++       CMakeFiles/protobuf_example.dir/main.cpp.o CMakeFiles/protobuf_example.dir/AddressBook.pb.cc.o  -o protobuf_example -rdynamic -lprotobuf -lpthread
make[2]: Leaving directory `/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build'
/usr/bin/cmake -E cmake_progress_report /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles  1 2 3
[100%] Built target protobuf_example
make[1]: Leaving directory `/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build'
/usr/bin/cmake -E cmake_progress_start /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles 0
$ make VERBOSE=1
/usr/bin/cmake -H/home/matrim/workspace/cmake-examples/03-code-generation/protobuf -B/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build --check-build-system CMakeFiles/Makefile.cmake 0
/usr/bin/cmake -E cmake_progress_start /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles/progress.marks
make -f CMakeFiles/Makefile2 all
make[1]: Entering directory `/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build'
make -f CMakeFiles/protobuf_example.dir/build.make CMakeFiles/protobuf_example.dir/depend
make[2]: Entering directory `/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build'
/usr/bin/cmake -E cmake_progress_report /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles 1
[ 33%] Running C++ protocol buffer compiler on AddressBook.proto
/usr/bin/protoc --cpp_out /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build -I /home/matrim/workspace/cmake-examples/03-code-generation/protobuf /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/AddressBook.proto
cd /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build && /usr/bin/cmake -E cmake_depends "Unix Makefiles" /home/matrim/workspace/cmake-examples/03-code-generation/protobuf /home/matrim/workspace/cmake-examples/03-code-generation/protobuf /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles/protobuf_example.dir/DependInfo.cmake --color=
Dependee "/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles/protobuf_example.dir/DependInfo.cmake" is newer than depender "/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles/protobuf_example.dir/depend.internal".
Dependee "/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles/CMakeDirectoryInformation.cmake" is newer than depender "/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles/protobuf_example.dir/depend.internal".
Scanning dependencies of target protobuf_example
make[2]: Leaving directory `/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build'
make -f CMakeFiles/protobuf_example.dir/build.make CMakeFiles/protobuf_example.dir/build
make[2]: Entering directory `/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build'
/usr/bin/cmake -E cmake_progress_report /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles 2
[ 66%] Building CXX object CMakeFiles/protobuf_example.dir/main.cpp.o
/usr/bin/c++    -I/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build    -o CMakeFiles/protobuf_example.dir/main.cpp.o -c /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/main.cpp
/usr/bin/cmake -E cmake_progress_report /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles 3
[100%] Building CXX object CMakeFiles/protobuf_example.dir/AddressBook.pb.cc.o
/usr/bin/c++    -I/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build    -o CMakeFiles/protobuf_example.dir/AddressBook.pb.cc.o -c /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/AddressBook.pb.cc
Linking CXX executable protobuf_example
/usr/bin/cmake -E cmake_link_script CMakeFiles/protobuf_example.dir/link.txt --verbose=1
/usr/bin/c++       CMakeFiles/protobuf_example.dir/main.cpp.o CMakeFiles/protobuf_example.dir/AddressBook.pb.cc.o  -o protobuf_example -rdynamic -lprotobuf -lpthread
make[2]: Leaving directory `/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build'
/usr/bin/cmake -E cmake_progress_report /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles  1 2 3
[100%] Built target protobuf_example
make[1]: Leaving directory `/home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build'
/usr/bin/cmake -E cmake_progress_start /home/matrim/workspace/cmake-examples/03-code-generation/protobuf/build/CMakeFiles 0

$ ls
AddressBook.pb.cc  CMakeCache.txt  cmake_install.cmake  protobuf_example
AddressBook.pb.h   CMakeFiles      Makefile

$ ./protobuf_example test.db
test.db: File not found.  Creating a new file.
Enter person ID number: 11
Enter name: John Doe
Enter email address (blank for none): wolly@sheep.ie
Enter a phone number (or leave blank to finish):

$ ls
AddressBook.pb.cc  CMakeCache.txt  cmake_install.cmake  protobuf_example
AddressBook.pb.h   CMakeFiles      Makefile             test.db
```

原文作者：[橘崽崽啊](https://www.cnblogs.com/juzaizai/)

原文链接：https://www.cnblogs.com/juzaizai/p/15069714.html