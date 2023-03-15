# 【NO.355】一文弄懂tcp连接中各种状态及故障排查

## **1.TCP网络常用命令**

了解TCP之前，先了解几个命令：

linux查看tcp的状态命令：
1）、netstat -nat 查看TCP各个状态的数量
2）、lsof -i:port 可以检测到打开套接字的状况
3)、 sar -n SOCK 查看tcp创建的连接数
4)、tcpdump -iany tcp port 9000 对tcp端口为9000的进行抓包
5)、tcpdump dst port 9000 -w dump9000.pcap 对tcp目标端口为9000的进行抓包保存pcap文件wireshark分析。
6)、tcpdump tcp port 9000 -n -X -s 0 -w tcp.cap 对tcp/http目标端口为9000的进行抓包保存pcap文件wireshark分析。

网络测试常用命令;

1）ping:检测网络连接的正常与否,主要是测试延时、抖动、丢包率。

但是很多服务器为了防止攻击，一般会关闭对ping的响应。所以ping一般作为测试连通性使用。ping命令后，会接收到对方发送的回馈信息，其中记录着对方的IP地址和TTL。TTL是该字段指定IP包被路由器丢弃之前允许通过的最大网段数量。TTL是IPv4包头的一个8 bit字段。例如IP包在服务器中发送前设置的TTL是64，你使用ping命令后，得到服务器反馈的信息，其中的TTL为56，说明途中一共经过了8道路由器的转发，每经过一个路由，TTL减1。

2）traceroute：raceroute 跟踪数据包到达网络主机所经过的路由工具

traceroute hostname

3）pathping：是一个路由跟踪工具，它将 ping 和 tracert 命令的功能与这两个工具所不提供的其他信息结合起来，综合了二者的功能

pathping [http://www.baidu.com](https://link.zhihu.com/?target=http%3A//www.baidu.com)

4）mtr：以结合ping nslookup tracert 来判断网络的相关特性

\5) nslookup:用于解析域名，一般用来检测本机的DNS设置是否配置正确。

## **2.TCP建立连接三次握手相关状态**

TCP是一个面向连接的协议，所以在连接双方发送数据之前，都需要首先建立一条连接。

**Client连接Server三次握手过程：**

当Client端调用socket函数调用时，相当于Client端产生了一个处于Closed状态的套接字。
( 1) 第一次握手SYN：Client端又调用connect函数调用，系统为Client随机分配一个端口，连同传入connect中的参数(Server的IP和端口)，这就形成了一个连接四元组，客户端发送一个带SYN标志的TCP报文到服务器。这是三次握手过程中的报文1。connect调用让Client端的socket处于SYN_SENT状态，等待服务器确认；SYN：同步序列编号(Synchronize Sequence Numbers)。

( 2)第二次握手SYN+ACK： 服务器收到syn包，必须确认客户的SYN（ack=j+1），同时自己也发送一个SYN包（syn=k），即SYN+ACK包，此时服务器进入SYN_RECV状态；

( 3) 第三次握手ACK：客户端收到服务器的SYN+ACK包，向服务器发送确认包ACK(ack=k+1)，此包发送完毕，客户器和客务器进入ESTABLISHED状态，完成三次握手。连接已经可以进行读写操作。

一个完整的三次握手也就是： 请求---应答---再次确认。

TCP协议通过三个报文段完成连接的建立，这个过程称为三次握手(three-way handshake)，过程如下图所示。

![img](https://pic4.zhimg.com/80/v2-5bd65e5e102cd2041a424d20f5040c57_720w.webp)

Server对应的函数接口：

当Server端调用socket函数调用时，相当于Server端产生了一个处于Closed状态的监听套接字
Server端调用bind操作，将监听套接字与指定的地址和端口关联，然后又调用listen函数，系统会为其分配未完成队列和完成队列，此时的监听套接字可以接受Client的连接，监听套接字状态处于LISTEN状态。
当Server端调用accept操作时，会从完成队列中取出一个已经完成的client连接，同时在server这段会产生一个会话套接字，用于和client端套接字的通信，这个会话套接字的状态是ESTABLISH。

从图中可以看出，当客户端调用connect时，触发了连接请求，向服务器发送了SYN J包，这时connect进入阻塞状态；服务器监听到连接请求，即收到SYN J包，调用accept函数接收请求向客户端发送SYN K ，ACK J+1，这时accept进入阻塞状态；客户端收到服务器的SYN K ，ACK J+1之后，这时connect返回，并对SYN K进行确认；服务器收到ACK K+1时，accept返回，至此三次握手完毕，连接建立。

我们可以通过wireshark抓包，可以清晰看到查看具体的流程：

第一次握手：syn的seq= 0
第二次握手：服务器收到syn包，必须确认客户的SYN（ACK=j+1=1）=，同时自己也发送一个SYN包（syn=0）

第三次握手：客户端收到服务器的SYN+ACK包，向服务器发送确认包ACK(ack=k+1)

![img](https://pic2.zhimg.com/80/v2-e5e5bb47ad0a3f2db7ef4ddf690d4509_720w.webp)

2、建立连接的具体状态：

1)、LISTENING：侦听来自远方的TCP端口的连接请求.

首先服务端需要打开一个socket进行监听，状态为LISTEN。

有提供某种服务才会处于LISTENING状态，TCP状态变化就是某个端口的状态变化，提供一个服务就打开一个端口，例如：提供www服务默认开的是80端口，提供ftp服务默认的端口为21，当提供的服务没有被连接时就处于LISTENING状态。FTP服务启动后首先处于侦听（LISTENING）状态。处于侦听LISTENING状态时，该端口是开放的，等待连接，但还没有被连接。就像你房子的门已经敞开的，但还没有人进来。
看LISTENING状态最主要的是看本机开了哪些端口，这些端口都是哪个程序开的，关闭不必要的端口是保证安全的一个非常重要的方面，服务端口都对应一个服务（应用程序），停止该服务就关闭了该端口，例如要关闭21端口只要停止IIS服务中的FTP服务即可。关于这方面的知识请参阅其它文章。
如果你不幸中了服务端口的木马，木马也开个端口处于LISTENING状态。

2)、SYN-SENT：客户端SYN_SENT状态

再发送连接请求后等待匹配的连接请求:客户端通过应用程序调用connect进行active open.于是客户端tcp发送一个SYN以请求建立一个连接.之后状态置为SYN_SENT. /*The socket is actively attempting to establish a connection. 在发送连接请求后等待匹配的连接请求 */
当请求连接时客户端首先要发送同步信号给要访问的机器，此时状态为SYN_SENT，如果连接成功了就变为ESTABLISHED，正常情况下SYN_SENT状态非常短暂。例如要访问网站[http://www.baidu.com](https://link.zhihu.com/?target=http%3A//www.baidu.com),如果是正常连接的话，用TCPView观察IEXPLORE.EXE（IE）建立的连接会发现很快从SYN_SENT变为ESTABLISHED，表示连接成功。SYN_SENT状态快的也许看不到。
如果发现有很多SYN_SENT出现，那一般有这么几种情况，一是你要访问的网站不存在或线路不好，二是用扫描软件扫描一个网段的机器，也会出出现很多SYN_SENT，另外就是可能中了病毒了，例如中了"冲击波"，病毒发作时会扫描其它机器，这样会有很多SYN_SENT出现。


3)、SYN-RECEIVED：服务器端握手状态SYN_RCVD

再收到和发送一个连接请求后等待对方对连接请求的确认

当服务器收到客户端发送的同步信号时，将标志位ACK和SYN置1发送给客户端，此时服务器端处于SYN_RCVD状态，如果连接成功了就变为ESTABLISHED，正常情况下SYN_RCVD状态非常短暂。
如果发现有很多SYN_RCVD状态，那你的机器有可能被SYN Flood的DoS(拒绝服务攻击)攻击了。

SYN Flood的攻击原理是：

在进行三次握手时，攻击软件向被攻击的服务器发送SYN连接请求（握手的第一步），但是这个地址是伪造的，如攻击软件随机伪造了51.133.163.104、65.158.99.152等等地址。服务器在收到连接请求时将标志位ACK和SYN置1发送给客户端（握手的第二步），但是这些客户端的IP地址都是伪造的，服务器根本找不到客户机，也就是说握手的第三步不可能完成。

这种情况下服务器端一般会重试（再次发送SYN+ACK给客户端）并等待一段时间后丢弃这个未完成的连接，这段时间的长度我们称为SYN Timeout，一般来说这个时间是分钟的数量级（大约为30秒-2分钟）；一个用户出现异常导致服务器的一个线程等待1分钟并不是什么很大的问题，但如果有一个恶意的攻击者大量模拟这种情况，服务器端将为了维护一个非常大的半连接列表而消耗非常多的资源----数以万计的半连接，即使是简单的保存并遍历也会消耗非常多的CPU时间和内存，何况还要不断对这个列表中的IP进行SYN+ACK的重试。此时从正常客户的角度看来，服务器失去响应，这种情况我们称做：服务器端受到了SYN Flood攻击（SYN洪水攻击）


4)、ESTABLISHED：代表一个打开的连接。

ESTABLISHED状态是表示两台机器正在传输数据，观察这个状态最主要的就是看哪个程序正在处于ESTABLISHED状态。

服务器出现很多ESTABLISHED状态： netstat -nat |grep 9502或者使用lsof -i:9502可以检测到。

当客户端未主动close的时候就断开连接：即客户端发送的FIN丢失或未发送:
这时候若客户端断开的时候发送了FIN包，则服务端将会处于CLOSE_WAIT状态；
这时候若客户端断开的时候未发送FIN包，则服务端处还是显示ESTABLISHED状态；
结果客户端重新连接服务器。
而新连接上来的客户端（也就是刚才断掉的重新连上来了）在服务端肯定是ESTABLISHED; 如果客户端重复的上演这种情况，那么服务端将会出现大量的假的ESTABLISHED连接和CLOSE_WAIT连接。
最终结果就是新的其他客户端无法连接上来，但是利用netstat还是能看到一条连接已经建立，并显示ESTABLISHED，但始终无法进入程序代码。

## **3. TCP关闭四次握手的相关状态**

　 由于TCP连接是全双工的，因此每个方向都必须单独进行关闭。这原则是当一方完成它的数据发送任务后就能发送一个FIN来终止这个方向的连接。收到一个 FIN只意味着这一方向上没有数据流动，一个TCP连接在收到一个FIN后仍能发送数据。首先进行关闭的一方将执行主动关闭，而另一方执行被动关闭。

### **3.1 四次握手关闭连接的过程**

建立一个连接需要三次握手，而终止一个连接要经过四次握手，这是由TCP的半关闭(half-close)造成的，如图：

![img](https://pic1.zhimg.com/80/v2-fef04d9d627497cb0f072f5600c063cc_720w.webp)

调用过程和对应函数接口：

- \1) 客户端发送一个FIN: 用来关闭客户A到服务器B的数据传送,当client想要关闭它与server之间的连接。client（某个应用进程）首先调用close主动关闭连接，这时TCP发送一个FIN M；client端处于FIN_WAIT1状态。
- \2) 服务器确认客户端FIN: 当server端接收到FIN M之后，执行被动关闭。对这个FIN进行确认，返回给client ACK。确认序号为收到的序号加1（报文段5）。和SYN一样，一个FIN将占用一个序号。当server端返回给client ACK后，client处于FIN_WAIT2状态，server处于CLOSE_WAIT状态。它的接收也作为文件结束符传递给应用进程，因为FIN的接收 意味着应用进程在相应的连接上再也接收不到额外数据；
- \3) 服务器B发送一个FIN关闭与客户端A的连接: 一段时间之后，当server端检测到client端的关闭操作(read返回为0)。接收到文件结束符的server端调用close关闭它的socket。这导致server端的TCP也发送一个FIN N；此时server的状态为LAST_ACK。
- \4) 客户端A发回ACK报文确认: 当client收到来自server的FIN后 。 client端的套接字处于TIME_WAIT状态，它会向server端再发送一个ack确认，此时server端收到ack确认后，此套接字处于CLOSED状态。

这样每个方向上都有一个FIN和ACK。

### **3.2 四次握手关闭连接的具体状态**

1）FIN-WAIT-1：

等待远程TCP连接中断请求，或先前的连接中断请求的确认

主动关闭(active close)端应用程序调用close，于是其TCP发出FIN请求主动关闭连接，之后进入FIN_WAIT1状态./* The socket is closed, and the connection is shutting down. 等待远程TCP的连接中断请求，或先前的连接中断请求的确认 */

如果服务器出现shutdown再重启，使用netstat -nat查看，就会看到很多FIN-WAIT-1的状态。就是因为服务器当前有很多客户端连接，直接关闭服务器后，无法接收到客户端的ACK。


2）FIN-WAIT-2：从远程TCP等待连接中断请求

主动关闭端接到ACK后，就进入了FIN-WAIT-2 ./* Connection is closed, and the socket is waiting for a shutdown from the remote end. 从远程TCP等待连接中断请求 */

这就是著名的半关闭的状态了，这是在关闭连接时，客户端和服务器两次握手之后的状态。在这个状态下，应用程序还有接受数据的能力，但是已经无法发送数据，但是也有一种可能是，客户端一直处于FIN_WAIT_2状态，而服务器则一直处于WAIT_CLOSE状态，而直到应用层来决定关闭这个状态。


3）CLOSE-WAIT：等待从本地用户发来的连接中断请求

被动关闭(passive close)端TCP接到FIN后，就发出ACK以回应FIN请求(它的接收也作为文件结束符传递给上层应用程序),并进入CLOSE_WAIT. /* The remote end has shut down, waiting for the socket to close. 等待从本地用户发来的连接中断请求 */


4）CLOSING：等待远程TCP对连接中断的确认

比较少见./* Both sockets are shut down but we still don't have all our data sent. 等待远程TCP对连接中断的确认 */

实际情况中应该是很少见，属于一种比较罕见的例外状态。正常情况下，当一方发送FIN报文后，按理来说是应该先收到（或同时收到）对方的ACK报文，再收到对方的FIN报文。但是CLOSING状态表示一方发送FIN报文后，并没有收到对方的ACK报文，反而却也收到了对方的FIN报文。什么情况下会出现此种情况呢？
有两种情况可能导致这种状态：
其一，如果双方几乎在同时关闭连接，那么就可能出现双方同时发送FIN包的情况；
其二，如果ACK包丢失而对方的FIN包很快发出，也会出现FIN先于ACK到达。

4）LAST-ACK：等待原来的发向远程TCP的连接中断请求的确认

被动关闭端一段时间后，接收到文件结束符的应用程序将调用CLOSE关闭连接。这导致它的TCP也发送一个 FIN,等待对方的ACK.就进入了LAST-ACK . /* The remote end has shut down, and the socket is closed. Waiting for acknowledgement. 等待原来发向远程TCP的连接中断请求的确认 */

使用并发压力测试的时候，突然断开压力测试客户端，服务器会看到很多LAST-ACK。


5）TIME-WAIT：等待足够的时间以确保远程TCP接收到连接中断请求的确认

在主动关闭端接收到FIN后，TCP就发送ACK包，并进入TIME-WAIT状态。/* The socket is waiting after close to handle packets still in the network.等待足够的时间以确保远程TCP接收到连接中断请求的确认 */

TIME_WAIT等待状态，这个状态又叫做2MSL状态，说的是在TIME_WAIT2发送了最后一个ACK数据报以后，要进入TIME_WAIT状态，这个状态是防止最后一次握手的数据报没有传送到对方那里而准备的（注意这不是四次握手，这是第四次握手的保险状态）。这个状态在很大程度上保证了双方都可以正常结束。

由于插口的2MSL状态（插口是IP和端口对的意思，socket），使得应用程序在2MSL时间内是无法再次使用同一个插口的，对于客户程序还好一些，但是对于服务程序，例如httpd，它总是要使用同一个端口来进行服务，而在2MSL时间内，启动httpd就会出现错误（插口被使用）。为了避免这个错误，服务器给出了一个平静时间的概念，这是说在2MSL时间内，虽然可以重新启动服务器，但是这个服务器还是要平静的等待2MSL时间的过去才能进行下一次连接。

6）CLOSED：没有任何连接状态

被动关闭端在接受到ACK包后，就进入了closed的状态。连接结束./* The socket is not being used. 没有任何连接状态 */

## **4.TCP状态迁移路线图**

client/server两条路线讲述TCP状态迁移路线图：

![img](https://pic4.zhimg.com/80/v2-6d56c6b3173a4c20f645190f15b6b0cf_720w.webp)

这是一个看起来比较复杂的状态迁移图，因为它包含了两个部分---服务器的状态迁移和客户端的状态迁移，如果从某一个角度出发来看这个图，就会清晰许多，这里面的服务器和客户端都不是绝对的，发送数据的就是客户端，接受数据的就是服务器。

客户端应用程序的状态迁移图

客户端的状态可以用如下的流程来表示：

CLOSED->SYN_SENT->ESTABLISHED->FIN_WAIT_1->FIN_WAIT_2->TIME_WAIT->CLOSED

以上流程是在程序正常的情况下应该有的流程，从书中的图中可以看到，在建立连接时，当客户端收到SYN报文的ACK以后，客户端就打开了数据交互地连接。而结束连接则通常是客户端主动结束的，客户端结束应用程序以后，需要经历FIN_WAIT_1，FIN_WAIT_2等状态，这些状态的迁移就是前面提到的结束连接的四次握手。

服务器的状态迁移图

服务器的状态可以用如下的流程来表示：

CLOSED->LISTEN->SYN收到->ESTABLISHED->CLOSE_WAIT->LAST_ACK->CLOSED

在建立连接的时候，服务器端是在第三次握手之后才进入数据交互状态，而关闭连接则是在关闭连接的第二次握手以后（注意不是第四次）。而关闭以后还要等待客户端给出最后的ACK包才能进入初始的状态。

其他状态迁移

还有一些其他的状态迁移，这些状态迁移针对服务器和客户端两方面的总结如下

LISTEN->SYN_SENT，对于这个解释就很简单了，服务器有时候也要打开连接的嘛。

SYN_SENT->SYN收到，服务器和客户端在SYN_SENT状态下如果收到SYN数据报，则都需要发送SYN的ACK数据报并把自己的状态调整到SYN收到状态，准备进入ESTABLISHED

SYN_SENT->CLOSED，在发送超时的情况下，会返回到CLOSED状态。

SYN_收到->LISTEN，如果受到RST包，会返回到LISTEN状态。

SYN_收到->FIN_WAIT_1，这个迁移是说，可以不用到ESTABLISHED状态，而可以直接跳转到FIN_WAIT_1状态并等待关闭。

![img](https://pic2.zhimg.com/80/v2-c3f60b5fa9c26d76c5e38a379186f9d5_720w.webp)

怎样牢牢地将这张图刻在脑中呢？那么你就一定要对这张图的每一个状态，及转换的过程有深刻的认识，不能只停留在一知半解之中。下面对这张图的11种状态详细解析一下，以便加强记忆！不过在这之前，先回顾一下TCP建立连接的三次握手过程，以及关闭连接的四次握手过程。

## **5.具体问题**

1．为什么建立连接协议是三次握手，而关闭连接却是四次握手呢？

> 这是因为服务端的LISTEN状态下的SOCKET当收到SYN报文的建连请求后，它可以把ACK和SYN（ACK起应答作用，而SYN起同步作用）放在一个报文里来发送。但关闭连接时，当收到对方的FIN报文通知时，它仅仅表示对方没有数据发送给你了；但未必你所有的数据都全部发送给对方了，所以你可以未必会马上会关闭SOCKET,也即你可能还需要发送一些数据给对方之后，再发送FIN报文给对方来表示你同意现在可以关闭连接了，所以它这里的ACK报文和FIN报文多数情况下都是分开发送的。

![img](https://pic4.zhimg.com/80/v2-78914591d205d84098862bcefdfeb2ff_720w.webp)

2、什么是2MSL

> MSL是Maximum Segment Lifetime英文的缩写，中文可以译为“报文最大生存时间”，他是任何报文在网络上存在的最长时间，超过这个时间报文将被丢弃。因为tcp报文（segment）是ip数据报（datagram）的数据部分，具体称谓请参见《数据在网络各层中的称呼》一文，而ip头中有一个TTL域，TTL是time to live的缩写，中文可以译为“生存时间”，这个生存时间是由源主机设置初始值但不是存的具体时间，而是存储了一个ip数据报可以经过的最大路由数，每经过一个处理他的路由器此值就减1，当此值为0则数据报将被丢弃，同时发送ICMP报文通知源主机。RFC 793中规定MSL为2分钟，实际应用中常用的是30秒，1分钟和2分钟等。
> 2MSL即两倍的MSL，TCP的TIME_WAIT状态也称为2MSL等待状态，当TCP的一端发起主动关闭，在发出最后一个ACK包后，即第3次握手完成后发送了第四次握手的ACK包后就进入了TIME_WAIT状态，必须在此状态上停留两倍的MSL时间，等待2MSL时间主要目的是怕最后一个ACK包对方没收到，那么对方在超时后将重发第三次握手的FIN包，主动关闭端接到重发的FIN包后可以再发一个ACK应答包。在TIME_WAIT状态时两端的端口不能使用，要等到2MSL时间结束才可继续使用。当连接处于2MSL等待阶段时任何迟到的报文段都将被丢弃。不过在实际应用中可以通过设置SO_REUSEADDR选项达到不必等待2MSL时间结束再使用此端口。
> 这是因为虽然双方都同意关闭连接了，而且握手的4个报文也都协调和发送完毕，按理可以直接回到CLOSED状态（就好比从SYN_SEND状态到ESTABLISH状态那样）：
> 一方面是可靠的实现TCP全双工连接的终止，也就是当最后的ACK丢失后，被动关闭端会重发FIN，因此主动关闭端需要维持状态信息，以允许它重新发送最终的ACK。
> 另一方面，但是因为我们必须要假想网络是不可靠的，你无法保证你最后发送的ACK报文会一定被对方收到，因此对方处于LAST_ACK状态下的SOCKET可能会因为超时未收到ACK报文，而重发FIN报文，所以这个TIME_WAIT状态的作用就是用来重发可能丢失的ACK报文。
> TCP在2MSL等待期间，定义这个连接(4元组)不能再使用，任何迟到的报文都会丢弃。设想如果没有2MSL的限制，恰好新到的连接正好满足原先的4元组，这时候连接就可能接收到网络上的延迟报文就可能干扰最新建立的连接。

3．为什么TIME_WAIT状态还需要等2MSL后才能返回到CLOSED状态？

> 第一，保证可靠的实现TCP全双工链接的终止：为了保证主动端A发送的最后一个ACK报文能够到达被动段B，确保被动端能进入CLOSED状态。
> 对照上图，在四次挥手协议中，当B向A发送Fin+Ack后，A就需要向B发送ACK+Seq报文，A这时候就处于TIME_WAIT 状态，但是这个报文ACK有可能会发送失败而丢失，因而使处在LAST-ACK状态的B收不到对已发送的FIN+ACK报文段的确认。B会超时重传这个FIN+ACK报文段，而A就能在2MSL时间内收到这个重传的FIN+ACK报文段。如果A在TIME-WAIT状态不等待一段时间，而是在发送完ACK报文段后就立即释放连接，就无法收到B重传的FIN+ACK报文段，因而也不会再发送一次确认报文段。这样B就无法按照正常的步骤进入CLOSED状态。
>
> 第二，防止已失效的连接请求报文段出现在本连接中。A在发送完ACK报文段后，再经过2MSL时间，就可以使本连接持续的时间所产生的所有报文段都从网络中消失。这样就可以使下一个新的连接中不会出现这种旧的连接请求的报文段。
> 假设在A的XXX1端口和B的80端口之间有一个TCP连接。我们关闭这个连接，过一段时间后在相同的IP地址和端口建立另一个连接。后一个链接成为前一个的化身。因为它们的IP地址和端口号都相同。TCP必须防止来自某一个连接的老的重复分组在连 接已经终止后再现，从而被误解成属于同一链接的某一个某一个新的化身。为做到这一点，TCP将不给处于TIME_WAIT状态的链接发起新的化身。既然 TIME_WAIT状态的持续时间是MSL的2倍，这就足以让某个方向上的分组最多存活msl秒即被丢弃，另一个方向上的应答最多存活msl秒也被丢弃。 通过实施这个规则，我们就能保证每成功建立一个TCP连接时。来自该链接先前化身的重复分组都已经在网络中消逝了。

4、解决linux发现系统存在大量TIME_WAIT状态

> 如果linux发现系统存在大量TIME_WAIT状态的连接，可以通过调整内核参数解决：vi /etc/sysctl.conf 加入以下内容：
> net.ipv4.tcp_max_tw_buckets=5000 #TIME-WAIT Socket 最大数量
> \#注意：不建议开启該设置，NAT 模式下可能引起连接 RST
> net.ipv4.tcp_tw_reuse = 1 #表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；
> net.ipv4.tcp_tw_recycle = 1 #表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭。
> 然后执行 /sbin/sysctl -p 让参数生效。

## **6.TCP的FLAGS说明**

在TCP层，有个FLAGS字段，这个字段有以下几个标识：SYN, FIN, ACK, PSH, RST, URG.
其中，对于我们日常的分析有用的就是前面的五个字段。
一、字段含义：
1、SYN表示建立连接：

步序列编号(Synchronize Sequence Numbers)栏有效。该标志仅在三次握手建立TCP连接时有效。它提示TCP连接的服务端检查序列编号，该序列编号为TCP连接初始端(一般是客户端)的初始序列编号。在这里，可以把TCP序列编号看作是一个范围从0到4，294，967，295的32位计数器。通过TCP连接交换的数据中每一个字节都经过序列编号。在TCP报头中的序列编号栏包括了TCP分段中第一个字节的序列编号。

2、FIN表示关闭连接：
3、ACK表示响应：

确认编号(Acknowledgement Number)栏有效。大多数情况下该标志位是置位的。TCP报头内的确认编号栏内包含的确认编号(w+1，Figure-1)为下一个预期的序列编号，同时提示远端系统已经成功接收所有数据。

4、PSH表示有DATA数据传输：


5、RST表示连接重置： 复位标志有效。用于复位相应的TCP连接。

一、字段组合含义：

![img](https://pic2.zhimg.com/80/v2-c9ed1c439eec32428d230c952987ff49_720w.webp)

其中，ACK是可能与SYN，FIN等同时使用的，比如SYN和ACK可能同时为1，它表示的就是建立连接之后的响应，
如果只是单个的一个SYN，它表示的只是建立连接。
TCP的几次握手就是通过这样的ACK表现出来的。
但SYN与FIN是不会同时为1的，因为前者表示的是建立连接，而后者表示的是断开连接。
RST一般是在FIN之后才会出现为1的情况，表示的是连接重置。
一般地，当出现FIN包或RST包时，我们便认为客户端与服务器端断开了连接；

![img](https://pic1.zhimg.com/80/v2-27c560ccbe6b9dc0596438aa97fc21b4_720w.webp)

RST与ACK标志位都置一了，并且具有ACK number，非常明显，这个报文在释放TCP连接的同时，完成了对前面已接收报文的确认。

而当出现SYN和SYN＋ACK包时，我们认为客户端与服务器建立了一个连接。
PSH为1的情况，一般只出现在 DATA内容不为0的包中，也就是说PSH为1表示的是有真正的TCP数据包内容被传递。
TCP的连接建立和连接关闭，都是通过请求－响应的模式完成的。

## **7.TCP通信中服务器处理客户端意外断开**

如果TCP连接被对方正常关闭，也就是说，对方是正确地调用了closesocket(s)或者shutdown(s)的话，那么上面的Recv或Send调用就能马上返回，并且报错。这是由于close socket(s)或者shutdown(s)有个正常的关闭过程，会告诉对方“TCP连接已经关闭，你不需要再发送或者接受消息了”。

但是，如果意外断开，客户端（3g的移动设备）并没有正常关闭socket。双方并未按照协议上的四次挥手去断开连接。

那么这时候正在执行Recv或Send操作的一方就会因为没有任何连接中断的通知而一直等待下去，也就是会被长时间卡住。

像这种如果一方已经关闭或异常终止连接，而另一方却不知道，我们将这样的TCP连接称为半打开的。

解决意外中断办法都是利用保活机制。而保活机制分又可以让底层实现也可自己实现。

1、自己编写心跳包程序

简单的说也就是在自己的程序中加入一条线程，定时向对端发送数据包，查看是否有ACK，如果有则连接正常，没有的话则连接断开

2、启动TCP编程里的keepAlive机制

一、双方拟定心跳（自实现）

一般由客户端发送心跳包，服务端并不回应心跳，只是定时轮询判断一下与上次的时间间隔是否超时（超时时间自己设定）。服务器并不主动发送是不想增添服务器的通信量，减少压力。

但这会出现三种情况：

情况1.

客户端由于某种网络延迟等原因很久后才发送心跳（它并没有断），这时服务器若利用自身设定的超时判断其已经断开，而后去关闭socket。若客户端有重连机制，则客户端会重新连接。若不确定这种方式是否关闭了原本正常的客户端，则在ShutDown的时候一定要选择send,表示关闭发送通道，服务器还可以接收一下，万一客户端正在发送比较重要的数据呢，是不？

情况2.

客户端很久没传心跳，确实是自身断掉了。在其重启之前，服务端已经判断出其超时，并主动close，则四次挥手成功交互。

情况3.

客户端很久没传心跳，确实是自身断掉了。在其重启之前，服务端的轮询还未判断出其超时，在未主动close的时候该客户端已经重新连接。

这时候若客户端断开的时候发送了FIN包，则服务端将会处于CLOSE_WAIT状态；

这时候若客户端断开的时候未发送FIN包，则服务端处还是显示ESTABLISHED状态；

而新连接上来的客户端（也就是刚才断掉的重新连上来了）在服务端肯定是ESTABLISHED;这时候就有个问题，若利用轮询还未检测出上条旧连接已经超时（这很正常，timer总有个间隔吧），而在这时，客户端又重复的上演情况3，那么服务端将会出现大量的假的ESTABLISHED连接和CLOSE_WAIT连接。

最终结果就是新的其他客户端无法连接上来，但是利用netstat还是能看到一条连接已经建立，并显示ESTABLISHED，但始终无法进入程序代码。个人最初感觉导致这种情况是因为假的ESTABLISHED连接和CLOSE_WAIT连接会占用较大的系统资源，程序无法再次创建连接（因为每次我发现这个问题的时候我只连了10个左右客户端却已经有40多条无效连接）。而最近几天测试却发现有一次程序内只连接了2，3个设备，但是有8条左右的虚连接，此时已经连接不了新客户端了。这时候我就觉得我想错了，不可能这几条连接就占用了大量连接把，如果说几十条还有可能。但是能肯定的是，这个问题的产生绝对是设备在不停的重启，而服务器这边又是简单的轮询，并不能及时处理，暂时还未能解决。

二、利用KeepAlive

其实keepalive的原理就是TCP内嵌的一个心跳包,

以服务器端为例，如果当前server端检测到超过一定时间（默认是 7,200,000 milliseconds，也就是2个小时）没有数据传输，那么会向client端发送一个keep-alive packet（该keep-alive packet就是ACK和当前TCP序列号减一的组合），此时client端应该为以下三种情况之一：

\1. client端仍然存在，网络连接状况良好。此时client端会返回一个ACK。server端接收到ACK后重置计时器（复位存活定时器），在2小时后再发送探测。如果2小时内连接上有数据传输，那么在该时间基础上向后推延2个小时。

\2. 客户端异常关闭，或是网络断开。在这两种情况下，client端都不会响应。服务器没有收到对其发出探测的响应，并且在一定时间（系统默认为1000 ms）后重复发送keep-alive packet，并且重复发送一定次数（2000 XP 2003 系统默认为5次, Vista后的系统默认为10次）。

\3. 客户端曾经崩溃，但已经重启。这种情况下，服务器将会收到对其存活探测的响应，但该响应是一个复位，从而引起服务器对连接的终止。

对于应用程序来说，2小时的空闲时间太长。因此，我们需要手工开启Keepalive功能并设置合理的Keepalive参数。

全局设置可更改/etc/sysctl.conf,加上:
net.ipv4.tcp_keepalive_intvl = 20
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_time = 60

在程序中设置如下:

```text
#include <sys/socket.h>
 #include <netinet/in.h>
 #include <arpa/inet.h>
 #include <sys/types.h>
 #include <netinet/tcp.h>
 
 int keepAlive = 1; // 开启keepalive属性
 int keepIdle = 60; // 如该连接在60秒内没有任何数据往来,则进行探测 
 int keepInterval = 5; // 探测时发包的时间间隔为5 秒
 int keepCount = 3; // 探测尝试的次数.如果第1次探测包就收到响应了,则后2次的不再发.
 
  setsockopt(rs, SOL_SOCKET, SO_KEEPALIVE, (void *)&keepAlive, sizeof(keepAlive));
  setsockopt(rs, SOL_TCP, TCP_KEEPIDLE, (void*)&keepIdle, sizeof(keepIdle));
  setsockopt(rs, SOL_TCP, TCP_KEEPINTVL, (void *)&keepInterval, sizeof(keepInterval));
  setsockopt(rs, SOL_TCP, TCP_KEEPCNT, (void *)&keepCount, sizeof(keepCount));
```

在程序中表现为,当tcp检测到对端socket不再可用时(不能发出探测包,或探测包没有收到ACK的响应包),select会返回socket可读,并且在recv时返回-1,同时置上errno为ETIMEDOUT.

## **8.Linux错误信息(errno)列表**

经常出现的错误：

22：参数错误，比如ip地址不合法，没有目标端口等

101：网络不可达，比如不能ping通

111：链接被拒绝，比如目标关闭链接等

115：当链接设置为非阻塞时，目标没有及时应答，返回此错误，socket可以继续使用。比如socket连接

附录：Linux的错误码表（errno table)

_ 124 EMEDIUMTYPE_ Wrong medium type
_ 123 ENOMEDIUM__ No medium found
_ 122 EDQUOT___ Disk quota exceeded
_ 121 EREMOTEIO__ Remote I/O error
_ 120 EISNAM___ Is a named type file
_ 119 ENAVAIL___ No XENIX semaphores available
_ 118 ENOTNAM___ Not a XENIX named type file
_ 117 EUCLEAN___ Structure needs cleaning
_ 116 ESTALE___ Stale NFS file handle
_ 115 EINPROGRESS +Operation now in progress

操作正在进行中。一个阻塞的操作正在执行。


_ 114 EALREADY__ Operation already in progress
_ 113 EHOSTUNREACH No route to host
_ 112 EHOSTDOWN__ Host is down
_ 111 ECONNREFUSED Connection refused

1、拒绝连接。一般发生在连接建立时。

拔服务器端网线测试，客户端设置keep alive时，recv较快返回0， 先收到ECONNREFUSED (Connection refused)错误码，其后都是ETIMEOUT。

2、an error returned from connect(), so it can only occur in a client (if a client is defined as the party that initiates the connection


_ 110 ETIMEDOUT_ +Connection timed out
_ 109 ETOOMANYREFS Too many references: cannot splice
_ 108 ESHUTDOWN__ Cannot send after transport endpoint shutdown
_ 107 ENOTCONN__ Transport endpoint is not connected

在一个没有建立连接的socket上，进行read，write操作会返回这个错误。出错的原因是socket没有标识地址。Setsoc也可能会出错。

还有一种情况就是收到对方发送过来的RST包,系统已经确认连接被断开了。


_ 106 EISCONN___ Transport endpoint is already connected

一般是socket客户端已经连接了，但是调用connect，会引起这个错误。


_ 105 ENOBUFS___ No buffer space available
_ 104 ECONNRESET_ Connection reset by peer

连接被远程主机关闭。有以下几种原因：远程主机停止服务，重新启动;当在执行某些操作时遇到失败，因为设置了“keep alive”选项，连接被关闭，一般与ENETRESET一起出现。

1、在客户端服务器程序中，客户端异常退出，并没有回收关闭相关的资源，服务器端会先收到ECONNRESET错误，然后收到EPIPE错误。

2、连接被远程主机关闭。有以下几种原因：远程主机停止服务，重新启动;当在执行某些操作时遇到失败，因为设置了“keep alive”选项，连接被关闭，一般与ENETRESET一起出现。

3、远程端执行了一个“hard”或者“abortive”的关闭。应用程序应该关闭socket，因为它不再可用。当执行在一个UDP socket上时，这个错误表明前一个send操作返回一个ICMP“port unreachable”信息。

4、如果client关闭连接,server端的select并不出错(不返回-1,使用select对唯一一个socket进行non- blocking检测),但是写该socket就会出错,用的是send.错误号:ECONNRESET.读(recv)socket并没有返回错误。

5、该错误被描述为“connection reset by peer”，即“对方复位连接”，这种情况一般发生在服务进程较客户进程提前终止。

主动关闭调用过程如下：

![img](https://pic2.zhimg.com/80/v2-1cabceadd31e988518e6c41a0deb2619_720w.webp)

服务器端主动关闭：

1）当服务器的服务因为某种原因，进程提前终止时会向客户 TCP 发送 FIN 分节，服务器端处于FIN_WAIT1状态。

2）客户TCP回应ACK后，服务TCP将转入FIN_WAIT2状态。

3）此时如果客户进程没有处理该 FIN （如阻塞在其它调用上而没有关闭 Socket 时），则客户TCP将处于CLOSE_WAIT状态。

4）当客户进程再次向 FIN_WAIT2 状态的服务 TCP 发送数据时，则服务 TCP 将立刻响应 RST。

一般来说，这种情况还可以会引发另外的应用程序异常，客户进程在发送完数据后，往往会等待从网络IO接收数据，很典型的如 read 或 readline 调用，此时由于执行时序的原因，如果该调用发生在RST分节收到前执行的话，那么结果是客户进程会得到一个非预期的 EOF 错误。此时一般会输出“server terminated prematurely”－“服务器过早终止”错误。


_ 103 ECONNABORTED Software caused connection abort

1、软件导致的连接取消。一个已经建立的连接被host方的软件取消，原因可能是数据传输超时或者是协议错误。

2、该错误被描述为“software caused connection abort”，即“软件引起的连接中止”。原因在于当服务和客户进程在完成用于 TCP 连接的“三次握手”后，客户 TCP 却发送了一个 RST （复位）分节，在服务进程看来，就在该连接已由 TCP 排队，等着服务进程调用 accept 的时候 RST 却到达了。POSIX 规定此时的 errno 值必须 ECONNABORTED。源自 Berkeley 的实现完全在内核中处理中止的连接，服务进程将永远不知道该中止的发生。服务器进程一般可以忽略该错误，直接再次调用accept。

当TCP协议接收到RST数据段，表示连接出现了某种错误，函数read将以错误返回，错误类型为ECONNERESET。并且以后所有在这个套接字上的读操作均返回错误。错误返回时返回值小于0。
_ 102 ENETRESET__ Network dropped connection on reset

网络重置时丢失连接。

由于设置了"keep-alive"选项，探测到一个错误，连接被中断。在一个已经失败的连接上试图使用setsockopt操作，也会返回这个错误。
_ 101 ENETUNREACH_ Network is unreachable

网络不可达。Socket试图操作一个不可达的网络。这意味着local的软件知道没有路由到达远程的host。
_ 100 ENETDOWN__ Network is down
_ 99 EADDRNOTAVAIL Cannot assign requested address
_ 98 EADDRINUSE_ Address already in use
_ 97 EAFNOSUPPORT Address family not supported by protocol
_ 96 EPFNOSUPPORT Protocol family not supported
_ 95 EOPNOTSUPP_ Operation not supported
_ 94 ESOCKTNOSUPPORT Socket type not supported

Socket类型不支持。指定的socket类型在其address family中不支持。如可选选中选项SOCK_RAW，但实现并不支持SOCK_RAW sockets。


_ 93 EPROTONOSUPPORT Protocol not supported

不支持的协议。系统中没有安装标识的协议，或者是没有实现。如函数需要SOCK_DGRAM socket，但是标识了stream protocol.。


_ 92 ENOPROTOOPT_ Protocol not available

该错误不是一个 Socket 连接相关的错误。errno 给出该值可能由于，通过 getsockopt 系统调用来获得一个套接字的当前选项状态时，如果发现了系统不支持的选项参数就会引发该错误。
_ 91 EPROTOTYPE_ Protocol wrong type for socket

协议类型错误。标识了协议的Socket函数在不支持的socket上进行操作。如ARPA Internet

UDP协议不能被标识为SOCK_STREAM socket类型。
_ 90 EMSGSIZE__ +Message too long

消息体太长。

发送到socket上的一个数据包大小比内部的消息缓冲区大，或者超过别的网络限制，或是用来接收数据包的缓冲区比数据包本身小。
_ 89 EDESTADDRREQ Destination address required

需要提供目的地址。

在一个socket上的操作需要提供地址。如往一个ADDR_ANY 地址上进行sendto操作会返回这个错误。
_ 88 ENOTSOCK__ Socket operation on non-socket

在非socket上执行socket操作。
_ 87 EUSERS___ Too many users
_ 86 ESTRPIPE__ Streams pipe error
_ 85 ERESTART__ Interrupted system call should be restarted
_ 84 EILSEQ___ Invalid or incomplete multibyte or wide character
_ 83 ELIBEXEC__ Cannot exec a shared library directly
_ 82 ELIBMAX___ Attempting to link in too many shared libraries
_ 81 ELIBSCN___ .lib section in a.out corrupted
_ 80 ELIBBAD___ Accessing a corrupted shared library
_ 79 ELIBACC___ Can not access a needed shared library
_ 78 EREMCHG___ Remote address changed
_ 77 EBADFD___ File descriptor in bad state
_ 76 ENOTUNIQ__ Name not unique on network
_ 75 EOVERFLOW__ Value too large for defined data type
_ 74 EBADMSG__ +Bad message
_ 73 EDOTDOT___ RFS specific error
_ 72 EMULTIHOP__ Multihop attempted
_ 71 EPROTO___ Protocol error
_ 70 ECOMM____ Communication error on send
_ 69 ESRMNT___ Srmount error
_ 68 EADV____ Advertise error
_ 67 ENOLINK___ Link has been severed
_ 66 EREMOTE___ Object is remote
_ 65 ENOPKG___ Package not installed
_ 64 ENONET___ Machine is not on the network
_ 63 ENOSR____ Out of streams resources
_ 62 ETIME____ Timer expired
_ 61 ENODATA___ No data available
_ 60 ENOSTR___ Device not a stream
_ 59 EBFONT___ Bad font file format
_ 57 EBADSLT___ Invalid slot
_ 56 EBADRQC___ Invalid request code
_ 55 ENOANO___ No anode
_ 54 EXFULL___ Exchange full
_ 53 EBADR____ Invalid request descriptor
_ 52 EBADE____ Invalid exchange
_ 51 EL2HLT___ Level 2 halted
_ 50 ENOCSI___ No CSI structure available
_ 49 EUNATCH___ Protocol driver not attached
_ 48 ELNRNG___ Link number out of range
_ 47 EL3RST___ Level 3 reset
_ 46 EL3HLT___ Level 3 halted
_ 45 EL2NSYNC__ Level 2 not synchronized
_ 44 ECHRNG___ Channel number out of range
_ 43 EIDRM____ Identifier removed
_ 42 ENOMSG___ No message of desired type
_ 40 ELOOP____ Too many levels of symbolic links
_ 39 ENOTEMPTY_ +Directory not empty
_ 38 ENOSYS___ +Function not implemented
_ 37 ENOLCK___ +No locks available
_ 36 ENAMETOOLONG +File name too long
_ 35 EDEADLK__ +Resource deadlock avoided
_ 34 ERANGE___ +Numerical result out of range
_ 33 EDOM____ +Numerical argument out of domain
_ 32 EPIPE___ +Broken pipe

接收端关闭(缓冲中没有多余的数据),但是发送端还在write:

1、Socket 关闭，但是socket号并没有置-1。继续在此socket上进行send和recv，就会返回这种错误。这个错误会引发SIGPIPE信号，系统会将产生此EPIPE错误的进程杀死。所以，一般在网络程序中，首先屏蔽此消息，以免发生不及时设置socket进程被杀死的情况。

2、write(..) on a socket that has been closed at the other end will cause a SIGPIPE.

3、错误被描述为“broken pipe”，即“管道破裂”，这种情况一般发生在客户进程不理会（或未及时处理）Socket 错误，继续向服务 TCP 写入更多数据时，内核将向客户进程发送 SIGPIPE 信号，该信号默认会使进程终止（此时该前台进程未进行 core dump）。结合上边的 ECONNRESET 错误可知，向一个 FIN_WAIT2 状态的服务 TCP（已 ACK 响应 FIN 分节）写入数据不成问题，但是写一个已接收了 RST 的 Socket 则是一个错误。


_ 31 EMLINK___ +Too many links
_ 30 EROFS___ +Read-only file system
_ 29 ESPIPE___ +Illegal seek
_ 28 ENOSPC___ +No space left on device
_ 27 EFBIG___ +File too large
_ 26 ETXTBSY___ Text file busy
_ 25 ENOTTY___ +Inappropriate ioctl for device
_ 24 EMFILE___ +Too many open files

打开了太多的socket。对进程或者线程而言，每种实现方法都有一个最大的可用socket数目处理，或者是全局的，或者是局部的。

_ 23 ENFILE___ +Too many open files in system
_ 22 EINVAL___ +Invalid argument

无效参数。提供的参数非法。有时也会与socket的当前状态相关，如一个socket并没有进入listening状态，此时调用accept，就会产生EINVAL错误。


_ 21 EISDIR___ +Is a directory
_ 20 ENOTDIR__ +Not a directory
_ 19 ENODEV___ +No such device
_ 18 EXDEV___ +Invalid cross-device link
_ 17 EEXIST___ +File exists
_ 16 EBUSY___ +Device or resource busy
_ 15 ENOTBLK___ Block device required
_ 14 EFAULT___ +Bad address地址错误
_ 13 EACCES___ +Permission denied
_ 12 ENOMEM___ +Cannot allocate memory
_ 11 EAGAIN___ +Resource temporarily unavailable

在读数据的时候,没有数据在底层缓冲的时候会遇到,一般的处理是循环进行读操作,异步模式还会等待读事件的发生再读

1、Send返回值小于要发送的数据数目，会返回EAGAIN和EINTR。

2、recv 返回值小于请求的长度时说明缓冲区已经没有可读数据，但再读不一定会触发EAGAIN，有可能返回0表示TCP连接已被关闭。

3、当socket是非阻塞时,如返回此错误,表示写缓冲队列已满,可以做延时后再重试.

4、在Linux进行非阻塞的socket接收数据时经常出现Resource temporarily unavailable，errno代码为11(EAGAIN)，表明在非阻塞模式下调用了阻塞操作，在该操作没有完成就返回这个错误，这个错误不会破坏socket的同步，不用管它，下次循环接着recv就可以。对非阻塞socket而言，EAGAIN不是一种错误。


_ 10 ECHILD___ +No child processes

__ 9 EBADF___ +Bad file descriptor

__ 8 ENOEXEC__ +Exec format error
__ 7 E2BIG___ +Argument list too long

__ 6 ENXIO___ +No such device or address

__ 5 EIO____ +Input/output error
__ 4 EINTR___ +Interrupted system call

阻塞的操作被取消阻塞的调用打断。如设置了发送接收超时，就会遇到这种错误。

只能针对阻塞模式的socket。读，写阻塞的socket时，-1返回，错误号为INTR。另外，如果出现EINTR即errno为4，错误描述Interrupted system call，操作也应该继续。如果recv的返回值为0，那表明连接已经断开，接收操作也应该结束。
__ 3 ESRCH___ +No such process

__ 2 ENOENT___ +No such file or directory

__ 1 EPERM___ +Operation not permitted

原文地址：https://zhuanlan.zhihu.com/p/451702640

作者：linux