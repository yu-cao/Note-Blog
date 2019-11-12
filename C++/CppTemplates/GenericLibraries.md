## Chapter 11 Generic Libraries

许多库包含客户端代码传递必须被“调用”的实体的接口。示例包括必须在另一个线程上调度的操作，描述如何对值进行哈希处理以将其存储在哈希表中的函数，一个对象。描述了对集合中的元素进行排序的顺序，以及提供一些默认参数值的通用包装。标准库也不例外：它定义了许多采用此类可调用实体的组件。

### 可调用

许多库包含客户端代码传递必须被“调用”的实体的接口。示例包括必须在另一个线程上调度的操作，描述如何对值进行哈希处理以将其存储在哈希表中的函数，一个对象。描述了对集合中的元素进行排序的顺序，以及提供一些默认参数值的通用包装。标准库也不例外：它定义了许多采用此类可调用实体的组件。

在这种情况下使用的一个术语是回调。传统上，该术语保留为作为函数调用参数（而不是例如模板参数）传递的实体，并且我们保持这种传统。例如，排序功能可以包括作为“排序标准”的回调参数，调用该回调参数可以确定一个元素是否按照所需的排序顺序位于另一个元素之前。

在C ++中，有几种类型可以很好地用于回调，因为它们都可以作为函数调用参数进行传递，并且可以使用语法`f(...)`直接调用：

+ Pointer-to-function types
+ 有`operator()`（包括lambda）的类类型，
+ 具有转换功能的类类型产生指向函数的指针或指向函数的引用

总之，这些类型被称为*function object types*，类型的值称为*function object*

C++标准库引入了更广泛的可调用类型（callable type）的概念，该类型可以是函数对象类型或指向成员的指针。可调用类型的对象是callable object，为方便起见，我们将其称为callable

#### 支持函数对象

以下就是我们自己去实现标准库的`std::for_each`功能

```cpp
template<typename Iter, typename Callable>
void foreach(Iter current, Iter end, Callable op)
{
	while(current!=end)
	{
		op(*current);
		++current;
	}
}

void func(int i)
{
	std::cout << "func() called for:" << i << '\n';
}

class FuncObj
{
public:
	void operator()(int i) const
	{
		std::cout << "FuncObj::op() called for:" << i << '\n';
	}
};

int main()
{
	std::vector<int> primes = {2, 3, 5, 7, 11, 13, 17, 19};
	foreach(primes.begin(), primes.end(), func);//range function as callable(decays to pointer)
	foreach(primes.begin(), primes.end(), &func);//range function as callable
	foreach(primes.begin(), primes.end(), FuncObj());//range function object as callable
	foreach(primes.begin(), primes.end(), [](int i)//range lambda as callable
	{ std::cout << "Lambda call for: " << i << '\n'; });
}
```

当我们将函数名称作为函数参数传递时，我们实际上并没有传递函数本身，而是传递了指向它的指针或引用。与数组一样，函数参数在通过值传递时会衰减为指针，对于类型为模板参数的参数，将推导出函数的指针类型。就像数组一样，函数可以通过引用传递而不会衰减。但是，函数类型不能真正用`const`限定。如果我们使用`Callable const＆`类型声明`foreach()`的最后一个参数，则`const`将被忽略

我们的第二个调用通过传递函数名称的地址来显式地获取函数指针。这等效于第一次调用（函数名隐式衰减为指针值），但也许更清晰一些

第三个调用：传递函数子（functor）时，我们将类类型对象作为可调用对象传递。通过类类型进行调用通常相当于调用其`operator()`。所以调用`op(*current);`也就展开成`op.operator()(*current);`。请注意，在定义`operator()`时，通常应将其定义为常量成员函数

类类型对象也可以隐式转换为指针或对代理调用函数的引用：

```cpp
op(*current);     =>     (op.operator F())(*current);
```

其中F是类类型对象可以转换为的指向函数或指向函数的指针的类型。这是相对不寻常的

Lambda表达式产生函数子functor（称为闭包），因此，这种情况与函数子情况没有区别。但是，Lambda是引入函数子的非常方便的简写形式，因此自C ++ 11起，它们通常出现在C ++代码中。

有趣的是，以`[]`开头的lambda（无捕获）会产生一个转换操作符，指向函数指针。但是，永远不要选择它作为代理调用函数，因为它总是比闭包的普通`operator()`更差

#### 处理成员函数和拓展参数

在上一个示例中未使用一个可能的实体进行调用：成员函数。那是因为调用非静态成员函数通常涉及使用`object.memfunc(...)`或`ptr-> memfunc(...)`之类的语法指定要对其应用调用的对象，并且与通常的模式`function-object(...)`不匹配。

幸运的是，自C++17以来，C++标准库提供了一个实用程序`std::invoke()`，可以很方便地将这种情况与普通的函数调用语法形式统一起来，从而允许以单一形式调用任何可调用对象。我们的`foreach()`模板的以下实现使用`std::invoke()`

```cpp
#include <utility>
#include <functional>

template<typename Iter, typename Callable, typename... Args>
void foreach(Iter current, Iter end, Callable op, Args const&... args)
{
	while (current != end)
	{
		std::invoke(op, args..., *current);//call passed callable with any additional args and the current element
		++current;
	}
}
```

在这里，除了可调用参数之外，我们还接受任意数量的附加参数。然后，`foreach()`模板使用给定的callable调用`std::invoke()`，然后调用其他给定的参数以及引用的元素。` std::invoke()`按以下方式处理：

+ 如果可调用的是一个指向成员的指针，则它将第一个附加参数用作此对象。其余所有其他参数只是作为参数传递给可调用对象。
+ 否则，所有其他参数都将作为参数传递给可调用对象。

请注意，我们不能在此处对可调用或其他参数使用完美转发：第一个调用可能会“窃取”它们的值，从而导致在后续迭代中调用`op`的意外行为。

通过此实现，我们仍然可以编译对上述`foreach()`的原始调用。现在，此外，我们还可以将其他参数传递给callable，而callable可以是成员函数。以下客户端代码对此进行了说明：

```cpp
template<typename Iter, typename Callable, typename... Args>
void foreach(Iter current, Iter end, Callable op, Args const &... args)
{
	while (current != end)
	{
		std::invoke(op, args..., *current);
		++current;
	}
}

class MyClass
{
public:
	void memfunc(int i) const
	{
		std::cout << "MyClass::memfunc() called for:" << i << '\n';
	}
};

int main()
{
	std::vector<int> primes = {2, 3, 5, 7, 11, 13, 17, 19};
	foreach(primes.begin(), primes.end(), [](std::string const &prefix, int i)
	{ std::cout << prefix << i << '\n'; }, "- value:");

	MyClass obj;
	foreach(primes.begin(), primes.end(), &MyClass::memfunc, obj);
}

//Output:
- value:2
- value:3
- value:5
- value:7
- value:11
- value:13
- value:17
- value:19
MyClass::memfunc() called for:2
MyClass::memfunc() called for:3
MyClass::memfunc() called for:5
MyClass::memfunc() called for:7
MyClass::memfunc() called for:11
MyClass::memfunc() called for:13
MyClass::memfunc() called for:17
MyClass::memfunc() called for:19
```

`foreach()`的第一次调用将其第四个参数（字符串`- value:`）传递给lambda的第一个参数，而向量中的当前元素绑定到lambda的第二个参数。第二次调用将成员函数`memfunc()`作为第三个参数传递给要作为第四个参数传递的obj调用。

#### 交换函数调用

`std::invoke()`的常见应用是包装单个函数调用（例如，记录调用，测量其持续时间或准备一些上下文，例如为其启动新线程）。现在，我们可以通过完美地转发可调用参数和所有传递参数来支持移动语义：

```cpp
template<typename Callable, typename... Args>
decltype(auto) call(Callable &&op, Args &&... args)
{
	return std::invoke(std::forward<Callable>(op), std::forward<Args>(args)...);
}
```

另一个有趣的方面是如何处理被调用函数的返回值，以将其“完全转发”回给调用者。为了支持返回的引用（例如`std::ostream＆`），所以必须使用`decltype(auto)`而不是`auto`：

如果要将`std::invoke()`返回的值临时存储在变量中以在执行其他操作后返回它（例如，处理返回值或记录调用结束），则还必须声明具有`decltype(auto)`的临时变量：

```cpp
decltype(auto) ret{std::invoke(std::forward<Callable>(op), std::forward<Args>(args)...)};
return ret;
```

请注意，使用`auto&&`声明`ret`是不正确的。作为参考，`auto &&`会将返回值的生存期延长到其作用域的末尾，但不超出函数调用者的return语句

但是，使用`decltype(auto)`也会有一个问题：如果可调用对象的返回类型为`void`，则不允许将`ret`初始化为`decltype(auto)`，因为`void`是不完整的类型。解决策略：

+ 在该语句之前的行中声明一个对象，该对象的析构函数执行您要实现的可观察的行为：

```cpp
struct cleanup
{
  ~cleanup(){
    ...//code to perform on return
  }
}dummy;
return std::invoke(std::forward<Callable>(op), std::forward<Args>(args)...);
```

+ 实现不同的代码执行`void`与`non-void`不同的case

```cpp
template<typename Callable, typename... Args>
decltype(auto) call(Callable &&op, Args &&... args)
{
	if constexpr(std::is_same_v<std::invoke_result_t<Callable, Args...>, void>)
	{
		std::invoke(std::forward<Callable>(op), std::forward<Args>(args)...);
		...
		return;
	}
	else
	{
		decltype(auto) ret{std::invoke(std::forward<Callable>(op), std::forward<Args>(args)...)};
		...
		return ret;
	}
}
```

### 其他组件去实现通用库

#### Type Traits

标准库提供了多种称为类型特征的实用程序，使我们能够评估和修改类型。这支持了通用代码必须适应或对其实例化的类型的功能做出反应的各种情况。例如

```cpp
template<typename T>
class C
{
	static_assert(!std::is_same_v<std::remove_cv_t<T>, void>, "invalid instantiation of class C for void type");
public:
	template<typename V>
	void f(V&& v)
	{
		if constexpr (std::is_reference_v<T>)
		{}
		if constexpr (std::is_convertible_v<std::decay_t<V>, T>)
		{}
		if constexpr (std::has_virtual_destructor_v<V>)
		{}
	}
};
```

通过检查某些条件，我们可以在模板的不同实现之间进行选择。在这里，我们使用了编译时if功能，该功能自C++17起可用，但是我们可以使用`std::enable_if`，偏特化或SFINAE来启用或禁用帮助程序模板。

```cpp
std::remove_const_t<int const&> //yields int const&
```

在这里，因为引用不是const（尽管您不能修改它），所以调用无效，并产生传递的类型。因此，删除引用和const的顺序很重要：

```cpp
std::remove_const_t<std::remove_reference_t<int const&>> //int
std::remove_reference_t<std::remove_const_t<int const&>> //int const
  
std::decay_t<int const&> //yields int
```

但是，这也会将原始数组和函数转换为相应的指针类型。

在某些情况下，类型特征也有要求。不满足这些要求会导致行为不确定。例如：

```cpp
make_unsigned_t<int> //unsigned int
make_unsigned_t<int const&> // undefined behavior
```

还有些操作的行为会出乎意料：

```cpp
add_rvalue_reference_t<int> //int&&
add_rvalue_reference_t<int const> //int const&&
add_rvalue_reference_t<int const&> //int const&（lvalue ref remains lvalue-ref）
```

在这里，我们可能期望`add_rvalue_reference`始终会产生右值引用，但是C ++的引用折叠规则会导致左值引用和右值引用的组合产生左值引用

虽然`is_copy_assignable`通常只检查是否可以将整数分配给另一个（检查左值的操作），但是`is_assignable`会考虑值类别（此处检查是否可以将prvalue分配给prvalue）

```cpp
is_copy_assignable_v<int> //yield true (generally, you can assign an int to an int)
is_assignable_v<int, int> //yidld false (can't call 42 == 42)
```

#### std::addressof()

`std::addressof<>()`函数模板产生对象或函数的实际地址。即使对象类型的运算符＆重载也可以使用。即使后者在某种程度上很少见，也可能会发生（例如，在智能指针中）。因此，如果需要任意类型的对象的地址，建议使用`addressof()`：

```cpp
template<typename T>
void f(T&& x)
{
  auto p = &x;
  auto q = std::addressof(x);
  ...
}
```

#### std::declval()

`std::declval<>()`函数模板可用作特定类型的对象引用的占位符。该函数没有定义，因此无法调用（也不会创建对象）。因此，它只能用于未求值的操作数（例如，`decltype`和`sizeof`构造的操作数）。因此，可以假设您具有相应类型的对象，而不是尝试创建对象。

```cpp
#include <utility>

template<typename T1, typename T2, typename RT = std::decay_t<decltype(true ? std::declval<T1>() : std::declval<T2>())>>
RT max(T1 a, T2 b)
{
  return b < a ? a : b;
}
```

为避免我们必须为T1和T2调用一个（默认的）构造函数以能够调用运算符`？：`在表达式中初始化`RT`，我们使用`std::declval`来“使用”相应类型的对象而不创建它们。但是，这仅在未评估的`decltype`上下文中才有可能。

不要忘记使用`std::decay <>`类型特征来确保默认返回类型不能作为引用，因为`std::declval()`本身会产生右值引用。否则，诸如`max（1、2）`之类的调用将获得`int &&`的返回类型。

### 完美的临时转发

```cpp
template<typename T>
void f(T&& t)//t is forwarding reference
{
  g(std::forward<T>(t));//perfectly forward passed argument t to g()
}
```

但是，有时我们必须以通用代码完美地转发不通过参数传递的数据。在这种情况下，我们可以使用`auto &&`创建可转发的变量。例如，假定我们已将对调用`get()`和`set()`的调用链接在一起，其中`get()`的返回值应完美转发到`set( )`

进一步假设我们需要更新代码以对`get()`产生的中间值执行一些操作。为此，我们将值保存在用`auto &&`声明的变量中

```cpp
template<typename T>
void foo(T x)
{
  auto&& val = get(x);
  ...
  // perfectly forward the return value of get() to set()
  set(std::forward<decltype(val)>(val));
}
```

这避免了中间值的多余副本。

### 引用作为模板参数

虽然不常见，但是模板类型参数可以成为引用类型。

```cpp
template<typename T>
void tmplParamIsReference(T)
{
	std::cout << "T is reference: " << std::is_reference_v<T> << '\n';
}

int main()
{
	std::cout << std::boolalpha;
	int i;
	int& r = i;
	tmplParamIsReference(i);//false
	tmplParamIsReference(r);//false
	tmplParamIsReference<int&>(i);//true
	tmplParamIsReference<int&>(r);//true
}
```

即使将参考变量传递给`tmplParamIsReference()`，模板参数T也会推导为参考类型的类型（因为对于参考变量v，表达式v具有参考类型；表达式的类型永远不会参考）。但是，我们可以通过显式指定T的类型来强制参考情况。

这样做可以从根本上改变模板的行为，并且很可能没有考虑到这种可能性来设计模板，从而触发错误或意外行为。

```cpp
template<typename T, T Z = T{}>
class RefMem
{
  private:
    T zero;
  public:
    RefMem() : zero(Z) {}
};
int null = 0;

int main()
{
  RefMem<int> rm1, rm2;
  rm1 = rm2;// OK
  
  RefMem<int&> rm3;//Error: invalid default value for N
  RefMem<int&, 0> rm4;//Error: invalid default value for N
  
  extern int null;
  RefMem<int&, null> rm5, rm6;
  rm5 = rm6;// Error: operator= is deleted due to reference member
}
```

这里有一个包含模板参数类型T的成员的类，并使用非零模板参数Z初始化，该参数的初始化值为零。用int类型实例化类按预期工作。但是，当尝试使用引用实例化它时，事情变得棘手：

+ 默认初始化不再起作用。
+ 您不再只能将0用作int的初始化程序。
+ 而且，也许最令人惊讶的是，赋值运算符不再可用，因为具有非静态引用成员的类已删除了默认赋值运算符。

同样，将引用类型用于非类型模板参数也很棘手，而且很危险。

```cpp
template<typename T, int& SZ>
class Arr
{
private:
	std::vector<T> elems;
public:
	Arr() : elems(SZ)
	{}
	void print() const
	{
		for(int i = 0; i < SZ; ++i)
		{
			std::cout << elems[i] << ' ';
		}
	}
};

int size = 10;

int main()
{
	Arr<int&, size> y;//Compile-time Error deep in the code of class std::vector<>
	Arr<int, size> x;//initializes internal vector with 10 elements
	
	x.print();//OK
	size += 100;//OOPS:modifies SZ in Arr<>
	x.print();//run-time Error: invalid memery access: loops over 120 elements
}
```

在这里，尝试为引用类型的元素实例化Arr会导致类`std::vector<>`的代码深处出现错误，因为无法将引用作为元素进行实例化

可能更糟的是由于将`size`参数作为参考而导致的运行时错误：它允许在容器不知道的情况下更改记录的大小值（即，大小值可能变为无效）。因此，使用大小的操作（如print（）成员）必然会遇到未定义的行为（导致程序崩溃或更糟）

请注意，将模板参数SZ更改为`int const＆`类型无法解决此问题，因为大小本身仍可修改。
可以说，这个例子牵强。但是，在更复杂的情况下，确实会发生此类问题。同样，在C ++17中，可以推导出非类型参数：

```cpp
template<typename T, decltype(auto) SZ>
class Arr;
```

使用`decltype(auto)`可以轻松生成引用类型，因此在这种情况下通常避免使用它（默认情况下使用`auto`）
因此，C ++标准库有时具有令人惊讶的规范和约束。例如：

```cpp
namespace std
{
  template<typename T1, typename T2>
  struct pair
  {
    T1 first;
    T2 second;
    
    //default copy/move constructors are OK even with reference
    pair(pair const&) = default;
    pair(pair&&) = default;
    
    //but assignment operator have to be defined to be available with references:
    pair& operator=(pair const& p);
    pair& operator=(pair&& p) noexcept{...};
  }
}
```

由于可能产生的副作用的复杂性，实例化C++ 17标准库类模板`std::optional<>`和`std::variant<>作为引用类型是非良定义的（至少在C ++ 17中）

```cpp
template<typename T>
class optional
{
  static_assert(!std::is_reference<T>::value, "Invalid instantiation of optional<T> for references");
  ...
}
```

参考类型通常与其他类型完全不同，并且要遵循几种独特的语言规则。例如，这会影响调用参数的声明以及定义类型特征的方式

### 延期计算

在实现模板时，有时会出现一个问题，即代码是否可以处理不完整的类型。考虑以下类模板：

```cpp
template<typename T>
class Cont
{
  private:
    T* elems;
  public:
    ...
}
```

到目前为止，此类可以与不完整的类型一起使用。例如，这对于引用其自身类型的元素的类很有用

```cpp
struct Node
{
  std::string value;
  Cont<Node> next;//only possible if Cont accepts incomplete types
};
```

但是，例如，仅通过使用某些特征，您可能会失去处理不完整类型的能力。例如

```cpp
template<typename T>
class Cont
{
  private:
    T* elems;
  public:
    ...
    typename std::conditional<std::is_move_constructible<T>::value, T&&, T&>::type foo();
}
```

在这里，我们使用特征`std::conditional`来确定成员函数`foo()`的返回类型是`T &&`还是`T＆`。该决定取决于模板参数类型`T`是否支持移动语义。
问题在于特质`std :: is_move_constructible`要求其参数为完整类型（且不是`void`或未知界限的数组）。因此，使用`foo()`的声明，struct节点的声明将失败。
我们可以通过用成员模板替换`foo()`来解决此问题，以便将`std :: is_move_constructible`的求值推迟到`foo()`的实例化点：

```cpp
template<typename T>
class Cont
{
  private:
    T* elems;
  public:
    template<typename D = T>
    typename std::conditional<std::is_move_constructible<D>::value, T&&, T&>::type foo();
};
```

现在，特征取决于模板参数`D`（默认为`T`，总之是我们想要的值），编译器必须等到`foo()`被调用为`Node`之类的具体类型，然后才能评估traits（此时，`Node`是一个完整类型；仅在定义时才不完整）

### 编写通用库时要考虑的事项

让我们列出实现通用库时要记住的一些事情：
+ 使用转发引用转发模板中的值。如果值不取决于模板参数，请使用`auto &&`
+ 当参数声明为转发引用时，传递左值时，请准备模板参数具有引用类型
+ 当根据模板参数需要对象的地址时，请使用`std::addressof()`，以避免在绑定到带有重载`operator＆`的类型的对象时引起意外
+ 对于成员函数模板，请确保它们的匹配度不比预定义的复制/移动构造函数或赋值运算符更好
+ 当模板参数可能是字符串文字且不按值传递时，请考虑使用`std::decay`
+ 如果根据模板参数有`out`或`inout`参数，请准备好处理可能会指定`const`模板参数的情况
+ 准备处理作为参考的模板参数的副作用。特别是，您可能要确保返回类型不能成为引用
+ 准备处理不完整的类型以支持例如递归数据结构
+ 对所有数组类型都重载，而不仅仅是`T [SZ]`
  