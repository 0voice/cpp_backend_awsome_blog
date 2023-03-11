# 【NO.187】网络不通？服务丢包？这篇 TCP 连接状态详解及故障排查，收好了~

我们通过了解TCP各个状态，可以排除和定位网络或系统故障时大有帮助。

### **1.TCP状态**

了解TCP之前，先了解几个命令：

**linux查看tcp的状态命令**：
\1) **`netstat -nat`** 查看TCP各个状态的数量
2)**`lsof -i:port`** 可以检测到打开套接字的状况
\3) **`sar -n SOCK`** 查看tcp创建的连接数
\4) **`tcpdump -iany tcp port 9000`** 对tcp端口为9000的进行抓包

网络测试常用命令;
1）ping:检测网络连接的正常与否,主要是测试延时、抖动、丢包率。

但是很多服务器为了防止攻击，一般会关闭对ping的响应。所以ping一般作为测试连通性使用。

ping命令后，会接收到对方发送的回馈信息，其中记录着对方的IP地址和TTL。TTL是该字段指定IP包被路由器丢弃之前允许通过的最大网段数量。

TTL是IPv4包头的一个8 bit字段。例如IP包在服务器中发送前设置的TTL是64，你使用ping命令后，得到服务器反馈的信息，其中的TTL为56，说明途中一共经过了8道路由器的转发，每经过一个路由，TTL减1。

2）traceroute：raceroute 跟踪数据包到达网络主机所经过的路由工具
**`traceroute hostname`**

3）pathping：是一个路由跟踪工具，它将 ping 和 tracert 命令的功能与这两个工具所不提供的其他信息结合起来，综合了二者的功能

**`pathping www.baidu.com`**

4）mtr：以结合ping nslookup tracert 来判断网络的相关特性

\5) nslookup:用于解析域名，一般用来检测本机的DNS设置是否配置正确。

LISTENING：侦听来自远方的TCP端口的连接请求.
首先服务端需要打开一个socket进行监听，状态为LISTEN。

有提供某种服务才会处于LISTENING状态，**TCP状态变化就是某个端口的状态变化**，提供一个服务就打开一个端口。

例如：提供www服务默认开的是80端口，提供ftp服务默认的端口为21，当提供的服务没有被连接时就处于LISTENING状态。

FTP服务启动后首先处于侦听（LISTENING）状态。处于侦听LISTENING状态时，该端口是开放的，等待连接，但还没有被连接。就像你房子的门已经敞开的，但还没有人进来。

看LISTENING状态最主要的是看本机开了哪些端口，这些端口都是哪个程序开的，关闭不必要的端口是保证安全的一个非常重要的方面，服务端口都对应一个服务（应用程序），停止该服务就关闭了该端口，例如要关闭21端口只要停止IIS服务中的FTP服务即可。关于这方面的知识请参阅其它文章。

如果你不幸中了服务端口的木马，木马也开个端口处于LISTENING状态。

**SYN-SENT：客户端SYN_SENT状态**：
再发送连接请求后等待匹配的连接请求:客户端通过应用程序调用connect进行active open.

于是客户端tcp发送一个SYN以请求建立一个连接.之后状态置为SYN_SENT.

> The socket is actively attempting to establish a connection. 在发送连接请求后等待匹配的连接请求

当请求连接时客户端首先要发送同步信号给要访问的机器，此时状态为SYN_SENT，如果连接成功了就变为ESTABLISHED，正常情况下SYN_SENT状态非常短暂。

例如要访问网站[http://www.baidu.com](https://link.zhihu.com/?target=http%3A//www.baidu.com),如果是正常连接的话，用TCPView观察IEXPLORE.EXE（IE）建立的连接会发现很快从SYN_SENT变为ESTABLISHED，表示连接成功。SYN_SENT状态快的也许看不到。

如果发现有很多SYN_SENT出现，那一般有这么几种情况，一是你要访问的网站不存在或线路不好。

二是用扫描软件扫描一个网段的机器，也会出出现很多SYN_SENT，另外就是可能中了病毒了，例如中了”冲击波”，病毒发作时会扫描其它机器，这样会有很多SYN_SENT出现。

**SYN-RECEIVED：服务器端状态SYN_RCVD**
再收到和发送一个连接请求后等待对方对连接请求的确认
当服务器收到客户端发送的同步信号时，将标志位ACK和SYN置1发送给客户端，此时服务器端处于SYN_RCVD状态，如果连接成功了就变为ESTABLISHED，正常情况下SYN_RCVD状态非常短暂。

**如果发现有很多SYN_RCVD状态，那你的机器有可能被SYN Flood的DoS(拒绝服务攻击)攻击了。**

SYN Flood的攻击原理是：
在进行三次握手时，攻击软件向被攻击的服务器发送SYN连接请求（握手的第一步），但是这个地址是伪造的，如攻击软件随机伪造了51.133.163.104、65.158.99.152等等地址。

服务器在收到连接请求时将标志位ACK和SYN置1发送给客户端（握手的第二步），但是这些客户端的IP地址都是伪造的，服务器根本找不到客户机，也就是说握手的第三步不可能完成。

这种情况下服务器端一般会重试（再次发送SYN+ACK给客户端）并等待一段时间后丢弃这个未完成的连接，这段时间的长度我们称为SYN Timeout，一般来说这个时间是分钟的数量级（大约为30秒-2分钟）；

一个用户出现异常导致服务器的一个线程等待1分钟并不是什么很大的问题，但如果有一个恶意的攻击者大量模拟这种情况，服务器端将为了维护一个非常大的半连接列表而消耗非常多的资源——数以万计的半连接。

即使是简单的保存并遍历也会消耗非常多的CPU时间和内存，何况还要不断对这个列表中的IP进行SYN+ACK的重试。

此时从正常客户的角度看来，服务器失去响应，这种情况我们称做：服务器端受到了**SYN Flood攻击（SYN洪水攻击）**

**ESTABLISHED：代表一个打开的连接。**
ESTABLISHED状态是表示两台机器正在传输数据，观察这个状态最主要的就是看哪个程序正在处于ESTABLISHED状态。

服务器出现很多**ESTABLISHED状态：netstat -nat |grep 9502或者使用lsof -i:9502可以检测到。**

**当客户端未主动close的时候就断开连接：即客户端发送的FIN丢失或未发送。**
这时候若客户端断开的时候发送了FIN包，则服务端将会处于CLOSE_WAIT状态；

这时候若客户端断开的时候未发送FIN包，则服务端处还是显示ESTABLISHED状态；

结果客户端重新连接服务器。

而新连接上来的客户端（也就是刚才断掉的重新连上来了）在服务端肯定是ESTABLISHED; 如果客户端重复的上演这种情况，那么服务端将会出现大量的假的ESTABLISHED连接和CLOSE_WAIT连接。

最终结果就是新的其他客户端无法连接上来，但是利用netstat还是能看到一条连接已经建立，并显示ESTABLISHED，但始终无法进入程序代码。

**FIN-WAIT-1**：等待远程TCP连接中断请求，或先前的连接中断请求的确认
主动关闭(active close)端应用程序调用close，于是其TCP发出FIN请求主动关闭连接，之后进入FIN_WAIT1状态./ *The socket is closed, and the connection is shutting down. 等待远程TCP的连接中断请求，或先前的连接中断请求的确认* /

如果服务器出现shutdown再重启，使用netstat -nat查看，就会看到很多FIN-WAIT-1的状态。就是因为服务器当前有很多客户端连接，直接关闭服务器后，无法接收到客户端的ACK。

**FIN-WAIT-2**：从远程TCP等待连接中断请求
主动关闭端接到ACK后，就进入了FIN-WAIT-2

> Connection is closed, and the socket is waiting for a shutdown from the remote end. 从远程TCP等待连接中断请求

这就是著名的半关闭的状态了，这是在关闭连接时，客户端和服务器两次握手之后的状态。

在这个状态下，应用程序还有接受数据的能力，但是已经无法发送数据，但是也有一种可能是，客户端一直处于FIN_WAIT_2状态，而服务器则一直处于WAIT_CLOSE状态，而直到应用层来决定关闭这个状态。

**CLOSE-WAIT：等待从本地用户发来的连接中断请求**
被动关闭(passive close)端TCP接到FIN后，就发出ACK以回应FIN请求(它的接收也作为文件结束符传递给上层应用程序),并进入CLOSE_WAIT.

> The remote end has shut down, waiting for the socket to close. 等待从本地用户发来的连接中断请求

**CLOSING：等待远程TCP对连接中断的确认**
比较少见

> Both sockets are shut down but we still don’t have all our data sent. 等待远程TCP对连接中断的确认

**LAST-ACK：等待原来的发向远程TCP的连接中断请求的确认**
被动关闭端一段时间后，接收到文件结束符的应用程序将调用CLOSE关闭连接。这导致它的TCP也发送一个
FIN,等待对方的ACK.就进入了LAST-ACK .

> The remote end has shut down, and the socket is closed. Waiting for acknowledgement. 等待原来发向远程TCP的连接中断请求的确认

使用并发压力测试的时候，突然断开压力测试客户端，服务器会看到很多LAST-ACK。

**TIME-WAIT：等待足够的时间以确保远程TCP接收到连接中断请求的确认**
在主动关闭端接收到FIN后，TCP就发送ACK包，并进入TIME-WAIT状态。

> The socket is waiting after close to handle
> packets still in the network.等待足够的时间以确保远程TCP接收到连接中断请求的确认

TIME_WAIT等待状态，这个状态又叫做2MSL状态，说的是在TIME_WAIT2发送了最后一个ACK数据报以后，要进入TIME_WAIT状态，这个状态是防止最后一次握手的数据报没有传送到对方那里而准备的（注意这不是四次握手，这是第四次握手的保险状态）。

这个状态在很大程度上保证了双方都可以正常结束，但是，问题也来了。

由于插口的2MSL状态（插口是IP和端口对的意思，socket），使得应用程序在2MSL时间内是无法再次使用同一个插口的，对于客户程序还好一些，但是对于服务程序，例如httpd，它总是要使用同一个端口来进行服务，而在2MSL时间内，启动httpd就会出现错误（插口被使用）。

为了避免这个错误，服务器给出了一个平静时间的概念，这是说在2MSL时间内，虽然可以重新启动服务器，但是这个服务器还是要平静的等待2MSL时间的过去才能进行下一次连接。

详情请看：TIME_WAIT引起Cannot assign requested address报错

**CLOSED：没有任何连接状态**
被动关闭端在接受到ACK包后，就进入了closed的状态。连接结束

> The socket is not being used. 没有任何连接状态

### **2.TCP状态迁移路线图**

client/server两条路线讲述TCP状态迁移路线图：

![img](https://pic4.zhimg.com/80/v2-7f4750683e6f671cc5b05bdef90f0193_720w.webp)

这是一个看起来比较复杂的状态迁移图，因为它包含了两个部分—-服务器的状态迁移和客户端的状态迁移，如果从某一个角度出发来看这个图，就会清晰许多，这里面的服务器和客户端都不是绝对的，发送数据的就是客户端，接受数据的就是服务器。

客户端应用程序的状态迁移图
客户端的状态可以用如下的流程来表示：

CLOSED->SYN_SENT->ESTABLISHED->FIN_WAIT_1->FIN_WAIT_2->TIME_WAIT->CLOSED

以上流程是在程序正常的情况下应该有的流程，从书中的图中可以看到，在建立连接时，当客户端收到SYN报文的ACK以后，客户端就打开了数据交互地连接。

而结束连接则通常是客户端主动结束的，客户端结束应用程序以后，需要经历FIN_WAIT_1，FIN_WAIT_2等状态，这些状态的迁移就是前面提到的结束连接的四次握手。

服务器的状态迁移图
服务器的状态可以用如下的流程来表示：

CLOSED->LISTEN->SYN收到->ESTABLISHED->CLOSE_WAIT->LAST_ACK->CLOSED

在建立连接的时候，服务器端是在第三次握手之后才进入数据交互状态，而关闭连接则是在关闭连接的第二次握手以后（注意不是第四次）。而关闭以后还要等待客户端给出最后的ACK包才能进入初始的状态。

**其他状态迁移**
还有一些其他的状态迁移，这些状态迁移针对服务器和客户端两方面的总结如下

LISTEN->SYN*SENT，对于这个解释就很简单了，服务器有时候也要打开连接的嘛。*

*SYN_SENT->SYN收到，服务器和客户端在SYN_SENT状态下如果收到SYN数据报，则都需要发送SYN的ACK数据报并把自己的状态调整到SYN收到状态，准备进入ESTABLISHED*
*SYN_SENT->CLOSED，在发送超时的情况下，会返回到CLOSED状态。*

*SYN*收到->LISTEN，如果受到RST包，会返回到LISTEN状态。

SYN_收到->FIN_WAIT_1，这个迁移是说，可以不用到ESTABLISHED状态，而可以直接跳转到FIN_WAIT_1状态并等待关闭。

![img](https://pic1.zhimg.com/80/v2-eda25767646980fc29d9ac7aef1860cc_720w.webp)

怎样牢牢地将这张图刻在脑中呢？那么你就一定要对这张图的每一个状态，及转换的过程有深刻的认识，不能只停留在一知半解之中。

下面对这张图的11种状态详细解析一下，以便加强记忆！不过在这之前，先回顾一下TCP建立连接的三次握手过程，以及关闭连接的四次握手过程。

### **3.TCP连接建立三次握手**

TCP是一个面向连接的协议，所以在连接双方发送数据之前，都需要首先建立一条连接。

**Client连接Server**：
当Client端调用socket函数调用时，相当于Client端产生了一个处于Closed状态的套接字。

**(1)第一次握手**：Client端又调用connect函数调用，系统为Client随机分配一个端口，连同传入connect中的参数(Server的IP和端口)，这就形成了一个连接四元组，客户端发送一个带SYN标志的TCP报文到服务器。

这是三次握手过程中的报文1。connect调用让Client端的socket处于SYN_SENT状态，等待服务器确认；SYN：同步序列编号(Synchronize Sequence Numbers)。

**(2)第二次握手**：服务器收到syn包，必须确认客户的SYN（ack=j+1），同时自己也发送一个SYN包（syn=k），即SYN+ACK包，此时服务器进入SYN_RECV状态；

**(3) 第三次握手**：客户端收到服务器的SYN+ACK包，向服务器发送确认包ACK(ack=k+1)，此包发送完毕，客户器和客务器进入ESTABLISHED状态，完成三次握手。连接已经可以进行读写操作。

一个完整的三次握手也就是：请求—-应答—-再次确认。
TCP协议通过三个报文段完成连接的建立，这个过程称为三次握手(three-way handshake)，过程如下图所示。

对应的函数接口：

![img](https://pic4.zhimg.com/80/v2-c9c0c9354295d05a8bc178c45f8d28bb_720w.webp)

2)Server
当Server端调用socket函数调用时，相当于Server端产生了一个处于Closed状态的监听套接字，Server端调用bind操作，将监听套接字与指定的地址和端口关联，然后又调用listen函数，系统会为其分配未完成队列和完成队列，此时的监听套接字可以接受Client的连接，监听套接字状态处于LISTEN状态。

当Server端调用accept操作时，会从完成队列中取出一个已经完成的client连接，同时在server这段会产生一个会话套接字，用于和client端套接字的通信，这个会话套接字的状态是ESTABLISH。

从图中可以看出，当客户端调用connect时，触发了连接请求，向服务器发送了SYN J包，这时connect进入阻塞状态；

服务器监听到连接请求，即收到SYN J包，调用accept函数接收请求向客户端发送SYN K ，ACK J+1，这时accept进入阻塞状态；客户端收到服务器的SYN K ，ACK J+1之后，这时connect返回，并对SYN K进行确认；服务器收到ACK K+1时，accept返回，至此三次握手完毕，连接建立。

我们可以通过网络抓包的查看具体的流程：

比如我们服务器开启9502的端口。使用tcpdump来抓包：tcpdump -iany tcp port 9502

然后我们使用telnet 127.0.0.1 9502开连接:

![img](https://pic1.zhimg.com/80/v2-a6d9516c6e81ad45ded6cde751f75060_720w.webp)

我们看到 （1）（2）（3）三步是建立tcp：
第一次握手：
**`14:12:45.104687 IP localhost.39870 > localhost.9502: Flags [S], seq 2927179378`**
客户端**`IP localhost.39870`** (客户端的端口一般是自动分配的) 向服务器**`localhost.9502`** 发送syn包(syn=j)到服务器》

syn的seq=**`2927179378`**

第二次握手：
**`14:12:45.104701 IP localhost.9502 > localhost.39870: Flags`** [S.], seq 1721825043, ack 2927179379,
服务器收到syn包，必须确认客户的SYN（ack=j+1），同时自己也发送一个SYN包（syn=k），即SYN+ACK包
SYN（ack=j+1）=ack 2927179379 服务器主机SYN包（syn=seq 1721825043）

第三次握手：
**`14:12:45.104711 IP localhost.39870 > localhost.9502: Flags [.], ack 1,`**
客户端收到服务器的SYN+ACK包，向服务器发送确认包ACK(ack=k+1)
客户端和服务器进入ESTABLISHED状态后，可以进行通信数据交互。此时和accept接口没有关系，即使没有accepte，也进行3次握手完成。

连接出现连接不上的问题，一般是网路出现问题或者网卡超负荷或者是连接数已经满啦。

紫色背景的部分：

IP localhost.39870 > localhost.9502: Flags [P.], seq 1:8, ack 1, win 4099, options [nop,nop,TS val 255478182 ecr 255474104], length 7
客户端向服务器发送长度为7个字节的数据，

IP localhost.9502 > localhost.39870: Flags [.], ack 8, win 4096, options [nop,nop,TS val 255478182 ecr 255478182], length 0
服务器向客户确认已经收到数据

IP localhost.9502 > localhost.39870: Flags [P.], seq 1:19, ack 8, win 4096, options [nop,nop,TS val 255478182 ecr 255478182], length 18
然后服务器同时向客户端写入数据。

IP localhost.39870 > localhost.9502: Flags [.], ack 19, win 4097, options [nop,nop,TS val 255478182 ecr 255478182], length 0
客户端向服务器确认已经收到数据

这个就是tcp可靠的连接，每次通信都需要对方来确认。

### **4. TCP连接的终止（四次握手释放）**

由于TCP连接是全双工的，因此每个方向都必须单独进行关闭。这原则是当一方完成它的数据发送任务后就能发送一个FIN来终止这个方向的连接。收到一个 FIN只意味着这一方向上没有数据流动，一个TCP连接在收到一个FIN后仍能发送数据。

首先进行关闭的一方将执行主动关闭，而另一方执行被动关闭。

建立一个连接需要三次握手，而终止一个连接要经过四次握手，这是由TCP的半关闭(half-close)造成的，如图：

![img](https://pic1.zhimg.com/80/v2-21dbe7a57efc7f7725cd3850d69c4674_720w.webp)

（1）客户端A发送一个FIN，用来关闭客户A到服务器B的数据传送（报文段4）。
（2）服务器B收到这个FIN，它发回一个ACK，确认序号为收到的序号加1（报文段5）。和SYN一样，一个FIN将占用一个序号。
（3）服务器B关闭与客户端A的连接，发送一个FIN给客户端A（报文段6）。
（4）客户端A发回ACK报文确认，并将确认序号设置为收到序号加1（报文段7）。

对应函数接口如图：

![img](https://pic1.zhimg.com/80/v2-0dad8dda95961a85d94ca68c897ff330_720w.webp)

调用过程如下：

\1) 当client想要关闭它与server之间的连接。client（某个应用进程）首先调用close主动关闭连接，这时TCP发送一个FIN M；client端处于FIN_WAIT1状态。

\2) 当server端接收到FIN M之后，执行被动关闭。对这个FIN进行确认，返回给client ACK。

当server端返回给client ACK后，client处于FIN_WAIT2状态，server处于CLOSE_WAIT状态。它的接收也作为文件结束符传递给应用进程，因为FIN的接收 意味着应用进程在相应的连接上再也接收不到额外数据；

\3) 一段时间之后，当server端检测到client端的关闭操作(read返回为0)。接收到文件结束符的server端调用close关闭它的socket。这导致server端的TCP也发送一个FIN N；此时server的状态为LAST_ACK。

\4) 当client收到来自server的FIN后 。client端的套接字处于TIME_WAIT状态，它会向server端再发送一个ack确认，此时server端收到ack确认后，此套接字处于CLOSED状态。

这样每个方向上都有一个FIN和ACK。

1．为什么建立连接协议是三次握手，而关闭连接却是四次握手呢？

这是因为服务端的LISTEN状态下的SOCKET当收到SYN报文的建连请求后，它可以把ACK和SYN（ACK起应答作用，而SYN起同步作用）放在一个报文里来发送。但关闭连接时，当收到对方的FIN报文通知时，它仅仅表示对方没有数据发送给你了；

但未必你所有的数据都全部发送给对方了，所以你可以未必会马上会关闭SOCKET,也即你可能还需要发送一些数据给对方之后，再发送FIN报文给对方来表示你同意现在可以关闭连接了，所以它这里的ACK报文和FIN报文多数情况下都是分开发送的。

2．为什么TIME_WAIT状态还需要等2MSL后才能返回到CLOSED状态？

这是因为虽然双方都同意关闭连接了，而且握手的4个报文也都协调和发送完毕，按理可以直接回到CLOSED状态（就好比从SYN_SEND状态到ESTABLISH状态那样）：

一方面是可靠的实现TCP全双工连接的终止，也就是当最后的ACK丢失后，被动关闭端会重发FIN，因此主动关闭端需要维持状态信息，以允许它重新发送最终的ACK。

另一方面，但是因为我们必须要假想网络是不可靠的，你无法保证你最后发送的ACK报文会一定被对方收到，因此对方处于LAST_ACK状态下的SOCKET可能会因为超时未收到ACK报文，而重发FIN报文，所以这个TIME_WAIT状态的作用就是用来重发可能丢失的ACK报文。

TCP在2MSL等待期间，定义这个连接(4元组)不能再使用，任何迟到的报文都会丢弃。设想如果没有2MSL的限制，恰好新到的连接正好满足原先的4元组，这时候连接就可能接收到网络上的延迟报文就可能干扰最新建立的连接。

3、发现系统存在大量TIME_WAIT状态的连接，可以通过调整内核参数解决：vi /etc/sysctl.conf 加入以下内容：
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_fin_timeout = 30
然后执行 /sbin/sysctl -p 让参数生效。
net.ipv4.tcp_syncookies = 1 表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭；
net.ipv4.tcp_tw_reuse = 1 表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；
net.ipv4.tcp_tw_recycle = 1 表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭。
net.ipv4.tcp_fin_timeout 修改系統默认的 TIMEOUT 时间

### **5.同时打开**

两个应用程序同时执行主动打开的情况是可能的，虽然发生的可能性较低。每一端都发送一个SYN,并传递给对方，且每一端都使用对端所知的端口作为本地端口。例如：

主机a中一应用程序使用7777作为本地端口，并连接到主机b 8888端口做主动打开。

主机b中一应用程序使用8888作为本地端口，并连接到主机a 7777端口做主动打开。

tcp协议在遇到这种情况时，只会打开一条连接。

这个连接的建立过程需要4次数据交换，而一个典型的连接建立只需要3次交换（即3次握手）

但多数伯克利版的tcp/ip实现并不支持同时打开。

![img](https://pic1.zhimg.com/80/v2-76b1093a6224a27b3b00107d7dcab3b8_720w.webp)

### **6.同时关闭**

如果应用程序同时发送FIN，则在发送后会首先进入FIN_WAIT_1状态。在收到对端的FIN后，回复一个ACK，会进入CLOSING状态。在收到对端的ACK后，进入TIME_WAIT状态。这种情况称为同时关闭。

同时关闭也需要有4次报文交换，与典型的关闭相同。

### **7. TCP的FLAGS说明**

在TCP层，有个FLAGS字段，这个字段有以下几个标识：SYN, FIN, ACK, PSH, RST, URG.
其中，对于我们日常的分析有用的就是前面的五个字段。

**一、字段含义**：
**1、SYN表示建立连接**：
步序列编号(Synchronize Sequence Numbers)栏有效。该标志仅在三次握手建立TCP连接时有效。它提示TCP连接的服务端检查序列编号，该序列编号为TCP连接初始端(一般是客户端)的初始序列编号。在这里，可以把TCP序列编号看作是一个范围从0到4，294，967，295的32位计数器。通过TCP连接交换的数据中每一个字节都经过序列编号。在TCP报头中的序列编号栏包括了TCP分段中第一个字节的序列编号。

**2、FIN表示关闭连接**：

**3、ACK表示响应**：
确认编号(Acknowledgement Number)栏有效。大多数情况下该标志位是置位的。TCP报头内的确认编号栏内包含的确认编号(w+1，Figure-1)为下一个预期的序列编号，同时提示远端系统已经成功接收所有数据。

**4、PSH表示有DATA数据传输**：

5、RST表示连接重置：复位标志有效。用于复位相应的TCP连接。

**二、字段组合含义**：

![img](https://pic1.zhimg.com/80/v2-9a6b838c32270ca3255100a78d022ea8_720w.webp)

其中，ACK是可能与SYN，FIN等同时使用的，比如SYN和ACK可能同时为1，它表示的就是建立连接之后的响应，
如果只是单个的一个SYN，它表示的只是建立连接。
TCP的几次握手就是通过这样的ACK表现出来的。
但SYN与FIN是不会同时为1的，因为前者表示的是建立连接，而后者表示的是断开连接。

RST一般是在FIN之后才会出现为1的情况，表示的是连接重置。
一般地，当出现FIN包或RST包时，我们便认为客户端与服务器端断开了连接；

![img](https://pic4.zhimg.com/80/v2-c1b1a6f08bc39ba2cb18557cb3ed98a3_720w.webp)

**RST与ACK标志位都置一了，并且具有ACK number，非常明显，这个报文在释放TCP连接的同时，完成了对前面已接收报文的确认。**

而当出现SYN和SYN＋ACK包时，我们认为客户端与服务器建立了一个连接。

PSH为1的情况，一般只出现在 DATA内容不为0的包中，也就是说PSH为1表示的是有真正的TCP数据包内容被传递。

TCP的连接建立和连接关闭，都是通过请求－响应的模式完成的。

### **8. TCP通信中服务器处理客户端意外断开**

如果TCP连接被对方正常关闭，也就是说，对方是正确地调用了closesocket(s)或者shutdown(s)的话，那么上面的Recv或Send调用就能马上返回，并且报错。这是由于close socket(s)或者shutdown(s)有个正常的关闭过程，会告诉对方“TCP连接已经关闭，你不需要再发送或者接受消息了”。

但是，如果意外断开，客户端（3g的移动设备）并没有正常关闭socket。双方并未按照协议上的四次挥手去断开连接。

那么这时候正在执行Recv或Send操作的一方就会因为没有任何连接中断的通知而一直等待下去，也就是会被长时间卡住。

像这种如果一方已经关闭或异常终止连接，而另一方却不知道，我们将这样的TCP连接称为半打开的。

解决意外中断办法都是利用保活机制。而保活机制分又可以让底层实现也可自己实现。

**1、自己编写心跳包程序**

简单的说也就是在自己的程序中加入一条线程，定时向对端发送数据包，查看是否有ACK，如果有则连接正常，没有的话则连接断开

**2、启动TCP编程里的keepAlive机制**

**一、双方拟定心跳（自实现）**
一般由客户端发送心跳包，服务端并不回应心跳，只是定时轮询判断一下与上次的时间间隔是否超时（超时时间自己设定）。服务器并不主动发送是不想增添服务器的通信量，减少压力。

但这会出现三种情况：

**情况1.**
客户端由于某种网络延迟等原因很久后才发送心跳（它并没有断），这时服务器若利用自身设定的超时判断其已经断开，而后去关闭socket。若客户端有重连机制，则客户端会重新连接。若不确定这种方式是否关闭了原本正常的客户端，则在ShutDown的时候一定要选择send,表示关闭发送通道，服务器还可以接收一下，万一客户端正在发送比较重要的数据呢，是不？

**情况2.**
客户端很久没传心跳，确实是自身断掉了。在其重启之前，服务端已经判断出其超时，并主动close，则四次挥手成功交互。

**情况3.**
客户端很久没传心跳，确实是自身断掉了。在其重启之前，服务端的轮询还未判断出其超时，在未主动close的时候该客户端已经重新连接。

这时候若客户端断开的时候发送了FIN包，则服务端将会处于CLOSE_WAIT状态；

这时候若客户端断开的时候未发送FIN包，则服务端处还是显示ESTABLISHED状态；

而新连接上来的客户端（也就是刚才断掉的重新连上来了）在服务端肯定是ESTABLISHED;这时候就有个问题，若利用轮询还未检测出上条旧连接已经超时（这很正常，timer总有个间隔吧），而在这时，客户端又重复的上演情况3，那么服务端将会出现大量的假的ESTABLISHED连接和CLOSE_WAIT连接。

最终结果就是新的其他客户端无法连接上来，但是利用netstat还是能看到一条连接已经建立，并显示ESTABLISHED，但始终无法进入程序代码。

个人最初感觉导致这种情况是因为假的ESTABLISHED连接和CLOSE_WAIT连接会占用较大的系统资源，程序无法再次创建连接（因为每次我发现这个问题的时候我只连了10个左右客户端却已经有40多条无效连接）。

而最近几天测试却发现有一次程序内只连接了2，3个设备，但是有8条左右的虚连接，此时已经连接不了新客户端了。

这时候我就觉得我想错了，不可能这几条连接就占用了大量连接把，如果说几十条还有可能。但是能肯定的是，这个问题的产生绝对是设备在不停的重启，而服务器这边又是简单的轮询，并不能及时处理，暂时还未能解决。

**二、利用KeepAlive**
其实keepalive的原理就是TCP内嵌的一个心跳包,

以服务器端为例，如果当前server端检测到超过一定时间（默认是 7,200,000 milliseconds，也就是2个小时）没有数据传输，那么会向client端发送一个keep-alive packet（该keep-alive packet就是ACK和当前TCP序列号减一的组合），此时client端应该为以下三种情况之一：

1. client端仍然存在，网络连接状况良好。此时client端会返回一个ACK。server端接收到ACK后重置计时器（复位存活定时器），在2小时后再发送探测。如果2小时内连接上有数据传输，那么在该时间基础上向后推延2个小时。
2. 客户端异常关闭，或是网络断开。在这两种情况下，client端都不会响应。服务器没有收到对其发出探测的响应，并且在一定时间（系统默认为1000 ms）后重复发送keep-alive packet，并且重复发送一定次数（2000 XP 2003 系统默认为5次, Vista后的系统默认为10次）。
3. 客户端曾经崩溃，但已经重启。这种情况下，服务器将会收到对其存活探测的响应，但该响应是一个复位，从而引起服务器对连接的终止。

对于应用程序来说，2小时的空闲时间太长。因此，我们需要手工开启Keepalive功能并设置合理的Keepalive参数。

全局设置可更改`/etc/sysctl.conf`,加上:
`net.ipv4.tcp_keepalive_intvl = 20`
`net.ipv4.tcp_keepalive_probes = 3`
`net.ipv4.tcp_keepalive_time = 60`

在程序中设置如下:

![img](https://pic2.zhimg.com/80/v2-dc678ec50496a74c4b7b402cdbbec385_720w.webp)

在程序中表现为,当tcp检测到对端socket不再可用时(不能发出探测包,或探测包没有收到ACK的响应包),select会返回socket可读,并且在recv时返回-1,同时置上errno为ETIMEDOUT.

### **9. Linux错误信息(errno)列表**

经常出现的错误：

22：参数错误，比如ip地址不合法，没有目标端口等

101：网络不可达，比如不能ping通

111：链接被拒绝，比如目标关闭链接等

115：当链接设置为非阻塞时，目标没有及时应答，返回此错误，socket可以继续使用。比如socket连接

原文地址：https://zhuanlan.zhihu.com/p/550687700

作者：linux