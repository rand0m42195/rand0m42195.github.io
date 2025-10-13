+++
date = '2025-10-14T06:39:46+08:00'
draft = true
title = 'Go Testing'
+++

# TL; DR
本文介绍了Go原生支持的testing的两种测试方法（包内测试和包外测试）和Go支持的四种测试类型（`TestXxxx`、`FuzzXxxx`、`BenchmarkXxxx`和`ExampleXxxx`）以及使用IDE提示的`run test`/`debug test`和手动执行`go test`命令的区别。和实际项目开发中的被测对象相比，本文的示例比较简单，只是用于说明如何编写四种Go原生支持的测试函数。

# Testing

## 测试的意义
测试是软件生命周期中一个重要部分，测试能带来很多好处：
- 减少代码缺陷，提升软件质量；
- 起到文档说明的作用，降低使用门槛；
- 加深开发人员对代码的理解，提高开发人员的自信；
- 提高软件开发效率；
- ......



## Go Testing
Go对test有很好的支持，go专门提供了用于测试的test子命令，测试代码需要写在以go项目中以_test.go结尾的文件中。Go提供了包内测试和包外测试，测试类型又可分为四种：`TestXxxx`、`BenchmarkXxxx`、`FuzzXxx`、`ExampleXxx`。

### 包内测试 vs 包外测试
#### 包内测试
包内测试面向实现。包内测试可以访问包内的所有符号（包括未导出的符号）；测试代码的测试数据构造和测试逻辑通常与被测包的数据结构以及具体实现逻辑紧密结合。因此，如果修改了被测包的数据结构/实现逻辑，一般需要同步调整包内测试代码。
#### 包外测试
包外测试面向接口。包外测试只能访问被测包导出的API；被测包的API是与外部交互的契约，契约一旦确定就应该长期保持稳定和向前兼容。因此一般修改被测包内部的数据结构和具体实现逻辑不影响包外测试代码。



#### 四种测试类型

目前Go支持4种测试类型：四种测试方法的命名分别为：`TestXxxx`、`BenchmarkXxxx`、`FuzzXxx`和`ExampleXxx`。



##### TestXxxx

**用途：**用来检查被测代码的输出是否符合预期，最常用的一种测试类型。

**示例：**实现一个查找[最长无重复子串](https://leetcode.cn/problems/longest-substring-without-repeating-characters/description/)的函数；([直接运行](https://go.dev/play/p/PHbBzl0pPsC)）

```Go
// mytest/mytest.go
// leetcode problem_3: https://leetcode.cn/problems/longest-substring-without-repeating-characters/description/
func LengthOfLongestSubstring(s string) int {
    if len(s) == 0 {
        return 0
    }

    // 哈希集合，记录每个字符是否出现过
    m := map[byte]int{}
    n := len(s)
    // 右指针，初始值为 -1，相当于我们在字符串的左边界的左侧，还没有开始移动
    rk, ans := -1, 0
    for i := 0; i < n; i++ {
        if i != 0 {
            // 左指针向右移动一格，移除一个字符
            delete(m, s[i-1])
        }
        for rk+1 < n && m[s[rk+1]] == 0 {
            // 不断地移动右指针
            m[s[rk+1]]++
            rk++
        }
        // 第 i 到 rk 个字符是一个极长的无重复字符子串
        ans = max(ans, rk-i+1)
    }
    return ans
}

func max(x, y int) int {
    if x < y {
        return y
    }
    return x
}
```

如何验证这个函数是否符合预期？可以写一个简单的main函数，构造一些测试用例，然后调用这个函数。但是Go有自己的测试方法，只需要在被测代码的目录下创建一个以`_test.go`结尾的文件，并编写一个测试函数即可：

```Go
// mytest/mytest_test.go
import "testing"

func TestLengthOfLongestSubstringV0(t *testing.T) {
    case1 := "abcabcbb" // "abc"

    got := LengthOfLongestSubstring(case1)
    if got != 3 {
        t.Fatalf("input: %s, want: %d, got: %d", case1, 3, got)
    }
}
```

在上面测试文件所在的目录下，执行如下命令即可：

```Bash
$ go test -run=^TestLengthOfLongestSubstringV0$
PASS
ok      letsgo/mytest   0.005s
```

为了验证上面函数的正确性和健壮性，需要更多的测试用例，下面是一个有5个测试用例的测试函数：

```Go
// mytest/mytest_test.go

func TestLengthOfLongestSubstringV1(t *testing.T) {
    case1 := "a"        // " ", 1
    case2 := "ab"       // "a", 1
    case3 := "abcabcbb" // "abc", 3
    case4 := "bbbbb"    // "b", 1
    case5 := "pwwkew"   // "wke", 3
    exp1 := 1
    exp2 := 2
    exp3 := 3
    exp4 := 1
    exp5 := 3

    got := LengthOfLongestSubstring(case1)
    if got != exp1 {
        t.Fatalf("input: %s, want: %d, got: %d", case1, exp1, got)
    }

    got = LengthOfLongestSubstring(case2)
    if got != exp2 {
        t.Fatalf("input: %s, want: %d, got: %d", case2, exp2, got)
    }

    got = LengthOfLongestSubstring(case3)
    if got != exp3 {
        t.Fatalf("input: %s, want: %d, got: %d", case3, exp3, got)
    }

    got = LengthOfLongestSubstring(case4)
    if got != exp4 {
        t.Fatalf("input: %s, want: %d, got: %d", case4, exp4, got)
    }

    got = LengthOfLongestSubstring(case5)
    if got != exp5 {
        t.Fatalf("input: %s, want: %d, got: %d", case5, exp5, got)
    }
}
```



执行上面的测试用例，全部通过。`TestLengthOfLongestSubstringV1`和`TestLengthOfLongestSubstringV0`比，多了四个测试用例，但是代码行数增加了许多，而且增加的代码逻辑完全相同。如果继续增加测试用例，测试代码的行数将会成倍增长，对后面维护测试代码不利。

```Go
$ go test -run=^TestLengthOfLongestSubstringV1$
PASS
ok      letsgo/mytest   0.007s
```

使用`table-driven`方法编写测试用例，可以避免上述问题，示例如下。`table-driven`专注于测试用例的构造，所有的测试用例使用同一套测试代码，随着开发的迭代，测试代码的维护一般只需要添加测试用例即可。

```Go
// mytest/mytest_test.go

func TestLengthOfLongestSubstringV2(t *testing.T) {

    tests := []struct {
        input  string
        expect int
    }{
        {"a", 1},
        {"ab", 2},
        {"abcabcbb", 3},
        {"bbbbb", 1},
        {"pwwkew", 3},
    }

    for i, tt := range tests {
        got := LengthOfLongestSubstring(tt.input)
        if got != tt.expect {
            t.Errorf("case_%d: %s, expect: %d, got: %d", i, tt.input, tt.expect, got)
            // Error vs Fatal
        }
    }
}
```

测试上述测试代码，全部通过！

```Bash
$ go test -run=^TestLengthOfLongestSubstringV2$
PASS
ok      letsgo/mytest   0.006s
```

Go还提供了统计代码覆盖率的工具，可以使用代码覆盖率统计工具来统计代码的覆盖率，对于测试代码，只需要在执行`go test`时添加`-cover`参数即可。从下面的输出可以看到被测代码的覆盖率为93.8%。

```Bash
$ go test -run=^TestLengthOfLongestSubstringV2$ -cover
PASS
coverage: 93.8% of statements
ok      letsgo/mytest   0.008s
```

如果想继续提高代码的覆盖率，就需要知道当前的测试用例覆盖了哪些代码，还有哪些代码没有覆盖到。Go也提供了相关的功能，只需要给`go test`命令添加`-coverprofile`参数，让`go test`生成代码覆盖信息即可。

```Bash
# 将代码覆盖信息输出到 c.out 文件
$ go test -cover -coverprofile c.out -run=^TestLengthOfLongestSubstringV2$
PASS
coverage: 93.8% of statements
ok          letsgo/mytest        0.005s
```

查看*c.out*文件，*c.out*是一个文本文件，从文件内容上看，大概可以猜到，源码行号后面的`0/1`标识代码是否被执行过。但是这个文件内容对人类不太友好，可以使用`go tool cover`将其转换成`html`查看(具体操作执行go tool cover查看帮助)。

```Bash
# 查看文件类型
$ file c.out
c.out: ASCII text

# 查看文件内容
$ cat c.out
mode: set
letsgo/mytest/mytest.go:4.45,5.17 1 1
letsgo/mytest/mytest.go:5.17,7.3 1 0
letsgo/mytest/mytest.go:10.2,14.25 4 1
letsgo/mytest/mytest.go:14.25,15.13 1 1
letsgo/mytest/mytest.go:15.13,18.4 1 1
letsgo/mytest/mytest.go:19.3,19.35 1 1
letsgo/mytest/mytest.go:19.35,23.4 2 1
letsgo/mytest/mytest.go:25.3,25.25 1 1
letsgo/mytest/mytest.go:27.2,27.12 1 1
letsgo/mytest/mytest.go:30.24,31.11 1 1
letsgo/mytest/mytest.go:31.11,33.3 1 1
letsgo/mytest/mytest.go:34.2,34.10 1 1
```

使用`go tool cover`就可以发现源码中的如下的`if`内的代码没有被执行，因此只需要构造一个空字符串（长度为0）作为测试用例，即可使代码覆盖率达到100%。

```Go
// mytest/mytest_test.go

func LengthOfLongestSubstring(s string) int {
    if len(s) == 0 {
        return 0
    }
    
    // ......
}
```

那么问题来了：

- 实际项目中测试几乎做不到100%，如何提高代码的测试用例？
- 代码覆盖率很高甚至达到了100%就能保证代码没有bug吗？

针对第二个问题，答案是**否定**的。举个很简单的例子(如下)，只要调用这个函数，代码的覆盖率就是100%，但是这个代码明显存在除0错误。虽然代码覆盖率达到了100%不能保证代码没有bug，但是代码覆盖率仍然是越高越好。

```Go
func MyDiv(a, b, c int) int {
    return a / (b - c)
}
```

有没有方法能提高代码覆盖率，发现移植代码的bug呢？答案是**肯定**的，那就是fuzzing！

##### FuzzXxx

**用途：**用于挖掘深藏的bug。

**思想**：变异+反馈（物竞天择，适者生存）；

**原理**：将被测对象看作一个有输入和输出的系统，对系统输入不同的测试用例，统计系统内部的代码执行情况和输出，并将有意义的输入作为种子进行变异再作为输入，不断重复。

![fuzzing](/images/posts/go-testing/fuzzing.png)
![fuzzing](/images/posts/go-testing/afl_fuzz.gif)


模糊测试的流行得益于模糊测试工具[AFL](https://lcamtuf.coredump.cx/afl/)，Go对fuzz的原生支持是从Go1.18开始的。使用Go的Fuzz并不难，只要准备初始测试用例即可。下面举个例子：

被测代码如下，还是求一个字符串的最长字串的实现，和前面的代码相比，这里调用了一个`bugInHere`函数，这个函数里面隐藏了一个bug。

```Go
// fuzz/fuzz.go
package fuzz

func bugInHere(s string) string {

    if len(s) > 8 {
        if s[0] == 'p' {
            if s[1] == 'a' {
                if s[2] == 'n' {
                    if s[3] == 'i' {
                        if s[4] == 'c' {
                            if s[5] == ' ' {
                                if s[6] == 'b' {
                                    if s[7] == 'u' {
                                        if s[8] == 'g' {
                                            return s[:10]
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }

    return s
}

func LengthOfLongestSubstring(s string) int {
    if len(s) == 0 {
        return 0
    }

    s = bugInHere(s) // panic

    // 哈希集合，记录每个字符是否出现过
    m := map[byte]int{}
    n := len(s)
    // 右指针，初始值为 -1，相当于我们在字符串的左边界的左侧，还没有开始移动
    rk, ans := -1, 0
    for i := 0; i < n; i++ {
        if i != 0 {
            // 左指针向右移动一格，移除一个字符
            delete(m, s[i-1])
        }
        for rk+1 < n && m[s[rk+1]] == 0 {
            // 不断地移动右指针
            m[s[rk+1]]++
            rk++
        }
        // 第 i 到 rk 个字符是一个极长的无重复字符子串
        ans = max(ans, rk-i+1)
    }

    return ans
}

func max(x, y int) int {
    if x < y {
        return y
    }
    return x
}
```

下面尝试fuzz上面的代码，fuzz测试代码如下，fuzz测试代码很简单，只需要两部即可：

1. 创建初始化测试用例并添加到`testing.F`中；
2. 调用`f.Fuzz`开始fuzz。

```Go
// fuzz/fuzz_test.go

func FuzzLengthOfLongestSubstring(f *testing.F) {
    // 创建初始测试用例（种子）
    tests := []string{""}
    for _, ts := range tests {
        f.Add(ts)
    }
    
    // 开始 fuzz
    f.Fuzz(func(t *testing.T, a string) {
        LengthOfLongestSubstring(a)
    })
}
```

执行下面的命令开始fuzz，从输出可以看到程序运行了32s触发了程序的bug，导致panic了。触发bug的输入在当前目录下的`testdata/fuzz/FuzzLenthOfLongestSubstring`目录下。

```Bash
# 执行fuzz命令
$ go test -fuzz=^FuzzLengthOfLongestSubstring$ -v
=== RUN   FuzzLengthOfLongestSubstringV2
=== RUN   FuzzLengthOfLongestSubstringV2/seed#0
=== RUN   FuzzLengthOfLongestSubstringV2/seed#1
=== RUN   FuzzLengthOfLongestSubstringV2/seed#2
=== RUN   FuzzLengthOfLongestSubstringV2/seed#3
=== RUN   FuzzLengthOfLongestSubstringV2/seed#4
=== RUN   FuzzLengthOfLongestSubstringV2/seed#5
--- PASS: FuzzLengthOfLongestSubstringV2 (0.00s)
    --- PASS: FuzzLengthOfLongestSubstringV2/seed#0 (0.00s)
    --- PASS: FuzzLengthOfLongestSubstringV2/seed#1 (0.00s)
    --- PASS: FuzzLengthOfLongestSubstringV2/seed#2 (0.00s)
    --- PASS: FuzzLengthOfLongestSubstringV2/seed#3 (0.00s)
    --- PASS: FuzzLengthOfLongestSubstringV2/seed#4 (0.00s)
    --- PASS: FuzzLengthOfLongestSubstringV2/seed#5 (0.00s)
=== RUN   FuzzLengthOfLongestSubstring
fuzz: elapsed: 0s, gathering baseline coverage: 0/22 completed
fuzz: elapsed: 0s, gathering baseline coverage: 22/22 completed, now fuzzing with 8 workers
fuzz: elapsed: 3s, execs: 220261 (73409/sec), new interesting: 0 (total: 22)
fuzz: elapsed: 6s, execs: 442482 (74025/sec), new interesting: 0 (total: 22)
fuzz: elapsed: 9s, execs: 675241 (77628/sec), new interesting: 0 (total: 22)
fuzz: elapsed: 12s, execs: 893474 (72688/sec), new interesting: 0 (total: 22)
fuzz: elapsed: 15s, execs: 1127763 (78167/sec), new interesting: 0 (total: 22)
fuzz: elapsed: 18s, execs: 1364191 (78804/sec), new interesting: 0 (total: 22)
fuzz: elapsed: 21s, execs: 1581999 (72609/sec), new interesting: 0 (total: 22)
fuzz: elapsed: 24s, execs: 1829063 (82274/sec), new interesting: 0 (total: 22)
fuzz: elapsed: 27s, execs: 2057236 (76138/sec), new interesting: 0 (total: 22)
fuzz: elapsed: 30s, execs: 2295217 (79327/sec), new interesting: 0 (total: 22)
fuzz: elapsed: 32s, execs: 2446099 (68996/sec), new interesting: 0 (total: 22)
--- FAIL: FuzzLengthOfLongestSubstring (32.19s)
    --- FAIL: FuzzLengthOfLongestSubstring (0.00s)
        testing.go:1504: panic: runtime error: slice bounds out of range [:10] with length 9
            goroutine 314427 [running]:
            runtime/debug.Stack()
                /usr/local/go/src/runtime/debug/stack.go:24 +0x9b
            testing.tRunner.func1()
                /usr/local/go/src/testing/testing.go:1504 +0x1ee
            panic({0x5e8300?, 0xc00001b9e0?})
                /usr/local/go/src/runtime/panic.go:914 +0x21f
            letsgo/fuzz.bugInHere(...)
                /home/nop/happycoding/wtfs/letsgo/fuzz/fuzz.go:15
            letsgo/fuzz.LengthOfLongestSubstring({0xc0091ae3c5, 0x9})
                /home/nop/happycoding/wtfs/letsgo/fuzz/fuzz.go:36 +0x885
            letsgo/fuzz.FuzzLengthOfLongestSubstring.func1(0x0?, {0xc0091ae3c5?, 0x0?})
                /home/nop/happycoding/wtfs/letsgo/fuzz/fuzz_test.go:38 +0x3b
            reflect.Value.call({0x5c37c0?, 0x607ad8?, 0x13?}, {0x5f83fc, 0x4}, {0xc0091b8150, 0x2, 0x2?})
                /usr/local/go/src/reflect/value.go:596 +0xce7
            reflect.Value.Call({0x5c37c0?, 0x607ad8?, 0x72f7c0?}, {0xc0091b8150?, 0x5f7a00?, 0x52220d?})
                /usr/local/go/src/reflect/value.go:380 +0xb9
            testing.(*F).Fuzz.func1.1(0x406420?)
                /usr/local/go/src/testing/fuzz.go:335 +0x347
            testing.tRunner(0xc0091b5520, 0xc0091ab3b0)
                /usr/local/go/src/testing/testing.go:1595 +0xff
            created by testing.(*F).Fuzz.func1 in goroutine 18
                /usr/local/go/src/testing/fuzz.go:322 +0x597
            
    
    Failing input written to testdata/fuzz/FuzzLengthOfLongestSubstring/87820115886ac14a
    To re-run:
    go test -run=FuzzLengthOfLongestSubstring/87820115886ac14a
=== NAME  
FAIL
exit status 1
FAIL    letsgo/fuzz     32.208s

# 触发bug的测试用例在 testdata/fuzz/FuzzLengthOfLongestSubstring 目录下。
$ tree
.
├── fuzz.go
├── fuzz_test.go
└── testdata
    └── fuzz
        └── FuzzLengthOfLongestSubstring
            └── 87820115886ac14a

# 触发程序bug的测试用例            
$ cat testdata/fuzz/FuzzLengthOfLongestSubstring/87820115886ac14a
go test fuzz v1
string("panic bug")
```

更多细节见：

- testing包文档；
- [Tutorial: Getting started with fuzzing](https://go.dev/doc/tutorial/fuzz)

##### BenchmarkXxx

**用途：**测试代码的性能（执行效率，内存分配、内存使用情况等）。

- **Inline vs Noinline；**

用benchmark来对比一下inline和noinline的性能差距有多大。默认情况下`go build`在编译源码时会对源码进行优化，其中一项就是将比较简单的函数inline，为了禁止函数编译器对函数做inline优化，可以使用*`//go:noinline`**。*

```Go
// fuzz/fuzz_test.go

func InlineMax(a, b int) int {
    if a > b {
        return a
    }

    return b
}

//go:noinline
func NonInlineMax(a, b int) int {
    if a > b {
        return a
    }

    return b
}
```

对上面两个函数做性能分析，测试代码如下：

```Go
// bench/bench_test.go
var Result int

func BenchmarkMaxInline(b *testing.B) {
    var r int
    for i := 0; i < b.N; i++ {
        r = InlineMax(-1, 1)
    }
    Result = r
}

func BenchmarkMaxNonInline(b *testing.B) {
    var r int
    for i := 0; i < b.N; i++ {
        r = NonInlineMax(-1, 1)
    }
    Result = r
}
```

执行上面的测试代码，输出了每个测试函数1s时间的执行情况：`BenchmarkMaxInline-8`执行了1000000000次，平均每次迭代耗时0.7953 ns，`BenchmarkMaxNonInline-8`执行了447639264次，平均每次迭代耗时3.841 ns。

```Bash
$ go test -bench=^BenchmarkMax -run=""
goos: linux
goarch: amd64
pkg: letsgo/bench
cpu: 12th Gen Intel(R) Core(TM) i7-12700H
BenchmarkMaxInline-8            1000000000               0.7953 ns/op
BenchmarkMaxNonInline-8         447639264                3.841 ns/op
PASS
ok      letsgo/bench    2.939s
```

- **Concat string；**

下面实现了三种拼接字符串的方法，具体实现方式分别为：使用字符串的`+`操作符拼接、使用`fmt.Sprintf`拼接和使用`strings.Join`拼接。

```Go
// bench/bench.go

func ConcatStringByOperator(sl []string) string {
    var s string
    for _, v := range sl {
        s += v
    }
    return s
}

func ConcatStringBySprintf(sl []string) string {
    var s string
    for _, v := range sl {
        s = fmt.Sprintf("%s%s", s, v)
    }
    return s
}

func ConcatStringByJoin(sl []string) string {
    return strings.Join(sl, "")
}
```

测试代码如下：

```Go
// bench/bench_test.go

var sl = []string{
    "Rob Pike ",
    "Robert Griesemer ",
    "Ken Thompson ",
}

func BenchmarkConcatStringByOperator(b *testing.B) {
    for n := 0; n < b.N; n++ {
        ConcatStringByOperator(sl)
    }
}

func BenchmarkConcatStringBySprintf(b *testing.B) {
    for n := 0; n < b.N; n++ {
        ConcatStringBySprintf(sl)
    }
}

func BenchmarkConcatStringByJoin(b *testing.B) {
    for n := 0; n < b.N; n++ {
        ConcatStringByJoin(sl)
    }
}
```

执行上面的测试代码，上面的字符串的拼接需额外的内存，可以在`go test`命令后添加`-benchmem`来查看内存分配情况。具体输出如下，以`BenchmarkConcatStringByOperator-8`输出为例：

1s内迭代了4614390次，平均每次迭代耗时240.6 ns，平均每次迭代申请了80bytes，平均每次迭代申请了2次内存。

```Bash
$ go test -run="" -bench=^BenchmarkConcat -benchmem
goos: linux
goarch: amd64
pkg: letsgo/bench
cpu: 12th Gen Intel(R) Core(TM) i7-12700H
BenchmarkConcatStringByOperator-8        4614390               240.6 ns/op            80 B/op          2 allocs/op
BenchmarkConcatStringBySprintf-8         1096496              1024 ns/op             176 B/op          8 allocs/op
BenchmarkConcatStringByJoin-8            7350830               152.6 ns/op            48 B/op          1 allocs/op
PASS
ok      letsgo/bench    4.872s
```

- **Start/Stop timer；**

go test默认给每个Benchmark函数分配1s时间，但是有些情况下需要先初始化，然后再进行测试，如果初始化比较耗时，可能会影响基准测试的测试结果。解决方法就是在执行耗时的操作之前停止计时，等到耗时操作执行完之后再继续计时。测试代码如下，用Init模拟一个比较耗时的操作。

```Go
// bench/bench_test.go

func Init() {
    time.Sleep(500 * time.Millisecond)
}

func BenchmarkWithoutInit(b *testing.B) {

    for n := 0; n < b.N; n++ {
        ConcatStringByJoin(sl)
    }
}

func BenchmarkWithInitV1(b *testing.B) {

    Init()

    for n := 0; n < b.N; n++ {
        ConcatStringByJoin(sl)
    }
}

func BenchmarkWithInitV2(b *testing.B) {
    b.StopTimer()
    Init()
    b.StartTimer()

    // b.ResetTimer()

    for n := 0; n < b.N; n++ {
        ConcatStringByJoin(sl)
    }
}
```

测试结果如下，可以看到`BenchmarkWithoutInit-8`和`BenchmarkWithInitV2-8`的平均每个迭代执行时间差不多。而未使用`StopTimer`和`StartTimer`的`BenchmarkWithInitV1-8 `的平均每个迭代的执行差不多时间是`BenchmarkWithInitV2-8`的2倍。

```Bash
$ go test -run="" -bench=^BenchmarkWith
goos: linux
goarch: amd64
pkg: letsgo/bench
cpu: 12th Gen Intel(R) Core(TM) i7-12700H
BenchmarkWithoutInit-8           8317146               154.0 ns/op
BenchmarkWithInitV1-8            4181710               272.9 ns/op
BenchmarkWithInitV2-8            7411749               153.3 ns/op
PASS
ok      letsgo/bench    18.368s
```

##### ExampleXxx

用于检测被测代码的输出是否符合预期，重点关注输出（**仅仅指stdout**）。

测试代码：

```Go
// example/example.go
package example

func HelloWrold() {
    fmt.Println("hello world")
}

func HelloWorld2() {
    fmt.Fprintf(os.Stderr, "hello world")
}
```

测试代码：

```Go
// example/example_test.go

package example_test

import (
    "letsgo/example"
)

func ExampleHelloWorld() {
    example.HelloWorld()
    // Output: hello world
}

func ExampleHelloWorld2() {
    example.HelloWorld2()
    // Output:
}
```

执行上面的测试代码：

```Bash
$ go test -run=^Example -v                                          ✔ ╱ nop@nop-vm  11:06:48 上午 
=== RUN   ExampleHelloWorld
--- PASS: ExampleHelloWorld (0.00s)
=== RUN   ExampleHelloWorld2
hello world--- PASS: ExampleHelloWorld2 (0.00s)
=== RUN   ExampleMap
--- PASS: ExampleMap (0.00s)
PASS
ok      letsgo/example  0.007s
```

另外，如果输出结果顺序是不确定的，可以用`Unordered output`，如：

```Go
// example/example.go

func PrintMap() {
    colors := map[string]string{
        "red":   "红",
        "green": "绿",
        "blue":  "蓝",
    }

    for k, v := range colors {
        fmt.Println(k, v)
    }
}

// example/example_test.go
package example_test

func ExamplePrintMap() {
    example.PrintMap()
    // Unordered output:
    // red 红
    // blue 蓝
    // green 绿
}
```

#### 其他

##### testdata

testdata是一个特殊的目录名，go在编译源码时会自动忽略这个文件中的文件（包括`*.go`文件），`testdata`一般用于存放测试需要的数据或者生成的数据。

### `go test` 

执行`go test`命令时，`go test`的处理步骤如下：

1. 生成测试程序的主函数（我们编写的测代码其实是这个测试主函数需要的数据）；
2. 调用`go build`编译测试程序，生成可执行文件；
3. 调用可执行文件，执行测试函数；
4. 删除步骤2生成的可执行文件；

#### go test用法

`go test`命令的用法，可以使用`go help test`命令查看。

```Go
$ go help test
usage: go test [build/test flags] [packages] [build/test flags & test binary flags]
......
The test binary also accepts flags that control execution of the test; these
flags are also accessible by 'go test'. See 'go help testflag' for details.

For more about build flags, see 'go help build'.
For more about specifying packages, see 'go help packages'.
```

从给出的 *usage* 可以看到`go test`的参数分为三类，分别如下：

- Build flags：即编译参数，具体参数见`go help build`；
- Test flags：`go test`所需的参数（如`-c`、`-json`），具体参数见`go help test`；
- Binary flags：测试程序所需的参数，具体参数见`go help testflag`。

接下来结合命令进行说明：

这条命令中的参数`-run=^TestLengthOfLongestSubstringV0$`和`-cover`属于binary flag。

```Go
$ go test -run=^TestLengthOfLongestSubstringV0$ -cover
```

这条命令中的`-c`参数愉test flag参数，作用是让`go test`直接生成可执行文件，但是不执行可执行文件。

```Go
$ go test -c
```

这条命令中的`-gcflags="-m -N -l"`属于build flags。

```Go
$ go test -gcflags="-m -N -l"
```

#### IDE自动执行 vs 手动执行

和IDE自动执行相比，手动执行更灵活：

- 可以一次执行多个（可以不是全部）测试函数；
- 可以指定输出信息（如输出格式`-json`，显示详情`-v`，输出代码覆盖率信息`-cover`，指定超时时间`-timeout`等）；