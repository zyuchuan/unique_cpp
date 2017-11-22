# C++ 模板元编程常用手法

## 1. 关于C++模板元编程

**模板元编程**，也就是**[Template Metaprogramming](https://en.wikipedia.org/wiki/Template_metaprogramming)**、或**TPM**，是每一个有追求的C++程序员必须掌握的编程范式之一。

## 2. 编译期常数

我们知道，在使用模板元编程时，是不能使用`true`, `false`这类的布尔型常量的，因为这是运行时常量。同样，`1`, `2`也是运行时常数，也不能在编译器使用。所以我们必需想办法，把整形和布尔型常量绑定到一个`type`上面，这样编译器才有可能在编译器使用这些常量完成计算。我们来看怎么做：

```
// file: type_traits

template <class T, T v>
struct integral_constant {
    static constexpr T value =    v;
    typedef T                     value_type;
    typedef integral_constant     type;
    constexpr operator value_type() const noexcept {return value;}
    constexpr value_type operator()() const noexcept {
        return value;
    } //since c++14
};
```