# 2. tuple

## 2.1 tuple简史

[cppreference](http://en.cppreference.com/w/cpp/utility/tuple)对tuple的解释是“fixed-size collection of heterogeneous values”，也就是有固定长度的异构数据的集合。

最简单的tuple就是`std::pair`，想必大家已经很熟悉了，但是`std::pair`只能容纳两个数据，而tuple可以容纳任意多个任意类型的数据。

## 2.2 tuple的用法

### 2.2.1 make_tuple

C++标准库对`tuple`进行了很好的包装，使得使用很方便，你用`make_tuple`创建一个`tuple`:

```
std::tuple<int, double, char> t1 = std::make_tuple<int, double, char>(1, 34.5, 'M');

auto t2 = std::make_tuple<std::string, double, int>("Jack", 30.0, 2);

```

### 2.2.2 tuple_size

### 2.2.3 tuple_element

### 2.2.4 std::get()

tuple的源代码及其复杂，大量使用元编程技巧，我不打算把本章编程C++元编程入门讲解，所以，我会实现一个简化版的*tuple*。