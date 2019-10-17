## Chapter6 Move Semantics and enable_if<>

C++11引入了最重要的功能之一就是移动语义。通过将内部资源从源对象移动到目标对象，而不是copy，可以通过这个来优化复制与分配。

移动语义对于模板的设计带来了重大的影响。

### 完美转发

我们想写这样的泛型代码，对于传过来的基本属性：

+ 应该转发可以修改的对象，以便于依然可以对其进行修改
+ 常量对象应该作为只读对象转发
+ 可移动的对象（也就是即将过期的对象，右值）应该保持作为可移动对象进行转发

```cpp
class X{};

void g(X&)
{
	std::cout << "g() for variable\n";
}

void g(X const&)
{
	std::cout << "g() for constant\n";
}

void g(X&&)
{
	std::cout << "g() for movable object\n";
}

void f(X& val)
{
	g(val);
}

void f(X const& val)
{
	g(val);
}

void f(X&& val)
{
	g(std::move(val));
}

int main()
{
	X v;
	X const c;
	f(v);
	f(c);
	f(X());
	f(std::move(v));
}
```

统一化到一个模板下：

```cpp
template<typename T>
void f (T val)
{
  g(T);
}
```

但是上面的代码对于右值的转发会失败，为此C++11引入完美转发：

```cpp
template<typename T>
void f(T&& val)
{
  g(std::forward<T>(val));
}
```

注意`forward`和`move`的区别，`move`通过传递的参数触发移动语义，`forward`通过传递的模板参数“转发”了潜在的移动语义

### 特别的成员函数模板

构造函数等也是类似的，通过类似的函数模板处理：

```cpp
template<typename STR>
explicit Person(STR&& n) : name(std::forward<STR>(n))
{
  std::cout << "TMPL-CONSTR for '" << name << "'\n";
}
```

但是这里有一个问题，如果不是`std::string`，这个构造就压根不对，我们接下来就解决这个问题

### 通过`enable_if<>`禁止模板

从C++11起，标准库提供一个帮助模板`std::enable_if<>`来在编译期ignore函数模板

```cpp
template<typename T>
typename std::enable_if<(sizeof(T) > 4)>::type
foo()
{}
```

这里定义了`foo<>()`来屏蔽掉`sizeof(T) > 4`为`false`的部分。如果`sizeof(T) > 4`结果为`true`，模板实例会拓展为：

```cpp
void foo(){}
```

也就是说：

+ 如果表达式结果为`true`，它的类型为：
  + 如果没有第二个模板参数传入，就类型为`void`
  + 否则就显示第二个模板参数的类型
+ 如果结果为`false`，这个`type`结果是未定义的。因为模板特性SFINAE（substitution failure is not an error），这将会影响函数模板当这个`enable_if`表达式被ignore时

从C++14起，所有的类型特征都会有一个别名模板：`std::enable_if_t<>`，使得可以跳过`typename`和`::type`：

```cpp
template<typename T>
std::enable_if_t<(sizeof(T) > 4)>
foo()
{}

//如果需要第二个参数
template<typename T>
std::enable_if_t<(sizeof(T) > 4)， T>
foo()
{}
```

但是很多时候在声明中间包含`enable_if`这个操作很笨拙，所以常用方法是使用具有默认值的附加函数模板参数：

```cpp
template<typename T, typename = std::enable_if_t<(sizeof(T) > 4)>>
void foo()
{}
```

如果`sizeof(T) > 4`，可以拓展为：

```cpp
template<typename T, typename = void>
void foo()
{}
```

如果觉得这还是太笨拙了，你还想更加显式的约束或要去，你能通过别名模板为它定义你自己的名字

```cpp
template<typename T>
using EnableIfSizeGreater4 = std::enable_if_t<(sizeof(T) > 4)>;

template<typename T, typename = EnableIfSizeGreater4<T>>
void foo()
{}
```

### 使用`enable_if<>`

这里，我们就能解决上面的问题：如果STR是一个正确的类型（`std::string`或可以转换到这个的类型），就pass，否则就禁止

```cpp
template<typename STR, typename = std::enable_if_t<std::is_convertible_v<STR, std::string>>>
Person(STR &&n) : name(std::forward<STR>(n))
{
  std::cout << "TMPL-CONSTR for '" << name << "'\n";
}
```

如果可以拓展，则声明会变成：

```cpp
template<typename STR, typename = void>
Person(STR&& n);
```

如果不能拓展，则整个函数会被ignore

用别名方法语法如下：

```cpp
template<typename T>
using EnableIfString = std::enable_if_t<std::is_convertible_v<T, std::string>>;

template<typename STR, typename = EnableIfString<STR>>
Person();
```

运行如下：

```cpp
class Person
{
private:
	std::string name;
public:
	template<typename STR, typename = std::enable_if_t<std::is_convertible_v<STR, std::string>>>//C++17
	Person(STR &&n) : name(std::forward<STR>(n))
	{
		std::cout << "TMPL-CONSTR for '" << name << "'\n";
	}

	Person(Person const &p) : name(p.name)
	{
		std::cout << "COPY-CONSTR Person '" << name << "'\n";
	}

	Person(Person &&p) : name(std::move(p.name))
	{
		std::cout << "MOVE-CONSTR Person '" << name << "'\n";
	}
};

int main()
{
	std::string s = "sname";
	Person p1(s);
	Person p2("tmp");
	Person p3(p1);
	Person p4(std::move(p1));
}

//Output:
TMPL-CONSTR for 'sname'
TMPL-CONSTR for 'tmp'
COPY-CONSTR Person 'sname'
MOVE-CONSTR Person 'sname'
```

除了使用`std::is_convertible<>`之外，还有一种选择：通过使用`std::is_constructible<>`，我们能够允许显式转换以适用于初始化。但是，注意要调转一下模板的参数顺序：

```cpp
template<typename T>
using EnableIfString = std::enable_if_t<std::is_constructible_v<std::string, T>>;
```

#### 禁止特殊的成员函数

请注意，通常我们不能使用`enable_if<>`禁用预定义的复制/移动构造函数和/或赋值运算符。原因是成员函数模板从不算作特殊成员函数，例如在需要复制构造函数时会被忽略。因此，使用此声明

```cpp
class C
{
public:
	template<typename T>
	C(T const &)
	{
		std::cout << "tmpl copy constructor\n";
	}
};
```

当请求C的副本时，仍使用预定义的副本构造函数（实际上，无法使用成员模板，因为无法指定或推导其模板参数T）：

```cpp
C x;
C y(x);//still uses the predefined copy constructor (not the member template)
```

删除预定义的复制构造函数不是解决方案，因为这时尝试复制C会导致error。

不过有一个tricky的解决方法：我们可以为`const volatile`参数声明一个副本构造函数，并将其标记为`=delete`。这样做可以防止隐式声明另一个副本构造函数。有了它，我们可以定义一个构造器模板，该模板将会比对非volatile类型的删除的拷贝构造器更受preferred

```cpp
class C
{
public:
	C(C const volatile&) = delete;

	template<typename T>
	C(T const &)
	{
		std::cout << "tmpl copy constructor\n";
	}
};
```

现在上面的copy变得正常了。然后在这样的模板构造函数中，我们可以使用`enable_if<>`施加其他约束。例如，如果template参数是整数类型，为了防止能够复制类模板`C<>`的对象，我们可以实现以下内容：

```cpp
template<typename T>
class C
{
public:
	C(C const volatile&) = delete;

	template<typename U, typename = std::enable_if_t <!std::is_integral<U>::value>>
	C(C<U> const &)
	{}
};
```

### 使用Concept去简化`enable_if<>`表达式

即使使用别名模板，enable_if语法也很笨拙，因为它使用了一种变通方法：为了获得理想的效果，我们添加了一个额外的模板参数，并“滥用”该参数以提供对功能模板的特定要求。完全可用。像这样的代码很难阅读，并使其余功能模板难以理解。

原则上，我们只需要一种语言功能，就可以在不满足要求/约束的情况下，以一种导致功能被忽略的方式来制定功能的要求或约束。我们能够使用自己的简单语法为模板制定要求/条件。

不幸的是，Concept没有成为C ++ 17标准的一部分。一些编译器为这种功能提供实验性支持，但是，概念很可能成为C ++ 17之后的下一个标准的一部分。

使用Concept，代码可以变成：

```cpp
template<typename STR>
requires std::is_convertible_v<STR, std::string>
Person(STR&& n) : name(std::forward<STR>(n))
{}
```

静待C++20的正式调整。