# Unique C++: C++ 11标准库源代码剖析

这个世界有两种C++语言：一种是C++，这是绝大多数C++程序员使用的编程语言；另一种是C++ 11，它在编程语言中是一种天书般的存在，C++大神们用它来开发Library或Framework。

要分析这种现象产生的原因，还得从C++的标准化说起。

C++的第一个标准化版本，C++ 98，无疑是成功的。伴随C++ 98发布的STL是一个划时代的作品，它在优雅，高效，易用性和复杂度之间达到了完美的平衡。众所周知，STL是纯粹基于模板的，而模板很难，C++标准委员会的大牛们也知道这一点，所以他们设计了简单易用的接口，将STL的复杂性隐藏在这些简单易用的接口后面，你无须成为一个C++模板专家就能很好的使用STL，这是C++ 98能够如此成功的一个重要原因。

任何一门现代编程语言，就语法层面而言，其进化方向都是趋近于自然语言。不过C++标准委员会似乎并不认同这一点，他们引领C++走上了另一条进化之路：抽象化。C++ 11是C++进化之路上的一个分水岭，C++ 11极大地扩展了模板语法，引入了“可变参数模板（Variadic Template）”，并在标准库中大量使用元编程（meta-programming）技巧，这使得C++代码越来越抽象，越来越难懂。在可预见的未来，这个趋势还会继续，C++标准库已经成为大神们炫技的舞台。

我相信C++标准委员会会继续让标准库的接口简单易用，同时我也相信标准委员会会持续引入新的语法以使C++变得更抽象，更难懂。对任何一个有尊严的C++程序员来说，看不懂标准库源代码，出门都不好意思和别人打招呼。

当然，要看懂标准库源代码，也不是什么“mission impossible”，Linus（没错，就是那个大神中的天神Linus Torvalds）早就给我们指明了方向：

>#### *Read the Fucking Source Code*
-- *Linus Torvalds*

这本书就是我在天神的指引下，花费数月，埋头阅读标准库源代码的成果。


## 这本书适合你吗？

这不是一本C++语言入门书，也不是一本讲述C++ 11新特性的书。本书的目的是剖析C++ 11标准库源代码，教你阅读那些如天书一般的C++源代码。为此你应该熟悉C++ Template；了解C++ 11新特性，比如*Move Sementics*，*Variadic Template*等；如果你懂*Template Meta-programming*，那这本书对你来说就彻底没难度了。


## 关于标准库版本

本书使用的标准库为[libc++ 6.0](http://libcxx.llvm.org)，这是Apple的XCode 9.0使用的标准库，我放了一份拷贝在[这里](https://github.com/zyuchuan/libcpp_source)。

## 目录索引

* [C++ Template Metaprogramming](ch01_cpp_metaprogramming.md)
* [Type Traits](ch02_type_traits.md)
* [std::array](ch03_std_array.md)
* [Smart Pointers](ch04_smart_pointers.md)
* [Chrono Library](ch05_chrono_library.md)
* [std::swap](ch06_std_swap.md)
* [std::tupple](ch07_std_tupple.md)
* [unordered containers](ch08_unordered_containers.md)
* [std::bind](ch09_std_bind.md)
* [std::function](ch10_std_function.md)
* concurrency



