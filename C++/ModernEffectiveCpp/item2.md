<h2>条款2：理解auto类型的推导</h2>

在之前的一篇文章中已经较为详细地介绍了`auto`的类型推导，除了一个例外，`auto`类型推导就是模板类型推导。

```cpp
template<typename T>
void func_for_x(T param);//推导x上的类型的模板

func_for_x(27);


template<typename T>
void func_for_cx(const T param);//推导cx上的模板

func_for_cx(x);


template<typename T>
void func_to_rx(const T& param);//推导rx上的模板

func_for_rx(x);
```

`auto`与条款1中模板类型推导之间的关系：

```cpp
//情况1
auto x = 27;
const auto cx = x;
const auto& rx = x;

//情况2
auto&& uref1 = x;
auto&& uref2 = cx;
auto&& uref3 = rx;

//情况3
const char name[] = "Hello";
auto arr1 = name;
auto& arr2 = name;

void someFunc(int, double);

auto& func2 = someFunc;
```

这几乎就是完全一样的存在，但是有一种情况是不一致的（对待花括号初始化的情况），下面我们想得到一个拥有值为27的`int`：

```cpp
//C++98
int x1 = 27;
int x2(27);

//C++11
int x3 = {27};
int x4{27};

//使用auto来声明变量，这四个并不完全一致
auto x1 = 27;
auto x2(27);
auto x3 = {27};
auto x4{27};
```

上面的`auto`建立得到的头两个的确声明了一个初始化值为27的`int`，但是后面2个声明了一个`std::initializer_list<int>`的变量，包含了一个单一元素27。

因为我们有这样的推导：

```cpp
auto x5 = {1, 2, 3.0};//编译错误，无法得到推导得到std::initializer_list<T>的T类型，因为花括号中数值不唯一
```

而且不能够直接推导类型`std::initializer_list<T>`，（因为这玩意也是一个模板）即

```cpp
template<typename T>
void f(T param);

f({11, 23, 9});//编译错误，无法推导得到T的类型

//只有你指明是std::initializer_list<T>才能编译通过
template<typename T>
void f(std::initializer_list<T> initList);

f({11, 23, 9});//编译通过
```

也就是说，使用`auto`声明并用花括号初始化类型推导的就是`std::initializer_list<T>`，一个经典的错误就是被误声明为`std::initializer_list`，实际上是另一个类型。

C++14额外允许了`auto`表示推导的函数返回值，而且lambda允许参数声明里面使用`auto`，但是这里使用的是模板的类型推导，而不是上面介绍的`auto`形式：

```cpp
auto createInitList()//in C++14
{
    return {1, 2, 3};//编译错误，无法推导{1, 2, 3}类型
}
```

```cpp
std::vector<int> v;
	
auto resetV = [&v](const auto& newValue){v = newValue;};//in C++14

resetV({1, 2, 3});//编译错误，无法推导{1, 2, 3}类型
```

<hr>

<h2>综上，本item需要记住：</h2>

+ `auto`类型推导通常和模板类型推导类似，但是`auto`类型推导假定花括号初始化代表的类型是`std::initializer_list`，但是模板类型推导却不是这样
+ `auto`在函数返回值或者lambda参数里面执行模板的类型推导，而不是通常意义的`auto`类型推导