# 【NO.41】深入源码理解TCP建立连接过程（3次握手）

从TCP源码上深入了解三次握手过程

## 1.应用程序的基本框架

```text
//server
int main()
{
    int fd = socket(AF_INET,SOCK_STREAM,0);
    bind(fd，...);
    listen(fd,256);
    accept(fd,...);
 
}
 
//client
int main()
{
    fd = connect(AF_INET,SOCK_STREAM,0);
    connect(fd,...);
}
```

**三次握手概括**

![img](https://pic2.zhimg.com/80/v2-ab82f66854555f94d39a649815e181d1_720w.webp)

分析开始

## 2.客户端connect

```text
int tcp_v4_connect(struct sock *sk, struct sockaddr *uaddr, int addr_len)
{
    //设置socket状态TCP_SYN_SENT
	tcp_set_state(sk, TCP_SYN_SENT);
 
	//动态选择一个端口
	err = inet_hash_connect(tcp_death_row, sk);
    
	//根据sk中的信息，构建一个SYNC的报文，并将它发送出去
	err = tcp_connect(sk);
}
 
 
int tcp_connect(struct sock *sk)
{
	//申请skb
	buff = sk_stream_alloc_skb(sk, 0, sk->sk_allocation, true);
 
 
	//添加到发送队列sk_write_queue
	tcp_connect_queue_skb(sk, buff);
	tcp_ecn_send_syn(sk, buff);
	tcp_rbtree_insert(&sk->tcp_rtx_queue, buff);
 
	//发送syn  tcp_transmit_skb
	err = tp->fastopen_req ? tcp_send_syn_data(sk, buff) :
	      tcp_transmit_skb(sk, buff, 1, sk->sk_allocation);
 
	//重启定时器
	/* Timer for repeating the SYN until an answer. */
	inet_csk_reset_xmit_timer(sk, ICSK_TIME_RETRANS,
				  inet_csk(sk)->icsk_rto, TCP_RTO_MAX);
	return 0;
}
```

connect 作用把本地socket状态设置成TCP_SYN_SENT；

选择一个可用的端口，发出SYN握手请求并重置定时器；

## 3.服务端响应SYN

```text
int tcp_v4_do_rcv(struct sock *sk, struct sk_buff *skb)
{
	//服务端收到SYN或者第三步ACK都会走到这里
	if (sk->sk_state == TCP_LISTEN) {
		struct sock *nsk = tcp_v4_cookie_check(sk, skb);//查看半连接队列
 
 
	} else
		sock_rps_save_rxhash(sk, skb);
    //不同的状态处理
	if (tcp_rcv_state_process(sk, skb)) {
		rsk = sk;
		goto reset;
	}
}
```



```text
//服务端处理syn连接请求
int tcp_conn_request(struct request_sock_ops *rsk_ops,
		     const struct tcp_request_sock_ops *af_ops,
		     struct sock *sk, struct sk_buff *skb)
{
 
	//查看半连接队列是否满了，则直接丢弃
	if ((net->ipv4.sysctl_tcp_syncookies == 2 ||
	     inet_csk_reqsk_queue_is_full(sk)) && !isn) {
		want_cookie = tcp_syn_flood_action(sk, rsk_ops->slab_name);
		if (!want_cookie)
			goto drop;
	}
	//查看全连接队列，如果满了则直接丢弃
	if (sk_acceptq_is_full(sk)) {
		NET_INC_STATS(sock_net(sk), LINUX_MIB_LISTENOVERFLOWS);
		goto drop;
	}
	//分配request_sock内核对象
	req = inet_reqsk_alloc(rsk_ops, sk, !want_cookie);
 
 
	if (fastopen_sk) {
		af_ops->send_synack(fastopen_sk, dst, &fl, req,
				    &foc, TCP_SYNACK_FASTOPEN);
		/* Add the child socket directly into the accept queue */
	} else {
		tcp_rsk(req)->tfo_listener = false;
		if (!want_cookie)//添加到半连接队列，并开启定时器
			inet_csk_reqsk_queue_hash_add(sk, req,
				tcp_timeout_init((struct sock *)req));
		//构造synack包
		af_ops->send_synack(sk, dst, &fl, req, &foc,
				    !want_cookie ? TCP_SYNACK_NORMAL :
						   TCP_SYNACK_COOKIE);
 
	}
```

.send_synack就是tcp_v4_send_synack

```text
static int tcp_v4_send_synack(const struct sock *sk, struct dst_entry *dst,
			      struct flowi *fl,
			      struct request_sock *req,
			      struct tcp_fastopen_cookie *foc,
			      enum tcp_synack_type synack_type)
{
 
	if (!dst && (dst = inet_csk_route_req(sk, &fl4, req)) == NULL)
		return -1;
	//构造syn+ack包
	skb = tcp_make_synack(sk, dst, req, foc, synack_type);
 
	if (skb) {
		__tcp_v4_send_check(skb, ireq->ir_loc_addr, ireq->ir_rmt_addr);
 
		rcu_read_lock();
		//将synack包发送出去
		err = ip_build_and_send_pkt(skb, sk, ireq->ir_loc_addr,
					    ireq->ir_rmt_addr,
					    rcu_dereference(ireq->ireq_opt));
		rcu_read_unlock();
		err = net_xmit_eval(err);
	}
 
	return err;
}
```

服务端响应SYN

1. 检查半连接与全连接队列是否满，满握手直接丢弃
2. 申请request_sock,并添加到半连接队列中
3. 构造syn+ack包，通过ip_build_and_send_pkt发送
4. 重启定时器tcp_timeout_init



## 4.客户端响应SYN+ACK

客户端收到服务端发来的syn+ack包的时候，进入tcp_rcv_state_process

```text
int tcp_rcv_state_process(struct sock *sk, struct sk_buff *skb)
{
	switch (sk->sk_state) {
	case TCP_CLOSE:
		goto discard;
	//第一次握手 服务器收到
	case TCP_LISTEN:
 
		goto discard;
	//客户端第二次握手处理
	case TCP_SYN_SENT:
		//synack包
		queued = tcp_rcv_synsent_state_process(sk, skb, th);
		return 0;
	}
}
```



```text
static int tcp_rcv_synsent_state_process(struct sock *sk, struct sk_buff *skb,
					 const struct tcphdr *th)
{
        //step1:
		tcp_ack(sk, skb, FLAG_SLOWPATH);
 
		//step2:tcp 建立完成
		tcp_finish_connect(sk, skb);
 
 
		if (sk->sk_write_pending ||
		    icsk->icsk_accept_queue.rskq_defer_accept ||
		    inet_csk_in_pingpong_mode(sk)) {
 
			return 0;
		} else {
            //step3:发送确认
			tcp_send_ack(sk);
		}
}
```

step1:

```text
/* This routine deals with incoming acks, but not outgoing ones. */
static int tcp_ack(struct sock *sk, const struct sk_buff *skb, int flag)
{
    //删除发送队列
   flag |= tcp_clean_rtx_queue(sk, prior_fack, prior_snd_una, &sack_state);
 
    //重置定时器
	if (flag & FLAG_SET_XMIT_TIMER)
		tcp_set_xmit_timer(sk);
}
```

step2

```text
void tcp_finish_connect(struct sock *sk, struct sk_buff *skb)
{
	//修改socket状态
	tcp_set_state(sk, TCP_ESTABLISHED);
	icsk->icsk_ack.lrcvtime = tcp_jiffies32;
 
	//拥塞控制
	tcp_init_transfer(sk, BPF_SOCK_OPS_ACTIVE_ESTABLISHED_CB);
	tp->lsndtime = tcp_jiffies32;
 
	//打开保活计时器
	if (sock_flag(sk, SOCK_KEEPOPEN))
		inet_csk_reset_keepalive_timer(sk, keepalive_time_when(tp));
 
}
```

step3：

```text
void __tcp_send_ack(struct sock *sk, u32 rcv_nxt)
{
	if (sk->sk_state == TCP_CLOSE)
		return;
 
	//申请和构造ack包
	buff = alloc_skb(MAX_TCP_HEADER,
			 sk_gfp_mask(sk, GFP_ATOMIC | __GFP_NOWARN));
 
	//发送出去
	__tcp_transmit_skb(sk, buff, 0, (__force gfp_t)0, rcv_nxt);
}
```

总结

1. 客户端响应synack，清除重传定时器
2. 设置当前状态为ESTABLISHED
3. 开启拥塞控制，保活机制
4. 发送ack包

## 5.服务器端响应ACK

```text
int tcp_v4_do_rcv(struct sock *sk, struct sk_buff *skb)
{
 
    //服务端收到SYN或者第三步ACK都会走到这里
	if (sk->sk_state == TCP_LISTEN) {
		struct sock *nsk = tcp_v4_cookie_check(sk, skb);//step1:查看半连接队列,新创建线程
 
 
    //step2:
	if (tcp_rcv_state_process(sk, skb)) 
}
```

step1：创建子socket；添加全连接队列

```text
//tcp_v4_cookie_check->cookie_v4_check->tcp_get_cookie_sock
struct sock *tcp_get_cookie_sock(struct sock *sk, struct sk_buff *skb,
				 struct request_sock *req,
				 struct dst_entry *dst, u32 tsoff)
{
	struct inet_connection_sock *icsk = inet_csk(sk);
	struct sock *child;
	bool own_req;
 
	//创建子sock 回调函数在下面tcp_v4_syn_recv_sock
	child = icsk->icsk_af_ops->syn_recv_sock(sk, skb, req, dst,
						 NULL, &own_req);
	if (child) {
		//添加全连接队列
		if (inet_csk_reqsk_queue_add(sk, req, child))
			return child;
		bh_unlock_sock(child);
		sock_put(child);
	}
	return NULL;
}
 
 
//添加到全连接队列
struct sock *inet_csk_reqsk_queue_add(struct sock *sk,
				      struct request_sock *req,
				      struct sock *child)
{
	struct request_sock_queue *queue = &inet_csk(sk)->icsk_accept_queue;
		req->sk = child;
		req->dl_next = NULL;
		if (queue->rskq_accept_head == NULL)
			WRITE_ONCE(queue->rskq_accept_head, req);
}
 
 
 
const struct inet_connection_sock_af_ops ipv4_specific = {
	.syn_recv_sock	   = tcp_v4_syn_recv_sock,
}
//子socket的创建过程
struct sock *tcp_v4_syn_recv_sock(const struct sock *sk, struct sk_buff *skb,
				  struct request_sock *req,
				  struct dst_entry *dst,
				  struct request_sock *req_unhash,
				  bool *own_req)
{
	struct ip_options_rcu *inet_opt;
 
	//判断队列是否满了
	if (sk_acceptq_is_full(sk))
		goto exit_overflow;
		
	//创建socket
	newsk = tcp_create_openreq_child(sk, req, skb);
	if (!newsk)
		goto exit_nonewsk;
}
```

step2:设置连接状态TCP_ESTABLISHED

```text
int tcp_rcv_state_process(struct sock *sk, struct sk_buff *skb)
{
 
	switch (sk->sk_state) {
	case TCP_CLOSE:
		goto discard;
	//第一次握手
	case TCP_LISTEN:
	//客户端第二次握手处理
	case TCP_SYN_SENT:
	switch (sk->sk_state) {
	//服务器第三次握手
	case TCP_SYN_RECV:
 
		//改变连接状态
		tcp_set_state(sk, TCP_ESTABLISHED);
}
```

总结

1. 服务端将状态设置为ESTABLISHED;
2. 创建新sock加入全连接队列中

## 6.最后accept过程

```text
struct sock *inet_csk_accept(struct sock *sk, int flags, int *err, bool kern)
{
	struct inet_connection_sock *icsk = inet_csk(sk);
	//全连接队列获取元素
	struct request_sock_queue *queue = &icsk->icsk_accept_queue;
 
	//取出一个给用户使用
	req = reqsk_queue_remove(queue, sk);
	newsk = req->sk;
}
```

从全连接队列中取出一个给用户使用

原文地址：https://zhuanlan.zhihu.com/p/575657861    

作者： linux