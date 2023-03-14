# 【NO.281】五种网络IO模型详解

## 1. IO操作本质

数据复制的过程中不会消耗CPU

\# 1 内存分为内核缓冲区和用户缓冲区

\# 2 用户的应用程序不能直接操作内核缓冲区，需要将数据从内核拷贝到用户才能使用

\# 3 而IO操作、网络请求加载到内存的数据一开始是放在内核缓冲区的

![img](https://pic3.zhimg.com/80/v2-767a42e4fc58f1543acb005db92dcd3a_720w.webp)

## 2.IO模型

### 2.1. BIO – 阻塞模式I/O

用户进程从发起请求，到最终拿到数据前，一直挂起等待； 数据会由用户进程完成拷贝

```
'''
举个例子：一个人去 商店买一把菜刀，
他到商店问老板有没有菜刀（发起系统调用）
如果有（表示在内核缓冲区有需要的数据）
老板直接把菜刀给买家（从内核缓冲区拷贝到用户缓冲区）
这个过程买家一直在等待
如果没有，商店老板会向工厂下订单（IO操作，等待数据准备好）
工厂把菜刀运给老板（进入到内核缓冲区）
老板把菜刀给买家（从内核缓冲区拷贝到用户缓冲区）
这个过程买家一直在等待
是同步io
'''
```

![img](https://pic4.zhimg.com/80/v2-dd42dccb828c389e1b7bf0b77bd968e7_720w.webp)

```
import socket
server = socket.socket()
server.bind(('127.0.0.1',8080))
server.listen(5)
while True:
    conn, addr = server.accept()
    while True:
        try:
            data = conn.recv(1024)
            if len(data) == 0:break
            print(data)
            conn.send(data.upper())
        except ConnectionResetError as e:
            break
    conn.close()
# 在服务端开设多进程或者多线程 进程池线程池 其实还是没有解决IO问题    
该等的地方还是得等 没有规避
只不过多个人等待的彼此互不干扰
```

### 2.2. NIO – 非阻塞模式I/O

用户进程发起请求，如果数据没有准备好，那么立刻告知用户进程未准备好；此时用户进程可选择继续发起请求、或者先去做其他事情，稍后再回来继续发请求，直到被告知数据准备完毕，可以开始接收为止； 数据会由用户进程完成拷贝

```
'''
举个例子：一个人去 商店买一把菜刀，
他到商店问老板有没有菜刀（发起系统调用）
老板说没有，在向工厂进货（返回状态）
买家去别地方玩了会，又回来问，菜刀到了么（发起系统调用）
老板说还没有（返回状态）
买家又去玩了会（不断轮询）
最后一次再问，菜刀有了（数据准备好了）
老板把菜刀递给买家（从内核缓冲区拷贝到用户缓冲区）
整个过程轮询+等待：轮询时没有等待，可以做其他事，从内核缓冲区拷贝到用户缓冲区需要等待
是同步io
同一个线程，同一时刻只能监听一个socket，造成浪费，引入io多路复用，同时监听读个socket
'''
```

![img](https://pic3.zhimg.com/80/v2-9b42ac950d82df33a70b5f73d225af9e_720w.webp)

```
"""
要自己实现一个非阻塞IO模型
"""
import socket
import time
server = socket.socket()
server.bind(('127.0.0.1', 8081))
server.listen(5)
server.setblocking(False)
# 将所有的网络阻塞变为非阻塞
r_list = []
del_list = []
while True:
    try:
        conn, addr = server.accept()
        r_list.append(conn)
    except BlockingIOError:
        # time.sleep(0.1)
        # print('列表的长度:',len(r_list))
        # print('做其他事')
        for conn in r_list:
            try:
                data = conn.recv(1024)  # 没有消息 报错
                if len(data) == 0:  # 客户端断开链接
                    conn.close()  # 关闭conn
                    # 将无用的conn从r_list删除
                    del_list.append(conn)
                    continue
                conn.send(data.upper())
            except BlockingIOError:
                continue
            except ConnectionResetError:
                conn.close()
                del_list.append(conn)
        # 挥手无用的链接
        for conn in del_list:
            r_list.remove(conn)
        del_list.clear()
# 客户端
import socket
client = socket.socket()
client.connect(('127.0.0.1',8081))
while True:
    client.send(b'hello world')
    data = client.recv(1024)
    print(data)
```

### 2.3. IO Multiplexing - I/O多路复用模型

类似BIO，只不过找了一个代理，来挂起等待，并能同时监听多个请求； 数据会由用户进程完成拷贝

```
'''
举个例子：多个人去 一个商店买菜刀，
多个人给老板打电话，说我要买菜刀（发起系统调用）
老板把每个人都记录下来（放到select中）
老板去工厂进货（IO操作）
有货了，再挨个通知买到的人，来取刀（通知/返回可读条件）
买家来到商店等待，老板把到给买家（从内核缓冲区拷贝到用户缓冲区）
多路复用：老板可以同时接受很多请求（select模型最大1024个，epoll模型），
但是老板把到给买家这个过程，还需要等待，
是同步io
强调：
 1. 如果处理的连接数不是很高的话，使用select/epoll的web server不一定比使用multi-threading + blocking IO的web server性能更好，可能延迟还更大。select/epoll的优势并不是对于单个连接能处理得更快，而是在于能处理更多的连接。
 2. 在多路复用模型中，对于每一个socket，一般都设置成为non-blocking，但是，如上图所示，整个用户的process其实是一直被block的。只不过process是被select这个函数block，而不是被socket IO给block。
 结论: select的优势在于可以处理多个连接，不适用于单个连接
'''
```

![img](https://pic4.zhimg.com/80/v2-ddb74c49bab40321c8058b72f1513203_720w.webp)

```
"""
当监管的对象只有一个的时候 其实IO多路复用连阻塞IO都比比不上！！！
但是IO多路复用可以一次性监管很多个对象
server = socket.socket()
conn,addr = server.accept()
监管机制是操作系统本身就有的 如果你想要用该监管机制(select)
需要你导入对应的select模块
"""
import socket
import select
server = socket.socket()
server.bind(('127.0.0.1',8080))
server.listen(5)
server.setblocking(False)
read_list = [server]
while True:
    r_list, w_list, x_list = select.select(read_list, [], [])
    """
    帮你监管
    一旦有人来了 立刻给你返回对应的监管对象
    """
    # print(res)  # ([<socket.socket fd=3, family=AddressFamily.AF_INET, type=SocketKind.SOCK_STREAM, proto=0, laddr=('127.0.0.1', 8080)>], [], [])
    # print(server)
    # print(r_list)
    for i in r_list:  #
        """针对不同的对象做不同的处理"""
        if i is server:
            conn, addr = i.accept()
            # 也应该添加到监管的队列中
            read_list.append(conn)
        else:
            res = i.recv(1024)
            if len(res) == 0:
                i.close()
                # 将无效的监管对象 移除
                read_list.remove(i)
                continue
            print(res)
            i.send(b'heiheiheiheihei')
 # 客户端
import socket
client = socket.socket()
client.connect(('127.0.0.1',8080))
while True:
    client.send(b'hello world')
    data = client.recv(1024)
    print(data)
```

### 2.4. IO Multiplexing - I/O多路复用模型

```
IO复用：为了解释这个名词，首先来理解下复用这个概念，复用也就是共用的意思，这样理解还是有些抽象，为此，咱们来理解下复用在通信领域的使用，在通信领域中为了充分利用网络连接的物理介质，往往在同一条网络链路上采用时分复用或频分复用的技术使其在同一链路上传输多路信号，到这里我们就基本上理解了复用的含义，即公用某个“介质”来尽可能多的做同一类(性质)的事，那IO复用的“介质”是什么呢？为此我们首先来看看服务器编程的模型，客户端发来的请求服务端会产生一个进程来对其进行服务，每当来一个客户请求就产生一个进程来服务，然而进程不可能无限制的产生，因此为了解决大量客户端访问的问题，引入了IO复用技术，即：一个进程可以同时对多个客户请求进行服务。也就是说IO复用的“介质”是进程(准确的说复用的是select和poll，因为进程也是靠调用select和poll来实现的)，复用一个进程(select和poll)来对多个IO进行服务，虽然客户端发来的IO是并发的但是IO所需的读写数据多数情况下是没有准备好的，因此就可以利用一个函数(select和poll)来监听IO所需的这些数据的状态，一旦IO有数据可以进行读写了，进程就来对这样的IO进行服务。
理解完IO复用后，我们在来看下实现IO复用中的三个API(select、poll和epoll)的区别和联系
select，poll，epoll都是IO多路复用的机制，I/O多路复用就是通过一种机制，可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知应用程序进行相应的读写操作。但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。三者的原型如下所示：
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
 1.select的第一个参数nfds为fdset集合中最大描述符值加1，fdset是一个位数组，其大小限制为__FD_SETSIZE（1024），位数组的每一位代表其对应的描述符是否需要被检查。第二三四参数表示需要关注读、写、错误事件的文件描述符位数组，这些参数既是输入参数也是输出参数，可能会被内核修改用于标示哪些描述符上发生了关注的事件，所以每次调用select前都需要重新初始化fdset。timeout参数为超时时间，该结构会被内核修改，其值为超时剩余的时间。
 select的调用步骤如下：
（1）使用copy_from_user从用户空间拷贝fdset到内核空间
（2）注册回调函数__pollwait
（3）遍历所有fd，调用其对应的poll方法（对于socket，这个poll方法是sock_poll，sock_poll根据情况会调用到tcp_poll,udp_poll或者datagram_poll）
（4）以tcp_poll为例，其核心实现就是__pollwait，也就是上面注册的回调函数。
（5）__pollwait的主要工作就是把current（当前进程）挂到设备的等待队列中，不同的设备有不同的等待队列，对于tcp_poll 来说，其等待队列是sk->sk_sleep（注意把进程挂到等待队列中并不代表进程已经睡眠了）。在设备收到一条消息（网络设备）或填写完文件数 据（磁盘设备）后，会唤醒设备等待队列上睡眠的进程，这时current便被唤醒了。
（6）poll方法返回时会返回一个描述读写操作是否就绪的mask掩码，根据这个mask掩码给fd_set赋值。
（7）如果遍历完所有的fd，还没有返回一个可读写的mask掩码，则会调用schedule_timeout是调用select的进程（也就是 current）进入睡眠。当设备驱动发生自身资源可读写后，会唤醒其等待队列上睡眠的进程。如果超过一定的超时时间（schedule_timeout 指定），还是没人唤醒，则调用select的进程会重新被唤醒获得CPU，进而重新遍历fd，判断有没有就绪的fd。
（8）把fd_set从内核空间拷贝到用户空间。
总结下select的几大缺点：
（1）每次调用select，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大
（2）同时每次调用select都需要在内核遍历传递进来的所有fd，这个开销在fd很多时也很大
（3）select支持的文件描述符数量太小了，默认是1024
2．  poll与select不同，通过一个pollfd数组向内核传递需要关注的事件，故没有描述符个数的限制，pollfd中的events字段和revents分别用于标示关注的事件和发生的事件，故pollfd数组只需要被初始化一次。
 poll的实现机制与select类似，其对应内核中的sys_poll，只不过poll向内核传递pollfd数组，然后对pollfd中的每个描述符进行poll，相比处理fdset来说，poll效率更高。poll返回后，需要对pollfd中的每个元素检查其revents值，来得指事件是否发生。
3．直到Linux2.6才出现了由内核直接支持的实现方法，那就是epoll，被公认为Linux2.6下性能最好的多路I/O就绪通知方法。epoll可以同时支持水平触发和边缘触发（Edge Triggered，只告诉进程哪些文件描述符刚刚变为就绪状态，它只说一遍，如果我们没有采取行动，那么它将不会再次告知，这种方式称为边缘触发），理论上边缘触发的性能要更高一些，但是代码实现相当复杂。epoll同样只告知那些就绪的文件描述符，而且当我们调用epoll_wait()获得就绪文件描述符时，返回的不是实际的描述符，而是一个代表就绪描述符数量的值，你只需要去epoll指定的一个数组中依次取得相应数量的文件描述符即可，这里也使用了内存映射（mmap）技术，这样便彻底省掉了这些文件描述符在系统调用时复制的开销。另一个本质的改进在于epoll采用基于事件的就绪通知方式。在select/poll中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而epoll事先通过epoll_ctl()来注册一个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait()时便得到通知。
epoll既然是对select和poll的改进，就应该能避免上述的三个缺点。那epoll都是怎么解决的呢？在此之前，我们先看一下epoll 和select和poll的调用接口上的不同，select和poll都只提供了一个函数——select或者poll函数。而epoll提供了三个函 数，epoll_create,epoll_ctl和epoll_wait，epoll_create是创建一个epoll句柄；epoll_ctl是注 册要监听的事件类型；epoll_wait则是等待事件的产生。
　　对于第一个缺点，epoll的解决方案在epoll_ctl函数中。每次注册新的事件到epoll句柄中时（在epoll_ctl中指定 EPOLL_CTL_ADD），会把所有的fd拷贝进内核，而不是在epoll_wait的时候重复拷贝。epoll保证了每个fd在整个过程中只会拷贝 一次。
　　对于第二个缺点，epoll的解决方案不像select或poll一样每次都把current轮流加入fd对应的设备等待队列中，而只在 epoll_ctl时把current挂一遍（这一遍必不可少）并为每个fd指定一个回调函数，当设备就绪，唤醒等待队列上的等待者时，就会调用这个回调 函数，而这个回调函数会把就绪的fd加入一个就绪链表）。epoll_wait的工作实际上就是在这个就绪链表中查看有没有就绪的fd（利用 schedule_timeout()实现睡一会，判断一会的效果，和select实现中的第7步是类似的）。
　　对于第三个缺点，epoll没有这个限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子, 在1GB内存的机器上大约是10万左右，具体数目可以cat /proc/sys/fs/file-max察看,一般来说这个数目和系统内存关系很大。
总结：
（1）select，poll实现需要自己不断轮询所有fd集合，直到设备就绪，期间可能要睡眠和唤醒多次交替。而epoll其实也需要调用 epoll_wait不断轮询就绪链表，期间也可能多次睡眠和唤醒交替，但是它是设备就绪时，调用回调函数，把就绪fd放入就绪链表中，并唤醒在 epoll_wait中进入睡眠的进程。虽然都要睡眠和交替，但是select和poll在“醒着”的时候要遍历整个fd集合，而epoll在“醒着”的 时候只要判断一下就绪链表是否为空就行了，这节省了大量的CPU时间，这就是回调机制带来的性能提升。
（2）select，poll每次调用都要把fd集合从用户态往内核态拷贝一次，并且要把current往设备等待队列中挂一次，而epoll只要 一次拷贝，而且把current往等待队列上挂也只挂一次（在epoll_wait的开始，注意这里的等待队列并不是设备等待队列，只是一个epoll内 部定义的等待队列），这也能节省不少的开销。
###############
这三种IO多路复用模型在不同的平台有着不同的支持，而epoll在windows下就不支持，好在我们有selectors模块，帮我们默认选择当前平台下最合适的
##############
#服务端
from socket import *
import selectors
sel=selectors.DefaultSelector()
def accept(server_fileobj,mask):
    conn,addr=server_fileobj.accept()
    sel.register(conn,selectors.EVENT_READ,read)
def read(conn,mask):
    try:
        data=conn.recv(1024)
        if not data:
            print('closing',conn)
            sel.unregister(conn)
            conn.close()
            return
        conn.send(data.upper()+b'_SB')
    except Exception:
        print('closing', conn)
        sel.unregister(conn)
        conn.close()
server_fileobj=socket(AF_INET,SOCK_STREAM)
server_fileobj.setsockopt(SOL_SOCKET,SO_REUSEADDR,1)
server_fileobj.bind(('127.0.0.1',8088))
server_fileobj.listen(5)
server_fileobj.setblocking(False) #设置socket的接口为非阻塞
sel.register(server_fileobj,selectors.EVENT_READ,accept) #相当于网select的读列表里append了一个文件句柄server_fileobj,并且绑定了一个回调函数accept
while True:
    events=sel.select() #检测所有的fileobj，是否有完成wait data的
    for sel_obj,mask in events:
        callback=sel_obj.data #callback=accpet
        callback(sel_obj.fileobj,mask) #accpet(server_fileobj,1)
#客户端
from socket import *
c=socket(AF_INET,SOCK_STREAM)
c.connect(('127.0.0.1',8088))
while True:
    msg=input('>>: ')
    if not msg:continue
    c.send(msg.encode('utf-8'))
    data=c.recv(1024)
    print(data.decode('utf-8'))
```

### 2.5. AIO – 异步I/O模型

发起请求立刻得到回复，不用挂起等待； 数据会由内核进程主动完成拷贝

```
'''
举个例子：还是买菜刀
现在是网上下单到商店（系统调用）
商店确认（返回）
商店去进货（io操作）
商店收到货把货发个卖家（从内核缓冲区拷贝到用户缓冲区）
买家收到货（指定信号）
整个过程无等待
异步io
AIO框架在windows下使用windows IOCP技术，在Linux下使用epoll多路复用IO技术模拟异步IO
市面上多数的高并发框架，都没有使用异步io而是用的io多路复用，因为io多路复用技术很成熟且稳定，并且在实际的使用过程中，异步io并没有比io多路复用性能提升很多，没有达到很明显的程度
并且，真正的AIO编码难度比io多路复用高很多
'''
```

![img](https://pic4.zhimg.com/80/v2-d7d6e252a28ac2ae5163e881b388e11b_720w.webp)

### 2.6.select poll 和epoll

```
#  1 select poll 和epoll都是io多路复用技术
select, poll , epoN都是io多路复用的机制。I/O多路复用就是通过一种机 制个进程可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。但select, poll , epoll本质上都是同步I/O ,因为他们都需要在读写事件就绪后自己负责进行读写， 也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异 步I/O的实现会负责把数据从内核拷贝到用户空间。
# 2 select
select函数监视的文件描述符分3类，分别是writefds、readfds、和 exceptfds。调用后select函数会阻塞，直到有描述副就绪（有数据可读、 可写、或者有except）,或者超时（timeout指定等待时间，如果立即返回 设为null即可），函数返回。当select函数返回后，可以通过遍历fdset,来 找到就绪的描述符。
select目前几乎在所有的平台上支持，其良好跨平台支持也是它的一个 优点。select的一个缺点在于单个进程能够监视的文件描述符的数量存在最大限制，在Linux上一般为1024 ,可以通过修改宏定义甚至重新编译内核的 方式提升这一限制，但是这样也会造成效率的降低。
# 3 poll
不同于select使用三个位图来表示三个fdset的方式，poll使用一个 pollfd的指针实现。
pollfd结构包含了要监视的event和发生的event,不再使用select '参数-值'传递的方式。同时，pollfd并没有最大数量限制（但是数量过大后 性能也是会下降）。和select函数一样，poll返回后，需要轮询pollfd来获取就绪的描述符。
从上面看，select和poll都需要在返回后，通过遍历文件描述符来获取 已经就绪的socket。事实上，同时连接的大量客户端在一时刻可能只有很少的处于就绪状态，因此随着监视的描述符数量的增长，其效率也会线性下降
# 4 epoll
epoll是在linux2.6内核中提出的，是之前的select和poll的增强版本。相对 于select和poll来说，epoll更加灵活，没有描述符限制。epoll使用一个文 件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。
# 5 更好的例子理解
老师检查同学作业，一班50个人，一个一个问，同学，作业写完了没？select，poll
老师检查同学作业，一班50个人，同学写完了主动举手告诉老师，老师去检查 epoll
# 6 总结
在并发高的情况下，连接活跃度不高，epoll比select好，网站http的请求，连了就断掉
并发性不高，同时连接很活跃，select比epoll好，websocket的连接，长连接，游戏开发
```

## 3.同步I/O与异步I/O

- 同步I/O
- - 概念：导致请求进程阻塞的I/O操作，直到I/O操作任务完成
  - 类型：BIO、NIO、IO Multiplexing
- 异步I/O
- - 概念：不导致进程阻塞的I/O操作
  - 类型：AIO

注意：

- 同步I/O与异步I/O判断依据是，是否会导致用户进程阻塞
- BIO中socket直接阻塞等待（用户进程主动等待，并在拷贝时也等待）
- NIO中将数据从内核空间拷贝到用户空间时阻塞（用户进程主动询问，并在拷贝时等待）
- IO Multiplexing中select等函数为阻塞、拷贝数据时也阻塞（用户进程主动等待，并在拷贝时也等待）
- AIO中从始至终用户进程都没有阻塞（用户进程是被动的）

## 4.并发-并行-同步-异步-阻塞-非阻塞

```
# 1 并发
并发是指一个时间段内，有几个程序在同一个cpu上执行，但是同一时刻，只有一个程序在cpu上运行
跑步，鞋带开了，停下跑步，系鞋带
# 2 并行
指任意时刻点上，有多个程序同时运行在多个cpu上
跑步，边跑步边听音乐
# 3 同步：
指代码调用io操作时，必须等待io操作完成才返回的调用方式
# 4 异步
异步是指代码调用io操作时，不必等io操作完成就返回调用方式
# 6 阻塞
指调用函数时候，当前线程别挂起
# 6 非阻塞
指调用函数时候，当前线程不会被挂起，而是立即返回
# 区别：
同步和异步是消息通讯的机制
阻塞和非阻塞是函数调用机制
```

## 5.IO设计模式

```
Reactor模式，基于同步I/O实现
- Proactor模式，基于异步I/O实现
```

Reactor模式通常采用IO多路复用机制进行具体实现

```
- kqueue、epoll、poll、select等机制
```

Proactor模式通常采用OS Asynchronous IO(AIO)的异步机制进行实现

```
- 前提是对应操作系统支持AIO，比如支持异步IO的linux(不太成熟)、具备IOCP的windows server(非常成熟)
```

Reactor模式和Proactor模式都是事件驱动，主要实现步骤：

1. 事件注册：将事件与事件处理器进行分离。将事件注册到事件循环中，将事件处理器单独管理起来，记录其与事件的对应关系。
2. 事件监听：启动事件循环，一旦事件已经就绪/完成，就立刻通知事件处理器
3. 事件分发：当收到事件就绪/完成的信号，便立刻激活与之对应的事件处理器
4. 事件处理：在进程/线程/协程中执行事件处理器

使用过程中，用户通常只负责**定义事件和事件处理器**并将其注册以及一开始的**事件循环的启动**，这个过程就会是以异步的形式执行任务。

### 5.1.Reactor模式

![img](https://pic2.zhimg.com/80/v2-3e46412eb280f1728c26a7dddcd8492d_720w.webp)

### 5.2.Proactor模式

![img](https://pic2.zhimg.com/80/v2-a1f360c7abb19fc51a111fcc28f42c11_720w.webp)

### 5.3.对比分析

Reactor模型处理耗时长的操作会造成事件分发的阻塞，影响到后续事件的处理；

Proactor模型实现逻辑复杂；依赖操作系统对异步的支持，目前实现了纯异步操作的操作系统少，实现优秀的如windows IOCP，但由于其windows系统用于服务器的局限性，目前应用范围较小；而Unix/Linux系统对纯异步的支持有限，因而应用事件驱动的主流还是基于select/epoll等实现的reactor模式

Python中：如asyncio、gevent、tornado、twisted等异步模块都是依据事件驱动模型设计，更多的都是使用reactor模型，其中部分也支持proactor模式，当然需要根据当前运行的操作系统环境来进行手动配置

原文链接：https://zhuanlan.zhihu.com/p/375177483

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)