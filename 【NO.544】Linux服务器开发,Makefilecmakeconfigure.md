# 【NO.544】Linux服务器开发,Makefile/cmake/configure

## 1.Makefile语法

目前，互联网主要应用cmake，但是cmake又依赖于Makefile，所以，学些Makefile是有必要的。
Makefile的三要素：

- 目标：是执行的结果的感觉
- 依赖：是执行的条件的感觉
- 命令：是具体执行操作的感觉

### 1.1 工作原理



![在这里插入图片描述](https://img-blog.csdnimg.cn/f0e440fe0ddc43c3aea94872a0c8579b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_16,color_FFFFFF,t_70,g_se,x_16)



#### 1.1.1 范例1：规则展示

```c
all:test



        @echo "hello all"



test:



        @echo "hello test"
```

**值得注意的是：在@echo前，要用tab，千万不能使用空格，否则报错后果自负。**

#### 1.1.2 范例2：伪对象

```c
$(EXE)：$(OBJS)



        $(CC) -o $(EXE) $(OBJS)
.PHONY:main clean



CC=gcc



RM=rm



EXE=simple



OBJS=main.o foo.o



$(EXE)=$(OBJS)



    $(CC) -o $(EXE) $(OBJS)



simple:main foo.o    



    @echo "gcc -o simple main.o foo.o"



    gcc -o simple main.o foo.o



main:main.c



    @echo "gcc -o main.o -c main.c"



    gcc -o main.o -c main.c



foo.o:foo.c



    @echo "gcc -o foo.o -c foo.c"



    fcc -o foo.o -c foo.c



clean:



    rm simple main.o foo.o
```

- 伪对象: .PHONY 后面加上原本有的文件名字，这样就会防止执行make时重名报错。解决提示”make 'clean' is up to date“的问题。
- =左面得符号名字用等号右面进行替换，gcc -o A B要遵循从左至右得原则。

#### 1.1.3 范例3：自动变量

```c
.PHONY:all



all:first second third



        @echo "\$$@ = $@"



        @echo "$$^ =$^"



        @echo "$$< =$<"



first:



        @echo "1 first"



second: 



        @echo "2 second"



third:



        @echo "3 third"
```

- $@ 表示一个规则中的目标。当我们的一个规则中有多个目标时，$@所指的是其中**任何造成命令被运行的目标**。
- $^ 表示规则中的所有先决条件。
- $< 表示规则中的第一个先决条件。
  个人理解$@指的是例子中$(EXE) $(OBJS),$<表示的main.o foo.o，$<表示的是main.c或是foo.c。

#### 1.1.4 范例4：相关函数

```c
。PHONY:clean



CC=gcc



RM=rm



EXE=simple



SRCS=$(wildcard *.c)



OBJS=$(patsubst %.c,%.o,$(SRCS))



$(EXE):$(OBJS)



        $(CC) -o $@ $^



%.o:%.c



        $(CC) -o $@ -c $^



clean:



$(RM) $(EXE) $(OBJS)



src:#调试make src 显示相应的xx.c



        @echo $(SRCS)



objs:#测试make objs显示相应的xx.o



        @echo $(OBJS)        
```

- $(wildcard *.c)表示通配出所有 .c 的文件。
- $(patsubst pattern,replacement,text)表示模式字符串替换函数。

## 2.CMake语法

### 2.1 范例一：规则展示



![在这里插入图片描述](https://img-blog.csdnimg.cn/abd5ddc45842418dab3af212a98c25fe.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16)



```c
#CMakeLists.txt文件



#单个目录实现



#CMake 最低版本号要求



cmake_minimum_required(version 2.8)



#工程，它不是执行文件名



PROJECT(0VOICE)



#手动加入文件



SET(SRC_LIST main.c)



SET(SRC_LIST2 main2.c)



MESSAGE(STATUS "THIS IS BINARY DIR " ${PROJECT_BINARY_DIR})



MESSAGE(STATUS "THIS IS SOURCE DIR" ${PROJECT_SOURCE_DIR})



 



#生成执行文件名0voice 0voice2



ADD_EXECUTABLE(0voice $(SRC_LIST))



ADD_EXECUTABLE(0voice2 ${SRC_LIST2})
```

### 2.2 范例二：层次结构



![在这里插入图片描述](https://img-blog.csdnimg.cn/0758cac7fe8c4908b51c15651f7785a7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_9,color_FFFFFF,t_70,g_se,x_16)



```c
#CMake 最低版本号要求



cmake_minium_required (VERSION 2.8)



PROJECT(0VOICE)



#添加子目录



ADD_SUBDIRECTORY(src)



#INSTALL(FILES COPYRIGHT README DESTINATION share/doc/cmake/ovoice)



#安装doc到 share/doc/cmake/ovoice 目录



#默认/usr/local/



#指定自定义目录，比如cmake -DCMAKE_INSTALL_PREFIX=/tm[/usr



INSTALL(DIRECTORY doc/ DESTINATION share/doc/cmake/ovoice)
```

- 安装目录到某个路径，终端输入

  cmake -DCMAKE_INSTALL_PREFIX=/tmp.usr ..

  #### 范例三：编译库

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/b24f7e4e90994c83807e674351829e73.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_9,color_FFFFFF,t_70,g_se,x_16)

```c
#单个目录实现



#CMake 最低版本号要求



cmake_mininun_required (VERSION 2.8)



#工程



PROJECT(0VOICE)



#手动加入文件



SET(SRC_LIST main.c)



MESSAGE(STATUS "THIS IS BINARY DIR " ${PROJECT_BINARY_DIR})



MESSAGE(STATUS "THIS IS SOURCE DIR" ${PROJECT_SOURCE_DIR})



 



#添加头文件路径



INCLUDE_DIRECTORIES("${CMAKE_CURRENT_SOURCE_DIR}/dir1")



INCLUDE_DIRECTORIES(dir1)



MESSAGE(STATUS "CMAKE_CURRENT_SOURCE_DIR ->" ${CMAKE_CURRENT_SOURCE_DIR})



#添加dur1子目录



ADD_SUBDIRETORY("${CMAKE_CURRENT_SOURCE_DIR}/dir1")



#添加头文件路径



INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR}/dir1)



#添加dir2子目录



ADD_SUBDIRECTORY("${CMAKE_CURRENT_SOURCE_DIR}"/dir2)



ADD_EXECUTABLE(darren ${SRC_LIST})



TARGET_LINK_LIBRARIES(darren dir1 dir2)
#dir1 的Makelist



#加载所有的源码，和makefile wildcard类似AUX_SOURCE_DIRECTORY(. DIR_SRCS)



# SET(DIR_SRCS dir1.c dir12.c)



#默认是静态库



ADD_LIBRARY (dir1 SHARED ${DIR_SRCS})



+ 如果静态库和动态库同时存在，会优先连接动态库。
#引用动态库



ADD_EXECUTABLE(darren ${SRC_LIST})



#TARGET_LINK_LIBRARIES(darren Dir1)



#引用静态库



#TARGET_LINK_LIBRARIES(darren libDira1,1)



#TARGET_LINK_LIBRARIES(darren lDir1)
```

### 2.3 范例四：省去子目录MakeList文件

```c
#单个目录实现



#CMake 最低版本号要求



cmake_mininun_required (VERSION 2.8)



#工程



PROJECT(0VOICE)



#手动加入文件



SET(SRC_LIST main.c)



MESSAGE(STATUS "THIS IS BINARY DIR " ${PROJECT_BINARY_DIR})



MESSAGE(STATUS "THIS IS SOURCE DIR" ${PROJECT_SOURCE_DIR})



#设置子目录



set(SUB_DIR_LIST "${CMAKE_CURRENT_SOURCE_DIR}/dir1" "${CMAKE_CURRENT_SOURCE_DIR}/dir2")



foreach(SUB_DIR ${sSUB_DIR_LIST})



#遍历源文件



aux_source_direatory(${SUB_DIR} SRC_LIST)



MESSAGE(STATUS "SUB_DIR->" ${SUB_DIR})



MESSAGE(STATUS "SUB_DIR->" ${SUB_DIR})



ebdfireach()



#添加头文件路径



INCLUDE_DIRECTORIES("dir1")



INCLUDE_DIRECTORIES("dir2")



ADD_EXECUTABLE(darren ${SRC_LIST})
```

### 2.4 范例五：设置版本

```c
cmake_minimum_requires(VERSION 2.8s)



#设置release版本还是debug版本



if(${CMAKE_BUILD_TYPE} MATCHES "Release")



        MESSAGE(STATUS "Release版本")



        SET(BuildType，”Release“)



else()



        SET(BuildType "Debug")



        MESSAGE(STATUS "Debug版本")



endif()



#设置lib库目录



SET(RELEASE_DIR ${PROJECT_SOURCE_DIR}/release)



#debug和release版本目录不一样



#设置生成的so动态库组后输出的路径



SET(LIBRARY_OUTPUT_PATH ${RELEASE_DIR}/linux/${BuildType})



ADD_COMPILE_OPTIONS(-fPIC)



 



#查找当前目录下所有与源文件并保存到DIR_LIB_SRCS变量



AUX_SOURCE_DIRECTORY(. DIR_LIB_SRCS)        
```

- CMakeList 是可以传递的。

原文作者：[[屯门山鸡叫我小鸡](https://blog.csdn.net/sinat_28294665)

原文链接：https://bbs.csdn.net/topics/604939855