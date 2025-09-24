+++
date = '2025-09-24T15:06:05+08:00'
draft = false
comments = true
title = 'Linux 网络编程——socket 系统调用实现剖析'
tags = ["Networking", "Linux", "Socket", "C", "Rust", "Go"]

+++

# Linux 下多语言网络编程对比

还记得Linux网络编程姿势吗？如果不记得了，这里有一个用C语言写的`tcp_echo`服务，用这段代码能帮我们回忆Linux的网络编程套路：

1. 调用socket创建一个网络套接字socket；
2. 调用bind给socket绑定地址；
3. listen设置
4. 调用accept接收网络请求；

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

int main(void) {
    int sd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_port = htons(12345),
        .sin_addr.s_addr = INADDR_ANY,
    };

    bind(sd, (struct sockaddr*)&addr, sizeof(addr));
    listen(sd, 5);

    while (1) {
        int client = accept(sd, NULL, NULL);
        char buf[1024];
        ssize_t n;
        while ((n = recv(client, buf, sizeof(buf), 0)) > 0) {
            send(client, buf, n, 0); // 把读到的内容发送回去
        }
        close(client);
    }

    close(sd);
    return 0;
}
```

如果你用的是Go、Python、Rust等高级编程语言，可能会对这段代码嗤之以鼻，这么简单一个功能，要创建一个可以通信的TCP连接完全不必这么复杂。

这是用`Go`写的，如果不考虑错误处理，只需要调用`Listen`和`Accept`。

```Go
package main

import (
        "io"
        "net"
)

func main() {
        ln, err := net.Listen("tcp", "127.0.0.1:12345")
        if err != nil {
                panic("listen err")
        }

        for {
                conn, err := ln.Accept()
                if err != nil {
                        panic("accept error")
                }

                go func(conn net.Conn) {
                        defer conn.Close()
                        io.Copy(conn, conn)
                }(conn)
        }
}
```

使用`Rust`标准库`std::net`的同步版本如下。

```rust
fn main() {
    let l = std::net::TcpListener::bind(("127.0.0.1", 12345)).unwrap();
    for s in l.incoming() {
        std::thread::spawn(move || {
            let mut w = s.unwrap();
            let mut r = w.try_clone().unwrap();
            let _ = std::io::copy(&mut r, &mut w);
        });
    }
}
```

使用`Rust`标准库`tokio::net`的异步版本如下。

```rust
#[tokio::main]
async fn main() {
    let l = tokio::net::TcpListener::bind(("127.0.0.1", 12345))
        .await
        .unwrap();
    loop {
        let (s, _) = l.accept().await.unwrap();
        tokio::spawn(async move {
            let (mut r, mut w) = s.into_split();
            let _ = tokio::io::copy(&mut r, &mut w).await;
        });
    }
}
```



可以看到，使用不同语言进行网络编程，风格差别很大。但是实际不管是C这样古老的编程语言，还是像Go、Rust这样比较现代化的高级编程语言，他们都是对内核提供的网络编程接口进行了封装，本质上底层上是一样的。怎么证明这一结论的正确性呢？可以使用`strace`来追踪进程的系统调用，看看进程调用了哪些系统调用。

以下是对上面四段代码编译后使用strace工具追踪的结果：

**C版追踪结果**

![echo_c](/images/posts/linux-networking-socket-syscall/strace_echo_c.png)

可以看到追踪结果和我们在代码中的调用顺序以及传入的参数一致，但实际我们的C代码并不是直接调用了系统调用的socket，而是调用的glibc封装的socket等函数，而这些函数再真正调用socket等系统调用。



**Go版追踪结果**

![echo_c](/images/posts/linux-networking-socket-syscall/strace_echo_go.png)

可以看到Go代码编译后执行过程中也调用了`socket`、`bind`、`listen`、`accept`等系统调用，而且好多参数都不是我们指定的，看到这里是不是觉得还是C更灵活，因为C可以自由指定系统调用的参数。



**Rust std::net版追踪结果**

![echo_c](/images/posts/linux-networking-socket-syscall/strace_echo_rust.png)

追踪结果和Go差不多，也是调用了`socket`、`bind`、`listen`、`accept`，很多参数也都不是我们自己传入的。



**Rust tokio::net版追踪结果**

![echo_c](/images/posts/linux-networking-socket-syscall/strace_echo_rust_async.png)

追踪结果和使用`std::net`实现的版本不太一样，主要是没有看到`accept`调用，但是多了`epool_ctl`和`futex`调用，这也是tokio能支持异步的关键。



说了这么多，其实就是想证明：不管使用什么语言进行网络编程，最终都是殊途同归。所以要想了解Linux的网络原理，就不得不深入研究这些系统调用都干了啥。接下来我们就一步一步来分析`socket`系统调用干了啥。



## `socket` 系统调用实现剖析

**说明**：以下分析使用的Linux内核源码版本为[v4.4.302](https://elixir.bootlin.com/linux/v4.4.302/source/)。



`socket`系统调用的实现在[net/socket.c](https://elixir.bootlin.com/linux/v4.4.302/source/net/socket.c#L1202)中，`socket`中最关键的动作有两步：

* 调用`sock_create`创建socket对象；
* 调用`sock_map_fd`将sock映射成fd（用于返回给用户态代码）；

```C
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
{
	int retval;
	struct socket *sock;
	int flags;

	// ......

	retval = sock_create(family, type, protocol, &sock);
	if (retval < 0)
		goto out;

	retval = sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK));
	// ......

	return retval;
}
```

从socket系统调用的实现可以看到：`socket`把接收的三个参数（`family`，`type`，`protocol`）全都传递给了`sock_create`，另外还把sock指针的地址也传进入了，如果`sock_create`成功创建了socket对象，就会把sock指针指向这个对象，只需要给socket函数返回一个表示是否出错的整数值即可。

### sock_create分析

下面就看看`sock_create`的[实现](https://elixir.bootlin.com/linux/v4.4.302/source/net/socket.c#L1190)，可以发现sock_create又加了一个参数，然后调用`__sock_create`。这里添加的第一个参数表示**当前进程所在的网络命名空间**（`current`指向当前进程`task_struct`，`current->nsproxy->net_ns`就是当前进程所在的网络命名空间），这个可以用来做*网络隔离*，如*docker*、*ip netns*就会用到这个。但是网络命名空间这个参数不能通过调用函数来传递，而是直接取自当前进程所处的命名空间，所以如果在某个网络命名空间创建socket，需要在调用`socket`之前先调用`setns`切换到目标命名空间，在socket返回之后，可以再切换到其他命名空间，此时创建的socket已经和目标socket绑定了，无论进程在哪个网络命名空间，都不会影响socket收发数据包。

```C
int sock_create(int family, int type, int protocol, struct socket **res)
{
	return __sock_create(current->nsproxy->net_ns, family, type, protocol, res, 0);
}
EXPORT_SYMBOL(sock_create);
```



[`__sock_create`](https://elixir.bootlin.com/linux/v4.4.302/source/net/socket.c#L1077)函数比较长，在研究原理的时候，我们把源码中参数检查、安全及返回值检查相关的代码删掉，只留下关键部分，就得到了下面的代码。可以看到主要分为：

* 调用`sock_alloc`创建一个`socket`对象；
* 根据`family`参数确定具体的协议族；
* 调用具体的协议族的`create`函数；

```C
int __sock_create(struct net *net, int family, int type, int protocol,
			 struct socket **res, int kern)
{
	int err;
	struct socket *sock;
	const struct net_proto_family *pf;

	sock = sock_alloc();	// 创建sock
	
	sock->type = type;		// type赋值给sock->type

	rcu_read_lock();
	pf = rcu_dereference(net_families[family]);		// 根据 family 找到对应的协议族
	
	err = pf->create(net, sock, protocol, kern);	// 调用目标协议族的create函数
	
	*res = sock; 	// 把成功创建的sock保存到res中，供调用方使用
    return 0;
}
EXPORT_SYMBOL(__sock_create);
```

我们调用socket函数的时候，传入的family是AF_INET，这个family对应的协议族是什么呢？其实就是`inet_family_ops`，这是[`inet_init`](https://elixir.bootlin.com/linux/v4.4.302/source/net/ipv4/af_inet.c#L1731)注册的。从`__sock_create`中看到会调用对应协议族的create函数，对于`inet_family_ops`而言，create函数就是`inet_create`。

```C
static const struct net_proto_family inet_family_ops = {
	.family = PF_INET,		// PF_INET 和 AF_INET 相等
	.create = inet_create,	// 使用AF_INET作为family调用socket，最终会执行inet_create函数。
	.owner	= THIS_MODULE,
};

static int __init inet_init(void)
{
	struct inet_protosw *q;
	struct list_head *r;
	int rc = -EINVAL;

	sock_skb_cb_check_size(sizeof(struct inet_skb_parm));

	rc = proto_register(&tcp_prot, 1);	// 注册tcp
	rc = proto_register(&udp_prot, 1);	// 注册udp
	rc = proto_register(&raw_prot, 1);	// 注册raw
	rc = proto_register(&ping_prot, 1);	// 注册ping

	(void)sock_register(&inet_family_ops);	// 注册协议族
    
    // 初始化inetsw列表
    /* Register the socket-side information for inet_create. */
	for (r = &inetsw[0]; r < &inetsw[SOCK_MAX]; ++r)
		INIT_LIST_HEAD(r);

    // 注册protocol(SOCK_STREAM, SOCK_DGRAM、SOCK_RAW)
	for (q = inetsw_array; q < &inetsw_array[INETSW_ARRAY_LEN]; ++q)
		inet_register_protosw(q);
}
```

下面就来分析[`inet_create`](https://elixir.bootlin.com/linux/v4.4.302/source/net/ipv4/af_inet.c#L249)干了啥。inet_create主要逻辑是根据`socket`系统调用的第二个参数`type`和第三个参数`protocol`来确定具体的传输层（如TCP、UDP、ICMP、RAW）的`inet_protosw`对象。并创建传输层的`sock`对象，最终将`inet_protosw`的操作和`socket`对象关联，以及`socket`对象和`sock`对象相互关联。

```C
static int inet_create(struct net *net, struct socket *sock, int protocol,
		       int kern)
{
	struct sock *sk;
	struct inet_protosw *answer;
	struct inet_sock *inet;
	struct proto *answer_prot;
	unsigned char answer_flags;
	int try_loading_module = 0;
	int err;

	if (protocol < 0 || protocol >= IPPROTO_MAX)
		return -EINVAL;

	sock->state = SS_UNCONNECTED;	// 初始化socket状态

	/* Look for the requested type/protocol pair. */
lookup_protocol:
	err = -ESOCKTNOSUPPORT;
	rcu_read_lock();
    // 根据sock->type（socket的第二个参数）遍历sock->type对应的链表，找到对应的protocol的
    // inet_protosw对象
	list_for_each_entry_rcu(answer, &inetsw[sock->type], list) {

		err = 0;
		/* Check the non-wild match. */
		if (protocol == answer->protocol) {	// 优先精确匹配protocol
			if (protocol != IPPROTO_IP)
				break;
		} else {
			/* Check for the two wild cases. */
			if (IPPROTO_IP == protocol) {
				protocol = answer->protocol;
				break;
			}
			if (IPPROTO_IP == answer->protocol)
				break;
		}
		err = -EPROTONOSUPPORT;
	}

	sock->ops = answer->ops;		// 将匹配到的 inet_protosw对象的ops赋值给sock->ops
	answer_prot = answer->prot;
	answer_flags = answer->flags;
    
	// 分配一个sock，注意是sock不是socket！！！
    // struct socket是内核套接字对象
    // struct sock是传输层的socket状态
	sk = sk_alloc(net, PF_INET, GFP_KERNEL, answer_prot, kern);	

	inet = inet_sk(sk);		// 将struct *sock转换成struct *inet_sk，换个视角看待内存
	inet->is_icsk = (INET_PROTOSW_ICSK & answer_flags) != 0;

	inet->nodefrag = 0;

	if (SOCK_RAW == sock->type) {
		inet->inet_num = protocol;
		if (IPPROTO_RAW == protocol)
			inet->hdrincl = 1;
	}
	// 关联struct socket和struct sock，并出书画struct sock内部某些字段
	sock_init_data(sock, sk);	

	sk->sk_destruct	   = inet_sock_destruct;
	sk->sk_protocol	   = protocol;
	sk->sk_backlog_rcv = sk->sk_prot->backlog_rcv;

out:
	return err;
}

```



[`sock_init_data`](https://elixir.bootlin.com/linux/v4.4.302/source/net/core/sock.c#L2407)的作用是对sock对象的成员初始化，如状态（sk_state）、收发缓冲区大小（sk_sndbuf、sk_rcvbuf）等，以及将内核的socket对象（struct socket）和传输层的sock对象（struct sock）相互关联。

```C
void sock_init_data(struct socket *sock, struct sock *sk)
{
	skb_queue_head_init(&sk->sk_receive_queue);
	skb_queue_head_init(&sk->sk_write_queue);
	skb_queue_head_init(&sk->sk_error_queue);

    // 初始化struct sock的一些字段，
	sk->sk_send_head	=	NULL;
	init_timer(&sk->sk_timer);
	sk->sk_allocation	=	GFP_KERNEL;
	sk->sk_rcvbuf		=	sysctl_rmem_default;
	sk->sk_sndbuf		=	sysctl_wmem_default;
	sk->sk_state		=	TCP_CLOSE;
    
    
	sk_set_socket(sk, sock);	// sk->sk_socket 指向 sock

	if (sock) {
		sk->sk_type	=	sock->type;
		sk->sk_wq	=	sock->wq;
		sock->sk	=	sk;			// sock->sk指向sk
	} else
		sk->sk_wq	=	NULL;

	// ......
	sk->sk_rcvtimeo		=	MAX_SCHEDULE_TIMEOUT;
	sk->sk_sndtimeo		=	MAX_SCHEDULE_TIMEOUT;
}
EXPORT_SYMBOL(sock_init_data);
```

至此，我们分析完了内核的socket对象的初始化过程。大概思路是：先根据socket系统调用的第一个参数family找到`net_proto_family`对象，调用这个对象的`create`函数。而`net_proto_family`的`create`函数再根据`socket`系统调用的第二、三个参数*type*和*protocol*来确定具体的传输层，然后把传输层的一些操作和socket对象关联，另外还创建了一个传输层的*sock*对象也和*socket*对象关联起来。至此一个*socket*对象就算创建并初始化完成。



### sock_map_fd分析

socket系统调用的本质就是创建一个socket内核对象，但是这个内核对象是在内核的，不能直接把这个内核对象的地址返回给用户态。我们知道Linux（或者是Unix）的哲学就是**一切皆文件**。所以socket系统调用最终会将socket内核对象以文件描述符的形式传递给用户态。[`sock_map_fd`](https://elixir.bootlin.com/linux/v4.4.302/source/net/socket.c#L392)函数就是做这个工作的。这个函数的逻辑主要分为三步：

* 申请一个新的可用fd；
* 为struct socket分配一个struct file；
* 将struct file和申请的fd关联，并最终返回这个fd；

```C
static int sock_map_fd(struct socket *sock, int flags)
{
	struct file *newfile;
	int fd = get_unused_fd_flags(flags);	// 申请一个可用的fd
	if (unlikely(fd < 0))
		return fd;

	newfile = sock_alloc_file(sock, flags, NULL);	// 为sock分配一个strict file对象
	if (likely(!IS_ERR(newfile))) {
		fd_install(fd, newfile);	// 把刚申请的fd和file对象绑定
		return fd;
	}

	put_unused_fd(fd);
	return PTR_ERR(newfile);
}
```



## 总结

通过剖析socket系统调用，我们了解了socket对象是如何创建并被初始化的。
