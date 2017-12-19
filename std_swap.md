# std::swap()：简约而不简单


`std::swap()`是一个很简单的函数：交换两个参数的值，仅此而已。但是这个看似平淡无奇的函数，背后的故事却不简单。不知道你考虑过下面几个问题没有：

1. 我们通常要求这个函数不能抛出异常，为什么？
2. 如何让`swap`函数能高效地工作与用户自定义类型？
3. 如果你自己定义了一个类，如何写一个高效的swap函数？
4. 如果要求swap函数能工作于多线程环境，该怎么做？

```
// file: type_traits

template<class T>
typename enable_if
<
  // T must be "move constructible" and "move assignable"
  is_move_constructible<T>::value &&
  is_move_assignable<T>::value
>::type
swap(T& x, T& y) noexcept( 
                  // if T's move constructor and move
                  // assignment does't not throw, 
                  //swap() does't throw
                  is_nothrow_move_constructible<T>::value && 
                  is_nothrow_move_assignable<T>::value) {
                  
  T t(std::move(x));
  x = std::move(y);
  y = std::move(t);
}
```

这个泛化的版本可以工作，但是不能高效地工作，标准库为每一种类型都重载了`swap()`函数，而这些个重载版本的写法也大有讲究，比如说，`std::vector`的重载版本如下：

```
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