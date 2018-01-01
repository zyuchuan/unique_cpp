# std::swap()


`std::swap()`是一个很简单的函数：交换两个参数的值，仅此而已。但是这个看似平淡无奇的函数，背后的故事却不简单。不知道你考虑过下面问题没有：

1. 我们通常要求这个函数不能抛出异常，为什么？
2. 如何才能让`std::swap`函数能高效地作用于你自己定义的类？

要回答这两个问题，就要从`std::swap`的实现说起。

## 1. std::swap()的实现

libc++中，`swap()`定义在文件“type_traits”中：

```
// file: type_traits

template<class T>
typename enable_if<
  is_move_constructible<T>::value &&
  is_move_assignable<T>::value
>::type
swap(T& x, T& y) noexcept( 
            is_nothrow_move_constructible<T>::value && 
            is_nothrow_move_assignable<T>::value) {      
  T t(std::move(x));
  x = std::move(y);
  y = std::move(t);
}
```

从定义可以看出：

1. 一个“swappable”的类型必须满足“move constructible”和“move assignable”两个前提条件，这是C++11的新要求。

2. 如果类型`T`的“move constructor”和“move assignment operator”不抛出异常，则`swap`也不能抛出异常。

虽然标准并没有强制规定`swap()`不能抛出异常，但是实际使用中，我们都要确保`swap()`不抛出异常，这是为什么呢？要回答这个问题，就要从**C++异常安全保证**说起。

## 2. 异常安全保证

维基百科对[异常安全保证](https://en.wikipedia.org/wiki/Exception_safety)（Exception Safety Guarantees）的解释是“***类的设计者和使用者在使用任何一门程序设计语言，特别是C++时可以遵守的一系列异常处理准则***”。

异常安全保证从弱到强可以分为三个等级：

* **基本保证（Basic Exception Safety）**: 也叫**无泄漏保证（No-Leak Guarantee）**，即发生异常时不会导致资源泄露（比如内存泄露），程序内的任何事物仍然保持在有效状态下，没有对象或数据结构会因此而破坏，所有对象都处于有效的状态，但是处于哪个状态不可预知。

* **强烈保证（Strong Exception Safety）**：如果抛出异常，程序状态不改变。就像数据库中的事务处理一样，要么成功，如果不成功，则程序回到调用之前的状态。

* **不抛出异常保证（No-Throw Guarantee）**：承诺绝不抛出异常。如果有异常发生，会在内部处理，保证不让异常逃逸。

**不抛出异常保证**还比较好理解，不过另外两个**保证**可能会让你有点摸不着头脑，下面我就举个例子来说明。假设你要实现一个`list`数据结构，你打算用一系列相互连接的节点来实现这个`list`：

```
template<typename T>
class list {
private:
    template<typename U>
    struct node {
        U value;
        node* next;
        
        node(U&& val) : value(std::move(val)) {}
    };
    
    node<T>*   _head;
    size_t     _length; // length of list
    
public:
    // ...
    
    void insert(T&& val);
    inline size_t length() { return _length;}
    
    //...
};
```

因为查询`list`的长度是一个常用的操作，所以你将`list`的长度保存在`_length`中。同时，`list`还应该支持从头部插入新元素：

```
template<typename T>
void list<T>::insert(T&& val) {
    ++_length;
    node<T>* n = new node<T>(std::forward<T>(val));
    n->next = _head;
    _head = n;
    
}
```

现在设想一下，如果`insert`函数抛出了异常会发生什么？要弄清楚问题的答案，首先要弄清楚哪行代码会抛出异常。很显然，在`insert`函数中，最有可能、或者说唯一有可能抛出异常的代码是

```
node<T>* n = new node<T>(std::forward<T>(val));
```

假如这行代码抛出异常，因为`insert`并没有捕捉异常，所以异常会向上传播，达到`insert`的调用者，进一步假设`insert`的调用者捕捉到了这个异常，并决定忽略这个错误，就像这样：

```
void insert_and_print(const list &l) {
    try {
        l.insert(1);
    }
    catch(...) {
        // log error here
    }
    
    std::cout << "There are " << l.length() << " element(s) in list";
}

list l;
insert_and_print(l);
```

如果`l.insert(1)`失败，你可能认为上面的代码会输出

```
There are 0 element(s) in list
```

**错！大错特错！**这才是你会看到的：

```
There are 1 element(s) in list
```

我们再来看一下`insert()`的代码：

```
template<typename T>
void list<T>::insert(T&& val) {
    ++_length;
    node<T>* n = new node<T>(std::forward<T>(val));
    n->next = _head;
    _head = n;
    
}
```

注意在调用`new`之前，`_length`的值已经被改变了，也就是说新元素并没有被插入，但是`list`的内部状态已经被改变了。

在这里，函数`insert_and_print`提供了基本异常保证，当异常发生了，程序可以继续执行，不会有资源泄露，但是`list`内部的状态已经被改变，而且变成了非法的状态，这是很危险的。

通常来说，基本异常安全保证并不是我们追求的目标，因为基本异常安全保证只保证不泄露资源，并不保证程序总是处于合法的状态，所以我们总希望提供强烈安全保证。你可能觉得这很难，其实并非如此，我们把`list::insert`方法稍做改动：

```
template<typename T>
void list<T>::insert(T&& val) {    
    node<T>* n = new node<T>(std::forward<T>(val));
    
    ++_length;
    n->next = _head;
    _head = n;
    
}
```

只需要把`++_length`放到`new`之后，`insert`就摇身一变，变成了一个提供强烈异常安全保证的函数，因为即使分配内存失败，函数异常退出，`_length`没有变，`_head`也没有变，整个`list`的状态没有任何改变，整个程序任然能愉快地运行！

扯了这么多，你可能已经不耐烦了：“这TMD和`swap`有什么关系？”

## 3. std::swap和异常安全保证

如果你有一个管理资源的类，比如智能指针，那么一个不抛出异常的`swap()`将会非常有用。对于管理资源的类，你通常都需要定义析构函数、拷贝构造函数，拷贝赋值函数、移动构造函数和移动赋值操作。有了`swap()`，拷贝赋值函数和移动构造函数将会变得很简单，而且提供强烈异常安全保证。

假如你有一个类`ResourceManager`，你可以这样实现上面

```
class ResourceManager {
    // ....
public:
    ResourceManager(const ResourceManager &other);
    
    // 拷贝赋值函数，注意参数是pass-by-value
    ResourceManager& operator=(ResourceManager other) {
        // 因为是pass-by-value，“other”相当于一个临时变量，
        // 函数退出后会被析构掉，避免了资源泄露
        swap(*this, other);
        return *this;
    }
    
    // 移动构造函数    
    ResourceManager(ResourceManager &&other)
        : /* 初始化成员变量 */ {
        swap(*this, other);       
    }
};
```

这个技巧在C++中是如此常见，以至于已经成为了一种“idiom”，也就是我们常说的“copy-and-swap idom”。Stack Overflow上有一篇很好的关于[swap-and-copy idiom](https://stackoverflow.com/questions/3279543/what-is-the-copy-and-swap-idiom)的文章，有兴趣的可以去看看。

现在我们知道了为什么一个不抛出异常的`swap()`很重要，但是还有一个重要的问题：如何才能让`swap()`能够高效地作用于我们自定义的类？

## 4. 实现你自己的swap()

前面我们已经看到了`std::swap()`的源代码，三行简单的代码居然调用了一次拷贝构造函数，两次拷贝赋值函数，很多时候这样的行为都包含着资源的分配和释放，而资源的分配和释放是非常低效率的，这对于为效率而生的C++而言是不可接受的，这就是为什么标准库中的几乎每个类都重载了`swap()`的原因。如果你有一个管理资源的类，你几乎总是需要重载`swap()`，那该怎样做呢？很简单，定义两个`swap()`：一个类成员`swap`和一个重载的`std::swap`：

```
class ResourceManager {
    // ...
    
public:
    // 1. 定义类成员函数
    void swap(ResourceManager &other) noexcept {
        // do swap here
    }
};

// 2. 重载swap
inline void swap(ResourceManager &lhs, ResourceManager &rhs) noexcept {
    using std::swap; // 保证有机会调用std::swap
    lhs.swap(rhs);
}
```

