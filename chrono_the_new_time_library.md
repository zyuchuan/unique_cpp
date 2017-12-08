说起来有点令人难以置信，直到C++ 11之前，标准库中唯一可以处理时间的就是`<ctime>`提供的有限的几个函数，而即使这有限的几个函数还是C函数，这不得不说是标准库的一个污点。标准委员会意识到了这个问题，为了抚平广大C++代码仔心中的创伤，他们为C++ 11标准库引入了一个优雅而强大的time library: `chrono`。

## 1. chrono简介

`chrono`是一个基于模板的，面向对象的，设计优雅且功能强大的*time library*。`chrono`内部定义了三种和时间相关的类型：

* **duration**：一个*duration*就代表了一个时间段，比如2分钟，4小时等等。

* **clock**: *clock*的作用就相当于我们日常使用的手表：显示时间。`chrono`内部定义了三种*clock*：`system clock`、`steady clock`和`high-resolution-clock`。

* **time point**：*time point*表示某个特定的时间点。


## 1. std::duration

前面说过，`duration`代表了一段时间，比如2分钟，4小时等。注意我不仅指出了数量（“2”、“4”），还指出了单位（“分钟”，“小时”），也就是说一个`duration`的定义应该包括两个部分：数量和单位。我们来看`chrono`是怎样做到这一点的。

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

typedef duration<long long,         nano> nanoseconds;
typedef duration<long long,        micro> microseconds;
typedef duration<long long,        milli> milliseconds;
typedef duration<long long              > seconds;
typedef duration<     long, ratio<  60> > minutes;
typedef duration<     long, ratio<3600> > hours;
```

可以看到，`duration`的声明包含两个模板参数：

* 第一个模板参数是C++的原生数值类型`long`, `long long`等，代表了`duration`的数值部分。

* 第二个模板参数`_Period`表达了量纲的概念，比如上面代码中的`milli`就表示*“千分之一秒”*的意思，它的类型是`std::ratio`。


### 1.1 std::ratio

`std::ratio`也是模板类，一个`ratio`就是一个编译期有理数，它的值表示的是*“1秒的倍数”*。

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