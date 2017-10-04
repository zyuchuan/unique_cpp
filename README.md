# 前言

在过去几年中，我作为面试官参与了多次C++面试（这并不是说我是公司认可的C++技术专家，而是他们实在是找不到人去做这种吃力不讨好的事情）。每次面试，我都会问一些有关C++11的问题，在这个过程中，我逐渐发现了一个让人震惊的事实：几乎没有人了解C++11。曾经有一个自称很懂C++11的印度程序员，滔滔不绝的给我讲述`std:move()`：

“虽然我没用过`std::move`，但是从名字就可以看出来，这个函数的作用就是把一个对象从一个内存地址搬到另一个内存地址...”

我耐着性子听他说完，问道：“C++标准委员会决定加入这个函数是有原因的，你认为把一个对象从一个内存地址搬到另一个内存地址有什么意义？”

他回答：“呃...嗯...哦...”

后来我给这位来自南亚次大陆的C++技术专家打了零分，但是这并不能让我的心情轻松一些，我开始思考一个问题：距离C++11标准公布已经过去了这么多年，为什么广大一线的代码仔仍然拒绝接纳这个标准？

C++98无疑是成功的，伴随标准发布的STL是一个划时代的作品，它在优雅，高效，易用性和复杂度之间达到了完美的平衡。众所周知，STL是纯粹基于模板的，而模板很难，C++标准委员会的那些大牛们也知道这一点，所以他们设计了简单易用的接口，将STL的复杂性隐藏在这些简单易用的接口后面，你无须成为一个C++模板专家就能很好的使用STL，这是C++98能够如此成功的一个重要原因。

任何一门现代的编程语言，就语法层面而言，其进化方向都是趋近自然语言。不过C++标准委员会似乎并不认同这一点，他们似乎认为只要接口足够简单就好。所以C++11标准委员会极大地扩展了模板语法，使得C++的语法更加抽象。也许C++标准委员会的大牛们希望通过语法的抽象，是得源代码更加简洁。看看C++11标准库的源代码，语法确实很简洁，而且，也非常像天书。

