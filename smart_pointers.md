# Smart Pointers

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

```
// file: <memory>

template<class _Tp, class _Dp = default_delete<_Tp> >
class unique_ptr {
public:
    typedef _Tp element_type;
    typedef _Dp deleter_type;
    typedef typename __pointer_type<_Tp, deleter_type>::type pointer;
    
private:
    __compressed_pair<pointer, deleter_type> __ptr_;
    
    // ...
};
```

`unique_ptr`可以指定一个`deleter`，如果没有指定，则用默认的`default_delete`，`default_delete`就是针对`delete`的一个函数对象

```
template<class T>
struct default_delete {
    void operator()(T* ptr) const noexcept {
        delete ptr;
    }
};
```

`unique_ptr`将参数保存在一个`__compressed_pair`中，前面已经说过，`__compressed_pair`是一个针对空基类优化的`pair`，大家将它当做一个`std::pair`就好。

下面我们来看如何构造一个`unique_ptr`，其实`unique_ptr`的构造函数比较简单，比如说

```
// file: memory

template<class _Tp, class _Dp = default_delete<_Tp> >
class unique_ptr {
    // ...

public:
    explicit unique_ptr(pointer __p) noexcept 
        : __ptr_(std::move(__p)){ 
    
    }    


