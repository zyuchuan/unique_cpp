# Smart Pointer

智能指针的概念已经在C++的世界里存在很久了，不过在C++ 11之前，能用的智能指针都在boost库中，这次C++标准委员会终于决定将之加入到标准库中。

C++标准库提供了四种智能指针：

* unique_ptr
* shared_ptr
* weak_ptr
* auto_ptr

`auto_ptr`自C++98其就存在了，但是很少有人用，所以就不讲了

## 1. unique_ptr

顾名思义，`unique_ptr`就是独有一份，在两种情况下，其管理的指针会被处理掉：

1. 指针被摧毁
2. 调用`operator=`或`reset()`的时候。

标准库中定义了两种类型的`unique_ptr`：

1. 管理单个指针，指针通过`new`获得
2. 管理指针数组，指针通过`new[]`获得。

### 1.1 unique_ptr的源代码

