# C++ 模板元编程常用手法

## 1. 关于C++模板元编程

**模板元编程**，也就是**[Template Metaprogramming](https://en.wikipedia.org/wiki/Template_metaprogramming)**、或**TPM**，是每一个有追求的C++程序员必须掌握的编程范式之一。

## 2. 编译期常数

我们知道，在使用模板元编程时，是不能使用`true`, `false`这类的布尔型常量的，因为这是运行时常量。同样，`1`, `2`也是运行时常数，也不能在编译器使用。所以我们必需想办法，把整形和布尔型常量绑定到一个`type`上面，这样编译器才有可能在编译器使用这些常量完成计算。我们来看怎么做：

### 2.1 整形常数

```
// file: type_traits

template <typename T, T v>
struct integral_constant {
    static constexpr T value =    v;
    typedef T                     value_type;
    typedef integral_constant     type;
    constexpr operator value_type() const noexcept {return value;}
    constexpr value_type operator()() const noexcept {return value;} //since C++14
};
```

上面的代码定义了一个`const integer`类型，它有一个静态类成员常量`value`，还有一个`typedef`的`type`。这是C++标准库的一个管理：如果一个模板类有静态常量，则用`value`表示，如果有type的，则用`type`表示，例如：

```
template<typename T>
struct my_meta_class {
    static constexpr int value = 0;
    typedef T            type;
};
```

这不是强制要求，但是是一个惯例，你写代码的时候，建议也遵守这个惯例。

`integral_const`的用法比较简单

```
typedef integral_const<int, 2> two_type;
typedef integral_const<int, 4> four_type;

static_assert(two_type::value * 2 == four_type::value);
```

### 2.2 布尔型常数

标准库还定义`true_type`和`false_type`，这两个类的作用和布尔型常量`true`和`false`一样

```
// file: type_traits

typedef std::integral_constant<bool, true>  true_type;
typedef std::integral constant<bool, false> false_type;
```

## 3. 递归

在模板元编程中，常用递归的手法定义，一个经典的例子是计算阶乘，比如

```
5! = 5 * 4 * 3 * 2 * 1
```

用模板元来表示

```
template<size_t N>
struct factorial {
    static constexpr size_t value = N * factorial<N-1>;
};

template<>
struct factorial<1> {
    static constexpr size_t value = 1;
};

int main() {
    std::cout << factorial<5>::value << std::endl; // 20
}

```

我们知道递归必须要有结束条件，在模板元编程中，通常都用一个特化来作为结束条件，比如上面代码中的`struct factorial<1>`就是递归的结束。

## 3. SFINAE

SFINA代表**S**ubstitution **F**ailure **I**s **N**ot **A**n **E**rror，