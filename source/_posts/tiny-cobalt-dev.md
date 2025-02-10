---
title: Tiny Cobalt 开发日志
date: 2025-02-10 09:49:30
tags:
 - 编译原理
 - Tiny Cobalt
categories:
 - Compiler
---

这里是[Tiny Cobalt](https://github.com/AI1379/tiny-cobalt)的开发日志。尽量持续更新。

<!--more-->

## 初期工作：框架的搭建

两年前和 [AndyShen](https://github.com/AndyShen2006) 最初决定写自己的编译器的时候，打算的技术方案是 CMake 作为构建系统，标准定在 C++17 。24年12月初的时候我决定重新开工，此时各大编译器对于 C++20 的支持已经逐渐完善了，此外 [xmake](https://xmake.io) 构建系统也已经比当年更加完善，因此重新启动 Cobalt 项目的时候，我决定采用 xmake 作为编译系统。这带来的一大好处是便捷的包管理，另一个好处是可以避免复杂的 cmake 脚本，可以注意力集中在代码上而不用折腾太多依赖。

最开始，我们打算基于 `std::variant` 作为运行时多态的实现，此外也引入了微软的 [`proxy`](https://github.com/microsoft/proxy) 库作为备用方案，目的是为了尝试着探索 C++ 传统的继承+虚函数方案以外的其他多态方案。然而，在实现的过程中，我们发现 `std::variant` 并不能完全实现我们需要的多态，此外它与 `proxy` 的兼容性也一般。因此，我们决定全面改用 `proxy` 。在 [Utility.h](https://github.com/AI1379/tiny-cobalt/blob/main/include/Common/Utility.h) 中可以看到一些早期代码的残留。后来在开发过程中发现了 `proxy` 方案的一些问题，此处暂且不表。

在架构上，我们采用了传统的 `Lexer/Parser -> Semantic Analyzer -> Backend` 的方案，词法分析和语法分析使用 `flex` 和 `bison` 生成，后端则直接使用 `LLVM` 作为后端。目前也有计划将手工实现词法分析和语法分析。每一部分之间都是解耦的，以便模块化。测试方面，采用 `googletest` 作为测试框架。

## Cobalt 的语法设计

总的来讲，Cobalt 的语法设计是类似 C++ 的，此外也有可能加入一些 Haskell 和 Rust 的语法。Cobalt 与 C++ 的语法差别主要是在类型系统上。我们抛弃了 C 中传统的指针和数组定义方式，转而使用类似 C++ 模板的方式来实现，一方面是为了编译器相对好写一些，另一方面是为了可读性。

此外，我们还要求类型以大写字母开头，具体原因是为了便于语法分析。

另外，在类型转换方面，我们的设计也是 C++ 的方案，而抛弃了 C 中的语法。

## AST 的设计

### 基础结构

这部分的代码主要在 [ASTNode.h](https://github.com/AI1379/tiny-cobalt/blob/main/include/AST/ASTNode.h) 和 [ASTVisitor.h](https://github.com/AI1379/tiny-cobalt/blob/main/include/AST/ASTVisitor.h)

首先我们参考 `libstdc++` 的源码实现了 C++23 的 `std::generator`，然后通过 `co_yield` 的方式把树的遍历抽象出来，以便 `ASTVisitor` 的实现。同时我们也引入了 `nlohmann/json` 库，以便实现 AST 的序列化，便于调试。

此外，我们还实现了一个 [`ASTBuilder`](https://github.com/AI1379/tiny-cobalt/blob/main/include/AST/ASTBuilder.h)，使得我们可以在 C++ 代码中直接以一种类似 `json` 的格式构建 AST 。

### 具体实现

这一部分比较机械化。我们设计了以下 AST 节点：

- `ASTRootNode`
- `ExprNode`
  - `ConstExprNode`
  - `VariableNode`
  - `BinaryNode`
  - `UnaryNode`
  - `MultiaryNode`
  - `CastNode`
  - `ConditionNode`
  - `MemberNode`
- `StmtNode`
  - `IfNode`
  - `WhileNode`
  - `ForNode`
  - `ReturnNode`
  - `BlockNode`
  - `BreakNode`
  - `ContinueNode`
  - `VariableDefNode`
  - `FuncDefNode`
  - `StructDefNode`
  - `AliasDefNode`
  - `ExprStmtNode`
  - `EmptyStmtNode`
- `TypeNode`
  - `SimpleTypeNode`
  - `FuncTypeNode`
  - `ComplexTypeNode`

基本上就是照抄了 C++ 中的实现。我们为三类节点分别实现了一个对应的 `proxy` 定义了相关 API，方便后续扩展。

### `ASTVisitor` 和 `ASTVisitorMiddleware`

这部分的代码在 [ASTVisitor.h](https://github.com/AI1379/tiny-cobalt/blob/main/include/AST/ASTVisitor.h) 中。

首先实现了一个模板 `BaseASTVisitor` ，实现了 AST 的遍历。一开始这个模板设计为一个 CRTP 的基类模板，后来改为传入一个 `Middleware` 作为模板参数，`Visitor` 负责调用。目前看来，这似乎是一个冗余设计，因此后期可能会重构。

此后定义了 `BaseASTVisitorMiddleware` ，利用 CRTP 自动生成 `Visitor` 所需要的接口；同时定义了对应的 `proxy` ，不过似乎没有用上。

在 `Visitor` 的设计中，我们定义了一个名为 `VisitorState` 的枚举，用于在中间件中方便地控制 `ASTVisitor` 的状态，实现在中间件中跳出整个遍历流程。

## LexerParser 的实现

Bison 对 C++ 的支持颇为不错，相比之下 flex 对 C++ 的支持就相对较差。

### 语法分析部分

具体可见 [Parser.yy](https://github.com/AI1379/tiny-cobalt/blob/main/src/LexerParser/Parser.yy)。

在 `parser` 的实现过程中，我们遇到了早期 C++ 编译器实现时遇到的许多问题，例如模板尖括号与大于小于号的区分问题，以及嵌套模板与右移运算符的区分问题。为了区分右移运算符，我们采取了早期 C++ 编译器要求的方案，即要求嵌套模板的两个右尖括号中插入空格。而比较运算符的冲突则比较难办。

查阅 `clang` 和 `gcc` 的源码，可以发现它们将一部分的语义分析流程提前到了语法分析的时候，在建立 AST 之前就建立符号表，从而将模板与变量区分开，解决这个问题。然而，这与我们当前的设计是矛盾的。我们目前的解决方案是强制要求所有类型（除了几个基本类型）与模板都以大写字母开头，而变量和函数是小写字母开头，从而将 `typename` 与 `identifier` 在词法分析阶段就区分开来。这只是一个折中的方案，然而似乎找不到更好的解决措施。

此外，在实现逗号表达式的时候也遇到了很多 `Shift/Reduce Conflict` 和 `Reduce/Reduce Conflict`，因此在目前的代码里还不支持逗号表达式。

### 词法分析部分

这部分相对简单，参考 bison 的样例很容易实现。

### Driver 的设计

在 [Parser.h](https://github.com/AI1379/tiny-cobalt/blob/main/include/LexerParser/Parser.h) 中我们定义了一个模板类，接受不同的 Driver 作为前端。由于 Lexer 由 flex 生成，而 flex 目前对 C++的支持相当糟糕，因此我们把这部分糟糕的代码隔离出来放在 [YaccDriver.h](https://github.com/AI1379/tiny-cobalt/blob/main/src/LexerParser/YaccDriver.h) 和 [YaccDriver.cpp](https://github.com/AI1379/tiny-cobalt/blob/main/src/LexerParser/YaccDriver.cpp) 中。这里参考了 StackOverflow 上的一篇文章。

此后有计划完善这部分的设计，然后手工实现完整的词法分析与语法分析。

## 语义分析

这一部分是目前还在进行的工作，主要有以下板块。

### `DeclMatcher`

这一部分是在词法分析中最先进行的。出于性能考虑，我们采用了 C++23 的 `std::flat_map`。由于目前标准库对其支持并不完善，因此我们自己实现了一个。

我们在 `DeclMatcher` 中维护了四个 `Scope`，分别存储不同类型的符号。

### `TypeAnalyzer`

在完成符号匹配后，才能进行类型的分析。这一部分也还没有完全实现。
