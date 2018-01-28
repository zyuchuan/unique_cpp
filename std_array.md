# std::array

`std::array`是对c-like数组的对象化包装，在保留c-style数组高效的同时，还有以下的有点：

1. 本身包含size信息
2. 本身就是容器的概念，因此可以当做容器使用

> 注意：不像c-style数组，std::array不会自动退化为指针。