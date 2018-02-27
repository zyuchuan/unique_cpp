# C++模板元编程

**元程序**一词来源于英文单词**_metaprogram_**。在英语中，**_metaprogram_**的意思是**_a program about a program_**，翻译过来就是**程序的程序**。说白了，**元程序**就是**用于操纵代码的程序**。

这听上去多少有点玄妙，其实你没少和元程序打交道，你用的C++编译器就是一个元程序，它操纵C++代码来生成汇编语言或机器码。

C++元编程不是被发明出来的，而是被无意中发现的。上个世纪九十年代末，某些爱钻研的小伙伴无意中发现C++模板可以用于编写元程序，刚开始他们只是编写元程序在编译期执行一些数值计算，后来更进一步发现除了数值计算，还可以通过模板在编译期执行类型计算。更进一步的研究发现，编译期类型计算有强大的威力，可以极大地提高代码的运行效率，于是模板元编程作为一种**_programming paradigm_**在C++社区流行开来。

促使模板元编程大行其道的最重要的原因就是效率。后面我们会看到，在模板元编程中，一些运行期的工作被转移到了编译期，也就是说程序在运行时会跑得更快。没有什么能阻止C++程序员对效率的向往，虽然模板元程序的源代码如天书般难懂，但是C++社区还是热情地拥抱这一新的**_programming paradigm_**，大量采用元编程的类库被开发出来，而C++ 11标准库几乎全部构建在元编程基础之上。所以，要看懂标准库源代码，首先就要懂一点元编程。

## C++模板元编程常用技巧

元程序是在编译期由编译器直接解析并执行的。在编译期，编译器只能做整数值计算和类型计算，这就导致了元程序的代码结构和我们熟知的代码结构，即运行时代码结构，有很大区别：

* 运行时代码结构通常包括常量、变量、数据结构、类、函数、循环，分支等；
* 元程序中只能出现常量（包括整形常量和布尔型常量）、循环和分支结构。

<br/>

### 常量

元程序中可以出现的常量包括**整形常量**和**布尔型常量**，为了配合编译器强大的类型计算能力，整形常量通常都被定义为某种类型，这就是C++ 11中的`integral_constant`：

#### 整形常量

```c++
// file: type_traits

template <typename T, T v>
struct integral_constant {
    static constexpr T value =    v;
    typedef T                     value_type;
    typedef integral_constant     type;
    
    // ...
};
```

`integral_constant`的定义虽然很简单，但是意义却很重要。它将一个数值包装成为一个对象，这从一个侧面揭示模板元编程的真谛：**编译期类型计算**。

`integral_constant`的使用方法很简单，比如你可以这样写：

```c++
typedef integral_const<int, 2> two_type;
typedef integral_const<int, 6> six_type;

static_assert(two_type::value * 3 == six_type::value, "2*3 != 6");
```

当然，上面的代码并不必直接写

```c++
static_assert(2 * 3 == 6, "2*3 != 6");
```

更有意义，甚至更繁琐。这里只是举个例子说明它的用法，后面我们会看到`integral_constant`的各种用法。

#### 布尔型常量

同整形常量一样，布尔型常量也有对应的元类型：`true_type`和`false_type`，这两个类其实是`integral_constant`的`typedef`：

```c++
// file: type_traits

typedef std::integral_constant<bool, true>  true_type;
typedef std::integral constant<bool, false> false_type;
```

### 循环

说完了常量，我们来说说循环。在模板元编程中，你得忘掉你最喜欢的`for`循环，因为在编译期，编译器只有一种循环方式：递归。下面的代码展示了通过模板实现递归循环的正确方式：

```c++
template<size_t N>
struct factorial {
    static constexpr size_t value = N * factorial<N-1>;
};

// 递归必须要有结束条件，一般用一个特化的模板来作为结束条件
template<>
struct factorial<1> {
    static constexpr size_t value = 1;
};
```

这段代码计算某个数的阶乘，比如要计算并输出`5!`，你可以这样写：

```c++
std::cout << factorial<5>::value << std::endl; // 20
```

需要指出的是，上面这行语句虽然在运行期才能看到结果，但实际的值，也就是`5!`，在编译期就已经被编译器计算出来了。


### 分支

在模板元编程中，分支结构的实现依赖于一个不太为人熟知的编译器特性：**SFINAE**。 **SFINAE**是**S**ubstitution **F**ailure **I**s **N**ot **A**n **E**rror的首字母缩写。意思是说，当编译器在解析模板时，如果模板参数匹配失败，编译器不会报错，而是默默地跳过这段代码，继续编译后面的代码。举个例子：

```c++
template<class T>
typename T::multiplication_result multiply(T t1, T t2) {
    return t1 * t2;
}

long multiply(int i, int j){
    return i * j;
}

int main() {
    multiply(1, 2);
}
```

当编译器看到`main()`中的`multiply(1, 2)`的时候，需要在前面定义的两个`multiply`中挑出一个匹配的，编译器很有可能会选中这个：

```c++
template<class T>
typename T::multiplication_result multiply(T t1, T t2) {
    return t1 * t2;
}
```

问题是`int::multiplication_result`并不存在，这会导致编译错误，但由于SFINAE规则的存在，编译器会忽略这个错误，转而匹配第二个`multiply`函数。最终第一个`multiple`会被编译器扔掉，就像代码中从来不存在这样一个函数一样。

后面我们还会看到，SFINAE规则是标准库中很多_type trait_存在的基础。在这里我们只讲SFINAE规则的一个具体应用：实现类似于`if...else`的分支结构。

在标准库中有一个模板类`enable_if`，定义如下：

```c++
// file: type_traits

template<bool B, class T = void>
struct enable_if {
};

template<class T>
struct enable_if<true, T> {
    typedef T type;
};
```

如果`B`是`true`，则`enable_if<T>::type`就是存在的，否则`enable_if<T>::type`就不存在。那它怎么用呢？我们来看个例子：

[source,c++]
----
#include <iostream>
#include <type_traits>

using namespace std;

// #1
template<class T>
typename enable_if<is_pointer<T>::value, void>::type
do_something(T) {
    cout << "calling do_something(T*), return nothing\n";
}

// #2
template<class T>
typename enable_if<!is_pointer<T>::value, T>::type
do_something(T t) {
    cout << "calling do_something(T), return " << t << endl;
    return t;
}

int main() {
    int i = 3;
    do_something(i);
    do_something(&i);
}
----

输出
[source,c++]
----
calling do_something(T), return 3
calling do_something(T*), return nothing
----

可见，当调用``do_something(i)``的时候，因为``i``的类型是``int``，``is_pointer<int>::value``的值为``false``，于是``enable_if<is_pointer<T>::value, void>::type``不存在，第一个``do_something``会导致一个匹配失败，于是编译器转而去匹配第二个``do_something``函数。同理可以知道``do_something(&i)``会匹配第一个``do_something``函数。总的来说，上面的代码相当于于一个如下的``if...else``语句：

[source,c++]
----
if T is a pointer {
    // do something
}
else {
    // do something else
}
----

=== 总结

整形常量，布尔常量，循环，分支，这基本就是元编程的全部要素了。很简单吧，你觉得它难懂，是因为你还不熟悉它的代码结构，一旦你熟悉了，也就没什么了。








