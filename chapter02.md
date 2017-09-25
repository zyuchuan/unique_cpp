# 2. tuple

## 2.1 tuple简史

[cppreference](http://en.cppreference.com/w/cpp/utility/tuple)对tuple的解释是“fixed-size collection of heterogeneous values”，也就是有固定长度的异构数据的集合。

最简单的tuple就是`std::pair`，想必大家已经很熟悉了，但是`std::pair`只能容纳两个数据，而tuple可以容纳任意多个任意类型的数据。

## 2.2 tuple的用法

### 2.2.1 make_tuple

C++标准库对`tuple`进行了很好的包装，使得使用很方便，你用`make_tuple`创建一个`tuple`:

