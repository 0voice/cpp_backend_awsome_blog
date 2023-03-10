# 【NO.74】linux操作系统是如何管理tcp连接的？

首先，在linux内核的网络模块里维护着一个全局实例，用来存储所有和tcp相关的socket：

```text
// net/ipv4/tcp_ipv4.c
struct inet_hashinfo tcp_hashinfo;
```

其次，在该实例的内部，又根据socket类型的不同，划分成四个hashtable：

```text
// include/net/inet_hashtables.h
struct inet_hashinfo {
        // key是由本地地址、本地端口、远程地址、远程端口组成的四元组
        // value是正在建立连接或已经建立连接的socket
        // 比如，当内核收到一个tcp消息时，它先从消息头里读出地址和端口等信息
        // 然后用该信息到ehash里获取对应的socket
        // 最后把剩余的tcp数据添加到该socket的recv buf中供用户程序读取
        struct inet_ehash_bucket        *ehash;

        // key是本地端口
        // value是使用这个端口的所有socket
        // 比如，当我们用socket监听一个端口时，该socket就在bhash里
        // 同理，由该监听端口建立的连接对应的那些socket也在这里
        // 因为它们也都是使用同样的本地端口
        struct inet_bind_hashbucket     *bhash;

        // key是本地地址和端口组成的二元组
        // value是对应的处于listen状态的socket
        struct inet_listen_hashbucket   *lhash2;

        // key是本地端口
        // value是对应的处于listen状态的socket
        struct inet_listen_hashbucket   listening_hash[INET_LHTABLE_SIZE];
};
```

在系统启动时，这个全局的tcp_hashinfo实例会在下面的方法中被初始化：

```text
// net/ipv4/tcp.c
void __init tcp_init(void)
{
        // 初始化tcp_hashinfo里的四个hashtable等信息
}
```

该tcp_hashinfo实例还会被赋值给下面tcp_prot实例中的对应字段：

```text
// net/ipv4/tcp_ipv4.c
struct proto tcp_prot = {
        // 在struct sock里会通过sk_prot字段引用该tcp_prot实例
        // 也就是说，如果拿到任一个struct sock实例
        // 就可以通过它的sk_prot字段获取tcp_prot实例
        // 进而也就可以获取tcp_hashinfo实例
        .h.hashinfo             = &tcp_hashinfo,
};
EXPORT_SYMBOL(tcp_prot);
```

好，以上就是操作系统管理tcp连接用到的全局的数据结构，接下来我们看一些具体操作。

在tcp编程中一般都分为客户端和服务端，我们先来看下服务端对应的操作。

首先，一个socket想要监听一个端口，必须要先bind一个地址，然后再执行listen操作。

其中bind操作就用到了上面的tcp_hashinfo实例里的bhash这个字段，用来判断该端口是否被占用。

来看下代码：

```text
// net/ipv4/inet_connection_sock.c
int inet_csk_get_port(struct sock *sk, unsigned short snum)
{
        // 该方法的调用栈：
        // SYSCALL_DEFINE3(bind)
        // __sys_bind
        // inet_bind
        // inet_csk_get_port
        
        // 下面的hinfo就是全局实例tcp_hashinfo
        struct inet_hashinfo *hinfo = sk->sk_prot->h.hashinfo;

        // 根据端口算出hash值，然后根据这个值找到bhash中对应的slot
        head = &hinfo->bhash[inet_bhashfn(net, port,
                                          hinfo->bhash_size)];

        // 遍历slot指向的链表，找到port对应的值
        inet_bind_bucket_for_each(tb, &head->chain)
                if (net_eq(ib_net(tb), net) && tb->l3mdev == l3mdev &&
                    tb->port == port)
                        goto tb_found;

        // 如果没找到，说明现在还没有人使用这个端口，就新创建一个
        // 新创建的实例就会放到bhash中，表明这个端口我在使用了
        tb = inet_bind_bucket_create(hinfo->bind_bucket_cachep,
                                     net, head, port, l3mdev);

tb_found:
        // 如果tb的owners字段不为空，则说明有人在使用这个端口
        if (!hlist_empty(&tb->owners)) {
                // 如果该端口被别人占用了，且不能共享使用，就返回错误给用户
                if (inet_csk_bind_conflict(sk, tb, true, true))
                        goto fail_unlock;
        }

        // 省略很多无关代码

        // 在该方法的最后，会调用inet_bind_hash方法
        // 方法内容会在下面描述
        if (!inet_csk(sk)->icsk_bind_hash)
                inet_bind_hash(sk, tb, port);
}
EXPORT_SYMBOL_GPL(inet_csk_get_port);
```

再来看下inet_bind_hash方法：

```text
// net/ipv4/inet_hashtables.c
void inet_bind_hash(struct sock *sk, struct inet_bind_bucket *tb,
                    const unsigned short snum)
{
         // 保存绑定端口
        inet_sk(sk)->inet_num = snum;
        
        // tb是上面方法中获取的或创建的bhash中的一个值
        // 它的owners字段存放的是所有使用该端口的sock
        // 下面语句的意思是，把这个sock也加入到owner里
        // 这样在其他人拿到tb时，就能知道哪些sock在使用这个tb对应的端口了
        sk_add_bind_node(sk, &tb->owners);

        // 将tb地址存放到sock的icsk_bind_hash字段里
        // 这样以后想知道该sock对应的bhash里的值时（比如在移除owners时）
        // 就可以通过下面的字段获取了
        inet_csk(sk)->icsk_bind_hash = tb;
}
```

好，bind方法涉及到tcp_hashinfo的地方，到这里就都已经讲完了，我们再看下listen方法：

```text
// net/ipv4/inet_hashtables.c
int __inet_hash(struct sock *sk, struct sock *osk)
{
        // 该方法的调用栈：
        // SYSCALL_DEFINE2(listen)
        // __sys_listen
        // inet_listen
        // inet_csk_listen_start
        // inet_hash
        // __inet_hash

        // hashinfo就是全局实例tcp_hashinfo
        struct inet_hashinfo *hashinfo = sk->sk_prot->h.hashinfo;

        // 根据本地端口，找到该sock在listening_hash中的slot
        ilb = &hashinfo->listening_hash[inet_sk_listen_hashfn(sk)];

        // 将该sock添加到slot对应的链表中
        if (IS_ENABLED(CONFIG_IPV6) && sk->sk_reuseport &&
                sk->sk_family == AF_INET6)
                hlist_add_tail_rcu(&sk->sk_node, &ilb->head);
        else
                hlist_add_head_rcu(&sk->sk_node, &ilb->head);

        // 以本地端口和地址作为key，将该sock加入到tcp_hashinfo里的lhash2里
        inet_hash2(hashinfo, sk);
}
EXPORT_SYMBOL(__inet_hash);
```

listen方法涉及到tcp_hashinfo的地方就是这一点点。

服务端的相关操作就是这些，我们再来看下客户端。

客户端第一步要做的事就是连接服务器，所以我们看下对应的connect方法：

```text
// net/ipv4/inet_hashtables.c
int __inet_hash_connect(struct inet_timewait_death_row *death_row,
                struct sock *sk, u32 port_offset,
                int (*check_established)(struct inet_timewait_death_row *,
                        struct sock *, __u16, struct inet_timewait_sock **))
{
        // 该方法的调用栈
        // SYSCALL_DEFINE3(connect)
        // __sys_connect
        // inet_stream_connect
        // __inet_stream_connect
        // tcp_v4_connect
        // inet_hash_connect
        // __inet_hash_connect

        // hinfo是全局的tcp_hashinfo实例
        struct inet_hashinfo *hinfo = death_row->hashinfo;

        // 一般来说，connect操作我们都不会主动指定本地端口
        // 而是让操作系统帮我们自由挑选
        // 下面的方法就是用于获取操作系统自由挑选的本地端口的范围
        // 该范围默认是 [32768-60999]
        // 当前范围可由以下命令查看：
        // $ cat /proc/sys/net/ipv4/ip_local_port_range
        inet_get_local_port_range(net, &low, &high);

        // 依次检测范围内的端口，找到第一个可以使用的
        // 第一个要检测的端口
        port = low + offset;
        for (i = 0; i < remaining; i += 2, port += 2) {
                // 找到该端口对应的bhash中的slot
                head = &hinfo->bhash[inet_bhashfn(net, port, hinfo->bhash_size)];

                // 遍历该slot指向的链表，查看是否有人已经在使用该端口
                inet_bind_bucket_for_each(tb, &head->chain) {
                        if (net_eq(ib_net(tb), net) && tb->l3mdev == l3mdev && tb->port == port) {
                                // 如果该端口已经被人使用
                                // 那就检查一下使用者中是否有处于连接状态的socket
                                // 且该socket的tcp四元组和我们的socket的tcp四元组完全一致（tcp四元组唯一确定一个tcp连接）
                                // 如果有，则该端口不可用
                                // 如果没有，则可用
                                if (!check_established(death_row, sk,
                                                       port, &tw))
                                        goto ok;
                                goto next_port;
                        }
                }

                // 如果该端口没人用，我们就在bhash中新创建一个对象，表示我们要用
                tb = inet_bind_bucket_create(hinfo->bind_bucket_cachep,
                                             net, head, port, l3mdev);
                goto ok;
next_port:
        }
ok:
        // 该方法上面有讲过，主要就是将该socket与tb实例联系起来
        // 详情可参考上面
        inet_bind_hash(sk, tb, port);
        if (sk_unhashed(sk)) {
                // 该方法下面会详细看
                inet_ehash_nolisten(sk, (struct sock *)tw);
        }
}
```

再来看下上面提到的inet_ehash_nolisten方法：

```text
// net/ipv4/inet_hashtables.c
bool inet_ehash_insert(struct sock *sk, struct sock *osk)
{
        // hashinfo就是全局的tcp_hashinfo实例
        struct inet_hashinfo *hashinfo = sk->sk_prot->h.hashinfo;

        // 根据本地和远程的地址端口信息算出该socket的hash值
        // 并保存到sk的sk_hash字段里，供后续使用
        sk->sk_hash = sk_ehashfn(sk);

        // 根据该hash值找到ehash中对应的slot
        head = inet_ehash_bucket(hashinfo, sk->sk_hash);

        // 把该socket加入到该slot指向的链表中
        if (ret)
                __sk_nulls_add_node_rcu(sk, list);
}

bool inet_ehash_nolisten(struct sock *sk, struct sock *osk)
{
        bool ok = inet_ehash_insert(sk, osk);
}
```

由上可见，tcp_hashinfo在connect操作里的作用是，先根据bhash和ehash里的信息，为该次connect操作挑选出一个合适的本地端口（该端口的使用也会被记录在bhash里），然后在syn消息发送给服务器之前，将该socket放入到ehash中，这样当内核收到服务器的应答消息时，就可以找到对应的socket了。

connect操作最终会发syn消息给服务器，所以下面我们就来看下服务器在收到这个syn消息时是如何处理的。

在此之前，我们先讲一些铺垫性的内容。

当操作系统收到任意tcp的消息时，都会调用下面的方法，找到该tcp消息所属的socket，然后再根据该socket的当前状态和tcp消息的内容做后续处理：

```text
// net/ipv4/tcp_ipv4.c
int tcp_v4_rcv(struct sk_buff *skb)
{
        struct sock *sk;

        // 该方法会从tcp_hashinfo中的各种hashtable中尝试找到对应的socket
        // th->source是发送方的本地端口
        // th->dest是接收方的本地端口
        sk = __inet_lookup_skb(&tcp_hashinfo, skb, __tcp_hdrlen(th), th->source,
                               th->dest, sdif, &refcounted);
}
```

再看下__inet_lookup_skb方法：

```text
// include/net/inet_hashtables.h
static inline struct sock *__inet_lookup_skb(struct inet_hashinfo *hashinfo,
                                             struct sk_buff *skb,
                                             int doff,
                                             const __be16 sport,
                                             const __be16 dport,
                                             const int sdif,
                                             bool *refcounted)
{
        const struct iphdr *iph = ip_hdr(skb);
        return __inet_lookup(dev_net(skb_dst(skb)->dev), hashinfo, skb,
                             doff, iph->saddr, sport,
                             iph->daddr, dport, inet_iif(skb), sdif,
                             refcounted);
}
```

该方法又调用了__inet_lookup方法：

```text
static inline struct sock *__inet_lookup(struct net *net,
                                         struct inet_hashinfo *hashinfo,
                                         struct sk_buff *skb, int doff,
                                         const __be32 saddr, const __be16 sport,
                                         const __be32 daddr, const __be16 dport,
                                         const int dif, const int sdif,
                                         bool *refcounted)
{
        u16 hnum = ntohs(dport);
        struct sock *sk;

        // 该方法会根据本地和远程的地址端口信息
        // 从tcp_hashinfo的ehash中找对应的socket
        sk = __inet_lookup_established(net, hashinfo, saddr, sport,
                                       daddr, hnum, dif, sdif);

       // 如果在ehash中没有找到对应的socket，则调用下面的方法
       // 从tcp_hashinfo的lhash2中找对应的处于listen状态的socket
       return __inet_lookup_listener(net, hashinfo, skb, doff, saddr,
                                      sport, daddr, hnum, dif, sdif);
}
```

好，铺垫性内容结束。

当服务端收到客户端发来的syn包后，会先通过上述方法，在lhash2中找到对应的listen状态的socket（listen方法把这个socket放入到lhash2中的），然后执行下面的逻辑：

```text
// net/ipv4/tcp_input.c
int tcp_conn_request(struct request_sock_ops *rsk_ops,
                     const struct tcp_request_sock_ops *af_ops,
                     struct sock *sk, struct sk_buff *skb)
{
        // 该方法的调用栈
        // tcp_v4_rcv
        // tcp_v4_do_rcv
        // tcp_rcv_state_process
        // tcp_v4_conn_request

        // 该方法的参数sk就是上面找到的处于listen状态的socket

        // 服务端在收到syn消息后，并不是直接创建一个struct sock
        // 而是创建一个struct request_sock，表示该socket还处于tcp三次握手过程中
        struct request_sock *req;
        req = inet_reqsk_alloc(rsk_ops, sk, !want_cookie);

        if (fastopen_sk) {
        } else {
                if (!want_cookie)
                        // 该方法会根据本地和远程的地址端口信息
                        // 将request_sock放到tcp_hashinfo里的ehash里
                        // 这样后续的消息就可以找到这个request_sock了
                        inet_csk_reqsk_queue_hash_add(sk, req,
                                tcp_timeout_init((struct sock *)req));
        }
}
```

服务端在处理完syn消息后，会发synack给客户端，客户端收到synack消息后，会再次发ack给服务端，同时将客户端的socket状态设置为TCP_ESTABLISHED。

由于客户端处理synack消息的逻辑不涉及到tcp_hashinfo里的内容，所以这里就不详细说了。

再看下服务端在收到ack消息之后的逻辑。

服务端在收到ack消息后，会先通过上面介绍过的__inet_lookup_skb方法，找到刚刚创建的request_sock，然后执行如下逻辑：

```text
// net/ipv4/tcp_ipv4.c
struct sock *tcp_v4_syn_recv_sock(const struct sock *sk, struct sk_buff *skb,
                                  struct request_sock *req,
                                  struct dst_entry *dst,
                                  struct request_sock *req_unhash,
                                  bool *own_req)
{
        // 该方法的调用栈
        // tcp_v4_rcv
        // tcp_check_req
        // tcp_v4_syn_recv_sock

        // 收到客户端的ack消息表示该tcp连接建立成功
        // 下面的方法会根据request_sock创建一个真正的struct sock
        struct sock *newsk;
        newsk = tcp_create_openreq_child(sk, req, skb);

        // 该方法会把新创建的socket放到tcp_hashinfo的bhash中
        // 对应的端口就是监听socket所使用的端口
        // 此时该端口对应的bhash中的value中的owners字段里包含监听socket和这个新创建的socket
        if (__inet_inherit_port(sk, newsk) < 0)
                goto put_and_exit;

        // 在上面创建request_sock时，把它放到tcp_hashinfo的ehash里
        // 到这里request_sock的任务已经完成
        // 所以下面的方法会把request_sock从ehash中移除
        // 而把新创建的socket放到ehash里
        *own_req = inet_ehash_nolisten(newsk, req_to_sk(req_unhash));
}
```

到现在一个完整的tcp连接已经建立好了，我们再重新梳理下整个思路。

首先，服务端的socket先执行了bind操作，把它自己放到了tcp_hashinfo的bhash中，然后执行了listen操作，把它自己放到了tcp_hashinfo的lhash2中。

然后，客户端执行connect方法，把对应的socket放到了bhash和ehash中，然后发了syn消息给服务端。

服务端收到syn后，先从lhash2中找到对应的listen状态的socket，然后又根据该socket和syn消息创建了request_sock，并放入ehash中，最后发synack给客户端。

客户端收到synack后，先从ehash中找到对应的socket，然后把其状态设置为TCP_ESTABLISHED，最后又返回ack给服务端。

服务端收到ack后，会先从ehash中找到之前创建的request_sock，然后根据该request_sock，创建真正的sock，最后将request_sock从ehash中移除，将新创建的sock放到bhash和ehash中。

至此，一个tcp连接就建立成功。

再之后，就是tcp连接的数据传输过程了，当操作系统收到对方发来的数据时，先根据tcp消息头里的地址端口等信息，从ehash中找到对应的socket，然后将该数据添加到这个socket的接受缓冲区里，这样用户就可以通过read等方法获取这些数据了。

这就是在tcp连接建立成功之后，tcp内的逻辑对tcp_hashinfo的使用。

下面我们再来看下在tcp的关闭流程中，tcp_hashinfo是如何被使用的。

假设客户端先调用了close方法，主动关闭了连接，看下对应代码：

```text
// net/ipv4/tcp.c
void tcp_close(struct sock *sk, long timeout)
{
        // 该方法的调用栈
        // SYSCALL_DEFINE1(close)
        // __close_fd
        // filp_close
        // fput
        // fput_many
        // ____fput
        // __fput
        // sock_close
        // __sock_release
        // inet_release
        // tcp_close

        // 下面的方法会将socket状态设置为TCP_FIN_WAIT1
        } else if (tcp_close_state(sk)) {
                // 发fin消息给对方
                tcp_send_fin(sk);
        }
}
```

服务端在收到fin包的处理逻辑为：

```text
// net/ipv4/tcp_input.c
void tcp_fin(struct sock *sk)
{
        // 该方法的调用栈
        // tcp_v4_rcv
        // tcp_v4_do_rcv
        // tcp_rcv_established
        // tcp_data_queue
        // tcp_fin

        switch (sk->sk_state) {
        case TCP_ESTABLISHED:
                tcp_set_state(sk, TCP_CLOSE_WAIT);
}
```

该方法将服务端socket的状态设置为TCP_CLOSE_WAIT，然后返回ack给客户端。

客户端收到ack后的处理逻辑：

```text
// net/ipv4/tcp_input.c
int tcp_rcv_state_process(struct sock *sk, struct sk_buff *skb)
{
        // 该方法的调用栈
        // tcp_v4_rcv
        // tcp_v4_do_rcv
        // tcp_rcv_state_process

        switch (sk->sk_state) {
        case TCP_FIN_WAIT1: {
                tcp_set_state(sk, TCP_FIN_WAIT2);
                break;
        }       
}
```

该方法收到ack后，将客户端对应的socket的状态设置为TCP_FIN_WAIT2。

此时，假设服务器的应用层也调用了socket的close方法，该方法会执行以下逻辑：

```text
// net/ipv4/tcp.c
void tcp_close(struct sock *sk, long timeout)
{
        // 该方法的调用栈
        // SYSCALL_DEFINE1(close)
        // __close_fd
        // filp_close
        // fput
        // fput_many
        // ____fput
        // __fput
        // sock_close
        // __sock_release
        // inet_release
        // tcp_close

        // 下面的方法会将socket状态设置为TCP_LAST_ACK
        } else if (tcp_close_state(sk)) {
                // 发fin消息给对方
                tcp_send_fin(sk);
        }
}
```

客户端收到fin消息的处理逻辑：

```text
// net/ipv4/tcp_input.c
void tcp_fin(struct sock *sk)
{
        // 该方法调用栈
        // tcp_v4_rcv
        // tcp_v4_do_rcv
        // tcp_rcv_state_process
        // tcp_data_queue
        // tcp_fin

        switch (sk->sk_state) {
        case TCP_FIN_WAIT2:
                // 发送ack给对方
                tcp_send_ack(sk);
                // 进入time wait逻辑处理
                tcp_time_wait(sk, TCP_TIME_WAIT, 0);
        }
}
```

继续看下tcp_time_wait方法：

```text
// net/ipv4/tcp_minisocks.c
void tcp_time_wait(struct sock *sk, int state, int timeo)
{
        // 类似于三次握手时服务端创建了request_sock
        // 这里也根据当前sock创建了一个inet_timewait_sock
        // 对应于处于time wait状态时的socket
        struct inet_timewait_sock *tw;
        tw = inet_twsk_alloc(sk, tcp_death_row, state);

        if (tw) {
                // 从sock拷贝各种必要数据到inet_timewait_sock

                // 进行time wait定时，超时后会调用inet_twsk_kill方法
                // 将inet_timewait_sock从ehash和bhash中移除
                inet_twsk_schedule(tw, timeo);

                // 该方法会将sock从ehash中移除
                // 将inet_timewait_sock加入到ehash和bhash中
                inet_twsk_hashdance(tw, sk, &tcp_hashinfo);     
        }

        // 该方法会将sock从bhash中移除，并将其销毁
        tcp_done(sk);
}
```

好，客户端的逻辑就全部完成了，我们再看下服务端逻辑。

当服务器处于TCP_LAST_ACK状态时，收到客户端的ack消息，进行下面的处理：

```text
// net/ipv4/tcp_input.c
int tcp_rcv_state_process(struct sock *sk, struct sk_buff *skb)
{
        // 该方法调用栈
        // tcp_v4_rcv
        // tcp_v4_do_rcv
        // tcp_rcv_state_process

        switch (sk->sk_state) {
        case TCP_LAST_ACK:
                if (tp->snd_una == tp->write_seq) {
                        // 该方法及其底层方法会将该sock从ebash和bhash中移除
                        tcp_done(sk);
                }       
        }
}
```

到这里，一个tcp连接就完全关闭了，并且前后端的socket都已经从tcp_hashinfo的ehash和bhash中移除。

现在系统又回到tcp连接之前的状态，即只有一个服务端的socket处于listen状态，该socket同时被存放于tcp_hashinfo的bhash、lhash2及listening_hash里。

我们看下当该listen状态的socket关闭的时候，对应的有关tcp_hashinfo的处理：

```text
// net/ipv4/tcp.c
void tcp_set_state(struct sock *sk, int state)
{
        // 该方法的调用栈
        // SYSCALL_DEFINE1(close)
        // __close_fd
        // filp_close
        // fput
        // fput_many
        // ____fput
        // __fput
        // sock_close
        // __sock_release
        // inet_release
        // tcp_close
        // tcp_set_state

        switch (state) {
        case TCP_CLOSE:
                // 下面的方法会将socket从lhash2及listening_hash里移除
                sk->sk_prot->unhash(sk);

                // 下面方法会将socket从bhash中移除
                if (inet_csk(sk)->icsk_bind_hash &&
                    !(sk->sk_userlocks & SOCK_BINDPORT_LOCK))
                        inet_put_port(sk);
        }
}
```

至此，所有socket都已关闭，且tcp_hashinfo又回到了最原始的空状态。

上文用了大量的篇幅讲述在tcp的各种操作中，tcp_hashinfo是如何被使用的。其实回过头来看一下，tcp_hashinfo在这其中的作用还是非常简单的，其主要目的就是辅助操作系统在各种情况下找到对应的socket。

比如，在syn消息来时，要找到对应的listen状态的socket，用了tcp_hashinfo中的lhash2。

比如，在syn消息之后的所有后续消息来时，要找到其对应的消息处理socket，用了tcp_hashinfo中的ehash。

比如，在绑定端口或挑选端口时，要用到bhash来查询端口是否被占用。

好，就这么多吧，文章到此就结束了。

原文地址：https://zhuanlan.zhihu.com/p/571947344

作者：linux