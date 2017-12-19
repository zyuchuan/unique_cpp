# std::function

`std::function`是一个泛化的函数外敷类(wrapper)，它可以容纳任何的`callable object`，比如函数，lambda表达式，functor，和函数指针等等。

在分析`std::function`的源代码之前，我们先通过一个例子来看看它的用法。

```
#include <functional>
#include <iostream>

struct Foo {
    int _num;
    
    Foo(int num) : _num(num){}
    void print_add(int i) const {
        std::cout << _num + i << std::endl;
    }
};

void print_num(int i) {
    std::cout << i << std::endl;
}

struct Functor {
    void operator()(int i) const {
        std::cout << i << std::endl;
    }
};

int main() {
    // store a free function
    std::function<void(int)> print_num_fun = print_num;
    print_num_fun(3);
    
    // store a lambda
    std::function<void(int)> lambda_fun = [](int i){
        std::cout << i << std::endl;
    }
    lambda_fun();
    
}
```