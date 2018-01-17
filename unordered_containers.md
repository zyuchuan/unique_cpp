# 无序关联容器

无序关联容器（Unordered associative container）是C++11标准库中新增的类型，包括

* unordered\_set
* unordered\_multiset
* unordered\_map
* unordered\_multimap

共四种类型，它们的共同特点是

1. 在容器内部，元素的排列是没有特定顺序的，这也正是它们被叫做“unordered container”的原因。

2. 都是通过hast table实现的。这点我们稍后会详细介绍。

标准库中已经有了`map`和`set`，为什么还要定义`unordered`的`map`和`set`呢？答案还是那两个字：**效率**。所以undered容器的效率更高一些，如果你对数据的顺序没有要求，建议使用新的unordered容器。

要了解unordered\_set和unordered\_map的工作原理，先要了解hash table的原理，要了解hash table，先要知道C++11标准库中的hash算法。

## 2.1 hash

C++11标准中明确指出：hash方程应该是一个_function object_，也就是重载了`operator()`的对象：

```
template<T>
struct hash {
    size_t operator()(T key) const noexcept;
};
```

可以看到，`hash<T>::operator()`接收一个`T`类型的参数，并返回一个`size_t`类型的值，这个值就是输入参数的hash值。注意关键字`noexcept`，C++标准不允许hash方程抛出异常。

另外，C++标准还对hash方程的计算结果有明确要求：

1. 对于类型为`T`的参数`t1`和`t2`，如果`t1 == t2`，则`hash<T>()(t1) == hash<T>()(t2)`;

2. 对于类型为`T`的参数`t1`和`t2`，如果`t1 != t2`，则`hash<T>()(t1) == hash<T>()(t2)`的概率应近似于`1.0/std::numeric_limits<size_t>::max()`（在我的MacBook Pro上，这个值为0.00000000000000000005.421，小数点后面19个`0`）。

不过，C++标准只是规定了hash方程的形式和必须满足的条件，具体到如何计算hash值，则没有要求。就libc++而言，针对不同的类型，其计算方法也不尽相同。

### 2.1.1 简单数值类型

对于简单数值类型，如`bool`、`int`、`char`等，libc++的hash算法也很简单：直接返回数值本身：

```
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

### 2.1.2 浮点数值类型

针对复杂数值类型，如`float`、`double`等，libc++提供了两种hash算法：[murmur2](https://en.wikipedia.org/wiki/MurmurHash)和[cityhash64](https://github.com/google/cityhash)：

```
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

```
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

### 2.1.3 内置非数值类型

对于标准库中的非数值类型，比如`string`等，标准库也提供了hash方程：

```
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

### 2.1.4 自定义类型

如果你有一个类，

```
struct Foo {
    int         _i;
    double      _d;
    std::string _s;

    Foo(int i, double d, const std::string &s)
        : _i(i), _d(d), _s(s) {}
};
```

你可以这样计算hash：

```
template<>
struct hash<Foo> : public unary_function<Foo, size_t> {
    size_t operator()（const Foo& foo）const noexcept {
        return murmur2_or_cityhash<size_t>()
            (static_cast<const void*>(&foo), sizeof(Foo));
    }
};
```

## 3 hash\_table



