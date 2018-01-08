# Smart Pointer

智能指针的概念已经在C++的世界里存在很久了，不过在C++ 11之前，能用的智能指针都在boost库中，这次C++标准委员会终于决定将之加入到标准库中。

C++标准库提供了四种智能指针：

* unique_ptr
* shared_ptr
* weak_ptr
* auto_ptr

`auto_ptr`自C++98其就存在了，但是很少有人用，所以就不讲了

## 1. unique_ptr

