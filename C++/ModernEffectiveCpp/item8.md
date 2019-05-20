##优先使用`nullptr`而不是`0`或者`NULL`

`0`是一个`int`类型而不是指针，当编译器读到一个`0`，但上下文中只有一个指针用到了它，编译器才会把`0`解释为空指针，但这只是权宜之策。**要记住，`0`是一个`int`，而不是指针！对于`NULL`同理，他们都不是指针类型**

在`C++98`中，重载指针和整数类型的函数行为会令人吃惊，也就是传递`0`或者`NULL`将不会调用那个指针重载的函数，所以当时程序员不被允许重载指针和整数类型

```cpp
void f(int);
void f(double);
void f(void*);

f(0);//调用f(int)
f(NULL);//可能无法编译，可能调用f(int)，但是不会调用f(void*)
```

`nullptr`的优势是它不再是一个整数类型，也不是严格意义上的指针类型，而是可以想象成一个可以指向任意类型的指针。类型是`std::nullptr_t`，可以隐式转换为所有的原始指针，通过它作为参数可以调用到`f(void*)`这个重载函数，因为它不能被视作整数类型

而且它能够提高如跟`auto`有关的代码的清晰度，比如类型为`auto`的变量执行分支语句`if(result == 0)`我们读不懂这到底是`result`是什么类型的，而`if(result == nullptr)`就消灭了歧义，提高可读性

在模板编程时，`nullptr`也十分有用。假设只有对应的互斥量被锁定才会调用这些函数，每个函数参数是不同类型的指针：

```cpp
int f1(std::shared_ptr<Widget> spw);
double f2(std::unique_ptr<Widget> upw);
bool f3(Widget* pw);

std::mutex f1m, f2m, f3m;
using MuxGuard = std::lock_guard<std::mutex>;

{
    MuxGuard g(f1m);//为f1锁定互斥量
    auto result = f1(0);//把0当做空指针作为参数传给f1
}//解锁互斥量

{
    MuxGuard g(f2m);
    auto result = f2(NULL);//把NULL当做空指针作为参数传给f2
}

{
    MuxGuard g(f3m);
    auto result = f3(nullptr);//把nullptr当做空指针作为参数传给f3
}
```

这种代码重复模式——锁定互斥量，调用函数，解锁互斥量等重复很令人厌倦，所以为了避免这些风格的代码，我们可以通过模板进行封装：

```cpp
int f1(std::shared_ptr<Widget> spw);
double f2(std::unique_ptr<Widget> upw);
bool f3(Widget* pw);

template <typename FuncType,
    	  typename MuxType,
          typename PtrType>
decltype(auto) lockAndCall(FuncType func, MuxType& mutex, PtrType ptr)
{
    MuxGuard g(mutex);
    return func(ptr);
}

auto result1 = lockAndCall(f1, f1m, 0);//编译错误
...
auto result2 = lockAndCall(f2, f2m, NULL);//编译错误
...
auto result3 = lockAndCall(f3, f3m, nullptr);//正确
```

这里result1错误是因为`0`的类型是`int`，在`lockAndCall`中调用了`func`，传入的是`int`类型，这与`f1`所期待的`std::share_ptr<Widget>`的参数不兼容，引发错误；同理`NULL`也会引发相似的错误，而是用`nullptr`时，`ptr`类型推导为`std::nullptr_t`，当其被传递给`f3`时自动转换为`Widget*`（`std::nullptr_t`可以隐式转换成任意类型指针）

##需要记住的东西

+ 相较于`0`和`NULL`，优先使用`nullptr`
+ 避免整数类型和指针类型之间的重载