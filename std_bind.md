# std::bind

“bind”就是“绑定”的意思，`std::bind`就是讲`callable object`与其参数一起绑定。我们来看一个例子：

```
#include <functional>
#include <iostream>

using namespace std;

void f(int n1, int n2, int n3) {
    cout << n1 << " " << n2 << " " << n3 << endl;
}

// 同时绑定函数f及参数
auto bind1 = bind(f, 1, 2, 3);
bind1();

// 绑定部分参数
auto bind2 = bind(f, _1, _2, 3);
bind2(1, 2)

```

关于`std::bind`的详细用法可以参考[c++ reference](en.cppreference.com/w/cpp/utility/functional/bind)。

## std::bind实现

```
// file: functional

template<class _Fp, class ..._BoundArgs>
inline __bind<_Fp, _BoundArgs...>
bind(_Fp&& __f, _BoundArgs&& ...__bound_args) {
    typedef __bind<_Fp, _BoundArgs...> type;
    return type(std::forward<_Fp>(__f), std::forward<_BoundArgs>(__bound_args)...);
}
```

`bind`其实什么也没做，它只是一个简单的wrapper，实质的内容都在一个叫做`__bind`的类中：

```
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

```

> `__apply_functor`的第三个参数`__tuple_indices<_Indx...>`并没有用到，似乎是多余的，但是，当我移除这个参数的时候，`clang`抱怨说找不到`__invoke`。说实话，我也不知道`clang`为什么会有这种奇葩的表现。也许在作者的机器上也有同样的问题，所以加了这个参数。

`__apply_functor`内部又调用了`__invoke`，我们对这个函数不做多的纠缠，只要知道它是调用函数`__f(...)`就好。最关键的是这个神秘的`__mu`，它的作用是解析绑定参数。我们知道参数绑定有两个情况，一是构造函数时绑定参数，而是占位符绑定。`__mu`必须能正确地区分这两种情况。
