# 【NO.615】linux服务器性能调优之tcp/ip性能调优

![img](https://pic2.zhimg.com/80/v2-e83d768d5402ceda6da9302ffcf0e039_720w.webp)

TCP状态转移图

## **一、TCP状态介绍：**

在TCP/IP协议中，TCP协议提供可靠的连接服务，采用三次握手建立一个连接。

> *第一次握手：建立连接时，客户端发送syn包(syn=x)到服务器，并进入SYN_SEND状态，等待服务器确认；*
> *第二次握手：服务器收到syn包，必须确认客户的SYN（ack=x+1），同时自己也发送一个SYN包（syn=y），即SYN+ACK包，此时服务器进入SYN_RECV状态；*
> *第三次握手：客户端收到服务器的SYN＋ACK包，向服务器发送确认包ACK(ack=y+1)，此包发送完毕，客户端和服务器进入ESTABLISHED状态，完成三次握手。*

完成三次握手，客户端与服务器开始传送数据，在上述过程中，还有一些重要的概念：

未连接队列：在三次握手协议中，服务器维护一个未连接队列，该队列为每个客户端的SYN包（syn=j）开设一个条目，该条目表明服务器已收到 SYN包，并向客户发出确认，正在等待客户的确认包。这些条目所标识的连接在服务器处于Syn_RECV状态，当服务器收到客户的确认包时，删除该条目， 服务器进入ESTABLISHED状态。
Backlog参数：表示未连接队列的最大容纳数目。

SYN-ACK 重传次数　服务器发送完SYN－ACK包，如果未收到客户确认包，服务器进行首次重传，等待一段时间仍未收到客户确认包，进行第二次重传，如果重传次数超 过系统规定的最大重传次数，系统将该连接信息从半连接队列中删除。注意，每次重传等待的时间不一定相同。

半连接存活时间：是指半连接队列的条目存活的最长时间，也即服务从收到SYN包到确认这个报文无效的最长时间，该时间值是所有重传请求包的最长等待时间总和。有时我们也称半连接存活时间为Timeout时间、SYN_RECV存活时间。

状态解释
CLOSED: 表示初始状态。

LISTEN:
表示服务器端的某个SOCKET处于监听状态，可以接受连接

SYN_RCVD:
表示接受到了SYN报文，在正常情况下，这个状态是服务器端的SOCKET在建立TCP连接时的三次握手会话过程中的一个中间状态，很短暂，基本上用 netstat你是很难看到这种状态的，除非你特意写了一个客户端测试程序，故意将三次TCP握手过程中最后一个ACK报文不予发送。因此这种状态时，当 收到客户端的ACK报文后，它会进入到ESTABLISHED状态。

SYN_SENT:
这个状态与SYN_RCVD遥想呼应，当客户端SOCKET执行CONNECT连接时，它首先发送SYN报文，因此也随即它会进入到了SYN_SENT状态，并等待服务端的发送三次握手中的第2个报文。SYN_SENT状态表示客户端已发送SYN报文。

ESTABLISHED：表示连接已经建立

FIN_WAIT_1:
FIN_WAIT_1和FIN_WAIT_2状态的真正含义都是表示等待对方的FIN报文。而这两种状态的区别是：FIN_WAIT_1状态实际上是当 SOCKET在ESTABLISHED状态时，它想主动关闭连接，向对方发送了FIN报文，此时该SOCKET即进入到FIN_WAIT_1状态。而当对 方回应ACK报文后，则进入到FIN_WAIT_2状态，当然在实际的正常情况下，无论对方何种情况下，都应该马上回应ACK报文，所以 FIN_WAIT_1状态一般是比较难见到的，而FIN_WAIT_2状态还有时常常可以用netstat看到。

FIN_WAIT_2:
FIN_WAIT_2状态下的SOCKET，表示半连接，也即有一方要求close连接，但另外还告诉对方，我暂时还有点数据需要传送给你，稍后再关闭连接。

TIME_WAIT:
表示收到了对方的FIN报文，并发送出了ACK报文，就等2MSL后即可回到CLOSED可用状态了。如果FIN_WAIT_1状态下，收到了对方同时带FIN标志和ACK标志的报文时，可以直接进入到TIME_WAIT状态，而无须经过FIN_WAIT_2状态。

CLOSING:
正常情况下，发送FIN报文后，按理来说是应该先收到（或同时收到）对方的ACK报文，再收到对方的FIN报文。但是CLOSING状态表示你发送FIN 报文后，并没有收到对方的ACK报文，反而却也收到了对方的FIN报文。什么情况下会出现此种情况呢？其实细想一下，也不难得出结论：那就是如果双方几乎 在同时close一个SOCKET的话，那么就出现了双方同时发送FIN报文的情况，也即会出现CLOSING状态，表示双方都正在关闭SOCKET连 接。

CLOSE_WAIT:
表示在等待关闭。怎么理解呢？当对方close一个SOCKET后发送FIN报文给自己，你系统毫无疑问地会回应一个ACK报文给对方，此时则进入到 CLOSE_WAIT状态。接下来呢，实际上你真正需要考虑的事情是察看你是否还有数据发送给对方，如果没有的话，那么你也就可以 close这个SOCKET，发送FIN报文给对方，也即关闭连接。所以你在CLOSE_WAIT状态下，需要完成的事情是等待你去关闭连接。

LAST_ACK:
被动关闭一方在发送FIN报文后，最后等待对方的ACK报文。当收到ACK报文后，也即可以进入到CLOSED可用状态了。

## **二、调优：**

所有的TCP/IP调优参数都位于/proc/sys/net/目录. 例如, 下面是最重要的一些调优参数, 后面是它们的含义:

\1. /proc/sys/net/core/rmem_max — 最大的TCP数据接收缓冲
\2. /proc/sys/net/core/wmem_max — 最大的TCP数据发送缓冲
\3. /proc/sys/net/ipv4/tcp_timestamps — 时间戳在(请参考RFC 1323)TCP的包头增加12个字节
\4. /proc/sys/net/ipv4/tcp_sack — 有选择的应答
\5. /proc/sys/net/ipv4/tcp_window_scaling — 支持更大的TCP窗口. 如果TCP窗口最大超过65535(64K), 必须设置该数值为1
\6. rmem_default — 默认的接收窗口大小
\7. rmem_max — 接收窗口的最大大小
\8. wmem_default — 默认的发送窗口大小
\9. wmem_max — 发送窗口的最大大小

------

net.core.rmem_default = 256960
net.core.rmem_max = 256960
net.core.wmem_default = 256960
net.core.wmem_max = 256960

net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_sack =1
net.ipv4.tcp_window_scaling = 1

------

其他调优：
开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN***，默认为0，表示关闭；
net.ipv4.tcp_syncookies = 1

开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；
net.ipv4.tcp_tw_reuse = 1

开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭；
net.ipv4.tcp_tw_recycle = 1

系统默认的TIMEOUT时间。
net.ipv4.tcp_fin_timeout = 5

当keepalive起用的时候，TCP发送keepalive消息的频度。缺省是2小时，改为20分钟(20*60s)
net.ipv4.tcp_keepalive_time = 1200

表示用于向外连接的端口范围。缺省情况下很小：32768到61000，改为10000到65000
net.ipv4.ip_local_port_range = 10000 65000

SYN队列的长度，默认为1024，加大队列长度为8192，可以容纳更多等待连接的网络连接数
net.ipv4.tcp_max_syn_backlog = 8192

系统同时保持TIME_WAIT的最大数量，如果超过这个数字，TIME_WAIT将立刻被清除并打印警告信息。默认为180000，改为5000
net.ipv4.tcp_max_tw_buckets = 5000

其他调优参数
tcp_syn_retries ：INTEGER
默认值是5
对于一个新建连接，内核要发送多少个 SYN 连接请求才决定放弃。不应该大于255，默认值是5，对应于180秒左右时间。(对于大负载而物理通信良好的网络而言,这个值偏高,可修改为2.这个值仅 仅是针对对外的连接,对进来的连接,是由tcp_retries1 决定的)

tcp_synack_retries ：INTEGER
默认值是5
对于远端的连接请求SYN，内核会发送SYN ＋ ACK数据报，以确认收到上一个 SYN连接请求包。这是所谓的三次握手( threeway handshake)机制的第二个步骤。这里决定内核在放弃连接之前所送出的 SYN+ACK 数目。不应该大于255，默认值是5，对应于180秒左右时间。(可以根据上面的 tcp_syn_retries 来决定这个值)

tcp_keepalive_time ：INTEGER
默认值是7200(2小时)
当keepalive打开的情况下，TCP发送keepalive消息的频率。(由于目前网络***等因素,造成了利用这个进行的***很频繁,曾经也有cu 的朋友提到过,说如果2边建立了连接,然后不发送任何数据或者rst/fin消息,那么持续的时间是不是就是2小时,空连接***? tcp_keepalive_time就是预防此情形的.我个人在做nat服务的时候的修改值为1800秒)

tcp_keepalive_probes：INTEGER
默认值是9
TCP发送keepalive探测以确定该连接已经断开的次数。(注意:保持连接仅在SO_KEEPALIVE套接字选项被打开是才发送.次数默认不需要修改,当然根据情形也可以适当地缩短此值.设置为5比较合适)

tcp_keepalive_intvl：INTEGER
默认值为75
探测消息发送的频率，乘以tcp_keepalive_probes就得到对于从开始探测以来没有响应的连接杀除的时间。默认值为75秒，也就是没有活动 的连接将在大约11分钟以后将被丢弃。(对于普通应用来说,这个值有一些偏大,可以根据需要改小.特别是web类服务器需要改小该值,15是个比较合适的 值)

tcp_retries1 ：INTEGER
默认值是3
放弃回应一个TCP连接请求前﹐需要进行多少次重试。RFC 规定最低的数值是3﹐这也是默认值﹐根据RTO的值大约在3秒 – 8分钟之间。(注意:这个值同时还决定进入的syn连接)

tcp_retries2 ：INTEGER
默认值为15
在丢弃激活(已建立通讯状况)的TCP连接之前﹐需要进行多少次重试。默认值为15，根据RTO的值来决定，相当于13-30分钟(RFC1122规定，必须大于100秒).(这个值根据目前的网络设置,可以适当地改小,我的网络内修改为了5)

tcp_orphan_retries ：INTEGER
默认值是7
在近端丢弃TCP连接之前﹐要进行多少次重试。默认值是7个﹐相当于 50秒 – 16分钟﹐视 RTO 而定。如果您的系统是负载很大的web服务器﹐那么也许需要降低该值﹐这类 sockets 可能会耗费大量的资源。另外参的考 tcp_max_orphans 。(事实上做NAT的时候,降低该值也是好处显著的,我本人的网络环境中降低该值为3)

tcp_fin_timeout ：INTEGER
默认值是 60
对于本端断开的socket连接，TCP保持在FIN-WAIT-2状态的时间。对方可能会断开连接或一直不结束连接或不可预料的进程死亡。默认值为 60 秒。过去在2.2版本的内核中是 180 秒。您可以设置该值﹐但需要注意﹐如果您的机器为负载很重的web服务器﹐您可能要冒内存被大量无效数据报填满的风险﹐FIN-WAIT-2 sockets 的危险性低于 FIN-WAIT-1 ﹐因为它们最多只吃 1.5K 的内存﹐但是它们存在时间更长。另外参考 tcp_max_orphans。(事实上做NAT的时候,降低该值也是好处显著的,我本人的网络环境中降低该值为30)

tcp_max_tw_buckets ：INTEGER
默认值是180000
系 统在同时所处理的最大 timewait sockets 数目。如果超过此数的话﹐time-wait socket 会被立即砍除并且显示警告信息。之所以要设定这个限制﹐纯粹为了抵御那些简单的 DoS ***﹐千万不要人为的降低这个限制﹐不过﹐如果网络条件需要比默认值更多﹐则可以提高它(或许还要增加内存)。(事实上做NAT的时候最好可以适当地增加 该值)

tcp_tw_recycle ：BOOLEAN
默认值是0
打开快速 TIME-WAIT sockets 回收。除非得到技术专家的建议或要求﹐请不要随意修改这个值。(做NAT的时候，建议打开它)

tcp_tw_reuse：BOOLEAN
默认值是0
该文件表示是否允许重新应用处于TIME-WAIT状态的socket用于新的TCP连接(这个对快速重启动某些服务,而启动后提示端口已经被使用的情形非常有帮助)

tcp_max_orphans ：INTEGER
缺省值是8192
系统所能处理不属于任何进程的TCP sockets最大数量。假如超过这个数量﹐那么不属于任何进程的连接会被立即reset，并同时显示警告信息。之所以要设定这个限制﹐纯粹为了抵御那些 简单的 DoS ***﹐千万不要依赖这个或是人为的降低这个限制(这个值Redhat AS版本中设置为32768,但是很多防火墙修改的时候,建议该值修改为2000)

tcp_abort_on_overflow ：BOOLEAN
缺省值是0
当守护进程太忙而不能接受新的连接，就象对方发送reset消息，默认值是false。这意味着当溢出的原因是因为一个偶然的猝发，那么连接将恢复状态。 只有在你确信守护进程真的不能完成连接请求时才打开该选项，该选项会影响客户的使用。(对待已经满载的sendmail,apache这类服务的时候,这 个可以很快让客户端终止连接,可以给予服务程序处理已有连接的缓冲机会,所以很多防火墙上推荐打开它)

tcp_syncookies ：BOOLEAN
默认值是0
只有在内核编译时选择了CONFIG_SYNCOOKIES时才会发生作用。当出现syn等候队列出现溢出时象对方发送syncookies。目的是为了防止syn flood***。
注意：该选项千万不能用于那些没有收到***的高负载服务器，如果在日志中出现synflood消息，但是调查发现没有收到synflood***，而是合法用户的连接负载过高的原因，你应该调整其它参数来提高服务器性能。参考:
tcp_max_syn_backlog
tcp_synack_retries
tcp_abort_on_overflow
syncookie严重的违背TCP协议，不允许使用TCP扩展，可能对某些服务导致严重的性能影响(如SMTP转发)。(注意,该实现与BSD上面使用 的tcp proxy一样,是违反了RFC中关于tcp连接的三次握手实现的,但是对于防御syn-flood的确很有用.)

tcp_stdurg ：BOOLEAN
默认值为0
使用 TCP urg pointer 字段中的主机请求解释功能。大部份的主机都使用老旧的 BSD解释，因此如果您在 Linux 打开它﹐或会导致不能和它们正确沟通。

tcp_max_syn_backlog ：INTEGER
对于那些依然还未获得客户端确认的连接请求﹐需要保存在队列中最大数目。对于超过 128Mb 内存的系统﹐默认值是 1024 ﹐低于 128Mb 的则为 128。如果服务器经常出现过载﹐可以尝试增加这个数字。警告﹗假如您将此值设为大于 1024﹐最好修改 include/net/tcp.h 里面的 TCP_SYNQ_HSIZE ﹐以保持 TCP_SYNQ_HSIZE*16<=tcp_max_syn_backlog ﹐并且编进核心之内。(SYN Flood***利用TCP协议散布握手的缺陷，伪造虚假源IP地址发送大量TCP-SYN半打开连接到目标系统，最终导致目标系统Socket队列资源耗 尽而无法接受新的连接。为了应付这种***，现代Unix系统中普遍采用多连接队列处理的方式来缓冲(而不是解决)这种***，是用一个基本队列处理正常的完 全连接应用(Connect()和Accept() )，是用另一个队列单独存放半打开连接。这种双队列处理方式和其他一些系统内核措施(例如Syn-Cookies/Caches)联合应用时，能够比较有 效的缓解小规模的SYN Flood***(事实证明<1000p/s)加大SYN队列长度可以容纳更多等待连接的网络连接数，所以对Server来说可以考虑增大该值.)

tcp_window_scaling ：INTEGER
缺省值为1
该 文件表示设置tcp/ip会话的滑动窗口大小是否可变。参数值为布尔值，为1时表示可变，为0时表示不可变。tcp/ip通常使用的窗口最大可达到 65535 字节，对于高速网络，该值可能太小，这时候如果启用了该功能，可以使tcp/ip滑动窗口大小增大数个数量级，从而提高数据传输的能力(RFC 1323)。（对普通地百M网络而言，关闭会降低开销，所以如果不是高速网络，可以考虑设置为0）

tcp_timestamps ：BOOLEAN
缺省值为1
Timestamps 用在其它一些东西中﹐可以防范那些伪造的 sequence 号码。一条1G的宽带线路或许会重遇到带 out-of-line数值的旧sequence 号码(假如它是由于上次产生的)。Timestamp 会让它知道这是个 ‘旧封包’。(该文件表示是否启用以一种比超时重发更精确的方法（RFC 1323）来启用对 RTT 的计算；为了实现更好的性能应该启用这个选项。)

tcp_sack ：BOOLEAN
缺省值为1
使 用 Selective ACK﹐它可以用来查找特定的遗失的数据报— 因此有助于快速恢复状态。该文件表示是否启用有选择的应答（Selective Acknowledgment），这可以通过有选择地应答乱序接收到的报文来提高性能（这样可以让发送者只发送丢失的报文段）。(对于广域网通信来说这个 选项应该启用，但是这会增加对 CPU 的占用。)

tcp_fack ：BOOLEAN
缺省值为1
打开FACK拥塞避免和快速重传功能。(注意，当tcp_sack设置为0的时候，这个值即使设置为1也无效)

tcp_dsack ：BOOLEAN
缺省值为1
允许TCP发送”两个完全相同”的SACK。

原文地址：https://zhuanlan.zhihu.com/p/602356909

作者：linux