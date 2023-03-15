# 【NO.314】redis7.0源码阅读：Redis中的IO多线程（线程池）

## 1.Redis中的IO多线程原理

![img](https://pic1.zhimg.com/80/v2-692b92d99f7f593e8f48505a26585604_720w.webp)

服务端收到一条信息，给它deconde成一条命令

然后根据命令获得一个结果(reply)

然后将结果encode后，发送回去

![img](https://pic2.zhimg.com/80/v2-8195df68ac78569eea083770801aab01_720w.webp)

redis的单线程是指，命令执行(logic)都是在单线程中运行的

接受数据read和发送数据write都是可以在io多线程（线程池）中去运行

在Redis中，生产者也可以作为消费者，反之亦然，没有明确界限。

![img](https://pic2.zhimg.com/80/v2-3f1dc75e56371654568370361c721ecd_720w.webp)

## 2.设置io多线程（调试设置）

在redis.conf中

设置io-threads-do-reads yes就可以开启io多线程

设置io-threads 2,设置为2（为了方便调试,真正使用的时候，可以根据需要设置），其中一个为主线程，另外一个是io线程

![img](https://pic3.zhimg.com/80/v2-e8ed7ea9f3fecc749e958e9e24f65a5a_720w.webp)

在networking.c中找到stopThreadedIOIfNeeded，如果在redis-cli中输入一条命令，是不会执行多线程的，因为它会判断，如果pending（需要做的命令）个数比io线程数少，就不会执行多线程

因此提前return 0，确保执行多线程,便于调试

```text
int stopThreadedIOIfNeeded(void) {
    int pending = listLength(server.clients_pending_write);

    /* Return ASAP if IO threads are disabled (single threaded mode). */
    if (server.io_threads_num == 1) return 1;
    return 0;//为了调试，提前退出（自己添加的一行）
    if (pending < (server.io_threads_num*2)) {
        if (server.io_threads_active) stopThreadedIO();
        return 1;
    } else {
        return 0;
    }
}
```

到此为止，只需要，运行redis-server,在networking.c的 readQueryFromClient中打个断点，然后在redis-cli中输入任意set key value就可以进入io多线程，进行调试

下图可以看到箭头指向的两个线程，一个是主线程，另一个是io线程

![img](https://pic1.zhimg.com/80/v2-5022d9afdf925c375a3412ab18f10d00_720w.webp)



## 3.Redis中的IO线程池

### 3.1读取任务`readQueryFromClient`

`postponeClientRead(c)`就是判断io多线程模式，并将任务添加到 任务队列中

```text
void readQueryFromClient(connection *conn) { 
    client *c = connGetPrivateData(conn);
    int nread, big_arg = 0;
    size_t qblen, readlen;

    /* Check if we want to read from the client later when exiting from
     * the event loop. This is the case if threaded I/O is enabled. */
    if (postponeClientRead(c)) return; 
	//后面省略......
}
```

### 3.2主线程将 待读客户端 添加到Read任务队列（生产者）`postponeClientRead`

如果是io多线程模式，那么将任务添加到任务队列。
（这个函数名的意思，延迟读，就是将任务加入到任务队列，后续去执行）

```text
int postponeClientRead(client *c) {
    if (server.io_threads_active &&
        server.io_threads_do_reads &&
        !ProcessingEventsWhileBlocked &&
        !(c->flags & (CLIENT_MASTER|CLIENT_SLAVE|CLIENT_BLOCKED)) &&
        io_threads_op == IO_THREADS_OP_IDLE)
    {
        listAddNodeHead(server.clients_pending_read,c);//往任务队列中插入任务
        c->pending_read_list_node = listFirst(server.clients_pending_read);
        return 1;
    } else {
        return 0;
    }
}
```

### 3.3多线程Read IO任务 `handleClientsWithPendingReadsUsingThreads`

基本原理和多线程Write IO是一样的，直接看多线程Write IO就行了。

其中`processInputBuffer`是解析协议

```text
int handleClientsWithPendingReadsUsingThreads(void) {
    if (!server.io_threads_active || !server.io_threads_do_reads) return 0;
    int processed = listLength(server.clients_pending_read);
    if (processed == 0) return 0;

    /* Distribute the clients across N different lists. */
    listIter li;
    listNode *ln;
    listRewind(server.clients_pending_read,&li);
    int item_id = 0;

    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        int target_id = item_id % server.io_threads_num;
        listAddNodeTail(io_threads_list[target_id],c);
        item_id++;
    }

    /* Give the start condition to the waiting threads, by setting the
     * start condition atomic var. */
    io_threads_op = IO_THREADS_OP_READ;
    for (int j = 1; j < server.io_threads_num; j++) {
        int count = listLength(io_threads_list[j]);
        setIOPendingCount(j, count);
    }

    /* Also use the main thread to process a slice of clients. */
    listRewind(io_threads_list[0],&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        readQueryFromClient(c->conn);
    }
    listEmpty(io_threads_list[0]);

    /* Wait for all the other threads to end their work. */
    while(1) {
        unsigned long pending = 0;
        for (int j = 1; j < server.io_threads_num; j++)
            pending += getIOPendingCount(j);
        if (pending == 0) break;
    }

    io_threads_op = IO_THREADS_OP_IDLE;

    /* Run the list of clients again to process the new buffers. */
    while(listLength(server.clients_pending_read)) {
        ln = listFirst(server.clients_pending_read);
        client *c = listNodeValue(ln);
        listDelNode(server.clients_pending_read,ln);
        c->pending_read_list_node = NULL;

        serverAssert(!(c->flags & CLIENT_BLOCKED));

        if (beforeNextClient(c) == C_ERR) {
            /* If the client is no longer valid, we avoid
             * processing the client later. So we just go
             * to the next. */
            continue;
        }

        /* Once io-threads are idle we can update the client in the mem usage buckets */
        updateClientMemUsageBucket(c);

        if (processPendingCommandsAndResetClient(c) == C_ERR) {
            /* If the client is no longer valid, we avoid
             * processing the client later. So we just go
             * to the next. */
            continue;
        }

        if (processInputBuffer(c) == C_ERR) {
            /* If the client is no longer valid, we avoid
             * processing the client later. So we just go
             * to the next. */
            continue;
        }

        /* We may have pending replies if a thread readQueryFromClient() produced
         * replies and did not install a write handler (it can't).
         */
        if (!(c->flags & CLIENT_PENDING_WRITE) && clientHasPendingReplies(c))
            clientInstallWriteHandler(c);
    }

    /* Update processed count on server */
    server.stat_io_reads_processed += processed;

    return processed;
}
```

### 3.4多线程write IO任务（消费者）handleClientsWithPendingWritesUsingThreads

1.判断是否有必要开启IO多线程

2.如果没启动IO多线程，就启动IO多线程

3.负载均衡：write任务队列，均匀分给不同io线程

4.启动io子线程

5.主线程执行io任务

6.主线程等待io线程写结束

```text
/* This function achieves thread safety using a fan-out -> fan-in paradigm:
 * Fan out: The main thread fans out work to the io-threads which block until
 * setIOPendingCount() is called with a value larger than 0 by the main thread.
 * Fan in: The main thread waits until getIOPendingCount() returns 0. Then
 * it can safely perform post-processing and return to normal synchronous
 * work. */
int handleClientsWithPendingWritesUsingThreads(void) {
    int processed = listLength(server.clients_pending_write);
    if (processed == 0) return 0; /* Return ASAP if there are no clients. */

    /* If I/O threads are disabled or we have few clients to serve, don't
     * use I/O threads, but the boring synchronous code. */
    if (server.io_threads_num == 1 || stopThreadedIOIfNeeded()) {//判断是否有必要开启IO多线程
        return handleClientsWithPendingWrites();
    }

    /* Start threads if needed. */
    if (!server.io_threads_active) startThreadedIO();//开启io多线程

    /* Distribute the clients across N different lists. */
    listIter li;
    listNode *ln;
    listRewind(server.clients_pending_write,&li);//创建一个迭代器li，用于遍历任务队列clients_pending_write
    int item_id = 0;//默认是0，先分配给主线程去做（生产者也可能是消费者），如果设置成1，则先让io线程1去做
    //io_threads_list[0] 主线程
    //io_threads_list[1] io线程
    //io_threads_list[2] io线程   
    //io_threads_list[3] io线程   
    //io_threads_list[4] io线程
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);//取出一个任务
        c->flags &= ~CLIENT_PENDING_WRITE;

        /* Remove clients from the list of pending writes since
         * they are going to be closed ASAP. */
        if (c->flags & CLIENT_CLOSE_ASAP) {//表示该客户端的输出缓冲区超过了服务器允许范围,将在下一次循环进行一个关闭,也不返回任何信息给客户端，删除待读客户端
            listDelNode(server.clients_pending_write, ln);
            continue;
        }

        /* Since all replicas and replication backlog use global replication
         * buffer, to guarantee data accessing thread safe, we must put all
         * replicas client into io_threads_list[0] i.e. main thread handles
         * sending the output buffer of all replicas. */
        if (getClientType(c) == CLIENT_TYPE_SLAVE) {
            listAddNodeTail(io_threads_list[0],c);
            continue;
        }
        //负载均衡：将任务队列中的任务 添加 到不同的线程消费队列中去，每个线程就可以从当前线程的消费队列中取任务就行了
        //这样做的好处是，避免加锁。当前是在主线程中，进行分配任务
        //通过取余操作，将任务均分给不同io线程
        int target_id = item_id % server.io_threads_num;
        listAddNodeTail(io_threads_list[target_id],c);
        item_id++;
    }

    /* Give the start condition to the waiting threads, by setting the
     * start condition atomic var. */
    io_threads_op = IO_THREADS_OP_WRITE;
    for (int j = 1; j < server.io_threads_num; j++) {
        int count = listLength(io_threads_list[j]);
        setIOPendingCount(j, count);//设置io线程启动条件，启动io线程
    }

    /* Also use the main thread to process a slice of clients. */
    listRewind(io_threads_list[0],&li);//让主线程去处理一部分任务（io_threads_list[0]）
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        writeToClient(c,0);
    }
    listEmpty(io_threads_list[0]);

    /* Wait for all the other threads to end their work. */
    while(1) {//剩下的任务io_threads_list[1]，io_threads_list[2].....给io线程去做，等待io线程完成任务
        unsigned long pending = 0;
        for (int j = 1; j < server.io_threads_num; j++)
            pending += getIOPendingCount(j);//等待io线程结束，并返回处理的数量
        if (pending == 0) break;
    }

    io_threads_op = IO_THREADS_OP_IDLE;

    /* Run the list of clients again to install the write handler where
     * needed. */
    listRewind(server.clients_pending_write,&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);

        /* Update the client in the mem usage buckets after we're done processing it in the io-threads */
        updateClientMemUsageBucket(c);

        /* Install the write handler if there are pending writes in some
         * of the clients. */
        if (clientHasPendingReplies(c) &&
                connSetWriteHandler(c->conn, sendReplyToClient) == AE_ERR)
        {
            freeClientAsync(c);
        }
    }
    listEmpty(server.clients_pending_write);

    /* Update processed count on server */
    server.stat_io_writes_processed += processed;

    return processed;
}
```

负载均衡：将任务队列中的任务 添加 到不同的线程消费队列中去，每个线程就可以从当前线程的消费队列中取任务就行了。这样做的好处是，避免加锁。当前是在主线程中，进行分配任务通过取余操作，将任务均分给不同的io线程。

## 4.线程调度

### 4.1开启io线程startThreadedIO

每个io线程都有一把锁，如果主线程把锁还回去了，那么io线程就会启动，不再阻塞

并设置io线程标识为活跃状态io_threads_active=1

```text
void startThreadedIO(void) {
    serverAssert(server.io_threads_active == 0);
    for (int j = 1; j < server.io_threads_num; j++)
        pthread_mutex_unlock(&io_threads_mutex[j]);
    server.io_threads_active = 1;
}
```

### 4.2关闭io线程`stopThreadedIO`

每个io线程都有一把锁，如果主线程拿了，那么io线程就会阻塞等待，也就是停止了IO线程
并设置io线程标识为非活跃状态`io_threads_active=0`

```text
void stopThreadedIO(void) {
    /* We may have still clients with pending reads when this function
     * is called: handle them before stopping the threads. */
    handleClientsWithPendingReadsUsingThreads();
    serverAssert(server.io_threads_active == 1);
    for (int j = 1; j < server.io_threads_num; j++)
        pthread_mutex_lock(&io_threads_mutex[j]);//
    server.io_threads_active = 0;
}
```

原文地址：https://zhuanlan.zhihu.com/p/527166907

作者：linux