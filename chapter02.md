# 2. tuple

## 2.1 tuple简史

[C++ Reference](http://en.cppreference.com/w/)对[tuple](http://en.cppreference.com/w/cpp/utility/tuple)的解释是“fixed-size collection of heterogeneous values”，也就是有固定长度的异构数据的集合。每一个C++代码仔都很熟悉的`std::pair`就是一种`tuple`。但是`std::pair`只能容纳两个数据，而C++11标准库中定义的`tuple`可以容纳任意多个、任意类型的数据。

## 2.2 tuple的用法

C++11 标准库中的`tuple`是一个模板类，使用时需要包含文件`<tuple>`：

```
#include <tuple>

using tuple_type = std::tuple<int, double, char>;
tuple_type t1(1, 2.0, 'a');
```

不过我们一般都用`std::make_tuple`函数来创建一个`tuple`，使用`std::make_tuple`的好处是不需要指定`tuple`参数的类型，编译器会自己推断：

```
#include <iostream>
#include <tuple>

auto t1 = std::make_tuple(1, 2.0, 'a');
std::cout << typeid(t1).name() << std::endl
```

可以使用`std::get`函数取出`tuple`中的数据：

```
auto t = std::make_tuple(1, 2.0, 'a');
std::cout << std::get<0>(t) << ", " << std::get<1>(t) << ", " << std::get<2>(t) << std::endl; // 1, 2.0, a
```

C++ 11标准库中还定义了一些辅助类，方便我们取得一个`tuple`类的信息：

```
using tuple_type = std::tuple<int, double, char>;

// tuple_size: 在编译期获得tuple元素个数
cout << std::tuple_size<tuple_type>::value << endl; // 3

// tuple_element: 在编译期获得tuple的元素类型
cout << typeid(std::tuple_element<2, tuple_type>::type).name() << endl; // c
```

## 2.3 tuple的实现原理

如果你对`boost::tuple`有所了解的话，应该知道`tuple`的实现使用了递归嵌套。`libc++`另辟袭击，采用了多重继承的手法实现。不过`tuple`的源代码极其复杂，大量使用了元编程技巧，很难理解。我不打算把这本书变成模板元编程入门，所以我在`libc++`的基础上，实现了一个极简版的`tuple`，希望能帮助你理解。

要理解`tuple`是怎样工作的，先要理解辅助类是怎样工作的

```
// forward declaration
template<class ...T> class tuple;

template<class ...T> class tuple_size;

template<class ...T>
class tuple_size<tuple<T...> > : public std::integral_constant<size_t, sizeof...(T)> {};
```

这个比较好理解，如果`tuple_size`作用于一个`tuple`，则`tuple_size`的值就是`sizeof...(T)`的值，关于`std::integral_constant`，可以看这里。

所以你可以这样写

```
cout << tuple_size<tuple<int, double, char> >::value << endl;    // 3
```
我们不需要知道`tuple`的具体定义，编译器只要知道它是一个可变参数模板类就行了。

下一个辅助类就是`tuple_types`：

```
template<class ...T> struct tuple_types{};

template<class T, size_t End = tuple_size<T>::value, size_t Start = 0>
struct make_tuple_types {};

template<class ...T, size_t End>
struct make_tuple_types<tuple<T...>, End, 0> {
    typedef tuple_types<T...> type;
};

template<class ...T, size_t End>
struct make_tuple_types<tuple_types<T...>, End, 0> {
        typedef tuple_types<T...> type;
};
    
```
注意这里有点简化了，

下面一个辅助类有点复杂：

```
template<size_t ...value> struct tuple_indices {};

template<class IndexType, IndexType ...values>
struct integer_sequence {
    template<size_t Start>
    using to_tuple_indices = tuple_indices<(values + Start)...>;
};

template<size_t End, size_t Start>
using make_indices_imp = typename __make_integer_seq<integer_sequence, size_t, End - Start>::template to_tuple_indices<Start>;

template<size_t End, size_t Start = 0>
struct make_tuple_indices {
    typedef make_indices_imp<End, Start> type;
};
```

`__make_integer_seq`是LLVM编译器的一个内置的函数，最终会展开成一系列的类

最后一个辅助类是`tuple_element`，它代表`tuple`中某一位置的类型



```
template<size_t Index, class Head>
class tuple_leaf {
    Head value;

public:
    tuple_leaf() : vlaue(){}
    
    template<class T>
    explicit tuple_leaf(cosnt T& t) : value(t){}
    
    Head& get(){return value;}
    const Head& get() const {return value;}
};
```

`tuple_leaf`是`tuple`的基本组成单位，每一个`tuple_leaf`都保存了一个索引，就是第一个模板参数，同时还有值。

```
template<class Index, class ...T> struct tuple_imp;

template<size_t ...Index, class ...T>
struct tuple_imp<tuple_indices<Index...>, T...> : 
    public tuple_leaf<Index, T>... {
    
    tuple_imp(){}
    
    template<size_t ...Uf, class ...Tf, class ...U>
    tuple_imp(tuple_indices<Uf...>, tuple_types<Tf...>, U&& ...u) 
        : tuple_leaf<Uf, Tf>(std::forward<U>(u))... {}
};

template<class ...T>
struct tuple {
    typedef tuple_imp<typename make_tuple_indices<sizeof...(T)>::type, T...> base;
    
    base base_;
    
    tuple(const T& ...t)
        : base(typename make_tuple_indices<sizeof...(T)>::type(),
               typename make_tuple_types<tuple, sizeof...(T)>::type(),
               t...){}
};
```

所以，如果有这样的代码

```
tuple(1, 2.0, 'a')
```

编译器会将只展开成

```
struct tuple_imp : public tuple_leaf<0, int>,    // value = 1
                   public tuple_leaf<1, double>    // value = 2.0
                   public tuple_leaf<2, char>    // value = 'a'
```

很巧妙不是？

为了方便使用，还定义了函数`make_tuple`：

```
template<class T>
struct make_tuple_return_imp {
    typedef T type;
};

template<class T>
struct make_tuple_return {
    typedef typename make_tuple_return_imp<typename std::decay<T>::type>::type type;
};

template<class ...T>
inline tuple<typename make_tuple_return<T>::type...> make_tuple<T&& ...t) {
    return tuple<typename make_tuple_return<T>::type...>(std::forward<T>(t)...);
}
```

你可以这样写`get`函数

```
template<size_t Index, class ...T>
inline typename tuple_element<Index, tuple<T...> >::type& get(tuple<T...>& t) {
    typedef typename tuple_element<Index, tuple<T...> >::type type;
    return static_cast<tuple_leaf<Index, type>&>(t.base_).get();
```

先通过`tuple_element`，取得对应的type和Index，然后将`tuple_imp`强制转型成相应的`tuple_leaf`，再取得值，很巧妙吧。