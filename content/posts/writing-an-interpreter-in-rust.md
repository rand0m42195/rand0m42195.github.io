+++
date = '2025-09-20T10:45:22+08:00'
draft = true
title = '用Rust实现Monkey解释器'
comments = true
tags = ["Rust", "Interpreter"]

+++

很多想学习或者了解编译器/解释器工作原理的人应该都听说过编译原理的龙书、虎书等，但是我猜大部分人和我一样——give up了。直到我前两年看到一本书[`Writing An Interpreter In Go`](https://interpreterbook.com/)，我一开始看这本书的书名，觉得有些吹牛。抱着试一试的态度，跟着这本书把代码敲了一遍，还真的就实现了一个`Monkey`语言的解释器。这本书带我解锁了学习编译原理的新姿势。刚好最近在学Rust，就想着用Rust实现Monkey的解释器，写完之后有三个比较重要的收获：

* 加深了对编译器/解释器的理解；
* 锻炼了Rust编程能力；
* 感受到了Go和Rust在编程方面的区别（如错误处理、非法值、抽象等）；

我准备写做一个关于*动手实现解释器*的系列博客，包括一下几个部分：

* 介绍Monkey语言，我们要实现的解释器就是专门用来解释Monkey语言的；
* 从源码到token——词法分析器（`Lexer`）：介绍Monkey语言的token分类有哪些，如何将Monkey语言的源码解析成token；
* 从token到抽象语法树（AST）——词法分析器：介绍Monkey语言的语法规则，并展示如何将Lexer解析得到的token按照语法规则生成AST；
* 从AST到解释执行——Evaluator：介绍如何将实现解释器的终极目标，即按照源码的意图进行执行，输出结果。



下面就开始我们的第一部分，介绍Monkey语言。





这篇博客介绍一个我用Rust写的小项目——[`Writing An Interpreter In Rust`](https://github.com/rand0m42195/monkey-rs)。这个项目是*借鉴*（或者说是抄袭😅）[`Writing An Interpreter In Go`](https://interpreterbook.com/)，原书是用Go实现了一个简单的解释器来解释、运行`Monkey`语言，而我的这个项目就是用Rust实现了一个解释和运行Monkey语言的解释器。

下面来说说Monkey语言（你很可能没听过，因为这根本就不是一个主流的编程语言😂），`Monkey`语言是`Writing An Interpreter In Go`作者自创的一个语言，但是它的语法和现代的主流编程语言很类似，所以解析`Monkey`。具体来说`Monkey`语言具有以下特性：

* 借鉴了主流编程语言的语法，如`C`, `JavaScript`等；
* 变量绑定，如 `let x = 1`；
* 支持多种数据类型：整型、布尔型、字符串、数组、甚至还支持Map；
* 算术表达式；
* 内置函数，如`len`、`append`等；
* 函数是first class；

下面以递归求解斐波那契数列为例，展示Monkey的写法：
```rust
>> let n = 10;
>> let fib = fn(x) { if (x < 3) { return x; } else { return fib(x - 1) + fib(x - 2); } }
>> fib(n)
89
```

这段语法展示了Monkey语言的变量定义，函数定义，if/else语句，函数调用等，这也展示了Monkey竟然还支持递归调用！

后续将一步一步介绍如何实现一个能解释Monkey语言的解释器。在开始工作之前，我们先把目标明确了：我们要实现的是一个**解释器**，而不是**编译器**，编译器和解释器有什么区别呢？

编译器（Compiler）和解释器（Interpreter）都是将高级语言程序转化为计算机可以理解的形式（通常是机器码或中间码）的工具，但它们的工作方式不同，导致了使用体验和性能上的差异。

