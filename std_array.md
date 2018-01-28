# std::array

`std::array`是对c-like数组的对象化包装，在保留c-style数组高效的同时，还有以下的有点：

1. 本身包含size信息
2. 本身就是容器的概念，因此可以当做容器使用

> 注意：不像c-style数组，std::array不会自动退化为指针。

## 1. 使用方法

```c++
#include <string>
#include <iterator>
#include <iostream>
#include <algorithm>
#include <array>
 
int main(){
    // construction uses aggregate initialization
    std::array<int, 3> a1{ {1, 2, 3} }; // double-braces required in C++11 (not in C++14)
    std::array<int, 3> a2 = {1, 2, 3};  // never required after =
    std::array<std::string, 2> a3 = { std::string("a"), "b" };
 
    // container operations are supported
    std::sort(a1.begin(), a1.end());
    std::reverse_copy(a2.begin(), a2.end(), 
          std::ostream_iterator<int>(std::cout, " "));
 
    std::cout << '\n';
 
    // ranged for loop is supported
    for(const auto& s: a3)
        std::cout << s << ' ';
}
```

输出：

```c++
3 2 1
a b
```

