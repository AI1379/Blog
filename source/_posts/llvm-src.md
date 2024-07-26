---
title: LLVM源码阅读 笔记
date: 2023-09-3 16:43:27
mathjax: true
tags:
 - 编译原理
categories:
 - Compiler
---

本文根据LLVM 15.0.7版本代码更新

# clang

## 主入口

`clang` 目录下貌似没有找到一个`main`函数，推测是在编译时生成。目前看来，`clang`的主入口是在`clang/tools/driver/driver.cpp`下面的`clang_main`函数。这里很多乱七八糟的边界处理，并不重要，一路可以追溯到同一个目录下面的`cc1_main.cpp`里的`cc1_main`函数。

在`cc1_main`里同样有很多异常处理。在函数开头构造了一个`std::unique_ptr<CompilerInstance> Clang(new CompilerInstance());`，这个东西是整个编译流程的核心。接下来基本上是在处理输入参数和判断异常，直到`250`行`ExecuteCompilerInvocation(Clang.get())`。

这个函数定义在`clang/include/clang/FrontendTool/Utils.h`里，实现在`clang/lib/FrontendTool/ExecuteCompilerInvocation.cpp`里。这里很多代码也是在处理各种边界条件以及加载插件之类的事情，把参数放在了一个叫做`Act`的东西里面，然后`266`行调用`Clang->ExecuteAction(*Act)`启动主程序。这后面就是`CompilerInstance`的工作了

## `CompilerInstance`

`clang/include/clang/Frontend/CompilerInstance.h`和`clang/lib/Frontend/CompilerInstance.cpp`。前面说的`ExecuteAction`就在这两个文件中定义。

实现是在`cpp`文件的`989`行，其中`1028`行开始遍历输入文件。这个循环如下：

```cpp
// LLVM clang/lib/Frontend/CompilerInstance.cpp Line 1028
// clang::CompilerInstance::ExecuteAction
  for (const FrontendInputFile &FIF : getFrontendOpts().Inputs) {
    // Reset the ID tables if we are reusing the SourceManager and parsing
    // regular files.
    if (hasSourceManager() && !Act.isModelParsingAction())
      getSourceManager().clearIDTables();

    if (Act.BeginSourceFile(*this, FIF)) {
      if (llvm::Error Err = Act.Execute()) {
        consumeError(std::move(Err)); // FIXME this drops errors on the floor.
      }
      Act.EndSourceFile();
    }
  }
```

~~甚至有个fixme~~

显然这里重要的代码是第二个`if`。它调用了`Act`里的`Execute`执行。这几个`if`主要是用来处理错误的。

然后我们追回去看`Act`。这个`Act`就是在调用`ExecuteAction`时传入的参数。找到它是指向一个`FrontendAction`的智能指针。这个类是一个抽象类，它有很多具体的实现。

再往前追到`ExecuteCompilerInvocation`，`FrontendAction`的类是在这里构造的。在这里调用了一个`CreateFrontendAction`和`CreateFrontendBaseAction`，构造了许许多多的`FrontendAction`。

接着可以找到`clang/include/clang/Frontend/FrontendOptions.h`，在这里有一个十分显眼的`enum ActionKind`。从这里的注释可以看到，通常我们在编译的时候调用的是`EmitObj`这条路。在`CreateFrontendBaseAction`中我们可以发现，`Emit`为前缀的几条路都是从`CodeGenAction`继承而来的。从这里可以找到`CodeGen`这个文件夹，这里就是`clang`在编译时核心工作所在的地方。

## `CodeGen`

首先，`CodeGenAction`继承自`ASTFrontendAction`，然后`ASTFrontendAction`则继承自`FrontendAction`。在`FrontendAction::Execute`中可以看到，它主要工作就是调用了对应子类的`ExecuteAction`。所以我们找到`CodeGenAction`的`ExecuteAction`。从这里看到，如果输入文件不是LLVM IR，那么将会调用`ASTFrontendAction::ExecuteAction`。

追过去看，代码是这样的：

```cpp
// LLVM clang/lib/Frontend/FrontendAction.cpp Line 1125
void ASTFrontendAction::ExecuteAction() {
  CompilerInstance &CI = getCompilerInstance();
  if (!CI.hasPreprocessor())
    return;

  // FIXME: Move the truncation aspect of this into Sema, we delayed this till
  // here so the source manager would be initialized.
  if (hasCodeCompletionSupport() &&
      !CI.getFrontendOpts().CodeCompletionAt.FileName.empty())
    CI.createCodeCompletionConsumer();

  // Use a code completion consumer?
  CodeCompleteConsumer *CompletionConsumer = nullptr;
  if (CI.hasCodeCompletionConsumer())
    CompletionConsumer = &CI.getCodeCompletionConsumer();

  if (!CI.hasSema())
    CI.createSema(getTranslationUnitKind(), CompletionConsumer);

  ParseAST(CI.getSema(), CI.getFrontendOpts().ShowStats,
           CI.getFrontendOpts().SkipFunctionBodies);
}
```

~~依然有个fixme~~

重点在最后四行：首先判断有没有建好的AST，没有就建树也就是`createSema`，然后解析AST。

`createSema`相当简洁：

```cpp
void CompilerInstance::createSema(TranslationUnitKind TUKind,
                                  CodeCompleteConsumer *CompletionConsumer) {
  TheSema.reset(new Sema(getPreprocessor(), getASTContext(), getASTConsumer(),
                         TUKind, CompletionConsumer));
  // Attach the external sema source if there is any.
  if (ExternalSemaSrc) {
    TheSema->addExternalSource(ExternalSemaSrc.get());
    ExternalSemaSrc->InitializeSema(*TheSema);
  }
}
```

换句话说，就是构造了一个新的`Sema`放进了`CompilerInstance`的`TheSema`里面。

`Sema`的构造函数在`clang/lib/Sema/Sema.cpp`里面，相当冗长。事实上这个得归功于C++标准。然而实际上他没有干什么实际的事情，实事是在`ParseAST`里面做的

`ParseAST`有两个实现，都在`clang/lib/Parse/ParseAST.cpp`里面，分别在`99`和`114`行。这里调用的是`114`行的实现。

## `ParseAST`

首先注意：此时我们还没有进行任何有实际意义的编译工作。事实上，这些工作是在这里进行的。

完整代码如下：

```cpp
// LLVM clang/lib/Parse/ParseAST.cpp Line 114
void clang::ParseAST(Sema &S, bool PrintStats, bool SkipFunctionBodies) {
  // Collect global stats on Decls/Stmts (until we have a module streamer).
  if (PrintStats) {
    Decl::EnableStatistics();
    Stmt::EnableStatistics();
  }

  // Also turn on collection of stats inside of the Sema object.
  bool OldCollectStats = PrintStats;
  std::swap(OldCollectStats, S.CollectStats);

  // Initialize the template instantiation observer chain.
  // FIXME: See note on "finalize" below.
  initialize(S.TemplateInstCallbacks, S);

  ASTConsumer *Consumer = &S.getASTConsumer();

  std::unique_ptr<Parser> ParseOP(
      new Parser(S.getPreprocessor(), S, SkipFunctionBodies));
  Parser &P = *ParseOP.get();

  llvm::CrashRecoveryContextCleanupRegistrar<const void, ResetStackCleanup>
      CleanupPrettyStack(llvm::SavePrettyStackState());
  PrettyStackTraceParserEntry CrashInfo(P);

  // Recover resources if we crash before exiting this method.
  llvm::CrashRecoveryContextCleanupRegistrar<Parser>
    CleanupParser(ParseOP.get());

  S.getPreprocessor().EnterMainSourceFile();
  ExternalASTSource *External = S.getASTContext().getExternalSource();
  if (External)
    External->StartTranslationUnit(Consumer);

  // If a PCH through header is specified that does not have an include in
  // the source, or a PCH is being created with #pragma hdrstop with nothing
  // after the pragma, there won't be any tokens or a Lexer.
  bool HaveLexer = S.getPreprocessor().getCurrentLexer();

  if (HaveLexer) {
    llvm::TimeTraceScope TimeScope("Frontend");
    P.Initialize();
    Parser::DeclGroupPtrTy ADecl;
    Sema::ModuleImportState ImportState;
    EnterExpressionEvaluationContext PotentiallyEvaluated(
        S, Sema::ExpressionEvaluationContext::PotentiallyEvaluated);

    for (bool AtEOF = P.ParseFirstTopLevelDecl(ADecl, ImportState); !AtEOF;
         AtEOF = P.ParseTopLevelDecl(ADecl, ImportState)) {
      // If we got a null return and something *was* parsed, ignore it.  This
      // is due to a top-level semicolon, an action override, or a parse error
      // skipping something.
      if (ADecl && !Consumer->HandleTopLevelDecl(ADecl.get()))
        return;
    }
  }

  // Process any TopLevelDecls generated by #pragma weak.
  for (Decl *D : S.WeakTopLevelDecls())
    Consumer->HandleTopLevelDecl(DeclGroupRef(D));

  // For C++20 modules, the codegen for module initializers needs to be altered
  // and to be able to use a name based on the module name.

  // At this point, we should know if we are building a non-header C++20 module.
  if (S.getLangOpts().CPlusPlusModules && !S.getLangOpts().IsHeaderFile &&
      !S.getLangOpts().CurrentModule.empty()) {
    // If we are building the module from source, then the top level module
    // will be here.
    Module *CodegenModule = S.getCurrentModule();
    bool Interface = true;
    if (CodegenModule)
      // We only use module initializers for interfaces (including partition
      // implementation units).
      Interface = S.currentModuleIsInterface();
    else
      // If we are building the module from a PCM file, then the module can be
      // found here.
      CodegenModule = S.getPreprocessor().getCurrentModule();
    // If neither. then ....
    assert(CodegenModule && "codegen for a module, but don't know which?");
    if (Interface)
      S.getASTContext().setModuleForCodeGen(CodegenModule);
  }
  Consumer->HandleTranslationUnit(S.getASTContext());

  // Finalize the template instantiation observer chain.
  // FIXME: This (and init.) should be done in the Sema class, but because
  // Sema does not have a reliable "Finalize" function (it has a
  // destructor, but it is not guaranteed to be called ("-disable-free")).
  // So, do the initialization above and do the finalization here:
  finalize(S.TemplateInstCallbacks, S);

  std::swap(OldCollectStats, S.CollectStats);
  if (PrintStats) {
    llvm::errs() << "\nSTATISTICS:\n";
    if (HaveLexer) P.getActions().PrintStats();
    S.getASTContext().PrintStats();
    Decl::PrintStats();
    Stmt::PrintStats();
    Consumer->PrintStats();
  }
}
```

代码非常冗长。
