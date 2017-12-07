说起来有点令人难以置信，直到C++ 11之前，标准库中唯一可以处理时间的就是`<ctime>`提供的有限的几个函数，而即使这有限的几个函数还是C函数，这不得不说是标准库的一个污点。标准委员会意识到了这个问题，为了抚平广大C++代码仔心中的创伤，他们为C++ 11标准库引入了一个优雅而强大的time library: `chrono`。

## 1. chrono简介

`chrono`是一个基于模板的，面向对象的，设计优雅且功能强大的*time library*。`chrono`内部定义了三种和时间相关的类型：

* **Clock**: *clock*的作用就相当于我们日常使用的手表：显示时间。`chrono`内部定义了三种*clock*：`system clock`、`steady clock`和`high-resolution-clock`，我们会在后面一一介绍。

* **duration**：

## 1. std::duration


```
// file: chrono

template<class _Rep, class _Period>
class duration {
public:
    typedef _Rep     rep;
    typedef _Period  period;
private:
    rep __rep_;
    
// ...
};
```

`duration`的定义很简单，它有两个类模板参数`_Rep`和`_Period`，而它本身只有一个成员`__rep_`。你肯定很好奇`_Rep`和`_Period`是什么，看到下面的定义你就会明白：

```
// file: chrono

typedef duration<long long,         nano> nanoseconds;
typedef duration<long long,        micro> microseconds;
typedef duration<long long,        milli> milliseconds;
typedef duration<long long              > seconds;
typedef duration<     long, ratio<  60> > minutes;
typedef duration<     long, ratio<3600> > hours;
```

这下明白了，`_Rep`就是C++支持的数据类型，这就是为什么我们说chrono是个轻量级的库，它本身只包含一个`long`或`long long`类型的数据成员。

那`nano`，`micro`又是什么呢，你可能已经猜到了，是`ratio`，那`ratio`又是什么呢？

### 1.1 std::ratio

`std::ratio`通过类模板的方式定义个一个编译期有理数。我们知道C++只支持编译期整数，那编译期有理数怎么实现呢？

```
// file: ratio

// _Num代表 'numerator'（分子）
// _Den代表 'denominator'（分母）
template<intmax_t _Num, intmax_t _Den = 1>
class ration {
    // 求__Num的绝对值
    static constexpr const intmax_t __na = __static_abs<_Num>::value;
    
    // 求_Den的绝对值
    static constexpr const intmax_t __da = __static_abs<_Den>::value;
    
    // __static_sign的作用是求符号运算
    static constexpr const intmax_t __s = __static_sign<_Num>::value * __static_sign<_Den>::value;
    
    // 求分子、分母的最大公约数
    static constexpr const intmax_t __gcd = __static_gcd<__na, __da>::value;
    
public:
    
    // num是化简后的_Num
    static constexpr const intmax_t num = __s * __na / __gcd;
    
    // den是化简后的_Den
    static constexpr const intmax_t den = __da / __gcd;
};
```

可以看到`ratio`用两个整数分别代表分子和分母来表示一个有理数。

标准库还定义了常见的量纲

```
// file: ratio

...

typedef ratio<1LL,          1000000000LL> nano;
typedef ratio<1LL,             1000000LL> micro;
typedef ratio<1LL,                1000LL> milli;
typedef ratio<1LL,                 100LL> centi;
typedef ratio<               1000LL, 1LL> kilo;
typedef ratio<            1000000LL, 1LL> mega;

...
```

`milli`表示千分之一，用`ratio`表示就是`ratio<1, 1000>`，很优雅，也很直观。

### 1.2 std::duration的用法

我们已经知道了什么是`duration`，下面我们来看如何使用

```
#include <iostream>
#include <chrono>
#include <thread>
 
int main()
{
    std::chrono::seconds two_seconds{2}; 
    std::cout << "Start waiting..." << std::endl;
    auto start = std::chrono::high_resolution_clock::now();
    std::this_thread::sleep_for(two_seconds);
    auto end = std::chrono::high_resolution_clock::now();
    std::chrono::duration<double, std::milli> elapsed = end-start;
    std::cout << "Waited " << elapsed.count() << " ms\n";
}

// output
Start waiting...
Waited 2002.58 ms
```