# 【NO.306】从6种I/O模式谈谈协程的作用

假设磁盘上有10个文件，你需要读取的内存，那么你该怎么用代码实现呢？

在接着往下看之前，先自己想一想这个问题，看看自己能想出几种方法，各自有什么样的优缺点。想清楚了吗(还在看吗)，想清楚了我们继续往下看。

### **1.最简单的方法——串行**

这可能是大多数同学都能想到的最简单方法，那就是一个一个的读取，读完一个接着读下一个。用代码表示是这样的：

```text
for file in files:
  result = file.read()
  process(result)
```

是不是非常简单，我们假设每个文件读取需要1分钟，那么10个文件总共需要10分钟才能读取完成。这种方法有什么问题呢？实际上这种方法只有一个问题，那就是**慢**。除此之外，其它都是优点：

1. 代码简单，容易理解
2. 可维护性好，这代码交给谁都能维护的了(论程序员的核心竞争力在哪里)

那么慢的问题该怎么解决呢？有的同学可能已经想到了，为啥要一个一个读取呢？并行读取不就可以加快速度了吗。

### 2.**稍好的方法，并行**

那么，该怎么并行读取文件呢？显然，地球人都知道，线程就是用来并行的。我们可以同时开启10个线程，每个线程中读取一个文件。用代码实现就是这样的：

```text
def read_and_process(file):
  result = file.read()
  process(result)

def main():
  files = [fileA，fileB，fileC......]
  for file in files:
     create_thread(read_and_process,
                   file).run()
  # 等待这些线程执行完成
```

怎么样，是不是也非常简单。那么这种方法有什么问题吗？在开启10个线程这种问题规模下没有问题。现在我们把问题难度加大，假设有10000个文件，需要处理该怎么办呢？有的同学可能想10个文件和10000个文件有什么区别吗，直接创建10000个线程去读不可以吗？实际上这里的问题其实是说创建多个线程有没有什么问题。我们知道，虽然线程号称“轻量级进程”，虽然是轻量级但当数量足够可观时依然会有性能问题。这里的问题主要有这样几个方面：

1. 创建线程需要消耗系统资源，像内存等(想一想为什么？)
2. 调度开销，尤其是**当线程数量较多且都比较繁忙时**(同样想一想为什么？)
3. 创建多个线程不一定能加快I/O(如果此时设备处理能力已经饱和)

既然线程有这样那样的问题，那么还有没有更好的方法？答案是肯定的，并行编程不一定只能依赖线程这种技术。

### **3.事件驱动 + 异步**

没错，即使在单个线程中，使用事件驱动+异步也可以实现IO并行处理，Node.js就是非常典型的例子。为什么单线程也可以做到并行呢？这是基于这样两个事实：

1. 相对于CPU的处理速度来说，IO是非常慢的
2. IO不怎么需要计算资源

因此，当我们发起IO操作后为什么要一直等着IO执行完成呢？**在IO执行完之前的这段时间处理其它IO难道不香吗**？这就是为什么单线程也可以并行处理多个IO的本质所在。回到我们的例子，该怎样用事件驱动+异步来改造上述程序呢？实际上非常简单。首先我们需要创建一个event loop，这个非常简单：

```text
event_loop = EventLoop()
```

然后，我们需要往event loop中加入原材料，也就是需要监控的event，就像这样：

```text
def add_to_event_loop(event_loop, file):
   file.asyn_read() # 文件异步读取
   event_loop.add(file)
```

注意当执行file.asyn_read这行代码时会**立即返回**，不会阻塞线程，当这行代码返回时可能文件还没有真正开始读取，这就是所谓的异步。file.asyn_read这行代码的真正目的仅仅是**发起IO**，而不是等待IO执行完成。此后我们将该IO放到event loop中进行监控，也就是event_loop.add(file)这行代码的作用。一切准备就绪，接下来就可以等待event的到来了：

```text
while event_loop:
   file = event_loop.wait_one_IO_ready()
   process(file.result)
```

我们可以看到，event_loop会一直等待直到有文件读取完成（event_loop.wait_one_IO_ready()），这时我们就能得到读完的文件了，接下来处理即可。全部代码如下所示：

```text
def add_to_event_loop(event_loop, file):
   file.asyn_read() # 文件异步读取
   event_loop.add(file)

def main():
  files = [fileA，fileB，fileC ...]
  event_loop = EventLoop()
  for file in files:
      add_to_event_loop(event_loop, file)
      
  while event_loop:
     file = event_loop.wait_one_IO_ready()
     process(file.result)
```

### **4.多线程 VS 单线程 + event loop**

接下来我们看下程序执行的效果。在多线程情况下，假设有10个文件，每个文件读取需要1秒，那么很简单，并行读取10个文件需要1秒。那么对于单线程+event loop呢？我们再次看下event loop + 异步版本的代码：

```text
def add_to_event_loop(event_loop, file):
   file.asyn_read() # 文件异步读取
   event_loop.add(file)

def main():
  files = [fileA，fileB，fileC......]
  event_loop = EventLoop()
  for file in files:
      add_to_event_loop(event_loop, file)
      
  while event_loop:
     file = event_loop.wait_one_IO_ready()
     process(file.result)
```

对于add_to_event_loop，由于文件异步读取，因此该函数可以瞬间执行完成，真正耗时的函数其实就是event loop的等待函数，也就是这样：

```text
file = event_loop.wait_one_IO_ready()
```

我们知道，一个文件的读取耗时是1秒，因此该函数在1s后才能返回，但是，但是，接下来是重点。但是虽然该函数wait_one_IO_ready会等待1s，不要忘了，我们利用这两行代码同时发起了10个IO操作请求。

```text
for file in files:  add_to_event_loop(event_loop, file)
```

因此在event_loop.wait_one_IO_ready等待的1s期间，剩下的9个IO也完成了，也就是说event_loop.wait_one_IO_ready函数只是在第一次循环时会等待1s，但是此后的9次循环会直接返回，**原因就在于剩下的9个IO也完成了**。因此整个程序的执行耗时也是1秒。是不是很神奇，我们只用一个线程就达到了10个线程的效果。这就是event loop + 异步的威力所在。

### 5.**一个好听的名字：Reactors模式**

本质上，我们上述给出的event loop简单代码片段做的事情本质上和生物一样：给出刺激，做出反应。我们这里的给出event，然后处理event。这本质上就是所谓的Reactors模式。现在你应该明白所谓的Reactors模式是怎么一回事了吧。所谓的一些看上去复杂的异步框架其核心不过就是这里给出的代码片段，只是这些框架可以支持更加复杂的多阶段任务处理以及各种类型的IO。而我们这里给出的代码片段只能处理文件读取这一类IO。

### **6.把回调也加进来**

如果我们需要处理各种类型的IO上述代码片段会有什么问题吗？问题就在于上述代码片段就不会这么简单了，针对不同类型会有不同的处理方法，因此上述process方法需要判断IO类型然后有针对性的处理，这会使得代码越来越复杂，越来越难以维护。幸好我们也有应对策略，这就是回调。我们可以把IO完成后的处理任务封装到回调函数中，**然后和IO一并注册到event loop**。就像这样：

```text
def IO_type_1(event_loop, io):
  io.start()
  
  def callback(result):
    process_IO_type_1(result)
    
  event_loop.add((io, callback))
```

这样，event_loop在检测到有IO完成后就可以把该IO和关联的callback处理函数一并检索出来，直接调用callback函数就可以了。

```text
while event_loop:
   io, callback = event_loop.wait_one_IO_ready()
   callback(io.result)
```

看到了吧，这样event_loop内部就极其简洁了，even_loop根本就不关心该怎么处理该IO结果，这是注册的callback该关心的事情，event_loop需要做的仅仅就是拿到event以及相应的处理函数callback，然后调用该callback函数就可以了。现在我们可以同单线程来并发编程了，也使用callback对IO处理进行了抽象，使得代码更加容易维护，想想看还有没有什么问题？

### 7.**回调函数的问题**

虽然回调函数使得event loop内部更加简洁，但依然有其它问题，让我们来仔细看看回调函数：

```text
def start_IO_type_1(event_loop, io):
  io.start()
  
  def callback(result):
    process_IO_type_1(result)
    
  event_loop.add((io, callback))
```

从上述代码中你能看到什么问题吗？在上述代码中，一次IO处理过程被分为了两部分：

1. 发起IO
2. IO处理

其中第2部分放到了回调函数中，这样的异步处理天然不容易理解，这和我们熟悉的发起IO，等待IO完成、处理IO结果的同步模块有很大差别。这里的给的例子很简单，所以你可能不以为意，但是当处理的任务非常复杂时，可能会出现回调函数中嵌套回调函数，也就是回调地狱，这样的代码维护起来会让你怀疑为什么要称为一名苦逼的码农。

### 8.**问题出在哪里**

让我们再来仔细的看看问题出在了哪里？同步编程模式下很简单，但是同步模式下发起IO，线程会被阻塞，这样我们就不得不创建多个线程，但是创建过多线程又会有性能问题。这样为了发起IO后不阻塞当前线程我们就不得不采用异步编程+event loop。在这种模式下，异步发起IO不会阻塞调用线程，我们可以使用单线程加异步编程的方法来实现多线程效果，但是在这种模式下处理一个IO的流程又不得不被拆分成两部分，这样的代码违反程序员直觉，因此难以维护。那么很自然的，有没有一种方法既能有同步编程的简单理解又会有异步编程的非阻塞呢？答案是肯定的，这就是协程。

### **9.Finally！终于到了协程**

利用协程我可以以同步的形式来异步编程。这是什么意思呢？我们之所以采用异步编程是为了发起IO后不阻塞当前线程，而是用协程，程序员可以自行决定在什么时刻挂起当前协程，这样也不会阻塞当前线程。而协程最棒的一点就在于**挂起后可以暂存执行状态**，**恢复运行后可以在挂起点继续运行**，这样我们就不再需要像回调那样将一个IO的处理流程拆分成两部分了。因此我们可以在发起异步IO，这样不会阻塞当前线程，同时在发起异步IO后挂起当前协程，当IO完成后恢复该协程的运行，这样我们就可以实现同步的方式来异步编程了。接下来我们就用协程来改造一下回调版本的IO处理方式：

```text
def start_IO_type_1(io):
  io.start() # IO异步请求
  yield      # 暂停当前协程 
  process_IO_type_1(result) # 处理返回结果
```

此后我们要把该协程放到event loop中监控起来：

```text
def add_to_event_loop(io, event_loop):
  coroutine = start_IO_type_1(io)
  next(coroutine)
  event_loop.add(coroutine)
```

最后，当IO完成后event loop检索出相应的协程并恢复其运行：

```text
while event_loop:
   coroutine = event_loop.wait_one_IO_ready()
   next(coroutine)
```

现在你应该看出来了吧，上述代码中没有回调，也没有把处理IO的流程拆成两部分，整体的代码都是以同步的方式来编写，最棒的是依然能达到异步的效果。实际上你会看到，采用协程后我们依然需要基于事件编程的event loop，因为本质上**协程并没有改变IO的异步处理本质**，只要IO是异步处理的那么我们就必须依赖event loop来监控IO何时完成，只不过我们采用协程消除了对回调的依赖，整体编程方式上还是采用程序员最熟悉也最容易理解的同步方式。

### **10.总结**

看上去简简单单的IO实际上一点都不简单吧。为了高效进行IO操作，我们采用的技术是这样演进的：

1. 单线程串行 + 阻塞式IO(同步)
2. 多线程并行 + 阻塞式IO(并行)
3. 单线程 + 非阻塞式IO(异步) + event loop
4. 单线程 + 非阻塞式IO(异步) + event loop + 回调
5. Reactor模式(更好的单线程 + 非阻塞式IO+ event loop + 回调)
6. 单线程 + 非阻塞式IO(异步) + event loop + 协程

最终我们采用协程技术获取到了异步编程的高效以及同步编程的简单理解，这也是当今**高性能服务器**常用的一种技术组合。希望这篇文章能对你理解高效IO有所帮助。

原文地址：https://zhuanlan.zhihu.com/p/532807036

作者：linux