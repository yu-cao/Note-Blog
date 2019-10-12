## Chapter3 Nontype Template Parameters

### Nontype Class Template Parmeters

```cpp
#include <array>
#include <cassert>

template<typename T, std::size_t Maxsize>
class Stack
{
private:
	std::array<T, Maxsize> elems;
	std::size_t numElems;
public:
	Stack();

	void push(T const &elem);

	void pop();

	T const &top() const;

	bool empty() const
	{
		return numElems == 0;
	}

	std::size_t size() const
	{
		return numElems;
	}
};

template<typename T, std::size_t Maxsize>
Stack<T, Maxsize>::Stack() : numElems(0)
{}

template<typename T, std::size_t Maxsize>
void Stack<T, Maxsize>::push(const T &elem)
{
	assert(numElems < Maxsize);
	elems[numElems] = elem;
	++numElems;
}

template<typename T, std::size_t Maxsize>
void Stack<T, Maxsize>::pop()
{
	assert(!elems.empty());
	--numElems;
}

template<typename T, std::size_t Maxsize>
T const &Stack<T, Maxsize>::top() const
{
	assert(!elems.empty());
	return elems[numElems - 1];
}

int main()
{
	Stack<int, 20> int20Stack;
	Stack<int, 40> int40Stack;
	Stack<std::string, 40> stringStack;

	// manipulate stack of up to 20 ints
	int20Stack.push(7);
	std::cout << int20Stack.top() << '\n';
	int20Stack.pop();

	// manipulate stack of up to 40 strings
	stringStack.push("hello");
	std::cout << stringStack.top() << '\n';
	stringStack.pop();
}
```

上面的Maxsize，类型为`int`，它特例化了Stack内部array的具体大小

### Nontype Function Template Parameters

```cpp
template<int Val, typename T>
T addValue(T x)
{
  return x + Val;
}

//具体使用举例：
std::transform(source.begin(), source.end(),//start and end of source
              dest.begin(),//start of destination
              addValue<5, int>);//operation

//使用推导来得到传入的模板的类型
template<auto Val, typename T = decltype(Val)>
T foo();
//或来确保类型的相同：
template<typename T, T = Val = T{}>
T bar();
```

### Restrictions for Nontype Template Parameters

非类型模板参数只支持const integral value（包括枚举）、指向对象/函数/成员的指针，对象/函数的左值引用，`std::nullptr_t`；**不支持浮点数，类对象，也不支持直接的字符串文本**（但是可以绕过）：

```cpp
template<double VAT>//ERROR
double process(double v)
{
  return v * VAT;
}

template<std::string name>//ERROR
class MyClass{
  ...
};

//避免直接的string literal策略
//C++ 11允许外部链接
//C++ 17额外允许了内部链接
extern char const s0[] = "hi";
char const s1[] = "hi";

int main()
{
  Message<s0> m0;//OK
  Message<s1> m1;//OK sine C++11
  
  static char const s2[] = "hi";
  Message<s2> m2;//OK sine C++17
}
```

### Template Parameter Type `auto`

从C++17开始，编译器支持了任意的非类型参数来允许所有编译器允许的参数并且自动进行推导

```cpp
#include <array>
#include <cassert>

template <typename T, auto MaxSize>
class Stack
{
public:
	using size_type = decltype(MaxSize);
private:
	std::array<T, MaxSize> elems;
	size_type numElems;
public:
	Stack();
	void push(T const& elem);
	void pop();
	T const& top() const;
	bool empty() const { return numElems == 0;}
	size_type size() const { return numElems;}//C++14后可以auto表达返回值类型
};

template<typename T, auto MaxSize>
Stack<T, MaxSize>::Stack() : numElems(0) {}

template<typename T, auto MaxSize>
void Stack<T, MaxSize>::push(const T &elem)
{
	assert(numElems < MaxSize);
	elems[numElems] = elem;
	++numElems;
}

template<typename T, auto MaxSize>
void Stack<T, MaxSize>::pop()
{
	assert(!elems.empty());
	--numElems;
}

template<typename T, auto MaxSize>
T const &Stack<T, MaxSize>::top() const
{
	assert(!elems.empty());
	return elems[numElems-1];
}

int main()
{
	Stack<int, 20u> int20Stack;
	Stack<std::string, 40> stringStack;

	int20Stack.push(7);
	std::cout << int20Stack.top() << '\n';
	auto size1 = int20Stack.size();

	stringStack.push("Hello");
	std::cout << stringStack.top() << '\n';
	auto size2 = stringStack.size();

  //因为我们存在不同的返回类型，所以size1和size2的类型是不同的
	if(!std::is_same_v<decltype(size1), decltype(size2)>)//C++17 only
  //if(!std::is_same<decltype(size1), decltype(size2)>::value)//非C++17
	{
		std::cout << "size types differ" << '\n';
	}
}
```

```cpp
//since C++17
template <auto T>
class Message
{
public:
	void print()
	{
		std::cout << T << '\n';
	}
};

int main()
{
	Message<42> msg1;
	msg1.print();

	static char const s[] = "hello";
	Message<s> msg2;
	msg2.print();
}

//Output:
42
hello
```

甚至现在可以`template<decltype(auto) N>`，允许实例化N作为一个引用

```cpp
template<decltype(auto) N>
class C {
  ...
};
int i;
C<(i)> x;//N is int&

```

