# 1. Type Traits

## 1.1 什么是Trait

“Trait”在英语中特指“a particular quality of your personality”，翻译成人话就是“逼格”的意思。那C++的“逼格”又是什么呢？C++之父[Bjarne Stroustrup](http://www.stroustrup.com/index.html)对此的解释是：

> Think of a trait as a small object whose main purpose is to carry information used by another object or algorithm to determine "policy" or "implementation" details.

翻译成人话就是：

> trait是一个小型对象，它的主要目的就是携带信息，而这些信息会被其它的对象或算法使用，用来决定某个“policy”或“implementation”的细节。

还是不知所云？我再举个例子，C++标准库中有两个字符串模板类`string`和`wstring`，类的声明如下（为了方便阅读，代码中省略一些和主题无关的细节）：

```
template <class T, class Traits = char_traits<T> >
class basic_string {
public:
    typedef Traits                             traits_type;
    typedef typename traits_type::char_type    value_type;
    
    ...
    
private:
    size_type min_cap = 16;
    value_type data[min_cap];
    
    ...
};

typedef basic_string<char, char_traits<char>, allocator<char> > string;
typedef basic_string<wchar_t, char_traits<wchar_t>, allocator<wchar_t> > wstring;
```

可见`string`和`wstring`其实都是`basica_string`的特化，`string`是针对`char`的特化，而`wstring`是针对`wchart_t`的特化。

字符串都是有长度的，我们希望`basic_string`能提供一个`length()`函数，方便用户获取字符串的长度。问题是，并没有一个通用的函数能同时获取`char`类型字符串和`wchar_t`类型字符串的长度，对`char`类型字符串，获取长度的函数是`strlen(char*)`，而对`wchar_t`类型，获取长度的函数是`wcslen(wchar_t*)`。

是时候让*Trait*登场了，再看一下`basic_string`的声明：

```
template<class T, class Traits = char_traits<T> > class basic_string
```

注意第二个模板参数`Traits`，看名字似乎是一个*Trait*，而它也确实是一个*Trait*，而且这个*Trait*有个默认值`char_traits<T>`，也是个模板类。来看看`char_traits`的定义：

```
template<class T>
struct char_traits {
    typedef T    char_type;

    ...
    
    // 声明函数，实现细节在特化的模板中
    static size_t length(const char_type *c);

    ...
};

// 针对char类型特化
template<>
size_t char_traits<char> {
    ...
    
    static size_t length(const char_type *c) { return ::strlen(c); }
    
    ...
};

// 针对wchar_t类型特化
template<>
size_t char_traits<wchar_t> {
    ...
    
    static size_t length(const char_type *c) { return ::wcslen(c); }
    
    ...
};

```

可以看到`char_traits<T>::length()`这个函数针对不同的模板参数`T`，有不同的实现。对于`char`类型，调用`strlen()`计算字符串长度；对于`wchar_t`类型，调用`wcslen()`计算字符串长度。

接下来的事情就很简单了，在`basic_string`中定义函数`length()`：

```
template <class T, class Traits>
size_t basic_string::length() { return Traits::length(data); }
```

齐活！借助*Trati*技术，一个优雅的解决方案闪亮登场！

<br/>

> **注**：这里为了解释*Trait*概念，对`basic_string`的定义做了简化处理，C++标准库的实现方式与此有很大的不同，[不信就狠戳这里](http://en.cppreference.com/w/cpp/string/basic_string)。

<br/>

### 小结

相信聪明如你者已经发现：“这TM不就是设计模式中的Strategy模式吗？” 恭喜你学会抢答，*Trait*确实和Strategy模式很像，但是，

1. Strategy模式中，strategy的选择是在运行期进行的，而*Trait*是基于模板的，所以strategy的选择是在**编译期**完成的。
2. Strategy模式中，strategy的选择通常都是基于变量的值，而*Trait*是基于模板的，所以的strategy的选择是基于**变量类型**的。

一句话，*Trait*的实质就是编译期类型推导。不仅*Trait*如此，C++中所有基于模板的解析其实质都是编译期类型推导。重要的话说三遍：

**编译期类型推导！**

**编译期类型推导！**

**编译期类型推导！**

<br/>
<hr/>

## 1.2 Type Trait

*Type Trait*是C++11中引入的新功能，用于在编译期查询或者编辑类型的属性。C++11的*Type Trait*由一系列的模板类组成，全部放在头文件`<type_traits>`中。关于C++ 11 *Type Trait*的详细信息，可以参考[cppreference.com](http://en.cppreference.com/w/cpp/header/type_traits)。

本书不打算对每个type trait都一一介绍，仅选择一些高逼格的*trait*，解释其设计思路和实现原理，目的是让你能透彻了解C++ *Type Trait*，顺便膜拜一下大神们的编码技巧。

### 1.2.1 is_const

我们先从最简单的*type trait* `is_const`入手，`is_const`检查一个类型声明有没有`const`修饰符，它的用法如下：

```
 std::cout << std::boolalpha;
 std::cout << std::is_const<int>::value << '\n';         // false
 std::cout << std::is_const<const int>::value  << '\n';  // true
 std::cout << std::is_const<const int*>::value  << '\n'; // false
 std::cout << std::is_const<int* const>::value  << '\n'; // true
 std::cout << std::is_const<const int&>::value  << '\n'; // false
```

实现原理也很简单，源代码如下（省略了和主题无关的细节）：

```
// header: <type_traits>

template <class T, T v>
struct integral_constant {
    static constexpr const T    value = v;
};

typedef integral_constant<bool, true> true_type;
typedef integral_constant<bool, false> false_type;


template<class T>
struct is_const : public false_type {};

// 针对const类型的特化版本
template<class T>
struct is_const<const T> : public true_type {};
```

代码很好理解，无非就是针对`const`类型的模板特化而已，这里就不详细解释了。如果你理解起来有难度，恐怕你得补习一下C++模板知识了。


### 1.2.2 is\_class

如果要你来写一个*type trait*，判断某个类型是否是一个class或struct，比如有如下代码：

```
struct A {};
class B {};
enum class C {};

std::cout << std::boolalpha;
std::cout << is_class<A>::value << std::endl;
std::cout << is_class<B>::value << std::endl;
std::cout << is_class<C>::value << std::endl;
std::cout << is_class<int>::value << std::endl;
```

我希望输出如下：

```
true
true
false
false
```

你该怎么做？

考虑一下什么是`class`，`class`无非就是一组数据以及用以操纵这些数据的函数的集合。对于类中的数据，C++允许你定义一个指向类成员变量的指针，这是`class`所特有的属性，那可不可以针对这些特有属性，在模板特化上做文章呢？答案是肯定的，而且这也正是`is_class`的实现原理：

```
// header <type_traits>

// helper class, sizeof(two) = 2
struct two {
    char c[2];
};

namespace is_class_imp {

    // 这个函数接受一个指向类成员变量的指针为参数
    template<class T> char test(int T::*);

    // 这个函数接受任何形式的参数
    template<class T> two test(...);
}

template<class T>
struct is_class 
    : public integral_constant<bool, sizeof(is_class_imp::test<T>(0)) == 1> {}
    
```

有趣的是，`is_class`是通过函数重载实现的。上面的代码重载了函数`test`，第一个重载函数接受一个，呃...，那个“T冒号冒号星号”是啥？...`int T::*`定义了一个`int`类型的指向类成员变量的指针，也就是说函数接受一个类成员变量指针，当然也接受一个结构体成员变量指针（C++中`struct`和`class`其实是一样的）。第二个`test`是个可变参数函数，接受任意数量和类型的参数。

当编译器看到`sizeof(is_class_imp::test<T>(0))`的时候，首先需要决定匹配哪个`test`函数。如果模板参数`T`确实是一个`class`或`struct`，那`int T::*`就是合法的C++表达式。至于`T`中有没有`int`类型的成员变量，编译器根本不关心，因为编译器关心的是如何求出`sizeof`的值。所以，如果`T`是个`class`或`struct`，第一个函数匹配得更好，编译器于是用第一个函数的返回类型去求`sizeof`。

如果`T`不是一个`class`或`struct`，那`int T::*`就是一个非法的表达式，根据[SFINAE](https://en.wikipedia.org/wiki/Substitution_failure_is_not_an_error)规则，编译器不会报错，而是试着匹配第二个函数，也就是`test`的三个点版本，而三个点是可以匹配任何参数类型的，于是`sizeof(....)`的值为2，也就是说代码变成这样:

```
template<class T>
struct is_class : public integral_constant<bool, false> {};
```


### 1.2.3 common\_type

`common_type`返回所有模板参数的最大公共类型，比如说如果有

```
common_type<int, float>::type           // float，因为int可以转换成float
common_type<int, float, double>::type   // double，因为int, float都可以转换成double
```

似乎是一件很复杂的是，确实很复杂，不过如果看了源代码，你一定会大呼，原来这么简单，我怎么就想不到，你如果想到了，就扔了这本书吧，你已经不需要了。

```
// header: <type_tratis>

// 类声明，注意三个点，这说明这个类可以有任意多个模板参数
template<class ...T> struct commont_type;

// 针对只有一个模板参数的特化
template<class T>
struct common_type<T> {
    typedef typename std::decay<T>::type type;
};

// 针对两个模板参数的特化
template<class T, class U>
struct common_type<T, U> {
private:
    static T&& t();
    statuc U&& u();
    static bool f();
public:
    typedef typename std::decay<decltype(f() ? t() : u())>::type type;
};

// 针对三个或以上模板参数的特化
template<class T, class U, class ...V>
struct common_type<T, U, V...> {
    typedef typename common_type<typename common_type<T, U>::type, V...>::type type;
};
```

代码比较简单，首先声明了一个模板类，然后分别针对模板参数的个数为一个和两个的情形做了特化，对于三个以上的模板参数的情况，则用递归定义。

等等，好像哪里不对？

1. 哪里能看出来推导公共类型了？
2. 这行代码有问题: `typedef typename std::decay<decltype(f() ? t() : u())::type type`，函数`f()`根本没有定义，所以三目运算符`? :`根本没法求值。

恭喜你，你有一双火眼金睛。让我来告诉你三目运算符是怎么回事。没错，`f()`没有定义，因为编译器根本不需要，编译器在碰到这种情况是，会自动计算公共类型，然后把这个公共类型作为`decltype`的参数。

```
decltype(true ? std::declval<int>() : std::declval<double>())) // double
decltype(false ? std::declval<int>() : std::declval<double>())) // double
```

在我的XCode 8.3中，上面两行代码都输出`d`，也就是`double`。这就证明了编译器在分析三目运算符时，自动对参数进行了转换，并返回最大公共类型。

用三目运算符来推导最大公共类型，真是“奇技淫巧”啊。其实在C++11的标准库中，类似的例子还有很多，这里就不一一介绍了。另外，知乎上有一篇关于C++奇技淫巧的讨论帖子，有兴趣的读者可以[狠戳这里](https://www.zhihu.com/question/27338446)


### 1.2.4 is\_function

`is_function`检查某个类型是否是`function`，注意，`std::function`，lambda表达式，重载了`operator()`的类，以及函数指针都不能算是`function`：

```
/*
 Sample code comes from
 http://en.cppreference.com/w/cpp/types/is_function
*/

strcut A {
    int fun();
};

template<typename T>
struct PM_traits{};

template<class T, class U>
struct PM_traits<U T::*> {
    using member_type = U;
}

int f();

std::cout << std::boolalpha;
std::cout << is_function<A>::value << std::endl;            // false
std::cout << is_function<int(int)>::value << std::endl;     // true
std::cout << is_function<decltype(f)>::value << std::endl;  // true
std::cout << is_function<int>::value << std::endl;          // false

using T = PM_traits<decltype(&A::fun)>::member_type;
std::cout << is_function<T>::value << std::endl;            // false

```

```
namspace libcpp_is_function_imp {
    struct dummy_type{};
    template<calss T> char    test(T*);
    template<class T> char    test(dummy_type);
    template<class T> two     test(...);
    
    template<calss T> T&     source(int);
    template<class T> dummy_type source(...);
}

template<class T, bool = is_class<T>::value ||
                         is_union<T>::value ||
                         is_void<T>::value  ||
                         is_reference<T>::value ||
                         is_nullptr_t<T>::value>
struct libcpp_is_function : public integral_const<bool,     
      sizeof(libcpp_is_function_imp::test<T>(libcpp_is_function_imp::source<T>(0))) == 1>
{};

template<class T>
struct libcpp_is_function<Tp, true> : public false_type {};

template<class T>
struct is_function : public libcpp_is_function<T> {};

```

这段代码比较难懂，需要详细解释一下：

1. 如果你对一个`class`, `union`, `void`, `reference`或`null pointer`，执行`is_function`操作，此时`libcpp_is_function`的第二个模板参数为`true`，而针对这种情况定义了一个特化版本，该特化版本继承于`false_type`，这是我们需要的结果。

2. 除去第一种情况，编译器会激活非特化版本，此时编译器会对模板类`integral_const`的第二个模板参数进行类型推导：
2.1 如果`T`是一个function对象，`T& source(int)`会被解析，并返回这个函数对象类型，
2.2 如果`T`不是一个function对象，`T& source(int)`还是会被解析，但是此时的返回值是`T&`，然后`test(...)`会被解析，`sizeof`返回2，
2.3 有的编译器可能不接受返回函数对象，此时`source(...)`会被解析

值得注意的是，代码中只是声明了函数，并没有定义，因为不需要。在做类型推导的时候，编译器不需要知道函数的定义，只需要知道函数的返回值就可以了。


### 1.2.5 is\_member\_function\_pointer

```
template<class T> struct libcpp_is_member_function_pointer 
    : public false_type {}

template<class Ret, class Kls> struct libcpp_is_member_function_pointer<Ret, Kls::*>
    : public is_function<Ret> {};

template<class T> struct is_member_function_pointer
    : public libcpp_is_member_function_pointer<typename remove_cv<T>::type>::type {};
```

### 1.2.6 is\_polymorphic

```
template<typename T> char& is_polymorphic_impl(
    typename enable_if<sizeof((T*)dynamic_cast<const volatile void*>(declval<T*>())) != 0, int>::type>;

template<typename T> struct two& is_polymorphic_impl(...);

template<class T> struct is_polymorphic
    : public integral_constant<bool, sizeof(is_polymorphic_impl<T>(0)) == 1>{};
```