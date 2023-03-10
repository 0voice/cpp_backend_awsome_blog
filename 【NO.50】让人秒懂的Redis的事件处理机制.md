# 【NO.50】让人秒懂的Redis的事件处理机制

redis是单进程，单线程模型，与nginx的多进程不同，与golang的多协程也不同，“工作的工人”那么少，可那么为什么redis能这么快呢？

## 1.**epoll多路复用**

这里重点要说的就是redis的IO编程模型，首先了解下

为什么要有多路复用呢？

如果没有多路复用，一个线程只能监听一个端口的一个连接，这样这个效率比较低。当然我们有几种办法可以破除这个，一个是使用多线程模型，我们还是监听一个端口，但是一个请求进来，我们为其创建一个线程。但是这种消耗是比较大的。所以我们一直想办法，有没有办法一个线程监听多个端口，或者多个一个端口的多个连接（fd）。

这里再说说fd， 文件描述符（file descriptor）是内核为了高效管理已被打开的文件所创建的索引，其是一个非负整数（通常是小整数），用于指代被打开的文件，所有执行I/O操作(包括网络socket操作)的系统调用都通过文件描述符。每个连接请求上来，都会创建一个连接套接字，一个连接使用一个连接套接字。

对于监听端口，我们会有一个监听套接字，对应监听fd。我们所有的监听业务都是从监听这个套接字开始的。

那么如果我一个程序能同时监听多个连接套接字，是不是就很赞了。是的，这就是linux的io多路复用逻辑。但是这么多连接套接字，传递数据等是断断续续的，A连接接收一个包，B连接再接收一个包，A连接再接收一个包，B连接再接收一个包....如果我等着A连接把包都接收完再处理B，那效率是非常慢的。所以，这里我们就需要有一个通知机制，让有收到包的时候通知下处理线程。

linux的IO多路复用逻辑主要有三种：select, poll, epoll。

### **1.1select**

select模型监听的三个事件：读数据事件，写数据事件，异常事件。

使用select模型的步骤如下：

- 我们确定要监听的监听fd列表

- 调用select监听所有监听fd，阻塞线程。

- select只有当有事件出现并且有事件的fd已经等待完毕

- 如果是创建一个连接事件：

- - 创建一个连接套接字，连接fd
  - 将连接fd和监听fd集合放在一起



- 如果是一个读写事件：

- - 遍历所有fd，判断是否是准备好的fd
  - 如果是准备好的fd，进行业务读写逻辑



- 循环进入select。

select一次可以监听1024个文件描述符。

### 1.2**poll模型**

poll传递给内核的是：

- 监听的fd集合
- 需要监听的事件类型
- 实际发生的事件类型

poll的模型逻辑是：

- 我们确定要监听的监听fd列表

- 调用poll监听所有监听fd，阻塞线程。

- poll只有当有事件出现才解除阻塞

- 如果是创建一个连接事件：

- - 创建一个连接套接字，连接fd
  - 将连接fd和监听fd集合放在一起



- 如果是一个读写事件：

- - 遍历所有fd，判断是否是有读写事件的fd
  - 如果fd有读写事件，进行业务读写逻辑



- 循环进入poll。

poll比select优秀在它没有了1024的限制了。但是还是有一些缺陷，就是必须要遍历所有fd。

### 1.3**epoll**

epoll的数据结构类似poll，但是在调用epoll的时候，它不是返回发生了事件的fd个数，而是返回了所有发生的事件，这个事件中可以查出发生事件的fd。

所以epoll的逻辑模型是：

- 我们确定要监听的监听fd列表

- 调用epoll监听所有监听fd，阻塞线程。

- epoll只有当有事件出现才解除阻塞，并且返回事件列表

- 遍历事件列表：

- 如果是创建一个连接事件：

- - 创建一个连接套接字，连接fd
  - 将连接fd和监听fd集合放在一起，继续epoll



- 如果是一个读写事件：

- - 处理这个事件



- 循环进入epoll。

说白了，epoll就是我们逻辑上能想到的最优的通知机制。一群人去排队，有多个事件发生，警察来了，那么就告诉警察有哪几个列发生了什么事件，警察一个个处理就行了。

## 2.**Reactor模型**

有了IO多路复用的机制，我们就可以实现一种模型，叫做Reactor模型了。

最经典的一张图就是这个

![img](https://pic1.zhimg.com/80/v2-620435931dcba05592e271fc38df11e8_720w.webp)

reactor的五大角色：

- Handle（句柄或者是描述符）
- Synchronous Event Demultiplexer(同步事件分离器)
- Event Handler(事件处理器)
- Concrete Event Handler(具体事件处理器)
- Initiation Dispatcher(初始分发器)

简要来说，Reactor就是我们现在最正常理解的“事件驱动”，对，就是字面理解的那种。比如订阅一个kafka，我们会创建一个监听程序，监听kafka的某个topic，然后在监听程序中挂载几个处理不同消息的处理程序，每当有一个事件从topic进入的时候，我们就会有通过这个监听程序，通知我们的处理程序。处理程序来处理不同的消息。

这种所谓的通知机制，就叫做reactor。

这个kafka的例子，里面有一个监听事件的程序，它一定是一个同步的，一条消息来了，投递一个消息，就叫做 Synchronous Event Demultiplexer(同步事件分离器)。而这个消息，就是Handle（句柄或者是描述符）。我们需要将某个具体的事件处理函数，也就是上图的Concrete Event Handler(具体事件处理器) 挂载到监听的处理程序中。当然这里的每个Concrete Event Handler(具体事件处理器) 都必须遵照某种格式，比如定义了handle_event和get_handle接口。这种格式我们统称为Event Handler(事件处理器)。再回到监听事件，监听事件一定有一个挂载的具体map之类的结构，即哪个事件对应哪个处理程序，这个挂载的核心我们叫它Initiation Dispatcher(初始分发器)。

标准的处理流程描述如下：

- 当应用向Initiation Dispatcher注册Concrete Event Handler时，应用会标识出该事件处理器希望Initiation Dispatcher在某种类型的事件发生发生时向其通知，事件与handle关联
- Initiation Dispatcher要求注册在其上面的Concrete Event Handler传递内部关联的handle，该handle会向操作系统标识
- 当所有的Concrete Event Handler都注册到 Initiation Dispatcher上后，应用会调用handle_events方法来启动Initiation Dispatcher的事件循环，这时Initiation Dispatcher会将每个Concrete Event Handler关联的handle合并，并使用Synchronous Event Demultiplexer来等待这些handle上事件的发生
- 当与某个事件源对应的handle变为ready时，Synchronous Event Demultiplexer便会通知 Initiation Dispatcher。比如tcp的socket变为ready for reading
- Initiation Dispatcher会触发事件处理器的回调方法。当事件发生时， Initiation Dispatcher会将被一个“key”（表示一个激活的handle）定位和分发给特定的Event Handler的回调方法
- Initiation Dispatcher调用特定的Concrete Event Handler的回调方法来响应其关联的handle上发生的事件

在这五种角色中，

其中的 Initiation Dispatcher(初始分发器) 是最重要的，我们也称其为Reactor。它本身定义了一些规范，同时提供了Handler的一些注册机制。

而Synchronous Event Demultiplexer(同步事件分离器) 在IO场景下，一般是由操作系统底层实现的，就是说操作系统底层必须能有这个能力，才能基于这个能力实现Reactor模型。在我们这个场景下，就是前面提到的linux的多路复用机制。

Handle（句柄或者是描述符）在IO场景下就是IO网络连接的fd。

而Event Handler(事件处理器) 和 Concrete Event Handler(具体事件处理器) 在IO场景下分为三种处理事件：连接事件，写事件，读事件。对于连接事件的处理器，我们称之为acceptor，读/写事件的处理器，我们统称为handler。

所以在IO场景下，Reactor 我们需要实现的三个关键角色为：reactor、acceptor、handler。

## 3.**Redis的实现**

在redis中，下面一张图就能说明其实现逻辑。

![img](https://pic1.zhimg.com/80/v2-1ced02d894a1107c68e301a7e18da870_720w.webp)

在redis中，有个reactor（叫做aeMain）接收客户端的redis请求。而在这个reactor中除了监听连接事件acceptor之外，还可以动态注册各种handler （aeCreateFileEvent）。当一个客户端请求进入的时候，调用 aeProcessEvents 来分发事件。

这个逻辑就很清晰了吧。整个就是redis的事件处理机制。

原文链接：https://zhuanlan.zhihu.com/p/486698266

原文作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)