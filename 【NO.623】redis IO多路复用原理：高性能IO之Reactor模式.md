# 【NO.623】redis IO多路复用原理：高性能IO之Reactor模式

首先你得了解redis是单线程的，然后你接着会有个疑问，单线程怎么会有高性能呢（据悉，在普通的笔记本上redis吞吐量亦能达到每秒几十W次），带着疑问看看下面转载的帖子吧。

## 1.先个人总结：

所谓的redis的多路复用原理

他是把IO操作再细分成多个事件去处理

比如IO涉及连接 读取 输入

把这三种当成三种事件分别起三个线程处理

如果一个连接来了显示被读取线程处理了，然后再执行写入，那么之前的读取就可以被后面的请求复用，吞吐量就提高了

## 2.正文：

讲到高性能IO绕不开Reactor模式，它是大多数IO相关组件如Redis在使用的IO模式，为什么需要这种模式，它是如何设计来解决高性能并发的呢？

最最原始的网络编程思路就是服务器用一个while循环，不断监听端口是否有新的套接字连接，如果有，那么就调用一个处理函数处理，类似：

```text
while(true){undefined
socket = accept();
handle(socket)
}
```

这种方法的最大问题是无法并发，效率太低，如果当前的请求没有处理完，那么后面的请求只能被阻塞，服务器的吞吐量太低。

之后，想到了使用多线程，也就是很经典的connection per thread，每一个连接用一个线程处理，类似：

```text
while(true){undefined
socket = accept();
new thread(socket);
}
```

多线程的方式确实一定程度上极大地提高了服务器的吞吐量，因为之前的请求在read阻塞以后，不会影响到后续的请求，因为他们在不同的线程中。这也是为什么通常会讲“一个线程只能对应一个socket”的原因。最开始对这句话很不理解，线程中创建多个socket不行吗？语法上确实可以，但是实际上没有用，每一个socket都是阻塞的，所以在一个线程里只能处理一个socket，就算accept了多个也没用，前一个socket被阻塞了，后面的是无法被执行到的。

缺点在于资源要求太高，系统中创建线程是需要比较高的系统资源的，如果连接数太高，系统无法承受，而且，线程的反复创建-销毁也需要代价。

线程池本身可以缓解线程创建-销毁的代价，这样优化确实会好很多，不过还是存在一些问题的，就是线程的粒度太大。每一个线程把一次交互的事情全部做了，包括读取和返回，甚至连接，表面上似乎连接不在线程里，但是如果线程不够，有了新的连接，也无法得到处理，所以，目前的方案线程里可以看成要做三件事，连接，读取和写入。

线程同步的粒度太大了，限制了吞吐量。应该把一次连接的操作分为更细的粒度或者过程，这些更细的粒度是更小的线程。整个线程池的数目会翻倍，但是线程更简单，任务更加单一。这其实就是Reactor出现的原因，在Reactor中，这些被拆分的小线程或者子过程对应的是handler，每一种handler会出处理一种event。这里会有一个全局的管理者selector，我们需要把channel注册感兴趣的事件，那么这个selector就会不断在channel上检测是否有该类型的事件发生，如果没有，那么主线程就会被阻塞，否则就会调用相应的事件处理函数即handler来处理。典型的事件有连接，读取和写入，当然我们就需要为这些事件分别提供处理器，每一个处理器可以采用线程的方式实现。一个连接来了，显示被读取线程或者handler处理了，然后再执行写入，那么之前的读取就可以被后面的请求复用，吞吐量就提高了。

几乎所有的网络连接都会经过读请求内容——》解码——》计算处理——》编码回复——》回复的过程，Reactor模式的的演化过程如下：

![img](https://pic1.zhimg.com/80/v2-e163d55554051be2cd569573392548dc_720w.webp)

这种模型由于IO在阻塞时会一直等待，因此在用户负载增加时，性能下降的非常快。

server导致阻塞的原因：

1、serversocket的accept方法，阻塞等待client连接，直到client连接成功。

2、线程从socket inputstream读入数据，会进入阻塞状态，直到全部数据读完。

3、线程向socket outputstream写入数据，会阻塞直到全部数据写完。

**改进：采用基于事件驱动的设计，当有事件触发时，才会调用处理器进行数据处理。**

![img](https://pic3.zhimg.com/80/v2-9dd344f8802cae8b7d1568825b019ff6_720w.webp)

Reactor：负责响应IO事件，当检测到一个新的事件，将其发送给相应的Handler去处理。

Handler：负责处理非阻塞的行为，标识系统管理的资源；同时将handler与事件绑定。

Reactor为单个线程，需要处理accept连接，同时发送请求到处理器中。

由于只有单个线程，所以处理器中的业务需要能够快速处理完。

**改进：使用多线程处理业务逻辑。**

![img](https://pic1.zhimg.com/80/v2-51d039b072e52f79abd5b0f4f09b3e18_720w.webp)

将处理器的执行放入线程池，多线程进行业务处理。但Reactor仍为单个线程。

**继续改进：对于多个CPU的机器，为充分利用系统资源，将Reactor拆分为两部分。**

**Using Multiple Reactors**

![img](https://pic2.zhimg.com/80/v2-d4b187ac622961a1502a29ff9d355c4d_720w.webp)

mainReactor负责监听连接，accept连接给subReactor处理，为什么要单独分一个Reactor来处理监听呢？因为像TCP这样需要经过3次握手才能建立连接，这个建立连接的过程也是要耗时间和资源的，单独分一个Reactor来处理，可以提高性能。

## 3.Reactor模式是什么，有哪些优缺点？

Wikipedia上说：“The reactor design pattern is an event handling pattern for handling service requests delivered concurrently by one or more inputs. The service handler then demultiplexes the incoming requests and dispatches them synchronously to associated request handlers.”。从这个描述中，我们知道Reactor模式首先是事件驱动的，有一个或多个并发输入源，有一个Service Handler，有多个Request Handlers；这个Service Handler会同步的将输入的请求（Event）多路复用的分发给相应的Request Handler。如果用图来表达：

![img](https://pic3.zhimg.com/80/v2-2d62ec283140a31ccd6643acded7043a_720w.webp)

从结构上，这有点类似生产者消费者模式，即有一个或多个生产者将事件放入一个Queue中，而一个或多个消费者主动的从这个Queue中Poll事件来处理；而Reactor模式则并没有Queue来做缓冲，每当一个Event输入到Service Handler之后，该Service Handler会主动的根据不同的Event类型将其分发给对应的Request Handler来处理。

## 4.Reactor模式结构

在解决了什么是Reactor模式后，我们来看看Reactor模式是由什么模块构成。图是一种比较简洁形象的表现方式，因而先上一张图来表达各个模块的名称和他们之间的关系：

![img](https://pic1.zhimg.com/80/v2-02a331574906e092b6795d469962608c_720w.webp)

Handle：即操作系统中的句柄，是对资源在操作系统层面上的一种抽象，它可以是打开的文件、一个连接(Socket)、Timer等。由于Reactor模式一般使用在网络编程中，因而这里一般指Socket Handle，即一个网络连接（Connection，在Java NIO中的Channel）。这个Channel注册到Synchronous Event Demultiplexer中，以监听Handle中发生的事件，对ServerSocketChannnel可以是CONNECT事件，对SocketChannel可以是READ、WRITE、CLOSE事件等。

Synchronous Event Demultiplexer：阻塞等待一系列的Handle中的事件到来，如果阻塞等待返回，即表示在返回的Handle中可以不阻塞的执行返回的事件类型。这个模块一般使用操作系统的select来实现。在Java NIO中用Selector来封装，当Selector.select()返回时，可以调用Selector的selectedKeys()方法获取Set，一个SelectionKey表达一个有事件发生的Channel以及该Channel上的事件类型。上图的“Synchronous Event Demultiplexer ---notifies--> Handle”的流程如果是对的，那内部实现应该是select()方法在事件到来后会先设置Handle的状态，然后返回。不了解内部实现机制，因而保留原图。

Initiation Dispatcher：用于管理Event Handler，即EventHandler的容器，用以注册、移除EventHandler等；另外，它还作为Reactor模式的入口调用Synchronous Event Demultiplexer的select方法以阻塞等待事件返回，当阻塞等待返回时，根据事件发生的Handle将其分发给对应的Event Handler处理，即回调EventHandler中的handle_event()方法。

Event Handler：定义事件处理方法：handle_event()，以供InitiationDispatcher回调使用。

Concrete Event Handler：事件EventHandler接口，实现特定事件处理逻辑。

**优点**

1）响应快，不必为单个同步时间所阻塞，虽然Reactor本身依然是同步的；

2）编程相对简单，可以最大程度的避免复杂的多线程及同步问题，并且避免了多线程/进程的切换开销；

3）可扩展性，可以方便的通过增加Reactor实例个数来充分利用CPU资源；

4）可复用性，reactor框架本身与具体事件处理逻辑无关，具有很高的复用性；

**缺点**

1）相比传统的简单模型，Reactor增加了一定的复杂性，因而有一定的门槛，并且不易于调试。

2）Reactor模式需要底层的Synchronous Event Demultiplexer支持，比如Java中的Selector支持，操作系统的select系统调用支持，如果要自己实现Synchronous Event Demultiplexer可能不会有那么高效。

3） Reactor模式在IO读写数据时还是在同一个线程中实现的，即使使用多个Reactor机制的情况下，那些共享一个Reactor的Channel如果出现一个长时间的数据读写，会影响这个Reactor中其他Channel的相应时间，比如在大文件传输时，IO操作就会影响其他Client的相应时间，因而对这种操作，使用传统的Thread-Per-Connection或许是一个更好的选择，或则此时使用Proactor模式。

原文地址：https://zhuanlan.zhihu.com/p/425969674

作者：Linux