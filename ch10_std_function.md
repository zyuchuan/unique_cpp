# std::function


`std::function`是一个泛化的“function wrapper”，可以保存、拷贝和调用函数，lambda表达式，`bind expression`、函数对象（functor），甚至类成员函数指针和类成员变量指针等等。

`std::function`的用法非常灵活，[cppreference.com](http://en.cppreference.com/w/cpp/utility/functional/function)列出了它的八大用法：

```c++
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
    // 1. 用于普通函数
    std::function<void(int)> f_display = print_num;
    f_display(9);
    
    // 2. 用于lambda表达式
    std::function<void()> f_display_42 = [](){print_num(42);};
    f_display_42();
    
    // 3. 用于std::bind
    std::function<void()> f_display_31337 = std::bind(print_num, 31337);
    f_display_31337();
 
    // 4. 用于类成员函数
    std::function<void(const Foo&, int)> f_add_display = &Foo::print_add;
    const Foo foo(314159);
    f_add_display(foo, 1);
    f_add_display(314159, 1);
 
    // 5. 用于类成员变量
    std::function<int(Foo const&)> f_num = &Foo::num_;
    std::cout << "num_: " << f_num(foo) << '\n';
 
    // 6. 用于对象和它的成员函数
    using std::placeholders::_1;
    std::function<void(int)> f_add_display2 = std::bind( &Foo::print_add, foo, _1 );
    f_add_display2(2);
 
    // 7. 用于对象指针和它的成员函数
    std::function<void(int)> f_add_display3 = std::bind( &Foo::print_add, &foo, _1 );
    f_add_display3(3);
 
    // 8. 用于函数对象
    std::function<void(int)> f_display_obj = PrintNum();
    f_display_obj(18);
}
```

## std::function源代码分析

`std::function`的源代码在文件`functional`中：

```c++
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

`std::function`内部有两个成员：一个可以容纳三个指针的缓冲区`__buf_`和一个`__base`类型的指针`__f_`。那这个`__base`又是啥呢？

```c++
// file: functional

namespace __function {

    template<class _Fp> class __base;

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

    // ...
}
```

`__base`其实是一个接口。我们应该能猜到，`std::function`一定是把“callable object”的信息存放在一个实现了`__base`接口的对象中。我们暂且不去理会这个实现了`__base`接口的对象，回到`std::function`的声明中来：

```c++
// file: functional 

template<class _Rp, class ..._ArgTypes>
class function<_Rp(_ArgTypes...)> {

    // ...
    
    template<class _Fp, class = _EnableIfCallable<_Fp>>
    function(_Fp);
};
```

`std::function`的构造函数是个模板函数，它的第二个模板参数有个默认值`_EnableIfCallable<_Fp>`，这个参数很重要，它决定了什么样的数据可以用于构造一个`std::function`：

```c++
// file: functional

template<class _Rp, class ..._ArgTypes>
class function<_Rp(_ArgTypes...)> {
    
    // ...

    template<class _Fp, bool = __lazy_and<
        integral_constant<bool, !is_same<__uncvref_t<_Fp>, function>::value>, __invokable<_Fp&, _ArgTypes...>
    >::value>
    struct __callable;

    template<class _Fp>
    struct __callable<_Fp, true> {
        static const bool value = is_same<void, _Rp>::value ||
            is_convertible<typename __invoke_of<Fp&, _ArgTypes...>::type, _Rp>::value;
    };

    template<class _Fp>
    struct __callable<_Fp, false> {
        static const bool value = false;
    };

    template<class _Fp>
    using _EnableIfCallable = typename enable_if<__callable<_Fp>::value>::type;

    // ...
};
```

简单来说，一个`callable object`必须：

1. 不能是`std::function`自身；
2. 可以被“invoke”（好像是废话）；
3. 返回类型是`void`或者可以转换成类型`_Rp`。

知道了什么是`callable object`，我们再来看如何通过这个`callable object`构造一个`function`对象：

```c++
// file: functional

template<class _Rp, class ..._ArgTypes>
template<class _Fp, class>
function<_Rp(_ArgTypes...)>::function(_Fp __f)
    : __f_(0) {

    if (__function::__not_null(__f)) {

        typedef __function::__func<_Fp, allocator<_Fp>, _Rp(_ArgTypes...)> _FF;

        if (sizeof(_FF) <= sizeof(__buf_) && is_nothrow_copy_constructible<_Fp>::value) {
            __f_ = ::new((void*)&__buf_) _FF(std::move(__f));
        }
        else {
            typedef allocator<_FF> _Ap;
            _Ap __a;
            typedef __allocator_destructor<_Ap> _Dp;
            unique_ptr<__base, _Dp> __hold(__a.allocate(1), _Dp(__a, 1));
            ::new (__hold.get()) _FF(std::move(__f), allocator<_Fp>(__a));
            __f_ = __hold.release();
        }
    }
}

// __func implements __base

template<class _FD, class _Alloc, class _FB> class __func;

template<class Fp, class _Alloc, class _Rp, class ..._ArgTypes>
class __func<_Fp, _Alloc, _Rp(_ArgTypes...)> : public __base<_Rp(_ArgTypes...)> {
    __compressed_pair<_Fp, _Alloc> __f_;
    
    // ...
};
}
```

真正的主角终于登场了，那就是`__function::__func`，这个类实现了`__base`接口，而且有一个成员变量`__f_`，类型为`__compressed_pair<_Fp, _Alloc>`。需要说明的是：

 1. `__function::__func`的定义中出现了一个模板参数`_Alloc`，这是个`allocator`。C++标准对这个`allocator`并没有做出详细的说明，这导致了不同的标准库实现在使用这个`allocator`使出现了一些混乱，标准委员会已经决定将`_Alloc`移出C++17，所以你可以忽略这个参数。

2. `__compressed_pair`是个优化了内部存储的`pair`，功能和`std::pair`完全相同。

理解上面两点，上面代码的意思就很清楚了：如果缓冲区`__buf_`可以容纳`__function::__func`，则通过`placement new`在`__buf`里，用参数`__f_`构造出一个`__function::__func`，并且让`__f_`指向`__buf`；否则就新开辟一块内存，构造出`__function::__func`，再让`__f_`指向新开辟的内存。