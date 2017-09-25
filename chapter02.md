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

## 2.3 tuple的实现原理

tuple的源代码及其复杂，大量使用元编程技巧，我不打算把本章编程C++元编程入门讲解，所以，我会实现一个简化版的*tuple*。

```
#include <utility>
#include <type_traits>

namespace unique_cpp {
    
    // forward declaration
    template<class ...T> class tuple;
    
    // tuple_element
    template<size_t Index, class T> class tuple_element;
    
    // tuple_size
    template<class ...T> class tuple_size;
    
    template<class ...T>
    class tuple_size<tuple<T...> > : public std::integral_constant<size_t, sizeof...(T)> {};
    
    
    
    // tuple_types
    template<class ...T> struct tuple_types {};
    
    // make_tuple_types
    template<class T, size_t End = tuple_size<T>::value, size_t Start = 0>
    struct make_tuple_types {};
    
    template<class ...T, size_t End>
    struct make_tuple_types<tuple<T...>, End, 0> {
        typedef tuple_types<T...> type;
    };
    
    template<class ...T, size_t End>
    struct make_tuple_types<tuple_types<T...>, End, 0> {
        typedef tuple_types<T...> type;
    };
    
    // tuple_indices
    template<size_t ...value> struct tuple_indices {
        //const static int values = sizeof...(value);
    };
    
    template<class IndexType, IndexType ...values>
    struct integer_sequence {
        template<size_t Start>
        using to_tuple_indices = tuple_indices<(values + Start)...>;
    };
    
    template<size_t End, size_t Start>
    using make_indices_imp = typename __make_integer_seq<integer_sequence, size_t, End - Start>::template
        to_tuple_indices<Start>;
    
    template<size_t End, size_t Start = 0>
    struct make_tuple_indices {
        typedef make_indices_imp<End, Start> type;
    };
    
    // tuple_element
    namespace indexer_detail {
        template<size_t Index, class T>
        struct indexed {
            using type = T;
        };
        
        template<class Types, class Indexes> struct indexer;
        
        template<class ...Types, size_t ...Index>
        struct indexer<tuple_types<Types...>, tuple_indices<Index...>>
            : public indexed<Index, Types>... {};
        
        template<size_t Index, class T>
        indexed<Index, T> at_index(indexed<Index, T> const&);
    } // namespace indexer_detail
    
    template<size_t Index, class ...Types>
    using type_pack_element = typename decltype(indexer_detail::at_index<Index>(
          indexer_detail::indexer<tuple_types<Types...>,
            typename make_tuple_indices<sizeof...(Types)>::type>{}))::type;
    
    template<size_t Index, class ...T>
    struct tuple_element<Index, tuple_types<T...> > {
        typedef type_pack_element<Index, T...> type;
    };
    
    template<size_t Index, class ...T>
    struct tuple_element<Index, tuple<T...> > {
        typedef type_pack_element<Index, T...> type;
    };
    
    // tuple_leaf
    template<size_t Index, class Head>
    class tuple_leaf {
        Head value;
        
    public:
        tuple_leaf() : value() {}
        
        template<class T>
        explicit tuple_leaf(const T& t) : value(t) {}
        
        Head& get() {return value;}
        const Head& get() const {return value;}
    };
    
    // tuple_imp
    template<class Index, class ...T> struct tuple_imp;

    template<size_t ...Index, class ...T>
    struct tuple_imp<tuple_indices<Index...>, T...> : public tuple_leaf<Index, T>... {
        
        tuple_imp() {}
        
        template<size_t ...Uf, class ...Tf, class ...U>
        tuple_imp(tuple_indices<Uf...>, tuple_types<Tf...>, U&& ...u) :
                tuple_leaf<Uf, Tf>(std::forward<U>(u))... {
                    
                }
    };

    template<class ...T>
    struct tuple {
        typedef tuple_imp<typename make_tuple_indices<sizeof...(T)>::type, T...> base;
        base base_;

        tuple(const T& ...t) : base_(typename make_tuple_indices<sizeof...(T)>::type(),
                                     typename make_tuple_types<tuple, sizeof...(T)>::type(),
                                     t...) {}
    };
    
    template<class T>
    struct make_tuple_return_imp {
        typedef T type;
    };
    
    template<class T>
    struct make_tuple_return {
        typedef typename make_tuple_return_imp<typename std::decay<T>::type>::type type;
    };
    
    template<class ...T>
    inline tuple<typename make_tuple_return<T>::type...> make_tuple(T&& ...t) {
        return tuple<typename make_tuple_return<T>::type...>(std::forward<T>(t)...);
    }
    
    // get
    template<size_t Index, class ...T>
    inline typename tuple_element<Index, tuple<T...> >::type& get(tuple<T...>& t) {
        typedef typename tuple_element<Index, tuple<T...> >::type type;
        return static_cast<tuple_leaf<Index, type>&>(t.base_).get();
    }
    
}
```