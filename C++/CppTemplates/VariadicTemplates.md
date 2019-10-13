## Chapter4 Variadic Templates

从C++11开始，模板可以接受多参数的模板变量。也就代表着这个feature允许在你可以传递任意数量的任意类型的参数的地方使用模板。典型的应用是通过类或框架传递任意数量的任意类型的参数。另一种应用是提供通用代码来处理任何数量的任何类型的参数

### 可变模板

#### 可变模板样例

```cpp
void print() {}

template<typename T, typename... Types>
void print (T firstArg, Types... args)
{
	std::cout << firstArg << '\n';
	print(args...);//递归展开参数类型
}

int main()
{
	std::string s("World");
	print(7.5, "Hello", s);//print<double, char const*, std::string> (7.5, "Hello", s);
}

//Output:
7.5
Hello
World
```

####  重载可变/不可变模板：

```cpp
template <typename T>
void print(T arg)
{
	std::cout << arg << '\n';
}

template<typename T, typename... Types>
void print(T firstArg, Types... args)
{
	print(firstArg);
	print(args...);//call print() for remaining arguments
}
```

也就是说，如果两个功能模板的区别仅在于尾随参数包，则首选不具有尾随参数包的功能模板。

#### 操作符`sizeof...`

从C++11开始，支持新的操作符`sizeof...`，它拓展为参数包包含的元素数目，而且允许传递模板参数包和函数参数包（两种都可以）：

```cpp
template<typename T, typename... Types>
void print(T firstArg, Types... args)
{
	std::cout << sizeof...(Types) << '\n';//print number of remaining types
	std::cout << sizeof...(args) << '\n';//print number of remaining types
}

int main()
{
	std::string s("World");
	print(7.5, "Hello", s);
}

//Output:
2
2
```

由此我们看似可以写出不需要额外的递归终点的形式：

```cpp
template<typename T, typename... Types>
void print(T firstArg, Types... args)
{
	std::cout << firstArg << '\n';
	if (sizeof...(args) > 0) {//error if sizeof...(args) == 0
		print(args...);// and no print() for no arguments declared
	}
}
```

但是，这种方法行不通，因为通常情况下，函数模板中所有if语句的两个分支都将会被实例化。实例化的代码是否有用是运行时的决定，而调用的实例化是编译时的决定。因此，如果为一个（最后一个）参数调用print（）函数模板，则调用print（args…）的语句仍会实例化为无参数，并且如果没有函数print（）为无参数提供的，这是一个错误。

当然在C++ 17后支持了编译时if，语法是`if constexpr (sizeof...(args) > 0)`

### 表达式折叠

从C++ 17起，就有一个feature去使用二元操作符去计算对于一个参数包的所有参数（包含可选的初始化值）。也就是说，使用表达式折叠，你可以应用operator对一个参数包中的所有参数。

计算传入的所有参数的和：

```cpp
template <typename... T>
auto foldSum (T... s)
{
   return (... + s);//((s1 + s2) + s3) ...
}
```

注意，这里如果参数个数为0，表达式是错误的

根据与root的left和right关系找到二叉树中某个节点：

```cpp
struct Node
{
	int value;
	Node* left;
	Node* right;
	Node(int i = 0) : value(i) , left(nullptr), right(nullptr) {}
};

auto left = &Node::left;
auto right = &Node::right;

template <typename T, typename... TP>
Node* traverse(T np, TP... paths)//找到与np对应的位置的节点
{
	std::cout << np << '\n';
	return (np ->* ... ->* paths);//np ->* paths1 ->* paths2
}

int main()
{
	Node* root = new Node {0};
	root->left = new Node {100};
	root->right = new Node {200};
	root->right->left = new Node {3000};
	root->left->right = new Node {2000};

	Node* node = traverse(root, right, right);//返回是NULL
  Node* node1 = traverse(root, left, right);//返回节点为2000
}
```

类似的也有：

```cpp
template<typename... Types>
void print(Types const&... args)
{
  (std::cout << ... << args) << '\n';
}
```

但是，请注意，在这种情况下，没有空格将所有元素与参数包分开。为此，您需要一个附加的类模板，以确保将任何参数的任何输出都用空格扩展：请注意，表达式AddSpace（args）使用类模板参数推导（请参阅第40页的2.9节）来使AddSpace <Args>（args）的效果，它为每个参数创建一个引用传递的参数的AddSpace对象，并在输出表达式中使用它时添加一个空格：

```cpp
template<typename T>
class AddSpace
{
private:
	T const &ref;
public:
	AddSpace(T const &r) : ref(r) {}

	friend std::ostream& operator<<(std::ostream &os, AddSpace<T> s)
	{
		return os << s.ref << ' ';
	}
};

template<typename... Types>
void print(Types ... args)
{
	(std::cout << ... << AddSpace(args)) << '\n';
}
```

### 可变模板应用

将参数传递给共享指针拥有的新堆对象的构造函数：

```cpp
auto sp = std::make_shared<std::complex<float>>(4.2, 7.7);
```

传递参数给一个从库中启动的线程：

```cpp
std::thread t(foo, 42, "hello");//call foo(42, "hello") in a separate thread
```

传递参数给一个构造器来push进入vector，这里通常使用移动语义来进行完美转发（最为经典的一种应用：转发任意数量的任意类型的参数）

```cpp
std::vector<Customer> v;
...
v.emplace("Tim", "Jovi", 1962);
```

还请注意，与普通参数一样，规则适用于可变参数模板参数。例如，如果通过值传递，则参数将被复制并衰减（例如，数组成为指针），而如果通过引用传递，则参数将引用原始参数，并且不会衰减

```cpp
//args are copies with decayed types:
template<typename... Args> void foo(Args... args);
//args are nondecayed references to passed objects:
template<typename... Args> void bar(Args const&... args);
```

### 可变类模板和可变表达式

#### 可变表达式

不仅可以转发所有参数，还可以做更多的事情。可以使用它们进行计算，这意味着可以使用参数包中的所有参数进行计算。
例如，以下函数将参数包args的每个参数加倍，并将每个加倍的参数传递给print（）：

```cpp
template<typename... T>
void printDoubled(T const& args)
{
	print(args + args...);
}

printDoubled(7.5, std::string("hello"), std::complex<float>(4,2));

//Output:
15 hellohello (8,4) 
```

如果做plus1，注意`...`的位置

```cpp
template<typename... T>
void addOne(T const&... args)
{
	print(args + 1...);//Error, 1... is the leteral with too many decimal point
	print(args + 1 ...);//OK
	print((args + 1)...);//OK
}
```

编译时表达式可以以相同的方式包括模板参数包。例如，以下函数模板返回所有参数的类型是否相同：

```cpp
template <typename T1, typename... TN>
constexpr bool isHomoGeneous(T1, TN...)
{
	return (std::is_same_v<T1, TN> && ...);//since c++17
}
```

#### 可变索引

以下函数使用索引的可变列表来访问传递的第一个参数的相应元素

```cpp
template<typename C, typename... Idx>
void printElems(C const& coll, Idx... idx)
{
  print(coll[idx]...);
}

std::vector<std::string> coll = {"good", "times", "say", "bye"};
printElems(coll, 2, 0, 3);
//Equal to
print(coll[2], coll[0], coll[3]);

//也可以有以下相同功能代码，通过声明非类型模板参数
template<std::size_t... Idx, typename C>
void printIdx(C const& coll)
{
  print(coll[Idx]...);
}
printIdx<2, 0, 3>(coll);
```

#### 可变类模板

可变参数模板也可以是类模板。比如一个类，其中任意数量的模板参数指定了相应成员的类型。

你能够定义一个类作为一个类型来代表一系列的索引：

```cpp
template<std::size_t...>
struct Indices
{};
```

这可以用来定义一个函数，该函数使用带有给定索引的`get<>()`的编译时访问为`std::array`或`std::tuple`的元素调用`print()`：

```cpp
template<typename T, std::size_t... Idx>
void printByIdx(T t, Indices<Idx...>)
{
	print(std::get<Idx>(t)...);
}

std::array<std::string, 5> arr = {"Hello", "my", "new", "!", "World"};
printByIdx(arr, Indices<0, 4, 3>());
auto t = std::make_tuple(12, "monkeys", 2.0);
printByIdx(t, Indices<0, 1, 2>());

//Output:
Hello World !
12 monkeys 2
```

#### 可变推断指导

甚至推断指导（deduction guides）也是可变的。C++标准库给出了对`std::array`以下推断指导：

```cpp
namespace std
{
	template<typename T, typename... U> array(T, U...)
		->array<enable_if_t<(is_same_v<T, U> && ...), T>,
		        (1 + sizeof...(U))>;
}

std::array a{42, 45, 77};
```

在指导中将T推导出为元素的类型，并将各种`U…`类型推导为后续元素的类型。因此，元素总数为`1 + sizeof...(U)`

```cpp
std::array<int, 3> a{42, 45, 77};
```

第一个的`std::enable_if<>`表达式对第一个array参数作为一个折叠参数，拓展实质为：

```cpp
is_same_v<T, U1> && is_same_v<T, U2> && ...
```

如果结果是非`true`的，则推断终止，整个推断失败。

#### 可变基类及其使用

我们定义了类`Customer`和独立的函数对象去hash和比较`Customer`对象：

```cpp
class Customer
{
private:
	std::string name;
public:
	Customer(std::string const &n) : name(n)
	{}

	std::string getName() const
	{ return name; }
};

struct CustomerEq
{
	bool operator()(Customer const &c1, Customer const &c2) const
	{
		return c1.getName() == c2.getName();
	}
};

struct CustomerHash
{
	std::size_t operator()(Customer const &c) const
	{
		return std::hash<std::string>()(c.getName());
	}
};

//我们可以定义一个从各种基类派生的类，这些基类从每个基类中引入operator()声明
// define class that combines operator() for variadic base classes:
template<typename... Bases>
struct Overloader : Bases ...
{
	using Bases::operator()...;// OK since C++17
};

int main()
{
	// combine hasher and equality for customers in one type:
	using CustomerOP = Overloader<CustomerHash, CustomerEq>;

	std::unordered_set<Customer, CustomerHash, CustomerEq> coll1;
	std::unordered_set<Customer, CustomerOP, CustomerOP> coll2;
}
```

