# std::chrono

说起来有点令人难以置信，直到C++ 11之前，标准库中唯一可以处理时间的就是`<ctime>`提供的有限的几个函数，而即使这有限的几个函数还是C函数，这使得在C++中处理时间成为一件头疼的事情。不过现在情况有了很大改观，因为有了`std::chrono`。

## chrono简介

`chrono`是一个基于模板的，面向对象的，设计优雅且功能强大的*time library*。`chrono`内部定义了三种和时间相关的类型：

* **duration**：一个*duration*就代表了一个时间段，比如2分钟，4小时等等。

* **clock**: *clock*的作用就相当于我们日常使用的手表：显示时间。`chrono`内部定义了三种*clock*：`system clock`、`steady clock`和`high-resolution-clock`。

* **time point**：*time point*表示某个特定的时间点。


## std::duration

前面说过，`duration`代表了一段时间，比如2分钟，4小时等。注意我不仅指出了数量（“2”、“4”），还指出了单位（“分钟”，“小时”），也就是说一个`duration`的定义应该包括两个部分：数量和单位。我们来看`chrono`是怎样做到这一点的。

```c++
// file: chrono

namespace chrono {

    template<class _Rep, class _Period = ratio<1> >
    class duration {
    public:
        typedef _Rep     rep;
        typedef _Period  period;
    private:
        rep __rep_;
    
    // ...
    };
}
```

`duration`的声明包含两个模板参数，第一个模板参数是C++的原生数值类型，如`long`, `long long`等，代表了`duration`的数值部分。第二个模板参数`_Period`又是一个模板类`std::ratio`，它的定义如下：


```c++
// file: ratio

namespace chrono {

    // ratio以模板的方式定义了有理数，比如ratio<1,60>就表示有理数 ‘1/60’
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
}
```

`ratio`用两个整数型模板参数来表示一个有理数的分子和分母部分，比如`ratio<1, 1000>`就表示有理数`0.001`。理解了这一点，我们再来看`duration`的定义：

```
template<class _Rep, class _Period = ratio<1> > class duration 
```

`ratio`在这里的确切含义为：以秒为单位的放大倍率，比如`ratio<60, 1>`表示一个`1秒的60倍，也就是1分钟`，而`ratio<1, 1000>`表示`1秒的千分之一倍，也就是1毫秒`。所以`duration<long, ratio<60, 1>>`就定义了一个类型为`long`的`duration`，而这个`duration`的单位为“分钟”。很巧妙的定义，是不是？

为了方便我们使用，对常用的时间单位，标准库都已经替我们定义好了：

```c++
namespace chrono {

    // 1nano = 1/1,000,000,000 秒
    typedef ratio<1LL, 1000000000LL> nano;

    // 1micro = 1/1,000,000秒
    typedef ratio<1LL, 1000000LL> micro;

    // 1milli = 1/1,000秒
    typedef ratio<1LL, 1000LL> milli;

    // 1centi = 1/100秒
    typedef ratio<1LL, 100LL> centi;

    // 1kilo = 1,000秒
    typedef ratio<1000LL, 1LL> kilo;

    // 1mega = 1,000,000秒
    typedef ratio<1000000LL, 1LL> mega;
    
    // ...
    
    typedef duration<long long,         nano> nanoseconds;
    typedef duration<long long,        micro> microseconds;
    typedef duration<long long,        milli> milliseconds;
    typedef duration<long long              > seconds;
    typedef duration<     long, ratio<  60> > minutes;
    typedef duration<     long, ratio<3600> > hours;
    
    // ...
}
```

### duration的用法

标准委员会重写了标准库中全部和时间有关的函数，使之可以和`duration`协同工作，这使得`duration`的使用非常简单直观，举个例子：

```
#include <iostream>
#include <chrono>
#include <thread>
 
int main()
{
    // 定义一个"chrono::seconds"
    std::chrono::seconds two_seconds{2}; 
    std::cout << "Start waiting..." << std::endl;
    auto start = std::chrono::high_resolution_clock::now();
    
    // sleep_for函数现在接受一个`duration`类型的参数
    std::this_thread::sleep_for(two_seconds);
    
    auto end = std::chrono::high_resolution_clock::now();
    
    // duration支持算术运算
    std::chrono::duration<double, std::milli> elapsed = end-start;
    std::cout << "Waited " << elapsed.count() << " ms\n";
}

// output
Start waiting...
Waited 2002.58 ms
```

### 为什么你应该使用duration

1. `duration`是轻量级的，在前面的源代码中我们已经看到了，`duration`内部仅有一个数值类型的变量，很是轻量级。至于`ratio`，因为其成员变量都是静态的，所以你不用担心系统内会有很多的`ratio`存在。

2. `duration`是类型安全的。