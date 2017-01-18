---
layout: post
title:  socket建立连接 sys_connect
categories: [网络]
tags: [Linux, TCP/IP]
description: ""
---


&emsp;&emsp;&emsp;&emsp;connect()函数调用过程

```
connect(fd, servaddr, addrlen);
-> SYSCALL_DEFINE3()
-> sock->ops->connect() == inet_stream_connect (sock->ops即inet_stream_ops)
-> tcp_v4_connect()
    -> inet_hash_connect()
        -> __inet_hash_connect()
            -> check_established()
                -> __inet_check_established()
```

```
SYSCALL_DEFINE3(connect, int, fd, struct sockaddr __user *, uservaddr,
        int, addrlen)
{
    struct socket *sock;
    struct sockaddr_storage address;
    int err, fput_needed;
    /* 找到文件描述符对应的BSD socket结构，在前面的socket调用中建立*/
    sock = sockfd_lookup_light(fd, &err, &fput_needed);
    if (!sock)
        goto out;
    /* copy对端的地址到内核空间 */
    err = move_addr_to_kernel(uservaddr, addrlen, (struct sockaddr *)&address);
    if (err < 0)
        goto out_put;

    err =
        security_socket_connect(sock, (struct sockaddr *)&address, addrlen);
    if (err)
        goto out_put;
    /* 调用该BSD socket对应的connect调用 */
    err = sock->ops->connect(sock, (struct sockaddr *)&address, addrlen,
                 sock->file->f_flags);
out_put:
    /* 释放文件的引用 */
    fput_light(sock->file, fput_needed);
out:
    return err;
}
```

```
/*
 *    Connect to a remote host. There is regrettably still a little
 *    TCP 'magic' in here.
 */
int inet_stream_connect(struct socket *sock, struct sockaddr *uaddr,
            int addr_len, int flags)
{
    struct sock *sk = sock->sk;
    int err;
    long timeo;

    lock_sock(sk);

    if (uaddr->sa_family == AF_UNSPEC) {
        err = sk->sk_prot->disconnect(sk, flags);
        sock->state = err ? SS_DISCONNECTING : SS_UNCONNECTED;
        goto out;
    }

    switch (sock->state) {
    default:
        err = -EINVAL;
        goto out;
    case SS_CONNECTED:     /* 该BSD socket已连接*/
        err = -EISCONN;
        goto out;
    case SS_CONNECTING:   /* 该BSD socket正在连接*/
        err = -EALREADY;
        /* Fall out of switch with err, set for this state */
        break;
    case SS_UNCONNECTED:
        err = -EISCONN;
        if (sk->sk_state != TCP_CLOSE)
            goto out;
            /* INET SOCKET 调用协议特有connect操作符 */
        err = sk->sk_prot->connect(sk, uaddr, addr_len);
        if (err < 0)
            goto out;
            /* 上面的调用完成后，连接并没有完成，*/
        sock->state = SS_CONNECTING;

        /* Just entered SS_CONNECTING state; the only
         * difference is that return value in non-blocking
         * case is EINPROGRESS, rather than EALREADY.
         */
        err = -EINPROGRESS;
        break;
    }
    /* 获取连接超时时间*/
    timeo = sock_sndtimeo(sk, flags & O_NONBLOCK);

    if ((1 << sk->sk_state) & (TCPF_SYN_SENT | TCPF_SYN_RECV)) {
        /* Error code is set above 进入定时等待 */
        if (!timeo || !inet_wait_for_connect(sk, timeo))
            goto out;

        err = sock_intr_errno(timeo);
        if (signal_pending(current))
            goto out;
    }

    /* Connection was closed by RST, timeout, ICMP error
     * or another process disconnected us.
     */
    if (sk->sk_state == TCP_CLOSE)
        goto sock_error;

    /* sk->sk_err may be not zero now, if RECVERR was ordered by user
     * and error was received after socket entered established state.
     * Hence, it is handled normally after connect() return successfully.
     */

    sock->state = SS_CONNECTED;
    err = 0;
out:
    release_sock(sk);
    return err;

sock_error:
    err = sock_error(sk) ? : -ECONNABORTED;
    sock->state = SS_UNCONNECTED;
    if (sk->sk_prot->disconnect(sk, flags))
        sock->state = SS_DISCONNECTING;
    goto out;
}
```

```
/* This will initiate an outgoing connection. */
int tcp_v4_connect(struct sock *sk, struct sockaddr *uaddr, int addr_len)
{
    struct inet_sock *inet = inet_sk(sk);
    struct tcp_sock *tp = tcp_sk(sk);
    struct sockaddr_in *usin = (struct sockaddr_in *)uaddr;
    struct rtable *rt;
    __be32 daddr, nexthop;
    int tmp;
    int err;

    if (addr_len < sizeof(struct sockaddr_in))
        return -EINVAL;

    if (usin->sin_family != AF_INET)
        return -EAFNOSUPPORT;
    /* 开始准备路由 */
    nexthop = daddr = usin->sin_addr.s_addr;
    if (inet->opt && inet->opt->srr) {
        if (!daddr)
            return -EINVAL;
        nexthop = inet->opt->faddr;
    }
    /* 调用路由模块获取出口信息，这里不深入 */
    tmp = ip_route_connect(&rt, nexthop, inet->saddr,
                   RT_CONN_FLAGS(sk), sk->sk_bound_dev_if,
                   IPPROTO_TCP,
                   inet->sport, usin->sin_port, sk, 1);
    if (tmp < 0) {
        if (tmp == -ENETUNREACH)
            IP_INC_STATS_BH(sock_net(sk), IPSTATS_MIB_OUTNOROUTES);
        return tmp;
    }
    /* 如果获取的路由是广播或多播域， 返回网络不可达，tcp不支持多播与广播 */
    if (rt->rt_flags & (RTCF_MULTICAST | RTCF_BROADCAST)) {
        ip_rt_put(rt);
        return -ENETUNREACH;
    }

    if (!inet->opt || !inet->opt->srr)
        daddr = rt->rt_dst;

    if (!inet->saddr)
        inet->saddr = rt->rt_src;
    inet->rcv_saddr = inet->saddr;

    if (tp->rx_opt.ts_recent_stamp && inet->daddr != daddr) {
        /* Reset inherited state */
        tp->rx_opt.ts_recent      = 0;
        tp->rx_opt.ts_recent_stamp = 0;
        tp->write_seq         = 0;
    }

    if (tcp_death_row.sysctl_tw_recycle &&
        !tp->rx_opt.ts_recent_stamp && rt->rt_dst == daddr) {
        struct inet_peer *peer = rt_get_peer(rt);
        /*
         * VJ's idea. We save last timestamp seen from
         * the destination in peer table, when entering state
         * TIME-WAIT * and initialize rx_opt.ts_recent from it,
         * when trying new connection.
         */
        if (peer != NULL &&
            peer->tcp_ts_stamp + TCP_PAWS_MSL >= get_seconds()) {
            tp->rx_opt.ts_recent_stamp = peer->tcp_ts_stamp;
            tp->rx_opt.ts_recent = peer->tcp_ts;
        }
    }

    inet->dport = usin->sin_port;
    inet->daddr = daddr;

    inet_csk(sk)->icsk_ext_hdr_len = 0;
    if (inet->opt)
        inet_csk(sk)->icsk_ext_hdr_len = inet->opt->optlen;
    /* mss_clamp */
    tp->rx_opt.mss_clamp = 536;

    /* Socket identity is still unknown (sport may be zero).
     * However we set state to SYN-SENT and not releasing socket
     * lock select source port, enter ourselves into the hash tables and
     * complete initialization after this.
     */
    tcp_set_state(sk, TCP_SYN_SENT);
    err = inet_hash_connect(&tcp_death_row, sk);
    if (err)
        goto failure;

    err = ip_route_newports(&rt, IPPROTO_TCP,
                inet->sport, inet->dport, sk);
    if (err)
        goto failure;

    /* OK, now commit destination to socket.  */
    sk->sk_gso_type = SKB_GSO_TCPV4;
    sk_setup_caps(sk, &rt->u.dst);

    if (!tp->write_seq)
        tp->write_seq = secure_tcp_sequence_number(inet->saddr,
                               inet->daddr,
                               inet->sport,
                               usin->sin_port);
    /* id是IP包头的id域 */
    inet->id = tp->write_seq ^ jiffies;

    err = tcp_connect(sk);
    rt = NULL;
    if (err)
        goto failure;

    return 0;

failure:
    /*
     * This unhashes the socket and releases the local port,
     * if necessary.
     */
    tcp_set_state(sk, TCP_CLOSE);
    ip_rt_put(rt);
    sk->sk_route_caps = 0;
    inet->dport = 0;
    return err;
}
```
&emsp;&emsp;&emsp;&emsp;当snum==0时，表明此时源端口没有指定，此时会随机选择一个空闲端口作为此次连接
的源端口。low和high分别表示可用端口的下限和上限，remaining表示可用端口的数，注意这里的可用只是指端口
可以用作源端口，其中部分端口可能已经作为其它socket的端口号在使用了，所以要循环1~remaining，
直到查找到空闲的源端口。

&emsp;&emsp;&emsp;&emsp;下面来看下对每个端口的检查，即//choose a valid port部分的代码。
这里要先了解下tcp的内核表组成，udp的表内核表udptable只是一张hash表，tcp的表则稍复杂，
它的名字是tcp\_hashinfo，在tcp_init()中被初始化，这个数据结构定义如下(省略了不相关的数据)：

```
struct inet_hashinfo {
    struct inet_ehash_bucket *ehash;
    ……
    struct inet_bind_hashbucket *bhash;
    ……
    struct inet_listen_hashbucket  listening_hash[INET_LHTABLE_SIZE]
                    ____cacheline_aligned_in_smp;
};
```

&emsp;&emsp;&emsp;&emsp;从定义可以看出，tcp表又分成了三张表ehash, bhash, listening\_hash，其中ehash, 
listening\_hash对应于socket处在TCP的ESTABLISHED, LISTEN状态，bhash对应于socket已绑定了本地地址。
三者间并不互斥，如一个socket可同时在bhash和ehash中，由于TIME\_WAIT是一个比较特殊的状态，所以ehash又分成了
chain和twchain，为TIME_WAIT的socket单独形成一张表。

&emsp;&emsp;&emsp;&emsp;回到刚才的代码，现在还只是建立socket连接，使用的就应该是tcp表中的bhash。
首先取得内核tcp表的bind表 – bhash，查看是否已有socket占用：

&emsp;&emsp;&emsp;&emsp;如果没有，则调用inet\_bind_bucket\_create()创建一个bind表项tb，
并插入到bind表中，跳转至goto ok代码段； 如果有，则跳转至goto ok代码段。

&emsp;&emsp;&emsp;&emsp;进入ok代码段表明已找到合适的bind表项(无论是创建的还是查找到的)，调用inet\_bind\_hash()赋值源端口inet_num。

&emsp;&emsp;&emsp;&emsp;inet\_hash\_connect()函数只是对\_\_inet\_hash\_connect()函数进行了简单的封装
。在\_\_inet\_hash\_connect()中如果已绑定了端口号，并且是和其他传输控制块共享绑定的端口号，
则会调用check_established参数指向的函数来检查这个绑定的端口号是否可用，代码如下所示：

```
int __inet_hash_connect(struct inet_timewait_death_row *death_row,
        struct sock *sk, u32 port_offset,
        int (*check_established)(struct inet_timewait_death_row *,
        struct sock *, __u16, struct inet_timewait_sock **),
        void (*hash)(struct sock *sk))
{
    struct inet_hashinfo *hinfo = death_row->hashinfo;
    const unsigned short snum = inet_sk(sk)->num;
    struct inet_bind_hashbucket *head;
    struct inet_bind_bucket *tb;
    int ret;
    struct net *net = sock_net(sk);

    if (!snum) {
        int i, remaining, low, high, port;
        static u32 hint;
        u32 offset = hint + port_offset;
        struct hlist_node *node;
        struct inet_timewait_sock *tw = NULL;

        inet_get_local_port_range(&low, &high);
        remaining = (high - low) + 1;

        local_bh_enable();
        for (i = 1; i <= remaining; i++) {
            port = low + (i + offset) % remaining;
            if (inet_is_reserved_local_port(port)
                continue;
            head = &hinfo->bhash[inet_bhashfn(net, port, hinfo->bhash_size)];
            spin_lock(&head->lock);
            inet_bind_bucket_for_each(tb, node, &head->chain) {
                if (net_eq(ib_net(tb), net) && tb->port == port) {
                    if (tb->fastreuse >= 0)
                        goto next_port;
                    WARN_ON(hlist_empty(&tb->owners));
                    if (!check_established(death_row, sk, port, &tw))
                        goto ok;
                    goto next_port;
                }
            }

            tb = inet_bind_bucket_create(hinfo->bind_bucket_cachep, net, head, port);
            if (!tb) {
                spin_unlock(&head->lock);
                break;
            }
            tb->fastreuse = -1;
            tb->fastreuseport = -1;
            goto ok;
        next_port:
            spin_unlock(&head->lock);
        }
        local_bh_enable();

        return -EADDRNOTAVAIL;

ok:
        hint += i;

        inet_bind_hash(sk, tb, port);
        if (sk_unhashed(sk)) {
            inet_sk(sk)->sport = htons(port);
            hash(sk);
        }
        spin_unlock(&head->lock);
        if (tw) {
            inet_twsk_deschedule(tw, death_row);
            inet_twsk_put(tw);
        }

        ret = 0;
        goto out;
    }

    head = &hinfo->bhash[inet_bhashfn(net, snum, hinfo->bhash_size)];
    tb  = inet_csk(sk)->icsk_bind_hash;
    spin_lock_bh(&head->lock);
    if (sk_head(&tb->owners) == sk && !sk->sk_bind_node.next) {
        hash(sk);
        spin_unlock_bh(&head->lock);
        return 0;
    } else {
        spin_unlock(&head->lock);
        /* No definite answer... Walk to established hash table */
        ret = check_established(death_row, sk, snum, NULL);
out:
        local_bh_enable();
        return ret;
    }
}
```

&emsp;&emsp;&emsp;&emsp;(sk\_head(&tb->owners) == sk && !sk->sk\_bind_node.next)这个判断条件就是用来
判断是不是只有当前传输控制块在使用已绑定的端口，条件为false时，会执行else分支，检查是否可用。
这么看来，调用bind()成功并不意味着这个端口就真的可以用。

&emsp;&emsp;&emsp;&emsp;check\_established参数对应的函数是\_\_inet\_check\_established()，
在inet\_hash\_connect()中可以看到。在上面的代码中我们还注意到调用check_established()时第三个参数为NULL，
这在后面的分析中会用到。

&emsp;&emsp;&emsp;&emsp;\_\_inet\_check\_established()函数中，会分别在TIME\_WAIT传输控制块和除TIME\_WIAT、
LISTEN状态外的传输控制块中查找是已绑定的端口是否已经使用，代码片段如下所示：

```
/* called with local bh disabled */
static int __inet_check_established(struct inet_timewait_death_row *death_row,
            struct sock *sk, __u16 lport,
            struct inet_timewait_sock **twp)
{
    struct inet_hashinfo *hinfo = death_row->hashinfo;
    struct inet_sock *inet = inet_sk(sk);
    __be32 daddr = inet->rcv_saddr;
    __be32 saddr = inet->daddr;
    int dif = sk->sk_bound_dev_if;
    INET_ADDR_COOKIE(acookie, saddr, daddr)
    const __portpair ports = INET_COMBINED_PORTS(inet->dport, lport);
    struct net *net = sock_net(sk);
    unsigned int hash = inet_ehashfn(net, daddr, lport, saddr, inet->dport);
    struct inet_ehash_bucket *head = inet_ehash_bucket(hinfo, hash);
    spinlock_t *lock = inet_ehash_lockp(hinfo, hash);
    struct sock *sk2;
    const struct hlist_nulls_node *node;
    struct inet_timewait_sock *tw;

    spin_lock(lock);

    /* Check TIME-WAIT sockets first. */
    sk_nulls_for_each(sk2, node, &head->twchain) {
        tw = inet_twsk(sk2);


        if (INET_TW_MATCH(sk2, net, hash, acookie,
                saddr, daddr, ports, dif)) {
            if (twsk_unique(sk, sk2, twp))
                goto unique;
            else
                goto not_unique;
        }
    }
    tw = NULL;

    /* And established part... */
    sk_nulls_for_each(sk2, node, &head->chain) {
        if (INET_MATCH(sk2, net, hash, acookie,
                saddr, daddr, ports, dif))
            goto not_unique;
    }

unique:
    ......
    return 0;

not_unique:
    spin_unlock(lock);
    return -EADDRNOTAVAIL;
}
```

&emsp;&emsp;&emsp;&emsp;如果是TCP套接字，twsk\_uniqueue()中会调用tcp\_twsk_uniqueue()来判断，返回true的条件如下所示：

```
int tcp_twsk_unique(struct sock *sk, struct sock *sktw, void *twp)
{
    const struct tcp_timewait_sock *tcptw = tcp_twsk(sktw);
    struct tcp_sock *tp = tcp_sk(sk);

    if (tcptw->tw_ts_recent_stamp &&
            (twp == NULL || (sysctl_tcp_tw_reuse &&
            get_seconds() - tcptw->tw_ts_recent_stamp > 1))) {
        ......
        return 1;
    }

    return 0;
}
```

```
/*
 * Build a SYN and send it off.
 */
int tcp_connect(struct sock *sk)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct sk_buff *buff;
    /* 初始化连接对应的INET socket结构的参数，为连接做准备 */
    tcp_connect_init(sk);
    /* 获取一个skb，由于是syn包，没有数据，所以大小是MAX_TCP_HEADER的16位对齐 */
    buff = alloc_skb_fclone(MAX_TCP_HEADER + 15, sk->sk_allocation);
    if (unlikely(buff == NULL))
        return -ENOBUFS;

    /* Reserve space for headers. */
    skb_reserve(buff, MAX_TCP_HEADER);

    tp->snd_nxt = tp->write_seq;
    /* 设置skb相关参数 */
    tcp_init_nondata_skb(buff, tp->write_seq++, TCPCB_FLAG_SYN);
    /* 设置ECN */
    TCP_ECN_send_syn(sk, buff);

    /* Send it off. */
    /* 保存该数据包的发送时间*/
    TCP_SKB_CB(buff)->when = tcp_time_stamp;
    tp->retrans_stamp = TCP_SKB_CB(buff)->when;
    skb_header_release(buff);
    /* 加入发送队列，待确认后在丢弃*/
    __tcp_add_write_queue_tail(sk, buff);
    sk->sk_wmem_queued += buff->truesize;
    sk_mem_charge(sk, buff->truesize);
    tp->packets_out += tcp_skb_pcount(buff);
    tcp_transmit_skb(sk, buff, 1, GFP_KERNEL);

    /* We change tp->snd_nxt after the tcp_transmit_skb() call
     * in order to make this packet get counted in tcpOutSegs.
     */
    tp->snd_nxt = tp->write_seq;
    tp->pushed_seq = tp->write_seq;
    TCP_INC_STATS(sock_net(sk), TCP_MIB_ACTIVEOPENS);

    /* Timer for repeating the SYN until an answer. */
    inet_csk_reset_xmit_timer(sk, ICSK_TIME_RETRANS,
                  inet_csk(sk)->icsk_rto, TCP_RTO_MAX);
    return 0;
}
```

```
/*
 * Do all connect socket setups that can be done AF independent.
 */
static void tcp_connect_init(struct sock *sk)
{
    struct dst_entry *dst = __sk_dst_get(sk);
    struct tcp_sock *tp = tcp_sk(sk);
    __u8 rcv_wscale;

    /* We'll fix this up when we get a response from the other end.
     * See tcp_input.c:tcp_rcv_state_process case TCP_SYN_SENT.
     */
    tp->tcp_header_len = sizeof(struct tcphdr) +
        (sysctl_tcp_timestamps ? TCPOLEN_TSTAMP_ALIGNED : 0);

#ifdef CONFIG_TCP_MD5SIG
    if (tp->af_specific->md5_lookup(sk, sk) != NULL)
        tp->tcp_header_len += TCPOLEN_MD5SIG_ALIGNED;
#endif

    /* If user gave his TCP_MAXSEG, record it to clamp */
    if (tp->rx_opt.user_mss)
        tp->rx_opt.mss_clamp = tp->rx_opt.user_mss;
    tp->max_window = 0;
    /* 初始化MTU probe*/
    tcp_mtup_init(sk);
    /* 设置mss */
    tcp_sync_mss(sk, dst_mtu(dst));

    if (!tp->window_clamp)
        tp->window_clamp = dst_metric(dst, RTAX_WINDOW);
    tp->advmss = dst_metric(dst, RTAX_ADVMSS);
    if (tp->rx_opt.user_mss && tp->rx_opt.user_mss < tp->advmss)
        tp->advmss = tp->rx_opt.user_mss;

    tcp_initialize_rcv_mss(sk);
    /* 根据接收空间大小初始化一个通告窗口 */
    tcp_select_initial_window(tcp_full_space(sk),
                  tp->advmss - (tp->rx_opt.ts_recent_stamp ? tp->tcp_header_len - sizeof(struct tcphdr) : 0),
                  &tp->rcv_wnd,
                  &tp->window_clamp,
                  sysctl_tcp_window_scaling,
                  &rcv_wscale);

    tp->rx_opt.rcv_wscale = rcv_wscale;
    tp->rcv_ssthresh = tp->rcv_wnd;

    sk->sk_err = 0;
    sock_reset_flag(sk, SOCK_DONE);
    tp->snd_wnd = 0;
    /* 更新一些滑动窗口的成员*/
    tcp_init_wl(tp, tp->write_seq, 0);
    tp->snd_una = tp->write_seq;
    tp->snd_sml = tp->write_seq;
    tp->snd_up = tp->write_seq;
    tp->rcv_nxt = 0;
    tp->rcv_wup = 0;
    tp->copied_seq = 0;

    inet_csk(sk)->icsk_rto = TCP_TIMEOUT_INIT;
    inet_csk(sk)->icsk_retransmits = 0;
    tcp_clear_retrans(tp);
}
```

&emsp;&emsp;&emsp;&emsp;skb发送后，connect并没有返回，因为此时连接还没有建立，tcp进入等待状态，此时回到前面的inet\_stream\_connect函数

&emsp;&emsp;&emsp;&emsp;在发送syn后进入等待状态

```
static long inet_wait_for_connect(struct sock *sk, long timeo)
{
    DEFINE_WAIT(wait);
    /* sk_sleep 保存此INET SOCKET的等待队列 */
    prepare_to_wait(sk->sk_sleep, &wait, TASK_INTERRUPTIBLE);

    /* Basic assumption: if someone sets sk->sk_err, he _must_
     * change state of the socket from TCP_SYN_*.
     * Connect() does not allow to get error notifications
     * without closing the socket.
     */
    /* 定时等待知道状态变化 */
    while ((1 << sk->sk_state) & (TCPF_SYN_SENT | TCPF_SYN_RECV)) {
        release_sock(sk);
        timeo = schedule_timeout(timeo);
        lock_sock(sk);
        if (signal_pending(current) || !timeo)
            break;
        prepare_to_wait(sk->sk_sleep, &wait, TASK_INTERRUPTIBLE);
    }
    finish_wait(sk->sk_sleep, &wait);
    return timeo;
}
```


{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用
* 转载: [参考]
{% endhighlight %}

[参考](http://blog.csdn.net/chensichensi/article/details/5272346)
[参考](http://blog.csdn.net/qy532846454/article/details/7882819)
[参考](http://www.2cto.com/kf/201303/198459.html)

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
