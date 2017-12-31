# std::swap()


`std::swap()`是一个很简单的函数：交换两个参数的值，仅此而已。但是这个看似平淡无奇的函数，背后的故事却不简单。不知道你考虑过下面问题没有：

1. 我们通常要求这个函数不能抛出异常，为什么？
2. 如何才能让`std::swap`函数能高效地作用于你自己定义的类？

要回答这两个问题，就要从`std::swap`的实现说起。

## 1. std::swap()的实现

libc++中，`swap()`定义在文件“type_traits”中：

```
// file: type_traits

template<class T>
typename enable_if<
  is_move_constructible<T>::value &&
  is_move_assignable<T>::value
>::type
swap(T& x, T& y) noexcept( 
            is_nothrow_move_constructible<T>::value && 
            is_nothrow_move_assignable<T>::value) {      
  T t(std::move(x));
  x = std::move(y);
  y = std::move(t);
}
```

从定义可以看出：

1. 一个“swappable”的类型必须满足“move constructible”和“move assignable”两个前提条件，这是C++11的新要求。

2. 如果类型`T`的“move constructor”和“move assignment operator”不抛出异常，则`swap`也不能抛出异常。

虽然标准并没有强制规定`swap()`不能抛出异常，但是实际使用中，我们都要确保`swap()`不抛出异常，这是为什么呢？要回答这个问题，就要从**C++异常安全保证**说起。

## 2 异常安全保证

维基百科对[异常安全保证](https://en.wikipedia.org/wiki/Exception_safety)（Exception Safety Guarantees）的解释是“***类的设计者和使用者在使用任何一门程序设计语言，特别是C++时可以遵守的一系列异常处理准则***”。

异常安全保证从弱到强可以分为三个等级：

* **基本保证（Basic Exception Safety）**: 也叫**无泄漏保证（No-Leak Guarantee）**，即发生异常时不会导致资源泄露（比如内存泄露），程序内的任何事物仍然保持在有效状态下，没有对象或数据结构会因此而破坏，所有对象都处于有效的状态，但是处于哪个状态不可预知。

* **强烈保证（Strong Exception Safety）**：如果抛出异常，程序状态不改变。就像数据库中的事务处理一样，要么成功，如果不成功，则程序回到调用之前的状态。

* **不抛出异常保证（No-Throw Guarantee）**：承诺绝不抛出异常。如果有异常发生，会在内部处理，保证不让异常逃逸。

我们希望所有函数都提供“no throw”保证，但这是不可能的。我们最低也要做到基本承诺。而且，你会看到，在一定条件下，我们可以做到强烈保证。而swap`函数可以很容易地做到这一点。



// file: vector

template<class T, class Allocator>
class vector {
public:

    // define a member function swpa(vector&)
    void swap(vector& x) {
        // doing swap here 
    }    
};

// overload swap for vector<T, Allocator>
template<class T, class Allocator>
inline void swap(vector<T, Allocator>& x, vector<T, Allocator>& y) {
    x.swap(y);
}
```

首先，`vector`定义了一个成员函数`swap`，然后再重载函数`std::swap`，这个重载的版本会调用`vector::swap`。

注意，根据《Effective C++》，在重载的版本中，应该

```
template<class T, class Allocator>
inline void swap(vector<T, Allocator>& x, vector<T, Allocator>& y) {
    using std::swap;
    x.swap(y);
}
```

如此大费周折，只是为了实现一个平淡无奇的函数，看上去得不偿失。其实不然，`swap`函数已经成为C++异常安全编程的很重要的组成部分。

## 2 异常安全保证

* **基本承诺**: 如果抛出异常，程序内的任何事物仍然保持在有效状态下，没有对象或数据结构会因此而破坏，所有对象都处于有效的状态，但是处于哪个状态不可预知。

* **强烈保证**：如果抛出异常，程序状态不改变。也就是说函数是一种transaction，要么成功，如果不成功，则程序回到调用之前的状态。

* **不抛出异常保证**：承诺绝不抛出异常。

我们希望所有函数都提供“no throw”保证，但这是不可能的。我们最低也要做到基本承诺。而且，你会看到，在一定条件下，我们可以做到强烈保证。而swap`函数可以很容易地做到这一点。

## 3 copy-and-swap

```
class Widget {
public:
    Widget(const Widget& other);
    // ...
};

Widget::Widget(const Widget& other) {
    // construction of temp could throw,
    // if it throw, this object would change
    // status
    Widget temp(other);
    
    // swap never throw, so we provide
    // strong exception guarent
    swap(*this, temp);
}
```