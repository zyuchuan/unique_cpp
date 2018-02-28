# Type Traits

**_Trait_**在英语中特指**_a particular quality of your personality_**，大致相当于汉语中的**逼格**的意思。那C++的**逼格**又是什么呢？C++之父[Bjarne Stroustrup]( http://www.stroustrup.com/index.html) 对此的解释是：

>Think of a trait as a small object whose main purpose is to carry information used by another object or algorithm to determine _policy_ or _implementation_ details.

翻译成人话就是：

> _trait_ 是一个小型对象，它的主要目的就是携带信息，而这些信息会被其它的对象或算法使用，用来决定某个 _policy_ 或 _implementation_ 的细节。

**_Trait_**在标准库中大量运用，比如我们熟悉的C++ 98 STL中的`iterator_traits`、`char_traits`等。C++ 11又增加了*type trait*。

## C++ 11 Type Traits

C++ 11标准库定义了了大量的 _type trait_ ，全部放在头文件 `<type_traits>` 中。这些*trait*大致可以分为一下几类：

* Primary type categories: `is_void`, `is_null_pointer`, `is_class`, `is_function`...
* Composite type categories: `is_fundamental`, `is_arithmetic`, `is_object`...
* Type properties: `is_const`, `is_volatile`, `is_trivival`
* Supported operations: 
* Property quiries
* Type relationships
* Const-volatility specifiers
* Reference
* Pointers
* Sign modifiers
* Arrays
* 


关于C++ 11 _type trait_ 的详细信息，可以参考 [cppreference.com](http://en.cppreference.com/w/cpp/header/type_traits)。

由于篇幅关系，本书只能选择一些有代表性的 _trait_，解释其设计思路和实现原理，目的是让你了解C++ _type trait_ 的编写方法，顺便膜拜一下大神们的编码技巧。

<br/>

### is_const

我们先从最简单的**trait** ``is_const``入手，``is_const``检查一个类型声明有没有``const``修饰符，它的用法如下：

```c++
 std::cout << std::boolalpha;
 std::cout << std::is_const<int>::value << '\n';         // false
 std::cout << std::is_const<const int>::value  << '\n';  // true
 std::cout << std::is_const<const int*>::value  << '\n'; // false
 std::cout << std::is_const<int* const>::value  << '\n'; // true
 std::cout << std::is_const<const int&>::value  << '\n'; // false
```

实现原理也很简单，源代码如下（省略了和主题无关的细节）：

```C++
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

代码很好理解，无非就是针对``const``类型的模板特化而已，这里就不详细解释了。如果你理解起来有难度，恐怕得补习一下C++模板知识了。

### is_class

如果要你来写一个**trait**，判断某个类型是否是一_class_或_struct_，比如有如下代码：

```c++
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

```c++
true
true
false
false
```

你该怎么做？

有点晕菜是不是？考虑一下什么是_class_?_class_无非就是一组数据以及用以操纵这些数据的函数的集合。对于类中的数据，C++允许你定义一个指向类成员变量的指针，这是_class_所特有的属性，那可不可以针对这些特有属性，在模板特化上做文章呢？答案是肯定的，而且这也正是``is_class``的实现原理：

```c++
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
    : public integral_constant<bool, sizeof(is_class_imp::test<T>(0)) == 1> {};
```

上面的代码重载了函数`test`，第一个重载函数接受一个，呃...，那个“T冒号冒号星号”是啥？...`int T::*`定义了一个`int`类型的指向类成员变量的指针，也就是说函数接受一个类成员变量指针作为参数，当然也接受一个结构体成员变量指针（C++中`struct`和`class`其实是一样的）作为参数。第二个`test`是个可变参数函数，接受任意数量和类型的参数。

当编译器看到`sizeof(is_class_imp::test<T>(0))`的时候，首先需要决定匹配哪个`test`函数。如果模板参数`T`确实是一个`class`或``struct``，那``int T::*``就是合法的C++表达式。至于``T``中有没有``int``类型的成员变量，编译器根本不关心。

等等！你又发现了问题，“``test``函数只有声明，没有定义，没有定义的函数该怎么编译？” 答案是根本不需要，编译器关心的是如何求出表达式``sizeof(...)``的值，而求解``sizeof(...)``只需要知道``is_class_imp::test<T>(0)``的返回类型，不需要看到函数的定义。所以如果``T``是个``class``或``struct``，那``int t::*``就是合法的类型定义，且精确匹配第一个重载函数，于是编译器用第一个函数的返回类型去求``sizeof``，于是``is_class``的声明就会被替换成

[source,c++]
----
template<class T>
struct is_class : public integral_constant<bool, true> {};
----

如果``T``不是一个``class``或``struct``，那``int T::*``就是一个非法的类型定义，根据**SFINAE**规则，编译器不会报错，而是试着匹配第二个重载函数，也就是``test``的三个点版本，而这个版本是可以匹配任何参数类型的，``is_class``的声明会被替换成

[source,c++]
----
template<class T>
struct is_class : public integral_constant<bool, false> {};
----

看到这里，相信你已经明白了``is_class``的实现原理，无非就是利用了重载函数的匹配规则而已。值得注意的是，上面代码中的``test``函数只有声明，没有定义。其实文件``type_traits``中声明了众多的辅助函数，却没有一个定义，因为根本不需要。正如前面反复强调的，编译器只是在做类型推导，唯一需要知道的就是参数类型和返回类型，至于有没有定义，编译器完全不关心。

=== common_type

``common_type``返回所有模板参数的最大公共类型，比如

[source,c++]
----
common_type<int, float>::type           // float，因为int可以转换成float
common_type<int, float, double>::type   // double，因为int, float都可以转换成double
----

这似乎是一件很复杂的事。确实很复杂，不过我们有一个巧妙的方法可以化繁为简，先看源代码：

[source,c++]
----
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
----

代码比较简单，首先声明了一个模板类，然后分别针对模板参数的个数为一个和两个的情形做了特化，对于三个以上的模板参数的情况，则用递归的方法定义。

好像哪里不对？

1. 哪里能看出来推导公共类型了？
2. 这行代码有问题: ``typedef typename std::decay<decltype(f() ? t() : u())::type type``，函数``f()``根本没有定义，所以三目运算符``? :``根本没法求值。

恭喜你，你有一只火眼金睛（另一只不是，所以看不到代码的精妙之处）。让我来告诉你怎么回事，这两个问题其实是一个问题。我们先从``f() ? t() : u()``说起，我再说一遍，编译器在解析模板时，做的是类型推导，所以``f()``根本不需要定义（即使有定义，编译器也不知道返回值是``true``还是``false``，只有到运行时才知道）。那问题又来了，不知道``f()``的返回值，编译器该如何求解三目运算符呢？答案还是不需要，编译器此时需要知道的是三目运算符的返回类型（而不是返回值），以满足解析``decltype(...)``的需要。问题是，不知道返回值，返回类型也无从谈起。似乎编译进入了死胡同，别急，C++编译器是你的贴心小棉袄，它会尽一切可能编译你的代码，为了让编译进行下去，编译器会自动检查冒号两边的类型，尽可能将其中一个类型转换为另一个类型，并将这个类型作为三目表达式的返回类型，传入``decltype(...)``中。如果你还有疑问，可以做一个简单的测试：

[source,c++]
----
std::cout << 
    typeid(decltype(true ? std::declval<int>() : std::declval<double>())).name() << std::endl;  // double

std::cout << 
    typeid(decltype(false ? std::declval<int>() : std::declval<double>())).name() << std::endl; // double
----

在我的XCode 9.2中，上面两行代码都输出``d``，也就是``double``。这就证明了编译器在三目表达式时，自动对参数类型进行了转换，并返回最大公共类型。

用三目运算符来推导最大公共类型，我只能用“顶（sang）礼（xin）膜（bing）拜（kuang）”来形容。在C++11的标准库中，类似的使用“奇技淫巧”例子还有很多，这里就不一一介绍了。知乎上有一篇关于C++“奇技淫巧”的讨论帖子，有兴趣的同学可以 https://www.zhihu.com/question/27338446[狠戳这里].

=== is_function

最后来一道硬菜：``is_function``。``is_function``检查某个类型是否是``function``。注意，``is_function``不能用于检查``std::function``，lambda表达式，重载了``operator()``的类，以及函数指针。

[source,c++]
----
// Sample code comes from http://en.cppreference.com/w/cpp/types/is_function

strcut A { int fun(); };

template<typename T> struct PM_traits{};

template<class T, class U>
struct PM_traits<U T::*> {
    using member_type = U;
}

int f();

std::cout << std::boolalpha;

// 1. A是个class，不是function;
std::cout << is_function<A>::value << std::endl;            // false

// 2. int(int)表示一个以int为参数，并返回int的function类型；
std::cout << is_function<int(int)>::value << std::endl;     // true

// 3. f是个function的名字，decltype(f)是个function类型
std::cout << is_function<decltype(f)>::value << std::endl;  // true

// 4. 显然int不是一个function
std::cout << is_function<int>::value << std::endl;          // false

// 5. T被解析成 int()，是个function
using T = PM_traits<decltype(&A::fun)>::member_type;
std::cout << is_function<T>::value << std::endl;            // true
----

是不是觉得很神奇？我们来看一下源代码：

[source,c++]
----
namspace libcpp_is_function_imp {
    template<calss T> char    test(T*);
    template<class T> two     test(...);
    template<calss T> T&     source(int);
}

// 如果T是class, union, void, reference或null pointer,
// 则第二个模板参数的值为true，而针对这种情况，有一个特化的版本
template<class T, bool = is_class<T>::value ||
                         is_union<T>::value ||
                         is_void<T>::value  ||
                         is_reference<T>::value ||
                         is_nullptr_t<T>::value>
struct libcpp_is_function : public integral_const<bool,     
      sizeof(libcpp_is_function_imp::test<T>(libcpp_is_function_imp::source<T>(0))) == 1>
{};

// 针对class, union, void, reference和null pointer的特化版本
template<class T>
struct libcpp_is_function<Tp, true> : public false_type {};

template<class T>
struct is_function : public libcpp_is_function<T> {};
----

这段代码比较难懂，需要详细解释一下：

如果你对一个``class``, ``union``, ``void``, ``reference``或``null pointer``，执行``is_function``操作，此时``libcpp_is_function``的第二个模板参数为``true``，而针对这种情况定义了一个特化版本，该特化版本继承于``false_type``，这是我们需要的结果。

除去第一种情况，编译器会激活非特化版本，此时编译器会对模板类``integral_const``的第二个模板参数进行类型推导：

如果``T``是一个_function_对象，比如``void(void)``，则``libcpp_is_function_imp::source<T>(0))``的返回值为``void(void)&``。在编译器眼里，函数对象和函数指针是一种类型，也就是说``void(void)``和``void(\*)(void)``是一种类型，编译器于是会匹配参数为``T*``的重载版本``test(T*)``，于是，``sizeof(...)``表达式被替换成``sizeof(test<void(void)>(void(*)(void))``，进而替换成``sizeof(char)``，最终，类的声明被替换成：

[source,c++]
----
template<class T>
struct libcpp_is_function : public integral_const<bool, true> {};
----

这也是我们需要的结果。
    
如果``T``不是一个_function_对象，比如为``int``，这时``source``函数的返回类型为``int&``。由于``int&``和``int*``不是同一个类型，编译器只能匹配``test(...)``函数，于是类的声明就成了：

[source,c++]
----
template<class T>
struct libcpp_is_function : public integral_const<bool, false>
----

这仍然是我们需要的结果。

C++11标准库定义的_type trait_还有很多，这里就不一一介绍了。总的来说，这些_type traits_都是基于模板特化和函数重载，利用编译器的类型推导能力，做一些“神奇”的事。因为所有这些都是在编译期进行了，所以对运行期完全没有冲击，完全不必担心效率问题。

== 自己动手写一个Type Trait

下面我们自己动手，写一个_trait_ ``has_to_string``，我们希望达到如下的效果：

[source,c++]
----
struct A {
    std::string to_string();
};

struct B {

}

std::cout << has_to_string<A>::value << std::endl; // 1
std::cout << has_to_string<B>::value << std::endl; // 0
----

这里给出一种可能的实现：

[source,c++]
----
template<typename T, typename = std::string>
struct has_to_string : std::false_type {};

template<typename T>
struct has_to_string : decltype(std::declval<T>().to_string())> : std::true_type {};
----