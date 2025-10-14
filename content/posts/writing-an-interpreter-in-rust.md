+++
 title = "Writing An Interpreter In Rust (1)"
 date = 2025-09-26
 draft = false
+++

# Writing An Interpreter In Rust (1)

最近在读 [*Writing An Interpreter In Go*](https://interpreterbook.com/)。
 书本身挺有意思的，它带着你一步一步写一个叫 [**Monkey**](https://monkeylang.org/) 的小语言。
 我边看边想：既然我现在正在学 Rust，那干脆用 Rust 来实现一遍吧，当作练习。

于是就有了这个项目：
 👉 [GitHub: rand0m42195/monkey-rs](https://github.com/rand0m42195/monkey-rs)

------

## 为什么做这个事

- 刚好最近在学rust，像用rust找点事情干；
- 解释器一直是我感兴趣的话题，但之前只是零散看过点原理；
- 用一个小项目把 Rust 和解释器放在一起，会有意思。

------

## 关于 Monkey 语言

Monkey 是书里的示例语言。它很小，但该有的都有：

- `let` 绑定
- 表达式运算
- 条件分支
- 函数（而且是一等公民，可以闭包）
- 后面还有数组、哈希，甚至宏

举个简单例子：

```monkey
let add = fn(a, b) { a + b; };
add(2, 3);  // => 5
```

------

## 我打算做什么

我的目标就是：用 Rust 写出一个能跑的 Monkey 解释器，最后能在 REPL 里玩一玩：

```
>> let a = 10;
>> let b = 20;
>> a + b;
30
```

解释器的基本流程大概是这样：

```
源码 → 词法分析 (Lexer) → 语法分析 (Parser)
    → 抽象语法树 (AST) → 求值 (Evaluator) → 结果
```

------

## 系列会写什么

这篇算是开个头，后面我会陆续写点笔记，大概是：

1. lexer 的实现
2. parser 的实现
3. evaluator（求值器）
4. macro
5. 最后简单总结一下

------

