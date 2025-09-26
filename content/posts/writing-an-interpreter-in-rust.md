+++
date = '2025-09-20T10:45:22+08:00'
draft = false
title = '用Rust实现Monkey解释器（一）'
comments = true
tags = ["Rust", "Interpreter"]

+++

## 背景

我一直对编译原理挺感兴趣，觉得编译器很神奇，所以我曾经也尝试看过编译原理的龙书，未果，也找了国内外一些高校的编译原理视频教程，最终还是没看完。感觉无论是书本还是学校的教学视频，他们都太注重理论，而很少涉及具体的代码。大概两年前，我看到了一本书——[`《Writing An Interpreter In Go》`](https://interpreterbook.com/)，看书名感觉作者在吹牛，但是还是忍不住抱着试一试的态度看了，并且跟着动手把书中的代码敲了一遍。结果还真的如书名所说，实现了一个像模像样的解释器。最让我感慨的是这本书只是用了Go语言内置的数据结构，另外只用到了标准库里的很少功能，可以说是完全从0到1自制解释器。

刚好最近学在学Rust，所以就想着用Rust把书中介绍的解释器实现一遍。最后花了几天时间用Rust写了一遍，对我个人而言还是有不少收获的：

* 加深了对编译器/解释器的理解；
* 锻炼了自己的Rust编程能力；
* 体会到了Rust和Go两种语言之间的差别，如错误处理、抽象、内置数据结构等；

为了趁热打铁，我打算来写一个关于自制解释器的系列博客，就叫做`用Rust实现Monkey解释器`。计划分为几个部分：

* 整体介绍，包括这个系列博客的产生背景、解释器的效果介绍；
* 从源码到Token：介绍词法分析器Lexer的实现；
* 从Token到AST：介绍语法分析器Parser的实现；
* 从AST到代码结果：介绍求职其Evaluator的实现；





## 解释器

首先要明确的是我们要实现的是一个**解释器**，而非~~编译器~~。关于解释器和编译器的区别我们这里就粗略的理解为：解释器读取源码然后解释并执行，输出结果；而编译器读取源码，将源码翻译为机器指令（即我们常见的可执行文件），但是不会执行源代码。

每种语言都有自己的解释器/编译器，这里我们要实现的是一个叫做[`Monkey`](https://monkeylang.org/)的语言的解释器。`Monkey`语言并不是一个可用于生产环境的编程语言，它只是一个用来学习解释器/编译器的语言。下面就介绍一下`Monkey`长什么样，它有哪些特性。

Monkey语言的语法和现代编程语言（如JavaScript、rust）很像

```rust
// Integers & arithmetic expressions...
let version = 1 + (50 / 2) - (8 * 3);

// ... and strings
let name = "The Monkey programming language";

// ... booleans
let isMonkeyFastNow = true;

// ... arrays & hash maps
let people = [{"name": "Anna", "age": 24}, {"name": "Bob", "age": 99}];
```











