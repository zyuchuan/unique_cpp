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