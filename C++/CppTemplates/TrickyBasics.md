## Chapter5 Tricky Basics

### 关键字：`typename`

```cpp
template<typename T>
class MyClass
{
public:
	void foo()
	{
		typename T::SubType* ptr;
	}
};
```

第二个`typename`被用来声明`SubType`作为一个定义在类T中的类型，ptr是一个指向`T::SubType`的指针

要是没有这个`typename`，`SubType`将会假定为非类型成员（如静态成员或者枚举常量）也就是说：

```cpp
T::SubType* ptr
```

会成为`T::SubType`这个静态成员与`ptr`对象进行乘法运算，编译器在这里将会编译出与我们愿望完全不同的代码

**通常情况下，只要依赖参数的名称是类型，就必须使用`typename`**

一种应用是在泛型的迭代器上，参数就是类型T的标准容器，使用`const_iterator`对容器中的所有元素进行遍历：

```cpp
template<typename T>
void printcoll(T const &coll)
{
	typename T::const_iterator pos;
	typename T::const_iterator end(coll.end());
	for (pos = coll.begin(); pos != end; pos++)
	{
		std::cout << *pos << ' ';
	}
	std::cout << '\n';
}
```

### 零初始化

就像系统中的int等，这些都是没有初始化的，而是使用一个任意缺省值填充的。现在要是我们编写模板并且模板类型的值使用默认值进行初始化，我们就会遇到一个问题：一个简单的定义无法对内置类型执行此操作的问题。

```cpp
template<typename T>
void foo()
{
  T x;//x has undefined value if T is built-in type
}
```

出于这个原因，可以为内置类型显式调用默认构造函数，以将其初始化为零（对于bool则为false，对于指针为nullptr）。因此，通过编写以下代码，即使对于内置类型，也可以确保正确的初始化：

```cpp
template<typename T>
void foo()
{
  T x();//x is zero (or false) if T is a build-in type
}
```

这种初始化称为值初始化。一位置要么调用提供的构造函数，要么把对象初始化为0.如果构造函数是显式的，这是可以运行的。

在C++11之前，我们可以用这样的语法保证初始化顺利：

```cpp
T x = T();
```

为了确保一个类模板的成员（它的类型已经被参数化）被初始化，可以定义一个默认构造函数，该构造函数使用初始化器来初始化该成员：

为确保类型模板已被参数化的类模板的成员被初始化，可以定义一个默认构造函数，该构造函数使用初始化器来初始化该成员：

```cpp
//C++11之前
template<typename T>
class MyClass
{
private:
	T x;
public:
	MyClass() : x() {}
};

//C++11之后
//允许提供一个缺省初始化给非静态成员
template<typename T>
class MyClass
{
private:
	T x{};//zero-initialize x unless otherwise specified
};
```

但是我们不能对默认参数使用这种语法：

```cpp
template<typename T>
void foo(T p{})//Error
{
  ...
}
```

正确做法：

```cpp
template<typename T>
void foo(T p = T{})//OK (must use T() before C++ 11)
{
  ...
}
```

### 使用`this->`

对于具有依赖于模板参数的基类的类模板，即使成员`x`是继承的，使用名称x本身也不总是等同于`this-> x`，我们应该尽量在需要时始终显式调用`Base<T>::`或`this->`：

```cpp
template<typename T>
class Base
{
public:
	void bar();
};

template<typename T>
class Derived : Base<T>
{
public:
	void foo()
	{
		bar();//calls external bar() or error
    //this->bar();//OK
    //Base<T>::bar();//OK too
	}
};
```

### 对原始数组与字符串文本的模板

**这里要注意值类型的decay（衰竭）问题。**首先，如果将模板参数声明为引用，则参数不会衰减。也就是说，传递的参数“hello”的类型为`char const [6]`。如果由于类型不同而传递了不同长度的原始数组或字符串参数，这可能会成为问题。仅当按值传递参数时，类型才会衰减，因此字符串文字将转换为`char const *`类型。

```cpp
template<typename T, int N, int M>
bool less(T(&a)[N], T(&b)[M])
{
	for(int i = 0; i < N && i < M; ++i)
	{
		if(a[i] < b[i]) return true;
		if(b[i] < a[i]) return false;
	}
	return N < M;
}

int main()
{
	int x[] = {1,2,3};
	int y[] = {1,2,3,4,5};
	std::cout << less(x,y) << '\n';//T -> int; N -> 3; M -> 5;
  std::cout << less("ab", "abc") << '\n';//T -> char const; N -> 3; M -> 4;
}
```

请注意，有时可能需要重载或部分专门处理未知范围的数组。以下程序说明了数组的所有可能的重载：

```cpp
template<typename T>
struct MyClass;

template<typename T, std::size_t SZ>
struct MyClass<T[SZ]>
{
	static void print()
	{
		std::cout << "print() for T[" << SZ << "]\n";
	}
};

template<typename T, std::size_t SZ>
struct MyClass<T(&)[SZ]>
{
	static void print()
	{
		std::cout << "print() for T(&)[" << SZ << "]\n";
	}
};

template<typename T>
struct MyClass<T[]>
{
	static void print()
	{
		std::cout << "print() for T[]\n";
	}
};

template<typename T>
struct MyClass<T(&)[]>
{
	static void print()
	{
		std::cout << "print() for T(&)[]\n";
	}
};

template<typename T>
struct MyClass<T *>
{
	static void print()
	{
		std::cout << "print() for T*\n";
	}
};


template<typename T1, typename T2, typename T3>
void foo(int a1[7], int a2[], int (&a3)[42], int (&x0)[], T1 x1, T2 &x2, T3 &&x3)
{
  //值类型出现衰竭
	MyClass<decltype(a1)>::print();//MyClass<T*>
	MyClass<decltype(a2)>::print();//MyClass<T*>
	MyClass<decltype(a3)>::print();//MyClass<T(&)[SZ]>
  
  //引用不衰竭，无论左值/右值引用
	MyClass<decltype(x0)>::print();//MyClass<T(&)[]>
	MyClass<decltype(x1)>::print();//MyClass<T*>这里有值衰竭，从数组衰竭成指针
	MyClass<decltype(x2)>::print();//MyClass<T(&)[]>
	MyClass<decltype(x3)>::print();//MyClass<T(&)[]>
}

int main()
{
	int a[42];
	MyClass<decltype(a)>::print();//MyClass<T[SZ]>

	extern int x[];
	MyClass<decltype(x)>::print();//MyClass<T[]>

	foo(a, a, a, x, x, x, x);
}

int x[] = {0, 8, 15};

//Output:
print() for T[42]
print() for T[]
print() for T*
print() for T*
print() for T(&)[42]
print() for T(&)[]
print() for T*
print() for T(&)[]
print() for T(&)[]
```

### 成员模板

类成员也可以是模板。嵌套类和船员函数都可以这样。例子如下：

```cpp
Stack<int> intStack1, intStack2;
Stack<float> floatStack;

intStack1 = intStack2;//OK
floatStack = intStack1;//Error
```

默认运算符要求两端具有相同的类型，但是如果堆栈具有不同的类型，显然就报错了。但是可以将赋值运算符定义为模板，启动具有合适类型转换的元素的堆栈赋值：

```cpp
template<typename T>
class Stack
{
private:
	std::deque<T> elems;
public:
	void push(T const&);
	void pop();
	T const& top() const;
	bool empty() const
	{
		return elems.empty();
	}
  
  //assign stack of elements of type T2
	template <typename T2>
	Stack& operator=(Stack<T2> const&);
  
  //to get access to private members of Stack<T2> for any type T2
  //因为不需要使用模板参数，所以可以省略
  template<typename> friend class Stack;
};

template<typename T>
template<typename T2>
Stack<T>& Stack<T>::operator=(Stack<T2> const& op2)
{
	Stack<T2> tmp(op2);
	elems.clear();

	while(!tmp.empty())
	{
		elems.push_front(tmp.top());
		tmp.pop();
	}
	return *this;
}

//通过友元类，可以实现以下改进代码，不然从op2得不到其中的elems
template<typename T>
template<typename T2>
Stack<T>& Stack<T>::operator=(Stack<T2> const& op2)
{
	elems.clear();
	elems.insert(elems.begin(), op2.elems.begin(), op2.elems.end());
	return *this;
}
```

首先，我们来看一下定义成员模板的语法。在模板参数为T的模板内部，定义了模板参数为T2的内部模板

在成员函数内部，您可能只是希望访问已分配堆栈op2的所有必要数据。但是，此堆栈具有不同的类型（如果为两种不同的参数类型实例化一个类模板，则会得到两种不同的类类型），因此只能使用公共接口。因此，访问元素的唯一方法是调用`top()`。但是，每个元素都必须成为顶部元素。因此，必须首先制作op2的副本，以便通过调用`pop()`从该副本中获取元素。因为`top()`返回被压入堆栈的最后一个元素，所以我们可能更喜欢使用一个容器来支持在集合的另一端插入元素。因此，我们使用`std::deque<>`，它提供`push_front()`将元素放置在集合的另一侧，为了得到所有的关于op2的成员，需要定义一个友元实例

现在，我们想要点更刺激的：

```cpp
Stack<std::string> stringStack;
Stack<float> floatStack;
floatStack = stringStack;//Error: std::string doesn't convert to float
```

这里提供2个参数的模板Stack

```cpp
template<typename T, typename Cont = std::deque<T>>
class Stack
{
private:
	Cont elems;
public:
	void push(T const&);
	void pop();
	T const& top() const;
	bool empty() const
	{
		return elems.empty();
	}

	template<typename T2, typename Cont2>
	Stack& operator= (Stack<T2, Cont2> const&);

	template <typename, typename> friend class Stack;
};

template<typename T, typename Cont>
template<typename T2, typename Cont2>
Stack<T, Cont>&
Stack<T, Cont>::operator= (Stack<T2, Cont2> const &op2)
{
	elems.clear();
	elems.insert(elems.begin(), op2.elems.begin(), op2.elems.end());
	return *this;
}
```

#### 特化成员函数模板

成员功能模板也可以是部分或完全专用的。例如，对于以下类：

```cpp
class BoolString
{
private:
	std::string value;
public:
	BoolString(std::string const &s) : value(s) {}
	template<typename T = std::string>
	T get() const
	{
		return value;
	}
};

//偏特化其中的东西：
template<>
inline bool BoolString::get<bool>() const
{
  return value == "true" || value == "1" || value == "on";
}
```

注意，不能声明偏特化，而只定义它们。因为它是完全特例化的，并且在头文件中，所以如果该定义包含在不同的翻译单元中，则必须用内联声明它以避免错误。

#### `.template`的建设

有时，在调用成员模板时，必须明确限定模板参数。在这种情况下，必须使用`template`关键字来确保`<`是模板参数列表的开头。考虑以下使用标准`bitset`类型的示例：

```cpp
template<unsigned long N>
void printBitset (std::bitset<N> const& bs)
{
	std::cout << bs.template to_string<char, std::char_traits<char>,
	        std::allocator<char>>();
}
```

对于bs，我们把成员函数模板`to_string()`来显式指定字符串类型的详细信息。这里，如果我们没有使用`.template`，编译器将会把`<`判定为小于而不是模板列表开头。注意，只有当在句点之前的构造取决于模板参数时才引发这个问题。示例中，参数bs取决于模板参数N。

`.template`（以及类似的符号，例如`->template`和`::template`）应仅在模板内部使用，并且仅当它们遵循依赖于模板参数的内容时使用。

#### 泛型Lambda与成员模板

```cpp
//最简单的Sum:
[] (auto x, auto y)
{
  return x + y;
}

//应用：
class SomeCompilerSpecificName
{
public:
  SomeCompilerSpecificName();
  template<typename T1, typename T2>
  auto operator() (T1 x, T2 y) const
  {
    return x + y;
  }
}
```

### 变量模板

从C++14起，变量也可以通过特定类型进行参数化。这被称为变量模板。要使用变量模板，必须指定其类型。

```cpp
//可能不会在函数内和块作用域中发生
template<typename T>
constexpr T pi{3.1415};
std::cout << pi<double> << '\n';
std::cout << pi<float> << '\n';
```

这里再次，请注意，即使在不同的翻译单元中对arr进行初始化和迭代时，仍会使用全局范围的相同变量`std::array<int，10> arr`

```cpp
template<int N>
std::array<int, N> arr{};
template<auto N>
constexpr decltype(N) dval = N;

int main()
{
	std::cout << dval<'c'> << '\n';
	arr<10>[0] = 42;
	for(std::size_t i = 0; i < arr<5>.size(); ++i)
		std::cout << arr<10>[i] << '\n';
}
```

#### 数据成员的变量模板

```cpp
template<typename T>
class MyClass
{
public:
	static constexpr int max = 1000;
};

template<typename T>
int myMax = MyClass<T>::max;
auto i = myMax<std::string>;//i -> int
```

```cpp
namespace std
{
	template<typename T> class numeric_limits
	{
	public:
		static constexpr bool is_signed = false;
	};
}

template <typename T>
constexpr bool isSigned = std::numeric_limits<T>::is_signed;

//从此以后
isSigned<char>
//可以取代
std::numeric_limits<char>::is_signed
```

#### 类型特征后缀`_v`

```cpp
std::is_const_v<T> //since c++17
std::is_const<T>::value//since c++11
```

### 模板模板参数

允许一个模板参数本本身成为一个类模板。

```cpp
Stack<int, std::vector<int>> vStack;
//太丑了，能不能简洁化成：
Stack<int, std::vector> vStack;
```

为此，我们必须将第二个模板参数作为模板模板参数

```cpp
template<typename T,
		template<typename Elem> typename Cont = std::deque>//Error before C++17
		//template<typename Elem> class Cont = std::deque>//OK
class Stack
{...};
```

但是这样写有问题：运行会报错，大意是缺省值`std::deque`不匹配模板模板参数`Cont`。这个问题主要原因是在C++17前，模板模板参数必须是模板，这个模板的**参数必须**和其替换的模板模板参数的参数**完全匹配**（可变参数模板等例外）。有关模板模板参数的缺省模板参数压根没有被编译器考虑，以至于匹配无法完成。（在C ++ 17中，将考虑默认参数）

也就是说这里有问题：标准库里面的`std::deque`可以接受多余一个的模板参数，这就导致了这里直接出现了不匹配！我们要真正完全匹配它的模板参数类型才行：

```cpp
template<typename T,
		template<typename Elem,
		typename = std::allocator<Elem>>
		class Cont = std::deque>
class Stack
{
private:
	Cont<T> elems;
public:
	void push(T const &);

	void pop();

	T const &top() const;

	bool empty() const
	{
		return elems.empty();
	}

	template<typename T2,
			template<typename Elem2,
			typename = std::allocator<Elem2>
	> class Cont2>
	Stack<T, Cont> &operator=(Stack<T2, Cont2> const &);

	template<typename, template<typename, typename> class> friend
	class Stack;
};

template<typename T, template<typename, typename> class Cont>
void Stack<T, Cont>::push(const T &elem)
{
	elems.push_back(elem);
}

template<typename T, template<typename, typename> class Cont>
void Stack<T, Cont>::pop()
{
	assert(!elems.empty());
	elems.pop_back();
}

template<typename T, template<typename, typename> class Cont>
T const &Stack<T, Cont>::top() const
{
	assert(!elems.empty());
	return elems.back();
}

template<typename T, template<typename, typename> class Cont>
template<typename T2, template<typename, typename> class Cont2>
Stack<T, Cont> &Stack<T, Cont>::operator=(Stack<T2, Cont2> const &op2)
{
	elems.clear();
	elems.insert(elems.begin(), op2.elems.begin(), op2.elems.end());
	return *this;
}

int main()
{
	Stack<int> iStack;
	Stack<float> fStack;

	iStack.push(1);
	iStack.push(2);
	std::cout << "iStack top:" << iStack.top() << '\n';

	fStack.push(3.3);
	std::cout << "fStack top:" << fStack.top() << '\n';

	fStack = iStack;
	fStack.push(4.4);
	std::cout << "fStack top:" << fStack.top() << '\n';

	Stack<double, std::vector> vStack;
	vStack.push(5.5);
	vStack.push(6.6);
	std::cout << "vStack top:" << vStack.top() << '\n';

	vStack = fStack;
	std::cout << "vStack:";
	while (!vStack.empty())
	{
		std::cout << vStack.top() << ' ';
		vStack.pop();
	}

	std::cout << '\n';
}

//Output:
iStack.top:2
fStack.top:3.3
fStack.top:4.4
vStack.top:6.6
vStack:4.4 2 1
```