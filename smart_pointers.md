# Smart Pointers

Smart pointer，也就是所谓的“智能指针”，是指那些能够自动不过在C++ 11之前，能用的智能指针都在boost库中，这次C++标准委员会终于决定将之加入到标准库中。

C++标准库提供了四种智能指针：

* unique_ptr
* shared_ptr
* weak_ptr
* auto_ptr

`auto_ptr`自C++98其就存在了，但是很少有人用，所以就不讲了

## 1. unique_ptr

顾名思义，`unique_ptr`就是独有一份，在两种情况下，其管理的指针会被处理掉：

1. 指针被摧毁
2. 调用`operator=`或`reset()`的时候。

标准库中定义了两种类型的`unique_ptr`：

1. 管理单个指针，指针通过`new`获得
2. 管理指针数组，指针通过`new[]`获得。

### 1.1 unique_ptr的源代码

```
// file: <memory>

template<class _Tp, class _Dp = default_delete<_Tp> >
class unique_ptr {
public:
    typedef _Tp element_type;
    typedef _Dp deleter_type;
    typedef typename __pointer_type<_Tp, deleter_type>::type pointer;
    
private:
    __compressed_pair<pointer, deleter_type> __ptr_;
    
    // ...
};
```

`unique_ptr`可以指定一个`deleter`，如果没有指定，则用默认的`default_delete`，`default_delete`就是针对`delete`的一个函数对象

```
template<class T>
struct default_delete {
    void operator()(T* ptr) const noexcept {
        delete ptr;
    }
};
```

`unique_ptr`将参数保存在一个`__compressed_pair`中，前面已经说过，`__compressed_pair`是一个针对空基类优化的`pair`，大家将它当做一个`std::pair`就好。

下面我们来看如何构造一个`unique_ptr`，其实`unique_ptr`的构造函数比较简单，比如说

```
// file: memory

template<class _Tp, class _Dp = default_delete<_Tp> >
class unique_ptr {
    // ...

public:
    inline constexpr unique_ptr() noexcept {
        : __ptr_(pointer()) {
    }
    
    inline constexpr unique_ptr(nullptr_t) noexcept 
        : __ptr_(pointer()) {
    }
    
    inline explicit unique_ptr(pointer __p) noexcept 
        : __ptr_(std::move(__p)){ 
    
    }
    
    inline unique_ptr<unique_ptr&& __u) noexcept
        : __ptr_(__u.release(), 
        std::forward<deleter_type>(__u.get_deleter())) {}
        
    inline unique_ptr& operator=(unique_ptr&& __u) noexcept {
        reset(__u.release());
        __ptr_.second() = std::forward<deleter_type>(__u.get_deleter());
        return *this;
    }
    
    inline ~unique_ptr(){reset();}
    
    // ...
};
```

指针的释放操作都在函数`reset()`和`release()`中。

C++标准明确规定`release()`的返回原生指针并且放弃所有权。`reset()`的功能是替换`managed object`：

```
// file: memory

template<class _Tp, class _Dp = default_delete<_Tp> >
class unique_ptr {
    // ...
    
public:
    inline pointer release() noexcept {
        pointer __t = __ptr_.first();
        __ptr_.first() = pointer();
        return __t;
    }
    
    inline void reset(pointer __p = pointer()) noexcept {
        pointer __tmp = __ptr_.first();
        __ptr_.first() = __p;
        if (__tmp)
            __ptr_.second()(__tmp);
    }
```

这几乎就是`unique_ptr`的全部源代码了，除了一点，目前为止，`unique_ptr`还不像个指针，因为缺少两个最主要的方法：

```
// file: memory

template<class _Tp, class _Dp = default_delete<_Tp> >
class unique_ptr {
    // ...
    
public:
    inline add_lvalue_reference<_Tp>::type operator*() const {
        return *__ptr_.first();
    }
    
    inline pointer operator->() const noexcept {
        return __ptr_.first();
    }
```

`unique_ptr`的源代码分析就到此为止，标准库中还有一个针对指针数组的`unique_ptr`，实现方式大同小异。

## 2. shared_ptr

`shared_ptr`的实现比`unique_ptr`的实现要复杂很多，不过也不是很复杂。我们还是先从声明入手：

```
// file: memory

template<class _Tp>
class shared_ptr {
public:
    typedef _Tp element_type;
    
private:
    element_type *__ptr_;
    __shared_weak_count* __cntrl_;
    
    // ...
};
```

可见`shared_ptr`内部除了维护原生指针`__ptr_`，还用一个类型为`__shared_weak_count*`的变量`__cntrl_`，不难想象，这一定是维护引用计数的对象：

```
file: memory

class __shared_count {
    // not copy constructible and not assignable
    __shared_count(const __shared_count&);
    __shared_count& operator=(const __shared_count&);
    
protected:
    long __shared_owners_; // how many owners do I have?
    virtual ~__shared_count();
    
public:
    explicit __shared_count(long __refs = 0) noexcept 
        : __shared_owners(__refs){}
        
        void __add_shared() noexcept;
        bool __release_shared() noexcept;
};

class __shared_weak_count : private __shared_count {
    long __shared_weak_owners_;
    
public:
    explicit __shared_weak_count(long __refs = 0) noexcept {
        : __shared_count(__refs), 
          __shared_weak_owners(__refs) {}
          
protected:
    virtual ~__shared_weak_count();
    
public:
    void __add_shared() noexcept;
    void __add_weak() noexcept;
    void __release_shared() noexcept;
    void __release_weak() noexcept;
    long use_count() const noexcept { return __shared_count::use_count();}
    
private:
    virtual void __on_zero_shared_weak() noexcept = 0;
};

```

 显然，`__shared_weak_count`是个虚基类，从标准库的源代码中我们可以看到，这个虚基类同时用于`shared_ptr`和`weak_ptr`，所以我们它声明了`xxx_shared()`函数和`xxx_weak()`函数，这种做法值得商榷。

虚基类的作用类似于接口，是没法直接使用的，所以还必须定义一个实在类：

```
template<class _Tp, class _Dp, class _Alloc>
class __shared_ptr_pointer : public __shared_weak_count {
    __compressed_pair<__compressed_par<_Tp, _Dp>, _Alloc> __data_;
    
public:
    inline __shared_ptr_pointer(_Tp __P, _Dp __d, _Alloc __a)
        : __data_(__compressed_pair<_Tp, _Dp>(__p, std::move(__d)), std::move(__a)) {}
        
    // ...

};
```

到此为止，辅助工作已经差不多了，主角该登场了


```
// file: memory

template<class _Tp>>
class shared_ptr {
public:
    typedef _Tp element_type;
    
private:
    element_type*           __ptr_;
    __shared_weak_count*    __cntrl_;
    
    struct __nat{int __for_bool_;}; // placeholder
    
    // ...
};
```

`shared_ptr`内部维护了两个变量，一个原生指针`__ptr_`和一个操控引用计数的变量`__cntrl_`。我们来看这两个变量是如何被使用的

```
template<class _Tp>
template<class _Yp>
shared_ptr<_Tp>::shared_ptr(_Yp* __p,
        typename enable_if<is_convertible<_Yp*, element_type*>::value, __nat>::type)
        : __ptr_(__p) {
        unique_ptr<_Yp> __hold(__p);
        typedef __shared_ptr_pointer<_Yp*, default_delete<_Yp>, allocator<_Yp> >_CntrBlk;
        __cntrl_ = new _CntrBlk(__p, default_delete<_Yp>(), allocator<_Yp>());
        __hold.release();
}
```

我们知道`shared_ptr`的特性就是`share`，我们来看share是怎样工作的

```
// copy constructor, increment reference count
template<class _Tp>
inline shared_ptr<Tp>::shared_ptr(const shared_ptr& __r) noexcept
    : __ptr(__r.__ptr_), __cntrl_(__r.__cntrl_) {
    if (__cntrl_)
        __cntrl_->__add_shared();
}

// move constructor, does't increment reference count
template<class _Tp>
inline shared_ptr<T>::shared_ptr(shared_ptr&& __r) noexcept
    : __ptr_(__r.__ptr_), __cntrl_(__r.__cntrl) {
        __r.__ptr_ = 0;
        __r.__cntrl_ = 0;
}

template<class _Tp>
shared_ptr<_Tp>::~shared_ptr(){
    if (__cntrl_)
        __cntrl_->__release_shared();
}
```

我们已经看到了，`shared_ptr`内部维护了两个指针，如果你直接调用构造函数，

```
class Widget;

auto sp = shared_ptr<Widget>(new Widget());
```

这里实际分配了两次内存，第一次是`new Widget()`，在构造`shared_ptr`的时候，还会分配一次。所以标准库还提供了`make_shared()`函数，让你一次分配全部所需的内存：

```
template<class _Tp, class ..._Args>
inline
typename enable_if<!is_array<_Tp>::value, shared_ptr<_Tp>>::type
make_shared(_Args&& ...__args) {
    return shared_ptr<_Tp>::make_shared(std::forward<_Args>(__args)...);
}

template<class _Tp>
template<class ...Args>
shared_ptr<_Tp> shared_ptr<_Tp>::make_shared(_Args&& ...__args) {
    typedef __shared_ptr_emplace<_Tp, allocator<_Tp>>_CntrlBlk;
    typedef allocator<_CntrlBlk> _A2;
    typedef __alocator_destructor<_A2> _D2;
    
    _A2 __a2;
    unique_ptr<_CntrlBlk, _D2> __hold2(__a2.allocate(1), _D2(__a2, 1));
    ::new(__hold2.get()) _CntrlBlk(__a2, std::forward<_Args>(_args)...);
    shared_ptr<_Tp> __r;
    __r.__ptr_ = __hold2.get()->get();
    __r.__cntrl_ = __hold2.release();
    return __r;
}
```

```
template<class _Tp, class _Alloc>
class __shared_ptr_emplace : public __shared_weak_count {
    __compressed_pair<_Alloc, _Tp> __data_;
    
public:
    template<class ..._Args>
    __shared_ptr_emplace(_Alloc __a, _Args&& ...__args)
        : __data_(piecewise_construct, std::forward_as_typle(__a),
        std::forward_as_tuple(std::forward<_Args>(__args)...)){}
        
    // ...
};
```