# 2 unordered_set和unordered_map

unordered容器是C++11标准库中新增的类型，包括`unordered_set`和`unordered_map`。我们知道C++标准库中已经有了`set`和`map`，他们的区别是什么呢？

1. `set`和`map`是有序的，而*unordered*容器，顾名思义，元素是没有序的
2. `set`和`map`的底层数据结构是红黑树，而`undered`容器的底层数据结构是`hash table`
3. 一个的复杂度为O(log(n))，一个是O(1)

所以undered容器的效率更高一些，如果你对数据的顺序没有要求，建议使用新的unordered容器。

要了解unordered_set和unordered_map的工作原理，先要了解hash table的原理，要了解hash table，先要知道C++11标准库中的hash算法。

## 2.1 hash

C++11标准库中的`hash`是一个模板类，声明如下：

```
template<class T> struct hash;

template<>
struct hash<bool> : pubic unary_function<bool, size_t> {
    size_t operator()(bool value) const {
        return static_cast<size_t>)(value);
    }
};

template<>
struct hash<char> : pubic unary_function<char, size_t> {
    size_t operator(char value) const {
        return static_cast<size_t>(value);
    }
};
```

可以看到，对于简单类型，C++标准库的hash算法很简单，就是数值本身，所以你可以这样使用hash

```
hash<int> int_hash;
cout << int_hash(3) << endl; // 3
```

标准库还提供了另外两种hash算法：[murmur2](https://en.wikipedia.org/wiki/MurmurHash)和[cityhash64](https://github.com/google/cityhash)。本书不是算法书，所以不打算详细介绍这两个算法，

```
// header: <memory>

template<class Size, size_t = sizeof(Size) * __CHAR_BIT__>
struct murmur2_or_cityhash;
```

上面的模板中，第二个参数用来指定使用哪种hash算法：

```
template<class Size>
struct murmur2_or_cityhash<Size, 32> {
    Size operator()(const void* key, size len) {
        // murmur2 hash算法
    }
};

template<class Size>
struct murmur2_or_cityhash<Size, 64> {
    Size operator()(const void* key, size len) {
        // cityhash64 hash算法
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
        return __murmur2_or_cityhash<size_t>()(&__u, sizeof(__u));
    }
};

template <class _Tp>
struct __scalar_hash<_Tp, 3>
    : public unary_function<_Tp, size_t>
{
    _LIBCPP_INLINE_VISIBILITY
    size_t operator()(_Tp __v) const _NOEXCEPT
    {
        union
        {
            _Tp __t;
            struct
            {
                size_t __a;
                size_t __b;
                size_t __c;
            } __s;
        } __u;
        __u.__t = __v;
        return __murmur2_or_cityhash<size_t>()(&__u, sizeof(__u));
    }
};

template <class _Tp>
struct __scalar_hash<_Tp, 4>
    : public unary_function<_Tp, size_t>
{
    _LIBCPP_INLINE_VISIBILITY
    size_t operator()(_Tp __v) const _NOEXCEPT
    {
        union
        {
            _Tp __t;
            struct
            {
                size_t __a;
                size_t __b;
                size_t __c;
                size_t __d;
            } __s;
        } __u;
        __u.__t = __v;
        return __murmur2_or_cityhash<size_t>()(&__u, sizeof(__u));
    }
};
```