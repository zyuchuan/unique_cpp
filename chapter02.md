# 2 unordered_set和unordered_map

Underred容器是C++11标准库中新增的类型，包括`unordered_set`和`unordered_map`。我们知道C++标准库中已经有了`set`和`map`，他们的区别是什么呢？

1. `set`和`map`是有序的，而*unordered*容器，顾名思义，元素是没有序的
2. `set`和`map`的底层数据结构是红黑树，而`undered`容器的底层数据结构是`hash table`
3. 一个的复杂度为O(log(n))，一个是O(1)

所以undered容器的效率更高一些，如果你对数据的顺序没有要求，建议使用新的unordered容器。

要了解unordered_set 和unordered_map的工作原理，先要了解hash table的原理，要了解hash table，先要知道C++11标准库中的hash

## 2.1 