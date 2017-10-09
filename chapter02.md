# 2 unordered_set和unordered_map

unordered容器是C++11标准库中新增的类型，包括`unordered_set`和`unordered_map`。我们知道C++标准库中已经有了`set`和`map`，他们的区别是什么呢？

1. `set`和`map`是有序的，而*unordered*容器，顾名思义，元素是没有序的
2. `set`和`map`的底层数据结构是红黑树，而`undered`容器的底层数据结构是`hash table`
3. 一个的复杂度为O(log(n))，一个是O(1)

所以undered容器的效率更高一些，如果你对数据的顺序没有要求，建议使用新的unordered容器。

要了解unordered_set和unordered_map的工作原理，先要了解hash table的原理，要了解hash table，先要知道C++11标准库中的hash算法。

## 2.1 hash

C++11标准中明确指出：hash方程应该是一个*function object*，也就是重载了`operator()`的对象：

```
template<T>
struct hash {
    size_t operator()(T key) const noexcept;
};
```

可以看到，`hash<T>::operator()`接收一个`T`类型的参数，并返回一个`size_t`类型的值，这个值就是输入参数的hash值。注意关键字`noexcept`，C++标准不允许hash方程抛出异常。

另外，C++标准还对hash方程的计算结果有明确要求：

1. 对于类型为`T`的参数`t1`和`t2`，如果`t1 == t2`，则`hash<T>()(t1) == hash<T>()(t2)`;

2. 对于类型为`T`的参数`t1`和`t2`，如果`t1 != t2`，则`hash<T>()(t1) == hash<T>()(t2)`的概率应近似于`1.0/std::numeric_limits<size_t>::max()`（在我的MacBook Pro上，这个值为0.00000000000000000005.421）。

不过，C++标准只是规定了hash方程的形式和必须满足的条件，具体到如何计算hash值，则没有要求，所以不同的实现由不同的计算方法，对libc++而言，

```
template<class T> struct hash;
```

类`hash`的声明及其简单，就是一个模板类，这个类没有定义，因为并没有一种通用的方法来计算hash值。相反，标准库中对每种类型都定义了特化版本，通过这些特化版本来计算hash值。

### 2.1.1 简单数值类型

C++标准库针对简单数值类型，如`bool`、`int`、`char`等，hash的计算很简单，就是返回数值本身：

```
// header: <functional>

template<>
struct hash<bool> : pubic unary_function<bool, size_t> {
    size_t operator()(bool value) const {
        return static_cast<size_t>)(value);
    }
};

template<>
struct hash<int> : public unary_function<int, size_t> {
    size_t operator()(int value) const {
        return static_cast<size_t>(value);
    }
};

template<>
struct hash<char> : pubic unary_function<char, size_t> {
    size_t operator(char value) const {
        return static_cast<size_t>(value);
    }
};
```

### 2.1.2 复杂数值类型

针对复杂数值类型，如`float`、`double`等，标准库提供了两种hash算法可供选择[murmur2](https://en.wikipedia.org/wiki/MurmurHash)和[cityhash64](https://github.com/google/cityhash)：

```
// header: <memory>

template<class Size, size_t = sizeof(Size) * __CHAR_BIT__>  // __CHAR_BIT_ = 8
struct murmur2_or_cityhash;

template<class Size>
struct murmur2_or_cityhash<Size, 32> {
    Size operator()(const void* key, Size len) {
        // murmur2 hash算法
        ...
    }
};

template<class Size>
struct murmur2_or_cityhash<Size, 64> {
    Size operator()(const void* key, Size len) {
        // cityhash64 hash算法
        ...
    }
};
```

为了方便使用

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

对于复杂数值类型，如`float`、`double`：

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

对于`string`，也定义了相应的hash

```
// header: <string>

template<class _CharT, class _Traits, class _Allocator>
struct hash<basic_string<_CharT, _Traits, _Allocator> >
    : public unary_function<basic_string<_CharT, _Traits, _Allocator>, size_t>
{
    size_t operator()(const basic_string<_CharT, _Traits, _Allocator>& __val) const;
};

template<class _CharT, class _Traits, class _Allocator>
size_t
hash<basic_string<_CharT, _Traits, _Allocator> >::operator()(
        const basic_string<_CharT, _Traits, _Allocator>& __val) const {
    return __do_string_hash(__val.data(), __val.data() + __val.size());
}
```

`__do_string_hash`定义在文件`<__string>`中：

```
// header: <__string>

template<class _Ptr>
inline size_t __do_string_hash(_Ptr __p, _Ptr __e) {
    typedef typename iterator_traits<_Ptr>::value_type value_type;
    return __murmur2_or_cityhash<size_t>()(__p, (__e-__p)*sizeof(value_type));
}
```


