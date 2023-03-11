# 【NO.152】同步与异步，回调与协程

**目录**

- 概念上下文：
- 同步的方式：
- 异步加回调的方式：
- 异步协程方式：
- 总结：

**正文**

　　本文主要介绍在网络请求中的同步与异步，以及异步的表现形式： 回调与协程，并通过python代码展示各自的优缺点。

## **1.概念上下文：**

回到顶部

　　当提到同步与异步，大家不免会想到另一组词语：阻塞与非阻塞。通常，同时提到这个这几个词语一般实在讨论network io的时候，在《unix network programming》中有详尽的解释，网络中也有许多讲解生动的文章。

　　本文所讨论的同步与异步，是指对于请求的发起者，是否需要等到请求的结果（同步），还是说请求完毕的时候以某种方式通知请求发起者（异步）。在这个语义环境下，阻塞与非阻塞，是指请求的受理者在处理某个请求的状态，如果在处理这个请求的时候不能做其它事情（请求处理时间不确定），那么称之为阻塞，否则为非阻塞。

　　举个例子，我去柜台办理业务，那我是请求者，柜员时受理者。如果我在柜台一直等着柜员办理，直到办理完毕，那么对于我来说，就是同步的；如果我只是在柜员那里登记，然后到一边歇着，等柜员办理完毕之后告诉我结果，那么就是异步的。对于柜员，办理业务的时候可能需要等待打印机打印，如果在这个时候柜员去处理其他人的业务，那么就是非阻塞的，如果一定得到把我的业务办完再接待下一位顾客，那么就是阻塞的。

　　本文站在请求发起者的角度来思考同步与异步，在实际开发中，一个最简单的例子就是http请求。假设这么一个场景，程序需要访问两个网址（通过url），如果只有一个线程。那么同步与异步分别怎么处理呢

## **2.同步的方式：**

回到顶部

　　python的urllib2提供了http请求的功能，我们来看看代码：

```
1 def http_blockway(url1, url2):
 2     import urllib2 as urllib
 3     import time
 4     begin = time.time()
 5     data1 = urllib.urlopen(url1).read()
 6     data2 = urllib.urlopen(url2).read()
 7     print len(data1), len(data2)
 8     print 'http_blockway cost', time.time() - begin
 9 
10 url_list = [ 'http://xlambda.com/gevent-tutorial/','https://www.bing.com']
11 if __name__ == '__main__':
12     http_blockway(*url_list)
```

运行结果：

　　45706 121483
　　http_blockway cost 2.22000002861

注意：对代码运行的时间应该多次执行求平均值，这里只是简单作为参考。下同。

　　python的urllib是同步的，即当一个请求结束之后才能发起下一个请求，我们知道http请求基于tcp，tcp又需要三次握手建立连接（https的握手会更加复杂），在这个过程中，程序很多时候都在等待IO，CPU空闲，但是又不能做其他事情。但同步模式的优点是比较直观，符合人类的思维习惯: 那就是一件一件的来，干完一件事再开始下一件事。在同步模式下，要想发挥duo多喝CPU的威力，可以使用多进程或者多线程。

## 3.**异步加回调的方式：**

回到顶部

如果发出了请求就立即返回，这个时候程序可以做其他事情，等请求完成了时候通过某种方式告知结果，然后请求者再继续再来处理请求结果，那么我们称之为异步，最常见的就是回调（callback），在python中，tornado提供了异步的http请求。对于上面的场景，我们来看看代码：

```
1 import tornado
 2 from tornado.httpclient import AsyncHTTPClient
 3 import time, sys
 4 
 5 def http_callback_way(url1, url2):
 6     http_client = AsyncHTTPClient()
 7     begin = time.time()
 8     count = [0]
 9     def handle_result(response, url):
10         print('%s : handle_result with url %s' % (time.time(), url))
11         count[0] += 1
12         if count[0] == 2:
13             print 'http_callback_way cost', time.time() - begin
14             sys.exit(0)
15 
16     http_client.fetch(url1,lambda res, u = url1:handle_result(res, u))
17     print('%s here between to request' % time.time())
18     http_client.fetch(url2,lambda res, u = url2:handle_result(res, u))
19 
20 url_list = [ 'http://xlambda.com/gevent-tutorial/','https://www.bing.com']
21 if __name__ == '__main__':
22     http_callback_way(*url_list)
23     tornado.ioloop.IOLoop.instance().start()
```

运行结果：

　　1487292402.45 **here between to reques**t
　　1487292403.09 : handle_result with url [http://xlambda.com/gevent-tutorial/](https://link.zhihu.com/?target=http%3A//xlambda.com/gevent-tutorial/)
　　1487292403.21 : handle_result with url [https://www.bing.com](https://link.zhihu.com/?target=https%3A//www.bing.com)
　　http_callback_way cost 0.759999990463

从代码可以看到，对请求的结果是放在一个额外的函数（handle_result）中进行的，这个处理函数在发出请求的时候注册的，这就是回调。从运行结果可以看到，在发出第一个请求之后就立即返回了（先于请求结果），而且运行时间大为缩短，这就是异步的优势所在：不用在IO上等待，在单核CPU上就有更好的性能。但是callback这种形式，导致代码逻辑因为异步请求分离，不符合人类的思维习惯，因为不直观。再来看一个很常见的例子：请求一个网址A，根据网页的内容来确定是接着访问B还是C，如果使用异步，那代码是这样的： 　　

```
1 import tornado
 2 from tornado.httpclient import AsyncHTTPClient
 3 http_client = AsyncHTTPClient()
 4 def handle_request_final(response):
 5     print('finally we got the result, do sth here')
 6 
 7 def handle_request_first(response, another1, another2):
 8     if response.error or 'some word' in response.body:
 9         target = another1
10     else:
11         target = another2
12     http_client.fetch(target, handle_request_final)
13 
14 def http_callback_way(url, another1, another2):
15     http_client.fetch(url, lambda res, u1 = another1, u2 = another2:handle_request_first(res, u1, u2))
16 
17 url_list = [ 'https://www.baidu.com', 'https://www.google.com','https://www.bing.com']
18 if __name__ == '__main__':
19     http_callback_way(*url_list)
20     tornado.ioloop.IOLoop.instance().start()
```

**代码表达的逻辑是连贯的，但代码的变现形式却是割裂的**，在这里分散到了三个函数里面。对于程序员来说，这样的代码难以阅读，不容易看一眼就大概明白其意图，而且这样的代码是难以维护的，更糟糕的是，编程实现中还得保存或者传递上下文（比如函数中的参数 another1, another2）。

　　在python中，由于lambda的功能还是比较弱，所以回调一般都是另外命令一个函数。而在javascript，特别是以异步IO为基石的NodeJS中，由于匿名函数的强大，很轻易嵌套实现callback，这也就出现了让编码者沉默、维护者流泪的callback hell。当然promise对callback hell有一定的改善，但还有没有更好的办法呢？

## 4.**异步协程方式：**

回到顶部

　　这个时候协程就出马了，本来是异步调用，但是程序上看上去变成了“同步”。关于协程，python中可以用原声的generator，也可以使用更严格的greenlet。首先来看看tornado封装的协程：

```
1 import tornado, sys, time
 2 from tornado.httpclient import AsyncHTTPClient
 3 from tornado import gen
 4 
 5 def http_generator_way(url1, url2):
 6     begin = time.time()
 7     count = [0]
 8     @gen.coroutine
 9     def do_fetch(url):
10         http_client = AsyncHTTPClient()
11         response = yield http_client.fetch(url, raise_error = False)
12         print url, response.error
13         count[0] += 1
14         if count[0] == 2:
15             print 'http_generator_way cost', time.time() - begin
16             sys.exit(0)
17 
18     do_fetch(url1)
19     do_fetch(url2)
20 
21 url_list = [ 'http://xlambda.com/gevent-tutorial/','https://www.bing.com']
22 if __name__ == '__main__':
23     http_generator_way(*url_list)
24     tornado.ioloop.IOLoop.instance().start()
```

运行结果：

　　[http://xlambda.com/gevent-tutorial/](https://link.zhihu.com/?target=http%3A//xlambda.com/gevent-tutorial/) None
　　[https://www.bing.com](https://link.zhihu.com/?target=https%3A//www.bing.com) None
　　http_generator_way cost 1.05999994278

　　从运行结果可以看到，效率还是优于同步的，而代码看起来是顺序执行的，但事实上有某种意义上的并行。代码中使用了decorator 和 yield关键字，在看看使用gevent的代码：

```
1 def http_coroutine_way(url1, url2):
 2     import gevent, time
 3     from gevent import monkey
 4     monkey.patch_all()
 5     begin = time.time()
 6 
 7     def looks_like_block(url):
 8         import urllib2 as urllib
 9         data = urllib.urlopen(url).read()
10         print url, len(data)
11 
12     gevent.joinall([gevent.spawn(looks_like_block, url1), gevent.spawn(looks_like_block, url2)])
13     print('http_coroutine_way cost', time.time() - begin)
14 
15 url_list = [ 'http://xlambda.com/gevent-tutorial/','https://www.bing.com']
16 if __name__ == '__main__':
17     http_coroutine_way(*url_list)
```

　　代码中没有特殊的关键字，使用的API也是跟同步方式一样的，这就是gevent的牛逼之处，通过monkey_patch就是原来的同步API变成异步，gevent的介绍可以参见《gevent调度流程解析》。

　　协程与回调对比，优势一目了然：代码更清晰直观，更加复合思维习惯。但协程也不是银弹，首先，协程是个新概念，需要花时间去理解；其次，程序员心里也得牢牢记住，在看似在一起的两句代码中间可能插入了很大其它逻辑（即使是在单线程）。但总体来说，协程给出了人们解决问题的新思路，利大于弊，在其他语言（C# Lua golang）中也有支持，还是值得程序员去了解和学习

## 5.**总结：**

回到顶部

　　最后，简单总结一下，同步比较直观，编码和维护都比较容易，但是效率低，绝大多数时间都是不能忍的。异步效率高，对于callback这种形式逻辑容易被代码割裂，代码不直观，而异步协程的方式既看起来直观，又在效率上有保证。

看完的都是真爱，点个赞再走呗？您的「三连」就是我的最大动力！

原文链接：https://zhuanlan.zhihu.com/p/330579450

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)