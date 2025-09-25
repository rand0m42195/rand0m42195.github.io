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



## `socket()` 系统调用实现剖析

**说明**：以下分析使用的Linux内核源码版本为[v4.4.302](https://elixir.bootlin.com/linux/v4.4.302/source/)。



`socket()`系统调用的实现在[net/socket.c](https://elixir.bootlin.com/linux/v4.4.302/source/net/socket.c#L1202)中，`socket()`中最关键的动作有两步：

* 调用`sock_create()`创建内核*socket对象*；
* 调用`sock_map_fd()`将*socket对象*映射成fd（用于返回给用户态代码）；

```C
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
{
	int retval;
	struct socket *sock;
	int flags;

	/* Check the SOCK_* constants for consistency.  */
	BUILD_BUG_ON(SOCK_CLOEXEC != O_CLOEXEC);
	BUILD_BUG_ON((SOCK_MAX | SOCK_TYPE_MASK) != SOCK_TYPE_MASK);
	BUILD_BUG_ON(SOCK_CLOEXEC & SOCK_TYPE_MASK);
	BUILD_BUG_ON(SOCK_NONBLOCK & SOCK_TYPE_MASK);

    // 检查、处理参数
	flags = type & ~SOCK_TYPE_MASK;
	if (flags & ~(SOCK_CLOEXEC | SOCK_NONBLOCK))
		return -EINVAL;
	type &= SOCK_TYPE_MASK;

	if (SOCK_NONBLOCK != O_NONBLOCK && (flags & SOCK_NONBLOCK))
		flags = (flags & ~SOCK_NONBLOCK) | O_NONBLOCK;

    // 创建socket对象
	retval = sock_create(family, type, protocol, &sock);
	if (retval < 0)
		goto out;
	// 为socket对象分配fd
	retval = sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK));
	if (retval < 0)
		goto out_release;

out:
	/* It may be already another descriptor 8) Not kernel problem. */
	return retval;

out_release:
	sock_release(sock);
	return retval;
}
```

从`socket()`系统调用的实现可以看到：`socket()`把接收的三个参数（`family`，`type`，`protocol`）全都传递给了`sock_create()`，另外还把保存*socket对象*地址的指针`sock`的地址也传进入了，如果`sock_create()`成功创建了*socket对象*，它就会把`sock`指针指向这个对象，只需要给`socket()`返回一个表示是否出错的整数值即可。

### sock_create分析

下面就看看`sock_create()`的[实现](https://elixir.bootlin.com/linux/v4.4.302/source/net/socket.c#L1190)，可以发现`sock_create()`又加了一个参数，然后调用`__sock_create()`。这里添加的第一个参数表示**当前进程所在的网络命名空间**（`current`指向当前进程`task_struct`，`current->nsproxy->net_ns`就是当前进程所在的网络命名空间），这个可以用来做*网络隔离*，如*docker*、*ip netns*就会用到这个。网络命名空间这个参数**不能**通过调用方通过参数来指定，而是直接取自当前进程所处的命名空间，所以如果在某个网络命名空间创建socket，需要在调用`socket`之前先调用`setns`切换到目标命名空间，在socket返回之后，可以再切换到其他命名空间，此时创建的socket已经和目标socket绑定了，无论进程在哪个网络命名空间，都不会影响socket收发数据包。

```C
int sock_create(int family, int type, int protocol, struct socket **res)
{
	return __sock_create(current->nsproxy->net_ns, family, type, protocol, res, 0);
}
EXPORT_SYMBOL(sock_create);
```



[`__sock_create()`](https://elixir.bootlin.com/linux/v4.4.302/source/net/socket.c#L1077)函数比较长，在研究原理的时候，我们把源码中参数检查、安全及返回值检查相关的代码删掉，只留下关键部分，就得到了下面的代码。可以看到主要分为：

* 调用`sock_alloc()`创建一个*socket对象*；
* 根据`family`参数确定具体的协议族；
* 调用具体的协议族的`create()`函数；

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

我们调用`socket()`的时候，传入的`family`是*AF_INET*，这个`family`对应的协议族是什么呢？其实就是`inet_family_ops`，这是[`inet_init`](https://elixir.bootlin.com/linux/v4.4.302/source/net/ipv4/af_inet.c#L1731)注册的。从`__sock_create()`中看到会调用对应协议族的`create()`函数，对于`inet_family_ops`而言，create函数就是`inet_create()`。

```C
static const struct net_proto_family inet_family_ops = {
	.family = PF_INET,		// PF_INET 和 AF_INET 相等
	.create = inet_create,	// 使用AF_INET作为family调用socket，最终会执行inet_create函数。
	.owner	= THIS_MODULE,
};

static int __init inet_init(void)
{
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

下面就来分析[`inet_create()`](https://elixir.bootlin.com/linux/v4.4.302/source/net/ipv4/af_inet.c#L249)干了啥。`inet_create()`主要逻辑是根据`socket()`系统调用的第二个参数`type`和第三个参数`protocol`来确定具体的传输层（如TCP、UDP、ICMP、RAW）的`inet_protosw`对象。并创建**传输层**的*sock对象*，最终将`inet_protosw()`的操作和*socket对象*关联，并将*socket对象*（struct socket）和传输层*sock对象*（struct sock）相互绑定。

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



[`sock_init_data()`](https://elixir.bootlin.com/linux/v4.4.302/source/net/core/sock.c#L2407)的作用是对传输层*sock对象*的成员初始化，如状态（sk_state）、收发缓冲区大小（sk_sndbuf、sk_rcvbuf）等，以及将*socket对象*和传输层的*sock对象*相互关联。

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

至此，我们分析完了*socket对象*的初始化过程。大概思路是：先根据`socket()`系统调用的第一个参数`family`找到`net_proto_family`对象，调用这个对象的`create()`函数。而`net_proto_family`的`create()`函数再根据`socket()`系统调用的第二、三个参数`type`和`protocol`来确定具体的传输层，然后把传输层的一些操作和*socket对象*关联，另外还创建了一个传输层的*sock对象*也和*socket对象*关联起来。至此一个*socket对象*就算创建并初始化完成。



### sock_map_fd分析

`socket()`系统调用的本质就是创建一个socket对象，但是内核不能直接把这个内核态的对象返回给用户态。我们知道Linux（或者是Unix）的哲学就是**一切皆文件**，所以`socket()`系统调用最终会将内核socket对象以文件描述符的形式传递给用户态，这样还有一个好处，即可以使用`read()`、`write()`等通用的文件操作。[`sock_map_fd`](https://elixir.bootlin.com/linux/v4.4.302/source/net/socket.c#L392)函数就是做这个工作的。这个函数的逻辑主要分为三步：

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

    // 为sock分配一个struct file对象
    // struct file的f_op也会被设置为socket专用操作集（socket_file_ops）
	newfile = sock_alloc_file(sock, flags, NULL);	
	if (likely(!IS_ERR(newfile))) {
		fd_install(fd, newfile);	// 把刚申请的fd和file对象绑定
		return fd;
	}

	put_unused_fd(fd);
	return PTR_ERR(newfile);
}
```



## 总结

在 Linux 内核中，`socket()` 系统调用并不是“神秘的黑盒”，而是逐层往下、逐步构建的过程。其核心流程可以总结为：

1. **参数解析与标志拆分** — 在 syscall 层处理 `type` 中的 `SOCK_CLOEXEC` 和 `SOCK_NONBLOCK` 等标志；
2. **`sock_create()` / `__sock_create()`** — 指定网络命名空间、根据协议族选择 `net_proto_family`，并调用对应 `create` 方法；
3. **具体协议族创建** — 比如 AF_INET 的 `inet_create()`，遍历 `inetsw` 匹配协议类型（TCP/UDP/RAW），分配 `struct sock`；
4. **初始化与绑定** — 通过 `sock_init_data()` 将 `struct socket` 与 `struct sock` 关联并初始化各字段；
5. **文件描述符映射** — 通过 `sock_map_fd()` 分配 fd、创建 `struct file`、将 file 和 fd 绑定，使用户态可以通过 fd 操作套接字。

通过这一过程，我们能够看到几个关键的设计思想：

- **对象-句柄分离**：用户态看到的只是一个整数 fd，而真正的 socket 结构体在内核中；
- **协议层抽象与接口复用**：不同协议（TCP/UDP/RAW）共享统一的处理框架，只在 `inet_protosw` 中区别；
- **网络命名空间隔离**：套接字从一开始就属于某个 `struct net`，支持容器、namespace 的网络隔离；

掌握了这条按照层级拆解的思路，以后要深入分析更复杂的网络机制（如 `connect()`、`accept()`、`send()`/`recv()`、中断处理、TCP 状态机等）就更轻松、系统化。

希望这篇剖析能帮助你揭开`socket()`系统调用的神秘面纱。
