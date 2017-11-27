

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

## 2 std::ration

