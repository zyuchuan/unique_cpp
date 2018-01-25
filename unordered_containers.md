# 无序关联容器

无序关联容器（Unordered associative container）是C++11标准库中新增的类型，包括

* unordered\_set
* unordered\_multiset
* unordered\_map
* unordered\_multimap

共四种类型，它们的共同特点是

1. 在容器内部，元素的排列是没有特定顺序的，这也正是它们被叫做“unordered container”的原因。

2. 都是通过hast table实现的。这点我们稍后会详细介绍。

标准库中已经有了`map`和`set`，为什么还要定义`unordered`的`map`和`set`呢？答案还是那两个字：**效率**。我们知道，`map`和`set`的底层数据结构是红黑树，执行查找、插入，删除等操作的时间复杂度为O(logn)；而`unordered map`和`unordered set`的底层数据为`hash table`，执行查找、插入和删除等操作的时间复杂度为O(1)，明显要快很多，所以如果对元素的排列顺序没有要求，建议使用无序关联容器。

既然无序关联容器都是基于`hash table`的，那我们就有必要先了解一下C++ 11中的hash算法。

## 1. C++ 11中的hash算法

C++ 11标准对如何计算hash有明确的要求：

1. hash方程应该是一个`function object`，声明如下所示：

```c++
template<T>
struct hash {
    size_t operator()(T key) const noexcept;
};
```

2. 对于类型为`T`的参数`t1`和`t2`，如果`t1 == t2`，则`hash<T>()(t1) == hash<T>()(t2)`;

3. 对于类型为`T`的参数`t1`和`t2`，如果`t1 != t2`，则`hash<T>()(t1) == hash<T>()(t2)`的概率应近似于`1.0/std::numeric_limits<size_t>::max()`（在我的MacBook Pro上，这个值为0.00000000000000000005.421，小数点后面19个`0`）。

不过，C++标准只是规定了hash方程的形式和必须满足的条件，具体到如何计算hash值，则没有要求。就libc++而言，针对不同的类型，其计算方法也不尽相同。

### 1.1 简单数值类型

对于简单数值类型，如`bool`、`int`、`char`等，libc++的hash算法也很简单：直接返回数值本身。

```c++
// header: <functional>

template<class T> struct hash; // forward declaration

// specialization for bool
template<>
struct hash<bool> : pubic unary_function<bool, size_t> {
    size_t operator()(bool value) const noexcept {
        return static_cast<size_t>)(value);
    }
};

// specialization for int
template<>
struct hash<int> : public unary_function<int, size_t> {
    size_t operator()(int value) const noexcept {
        return static_cast<size_t>(value);
    }
};

// specialization for char
template<>
struct hash<char> : pubic unary_function<char, size_t> {
    size_t operator(char value) const noexcept {
        return static_cast<size_t>(value);
    }
};
```

### 1.2 浮点数值类型

针对复杂数值类型，如`float`、`double`等，libc++提供了两种hash算法：[murmur2](https://en.wikipedia.org/wiki/MurmurHash)和[cityhash64](https://github.com/google/cityhash)：

```c++
// header: <memory>

template<class Size, size_t = sizeof(Size) * 8>
struct murmur2_or_cityhash;

// use murmur2 on 32-bit system, 
// because size_t is 32 bits on 32-bit system
template<class Size>
struct murmur2_or_cityhash<Size, 32> {
    Size operator()(const void* key, Size len) {
        // murmur2 hash算法
        ...
    }
};

// use cityhash64 on 64-bit system,
// because size_t is 64 bits on 64-bit system
template<class Size>
struct murmur2_or_cityhash<Size, 64> {
    Size operator()(const void* key, Size len) {
        // cityhash64 hash算法
        ...
    }
};
```

`murmur2_or_cityhash::operator()`接受两个参数，并不满足C++标准的要求，为了方便使用，libc++又定义了一个外敷类`scalar_hash`：

```c++
template<class T, size_t = sizeof(T) / sizeof(size_t)>
struct scalar_hash;

template<class T>
struct scalar_hash<T, 0> : public unary_function<T, size_t> {
    size_t operator()(T value) const {
        union {
            T      t;
            size_t a;
        } _u;
        _u.a = 0;
        _u.t = value;
        return _u.a;
    }
};

template <class T>
struct scalar_hash<T, 1> : public unary_function<T, size_t> {
    size_t operator()(T value) const {
        union{
            T        t;
            size_t   a;
        } _u;
        _u.t = value;
        return _u.a;
    }
};

template <class T>
struct scalar_hash<T, 2> : public unary_function<T, size_t> {
    size_t operator()(Tp value) const {
        union {
            Tp t;
            struct {
                size_t a;
                size_t b;
            } s;
        } _u;
        _u.t = value;
        return murmur2_or_cityhash<size_t>()(&_u, sizeof(_u));
    }
};

template <class T>
struct scalar_hash<T, 3> : public unary_function<T, size_t> {
    size_t operator()(T value) const {
        union {
            T t;
            struct {
                size_t _a;
                size_t _b;
                size_t _c;
            } s;
        } _u;
        _u.t = value;
        return murmur2_or_cityhash<size_t>()(&_u, sizeof(_u));
    }
};

template <class T>
struct scalar_hash<T, 4> : public unary_function<T, size_t> {
    size_t operator()(T value) const {
        union {
            T t;
            struct {
                size_t a;
                size_t b;
                size_t c;
                size_t d;
            } s;
        } _u;
        _u.t = value;
        return murmur2_or_cityhash<size_t>()(&_u, sizeof(_u));
    }
};
```

浮点数值类型最终的hash计算方法如下：

```
template <>
struct hash<float> : public scalar_hash<float> {
    size_t operator()(float value) const
    {
        // -0.0 and 0.0 should return same hash
       if (value == 0)
           return 0;
        return scalar_hash<float>::operator()(value);
    }
};

template <>
struct hash<double> : public scalar_hash<double> {
    size_t operator()(double value) const
    {
        // -0.0 and 0.0 should return same hash
       if (value == 0)
           return 0;
        return scalar_hash<double>::operator()(value);
    }
};
```

### 1.3 内置非数值类型

对于标准库中的非数值类型，比如`string`等，标准库也提供了hash方程：

```c++
// header: <string>

template<class CharT, class Traits, class Allocator>
struct hash<basic_string<CharT, Traits, Allocator> >
    : public unary_function<basic_string<CharT, Traits, Allocator>, size_t> {

    size_t operator()(const basic_string<CharT, Traits, Allocator>& val) const noexcept;
};

template<class CharT, class Traits, class Allocator>
size_t hash<basic_string<CharT, Traits, Allocator> >::operator()(
        const basic_string<CharT, Traits, Allocator>& val) const noexcept {
    return __do_string_hash(val.data(), val.data() + val.size());
}
```

`__do_string_hash`定义在文件`<__string>`中：

```
// header: <__string>

template<class Ptr>
inline size_t __do_string_hash(Ptr p, Ptr e) {
    typedef typename iterator_traits<Ptr>::value_type value_type;
    return murmur2_or_cityhash<size_t>()(p, (e-p)*sizeof(value_type));
}
```

### 1.4 自定义类型

如果你有一个类，

```c++
struct Foo {
    int         _i;
    double      _d;
    std::string _s;

    Foo(int i, double d, const std::string &s)
        : _i(i), _d(d), _s(s) {}
};
```

你可以这样计算hash：

```c++
template<>
struct hash<Foo> : public unary_function<Foo, size_t> {
    size_t operator()（const Foo& foo）const noexcept {
        return murmur2_or_cityhash<size_t>()
            (static_cast<const void*>(&foo), sizeof(Foo));
    }
};
```

现在我们已经对`hash`算法有所了解，接下来我们就讲讲那些基于`hash`算法的容器是如何实现的。虽然C++ 11标准库中基于`hash`算法的容器一共有四种，但是它们的实现方式大同小异，所以我们就讲讲最典型的`hash`容器：`unordered_map`。

## 2. unordered_map

首先来看`unordered_map`的声明：

```c++
// file: unordered_map

template<class _Key, class _Tp, class _Hash = hash<_Key>, class _Pred = equal_to<_key>, 
         class _Alloc = allocator<pair<const _Key, _Tp> > >
class unordered_map {
public:
    // types
    typedef _Key                                           key_type;
    typedef _Tp                                            mapped_type;
    typedef _Hash                                          hasher;
    typedef _Pred                                          key_equal;
    typedef _Alloc                                         allocator_type;
    typedef pair<const key_type, mapped_type>              value_type;
    typedef pair<key_type, mapped_type>                    __nc_value_type;
    typedef value_type&                                    reference;
    typedef const value_type&                              const_reference;

private:
    typedef __hash_table<__value_type, __hasher, __key_equal, __allocator_type>   __table;
    __table __table_;

    // ...
};
```

可以看到`unordered_map`的实现采用了`Pimpl idiom`，`unordered_map`只是个`wrapper`，真正的实现是在`__hash_table`中。要讲清楚`__hash_table`不是一件容易的事情，好在`libc++`的`__hash_table`从技术上讲并没有什么奇特之处，仍然采用了大家都很熟悉的`bucket list`形式，如下图所示：


![Hash table示意图](/assets/unordered_containers_hash_table.jpg)

在进入`__hash_table`之前，先看几个辅助类，这几个辅助类对我们理解`__hash_table`很有帮助：

```c++
// file: __hash_table

// __hash_value_type只有声明，没有定义。
template<class _Key, class _Tp> struct __hash_value_type;

template<class _Key, class _Tp>
struct __hash_key_value_types<__hash_value_type<_Key, _Tp> > {
    typedef _Key                            key_type;
    typedef _Tp                             mapped_type;
    typedef __hash_value_type<_Key, _Tp>    __node_value_type;
    typedef pair<const _Key, _Tp>           __container_value_type;
    typedef pair<_Key, _Tp>                 __nc_value_type;
    typedef __container_value_type          __map_value_type;
    static const bool __is_map = true;

    inline static key_type const& __get_key(__container_value_type const& __v) {
        return __v.first;
    }

    template <class _Up>
    inline static typename enable_if<__is_same_uncvref<_Up, __node_value_type>::value, __container_value_type const&>::type
    __get_value(_Up& __t) {
        return __t.__cc;
    }

    template <class _Up>
    inline static typename enable_if<__is_same_uncvref<_Up, __container_value_type>::value, __container_value_type const&>::type
    __get_value(_Up& __t) {
        return __t;
    }

    inline static __container_value_type* __get_ptr(__node_value_type& __n) {
        return addressof(__n.__cc);
    }
};
```

下面再看几个辅助类：

```c++
template<class _NodePtr>
struct __hash_node_base {
    typedef typename std::pointer_traits<_NodePtr>::element_type __node_type;
    typedef __hash_node_base __first_node;
    typedef typename std::__rebind_pointer<_NodePtr, __first_node>::type __node_base_pointer;
    typedef _NodePtr __node_pointer;
    typedef __node_base_pointer __next_pointer;

    __next_pointer __next_;
};

template<class _Tp, class _VoidPtr>
struct __hash_node : public __hash_node_base<typename std::__rebind_pointer<_VoidPtr, __hash_node<_Tp, _VoidPtr> >::type> {
    typedef _Tp __node_value_type;

    size_t  __hash_;
    __node_value_type __value_;
};

template <class _NodePtr, class _NodeT = typename pointer_traits<_NodePtr>::element_type>
struct __hash_node_types {
    // ...
};

template<class _NodeValueTp, class _VoidPtr>
struct __make_hash_node_types {
    typedef __hash_node<_NodeValueTp, _VoidPtr> _NodeTp;
    typedef typename std::__rebind_pointer<_VoidPtr, _NodeTp>::type _NodePtr;
    typedef std::__hash_node_types<_NodePtr> type;
};
```

到目前为止， 我们已经定义了`__hash_node_type，代码看似很复杂抽象，其实也没啥，就是定义了一个node，只是这么抽象的代码，真不知道作者是怎么写出来的，又是怎么测试的。

下面正式进入`__hash_table`：

```c++
template<class _Tp, class _Hash, class _Equal, class _Alloc>
class __hash_table {
public:
    typedef _Tp         value_type;
    typedef _Hash       hasher;
    typedef _Equal      key_equal;
    typedef _Alloc      allocator_type;

private:
    typedef std::allocator_traits<allocator_type> __alloc_traits;
    typedef typename __make_hash_node_types<value_type, typename __alloc_traits::void_pointer>::type _NodeTypes;

public:
    typedef typename _NodeTypes::__node_value_type           __node_value_type;
    typedef typename _NodeTypes::__container_value_type      __container_value_type;
    typedef typename _NodeTypes::key_type                    key_type;
    typedef value_type&                              reference;
    typedef const value_type&                        const_reference;
    typedef typename __alloc_traits::pointer         pointer;
    typedef typename __alloc_traits::const_pointer   const_pointer;
    typedef typename __alloc_traits::size_type       size_type;
    typedef typename _NodeTypes::difference_type     difference_type;

public:
    // Create __node

    typedef typename _NodeTypes::__node_type __node;
    typedef typename std::__rebind_alloc_helper<__alloc_traits, __node>::type __node_allocator;
    typedef std::allocator_traits<__node_allocator>       __node_traits;
    typedef typename _NodeTypes::__void_pointer      __void_pointer;
    typedef typename _NodeTypes::__node_pointer      __node_pointer;
    typedef typename _NodeTypes::__node_pointer      __node_const_pointer;
    typedef typename _NodeTypes::__node_base_type    __first_node;
    typedef typename _NodeTypes::__node_base_pointer __node_base_pointer;
    typedef typename _NodeTypes::__next_pointer      __next_pointer;

private:
    typedef typename std::__rebind_alloc_helper<__node_traits, __next_pointer>::type __pointer_allocator;
    typedef std::__bucket_list_deallocator<__pointer_allocator> __bucket_list_deleter;
    typedef std::unique_ptr<__next_pointer[], __bucket_list_deleter> __bucket_list;
    typedef std::allocator_traits<__pointer_allocator>          __pointer_alloc_traits;
    typedef typename __bucket_list_deleter::pointer       __node_pointer_pointer;

    // --- Member data begin ---
    __bucket_list                               __bucket_list_;
    std::pair<__first_node, __node_allocator>   __p1_;
    std::pair<size_type, hasher>                __p2_;
    std::pair<float, key_equal>                 __p3_;

public:
    size_type size() const noexcept { return __p2_.first();}

    float max_load_factor() const noexcept { return __p3_.first();}
};
```

 源代码太长，从头读到尾是不可能的，我们只能找几个有代表性的函数来讲解

默认构造函数

```c++
// file: __hash_table

template<class _Tp, class _Hash, class _Equal, class _Alloc>
inline __hash_table<_Tp, _Hash, _Equal, _Alloc>::__hash_table()
    : __p2_(0), __p3(1.0f) {

}
```

插入一个<key, value>

```c++
// file: unordered_map

// ...

pair<iterator, bool> insert(const value_type& __) {
    return __table_.__insert_unique(__x);
}

// file: __hash_table

pair<iterator, bool> __insert_unique(__container_value_type&& __x) {
    return __emplace_unique_key_args(_NodeType::__get_key(__x), std::move(__x));
}

template<class _Tp, class _Hash, class _Equal, class _Alloc>
template<class _key, class ..._Args>
pair<typename __hash_table<_Tp, _Hash, _Equal, _Alloc>::iterator, bool>
__hash_table<_Tp, _Hash, _Equal, _Alloc>::__emplace_unique_key_args(_Key const& __k, _Args&&... __args) {
    size_t __hash = hash_function()(__k);
    size_type __bc = bucket_count();
    bool __inserted = false;
    __next_pointer __nd;
    size_t __chash;

    if (__bc != 0) {
        __chash = __constrain_hash(__hash, __bc);
        __nd = __bucket_list_[_chash];
        if (__nd != nullptr) {
            for (__nd = __nd->next; __nd != nullptr && (__nd->__hash() == __hash || __constrain_hash(__nd->__hash(), __bc) == __chash);
                    __nd = __nd->__next_) {
                if (key_eq()(__nd->__upcast()->__value, __k))
                    goto __done;
            }
        }
    }
    {
        __node_holder __h = __construct_node_hash(__hash, std::forward<_Args>(__args)...);
        if (size() + 1 > __bc * max_load_factor() || __bc == 0) {
            // rehash
        }

        __next_pointer __pn = __bucket_list_[__chash];
        if (__pn == nullptr) {
            __pn = __p1_.first().__ptr();
            __h->__next_ = __pn->__next_;
            __pn->__next_ = __h.get()->__ptr();

            __bucket_list_[__chash] = __pn;
            if (__h->__next_ != nullptr)
                __bucket_list_[__constrain_hash(__h->__next_->__hash(), __bc)] = __h.get()->__ptr();
        }
        else {
            __h->__next_ = __pn->__next_;
            __pn->__next_ = static_cast<__next_pointer>(__h.get());
        }
        __nd = static_cast<__next_pointer>(__h.release());
        ++size();
        __inserted = true;
    }
__done:
    return pair<iterator, bool>(iterator(__nd), __inserted);
}
```
