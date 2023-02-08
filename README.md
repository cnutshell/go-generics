# Golang Generics

[TOC]

## 1. golang 泛型介绍

长期以来，泛型一直是 golang 社区呼声较高的特性之一，直到 golang 1.18 泛型这个特性终于得到了支持。

什么是泛型？泛型 (generics) 用来编写模版代码以适应不同的类型。

在工程实践中，我们经常碰到这样的情况：同一份逻辑可以应用到多个不同的类型。例如排序算法，不管是 int 类型还是 uint 类型的数据集，都可以应用相同的排序算法；例如链表，不管链表节点上保存什么类型的数据，链表本身的操作都是一致的。

对于类似的情况，同样的算法或者逻辑，难道要为每种类型都实现一遍吗？也不是不可以。至少 golang 1.18 之前，golang 生态里已经有很多成熟的大型项目是在没有泛型的情况开发的。

只是没有泛型，代码会变得冗余，代码可读性和可维护性也会受到影响。泛型就是为了解决这样的问题：以通用类型来编写模版，涉及某个具体类型的调用时，编译器会生成一份类型特定的代码，这个过程称为 [monomorphization](https://en.wikipedia.org/wiki/Monomorphization)。

> 通用类型通常要满足一定的约束，取决于模版代码中通用类型的使用方式。对于排序代码来说，通用类型至少需要是可比较的。

### 1.1 golang 泛型设计

golang 一直以来的设计哲学都是 `simple and clean`，对于新特性的增加，它保持很谨慎的态度。

这样的坚持带来的好处是语言简单，学习成本不高，没有使用过 golang 的开发人员，熟悉一周语法就可以上手。

并且维持了极好的向前兼容性：基于 golang 早期版本写的代码，几乎不用更改就能在 golang 新版本下运行。

这一设计哲学导致 golang 直到 1.18 才引入泛型，并且对于 golang 泛型的设计选择也产生了决定性的影响。

我们先看看 C++ 和 Rust 的泛型设计：它们使用完全的单态化，即为使用模版代码的每一个具体类型生成一份代码。这种做法带来的结果是更长的编译时间以及更大的二进制文件，好处是编译器可以很“从容的”为每个类型的实例化代码作出优化。

泛型的设计选择其实是一种 tradeoff：在编译时间、二进制文件大小和代码执行效率之间的权衡。

golang 的泛型实现选择了一种折衷的方案：部分单态化。用 golang 自创的术语来说是 `GCShape stenciling with Dictionaries`。详细的设计在 golang proposal 库中可以找到：[Generics implementation - GC Shape Stenciling](https://github.com/golang/proposal/blob/master/design/generics-implementation-gcshape.md)。

简单来说，golang 为了避免实例化代码的膨胀，没有选择为每个类型生成具体代码，而是选择了一个更大的范围：基于 `GCShape` 来生成具体代码。`GCShape` 是特定于 golang 泛型的概念，如果两个类型的实际类型相同或者都是指针类型，那么它们拥有相同的 gcshape。什么意思呢？类型 uint32 和 float32 拥有不同的 gcshape，golang 编译器会为它们生成两份代码；所有的指针拥有相同的 gcshape，golang 编译器会为它们生成同一份代码，而不管指针所指向的对象是否拥有相同的 gcshape。

对于 *time.Time 和 *bytes.Buffer，它们有相同的 gcshape，但是它们的方法不同，不能简单的包含在 gcshape 中。为了向实例化后的代码传递类型参数 (type parameter) 的信息，golang 编译器额外引入了一个静态 dictionary，用来承载类型转换以及调用方法时所需的信息（细节可以参考：[Generics implementation - GC Shape Stenciling](https://github.com/golang/proposal/blob/master/design/generics-implementation-gcshape.md) ）。

将所有的指针类型按同一个 gcshape 来处理，这样可以极大地控制实例化后代码的规模。但缺点是编译器无法执行内联优化，进而内联后允许的进一步优化也变得不可能。

正如一位开发者在 [golang issue 50182](https://github.com/golang/go/issues/50182) 中 [回复](https://github.com/golang/go/issues/50182#issuecomment-1001258256) 的那样：用泛型重写代码后，代码运行有可能变快，也可能变慢。

> The **goal of generics** is not to provide faster execution time. The goal is to make it easier to write certain kinds of code with less repetitive boilerplate and more compile-time type checking. **Rewriting code to use generics may sometimes make it faster. It may sometimes make it slower** (as in this example). While we will continue to improve the compiler balancing the needs of fast builds and fast runtimes, it will never be the case that generic code will always be faster than non-generic code. That is not a goal.

### 1.2 golang 泛型演进

这节是关于 golang 泛型演进的不完整介绍。

基于 golang 1.18 版本的泛型实现，来自 PlanetScale 的性能工程师 [Vicent Martí](https://twitter.com/vmg)  在博客 [Generics can make your Go code slower](https://planetscale.com/blog/generics-can-make-your-go-code-slower) 中全面的分析了泛型的代码性能。这个 blog 因其透彻深入并且相对全面的分析得到了广泛的传播，在 Reddit 和 Hacker News 上都可以看到因这篇博客引发的关于 golang 泛型“性能”的讨论。

博客的最后，Vicent Martí 给出了关于使用泛型的一些建议。如果采用的是 golang 1.18 版本，不论这篇博客前面的分析部分对你而言是否“艰深晦涩”，至少文章最后给出这些建议是需要认真理解的。抛开代码性能不谈，golang 1.18 引入泛型，这项工作本身就是一个巨大的进步。

golang 1.18 引入泛型之后，开发人员一直在对其进行优化，例如来自 [James Roberts](https://github.com/jproberts) 的提交 [stop interface conversions for generic method calls from allocating](https://github.com/jproberts/go/commit/9d622d8168a8470536a6d6b9e0c98143c7b07529) 针对向泛型函数传入接口类型参数的情况进行了优化，避免了因向接口类型转换到导致的不必要动态内存分配。

golang 1.19 中包含了对泛型的优化，按照 [官方 release 文档](https://go.dev/blog/go1.19) 中所提到的数据：相比 golang 1.18 某些泛型程序的性能获得了 20% 的提升。不过该 release 文档没有介绍数据的来源。

实际上，在 Issue [#54238: Go 1.19 might make generic types slower](https://github.com/golang/go/issues/54238) 中，[有人员提到](https://github.com/golang/go/issues/54238#issuecomment-1204278680)：为了解决某些 bug，golang 1.19 特意禁用了内联优化。这会导致某些泛型相关的代码出现性能降级，问题已经在 golang 1.20 中得到修复。不过 1.20 版本中有关的更新并不会“反哺”到 1.19 版本中。从另一个 issue [#56280: functions with type parameters cannot inline multiple levels deep across packages](https://github.com/golang/go/issues/56280) 下面 [Keith Randall 的回复](https://github.com/golang/go/issues/56280#issuecomment-1313876001) 中，我们了解到：通常情况下关于性能的修复不会追加到旧版本上。Issue #56280 中提及的问题是：在 golang 1.19 中，如果一个导入的、非泛型的函数调用了泛型函数，编译器不会对这个泛型函数执行可能的内联优化。同 issue #54238 一样，这个问题的修复也放在了 golang 1.20 中。

Issue [#57505: cmd/compile: performance regression in 1.20](https://github.com/golang/go/issues/57505) 是 golang 1.20 中与内联优化有关的 generics 性能问题。这个问题与编译器 `unified IR` 的实现有关，[Matthew Dempsky](https://github.com/mdempsky) 给出了两个 workarounds，问题最终的解决或者编译器给出有效的提示可能要等到 golang 1.21 版本。

## 2. golang 泛型与性能

### 2.1 性能的定义

如果不限定范围，“性能”这个概念本身可以覆盖比较大的范畴：

1. 程序运行是否效率
2. 程序开发是否高效
3. 程序维护是否容易

如果单纯追求程序运行效率，用汇编来实现应用无疑是最效率的。但是当今软件世界，大部分应用都是高级语言实现。在软件的生命周期中，开发效率和维护效率，和程序运行效率一样，都是不容忽视的方面。

采用 golang 泛型可以减少对样板代码的 copy-paste，提高开发效率并有利于提高代码的可读性，这是软件工程方面的收益。

回到程序运行效率：采用泛型后，代码性能可能会提高，例如 [Google’s B-Tree 的实现采用泛型后](https://www.scylladb.com/2022/04/27/shaving-40-off-googles-b-tree-implementation-with-go-generics/)，有了 40% 的提升。但是，代码性能也可能降低，例如 [Generics can make your Go code slower](https://planetscale.com/blog/generics-can-make-your-go-code-slower) 中分析的那样，因为 golang 泛型不是完全的单态化实现，导致代码实例化之后，相比没有采用泛型的代码，失去了内联优化以及内联后进一步优化的机会。

所以关于 golang 泛型对于代码性能的影响，需要视具体情况而定。

### 2.2 golang 泛型 hints

下面结合调研情况，给出 golang 泛型使用的一些简单提示：

- 结构体中尽量使用泛型，能用泛型实现多态就尽量不用接口类型
- 对性能敏感的代码，如果使用泛型会导致代码性能降级，不必采用泛型，维持原来的代码即可
- 对性能敏感的代码，建立必要的性能测试集，评估 golang 版本演进可能带来的代码性能方面的变化

## 参考资料

[Generics implementation - GC Shape Stenciling](https://github.com/golang/proposal/blob/master/design/generics-implementation-gcshape.md)

[Generics can make your Go code slower](https://planetscale.com/blog/generics-can-make-your-go-code-slower)

[Shaving 40% Off Google’s B-Tree Implementation with Go Generics](https://www.scylladb.com/2022/04/27/shaving-40-off-googles-b-tree-implementation-with-go-generics/)

[golang issue 50182: generic functions are significantly slower than identical non-generic functions in some cases](https://github.com/golang/go/issues/50182)

[golang issue 54238: Go 1.19 might make generic types slower](https://github.com/golang/go/issues/54238)

[golang issue 57505: cmd/compile: performance regression in 1.20](https://github.com/golang/go/issues/57505)

[golang issue 56280: functions with type parameters cannot inline multiple levels deep across packages](https://github.com/golang/go/issues/56280)

## TODO

- [ ] 增加性能测试代码，针对泛型不同的场景进行回归测试