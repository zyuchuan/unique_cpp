原文：[std::function源码分析](https://my.oschina.net/u/1866819/blog/614382)


```
template<class _Rp, class ..._ArgTypes>
class function<_Rp(_ArgTypes...)>
    : public __function::__maybe_derive_from_unary_function<_Rp(_ArgTypes...>,
    : public __function::__maybe_derive_from_binary_function<_Rp(_ArgTypes...> {
    __base* __f_; // points to __func
    aligned_storage<3 * sizeof(void*)>::type __buf_;
    //...
};
```

`std::function`最重要的部分就是这个`__base*`指针，及其所指向的存储了实际可调用对象的多态类`__func`。`__base`类充当了`__func`类的接口，定义了`clone`，`operator()`等纯虚函数。