<h2>前言</h2>

C++11最重要的特性是：移动语义。右值代表你可以引用的临时对象。可以进行取址的都是左值（个人总结：具名对象都是左值）

形参只能是左值，而实参可以是右值或左值

```cpp
class Widget{
public:
    Widget(Widget&& rhs);//rhs是一个左值，尽管他的类型是一个右值引用类型（同理，所有参数都是左值）
    ...
};
```

这在完美转发中很重要，可以传递给一个函数的实参再传递给第二个函数，保证原始参数的左值/右值特性依然是保留的

通过lambda表达式创建的函数对象被称为闭包，通常不做区分。

<hr>

<h2>类型推导</h2>

<h3>条款1：理解模板类型推导</h3>

C++98：函数模板；C++11：增加`auto`和`decltype` ；C++14：继续拓展`auto`和`decltype`使用情况。

这里存在了3种情况：

+ `ParamType`是一个非通用的引用或一个指针（最简单的情况）

```cpp
template<typename T>
void f(T& param);

int x = 27;
const int cx = x;
const int& rx = x;

f(x);//T推导为int，param类型是int&

f(cx);//T推导为const int，param的类型是const int&

f(rx);//T推导为const int，param的类型是const int&
```

这里随着`cx`和`rx`两者被钦定为`const`类型变量，所以`T`也被推导成`const int`，参数被推导为`const int&`以保留了对象的常量特性。即，**对象的`const`特性是`T`类型推导的一部分得以保留**

但是第三个例子中的`rx`是一个左值引用，但是`T`还是被推导成非引用吗，即**引用特性会被类型推导所忽略**

当以下情况时效果也是类似的，也是完全符合我们预期的：

```cpp
void f(const T& param);//上面传入得到的T推导都是int，param类型都是const int&
void f(T* param);//略
```

+ `ParamType`是一个通用的引用

**如果`expr`是一个左值，`T`和`ParamType`会被推导成左值引用，这是模板类型`T`被推导成为引用的唯一情况（发生引用折叠）**，如果是一个右值，就按照普通的法则进行推导

```cpp
template<typename T>
void f(T&& param);//通用引用

int x = 27;
const int cx = x;
const int& rx = x;

f(x);//x是左值，所以T是int&，param类型是int&

f(cx);//cx是左值，所以T是const int&，param类型是const int&

f(rx);//rx是左值，所以T是const int&，param类型是const int&

f(27);//27是右值，所以T是int，param类型是int&&
```

+ `ParamType`既不是指针也不是引用

```cpp
template<typename T>
void f(T param);//param现在是pass-by-value
```

`param`就是完全给他的参数一份全新的拷贝，如果`expr`是一个引用，则忽略引用部分，而在忽略引用的同时，也会忽略`const`（和`volatile`）

```cpp
int x = 27;
const int cx = x;
const int& rx = x;

f(x);//T和param类型都是int

f(cx);//T和param类型都是int

f(rx);//T和param类型都是int
```

因为这里`param`和`cx`等是相互独立的对象（只是一份拷贝），`cx`的不可修改性与`param`的可修改性之间不应该有关联。

但是引用的`const`或者指针指向`const`对象，这个特性将会在类型推导时保留

```cpp
template<typename T>
void f(T param);

const char* const ptr = "Hello";

f(ptr);//指针是按值传递的，也就是推导得到const char*，即：可修改的指针，指向一个const的字符串
```

+ 数组作为参数

**注意：数组类型与指针类型是不一样的**，尽管在很多情况下，一个数组会退化成一个指向第一个元素的指针

```cpp
const char name[] = "Hello";
const char *ptrToName = name;

template<typename T>
void f(T param);

void myFunc(int param[]);
void myFunc(int* param);//两个函数是一致的

f(name);//T被推导成为const char*
```

但是：

```cpp
template<typename T>
void f(T& param);

f(name);//T被推导成数组，即const char[6]，参数被推导为const char(&)[6]
```

我们可以声明数组的引用来创造出一个推导出一个数组包含元素长度的模板

```cpp
template<typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept//数组参数是匿名的
{
    return N;
}
```

+ 函数作为参数

函数类型在很多时候也会退化成为函数指针：

```cpp
void someFunc(int, double);

template <typename T>
void f1(T param);//在f1中，参数直接按值传递

template <typename T>
void f2(T& param);//在f2中，参数按引用传递

f1(someFunc);//param被推导成函数指针，类型为void(*)(int, double)
f2(someFunc);//param被推导成函数指针，类型为void(&)(int, double)
```
<hr>

<h2>综上，本条款我们要记住以下内容：</h2>

+ 在模板类型推导的时候，有引用特性的参数的引用特性会被忽略
+ 在推导通用引用参数的时候（`void f(T&& param);`），左值会被特殊处理
+ 在推导按值传递的参数时候，`const`和/或`volatile`参数会被视为非`const`和非`volatile`
+ 在模板类型推导的时候，参数如果是数组或者函数名称，他们会被退化成指针，除非是用在初始化引用类型
