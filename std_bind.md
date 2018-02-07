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
};
```

