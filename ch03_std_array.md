# std::array


`std::array`是*c-style*数组的对象化包装。我们知道*c-style*数组在使用时很痛苦的一点是数组本身并不知道自己有多少个元素，所以我们不得不花很大的精力确保数组不会越界。而``std::array``克服了这一缺陷，它自带``size``信息，这使得它使用起来很方便，我们来看几个例子：

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
    std::reverse_copy(a2.begin(), a2.end(), std::ostream_iterator<int>(std::cout, " "));
 
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


## std::array源代码分析


相较于很多类——比如``tuple``、``bind``等——那如天书一般的源代码，``std::array``的源代码算是真正的良心之作，几乎不需要什么智商就能看明白：

```c++
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
   
   // ...
```

看到了吧，刨去那一大堆的``typedef``，``std::array``就只有一个成员：``__elems_``，这是一个`c-style`的数组``std::array``就是就是C原生数组的一个马甲而已。

当然，这个马甲有个很好的特性：自带``size``信息：

```c++
template<class _Tp, size_t _Size>
struct array {
   // ...
   // capacity:
   inline constexpr size_type size() const noexcept {return _Size;}
   inline constexpr bool empty() const noexcept {return _Size == 0;}
   // ...
};
```

另外，这个穿上马甲的``std::array``已经摇身一变成了一个容器，既然是容器，就必须遵循C++标准可以关于容器的``concept``，比如遍历，获取元素等等：

```c++
template<class _Tp, size_t _Size>
struct array {
   // ...
   // iterators:
   inline constexpr iterator begin() noexcept {return iterator(__elems_);}
   inline constexpr iterator end() noexcept {return iterator(__elems_ + _Size);}
   // ...
   
   // element access:
   inline constexpr reference operator[](size_type __n) {return __elems_[__n];}
   constexpr reference at(size_type __n) {
      if (__n >= _Size) __throw_out_of_range("array:at");
      return __elems_[__n];
   }
   // ...
};
```

源代码确实很简单，就不多讲了。最后再说点题外话：``swap``，我们来看看``std::array``是怎样实现``swap``的：

```c++
template <class _Tp, size_t _Size>
struct array{
   // ...
   
   inline void swap(array& __a) noexcept(_Size == 0 || __is_nothrow_swappable<_Tp>::value) {
      __swap_dispatch((std::integral_constant<bool, _Size == 0>()), __a);
   }
   
   inline void __swap_dispatch(std::true_type, array&){}
   
   inline void __swap_dispatch(std::false_type, array& __a) {
      std::swap_range(__elems_, __elems_ + _Size, __a.__elems_);
   }
};

template<class _Tp, size_t _Size>
inline
typename enable_if<_Size == 0 || __is_swappable<_Tp>::value, void>::type
swap(array<_Tp, _Size>& __x, array<_Tp, _Size>& __y) {
   __x.swap(__y);
}
```