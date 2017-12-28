
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
};
```