+++
date = '2025-11-27T19:16:56+08:00'
draft = false
title = '使用eBPF观测Go程序的goroutine状态变化'
comments = true
tags = ["Go", "eBPF", "C"]

+++

[eBPF](https://ebpf.io/)号称能动态地对内核进行编程，可以实现高效的网络处理、观测、追踪和安全功能。

> Dynamically program the kernel for efficient networking, observability, tracing, and security

这篇博客就来体验一下用eBPF观测Go的goroutine调度，会涉及到uprobe原理、eBPF开发框架选择、eBPF程序的编写、用户态程序加载eBPF程序以及用户态程序和eBPF程序通过map交互等内容。

## uprobe原理

`uprobe`（user-space probe）是 Linux 内核提供的一种机制，用来在 用户态程序的特定指令处 插入动态探针（probe）。`uprobe` 通过在用户态程序代码段指定位置写入一个 1 字节的断点指令（INT3, 0xCC），触发内核陷入（trap）进入 uprobe handler，再运行hook的代码。由于eBPF具有很好的安全性，所以可以用eBPF来编写hook代码，保证安全。



## eBPF开发框架选择

最近几年eBPF发展的非常迅速，出现了优秀的开源eBPF开发框架，使得eBPF的开发难度大大降低。下面选取一些我知道的eBPF开发框架做介绍：

| 项目                                                         | 特点                                                       | 语言      |
| ------------------------------------------------------------ | ---------------------------------------------------------- | --------- |
| [bcc](https://github.com/iovisor/bcc)                        | eBPF开发工具箱，适合教学&运维，依赖LLVM，较重。            | C、Python |
| [libbpf-bootstrap](https://github.com/libbpf/libbpf-bootstrap/tree/master?tab=readme-ov-file) | 用C开发eBPF的模版/框架，官方推荐。                         | C         |
| [cilium/ebpf](https://github.com/libbpf/libbpf-bootstrap/tree/master?tab=readme-ov-file) | 用Go开发eBPF的首选，eilium开源，能用于生产环境，无LLVM依赖 | C、Go     |
| [bpftrace](https://github.com/bpftrace/bpftrace)             | 用类似awk的脚本语言开发eBPF，适合快速调试、验证。          | 专用语法  |
| [eunomia-bpf](https://github.com/eunomia-bpf/eunomia-bpf)    | 面向云原生的eBPF WASM化                                    | Rust      |

经过对比最终选择了使用cilium/ebpf来实现观测goroutine状态变化，原因有：bcc是一个很重的项目，有很多依赖环境。libbpf-bootstrap开发用户态程序相对麻烦一些。使用bpftrace开发需要了解他的脚本语言的语法（并不是通用的脚本语言），有一定的学习成本，而且开发复杂的eBPF程序也不太灵活。eunomia-bpf还没仔细研究过，以后会尝试。经过对比，我最终选择了对我来说最简单的cilium/ebpf，而且这个项目里有现成的[examples](https://github.com/cilium/ebpf/tree/main/examples)，可以照猫画虎，很快就能实现一个简单uprobe功能。



## 用eBPF观测goroutine状态变化

### eBPF程序

Go支持高并发的利器就是goroutine，而goroutine之所以能轻而易举的实现上万的并发，要归功于Go的调度策略，Go使用GMP调度模型。核心就是在用户态实现调度逻辑，避免系统级线程调度时用户态和内核态之间的切换开销。

Go的调度逻辑在源码的src/runtime/proc.go中，当scheduler切换goroutine状态时会调用`runtime.casgstatus`，runtime.casgstatus的原型如下，它的作用是：检查当前goroutine的状态是否old，如果是就把状态原子的更新为new，如果不是，不做更新。

```Go
func casgstatus(gp *g, oldval, newval uint32) 
```

要观测goroutine的状态切换，只需要把eBPF函数hook到这个函数的入口处，拿到这三个参数即可。

首先把[cilium/ebpf](https://github.com/cilium/ebpf)拉到本地，进入examples目录，在创建一个uprobe_sched目录，创建一个uprobe_sched.c文件。

```bash
$ cd ebpf/examples
$ mkdir uprobe_sched
$ cd uprobe_sched
$ touch uprobe_sched.c
```

然后参考隔壁的[examples/uretprobe/uretprobe.c](https://github.com/cilium/ebpf/blob/main/examples/uretprobe/uretprobe.c)，编写eBPF程序，主要逻辑就是，获取三个参数，第一个参数是指向g结构体的指针，第二个参数是old，第三个参数是new。其中goid在g结构体偏移0xA0（这个偏移和Go的版本相关，具体计算方法后面介绍），需要借助`bpf_probe_read_user`函数读取，读到goid之后将data_t结构体写入map中。

```bash
//go:build ignore

#include "common.h"
#include "bpf_tracing.h"

#define GOID_OFFSET 0xA0	// 0xA0是goid在g结构体中的偏移

struct data_t {
	u64 goid;
	u32 old_status;
	u32 new_status;
};

// 定义一个PERF_EVENT map,将执行的结果保存到map中，供用户态程序消费
struct {
	__uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
	__type(value, struct data_t);
} events SEC(".maps");

SEC("uprobe/runtime.casgstatus")
int monitor_sched(struct pt_regs *ctx) {
	u64 goid = 0;
	u64 gp   = (u64)PT_REGS_PARM1(ctx);		// 获取第一个参数
	u32 old   = (u64)PT_REGS_PARM2(ctx);	// 获取第二个参数
	u32 new   = (u64)PT_REGS_PARM3(ctx);	// 获取第三个参数

	if (gp == 0)
		return 0;
	if (bpf_probe_read_user(&goid, sizeof(goid), (void *)(gp + GOID_OFFSET))) {	// 通过bpf helper函数读取指定地址的内容
		return 0;
	}

	struct data_t data = {
		.goid       = goid,
		.old_status = old,
		.new_status = new,
	};

	// bpf_printk("casgstatus: g=0x%d old=%d new=%d\n", goid, old, new);

	bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU, &data, sizeof(data));
	return 0;
}

char LICENSE[] SEC("license") = "GPL";
```

这就是eBPF程序，很简单，长度50行以内。



### 用户态程序

用户态程序的职责就是把eBPF程序加载到内核，attach到指定点，然后读取map内容并打印。

在uprobe_sched目录下创建main.go文件

```bash
$ touch main.go
```

参考[examples/uretprobe/main.go](https://github.com/cilium/ebpf/blob/main/examples/uretprobe/main.go)，可以写出我们自己的main.go，其中多了一个将表示状态的整数转换为可读性更好的字符串函数。

```go
//go:build amd64 && linux

package main

import (
	"bytes"
	"encoding/binary"
	"errors"
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"

	"github.com/cilium/ebpf/link"
	"github.com/cilium/ebpf/perf"
	"github.com/cilium/ebpf/rlimit"
)

//go:generate go tool bpf2go -tags linux -target amd64 bpf uprobe_sched.c -- -I../headers

// G status from src/runtime/runtime2.go
const (
	Gidle = iota
	Grunnable
	Grunning
	Gsyscall
	Gwaiting
	GmoribundUnused
	Gdead
	GenqueueUnused
	Gcopystack
	Gpreempted
)

func GStatus(n uint32) string {
	switch n {
	case Gidle:
		return "idle"
	case Grunnable:
		return "runnable"
	case Grunning:
		return "running"
	case Gsyscall:
		return "syscall"
	case Gwaiting:
		return "waiting"
	case GmoribundUnused:
		return "moribund"
	case Gdead:
		return "dead"
	case GenqueueUnused:
		return "enqueue"
	case Gcopystack:
		return "copystack"
	case Gpreempted:
		return "preempted"
	default:
		return "unkonwn"
	}
}

func main() {
	if len(os.Args) != 3 {
		fmt.Printf("Usage: %s BINARY SYMBOL\n", os.Args[0])
    return
	}

	binPath := os.Args[1]
	symbol := os.Args[2]

	stopper := make(chan os.Signal, 1)
	signal.Notify(stopper, os.Interrupt, syscall.SIGTERM)

	// Allow the current process to lock memory for eBPF resources.
	if err := rlimit.RemoveMemlock(); err != nil {
		log.Fatal(err)
	}

	// Load pre-compiled programs and maps into the kernel.
	objs := bpfObjects{}
	if err := loadBpfObjects(&objs, nil); err != nil {
		log.Fatalf("loading objects: %s", err)
	}
	defer objs.Close()

	// Open an ELF binary and read its symbols.
	ex, err := link.OpenExecutable(binPath)
	if err != nil {
		log.Fatalf("opening executable: %s", err)
	}

	up, err := ex.Uprobe(symbol, objs.MonitorSched, nil)
	if err != nil {
		log.Fatalf("creating uretprobe: %s", err)
	}
	defer up.Close()

	// Open a perf event reader from userspace on the PERF_EVENT_ARRAY map
	// described in the eBPF C program.
	rd, err := perf.NewReader(objs.Events, os.Getpagesize())
	if err != nil {
		log.Fatalf("creating perf event reader: %s", err)
	}
	defer rd.Close()

	go func() {
		// Wait for a signal and close the perf reader,
		// which will interrupt rd.Read() and make the program exit.
		<-stopper
		log.Println("Received signal, exiting program..")

		if err := rd.Close(); err != nil {
			log.Fatalf("closing perf event reader: %s", err)
		}
	}()

	log.Printf("Listening for events..")

	// bpfEvent is generated by bpf2go.
	var event bpfDataT
	fmt.Printf("%-15s %-20s %-20s\n", "Goroutine ID", "Old Status", "New Status")
	for {
		record, err := rd.Read()
		if err != nil {
			if errors.Is(err, perf.ErrClosed) {
				return
			}
			log.Printf("reading from perf event reader: %s", err)
			continue
		}

		if record.LostSamples != 0 {
			log.Printf("perf event ring buffer full, dropped %d samples", record.LostSamples)
			continue
		}

		// Parse the perf event entry into a bpfEvent structure.
		if err := binary.Read(bytes.NewBuffer(record.RawSample), binary.LittleEndian, &event); err != nil {
			log.Printf("parsing perf event: %s", err)
			continue
		}

		fmt.Printf("%-10d %-20s %-20s\n", event.Goid, GStatus(event.OldStatus), GStatus(event.NewStatus))
	}
}
```

然后生成ebpf程序和加载程序：

```Go
$ go generate // 生成ebpf程序和操作ebpf程序的函数
$ go build	// 编译生成加载eBPF的可执行程序
$ ls
bpf_x86_bpfel.go  bpf_x86_bpfel.o  main.go  uprobe_sched*  uprobe_sched.c
$ ./uprobe_sched
Usage: ./uprobe_sched BINARY SYMBOL
```



接下来就找一个Go程序测试一下，随便写一个Go程序到`/tmp/test.go`中。

```Go
// 被观测程序的源码
package main

import (
	"math/rand"
	"time"
)

func task() {
	n := rand.Int() % 5

	for i := 0; i < n; i++ {
		t := rand.Int() % 10
		time.Sleep(time.Duration(t * 100 * int(time.Millisecond)))
	}

}

func main() {

	for {
		go task()

		time.Sleep(time.Second)
	}
}
```

创建并编译上述代码。

```
$ cd /tmp
$ touch test.go
$ go build test.go
```

接下来就是见证奇迹的时刻了，运行uprobe_sched程序，指定被观测二进制的路径以及要观测的函数，观察输出结果如下：

```bash
$ sudo ./uprobe_sched /tmp/test runtime.casgstatus
2025/11/27 21:01:36 Listening for events..
Goroutine ID    Old Status           New Status
0          unkonwn              unkonwn
0          unkonwn              idle
14757395258964568840 unkonwn              unkonwn
0          unkonwn              idle
14757395258964568840 unkonwn              unkonwn
0          unkonwn              idle
14757395258964568840 unkonwn              unkonwn
```

貌似这个输出不对，goroutine ID以及状态都不符合预期，这是为什么呢？经过一番折腾，最终找到了原因——eBPF程序获取的参数有问题，`runtime.casgstatus`函数的三个参数分别在rax，rbx和rcx寄存器中，但是`PT_REGS_PARMn(ctx)`宏获取到的寄存器分别是rdi，rsi和rdx寄存器。前三个参数分别用rdi，rsi和rdx寄存器传递确实是x86_64的Linux的规范，但问题是Go的runtime不遵守这个规范，所以使用`PT_REGS_PARMn(ctx)`宏并没有正确获取参数。那怎么知道到底是怎么传递这三个参数的呢？可以对被测试程序的指定函数进行反汇编来确认参数是怎么传递的。以被测试的test程序为例，反汇编命令如下：

```bash
$ go tool objdump -S -gnu -s runtime.casgstatus test
TEXT runtime.casgstatus(SB) /usr/local/go/src/runtime/proc.go
func casgstatus(gp *g, oldval, newval uint32) {
  0x4363e0		55			PUSHQ BP                             										// push %rbp
  0x4363e1		4889e5			MOVQ SP, BP                          								// mov %rsp,%rbp
  0x4363e4		4883ec40		SUBQ $0x40, SP                       								// sub $0x40,%rsp
	if (oldval&_Gscan != 0) || (newval&_Gscan != 0) || oldval == newval {
  0x4363e8		4889442450		MOVQ AX, 0x50(SP)                    							// mov %rax,0x50(%rsp)
  0x4363ed		895c2458		MOVL BX, 0x58(SP)                    								// mov %ebx,0x58(%rsp)
  0x4363f1		894c245c		MOVL CX, 0x5c(SP)                    								// mov %ecx,0x5c(%rsp)
  ......
```

如果对汇编比较熟悉，可以确定三个参数分别是通过rax、rbx和rcx传递的，如果不熟悉也没关系，直接把反汇编结果扔给GPT，让GPT给你答案即可。确定了问题原因，下面把eBPF程序中获取参数的三行代码改成如下。

```C
	// u64 gp   = (u64)PT_REGS_PARM1(ctx);
	// u32 old   = (u64)PT_REGS_PARM2(ctx);
	// u32 new   = (u64)PT_REGS_PARM3(ctx);
	u64 gp  = (u64)ctx->rax;
	u32 old = (u32)ctx->rbx;
	u32 new = (u32)ctx->rcx;
```

再重新生成eBPF程序和加载程序，然后观察，这次没有出现意外，显示正常了。

```bash
$ go generate
$ go build
$ sudo ./uprobe_sched /tmp/test runtime.casgstatus
2025/11/27 21:18:26 Listening for events..
Goroutine ID    Old Status           New Status
0          idle                 dead
0          dead                 runnable
1          runnable             running
0          idle                 dead
0          dead                 runnable
0          idle                 dead
2          runnable             running
2          running              waiting
```



## 总结

使用cilium/ebpf开发uprobe分为几个部分：

1. 分析被观测的对象，确定要观测的位置；
2. 编写eBPF程序，对于观测程序，一般是获取一些环境信息（如参数、返回值等），而不改变被观测对象的环境。获取到的环境信息一般要写入map供用户态程序消费；获取环境信息就需要根据实际情况进行分析，没有绝对统一的方法。
3. 编写用户态程序，用户态程序负责open、load和attach eBPF程序，并且从map中读出eBPF程序写入的内容并进行处理。

借助eBPF的开发框架，开发uprobe难度并不大。但是需要注意的是如何获取环境信息，像这里举的例子，获取参数就不能简单的用宏来获取。另外这个例子里要确定goid在g结构体中的偏移，我使用的方法是用dlv动态调试。

```bash
$ dlv exec test // 调试被测程序
Type 'help' for list of commands.
(dlv) b runtime.casgstatus	// 在观测点设置断点
Breakpoint 1 set at 0x4363e4 for runtime.casgstatus() /usr/local/go/src/runtime/proc.go:1175
(dlv) c	// 运行程序
> [Breakpoint 1] runtime.casgstatus() /usr/local/go/src/runtime/proc.go:1175 (hits total:1) (PC: 0x4363e4)
Warning: debugging optimized function
  1170:	// and casfrom_Gscanstatus instead.
  1171:	// casgstatus will loop if the g->atomicstatus is in a Gscan status until the routine that
  1172:	// put it in the Gscan state is finished.
  1173:	//
  1174:	//go:nosplit
=>1175:	func casgstatus(gp *g, oldval, newval uint32) {
  1176:		if (oldval&_Gscan != 0) || (newval&_Gscan != 0) || oldval == newval {
  1177:			systemstack(func() {
  1178:				// Call on the systemstack to prevent print and throw from counting
  1179:				// against the nosplit stack reservation.
  1180:				print("runtime: casgstatus: oldval=", hex(oldval), " newval=", hex(newval), "\n")
(dlv) args	// 查看参数信息
gp = (*runtime.g)(0xc0000061c0)	// 这里是g结构体的起始地址
oldval = 0
newval = 6
(dlv) p &gp.goid
(*uint64)(0xc000006260)	// 这里是g结构体中goid成员的地址，所以goid在g中的偏移量是 0xc000006260 - 0xc0000061c0 = 0xa0
```

