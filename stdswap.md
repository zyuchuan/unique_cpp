# std::swap()：简约而不简单


`std::swap()`是一个很简单的函数，

```
// file: type_traits

template<class T>
typename enable_if
<
  // T must be "move constructible" and "move assignable"
  is_move_constructible<T>::value &&
  is_move_assignable<T>::value
>::type
swap(T& x, T& y) noexcept( 
                  // if T's move constructor and move
                  // assignment does't not throw, 
                  //swap() does't throw
                  is_nothrow_move_constructible<T>::value && 
                  is_nothrow_move_assignable<T>::value) {
                  
  T t(std::move(x));
  x = std::move(y);
  y = std::move(t);
}
```
