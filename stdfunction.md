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

而`__func`对象可能存储的区域之一就是自带的默认缓冲区`__buf_`，部分MIPS指令集要求指令必须对其，所以这里的存储地址也要遵循平台默认的对齐方式。默认的大小是`3 * sizeof(void*)`，这是纯经验数据，对大部分的函数指针以及成员函数指针这个大小都够用。加上`__base*`指针，`__func`对象大小应该恰好等于`4*sizeof(void*)`。但因为可调用对象的大小千变万化，所以实际存储的区域可能也会在新开的堆上。

```
template<class _Fp, class _Alloc, class _Rp, class ...ArgTypes>
class __func<_Fp, _Alloc, _Rp(ArgTypes...)>
    : public __base<_Rp(_ArgTypes...)> {
    
    __compressed_pair<_Fp, _Alloc> __f_;
    // ...
};
```

`__func`是实际存储可调用对象的类，其继承了`__base`这个接口。可调用对象与`allocator`都存储在一个`__compressed_pair`中。