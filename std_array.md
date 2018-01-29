# std::array

`std::array`是“c-style”数组的对象化包装，在保留了“c-style”数组高效易用特性的同时，还提供了一个额外的福利：`std::array`知道自己的元素个数，这使得它使用起来很方便，我们来看几个例子：

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

## 2. std::array源代码分析

总的来说，`std::array`的源代码比较容易看懂，所以下面我们就给出源代码，并不多做解释。

```c++
// file: array

template<class _Tp, size_t _Size>
struct array {
   // types:
   typedef array __self;
   typedef _Tp                                   value_type;
   typedef value_type&                           reference;
   typedef const value_type&                     const_reference;
   typedef value_type*                           iterator;
   typedef const value_type*                     const_iterator;
   typedef value_type*                           pointer;
   typedef const value_type*                     const_pointer;
   typedef size_t                                size_type;
   typedef ptrdiff_t                             difference_type;
   typedef std::reverse_iterator<iterator>       reverse_iterator;
   typedef std::reverse_iterator<const_iterator> const_reverse_iterator;
   
   // 唯一的成员，c-style数组
   value_type __elems_[_Size > 0 ? _Size : 1];
   
   // iterators:
   inline constexpr iterator begin() noexcept {return iterator(__elems_);}
   inline constexpr iterator end() noexcept {return iterator(__elems_ + _Size);}
   // ...
   
   // capacity:
   inline constexpr size_type size() const noexcept {return _Size;}
   inline constexpr bool empty() const noexcept {return _Size == 0;}
   // ...
   
   // element access:
   inline constexpr reference operator[](size_type __n) {return __elems_[__n];}
   constexpr reference at(size_type __n);
   
};
```

