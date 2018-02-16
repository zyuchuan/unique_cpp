# Unique C++: C++ 11 源代码剖析

这个世界有两种C++语言：一种是C++，这是绝大多数C++程序员使用的编程语言；另一种是C++ 11，它在编程语言中是一种天书般的存在，C++大神们用它来开发Library或Framework。

要分析这种现象产生的原因，还得从C++的标准化说起。

C++的第一个标准化版本，C++ 98，无疑是成功的。伴随C++ 98发布的STL是一个划时代的作品，它在优雅，高效，易用性和复杂度之间达到了完美的平衡。众所周知，STL是纯粹基于模板的，而模板很难，C++标准委员会的大牛们也知道这一点，所以他们设计了简单易用的接口，将STL的复杂性隐藏在这些简单易用的接口后面，你无须成为一个C++模板专家就能很好的使用STL，这是C++ 98能够如此成功的一个重要原因。

任何一门现代编程语言，就语法层面而言，其进化方向都是趋近于自然语言。不过C++标准委员会似乎并不认同这一点，他们引领C++走上了另一条进化之路：抽象化。C++ 11是C++进化之路上的一个分水岭，C++ 11极大地扩展了模板语法，引入了“可变参数模板（Variadic Template）”，并在标准库中大量使用元编程（meta-programming）技巧，这使得C++代码越来越抽象，越来越难懂。在可预见的未来，这个趋势还会继续，C++标准库已经成为大神们炫技的舞台。

我相信C++标准委员会会继续让标准库的接口简单易用，同时我也相信标准委员会会持续引入新的语法以使C++变得更抽象，更难懂。对任何一个有尊严的C++程序员来说，看不懂标准库源代码，出门都不好意思和别人打招呼。

当然，要看懂标准库源代码，也不是什么“mission impossible”，Linus（没错，就是那个大神中的天神Linus Torvalds）早就给我们指明了方向：

### *Read the Fucking Source Code*
-- *Linus Torvalds*

这本书就是我在天神的指引下，花费数月，埋头阅读标准库源代码的成果。




## 这本书适合你吗？

这不是一本C++语言入门书，也不是一本讲述C++11新特性的书。这本书的目的是带你剖析C++11标准库源代码。

(未完待续）

## 关于开发环境

XCode 9.0

[libc++ 6.0](http://libcxx.llvm.org)

（未完待续）

## 本书源代码

在[Github](https://github.com/zyuchuan/unique_cpp)上




* [前言](prefix.md)
* [C++模板元编程常用技巧](cpp_metaprogramming_idioms.md)
* [type_traits](type_traits.md)
* [std::tupple](std_tupple.md)
* [chrono library](chrono_library.md)
* [std::swap](std_swap.md)
* [std::function](std_function.md)
* [smart pointers](smart_pointers.md)
* [unordered containers](unordered_containers.md)
* [std::array](std_array.asciidoc)
* [std::bind](std_bind.md)
* concurrency
* std::initializer\_list



