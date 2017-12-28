
`std::function`是一个泛化的模板类，可以保存、拷贝和调用任何`callable`对象-- 如函数，lambda表达式，`bind expression`、函数对象（functor），甚至类成员函数指针等等。

`std::function`的用法非常灵活，下面的例子摘自[cppreference.com](http://en.cppreference.com/w/cpp/utility/functional/function)：

```
#include <functional>
#include <iostream>

struct Foo {
    Foo(int num) : num_(num){}
    void print_add(int i) const {std::cout << num+i << std::endl;}
    int num_;
};

void print_num(int i) {
    std::cout << i << '\n';
}

struct PrintNum() {
    void operator()(int i) const {
        std::cout << i << '\n';
    }
};

int main() {
    // store a free function
    std::function<void(int)> f_display = print_num;
    f_display(9);
    
    // store a lambda
    std::function<void()> f_display_42 = [](){print_num(42);};
    f_display_42();
    
    // store the result of a call to std::bind
    std::function<void()> f_display_31337 = std::bind(print_num, 31337);
    f_display_31337();
 
    // store a call to a member function
    std::function<void(const Foo&, int)> f_add_display = &Foo::print_add;
    const Foo foo(314159);
    f_add_display(foo, 1);
    f_add_display(314159, 1);
 
    // store a call to a data member accessor
    std::function<int(Foo const&)> f_num = &Foo::num_;
    std::cout << "num_: " << f_num(foo) << '\n';
 
    // store a call to a member function and object
    using std::placeholders::_1;
    std::function<void(int)> f_add_display2 = std::bind( &Foo::print_add, foo, _1 );
    f_add_display2(2);
 
    // store a call to a member function and object ptr
    std::function<void(int)> f_add_display3 = std::bind( &Foo::print_add, &foo, _1 );
    f_add_display3(3);
 
    // store a call to a function object
    std::function<void(int)> f_display_obj = PrintNum();
    f_display_obj(18);
}
```

看来`std::function`还真是个神奇的东西，什么东西都可以往里装，那它是怎样实现的呢？`std::function`的源代码在文件`functional`中：

```
// file: functional

// general declaration, undefined
template<class _Fp> class function;

// specialization for callable types
template<class _Rp, class ..._ArgTypes>
class function<_Rp(_ArgTypes...)> {
    typedef __function::__base<_Rp(_ArgTypes...)> __base;
    typename aligned_storage<3 * sizeof(void*)>::type __buf_;
    __base* __f_;
    // ...
};
```

`std::function`内部有两个成员，一个可以容纳三个指针的缓冲区和一个`__base`类型的指针`__f_`。下面我们将看到，这个`__f_`才是真正存储数据的地方。

```
// file: functional

namespace __function {

template<class _Fp> class __base;

// __base is an interface
template<class _Rp, class ..._ArgTypes>
class __base<_Rp(_ArgTypes...)> {
    // not copy constructible, not assignable
    __base(const __base&);
    __base& operator=(const __base&);
    
public:
    __base(){}
    virtual ~__base(){}
    virtual __base* __clone() const = 0;
    virtual void __clone(__base*) const = 0;
    virtual void destroy() noexcept = 0;
    virtual _Rp operator()(_ArgTypes&& ...) = 0;
};

// __func implements __base

template<class _FD, class _Alloc, class _FB> class __func;

template<class Fp, class _Alloc, class _Rp, class ..._ArgTypes>
class __func<_Fp, _Alloc, _Rp(_ArgTypes...)> : public __base<_Rp(_ArgTypes...)> {
    __compressed_pair<_Fp, _Alloc> __f_;
    
    // ...
};

}
```

几点说明：

1. 在`__function::__func`的定义中出现了一个模板参数`_Alloc`，这是个`allocator`。C++标准对这个`allocator`并没有做出详细的说明，这导致了不同的标准库实现在使用这个`allocator`使出现了一些混乱，标准委员会已经决定将`_Alloc`移出C++17，所以在后面的讨论中，我们会假装这个参数不存在。

2. `__compressed_pair`是个优化了内部存储的`pair`，功能和`std::pair`完全相同。

`std::function`实际就是个wrapper，真正的数据都存储在`std::__function::__func`中：

```
// file: functional


template<class Fp, class _Alloc, class _Rp, class ..._ArgTypes>
class __func<_Fp, _Alloc, _Rp(_ArgTypes...)> : public __base<_Rp(_ArgTypes...)> {

    __compressed_pair<_Fp, _Alloc> __f_;
    
public:
    explicit __func(_Fp&& __f)
        : __f_(piecewise_construct, 
               std::forward_as_tuple(std::move(__f)),
               std::forward_as_tuple()){}
         
    virtual __base<_Rp(_ArgTypes...)>* __clone() const;
    virtual void __clone(__base<_Rp(_ArgTypes...)>*) const;
    virtual void destroy() noexcept;
    virtual _Rp operator()(_ArgTypes&& ... __arg);
};
```

注意上面代码中的`__f_(piecewise_construct...`，这表明调用的“piecewise construct”版本的构造函数。关于`piecewise_construct`的含义可以参考[cppreference.com](http://en.cppreference.com/w/cpp/utility/piecewise_construct)。



template<class _Fp, class _Alloc, class _Rp, class ..._ArgType>
__base<_Rp(_ArgTypes...)>* 
__func<_Fp, _Alloc, _Rp(_ArgTypes...)>::__clone() const {
    typedef allocator_traits<_Alloc> _alloc_traits;
    typedef typename __rebind_alloc_helper<__alloc_traits, __func>::type _Ap;
    _Ap __a(__f_.second());
    typedef __allocator_destroy<_Ap> _Dp;
    
    unique_ptr<__func, _Dp> __hold(__a.allocate(1), _Dp(__a, 1));
    
    ::new (__hold.get()) __func(__f_.first(), _Alloc(__a));
    return __hold.release();
}


