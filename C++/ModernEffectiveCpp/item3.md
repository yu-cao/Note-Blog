<h2>条款3：理解decltype</h2>

给定一个变量名或者表达式，`decltype`会告诉你这个变量名或表达式的类型。一般情况下，`decltype`只是会复述一遍你给他的变量名/表达式的类型：

```cpp
bool f(const Widget& w);  //decltype(f)为bool(const Widget&)
std::vector<int> vec;
if(vec[0]==0)    //decltype(vec[0])为int&
```

C++14允许多语句lambda表达式均可以推导返回类型，参考下面的代码：

```cpp
template <typename Container, typename Index>
auto authAndAccess(Container& c, Index i)->decltype(c[i])//尾置返回类型，也可以不用，auto也会正确推导得出返回类型
{
    authenticateUser();
    return c[i];
}
```

对于下面代码，将无法通过编译（如果像上面一样用显式的尾置返回类型编译时可以通过的）：

```cpp
template <typename Container, typename Index>
auto authAndAccess(Container& c, Index i)
{
    authenticateUser();
    return c[i];
}

std::deque<int> d;
...
authAndAccess(d, 5) = 10;//编译错误
```

原因是：`d[5]`返回的是`int&`，但是这个`auto`返回类型会把`&`剥离而返回类型为`int`，这是一个右值，给一个右值`int`赋值将引发编译错误

而为了能够正常工作，需要对返回值使用`decltype`类型推导，指定返回的类型就是`c[i]`的返回类型：用`auto`推导类型，用`decltype`推导规则：

```cpp
template <typename Container, typename Index>
decltype(auto) authAndAccess(Container& c, Index i)
{
    authenticateUser();
    return c[i];
}
```

这个策略也能让我们快乐地声明一些变量：

```cpp
Widget w;
const Widget& cw = w;
auto myWidget1 = cw;//类型为Widget，丢失了引用和const

decltype(auto) myWidget2 = cw;//类型为const Widget&，与cw类型完全一致
```

这里有一个问题：`Container`参数是通过非`const`左值引用传入，这意味着：**我们无法将右值传入这个函数中**，传入右值这个操作看上去很疯狂，返回的引用在创建它的语句结束的地方就会被悬空，但其实很还是有意义的，假如用户只想拷贝一个临时容器中的一个元素：

```cpp
std::deque<std::string> makeStringDeque();
auto s = authAndAccess(makeStringDeque(), 5);
```

我们可以使用**统一引用**的思想进行处理，这里我们依然保持住值传递的方法，但是我们模糊了其类型，也就可能出现不必要的复制等的性能开销，但值传递的方式与标准库统一且比较合理，即改为：

```cpp
//C++14
template <typename Container, typename Index>
decltype(auto) authAndAccess(Container&& c, Index i)
{
    authenticateUser();
    return std::forward<Container>(c)[i];
}

//或C++11
template <typename Container, typename Index>
auto authAndAccess(Container&& c, Index i)->decltype(std::forward<Container>(c)[i])
{
    authenticateUser();
    return std::forward<Container>(c)[i];
}
```

<hr>

回到最初，我们说过：`decltype`会告诉你这个变量名或表达式的类型，如果是一个比变量名更复杂的左值表达式（即使只是在外面简单加一个括号），则`decltype`保证返回的类型是左值引用，这通常很好。

但是我们在C++14后要注意这样的一个问题：

```cpp
decltype(auto) f1()
{
    int x = 0;
    ...
    return x;//decltype(x)是int，返回值是int
}

decltype(auto) f2()
{
    int x = 0;
    return (x);//decltype((x))是int&，返回值是int&
}
```

**这两个的返回值是不同的，下面这个函数对于一个已经销毁的局部变量传递回一个引用将很可能误操作引发UB**。经验就是当使用`decltype(auto)`时要多留心一些。被推导的表达式中看上去无关紧要的细节都可能影响`decltype`返回的类型。

<hr>

<h2>总之，本节需要记住的内容</h2>

+ `decltype`几乎总是得到一个变量或表达式的类型而不需要任何修改
+ 对于非变量名的类型为`T`的左值表达式，`decltype`总是返回`T&`
+ C++14 支持`decltype(auto)`，它的行为就像`auto`,从初始化操作来推导类型，但是它推导类型时使用`decltype`的规则