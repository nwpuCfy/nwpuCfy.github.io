最近看了type_traits源码，发现其中用到了元编程，特意摘出来看看。

首先利用`integral_constant`定义了`true_type`和`false_type`两个类，类`__or_`、`__and_`和`__not_`利用元编程可以在编译期根据多个类型特性的与、或、非值决定执行哪种策略。

```c++
  //gcc version 7.5.0  

  /// integral_constant
  template<typename _Tp, _Tp __v>
    struct integral_constant
    {
      static constexpr _Tp                  value = __v;
      typedef _Tp                           value_type;
      typedef integral_constant<_Tp, __v>   type;
      constexpr operator value_type() const noexcept { return value; }
      constexpr value_type operator()() const noexcept { return value; }
    };

  /// The type used as a compile-time boolean with true value.
  typedef integral_constant<bool, true>     true_type;

  /// The type used as a compile-time boolean with false value.
  typedef integral_constant<bool, false>    false_type;

  // Meta programming helper types.

  template<bool, typename, typename>
    struct conditional;

  template<typename...>
    struct __or_;

  template<>
    struct __or_<>
    : public false_type
    { };

  template<typename _B1>
    struct __or_<_B1>
    : public _B1
    { };

  template<typename _B1, typename _B2>
    struct __or_<_B1, _B2>
    : public conditional<_B1::value, _B1, _B2>::type
    { };

  template<typename _B1, typename _B2, typename _B3, typename... _Bn>
    struct __or_<_B1, _B2, _B3, _Bn...>
    : public conditional<_B1::value, _B1, __or_<_B2, _B3, _Bn...>>::type
    { };

  template<typename...>
    struct __and_;

  template<>
    struct __and_<>
    : public true_type
    { };

  template<typename _B1>
    struct __and_<_B1>
    : public _B1
    { };

  template<typename _B1, typename _B2>
    struct __and_<_B1, _B2>
    : public conditional<_B1::value, _B2, _B1>::type
    { };

  template<typename _B1, typename _B2, typename _B3, typename... _Bn>
    struct __and_<_B1, _B2, _B3, _Bn...>
    : public conditional<_B1::value, __and_<_B2, _B3, _Bn...>, _B1>::type
    { };

  template<typename _Pp>
    struct __not_
    : public __bool_constant<!bool(_Pp::value)>
    { };
```

其中`conditional`模板的定义如下

```c++
  template<bool _Cond, typename _Iftrue, typename _Iffalse>
    struct conditional
    { typedef _Iftrue type; };

  // Partial specialization for false.
  template<typename _Iftrue, typename _Iffalse>
    struct conditional<false, _Iftrue, _Iffalse>
    { typedef _Iffalse type; };
```

利用`enable_if`写一段代码测试一下，其中`fun`函数只能接受指针和数组作为参数

```c++
#include <bits/stdc++.h>
using namespace std;


template<class T1, class T2>
    using require = typename __and_<is_pointer<T1>, is_array<T2>>::type;

template<class T1, class T2>
void fun(T1 x, T2& y, typename enable_if<require<T1, T2>::value, void>::type *tmp = 0) {//T2& 避免数组退化成指针
    cout << "T1 is pointer type\n";
    cout << "T2 is array type\n";
}

int main() {
    int a = 1;
    int *pa = &a;
    int b[] = {1, 2, 3};
    fun(pa, b);//fun 的第一个参数只能传入指针, 第二个只能传入数组
    return 0;
}
```

