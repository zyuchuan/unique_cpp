# std::bind

`std::bind`将一个`callable object`与其参数绑定在一起，并生成一个新的`callable object`，我们来看一个例子：

```c++
#include <functional>
#include <iostream>

using namespace std;

void f(int n1, int n2, int n3) {
    cout << n1 << " " << n2 << " " << n3 << endl;
}

// 同时绑定函数f及参数
auto bind1 = bind(f, 1, 2, 3); 
bind1(); // 相当于调用 f(1, 2, 3)
```

在上面的例子中，`bind`函数将函数`f`和整数`1, 2, 3`“绑定”在一起，并生成了一个新的对象`bind1`，这是一个`callable object`，调用`bind1()`就相当于调用`f(1, 2, 3)`。

`std::bind`的用法很灵活，可以绑定各种各样的`callable object`：普通函数，函数对象（functor），类成员函数等等，有兴趣的同学可以参考[C++ Reference上的说明](en.cppreference.com/w/cpp/utility/functional/bind)。

`std::bind`还有一个巨牛X的功能：绑定占位符。还是刚才的例子，你可以这样绑定：

```c++
auto bind2 = bind(f, _1, _2, 3); // _1, _2 是占位符
bind2(1, 2); // 1 替换 _1, 2 替换 _2
```

`_1`、`_2`是定义在标准库中的模板类，它们的作用相当于“占位符”，也就是说你可以先用占位符把位子占着，待调用`bind`对象的时候再用实参去替换那些占位符。这是个很神奇的功能，就好像是“先上车，再买票”。多年来我都很好奇它是怎样实现的，今天我就要一探究竟。

## std::bind源代码剖析
```c++
// file: functional

template<class _Fp, class ..._BoundArgs>
inline __bind<_Fp, _BoundArgs...>
bind(_Fp&& __f, _BoundArgs&& ...__bound_args) {
    typedef __bind<_Fp, _BoundArgs...> type;
    return type(std::forward<_Fp>(__f), std::forward<_BoundArgs>(__bound_args)...);
}
```

`bind`其实什么也没做，它只是一个简单的wrapper，实质的内容都在一个叫做`__bind`的类中：

```c++
template<class _Fp, class ..._BoundArgs>
class __bind {
protected:
    typedef typename decay<_Fp>::type _Fd;
    typedef tuple<typename decay<_BoundArgs>::type...> _Td;
private:
    _Fd __f_;
    _Td __bound_args_;
    
    typedef typename __make_tuple_indices<sizeof...(_BoundArgs)>::type __indices;
    
public:
    template<class _Gp,
             class ..._BA,
             class = typename enable_if<is_constructible<_Fd, _Gp>::value &&
                     !is_same<typename remove_reference<_Gp>::type, __bind>::value
          >::type>
    explicit __bind(_Gp&& __f, _BA&& ...__bound_args)
      : __f_(std::forward<_Gp>(__f)),
        __bound_args_(std::forward<_BA>(__bound_args)...){}
    
   template<class ...Args>
   inline typename __bind_return<_Fd, _Td, tuple<_Args&&...> >::type
   operator()(_Args&& ...__args) {
       return __apply_functor(__f_, __bound_args_, __indices(),
                      tuple<_Args&&...>(std::forward<_Args>(__args)...));
   }
   
   // ...
};
```

如果你觉得你看到了假的C++代码，恭喜你！你终于上道了！C++标准库的源代码就是这么抽象！让我来给你解释一下：

`__bind`类有两个成员变量，一个是`__f_`，类型是`_Fd`，那`_Fd`又是啥？`_Fd`通常是个函数对象，这是由编译器自动生成的。另一个是`__bound_args_`，类型是个`tuple`。这个`tuple`是理解`__bind`的关键，我们后面还要详细解释。

`__bind`有个模板构造函数，但是，要使这个模板构造函数存在的前提条件是：
1. 可以通过`_Gp`构造出`_Fd`
2. `_Gp`不能是`__bind`自己
这就是第三个模板参数
```
typename enable_if<is_constructible<_Fd, _Gp>::value && 
!is_same<typename remove_reference<_Gp>::type, __bind>::value>::type
```

`__bind`是个`callable object`，所以定义了`operator()`，`operator()`会调用函数`__apply_functor`，注意这行

```
__apply_functor(__f_, __bound_args_, __indices(),
                  tuple<_Args&&...>(std::forward<_Args>(__args)...));
```

这里有两个`tuple`，一个是`__bound_args_`，是构造`__bind`对象时生成的，另一个是`tuple<_Args&&...>(std::forward<_Args>(__args)...`，这是调用`operator()`时根据传入的参数生成的。这两个`tuple`对理解`__bind`至关重要。

我们接着看`__apply_functor`:

```
template<class _Fp, class _BoundArgs, size_t ..._Indx, class _Args>
inline
typename __bind_return<_Fp, _BoundArgs, _Args>::type
__apply_functor(_Fp& __f, _BoundArgs& __bound_args, __tuple_indices<_Indx...>, _Args&& _args) {
    return __invoke(__f, __mu(std::get<_Indx>(__bound_args), __args)...);
}

```c++

> `__apply_functor`的第三个参数`__tuple_indices<_Indx...>`并没有用到，似乎是多余的，但是，当我移除这个参数的时候，`clang`抱怨说找不到`__invoke`。说实话，我也不知道`clang`为什么会有这种奇葩的表现。也许在作者的机器上也有同样的问题，所以加了这个参数。

`__apply_functor`内部又调用了`__invoke`，我们对这个函数不做多的纠缠，只要知道它是调用函数`__f(...)`就好。最关键的是这个神秘的`__mu`，它的作用是解析绑定参数。我们知道参数绑定有两个情况，一是构造函数时绑定参数，而是占位符绑定。`__mu`必须能正确地区分这两种情况。

我们先来看`placeholder`长什么样

```
namespace placeholders {
    template<int _Np> struct __ph{};
    constexpr __ph<1>   _1{};
    constexpr __ph<2>   _2{};
    constexpr __ph<3>   _3{};
    constexpr __ph<4>   _4{};
    constexpr __ph<5>   _5{};
    constexpr __ph<6>   _6{};
    constexpr __ph<7>   _7{};
    constexpr __ph<8>   _8{};
    constexpr __ph<9>   _9{};
    constexpr __ph<10> _10{};
}
```
这就是`placeholder`，它是空的，啥也没有，就是个占位符。那该如何判断一个变量或类是不是`placeholder`呢？

```
template<class _Tp>
struct __is_placeholder : public integral_constant<int, 0> {};

template<int _Np>
struct __is_placeholder<placeholders::__ph<_Np> > : public integral_constant<int, _Np> {
};
    
template<class _Tp>
struct is_placeholder : public __is_placeholder<typename std::remove_cv<_Tp>::type>{
};
```

如果一个对象不是`placeholder`，那`is_placeholder<T>`就继承自`integral_constant<int, 0>`，也就是说有`is_placeholder<T>::value = 0`。如果`T`是个`placeholder`，比如`__ph<5>`，那`is_placeholder<T>就继承自`integral_constant<int, 5>`，也就是说有`is_placeholder<T>::value = 5`。后面我们呢会看到，这正是分派绑定参数的关键所在。


```
// 如果_Ti不是一个placeholder
template<class _Ti, class _Uj>
inline
typename enable_if
<
    !is_bind_expression<_Ti>::value &&
    is_placeholder<_Ti>::value == 0 &&
    !__is_reference_wrapper<_Ti>::value,
    _Ti&
>::type
__mu(_Ti& __ti, _Uj&) {
    return __ti;
}

// 如果_Ti是一个placeholder
template<class _Ti, class _Uj>
inline
typename enable_if
<
    0 < is_placeholder<_Ti>::value,
    typename __mu_return2< 0 < is_placeholder<_Ti>::value, _Ti, _Uj>::type
>::type
__mu(_Ti&, _Uj& __uj) {
    const size_t _Indx = is_placeholder<_Ti>::value - 1;
    return std::forward<typename tuple_element<_Indx, _Uj>::type>(get::<_Indx>(__uj));
}
```

看明白了吗？如果参数是一个`placeholder`，则`is_placeholder<_Ti>::value > 0`，于是第二个`__mu`会被调用，因为`placeholder`的`value`同时也是`tuple`的`index`，所以就直接通过取得是不是很巧妙。

巧妙归巧妙，但我知道你还是不太明白，现在我用一个生动活泼的例子来说明。

还是回到文章开头的那个例子，假设你有个函数

```
void f(int n1, int n2, int n3) {
    cout << n1 << " " << n2 << " " << n3 << endl;
}
```

你把这个函数绑定到

```
auto bind2 = bind(f, _1, _2, 3);
```
编译器会为你生成一个`__bind`对象：

```
__bind<void (&)(int, int, int), const __ph<1>&, const __ph<2>&, int> f1 = {
    __f_ = 0x0000000100000ae0;
    __bound_args_ = {
        base = {__tuple_leaf<2, int, flase> = (value = 3)
    }
}
```

在上面的代码中，`__f_`的值是函数的入口地址，而`__bound_args_`的类型是

```
tuple<__ph<1>, __ph<2>, int>
```

我们知道`tuple`是从`__tuple_leaf`继承而来的，`tuple`有到少个模板参数，就继承自多少个`__tuple_leaf`，但是这里的情况有点特殊，`_1`和`_2`其实是个空类，所以在这里编译器就不会生成`__tuple_leaf`，所以我们看到`__bound_args_`的类型是`tuple<__ph<1>, __ph<2>, int>`，但是只有一个`base class`：`__tuple_leaf<2, int, false>`。这点很重要，这是`__mu`函数能够正确取得参数的关键。

当我们调用`bind2(1, 2)`的时候，就调用了`__bind::operator()`，进而调用了

```
template <class ..._Args>
typename __bind_return<_Fd, _Td, tuple<_Args&&...> >::type
operator()(_Args&& ...__args)
{
    return __apply_functor(__f_, __bound_args_, __indices(),
              tuple<_Args&&...>(_VSTD::forward<_Args>(__args)...));
}
```

在这里，`__f_`和`__bound_args_`就不多说了，`__indices()`的存在仅仅是为了让编译器满意。重要的来了，我们调用`operator()`时的参数，又被打包成了一个`tuple`，现在我们有了两个`tuple`，

```
__apply_function<_Fp& __f, _BoundArgs& __bound_args, __tuple_indices<_Indx...>, _Args&& __args) {
    return __invoke(__f, __mu(std::get<_Indx>(__bound_args), __args)...);
}
```

绝妙之笔来了，`_Indx`是一系列的整数索引，编译器在碰到

```
__mu(std::get<_Indx>(__bound_args), __args)...)
```
的时候，会调用`__mu_`三次：

```
__mu(get<0>(__bound_args), __args)
__mu(get<1>(__bound_args), __args)
__mu(get<2>(__bound_args), __args)
```

当调用`__mu(get<0>(__bound_args), __args)`的时候，`get<0>(__bound_args)`即

```
get<0>(tuple<__ph<1>, __ph<2>, int>)
```
会返回`__ph<1>`，而`__ph<1>`是一个`placeholder`，于是下面的函数会被激活

```
template<class _Ti, class _Uj>
inline
typename enable_if
<
    0 < is_placeholder<_Ti>::value,
    typename __mu_return2<0 < is_placeholder<_Ti>::value, _Ti, _Uj>::type
>::type
__mu(_Ti&, _Uj& __uj) {
    const size_t _Indx = is_placeholder<_Ti>::value - 1; // _Indx = 0
    return forward<typename tuple_element<_Indx, _Uj>::type>(_get<_Indx>(__uj));
}
```

也就是说，如果是`placeholder`，就从`tuple`中取出相应的值。

你可能会说这段代码无法编译，因为`std::get<0>(__bound_arg)`没有值。实际情况是，编译器根本不在乎，因为编译器激活了第二个`__mu`，而这个`__mu`根本不需要这个值。

同样，当`_Indx`为1的时候

当`_Indx`为2，于是下面的函数被激活

```
template <class _Ti, class _Uj>
inline
typename enable_if
<
    !is_bind_expression<_Ti>::value &&
    is_placeholder<_Ti>::value == 0 &&
    !__is_reference_wrapper<_Ti>::value,
    _Ti&
>::type
__mu(_Ti& __ti, _Uj&) {
    return __ti;
}
```

此时`__ti`是有值得，所以就从

明白了吧，很精妙是不是？