# 【NO.490】分布式缓存--缓存与数据库强一致场景下的方案

## **1. 概述**

缓存与数据库的强一致性，也称线性一致性，核心要求是：数据库中的值发生变更，缓存数据要实现同步复制，并且一旦操作完成，随后任意客户端的查询都必须返回这一新值。以下图为例，一旦`写入b`完成，必须保证读到；而写入过程中，认为值的跳变可能发生在某一瞬间，因此读到a或b都是可能的。数据库与缓存作为一个整体，在向外提供服务的过程中，无论数据是否变更过，都时刻保持数据一致，因为它内部的数据`仿佛`只有一份，即使并发访问不同节点。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216162214_88937.jpg)

## **2. 场景**

秒杀是一个比较典型的强一致场景，一般秒杀系统的库存同时保持在数据库与缓存中，如果查询缓存有数据，直接可以走秒杀流程，将数据库中的库存数量进行扣减，同时将最新的数据更新到缓存，使缓存中数据与数据库中数据保持强一致，这里只是拿秒杀的场景来举例，类似秒杀的场景有很多，像抢门票系统、12306抢火车票等，资源比较少用户比较多，需要在特定时间内进行抢购的业务场景。真实秒杀场景的设计，是在缓存中扣库存，不会直接在数据库中进行扣库存，因为数据库的性能远远比缓存差，所以本篇也只是拿类似秒杀这样的场景，来阐述强一致下的设计思想与相关实现。

## **3. 方案**

分布式系统里面，有个众所周知的理论，就是`CAP理论`，CAP即：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216162215_49747.jpg)

Consistency（一致性）
Availability（可用性）
Partition tolerance（分区容忍性）
这三个性质对应了分布式系统的三个指标。
而CAP理论说的就是一个分布式系统，不可能同时做到这三点。如果默认是分区的，那么只能选择P的情况下，出现了两种选择组合，AP与CP，AP保证可用性则牺牲了一致性，CP保证了一致性则牺牲了可用性，所以我们在讲缓存与数据库强一致的同时，不可避免牺牲了系统可用性的指标，所以看到12306网站这种体验不好，总是抢不到票，或者在一直提示排队中这种情况，就是系统可用性不佳的表现，因为火车站的票源是个稀缺资源，而且在各个站点之间查到的数量又是动态的，在这种强一致性下的业务场景，可用性必然会出现问题。这里不深入讨论12306网站具体是如何实现的，只是拿该场景做个引入。

假设现有一般抢购系统，某些商品搞促销活动，库存也就1000，抢完为止，在开枪时间未到来前，页面显示初始库存，在抢购过程中，只要刷新页面库存还有，按钮就不会置灰，还可以接着点击抢购，直到页面显示库存为0，活动结束。

这是个比较典型的`读多写少`场景，大量请求来集中访问，少部分请求能真正完成下单，我们很容易想到做`读写分离`，将商品的库存提前从数据库预加载到缓存，用户读的时候，从缓存读取数量，只要能看到数量，也就可以直接下单，至于用户能否抢到，得看用户运气了，让真正下单成功的用户去走后续付款操作。注意，这里对于某个用户下单成功后，后台要做的操作是先扣数据库库存数量，随后`实时同步`更新库存到缓存中。如果这一步更新不及时，很有可能数据库与缓存库存不一致，导致缓存中的数量比实际数据库库存还多，最终缓存库存减为零，而数据库已经是负数，结果导致超卖。

### **3.1 数据库与缓存双写+读取操作异步串行化**

当库存发生变化后，更新数据库，同时更新缓存，如果在读并发高的情况下，更新数据库与更新缓存的时间间隔中，被读操作打断，那么读到的将是缓存中旧的库存，数据库已经是新库存，此时会出现不一致；

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216162215_70124.jpg)

为了解决这种问题，应先更新数据库后，立即删除缓存

更新数据的时候，根据数据的唯一标识，将操作路由之后，发送到一个jvm内部的内存队列中，同时删除缓存。
读取数据的时候，那么将重新读取数据，并更新缓存的操作，根据唯一标识路由之后，也发送同一个jvm内部的队列中。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216162216_83444.jpg)

一个队列对应一个工作线程，每个工作线程串行拿到对应的操作，然后一条一条的执行，这样的话，一个数据变更的操作先执行，删除缓存。如果一个读请求过来，读到了空的缓存，就从数据库将更新后的值加载到缓存。如果并发高的情况下，会出现多个读操作并发的读数据库并加载缓存，可以先将缓存更新的请求发送到队列中，此时会在队列中积压，然后同步等待缓存更新完成。

这里有一个优化点，一个队列中，其实多个更新缓存请求串在一起是没意义的，因此可以做过滤，如果发现队列中已经有一个更新缓存的请求了，那么就不用再放个更新请求操作进去了，直接等待前面的更新操作请求完成即可；

如果请求还在等待时间范围内，不断轮询发现可以取到值了，那么就直接返回； 如果请求等待的时间超过一定时长，那么直接尝试从数据库中读取数据，并写入缓存。

实现代码如下：
`step1`: 注册监听器，初始化工作线程池和内存队列
在SpringBoot的启动类中注册如下监听器类InitListener

```
@EnableAutoConfiguration
@SpringBootApplication
@ComponentScan
@MapperScan("com.roncoo.eshop.inventory.mapper")
public class Application {
    /**
     * 注册监听器
     * @return
     */
    @SuppressWarnings({ "rawtypes", "unchecked" })
    @Bean
    public ServletListenerRegistrationBean servletListenerRegistrationBean() {
        ServletListenerRegistrationBean servletListenerRegistrationBean = 
                new ServletListenerRegistrationBean();
        servletListenerRegistrationBean.setListener(new InitListener());  
        return servletListenerRegistrationBean;
    }
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

监听器的实现类如下如下：

```
/**
 * 系统初始化监听器
 *
 */
public class InitListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent sce) {
        // 初始化工作线程池和内存队列
        RequestProcessorThreadPool.init();
    }
}
```

请求处理线程池的类如下，完成线程池与内存队列的初始化：

```
/**
 * 请求处理线程池：单例
 *
 */
public class RequestProcessorThreadPool {
    /**
     * 线程池
     */
    private ExecutorService threadPool = Executors.newFixedThreadPool(10);
    public RequestProcessorThreadPool() {
        RequestQueue requestQueue = RequestQueue.getInstance();
        for(int i = 0; i < 10; i++) {
            ArrayBlockingQueue<Request> queue = new ArrayBlockingQueue<Request>(100);
            requestQueue.addQueue(queue);  
            threadPool.submit(new RequestProcessorThread(queue));  
        }
    }
    /**
     * 
     * 静态内部类的方式，去初始化单例
     * 
     *
     */
    private static class Singleton {
        private static RequestProcessorThreadPool instance;
        static {
            instance = new RequestProcessorThreadPool();
        }
        public static RequestProcessorThreadPool getInstance() {
            return instance;
        }
    }
    public static RequestProcessorThreadPool getInstance() {
        return Singleton.getInstance();
    }
    /**
     * 初始化方法
     */
    public static void init() {
        getInstance();
    }
}
```

请求内存队列

```
/**
 * 请求内存队列
 *
 */
public class RequestQueue {
    /**
     * 内存队列
     */
    private List<ArrayBlockingQueue<Request>> queues = 
            new ArrayList<ArrayBlockingQueue<Request>>();
    /**
     * 标识位map
     */
    private Map<Integer, Boolean> flagMap = new ConcurrentHashMap<Integer, Boolean>();
    /**
     * 
     * 静态内部类的方式，去初始化单例
     * 
     *
     */
    private static class Singleton {
        private static RequestQueue instance;
        static {
            instance = new RequestQueue();
        }
        public static RequestQueue getInstance() {
            return instance;
        }
    }
    public static RequestQueue getInstance() {
        return Singleton.getInstance();
    }
    /**
     * 添加一个内存队列
     * @param queue
     */
    public void addQueue(ArrayBlockingQueue<Request> queue) {
        this.queues.add(queue);
    }
    /**
     * 获取内存队列的数量
     * @return
     */
    public int queueSize() {
        return queues.size();
    }
    /**
     * 获取内存队列
     * @param index
     * @return
     */
    public ArrayBlockingQueue<Request> getQueue(int index) {
        return queues.get(index);
    }
    public Map<Integer, Boolean> getFlagMap() {
        return flagMap;
    }
}
```

每个工作线程如下：

```
/**
 * 执行请求的工作线程
 *
 */
public class RequestProcessorThread implements Callable<Boolean> {
    /**
     * 自己监控的内存队列
     */
    private ArrayBlockingQueue<Request> queue;
    public RequestProcessorThread(ArrayBlockingQueue<Request> queue) {
        this.queue = queue;
    }
    @Override
    public Boolean call() throws Exception {
        try {
            while(true) {
                // ArrayBlockingQueue，线程安全的内存队列
                // Blocking就是说明，如果队列满了，或者是空的，那么都会在执行操作的时候，阻塞住
                Request request = queue.take();
                boolean forceRfresh = request.isForceRefresh();
                // 先做读请求的去重
                if(!forceRfresh) {
                    RequestQueue requestQueue = RequestQueue.getInstance();
                    Map<Integer, Boolean> flagMap = requestQueue.getFlagMap();
                    if(request instanceof ProductInventoryDBUpdateRequest) {
                        // 如果是一个更新数据库的请求，那么就将那个productId对应的标识设置为true
                        flagMap.put(request.getProductId(), true);
                    } else if(request instanceof ProductInventoryCacheRefreshRequest) {
                        Boolean flag = flagMap.get(request.getProductId());
                        // 如果flag是null
                        if(flag == null) {
                            flagMap.put(request.getProductId(), false);
                        }
                        // 如果是缓存刷新的请求，那么就判断，如果标识不为空，而且是true，就说明之前有一个这个商品的数据库更新请求
                        if(flag != null && flag) {
                            flagMap.put(request.getProductId(), false);
                        }
                        // 如果是缓存刷新的请求，而且发现标识不为空，但是标识是false
                        // 说明前面已经有一个数据库更新请求与一个缓存刷新请求了
                        if(flag != null && !flag) {
                            // 对于这种读请求，直接就过滤掉，不要放到后面的内存队列里面去了
                            return true;
                        }
                    }
                }
                System.out.println("===========日志===========: 工作线程处理请求，商品id=" + request.getProductId()); 
                // 执行这个request操作
                request.process();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return true;
    }
}
```

`step2`: 封装两种请求接口

```
/**
 * 请求接口
 *
 */
public interface Request {
    void process();
    Integer getProductId();
    boolean isForceRefresh();
}
```

更新数据库请求实现类

```
public class ProductInventoryDBUpdateRequest implements Request {
    /**
     * 商品库存
     */
    private ProductInventory productInventory;
    /**
     * 商品库存Service
     */
    private ProductInventoryService productInventoryService;
    public ProductInventoryDBUpdateRequest(ProductInventory productInventory,
            ProductInventoryService productInventoryService) {
        this.productInventory = productInventory;
        this.productInventoryService = productInventoryService;
    }
    @Override
    public void process() {
        System.out.println("===========日志===========: 数据库更新请求开始执行，商品id=" + productInventory.getProductId() + ", 商品库存数量=" + productInventory.getInventoryCnt());  
        // 修改数据库中的库存
        productInventoryService.updateProductInventory(productInventory);  
                // 删除redis中的缓存
        productInventoryService.removeProductInventoryCache(productInventory);
    }
    /**
     * 获取商品id
     */
    public Integer getProductId() {
        return productInventory.getProductId();
    }
    @Override
    public boolean isForceRefresh() {
        return false;
    }
}
```

更新缓存类请求类

```
/**
 * 重新加载商品库存的缓存
 * @author Administrator
 *
 */
public class ProductInventoryCacheRefreshRequest implements Request {
    /**
     * 商品id
     */
    private Integer productId;
    /**
     * 商品库存Service
     */
    private ProductInventoryService productInventoryService;
    /**
     * 是否强制刷新缓存
     */
    private boolean forceRefresh;
    public ProductInventoryCacheRefreshRequest(Integer productId,
            ProductInventoryService productInventoryService,
            boolean forceRefresh) {
        this.productId = productId;
        this.productInventoryService = productInventoryService;
        this.forceRefresh = forceRefresh;
    }
    @Override
    public void process() {
        // 从数据库中查询最新的商品库存数量
        ProductInventory productInventory = productInventoryService.findProductInventory(productId);
        System.out.println("===========日志===========: 已查询到商品最新的库存数量，商品id=" + productId + ", 商品库存数量=" + productInventory.getInventoryCnt());  
        // 将最新的商品库存数量，刷新到redis缓存中去
        productInventoryService.setProductInventoryCache(productInventory); 
    }
    public Integer getProductId() {
        return productId;
    }
    public boolean isForceRefresh() {
        return forceRefresh;
    }
}
```

`step3`: 请求异步执行Service封装

```
/**
 * 请求异步执行的service
 *
 */
public interface RequestAsyncProcessService {
    void process(Request request);
}
```

实现类：

```
/**
 * 请求异步处理的service实现
 * @author Administrator
 *
 */
@Service("requestAsyncProcessService")  
public class RequestAsyncProcessServiceImpl implements RequestAsyncProcessService {
    @Override
    public void process(Request request) {
        try {
            // 做请求的路由，根据每个请求的商品id，路由到对应的内存队列中去
            ArrayBlockingQueue<Request> queue = getRoutingQueue(request.getProductId());
            // 将请求放入对应的队列中，完成路由操作
            queue.put(request);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    /**
     * 获取路由到的内存队列
     * @param productId 商品id
     * @return 内存队列
     */
    private ArrayBlockingQueue<Request> getRoutingQueue(Integer productId) {
        RequestQueue requestQueue = RequestQueue.getInstance();
        // 先获取productId的hash值
        String key = String.valueOf(productId);
        int h;
        int hash = (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
        // 对hash值取模，将hash值路由到指定的内存队列中，比如内存队列大小8
        // 用内存队列的数量对hash值取模之后，结果一定是在0~7之间
        // 所以任何一个商品id都会被固定路由到同样的一个内存队列中去的
        int index = (requestQueue.queueSize() - 1) & hash;
        System.out.println("===========日志===========: 路由内存队列，商品id=" + productId + ", 队列索引=" + index);  
        return requestQueue.getQueue(index);
    }
}
```

`step4`: 两种请求controller封装

```
@Controller
public class ProductInventoryController {
    @Resource
    private RequestAsyncProcessService requestAsyncProcessService;
    @Resource
    private ProductInventoryService productInventoryService;
    /**
     * 更新商品库存
     */
    @RequestMapping("/updateProductInventory")
    @ResponseBody
    public Response updateProductInventory(ProductInventory productInventory) {
        System.out.println("===========日志===========: 接收到更新商品库存的请求，商品id=" + productInventory.getProductId() + ", 商品库存数量=" + productInventory.getInventoryCnt());
        Response response = null;
        try {
            Request request = new ProductInventoryDBUpdateRequest(
                    productInventory, productInventoryService);
            requestAsyncProcessService.process(request);
            response = new Response(Response.SUCCESS);
        } catch (Exception e) {
            e.printStackTrace();
            response = new Response(Response.FAILURE);
        }
        return response;
    }
    /**
     * 获取商品库存
     */
    @RequestMapping("/getProductInventory")
    @ResponseBody
    public ProductInventory getProductInventory(Integer productId) {
        System.out.println("===========日志===========: 接收到一个商品库存的读请求，商品id=" + productId);  
        ProductInventory productInventory = null;
        try {
            Request request = new ProductInventoryCacheRefreshRequest(
                    productId, productInventoryService, false);
            requestAsyncProcessService.process(request);
            // 将请求扔给service异步去处理以后，就需要while(true)一会儿，在这里hang住
            // 去尝试等待前面有商品库存更新的操作，同时缓存刷新的操作，将最新的数据刷新到缓存中
            long startTime = System.currentTimeMillis();
            long endTime = 0L;
            long waitTime = 0L;
            // 等待超过200ms没有从缓存中获取到结果
            while(true) {
                // 一般公司里面，面向用户的读请求控制在200ms就可以了
                if(waitTime > 200) {
                    break;
                }
                // 尝试去redis中读取一次商品库存的缓存数据
                productInventory = productInventoryService.getProductInventoryCache(productId);
                // 如果读取到了结果，那么就返回
                if(productInventory != null) {
                    System.out.println("===========日志===========: 在200ms内读取到了redis中的库存缓存，商品id=" + productInventory.getProductId() + ", 商品库存数量=" + productInventory.getInventoryCnt());  
                    return productInventory;
                }
                // 如果没有读取到结果，那么等待一段时间
                else {
                    Thread.sleep(20);
                    endTime = System.currentTimeMillis();
                    waitTime = endTime - startTime;
                }
            }
            // 直接尝试从数据库中读取数据
            productInventory = productInventoryService.findProductInventory(productId);
            if(productInventory != null) {
                // 将缓存刷新一下
                // 这个过程，实际上是一个读操作的过程，但是没有放在队列中串行去处理，还是有数据不一致的问题
                request = new ProductInventoryCacheRefreshRequest(
                        productId, productInventoryService, true);
                requestAsyncProcessService.process(request);
                // 代码会运行到这里，只有三种情况：
                // 1、就是说，上一次也是读请求，数据刷入了redis，但是redis LRU算法给清理掉了，标志位还是false
                // 所以此时下一个读请求是从缓存中拿不到数据的，再放一个读Request进队列，让数据去刷新一下
                // 2、可能在200ms内，就是读请求在队列中一直积压着，没有等待到它执行
                // 所以就直接查一次库，然后给队列里塞进去一个刷新缓存的请求
                // 3、数据库里本身就没有，缓存穿透，穿透redis，请求到达mysql库
                return productInventory;
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return new ProductInventory(productId, -1L);  
    }
}
```

上述实现过程中有两个优化点
`优化点1`: 读请求去重优化
因为那个写请求肯定会更新数据库，然后那个读请求肯定会从数据库中读取最新数据，然后刷新到缓存中，自己只要hang一会儿就可以从缓存中读到数据了。

`优化点2`: 空数据读请求过滤优化
可能某个数据，在数据库里面压根儿就没有，那么那个读请求是不需要放入内存队列的，而且读请求在controller那一层，直接就可以返回了，不需要等待。

如果缓存里没数据，就两个情况，第一个是数据库里就没数据，缓存肯定也没数据；第二个是数据库更新操作过来了，先删除了缓存，此时缓存是空的，但是数据库里是有的。我们做了之前的读请求去重优化，用了一个flag map，只要前面有数据库更新操作，flag就肯定是存在的，你只不过可以根据true或false，判断你前面执行的是写请求还是读请求。但是如果flag压根儿就没有呢，就说明这个数据，无论是写请求，还是读请求，都没有过。那这个时候过来的读请求，发现flag是null，就可以认为数据库里肯定也是空的，那就不会去读取了。或者说，我们也可以认为每个商品有一个最初始的库存，但是因为最初始的库存肯定会同步到缓存中去的，有一种特殊的情况，就是说，商品库存本来在redis中是有缓存的，但是因为redis内存满了，就给干掉了，但是此时数据库中是有值的，那么在这种情况下，可能就是之前没有任何的写请求和读请求的flag的值，此时还是需要从数据库中重新加载一次数据到缓存中的。

### **3.2 方案改进**

上述方案是笔者的朋友在互联网大厂的经验总结，在思路上是没有问题的，但是在工业级项目的落地过程中，会有不少问题。
比如机器突然挂了，那内存队列就会`丢数据`，再比如，如果并发读的数量很大，那么`内存队列积压`的数据为会越来越多，导致后面的请求也有可能hang在那很长时间，一直读不到数据。

`问题1: 如果内存队列丢数据，怎么办？`
这种情况比较常见，比如机器突然挂了，内存队列数据丢了，该如何处理？首先明确一点，这里的请求都是`同步阻塞`的，如果业务系统挂了，那上游的路由网关会出现请求异常或者超时，外部系统或者外部用户请求也会异常或者超时，那调用端会重试请求，直到机器重启ok，请求会再次进队列，数据只不过重新进入队列。

`问题2：数据积压如何处理？`
这里确实会存在内存队列积压大量的读请求，导致后续的请求hang在那几秒甚至十几秒都没有得到处理。

```
问题3：业务系统需要完成强一致的需求，需要引入内存队列，路由网关，导致大量的开发成本，并且稍微控制不好，就会出现隐藏的bug。
```

针对以上问题，作如下改进：

```
改进点1：封装缓存代理客户端与缓存服务端，引入RocketMQ，将消息写入带上时间戳版本。
```

封装缓存客户端，一方面是省去路由网关，另一方面是充当消息写入的角色。RocketMQ替换原来的内存队列，因为消息本身按照key分区写入，就能保证相同的key会写到相同的分区队列里面。然后换成代理服务端按照消息有序消费，再写入缓存集群，架构设计如下：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216162216_72617.jpg)

上图中，首先事务写入保证消息不会丢，其次写入时带上时间戳作为版本号，防止读取的旧值后写入 ，更新的新值先写入，当写入缓存集群中时，比较时间戳是否是较新的，防止旧值覆盖新值。

`改进点2：解决MQ吞吐问量问题，缓存代理服务端使用内存队列。`
针对改点1中的情形，为保证消息绝对有序，只有一个线程消费MQ中一个分区的消息，再写入缓存，会带来吞吐量的下降，因此在缓存代理服务端使用多个内存队列，让多个线程依次消费多个队列，增加吞吐量。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216162217_39531.jpg)

`改进点3: 增加手动ack，增加消息进入重试队列与死信队列的机制。`
如果Redis缓存挂了，此时需要通过更新缓存后拿到结果，并手动通知ack到消息中间件，确保消息消费者处理完后，才丢弃该消息，防止消息在消费者端丢失，时间戳来保证更新缓存幂等性，此外一直更新缓存失败的消息进入消息中间件进入重试队列死信队列，待下次发消息后再消费。

### **3.3 终极方案**

通过上述改进，一套思想完备，可落地到生产级的方案基本完成，有人会说这不是`分布式锁`的思想么？说对了，多年前，分布式锁还没有发展成熟的时候，就是通过类似的这种消息正中间将写入分区的操作串行化，进行消费，再通过幂等性保证最终写入不会被乱序覆盖，现在分布式锁的实现已经比较成熟，完全可以用分布式锁来解决，比如用Redis的提供的客户端Redission来实现，不但简化流程，而且保证只有抢到锁的线程才可以更新数据库与缓存，再释放锁，当然加锁与释放锁的占用时间也是较快的，因为更新数据与写一条缓存也就几毫秒或者是十几毫秒的时间，可以保证后续更新的操作在很快时间内可以再次抢到锁。

## **4. 总结**

本篇讲述了数据库与缓存双写在强一致下的实现思路与方案，从一开始的方案设计到落地，再到落地后的优化改进，最后到比较可行又简单的方式，你会发现，好的架构不是一步到位，而是逐步`演进`而来；其次几年前的方案，也许比较`合适`，但是现在看起来就会显得过于复杂，原因是解决特定问题的专业技术已经出现，专门解决了该类问题，最终通过重构来解决之前过于复杂又容易出问题的方案，由此也不难发现，好的架构是比较`简单`的。

原文链接：https://zhuanlan.zhihu.com/p/501195720

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)