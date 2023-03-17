# 【NO.550】Posix API 与 网络协议栈 详细介绍

## 0.前言

  本文详细介绍 Posix API 与 网络协议栈 之间的关系；三次握手、数据传输、四次挥手的过程。上下文耦合性较高，不建议跳跃阅读。

  本专栏知识点是通过零声教育的线上课学习，进行梳理总结写下文章，对c/c++linux课程感兴趣的读者，可以点击链接 C/C++后台高级服务器课程介绍 详细查看课程的服务。

## 1.Posix API 有哪些

  哪些是Posix API呢，就是Linux网络编程的这些API，本文介绍下列8种。

Tcp Server

1.socket 2.bind 3.listen 4.accept 5.recv 6.send 7.close

Tcp Client
1.socket 2.bind(option可选) 3.connect 4.send 5.recv 6.close

设置socket参数
1.setsockopt 2.getsockopt 3.shudown 4.fcntl

![image-20230224205136249](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230224205136249.png)

### 1.1 socket

  socket 是什么东西？中文翻译过来是插座的意思，那插座有两个部分，一个插一个座。

  socket也是由两部分组成，一个是fd（文件系统的文件描述符，是插头），任何我们能对 socket 进行操作的地方都是对这个 fd 进行操作。

  那插到什么地方呢，插头插到插座上，对于TCP而言，每个连接背后都有一个TCB（tcp控制块，tcp control block）。什么是TCB呢，服务器和客户端建立TCP连接，先建立了socket，那么socket底层看不到的东西，就是tcp控制块TCB，而能够看到的，就是文件描述符fd。

  我们操作fd，调用send，其实就是将数据放到TCB里面。调用recv，就是从TCB里拷贝出来。这个fd是我们用户操作的，而TCB是tcp协议栈里面的。一个fd对应一个TCB，这就是socket。

![image-20230224205151107](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230224205151107.png)

### 1.2 bind

  刚开始创建socket的时候，其底层的TCB是没有被初始化的，没有任何数据，TCB里面的状态机的状态也是close的，发送不了数据，也接收不了数据。

  下面介绍一个新的概念：五元组，当连接有很多很多的时候，哪个包到底是哪个连接的呢，这个时候就需要五元组。也就是说，通过五元组来确定一个TCB。五元组 < 源IP地址 , 源端口 , 目的IP地址 , 目的端口 , 协议 >

  bind的作用就是绑定本地的ip和端口，还有协议。也就是将TCB的五元组填充 <目的IP地址，目的端口，协议> ，注意客户端可以不使用bind函数，但其会默认分配。

## 2.三次握手 建立连接的过程

### 2.1 connect

  三次握手发生在协议栈和协议栈之间，而posix api connect 只是一个导火索，我们写的代码里面是没有写三次握手的。

  首先客户端先发三次握手的第一次数据包，这时候里面带有一个同步头syn，seq=x，这是由客户端内核协议栈发送的数据包。

  服务端接收到之后，返回三次握手的第二个数据包，syn=1,ack=1,seq=y,ack=x+1 。其中ack=x+1代表确认了x+1以前的都收到了，也就是说告诉对端，你发送的数据包，序号在x+1之前的我都收到了。同样也携带自己的一个同步头给对端。

  再往下面走，就是三次握手的第三次，返回一个ack确认包给服务器。

  这就反应了一个现象，tcp是双向的。双向怎么体现的呢，客户端发送一个数据包告诉服务器我现在发送的数据包序号是多少，服务端返回的时候也告诉客户端我这个数据包的序号是多少。这就是双向。

  有人会问三次握手为什么是三次？因为tcp是双向的，A给B发syn，B回一个ack，这里确定了B是存在的，这里两次。B给A发syn，A回一个ack，这里确定了A是存在的，这里两次，而中间的两次可以合到一起，那就是三次了。客户端发送一次syc，服务器返回一个ack并且携带自己的syn，这时候能确定服务器存在，客户端再返回一个ack，这时候能确定客户端存在，这时候就确定了这个双向通道是ok了。

![image-20230224205209641](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230224205209641.png)

  connect调用connect之后，协议栈开始三次握手。那么connect函数到底组不阻塞呢，取决于传进去的fd，如果fd是阻塞，那么直到第三次握手包发送之后就会返回。如果fd是非阻塞，那么返回-1的时候说明连接建立中，返回0代表连接建立成功。

```
n=-1  , err:Operation now in progress
n=0 
```



### 2.2 listen

  服务器内核协议栈在接收到三次握手的第一次syn包的时候，从这个sync包里面可以解析出来源IP地址 , 源端口 , 目的IP地址 , 目的端口 , 协议 ，那么五元组五元组 < 源IP地址 , 源端口 , 目的IP地址 , 目的端口 , 协议 > 就可以确定下来了，从而构建出来一个TCB,只不过目前这个TCB还不能用，因为还没有分配socket，还没有分配fd。这时候会将TCB加入到半连接队列（sync队列）里面。

  同样当第三次握手收到ACK之后，也有一个队列，叫做全连接队列。所以在第一次收到syn包的时候，服务端做两件事，1返回ACK，2创建一个TCB结点，加入半连接队列里面。在第三次握手的时候，先查找半连接队列。那么怎么查找呢，通过五元组查找。

![image-20230224205231716](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230224205231716.png)

  那这个半连接队列和全连接队列的建立有没有前提？难道是想建立就建立吗，建立的前提就是必须先进入listen状态。我们来看一下TCP状态转移过程。

  服务器首先进入Listen状态，收SYN，发SYN和，ACK，然后就进入了SYN_RCVD的状态。注意，这里进入SYN_RCVD的状态是刚才新创建的那个TCB，改变的是新创建的TCB的状态，而不是被动监听fd的状态。SYN_RCVD这个状态就暗示这TCB已经进入了半连接队列里面，也就是说半连接队列里面所有的TCB的状态都是SYN_RCVD。

  客户端首先发收SYN包，则客户端进入SYN_SENT状态

![image-20230224205244819](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230224205244819.png)

![image-20230224205258105](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230224205258105.png)  

服务器当收到第三次握手的ACK的时候，会将对应的在半连接队列里面的TCB移到全连接队列里面，这个时候TCB的状态由SYN_RCVD变成ESTABLISHED状态。

  客户端发送第三次握手的ACK的时候，也会进入ESTABLISHED状态。

![image-20230224205315759](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230224205315759.png)


![image-20230224205338370](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230224205338370.png)现在有一个问题，listen这个函数有两个参数,这个backlog是什么意思？

```
listen(fd,backlog);
```


backlog有两种理解情况

在unix，mac系统里面，半连接队列与全连接队列的总和 <= backlog
在Linux系统里面, 全连接队列<=backlog
ddos攻击：客户端不断的模拟三次握手的第一次，发syn包

如果在Linux系统中，backlog无论设置多少都是没用的。如果是在unix，mac系统中，设置backlog还是有一定作用的。

### 2.3 accept

  这个时候连接已经建立完了，双方都知道对方的存在了，现在就可以调用accept了，accept函数只做两件事情

```
int clientfd=accept()
```


从全连接队列里面取出一个TCB结点
为这个TCB结点分配一个fd，把fd和TCB做一个一对一对应的关系。
直到现在，应用程序才可以操作这个tcp的会话。

![image-20230224205437450](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230224205437450.png)

## 3.数据传输 发送与接收

### 3.1 send & recv

  我们知道发送用send，接收用recv。

  send只是了将用户态的数据拷贝到内核协议栈对应的TCB里面。至于真正数据发送的时机，什么时候发送的，发送的数据有没有与之前的数据粘在一起，都不是由应用程序决定的，应用程序只能将数据拷贝到内核buffer缓冲区里面。 然后协议栈将sendbuffer的数据，加上TCP的头，加上IP的头，加上以太网的头，一起打包发出去。所以调用send将buffer拷贝到内核态缓冲区，与tcp发送数据到对端，是异步的过程。

  对端网络协议栈接收到数据，同样开始解析，以太网的头mac地址是谁，ip地址从哪里来的，源端口是多少，目的端口是发到哪个进程里面，然后将数据写进对应的TCB里。recv只是将内核态的缓冲区数据拷贝到用户态里面，所以tcp数据到达TCP的recv buffer缓冲区里，与调用recv将缓冲区buffer拷贝到用户态，这两个过程也是异步的。

![image-20230224205627650](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230224205627650.png)

![image-20230224205638673](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230224205638673.png)

### 3.2 滑动窗口

  如果客户端不断的send，服务器对应的tcb的recvbuffer缓冲区满了怎么办？

  首先，如果不停的send，直到sendbuffer缓冲区满了，这个时候send会返回-1，代表内核缓冲区满了，send的copy失败。而如果recvbuffer缓冲区满了而应用程序没有去接收，这时候TCP协议栈会告诉对端，我的缓冲区空间还有多大，超过这个大小就不要发（滑动窗口），也就是说recvbuffer缓冲区会通知对端，我能接收多少数据，而对端发送的数据量一定要在这个范围内才行。

![image-20230224205648596](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230224205648596.png)

  一般send的时候，在TCP协议头里面有一个push的标志位，置1代表立即通知应用程序来读取。

![image-20230224205656952](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230224205656952.png)

### 3.3 粘包 & 分包

  假设我们连续send 1k的数据三次，那么在内核tcb的缓冲区里面，就有3k的数据，这3k的数据是1次tcp发走，还是分2次，分3次，都不是由应用程序控制的。这就出现了 粘包 和 分包 的问题。

  假设send 了3次，而协议栈只发送了2次，那么在recv的时候读两次，就避免不了数据包合在一起的现象。

解决的方法有两种：

第一种：在数据包前面加上这个包有多长
第二种：为每一个包加一个特定的分隔符

![image-20230224205710246](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230224205710246.png)

### 3.4 延迟确认（延迟ACK）

  上面两种解决方法有一个很大的前提，就是这个数据包是顺序的。先发的先到，后发的后到，这就是顺序。那TCP是怎么保证顺序的？

  在TCP发送的时候，数据包都是确定的，第一个包发完之后，对端等待一会，再确认这个包。假设现在发5个包，A到了，B到了，每收到一个包，对端都会重置延迟定时器200ms。

  现在假设B包第一个到，现在启动定时器200ms，然后C包到了，重置定时器，在200ms以内A包也到了，再重置，E包到了，重置，最后200ms超时了，D包没到。这时候就会ACK=D，代表D之前的都收到了。接下来D以及D后面的数据包都会重发。

  这样就解决了包的无序的问题，这里的操作都是TCP协议栈来做的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2b15432170cb41c0b6ba23b07d263b6c.png)



![在这里插入图片描述](https://img-blog.csdnimg.cn/2c9697faf13044cbb6a5c9ecb74e860a.png)

### 3.5 udp场景

  延迟ACK确认时间长，超时重传的时候，重传的包较多，很费带宽。这就有了udp的使用空间。

  随着带宽越来越高，udp的使用场景在不断弱化，但是在弱网的环境下，做大量数据传输的时候，TCP就不合适了，因为一旦出现丢包的情况，后面的包都要重传了。

  并且TCP也没有办法保证实时性，虽然可以关闭延迟ACK来解决这个问题。但是实时性也会用到udp。

  udp场景：1弱网的环境下（电梯里网就很烂） 2实时性要求高的环境下（游戏打团，秒人只在一瞬间）。

## 4.四次挥手 断开连接的过程

  这6个状态就是四次挥手的过程，对于TCP而言建立连接时间很短，断开连接时间很短，中间传输数据是绝大部分时间，但是中间传输数据的状态只有一个，就是ESTABLISHED。断开连接的过程只有一个函数，就是close。

![image-20230224205751721](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230224205751721.png)

### 4.1 正常情况

  在四次挥手的过程中，没有客户端和服务器之分，只有主动方和被动方之分。主动方首先发送一个fin，被动方返回一个ack。被动方再发送一个fin，主动方返回一个ack。这就是4次挥手

  第一次的fin是哪来的呢，调用close这个函数，协议栈会将最后一个包fin位，置1，被动方接收之后，会触发一个可读事件，recv=0。被动方会做两件事情，第一件事情推给应用程序一个空包，第二件事情直接返回一个ack的包返回给对端。然后被动方recv=0读到之后，应用程序会调用close，这时候被动方也会发送一个fin，对端收到fin会回复一个ack，至此四次挥手完毕。

![image-20230224205805923](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230224205805923.png)

  四次挥手为什么要有四次？因为tcp是双向的，就跟两个人分手一样，女的跟男的分手，男的说我知道了，那我也跟你分手吧。第一对fin和ack就是女方终止与男方交往，第二对fin和ack就是男方终止与女方交往。这个过程就形成断开了。四次是因为女方跟男方分手之后，男方需要缓和一段时间，中间间隔一段时间（因为这个时候被动方的数据可能还没有发完，在缓和期要把数据都发过去）。

  主动方在调用close之前它的状态是确定的ESTABLISHED状态，发送fin后，进入FIN_WAIT_1状态，收到ack以后进入FIN_WAIT_2状态。如果中间两次ack和fin一起的话，那就没有这个状态，直接进入TIME_WAIT状态。也就是说收到fin后进入TIME_WAIT状态。在等待2MSL后，进入CLOSED状态。TIME_WAIT存在的原因避免最后一个ack丢失,而对端一直超时重发fin，导致连接得不到释放。

  被动方在接收到fin后，进入CLOSE_WAIT状态，之后调用close发送fin后，进入LAST_ACK状态，收到ack之后，进入CLOSED状态。

![image-20230224205826568](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230224205826568.png)



![image-20230224205838425](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230224205838425.png)

![image-20230224205847590](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230224205847590.png)

### 4.2 特殊情况

  有没有一种可能，主动方调用close，被动方也调用close。也就是说一对情侣同时提出分手。在FIN_WAIT1状态期间接收到fin，这时候就进入CLOSING状态，之后收到ack就进入TIME_WAIT状态。

![image-20230224205900107](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230224205900107.png)

![image-20230224205915052](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20230224205915052.png)

### 4.3 一些面试问题

  现在假设客户端连接进入FIN_WAIT_1状态，会在这个状态很久吗？不会，因为即使没有收到ack，也会超时重传fin，进入FIN_WAIT_2状态。

  如果连接停在FIN_WAIT_2状态的时间很久怎么办呢？既然客户端停在FIN_WAIT_2状态，那也就是说服务器停在CLOSE_WAIT状态。服务器出现大量CLOSE_WAIT状态，造成这一现象是因为对方调用关闭，而服务器没有调用close，再去分析，其实就是业务逻辑的问题。

  recv=0，但是没有调用close，原因在哪呢。也就是说从recv到调用close这个过程中间，时间太长，为什么时间太长呢，可能在调用close之前，有去关闭一些fd相关联的业务信息，造成比较耗时的情况。假设现在是一款即时通讯的程序，现在客户端掉线调用了close，服务器接收到recv=0后，服务器需要把这个客户端对应的临时数据同步到数据库里面，会出现一个很耗时的操作，那么这个时间内就是出现CLOSE_WAIT的状态。

  那么如何解决呢，1. 要么先调用close 2. 要么把业务信息抛到消息队列里面交给线程池进行处理。把业务的清理当成一个任务交给另一个线程处理。 原来的线程把网络这一层处理好。

  作为客户端去连第三方服务，长时间卡在FIN_WAIT_2状态，有没有办法去终止它？从FIN_WAIT_2是不能直接到CLOSED状态的，所以这个问题要么再起一个连接，要么就杀死进程，要么就等待FIN_WAIT_2定时器超时。

  如果服务器在调用close之前宕机了，fin是肯定发不到客户端的，那么客户端一直在FIN_WAIT_2状态，这个时候怎么办呢,如果开启了keepalive，检测到是死链接后会被终止掉。那没有开启keepalive呢？

  FIN_WAIT_2 状态的一端一直等不到对端的FIN。如果没有外力的作用，连接两端会一直分别处于 FIN_WAIT_2 和 CLOSE_WAIT 状态。这会造成系统资源的浪费，需要对其进行处理。（内核协议栈就有参数提供了对这种异常情况的处理，无需应用程序操作），也就是说，等着就行。

  如果应用程序调用的是完全关闭（而不是半关闭），那么内核将会起一个定时器，设置最晚收到对端FIN报文的时间。如果定时器超时后仍未收到FIN，且此时TCP连接处于空闲状态，则TCP连接就会从 FIN_WAIT_2 状态直接转入 CLOSED 状态，关闭连接。在Linux系统中可以通过参数 net.ipv4.tcp_fin_timeout 设置定时器的超时时间，默认为60s。

## 5.回收fd和tcb

  被动方调用close之后，fd被回收。在接收到ack以后进入CLOSED后，TCB被回收

  主动方调用close之后，fd被回收，在time_wait时间到了进入CLOSED后，TCB被回收

![在这里插入图片描述](https://img-blog.csdnimg.cn/fbc3c205bc764e0a97a57c75b663fbee.png)

## 6.TCP状态迁移图

![在这里插入图片描述](https://img-blog.csdnimg.cn/b982966e09bc4a15accf484e86a2324a.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/f09987d83e22424d909b1e91faeca7d1.png)————————————————
版权声明：本文为CSDN博主「cheems~」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_42956653/article/details/125727563