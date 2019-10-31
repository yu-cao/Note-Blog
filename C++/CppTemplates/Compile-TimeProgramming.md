## Chapter8 Compile-Time Programming

C++包含着一些在编译期计算简单值的方法，模板很大程度上拓展了这部分的可能性

在简单的情况下，我们可以决定是否使用某些模板代码或在不同的模板代码之间进行选择。但是，只要所有必要的输入可用，编译器甚至可以在编译时计算控制流的结果

C++对编译时编程的支持：

+ 在C++98之前，模板已经有能力在编译期进行计算，包括支持循环与分支路径选择（但是不直观，有滥用的嫌疑）
+ 通过偏特化，我们可以选择在特定要求下使用不同的类模板实现之间进行选择
+ 借助SFINAE原理，我们可以针对不同类型或不同约束在不同功能模板实现之间进行选择
+ 在C++11与C++14中，由于引入`constexpr`，可以直观地选择执行路径；从C++14起支持`for`循环，`switch`语句等
+ C++17开始，支持了编译期`if`，以根据编译时条件或约束条件舍弃语句。它甚至可以在模板之外工作

### 模板元编程

模板是在编译时实例化的。事实证明，可以将C++模板的某些功能与实例化过程结合起来，以在C++语言本身内部生成一种原始的递归“编程语言”。因此，可以将模板用于“计算程序”。 

```cpp
template<unsigned p, unsigned d>//p: number to check, d: current divisor
struct DoIsPrime
{
  static constexpr bool value = (p % d != 0) && DoIsPrime<p, d - 1>::value;
};

template<unsigned p>
struct DoIsPrime<p, 2>//end recursion if divisor is 2
{
  static constexpr bool value = (p % 2 != 0);
};

template<unsigned p>//primary template
struct IsPrime
{
  //static recursion with divisor from p/2:
  static constexpr bool value = DoIsPrime<p, p / 2>::value;
};

//special case
template<>
struct IsPrime<0> {static constexpr bool value = false;};
template<>
struct IsPrime<1> {static constexpr bool value = false;};
template<>
struct IsPrime<2> {static constexpr bool value = true;};
template<>
struct IsPrime<3> {static constexpr bool value = true;};
```

无论传递的模板参数p是素数，`IsPrime<>`模板都会在成员值中返回。为了实现这一点，它实例化了`DoIsPrime<>`，该方法递归扩展为表达式，以检查`p / 2`与`2`之间的每个除数`d`是否除数除以`p`，而没有余数。比如：

```cpp
IsPrime<9>::value
//拓展成为：
DoIsPrime<9, 4>::value
//再拓展成为：
9 % 4 != 0 && DoIsPrime<9, 3>::value
//再拓展为
9 % 4 != 0 && 9 % 3 != 0 && DoIsPrime<9, 2>::value
//再拓展为
9 % 4 != 0 && 9 % 3 != 0 && 9 % 2 != 0
//结果为false，因为9 % 3 == 0
```

### 使用`constexpr`进行计算

C++11引入了新功能`constexpr`，该功能大大简化了各种形式的编译时计算。特别是，只要输入正确，就可以在编译时评估`constexpr`函数。虽然在C++11中`constexpr`函数引入了严格的限制（例如，每个`constexpr`函数定义本质上都被限制为由`return`语句组成），但大多数此类限制在C++14中已被删除。当然，成功评估`constexpr`函数仍然需要所有计算步骤在编译时都是可行且有效的：当前，不包括堆分配或引发异常等事情，使用`constexpr`重构上面的函数：

```cpp
constexpr bool
doIsPrime(unsigned p, unsigned d)
{
	return d != 2 ? (p % d != 0) && doIsPrime(p, d - 1) : (p % 2 != 0);
}

constexpr bool isPrime(unsigned p)
{
	return p < 4 ? !(p < 2) : doIsPrime(p, p / 2);
}
```

由于只有一个语句的限制，我们只能使用条件运算符作为选择机制，我们仍然需要递归来迭代元素。但是该语法是普通C++函数代码，使它比依赖模板实例化的第一个版本更容易访问。

用C++ 14，`constexpr`函数可以利用一般C++代码中可用的大多数控制结构。因此，我们现在可以使用一个简单的for循环，而不是编写笨拙的模板代码或有点晦涩的单行代码

```cpp
constexpr bool isPrime(unsigned int p)
{
	for (unsigned int d = 2; d < p / 2; d++)
	{
		if (p % d == 0)
			return false;
	}
	return p > 1;
}
```

这里就会进行编译期求值，如果一定需要一个编译期值（比如为数组分配空间等），如果它不可能，就会发生错误；如果不需要一定要编译期给出值，如果编译器求值失败，则不报错，而是将调用保留为运行时调用

### 偏特化的执行路径选择

例如，根据模板参数是否为质数，我们可以在不同的偏特化之间进行选择

```cpp
template<int SZ, bool = isPrime(SZ)>
struct Helper;

template<int SZ>
struct Helper<SZ, false>
{
	
};

template<int SZ>
struct Helper<SZ, true>
{
	
};

template<typename T, std::size_t SZ>
long foo(std::array<T, SZ> const& coll)
{
	Helper<SZ> h;//implementation depends on whether array has prime number as size
}
```

这种偏特化的应用程序广泛适用于在函数模板的不同实现中进行选择，具体取决于调用它的参数的属性

在上面，我们使用了两个部分专门化来实现两个可能的替代方案。相反，我们还可以将主模板用于一个备选（默认）案例，而将部分专门化用于任何其他特殊案例（也就是兜底策略）：

```cpp
template<int SZ, bool = isPrime(SZ)>
struct Helper
{
  ...
};
```

由于函数模板不支持部分专门化，因此必须使用其他机制根据某些约束更改函数实现。我们的选择包括：

+ 使用带有静态函数的类
+ 使用`std::enable_if`
+ 使用*SFINAE*特性
+ 使用编译期的if（since c++17）

### SFINAE(Substitution Failure is Not An Error)

在C++中，重载函数以适应各种参数类型是非常常见的。当编译器看到对重载函数的调用时，它必须分别考虑每个候选函数，评估调用的参数并选择最匹配的候选函数

如果一个调用的候选集包含函数模板，编译器首先必须确定应该为该候选集使用哪些模板参数，然后替换函数参数列表及其返回类型中的这些参数，然后评估匹配程度（就像普通函数一样）。然而，替换过程可能会遇到一些问题：它可能产生毫无意义的构造。语言规则并没有认定这些无意义的替换会导致错误，相反，那些有替换问题的候选者会被忽略。

为此，我们引入SFINAE，表达：替换失败不是错误

注意，这里描述的替换过程不同于按需实例化过程：即使对于不需要的潜在实例化，也可以进行替换（这样编译器可以评估是否确实不需要它们）。它是对直接出现在函数声明中的结构（而不是其主体）的替换。

```cpp
template <typename T, unsigned N>
std::size_t len(T(&)[N])
{
	return N;
}

template <typename T>
typename T::size_t len(T const& t)
{
	return t.size();
}
```

我们定义了2个函数模板来执行一个泛型参数：

+ 第一个函数模板声明了一个参数`T(&)[N]`，意味着参数必须数组，具有N的元素，类型为T
+ 第一个函数模板声明了T，不对参数进行约束，但是要求返回值类型是`T::size_type`

传递原始数组或字符串文本时，只有原始数组的函数模板匹配：

```cpp
int a[10];
std::cout << len(a);
std::cout << len("tmp");
```

根据其签名，第二个函数模板在（分别）`int[10]`和`char const[4]`替换`T`时也匹配，但这些替换会导致返回类型`t::size_type`中潜在的错误。因此，这些调用将忽略第二个模板

传递`vector`时，就只有第二个函数模板可以匹配，传递原始指针时，没有哪个函数是匹配的，所以编译器会报告`len()`没有匹配的函数。请注意，这与传递具有`size_type`成员但没有`size()`成员函数的类型的对象不同，例如，对于`std::allocator<>`

```cpp
std::allocator<int> x;
std::cout << len(x);//Error, len() function found, but can't size()
```

当传递这种类型的对象时，编译器会将第二个函数模板作为匹配的函数模板。因此，如果没有找到匹配的`len()`函数，这将导致编译时错误，即为`std::allocator<int>`调用`size()`无效。这一次，第二个函数模板不会被忽略

在替换返回类型时忽略一个候选者是没有意义的，这会导致编译器选择另一个参数匹配性更差的候选者：

```cpp
template <typename T, unsigned N>//number of elements in a raw array
std::size_t len(T(&)[N])
{
	return N;
}

template <typename T>
typename T::size_t len(T const& t)//number of elements for a type having size_type
{
	return t.size();
}

std::size_t len(...)//fallback for all other type
{
	return 0;
}
```

因此，对于原始数组和向量，我们有两个匹配项，其中特定匹配项是更好的匹配项。对于指针，只有fallback匹配，这样编译器就不再报错。但是对于allocator，第二个和第三个函数模板匹配，第二个函数模板匹配得更好。因此，仍然会导致错误，无法调用`size()`成员函数

#### SFINAE和重载决议

随着时间的推移，sfinae原则变得如此重要，在模板设计人员中如此普遍，以至于缩写成了动词。如果我们打算应用sfinae机制，通过检测模板代码来确保忽略某些约束的函数模板，从而导致这些约束的代码无效，我们称之为“we sfinae out a function”。并且当您在C++标准中阅读时，函数模板“不应参与重载决议，除非…”这意味着SFANE用于某些情况下的“函数模板”

```cpp
namespace std
{
	class thread
	{
	public:
		...
		template<typename F, typename... Args>
		explicit thread(F&& f, Args&&... args);
		...
	};
}
//这个构造不会参与重载决议，如果decay_t<F>与std::thread类型相同
```

这意味着如果使用`std::thread`作为第一个也是唯一的参数调用模板构造函数，则会忽略它。原因是，否则，这样的成员模板有时可能比任何预定义的复制或移动构造函数更好地匹配。通过在调用线程时SFINAE'ing out模板构造器，我们确保在从另一个线程构造线程时始终使用预定义的复制或移动构造函数。

应用这种技术需要更改大量代码，可能会很难。幸运的是，标准库提供了更容易禁用模板的工具。最著名的此类功能是`std::enable_if<>`。它允许我们通过将类型替换为包含禁用条件的构造来禁用模板。

```cpp
namespace std
{
	class thread
	{
	public:
		...
		template<typename F, typename... Args,
		        typename = std::enable_if_t<!std::is_same_v<std::decay_t<F>, thread>>>
		explicit thread(F&& f, Args&&... args);
		...
	};
}
```

在某些情况下，要找出并制定合适的表达式来筛选出函数模板并不总是件容易的事。

例如，假设我们要确保对于具有`size_type`成员但不具有`size()`成员函数的类型的参数，忽略函数模板`len()`。如果在函数声明中对`size()`成员函数没有任何形式的要求，则会选择函数模板并最终实例化它，从而导致错误（比如上面的allocator）

处理这种情况有一个共同的模式：

+ 使用尾部返回类型语法指定返回类型（在前面使用auto，在结尾使用->在返回类型之前）。
+ 使用decltype和逗号运算符定义返回类型。
+ 制定逗号运算符开头必须有效的所有表达式（如果逗号运算符重载，则转换为void）。
+ 在逗号运算符末尾定义实际返回类型的对象。

```cpp
template<typename T>
auto len(T const &t)->decltype((void) (t.size()), T::size_type())
{
	return t.size();
}
```

`decltype`构造的操作数是用逗号分隔的表达式列表，因此最后一个表达式`t::size_type()`会生成所需返回类型的值（`decltype`使用该值转换为返回类型）。在（最后一个）逗号之前，我们有必须有效的表达式，在本例中，它只是`t.size()`。将表达式强制转换为`void`是为了避免为表达式类型重载用户定义的逗号运算符

这里，`decltype`是一个没有被计算的操作方式，这意味着我们可以创建一个“dummy obj”而不调用任何构造器。

### 编译期if

偏特化、SFINAE和`std::enable_if`允许我们整体启用或禁用模板。C++ 17还引入了一个编译时if语句，它允许根据编译时条件启用或禁用特定语句。对于语法`if constexpr(...)`，编译器使用编译时表达式来决定是应用then部分还是else部分（如果有的话）（从C++17起）

注意，`constexpr if`可以在任何函数中使用，而不仅仅是在模板中。我们只需要生成布尔值的编译时表达式。