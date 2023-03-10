# 【NO.24】Makefile入门(超详细一文读懂)

## 1. [Makefile](https://link.zhihu.com/?target=https%3A//so.csdn.net/so/search%3Fq%3DMakefile%26spm%3D1001.2101.3001.7020)编译过程

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212122020445061817.png)

Makefile文件中的命令有一定规范，一旦该文件编写好以后在Linux命令行中执行一条make命令即可自动编译整个工程。不同厂家的make可能会稍有不同，并且语法上也有区别，不过基本思想都差不多，主要还是落在目标依赖上，最广泛使用的是GNUmake。

## 2. 语法规则

```
目标 ... : 依赖 ...    命令1    命令2    . . .
```

Makefile的核心规则，类似于一位厨神做菜，目标就是做好一道菜，那么所谓的依赖就是各种食材，各种厨具等等，然后需要厨师好的技术方法类似于命令，才能作出一道好菜。

同时这些依赖也有可能此时并不存在，需要现场制作，或者是由其他厨师做好，那么这个依赖就成为了其他规则的目标，该目标也会有他自己的依赖和命令。这样就形成了一层一层递归依赖组成了Makefile文件。

Makefile并不会关心命令是如何执行的，仅仅只是会去执行所有定义的命令，和我们平时直接输入命令行是一样的效果。

1、目标即要生成的文件。如果目标文件的更新时间晚于依赖文件更新时间，则说明依赖文件没有改动，目标文件不需要重新编译。否则会进行重新编译并更新目标文件。

2、默认情况下Makefile的第一个目标为终极目标。

3、依赖：即目标文件由哪些文件生成。

4、命令：即通过执行命令由依赖文件生成目标文件。注意每条命令之前必须有一个tab保持缩进，这是语法要求（会有一些编辑工具默认tab为4个空格，会造成Makefile语法错误）。

5、all：Makefile文件默认只生成第一个目标文件即完成编译，但是我们可以通过all 指定所需要生成的目标文件。

## 3. 变量

$符号表示取变量的值，当变量名多于一个字符时，使用”( )”
$符的其他用法

$^ 表示所有的依赖文件

$@ 表示生成的目标文件

$< 代表第一个依赖文件

```
SRC = $(wildcard *.c)OBJ = $(patsubst %.c, %.o, $(SRC))ALL: hello.outhello.out: $(OBJ)        gcc $< -o $@$(OBJ): $(SRC)        gcc -c $< -o $@
```

## 4. 变量赋值

1、”**=**“是最普通的等号，在Makefile中容易搞错赋值等号，使用 “=”进行赋值，变量的值是整个Makefile中最后被指定的值。

```
VIR_A = AVIR_B = $(VIR_A) BVIR_A = AA
```

经过上面的赋值后，最后VIR_B的值是AA B，而不是A B，在make时，会把整个Makefile展开，来决定变量的值

2、”**:=**“ 表示直接赋值，赋予当前位置的值。

```
VIR_A := AVIR_B := $(VIR_A) BVIR_A := AA
```

最后BIR_B的值是A B，即根据当前位置进行赋值。因此相当于“=”，“：=”才是真正意义上的直接赋值

3、”**?=**“ 表示如果该变量没有被赋值，赋值予等号后面的值。

```
VIR ?= new_value
```

如果VIR在之前没有被赋值，那么VIR的值就为new_value。

```
VIR := old_valueVIR ?= new_value
```

这种情况下，VIR的值就是old_value
4、”**+=**“和平时写代码的理解是一样的，表示将符号后面的值添加到前面的变量上

## 5. 预定义变量

CC：c编译器的名称，默认值为cc。cpp c预编译器的名称默认值为$(CC) -E

```
CC = gcc
```

回显问题，Makefile中的命令都会被打印出来。如果不想打印命令部分 可以使用@去除回显

```
@echo "clean done!"
```

## 6. 函数

通配符

```
SRC = $(wildcard ./*.c)
```

匹配目录下所有.c 文件，并将其赋值给SRC变量。

```
OBJ = $(patsubst %.c, %.o, $(SRC))
```

这个函数有三个参数，意思是取出SRC中的所有值，然后将.c 替换为.o 最后赋值给OBJ变量。

示例：如果目录下有很多个.c 源文件，就不需要写很多条规则语句了，而是可以像下面这样写

```
SRC = $(wildcard *.c)OBJ = $(patsubst %.c, %.o, $(SRC))ALL: hello.outhello.out: $(OBJ)        gcc $(OBJ) -o hello.out$(OBJ): $(SRC)        gcc -c $(SRC) -o $(OBJ)
```

这里先将所有.c 文件编译为 .o 文件，这样后面更改某个 .c 文件时，其他的 .c 文件将不在编译，而只是编译有更改的 .c 文件，可以大大提高大项目中的编译速度。

## 7. 伪目标 .PHONY

伪目标只是一个标签，clean是个伪目标没有依赖文件，只有用make来调用时才会执行

当目录下有与make 命令 同名的文件时 执行make 命令就会出现错误。

解决办法就是使用伪目标

```
SRC = $(wildcard *.c)OBJ = $(patsubst %.c, %.o, $(SRC))ALL: hello.outhello.out: $(OBJ)        gcc $< -o $@$(OBJ): $(SRC)        gcc -c $< -o $@clean:        rm -rf $(OBJ) hello.out.PHONY: clean ALL
```

通常也会把ALL设置成伪目标

## 8. 其他常用功能

代码清理clean
我们可以编译一条属于自己的clean语句，来清理make命令所产生的所有文件，列如

```
SRC = $(wildcard *.c)OBJ = $(patsubst %.c, %.o, $(SRC))ALL: hello.outhello.out: $(OBJ)        gcc $< -o $@$(OBJ): $(SRC)        gcc -c $< -o $@clean:        rm -rf $(OBJ) hello.out
```

## 9. 嵌套执行Makefile

在一些大工程中，会把不同模块或不同功能的源文件放在不同的目录中，我们可以在每个目录中都写一个该目录的Makefile这有利于让我们的Makefile变的更加简洁，不至于把所有东西全部写在一个Makefile中。

列如在子目录subdir目录下有个Makefile文件，来指明这个目录下文件的编译规则。外部总Makefile可以这样写

```
subsystem:            cd subdir && $(MAKE)其等价于：subsystem:            $(MAKE) -C subdir
```

定义$(MAKE)宏变量的意思是，也许我们的make需要一些参数，所以定义成一个变量比较有利于维护。两个例子意思都是先进入”subdir”目录，然后执行make命令

我们把这个Makefile叫做总控Makefile，总控Makefile的变量可以传递到下级的Makefile中，但是不会覆盖下层Makefile中所定义的变量，除非指定了 “-e”参数。

如果传递变量到下级Makefile中，那么可以使用这样的声明

export

如果不想让某些变量传递到下级Makefile，可以使用

unexport

```
export variable = value等价于variable = valueexport variable等价于export variable := value等价于variable := valueexport variable如果需要传递所有变量，那么只要一个export就行了。后面什么也不用跟，表示传递所有变量
```

## 10. 指定头文件路径

一般都是通过”**-I**“（大写i）来指定，假设头文件在：

```
/home/develop/include
```

则可以通过-I指定：

```
-I/home/develop/include
```

将该目录添加到头文件搜索路径中

在Makefile中则可以这样写：

```
CFLAGS=-I/home/develop/include
```

然后在编译的时候，引用CFLAGS即可，如下

```
yourapp:*.c    gcc $(CFLAGS) -o yourapp
```

## 11. 指定库文件路径

与上面指定头文件类似只不过使用的是”**-L**“来指定

```
LDFLAGS=-L/usr/lib -L/path/to/your/lib
```

告诉链接器要链接哪些库文件，使用”**-l**“（小写L）如下：

```
LIBS = -lpthread -liconv
```

## 12. 简单的Makefile实例

目录结构

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/12/20221212202218_25631.png)

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/12/20221212202219_80074.png)

### 12.1 nclude

myinclude.h

```
#include <stdio.h>void print1() ;void print2() ;
```

### 12.2 f1

f1.c

```
#include "../include/myinclude.h"                                                                              void print1()  {      printf("Message f1.c\n");      return;  } 
```

Makefile

目标前面的路径，意思是将目标生成到指定的目录下

```
../$(OBJS_DIR)/f1.o:f1.c                                                                                           @$(CC) -c $^ -o $@  
```

### 12.3 main

main.c

```
#include "../include/myinclude.h"                                                                                            int main(int argc, char const *argv[]){    print1();      print2();      return 0;}
```

Makefile

```
../$(OBJS_DIR)/main.o:main.c                                                                                       @$(CC) -c $^ -o $@  
```

### 12.4 obj

```
此目录用来存放相关生成的目标文件
```

Makefile

```
../$(BIN_DIR)/$(BIN) : $(OBJS)    @$(CC) $^ -o $@ 
```

### 12.5 主Makefile

```
#预定义变量CC = gcc#预定义编译目录SUBDIRS = f1 \        f2 \        main \        obj#预定义目标OBJS = f1.o f2.o main.oBIN = myappOBJS_DIR = objBIN_DIR = bin#传递预定义参数export CC OBJS BIN OBJS_DIR BIN_DIRall:CHECK_DIR $(SUBDIRS)CHECK_DIR:    @mkdir -p $(BIN_DIR)$(SUBDIRS):ECHO    @make -C $@ ECHO:    @echo $(SUBDIRS)    @echo begin compileclean:    @$(RM) $(OBJS_DIR)/*.o    @rm -rf $(BIN_DIR)
```

### 12.6 bin

此文件用来存放生成的二进制文件

### 12.7 执行结果

主Makefile执行过程

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/12/20221212202220_84792.jpg)

生成.o文件

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/12/20221212202220_23459.png)

生成二进制文件执行结果

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/12/20221212202221_69485.jpg)

————————————————

版权声明：本文为CSDN博主「晨曦艾米」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：[Makefile入门(超详细一文读懂)*晨曦艾米的博客-CSDN博客*makefile](https://link.zhihu.com/?target=https%3A//blog.csdn.net/ZBraveHeart/article/details/123187908)

原文作者：零声Github整理库