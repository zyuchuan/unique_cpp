# 2. tuple

## 2.1 tuple简史

[C++ Reference](http://en.cppreference.com/w/)对[tuple](http://en.cppreference.com/w/cpp/utility/tuple)的解释是“fixed-size collection of heterogeneous values”，也就是有固定长度的异构数据的集合。每一个C++代码仔都很熟悉的`std::pair`就是一种`tuple`。但是`std::pair`只能容纳两个数据，而C++11标准库中定义的`tuple`可以容纳任意多个、任意类型的数据。

## 2.2 tuple的用法

C++ 11标准库中的`tuple`是一个模板类，使用时需要包含头文件`<tuple>`：

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

关于`tuple`的用法就简要介绍到这里，C++ Reference上有关于[`std::tuple`](http://en.cppreference.com/w/cpp/utility/tuple)的详细介绍，感兴趣的同学可以去看看。下面我们着重讲一下`tuple`的实现原理。

## 2.3 tuple的实现原理

如果你对`boost::tuple`有所了解的话，应该知道`boost::tuple`是使用递归嵌套实现的，这也是大多数类库--比如Loki和 MS VC--实现`tuple`的方法。而`libc++`另辟蹊径，采用了多重继承的手法实现。`libc++`的`tuple`的源代码极其复杂，大量使用了元编程技巧，如果我一行行解读这些源代码，那本章就会变成C++模板元编程入门。为了让你有继续看下去的勇气，我将`libc++ tuple`的源代码简化，实现了一个极简版`tuple`，希望能帮助你理解`tuple`的工作原理。

### 2.3.1 tuple_size

我们先从辅助类开始：

```
// forward declaration
template<class ...T> class tuple;

template<class ...T> class tuple_size;

// 针对tuple类型的特化
template<class ...T>
class tuple_size<tuple<T...> > : public std::integral_constant<size_t, sizeof...(T)> {};
```

这个比较好理解，如果`tuple_size`作用于一个`tuple`，则`tuple_size`的值就是`sizeof...(T)`的值。所以你可以这样写：

```
cout << tuple_size<tuple<int, double, char> >::value << endl;    // 3
```

### 2.3.2 tuple_types

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

这个简化版的`typle_types`并不做具体的事，就是纯粹的类型定义。需要说明的是，如果你要使用这个简化版的`tuple_types`，最好保证`End == sizeof...(T) - 1`，否则有可能编译器会报错。


### 2.3.3 type_indices

下面这个有点复杂：

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

`__make_integer_seq`是LLVM编译器的一个内置的函数，它的作用--顾名思义--是在编译期生成一个序列，如果你写下这样的代码：

```
__mkae_integer_seq<integer_sequence, size_t, 3>
```

则编译器会将它展开成：

```
integer_sequence<0>, integer_sequence<1>, integer_sequence<2>
```

所以，对于下面的代码：

```
make_tuple_indices<3>
```

编译器最终会展开成：

```
tuple_indices<0>, tuple_indices<1>, tuple_indices<2>
```

这样就定义了一个`tuple`的索引。


### 2.3.4 tuple_element

最后一个辅助类是`tuple_element`：

```
namespace indexer_detail {
    template<size_t Index, class T>
    struct indexed {
        using type = T;
    };
        
    template<class Types, class Indexes> struct indexer;
        
    template<class ...Types, size_t ...Index>
    struct indexer<tuple_types<Types...>, tuple_indices<Index...> > : public indexed<Index, Types>... {};
        
    template<size_t Index, class T>
    indexed<Index, T> at_index(indexed<Index, T> const&);
} // namespace indexer_detail
    
template<size_t Index, class ...Types>
using type_pack_element = typename decltype(indexer_detail::at_index<Index>(
    indexer_detail::indexer<tuple_types<Types...>,
    typename make_tuple_indices<sizeof...(Types)>::type>{}))::type;
    
template<size_t Index, class ...T>
struct tuple_element<Index, tuple_types<T...> > {
    typedef type_pack_element<Index, T...> type;
};
    
template<size_t Index, class ...T>
struct tuple_element<Index, tuple<T...> > {
    typedef type_pack_element<Index, T...> type;
};
```

我知道你又迷糊了，让我详细解释一下，如果你写下这样的代码：

```
tuple_element<1, tuple<int, double, char> >::type
```

编译器会展开成`tuple_pack_element<1, int, double, char>`，再展开成

```
decltype(
    at_index<1>(indexer<tuple_types<int, double, char>, tuple_indices<3>>{})
)
```

注意，上面的代码中定义了类`indexer`，而函数`at_index`只接受`at_index`类型的参数，于是编译器再来个向上转型，将`indexer`向上转型成`indexed<1,double>`（仔细想想为什么？），而`indexed<1, double>::type`就是`double`。

看似很复杂，其实无非就是变量代换而已，编译器只不过是机器，做不了复杂的推理。

好了，酒水备齐了，下面上主菜：

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

`tuple_leaf`是`tuple`的基本组成单位，每一个`tuple_leaf`都保存了一个索引（就是第一个模板参数），同时还有值。

继续看：

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
还是多重继承，每一个`tuple`都继承自数个`tuple_leaf`，而前面说过，每个`tuple_leaf`都有索引和值，你看，齐活了不是。所以，如果有这样的代码

```
tuple(1, 2.0, 'a')
```

编译器会展开成

```
struct tuple_imp : public tuple_leaf<0, int>,       // value = 1
                   public tuple_leaf<1, double>     // value = 2.0
                   public tuple_leaf<2, char>       // value = 'a'
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