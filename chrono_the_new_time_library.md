

长久以来，C++标准库都缺了一样重要的工具：Time library，这种情况终于在C++标准委员会指定C++11时得到了改变。那就是`chrono`，`chrono`是一个优雅的，轻量级，功能强大，强类型的时间库。

chrono的特点如下

* 轻量级，比如`chrono::duration`实际就是一个`long long`类型的整数而已。
* 强类型，比如`seconds`就是`seconds`
* 支持自定义

`std::chrono`包括以下三个部件

1. duration
2. time point
3. clock

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

那`nano`，`micro`又是什么呢，你可能已经猜到了，是`ration`，那`ration`又是什么呢？

### 1.1 std::ration

`std::ration`通过类模板的方式定义个一个编译期有理数。我们知道C++只支持编译期整数，那编译期有理数怎么实现呢？

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