## C++ Templates: The Complete Guide Second Edition

### Chapter 1 Function Template

在模板推导时自动类型转换有以下2个限制

1、当通过引用声明调用参数时，即使是微不足道的转换也不适用于类型推导。使用相同模板参数T声明的两个参数必须完全匹配

2、按值声明调用参数时，只支持衰减（decay）的简单转换：忽略const或volatile的限定，引用转换为被引用的类型，原始数组或函数转换为相应的指针类型。对于使用相同模板参数T声明的两个参数，衰减类型必须匹配：

```cpp
template<typename T>
T max(T a, T b)
{
	return b < a ? a : b;
}

max(4, 7.2);        // ERROR: T can be deduced as int or double
std::string s;
foo("hello", s);    //ERROR: T can be deduced as char const[6] or std::string”
```

我们可以通过以下策略处理：

1、转换参数使之匹配

```cpp
max(static_cast<double>(4), 7.2);
```

2、明确一个指定的转换类型进行转换

```cpp
max<double>(4, 7.2);
```

3、指定参数可能具有不同的类型。

另请注意，类型推导不适用于默认调用参数，如果想要支持，则需要提供默认的参数给模板参数

```cpp
template<typename T>
void f(T = "");

f(1);        // OK: deduced T to be int, so that it calls f<int>(1)
f();         // ERROR: cannot deduce T”

//修改：
template<typename T = std::string>
void f(T = "");
...
f();        //OK
```

多模板参数

```cpp
template<typename T1, typename T2>
T1 max (T1 a, T2 b)
{
	return b < a ? a : b; 
}

auto m = ::max(4, 7.2);
```

这里存在问题，就是max出来结果是7，而不再是7.2。这里有以下方法处理：

1、引入第三个模板参数来作为返回值

```cpp
template<typename T1, typename T2, typename RT>
RT max (T1 a, T2 b);

//这里我们甚至能自己控制返回值类型
::max<int, double, double>(4, 7.2);

//当缺省部分模板类型时
template<typename RT, typename T1, typename T2>
RT max (T1 a, T2 b);
::max<double>(4, 7.2)        //OK: return type is double, T1 and T2 are deduced
```

2、编译器自己去得到返回值类型

```cpp
//in c++14
//最优雅
template<typename T1, typename T2>
auto max (T1 a, T2 b)
{
  return b < a ? a : b;
}

//in c++11的讨论
template<typename T1, typename T2>
auto max (T1 a, T2 b) -> decltype(b < a ? a : b)
{
  return b < a ? a : b;
}
```

注意到，这里的

```cpp
template<typename T1, typename T2>
auto max (T1 a, T2 b) -> decltype(b < a ? a : b);
```

只是一个声明，编译器会在编译期使用三元运算符来调用a，b参数找到max()的返回值。实现上不需要匹配，使用true作为条件作为声明也足够了：

```cpp
template<typename T1, typename T2>
auto max (T1 a, T2 b) -> decltype(true ? a : b);
```

但是，在任何情况下，这个定义都有一个明显的缺点：可能会发生返回类型是引用类型。因此，应该返回从T中衰减（decayed）的类型：

```cpp
#include <type_traits>
template<typename T1, typename T2>
auto max(T1 a, T2 b)->typename std::decay<decltype(true ? a : b)>::type
{
	return b < a ? a : b;
}
```

这里使用了类型trait `std::decay <>`，它返回成员`type`中的结果类型。它由<type_trait>中的标准库定义。因为成员`type`是一个类型，所以我们必须要使用`typename`使之合法化（其实这里也就是返回了这两个参数的common type）

3、使用两个参数的公共的一种类型"common type"

从C++11开始，C++标准库提供了一种更加通用的类型：`std::common_type<>::type`

```cpp
#include<type_traits>
template<typename T1, typename T2>
std::common_type_t<T1, T2> max(T1 a, T2 b)
{
  return b < a ? a : b;
}
```

在C++14中，我们可以使用`std::common_type_t<T1,T2>`来代替`std::common_type<T1,T2>::type`

缺省的模板参数（也就类似前面的第一个，再定义一个参数）

```cpp
#include<type_traits>
template<typename T1, typename T2, typename RT = std::decay_t<decltype(true ? T1() : T2())>>
RT max(T1 a, T2 b)
{
  return b < a ? a : b;
}
```

或者：

```cpp
#include<type_traits>
template<typename T1, typename T2, typename RT = std::common_type_t<T1,T2>>
RT max(T1 a, T2 b)
{
  return b < a ? a : b;
}
```

<hr>

重载函数模板

与普通函数类似，函数模板也能重载（或者说叫特例化）

```cpp
// maximum of two int values:
int max(int a, int b)
{
	return b < a ? a : b;
}

// maximum of two values of any type:
template<typename T>
T max(T a, T b)
{
	return b < a ? a : b;
}

int main()
{
	::max(7, 42);// calls the nontemplate for two ints
	::max(7.0, 42.0);// calls max<double> (by argument deduction)
	::max('a', 'b');//calls max<char> (by argument deduction)
	::max<>(7, 42);// calls max<int> (by argument deduction)
	::max<double>(7, 42);// calls max<double> (no argument deduction)
	::max('a', 42.7);//calls the nontemplate for two ints
}
```

特别点：

```cpp
template<typename T1, typename T2>
auto max (T1 a, T2 b)//max1
{
	return b < a ? a : b;
}
template<typename RT, typename T1, typename T2>
RT max (T1 a, T2 b)//max2
{
	return b < a ? a : b;
}

auto a = ::max(4, 7.2);//使用max1
auto b = ::max<long double>(7.2, 4);//使用max2

auto c = ::max<int>(4, 7.2);//ERROR，二义性错误，匹配性一致
```

模板中，我们往往不使用内联，不使用`constexpr`关键字，更偏爱与值传递（这些都不是定死的，而是在特殊情形有特殊功能的）