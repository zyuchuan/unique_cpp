# 2. tuple

## 2.1 tuple简史

[cppreference](http://en.cppreference.com/w/cpp/utility/tuple)对tuple的解释是“fixed-size collection of heterogeneous values”，也就是有固定长度的异构数据的集合。

最简单的tuple就是`std::pair`，想必大家已经很熟悉了，但是`std::pair`只能容纳两个数据，而tuple可以容纳任意多个任意类型的数据。

## 2.2 tuple的用法

### 2.2.1 make_tuple

C++标准库对`tuple`进行了很好的包装，使得使用很方便，你用`make_tuple`创建一个`tuple`:

## 2.3 tuple的实现原理

如果你对`boost::tuple`有所了解的话，应该知道`tuple`的实现使用了递归嵌套。`libc++`另辟袭击，采用了多重继承的手法实现。不过`tuple`的源代码极其复杂，大量使用了元编程技巧，很难理解。我不打算把这本书变成模板元编程入门，所以我在`libc++`的基础上，实现了一个极简版的`tuple`，希望能帮助你理解。

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
```