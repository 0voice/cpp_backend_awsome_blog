# 【NO.213】【Redis】利用 Redis 实现分布式锁

## 1.技术背景

首先我们需要先来了解下什么是分布式锁，以及为什么需要分布式锁。

对于这个问题，我们可以简单将锁分为两种——内存级锁以及分布式锁，内存级锁即我们在 Java 中的 synchronized 关键字（或许加上进程级锁修饰更恰当些），而分布式锁则是应用在分布式系统中的一种锁机制。分布式锁的应用场景举例以下几种：

- 互联网秒杀
- 抢优惠卷
- 接口幂等校验

我们接下来以一段简单的秒杀系统中的判断库存及减库存来描述下为什么需要到分布式锁：

```
Copypublic String deductStock() throws InterruptedException {
    // 1.从 Redis 中获取库存值
    int stock = Integer.parseInt(stringRedisTemplate.opsForValue().get("stock"));
    // 2.判断库存
    if (stock > 0) {
        int readStock = stock - 1;
        // 3.从新设置库存
        stringRedisTemplate.opsForValue().set("stock", realStock + ""); 
        System.out.println("扣减成功，剩余库存：" + readStock + "");
    } else {
        System.out.println("扣减失败，库存不足");
    }
    return "end";
}
```

上面这段代码中，实现了电商系统中的一个简单小需求，即判断商品的剩余数量是否充足，充足则可以成功卖出商品，并将库存减去 1。我们很容易了解这段代码的目的。接下来我们就来一步一步地分析这段代码的缺陷。

## 2.基本实现

## 3.原子性问题

上面代码中的注释1~3部分，并没有实现原子性的逻辑。所以假设现在如果只剩下一件商品，那么可能会出现以下情况：

- 线程 A 运行到代码2，判断库存大于0，进入条件体中将 stock - 1 赋值给 readStock，在执行代码 3 前停止了下来；
- 线程 B 同样运行到代码2，判断出库存大于0（线程A并没有写回Redis），之后并没有停止，而是继续执行到方法结束；
- 线程 A 此时恢复执行，执行完代码 3，将库存写回 Redis。

现在我们就发现了问题，明明只有一件商品，却被两个线程卖出去了两次，这就是没有保证这部分代码的原子性所带来的安全问题。

那对于这个问题如何解决呢？

常规的方式自然就是加锁以保证并发安全。那么以我们 Java 自带的锁去保证并发安全，如下：

```
Copypublic Synchronized String deductStock() throws InterruptedException {    
    // 业务逻辑...
}
```

我们知道 synchronized 和 Lock 支持 JVM 内同一进程内的线程互斥，所以如果我们的项目是单机部署的话，到这里也就能保证这段代码的原子性了。不过以互联网项目来说，为了避免单点故障以及并发量的问题，一般都是以分布式的形式部署的，很少会以单机部署，这种情况就会带来新的问题。

## 4.分布式问题

刚刚我们将到了如果项目分布式部署的话，那么就会产生新的并发问题。接下来我们以 Nginx 配置负载均衡为例来演示并发问题，同样的请求可能会被分发到多台服务器上，那么我们刚刚所讲的 synchronized 或者 Lock 在此时就失效了。同样的代码，在 A 服务器上确实可以避免其他线程去竞争资源，但是此时 A 服务器上的那段 synchronized 修饰的方法并不能限制 B 服务器上的程序去访问那段代码，所以依旧会产生我们一开始所讲到的线程并发问题。

![img](https://pic4.zhimg.com/80/v2-f24ccd6b6bd815c03a85bfb17034d9e7_720w.webp)

那么如何解决掉这个问题呢？这个是否就需要 Redis 上场了，Redis 中有一个命令SETNX key value，SETNX 是 “SET if not exists” （如果不存在，则 SET）的缩写。那么这条指令只在 key 不存在的情况下，将键 key 的值设置为 value。若键 key 已经存在，则 SETNX 命令不做任何动作。

有了上面命令做支撑，同时我们了解到 Redis 是单线程模型（不要去计较它的网络读写和备份状态下的多线程）。那么我们就可以这么实现，当一个服务器成功的向 Redis 中设置了该命令，那么就认定为该服务器获得了当前的分布式锁，而其他服务器此时就只能一直等待该服务器释放了锁为止。我们来看下代码实现：

```
Copy// 为了演示方便，这里简单定义了一个常量作为商品的id
public static final String PRODUCT_ID = "100001";
public String deductStock() throws InterruptedException {
    // 通过 stringRedisTemplate 来调用 Redis 的 SETNX 命令，key 为商品的id，value的值在这不重要
    Boolean result = stringRedisTemplate.opsForValue().setIfAbsent(RODUCT_ID, "jojo");
    if (!result) {
        return "error";
    }
    int stock = Integer.parseInt(stringRedisTemplate.opsForValue().get("stock"));
    if (stock > 0) {
        int readStock = stock - 1;
        stringRedisTemplate.opsForValue().set("stock", realStock + ""); 
        System.out.println("扣减成功，剩余库存：" + readStock + "");
    } else {
        System.out.println("扣减失败，库存不足");
    }
    // 业务执行完成，删除PRODUCT_ID key
    stringRedisTemplate.delete(PRODUCT_ID);
    return "end";
}
```

到这里我们就成功地利用 Redis 实现了一把简单的分布式锁，那么这样实现是否就没有问题了呢？

## 5.锁释放问题

生产环境比我们想象中要复杂得多，上面代码并不能正真地运用在我们的生产环境中，我们可以试想一下，如果服务器 A 中的程序成功地给线程加锁，并且执行完了减库存的逻辑，但是最终却没有安全地运行stringRedisTemplate.delete(PRODUCT_ID)这行代码，也就是没有成功释放锁，那其他服务器就永远无法拿到 Redis 中的分布式锁了，也就会陷入死锁的状态。

解决这个方法，可能许多人都会想到想到——try-finally语句块，像下面代码这样：

```
Copypublic String deductStock() throws InterruptedException {
    Boolean result = stringRedisTemplate.opsForValue().setIfAbsent(RODUCT_ID, "jojo");
    if (!result) {
        return "error";
    }
    try {
        int stock = Integer.parseInt(stringRedisTemplate.opsForValue().get("stock"));
        if (stock > 0) {
            int readStock = stock - 1;
            stringRedisTemplate.opsForValue().set("stock", realStock + ""); 
            System.out.println("扣减成功，剩余库存：" + readStock + "");
        } else {
            System.out.println("扣减失败，库存不足");
        }
    } finally {
        //业务执行完成，删除PRODUCT_ID key
        stringRedisTemplate.delete(PRODUCT_ID);
    }
    return "end";
}
```

但是上面代码是否正真解决问题了呢？单看代码本身是没什么问题的，但是前面提到，生产环境是非常复杂的。我们假设这种情况：当线程在成功加锁之后，执行业务代码时，还没来得及删除 Redis 中的锁标志，此时，这台服务器宕机了，程序并没有想我们想象中地去执行 finally 块中的代码。这种情况也会使得其他服务器或者进程在后续过程中无法去获取到锁，从而导致死锁，最终导致业务崩溃的情况。所以说，对于锁释放问题来说，try-finally 语句块在这里还不够，那么我们就需要新的方法来解决这个问题了。

## 6.Redis 超时机制

Redis 中允许我们设置缓存的自动过期时间，我们可以将其引入我们上面的锁机制中，这样就算 finally 语句块中的释放语句没有被正确执行，Redis 中的缓存也能在设定时间内自动过期，不会形成死锁：

```
Copy// 设置过期时间
stringRedisTemplate.expire(lockKey, 10, TimeUnit.SECONDS);
```

当然如果只是简单的在代码中加入上述语句的话，还是有可能产生死锁的，因为加锁以及设置过期时间是分开来执行的，并不能保证原子性。所以为了解决这个问题，Redis 中也提供了将设置值与设置过期时间合一的操作，对于 Java 代码如下：

```
Copy// 将设置值与设置过期时间合一
stringRedisTemplate.opsForValue().opsForValue().setIfAbsent(lockKey, "jojo", 10, TimeUnit.SECONDS);
```

到这一步，我们可以确保 Redis 中我们上的锁，最终无论如何都能成功地被释放掉，避免了造成死锁的情况。但是以当前的代码实现来看，在一些高并发场景下还是可能产生锁失效的情况。我们可以试想一下，上面代码我们设置的过期时间为 10s，那么如果这个进程在 10s 内并没有完成这段业务逻辑，会产生什么样的情况？不过在此之前我们先将代码的公共部分抽出作一个组件类，这样有助于我们关注锁的逻辑。

## 7.代码集成

## 8.公共方法的提取

我们这里先定义一个 RedisLock 接口，代码如下所示：

```
Copypublic interface RedisLock {
    /**
     * 尝试加锁
     */
    boolean tryLock(String key, long timeout, TimeUnit unit);
    /**
     * 解锁操作
     */
    void releaseLock(String key);
}
```

接下来，我们基于上面已经实现的分布式锁的思路，来实现这个接口，代码如果所示：

```
Copypublic class RedisLockImpl implements RedisLock {
    @Autowired
    private StringRedisTemplate stringRedisTemplate;
    @Override
    public boolean tryLock(String key, long timeout, TimeUnit unit) {
        return stringRedisTemplate.opsForValue().setIfAbsent(key, "jojo", timeout, unit);
    }
    @Override
    public void releaseLock(String key) {
        stringRedisTemplate.delete(key);
    }
}

```

## 9.加锁&解锁的归一化

我们先来继续分析上面代码。从开发的角度来说，当一个线程从上到下执行一个需要加分布式锁的业务时，它首先需要进行加锁操作，当业务执行完毕后，再进行释放锁的操作。也就是先调用 tryLock() 函数再调用 releaseLock() 函数。

但是真正可靠代码并不依靠人性，其他开发人员有可能在编写代码的时候并没有调用 tryLock() 方法，而是直接调用了 releaseLock() 方法，并且可能在调用 releaseLock() 时传入的 Key 值与你调用 tryLock() 时传入的 Key 值是相同的，那么此时就可能出现问题：另一段代码在运行时，硬生生将你代码中加的锁给释放掉了，那么此时的锁就失效了。所以上述代码依旧是有不可靠的地方，锁的可能误删操作会使得程序存在很严重的问题。

那么针对这一问题，我们就需要实现加锁&解锁的归一化。

首先我们解释一下什么叫做加锁和解锁的归一化，简单来说，就是一个线程执行了加锁操作后，后续的解锁操作只能由该线程来执行，即加锁操作和解锁只能由同一线程来进行。

这里我们使用 ThreadLocal 和 UUID 来实现，代码如下：

```
Copypublic class RedisLockImpl implements RedisLock {
    @Autowired
    private StringRedisTemplate stringRedisTemplate;
    private ThreadLock<string> threadLock = new ThreadLock<>();
    @Override
    public boolean tryLock(String key, long timeout, TimeUnit unit) {
        String uuid = UUID.randomUUID().toString();
        threadlocal.set(uuid);
        return stringRedisTemplate.opsForValue().setIfAbsent(key, uuid, timeout, unit);
    }
    @Override
    public void releaseLock(String key) {
        if (threadLocal.get().equals(stringRedisTemplate.opsForValue().get(key))) {
            stringRedisTemplate.delete(key);
        }
    }
}
```

## 10.可重入发布式锁实现

上面的代码实现，可以保证当一个线程成功在 Redis 中设置了锁标志位后，其他线程再设置锁标志位时，返回 false。但是在一些场景下我们需要实现线程的重入，即相同的线程能够多次获取同一把锁，不需要等待锁释放后再去加锁。所以我们需要利用一些方式来实现分布式锁的可重入型，在 JDK 1.6 之后提供的内存级锁很多都支持可重入型，比如 synchronized 和 J.U.C 下的 Lock，其本质都是一样的，比对已经获得锁的线程是否与当前线程相同，是则重入，当释放锁时则需要根据重入的次数，来判断此时锁是否真正释放掉了。那么我们就按照这个思路来实现一个可重入的分布式锁：

```
Copypublic class RedisLockImpl implements RedisLock {
    @Autowired
    private StringRedisTemplate stringRedisTemplate;
    private ThreadLocal<String> threadLocal = new ThreadLocal<String>();
    private ThreadLocal<Integer> threadLocalInteger = new ThreadLocal<Integer>();
    @Override
    public boolean tryLock(String key, long timeout, TimeUnit unit) {
        Boolean isLocked = false;
        if (threadLocal.get() == null) {
            String uuid = UUID.randomUUID().toString();
            threadLocal.set(uuid);
            isLocked = stringRedisTemplate.opsForValue().setIfAbsent(key, uuid, timeout, unit);
        } else {
            isLocked = true;
        }
        // 重入次数加1
        if (isLocked) {
            Integer count = threadLocalInteger.get() == null ? 0 : threadLocalInteger.get();
            threadLocalInteger.set(count++);
        }
        return isLocked;
    }
    @Override
    public void releaseLock(String key) {
        // 判断当前线程所对应的uuid是否与Redis对应的uuid相同，再执行删除锁操作
        if (threadLocal.get().equals(stringRedisTemplate.opsForValue().get(key))) {
            Integer count = threadLocalInteger.get();
            // 计数器减为0时才能释放锁
            if (count == null || --count <= 0) {
                stringRedisTemplate.delete(key);
            }
        }
    }
}
```

## 11.分布式自旋锁实现

上面代码实现中，加入我们不能一次性获取到锁，那么就会直接返回失败，这对业务来说是十分不友好的，假设用户此时下单，刚好有另外一个用户也在下单，而且获取到了锁资源，那么该用户尝试获取锁之后失败，就只能直接返回“下单失败”的提示信息的。所以我们需要实现以自旋的形式来获取到锁，即不停的重试，基于这个想法，实现代码如下：

```
Copypublic class RedisLockImpl implements RedisLock {
    @Autowired
    private StringRedisTemplate stringRedisTemplate;
    private ThreadLocal<String> threadLocal = new ThreadLocal<>();
    private ThreadLocal<Integer> threadLocalInteger = new ThreadLocal<>();
    @Override
    public boolean tryLock(String key, long timeout, TimeUnit unit) {
        Boolean isLocked = false;
        if (threadLocal.get() == null) {
            String uuid = UUID.randomUUID().toString();
            threadLocal.set(uuid);
            isLocked = stringRedisTemplate.opsForValue().setIfAbsent(key, uuid, timeout, unit);
            // 尝试获取锁失败，则自旋获取锁直至成功
            if (!isLocked) {
                for (;;) {
                    isLocked = stringRedisTemplate.opsForValue().setIfAbsent(key, uuid, timeout, unit);
                    if (isLocked) {
                        break;
                    }
                }
            }
        } else {
            isLocked = true;
        }
        // 重入次数加1
        if (isLocked) {
            Integer count = threadLocalInteger.get() == null ? 0 : threadLocalIntger.get();
            threadLocalInteger.set(count++);
        }
        return isLocked;
    }
    @Override
    public void releaseLock(String key) {
        // 判断当前线程所对应的uuid是否与Redis对应的uuid相同，再执行删除锁操作
        if (threadLocal.get().equals(stringRedisTemplate.opsForValue().get(key))) {
            Integer count = threadLocalInteger.get();
            // 计数器减为0时才能释放锁
            if (count == null || --count <= 0) {
                stringRedisTemplate.delete(key);
            }
        }
    }
}
```

## 12.基础优化

## 13.超时问题

在高并发场景下，一把锁可能会被 N 多的进程竞争，获取锁后的业务代码也可能十分复杂，其运行时间可能偶尔会超过我们设置的过期时间，那么这个时候锁就会自动释放，而其他的进程就有可能来争抢这把锁，而此时原来获得锁的进程也在同时运行，这就有可能导致超卖现象或者其他并发安全问题。

那么如何解决这个问题呢？思路很简单，就是每隔一段时间去检查当前线程是否还在运行，如果还在运行，那么就继续更新锁的占有时长，而在释放锁的时候。具体的实现稍微复杂些，这里给出简易的代码实现：

```
Copypublic class RedisLockImpl implements RedisLock {
    @Autowired
    private StringRedisTemplate stringRedisTemplate;
    private ThreadLocal<String> threadLocal = new ThreadLocal<>();
    private ThreadLocal<Integer> threadLocalInteger = new ThreadLocal<>();
    @Override
    public boolean tryLock(String key, long timeout, TimeUnit unit) {
        Boolean isLocked = false;
        if (threadLocal.get() == null) {
            String uuid = UUID.randomUUID().toString();
            threadLocal.set(uuid);
            isLocked = stringRedisTemplate.opsForValue().setIfAbsent(key, uuid, timeout, unit);
            // 尝试获取锁失败，则自旋获取锁直至成功
            if (!isLocked) {
                for (;;) {
                    isLocked = stringRedisTemplate.opsForValue().setIfAbsent(key, uuid, timeout, unit);
                    if (isLocked) {
                        break;
                    }
                }
            }
            // 启动新的线程来定期检查当前线程是否执行完成，并更新过期时间
            new Thread(new UpdateLockTimeoutTask(uuid, stringRedisTemplate, key)).start();
        } else {
            isLocked = true;
        }
        // 重入次数加1
        if (isLocked) {
            Integer count = threadLocalInteger.get() == null ? 0 :threadLocalInteger.get();
            threadLocalInteger.set(count++);
        }
        return isLocked;
    }
    @Override
    public void releaseLock(String key) {
        // 判断当前线程所对应的uuid是否与Redis对应的uuid相同，再执行删除锁操作
        if (threadLocal.get().equals(stringRedisTemplate.opsForValue().get(key))) {
            Integer count = threadLocalInteger.get();
            // 计数器减为0时才能释放锁
            if (count == null || --count <= 0) {
                stringRedisTemplate.delete(key);
                // 获取更新锁超时时间的线程并中断
                long threadId = stringRedisTemplate.opsForValue().get(uuid);
                Thread updateLockTimeoutThread = ThreadUtils.getThreadByThreadId(threadId);
                if (updateLockTimeoutThread != null) {
                    // 中断更新锁超时时间的线程
                    updateLockTimeoutThread.interrupt();
                    stringRedisTemplate.delete(uuid);
                }
            }
        }
    }
}
```

接下来我们就创建 UpdateLockTimeoutTask 类来执行更新锁超时的时间。

```
Copypublic class UpdateLockTimeoutTask implements Runnable {
    private long uuid;
    private String key;
    private StringRedisTemplate stringRedisTemplate;
    public UpdateLockTimeoutTask(long uuid, StringRedisTemplate stringRedisTemplate, String key) {
        this.uuid = uuid;
        this.key = key;
        this.stringRedisTemplate = stringRedisTemplate;
    }
    @Override
    public void run() {
        // 将以uuid为Key，当前线程Id为Value的键值对保存到Redis中
        stringRedisTemplate.opsForValue().set(uuid, Thread.currentThread().getId());
        // 定期更新锁的过期时间
        while (true) {
            stringRedisTemplate.expire(key, 10, TimeUnit.SECONDS);
            try{
                // 每隔3秒执行一次
                Thread.sleep(10000);
            }catch (InterruptedException e){
                e.printStackTrace();
            }
        }
    }
}
```

最后，我们定义一个 ThreadUtils 工具类，这个工具类中我们定义一个根据线程 id 获取线程的方法 getThreadByThreadId(long threadId)，代码如下所示：

```
Copypublic class ThreadUtils {
    // 根据线程 id 获取线程句柄
    public static Thread getThreadByThreadId(long threadId) {
        ThreadGroup group = Thread.currentThread().getThreadGroup();
        while(group != null){
            Thread[] threads = new Thread[(int)(group.activeCount() * 1.2)];
            int count = group.enumerate(threads, true);
            for (int i = 0; i < count; i++){
                if (threadId == threads[i].getId()) {
                    return threads[i];
                }
            }
        }
    }
}
```

上述解决分布式锁失效的方案在分布式锁领域有一个专业的术语叫做 **“异步续命”** 。需要注意的是：当业务代码执行完毕后，我们需要停止更新锁超时时间的线程。所以，这里，我对程序的改动是比较大的，首先，将更新锁超时的时间任务重新定义为一个 UpdateLockTimeoutTask 类，并将 uuid 和StringRedisTemplate 注入到任务类中，在执行定时更新锁超时时间时，首先将当前线程保存到Redis中，其中Key为传递进来的 uuid。

## 14.高并发

如果我们系统中利用 Redis 来实现分布式锁，而 Redis 的读写并发量约合 5 万左右。假设现在一个秒杀业务需要支持的并发量超过百万级别，那么如果这 100万的并发全部打入 Redis 中去请求锁资源，Redis 将会直接挂掉。所以我们现在应该来考虑如何解决这个问题，即如何在高并发的环境下保证 Redis 实现的分布式锁的可用性，接下来我们就来考虑一下这个问题。

> 在高并发的商城系统中，如果采用 Redis 缓存数据，则 Redis 缓存的并发能力是关键，因为很多的前缀操作都需要访问 Redis。而异步削峰只是基本操作，关键还是要保证 Redis 的并发处理能力。

解决这个问题的关键思想就是：分而治之，将商品库存分开放。

我们在 Redis 中存储商品的库存数量时，可以将商品的库存进行“分割”存储来提升 Redis 的读写并发量。

例如，原来的商品的 id 为 10001，库存为1000件，在Redis中的存储为(10001, 1000)，我们将原有的库存分割为5份，则每份的库存为200件，此时，我们在Redis 中存储的信息为(10001_0, 200)，(10001_1, 200)，(10001_2, 200)，(10001_3, 200)，(10001_4, 200)。

![img](https://pic1.zhimg.com/80/v2-850e6e2e3f618a411504afc939925790_720w.webp)

此时，我们将库存进行分割后，每个分割的库存使用商品 id 加上一个数字标识来存储，这样，在对存储商品库存的每个 key 进行 Hash 运算时，得出的 Hash 结果是不同的，这就说明，存储商品库存的 Key 有很大概率不在 Redis 的同一个槽位中，这就能够提升 Redis 处理请求的性能和并发量。

分割库存后，我们还需要在 Redis 中存储一份商品 ID 和 分割库存后的 Key 的映射关系，此时映射关系的 Key 为商品的 ID，也就是 10001，Value 为分割库存后存储库信息的 Key，也就是 10001_0，10001_1，10001_2，10001_3，10001_4。在 Redis 中我们可以使用 List 来存储这些值。

在真正处理库存信息时，我们可以先从 Redis 中查询出商品对应的分割库存后的所有 Key，同时使用 AtomicLong 来记录当前的请求数量，使用请求数量对从Redis 中查询出的商品对应的分割库存后的所有Key的长度进行求模运算，得出的结果为0，1，2，3，4。再在前面拼接上商品id就可以得出真正的库存缓存的Key。此时，就可以根据这个Key直接到Redis中获取相应的库存信息。

同时，我们可以将分隔的不同的库存数据分别存储到不同的 Redis 服务器中，进一步提升 Redis 的并发量。

## 15.基础升级

## 16.移花接木

> 在高并发业务场景中，我们可以直接使用 Lua 脚本库（OpenResty）从负载均衡层直接访问缓存。

这里，我们思考一个场景：如果在高并发业务场景中，商品被瞬间抢购一空。此时，用户再发起请求时，如果系统由负载均衡层请求应用层的各个服务，再由应用层的各个服务访问缓存和数据库，其实，本质上已经没有任何意义了，因为商品已经卖完了，再通过系统的应用层进行层层校验已经没有太多意义了！而应用层的并发访问量是以百为单位的，这又在一定程度上会降低系统的并发度。

为了解决这个问题，此时，我们可以在系统的负载均衡层取出用户发送请求时携带的用户Id，商品id和活动Id等信息，直接通过 Lua 脚本等技术来访问缓存中的库存信息。如果商品的库存小于或者等于 0，则直接返回商品已售完的提示信息，而不用再经过应用层的层层校验了。

## 17.数据同步

![img](https://pic3.zhimg.com/80/v2-80d1bdd6a9f01e798e41c5dd84422ee2_720w.webp)

假设我们使用 Redis 来实现分布式锁，我们知道 Redis 是基于 CAP 中 AP 来实现的，那么就可能存在数据未同步的问题。具体的场景就是，我在 Redis 的 Master 上设置了锁标志，然而在 Redis 的主从节点上还未完全同步之时，Redis 主节点宕机了，那么此时从节点上就没有锁标志，从而导致并发安全问题。对于这个问题，常见的解法有两种，基于 Zookeeper 来实现分布式锁（废话），而另外一种就是 RedLock 了。

Redlock 同很多的分布式算法一样，也使用“大多数机制”。加锁时，它会向过半节点发送 set(key, value, nx=True, ex=xxx) 指令，只要过半节点 set 成功，就认为加锁成功。释放锁时，需要向所有节点发送 del 指令。不过 Redlock 算法还需要考虑出错重试、时钟漂移等很多细节，同时因为 RedLock 需要向多个节点进行读写，意味着其相比单实例 Redis 的性能会下降一些。

如果你很在乎高可用性，希望即使挂了一台 Redis 也完全不受影响，就应该考虑 Redlock。不过代价也是有的，需要更多的 Redis 实例，性能也下降了，代码上还需要引入额外的 library，运维上也需要特殊对待，这些都是需要考虑的成本。

原文链接：https://zhuanlan.zhihu.com/p/368633034

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)