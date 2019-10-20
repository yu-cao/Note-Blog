## Chapter7 By Value or by Reference?

### 按值传递

当我们传递一个左值的时候，这里的开销总是很大的，我们需要一个完整的拷贝。尤其比如字符串的拷贝，拷贝将会被创建通过拷贝构造，这里通常会有一个全拷贝或者深拷贝以申请他们自己的内存。

### 按值衰减传递

比如原始数组会衰减成为指针，`const`、`volatile`等限定符会被删除，函数衰减成为函数指针等

### 按引用传递

优点：不会有copy的开销，不会有衰减。但是有些时候传递是不可行的，也有时传递可行但是会导致错误：

#### 传递只读引用

```cpp
template<typename T>
void printR(T const& arg)
{}
```

在后台，通过引用传参实现是通过传递参数地址完成的。地址是紧凑编码的，所以把地址从调用方传递掉被调用方是高效的。但是传递地址会对编译器创造出一些不确定性：被调用者是不是正在用这些地址呢？理论上来说，被调用者可以更改这些地址上的值，意味着编译器必须假定所有已经有了缓存的值都在调用后不再有效。reload这些开销非常大。尽管我们现在用常量引用，但是，编译器并不认可这个我们这个担保行为，因为调用者可以修改引用通过它自己的非const引用。

通过`inline`可以缓解这个问题，也就是通过一些简单的内联，也就可以推断出我们的担保是可靠的，但是如果我们的封装是复杂的，内联可能性就很低了。

#### 传递不会衰竭的引用

通过引用将参数传递给参数时，它们不会衰减。但是，由于将调用参数声明为`T const＆`，因此模板参数`T`本身不推导为`const`。

#### 传递非只读引用

如果我们不希望这里传递进入const参数（因为这种形式通常是为了修改传入的参数，再通过引用保持出去），通过`static_assert(!std::is_const_v<T>, "out para of foo<T>(T&) is const");`或者`enable_if`处理

```cpp
template<typename T,
        typename = std::enable_if_t<!std::is_const_v<T>> >//拒绝掉const参数
void outR(T& arg)
{
	
}
```

#### 传递通过转发引用

使用按引用调用的一个原因是能够完善转发参数。但是使用转发引用（定义为模板参数的右值引用）时，将应用特殊规则。

```cpp
template<typename T>
void passR(T&& arg)
{}
```

你能传递任何东西给转发引用，同时就像其他一样，不会有复制的开销，而且甚至会正确推断出：右值/const左值/non-const左值

但是天下没有免费的午餐。例如，模板参数T隐式成为引用类型的情况。结果，使用T声明本地对象而不进行初始化可能会出错：

```cpp
template<typename T>
void passR(T&& arg)
{
  T x;//传递过来的arg是一个引用，这里就直接挂了
}
```

### 使用`std::ref()`与`std::cref()`

自`C++11`起，您可以让调用者为函数模板参数决定是按值还是按引用传递它。当声明一个模板按值接受参数时，调用者可以使用在头文件`<functional>`中声明的`std::cref()`和`std::ref()`来通过引用传递参数。

```cpp
template<typename T>
void printT(T arg)
{}

int main()
{
	const std::string s = "";
	printT(s);
	printT(std::cref(s));
}
```

但是，请注意`std::cref()`不会更改模板中参数的处理。取而代之的是，它使用了一个技巧：将传递的参数s包装成一个类似于引用的对象。实际上，它创建了一个关于原参数的`std::reference_wrapper<>`类型的对象，并按值传递了该对象。包装器或多或少仅支持一种操作：将隐式类型转换回原始类型，从而产生原始对象。因此，只要对传递的对象具有有效的运算符，就可以使用引用包装器

```cpp
void printString(std::string const& s)
{
	std::cout << s << '\n';
}

template<typename T>
void printT(T arg)
{
	printString(arg);
}

int main()
{
	const std::string s = "";
	printT(s);
	printT(std::cref(s));
}
```

最后一次调用按值将`std::reference_wrapper<string const>`类型的对象传递给参数`arg`，然后传递该参数，并将其转换回其基础类型`std::string`

请注意，编译器必须知道必须进行隐式转换回原始类型。因此，`std::ref()`和`std::cref()`通常仅在通过通用代码传递对象时才能正常工作。例如，直接尝试输出通用类型T的传递对象将失败，因为没有为`std::reference_wrapper<>`定义输出操作符：

```cpp
template <typename T>
void printV(T arg)
{
	std::cout << arg << '\n';
}

int main()
{
	const std::string s = "hello";
	printV(s);
	printV(std::cref(s));//error: invalid operands to binary expression ('std::__1::ostream' (aka 'basic_ostream<char>') and 'std::__1::reference_wrapper<const std::__1::basic_string<char> >')
}
```

这样的操作也是不行（不能通过比较`char const*`或者`std::string`来比较一个引用包装器）：

```cpp
template <typename T1, typename T2>
bool isless(T1 arg1, T2 arg2)
{
	return arg1 < arg2;
}

int main()
{
	const std::string s = "hello";
	if(isless(std::cref(s) < "world"))//error
	{}
	if(isless(std::cref(s) < std::string("world")))//error
	{}
}
```

因此，类`std::reference_wrapper<>`的作用是能够将引用用作成为“第一类对象”，您可以复制该引用并将其按值传递给功能模板。也可以在类中使用它，例如，保存对容器中对象的引用。但是，最终总是需要将其转换回基础类型

### 处理字符串文本与原始指针

我们已经看到了，如果模板参数是字符串文本与原始指针的时候，效果会有不同：

+ 传值衰减
+ 引用不衰减

这样有好的一面，也有不好的一面：当数组衰减成为指针时，将失去区分处理指向元素的指针和处理传递的数组的能力。另一方面，在处理可能传递字符串文字的参数时，不衰减可能会成为问题，因为不同大小的字符串文字具有不同的类型。

```cpp
template<typename T>
void foo(const T& arg1, const T& arg2)
{}

foo("hi","guy");//error,char const[3]与char const[4]不匹配到同一个T上


//如果改成了传值来衰减
template<typename T>
void foo1(T arg1, T arg2)
{
  if(arg1 == arg2)//这里可能会引发runtime-error
  {
    ...
  }
}

foo("hi", "guy");
```

你要将传递的字符串指针解释为字符串。但是无论如何，因为模板要处理的字符串文本参数已经发生衰竭。

不过，在许多情况下，衰减是有帮助的，特别是对于检查两个对象（都作为参数传递或一个作为参数传递而另一个期望作为参数）是否具有或转换为同一类型。一种典型用法是完美转发。但是，如果要使用完美转发，则必须将参数声明为转发引用。在这些情况下，可以使用type trait std::decay <>()`显式地衰减参数。

请注意，其他类型特征有时也会隐式衰减，例如`std::common_type<>`，它会产生两个传递的参数类型的通用类型


#### 对于字符串文本与原始指针的特别实现

你也许必须清楚地知道你的实现根据是否传入一个指针还是数组。（当然这是建立在数组没有衰竭的情况下）区分是否传递了数组：

+ 可以声明只对于数组有效的模板参数

```cpp
template<typename T, std::size_t L1, std::size_t L2>
void foo (T (&arg1)[L1], T (&arg2)[L2])
{
  T* pa = arg1;
  T* pb = arg2;
  if (compareArray(pa, L1, pb, L2))
  {...}
}
```

+ 使用类型特征来检测是否传递了数组

```cpp
template<typename T, typename = std::enable_if_t<std::is_array<T>>>
void foo (T&& arg1, T&& arg2)
{}
```

对于这些特殊处理，通常以不同方式处理数组的最佳方法就是简单地使用不同的函数名。当然，更好的办法是确保模板的调用者使用`std::vector`或`std::array`。但是，只要字符串文字是原始数组，我们就始终必须考虑它们

### 处理返回值

对于返回值，我们可以决定是按值返回还是按引用返回。但是，返回引用可能会引起麻烦，因为引用的内容超出了您的控制范围。在某些情况下，返回引用是常见的编程习惯：

+ 返回容器中的某个元素（比如`operator[]`或`front()`等）
+ 向类成员授予写的权限
+ 返回用于链式调用的对象（一般来说，流的`operator<<`和`operator>>`和类对象的`operator=`）

此外，通常通过返回const引用来授予成员读取权限。注意以下的cases，当使用不当的时候，也许会导致问题：

```cpp
std::string *s = new std::string("whatever");
auto& c = (*s)[0];
delete s;
std::cout << c;//未定义行为
```

但是在复杂情况下，很多时候就不是那么明显了：

```cpp
auto s = std::make_shared<std::string>("whatever");
auto& c = (*s)[0];
s.reset();
std::cout << c;//也是UB
```

因此，我们应确保功能模板按值返回其结果。但是，使用模板参数T不能保证它不是引用，因为有时可能会隐式推导T作为引用：

```cpp
template<typename T>
T retR (T&& p)//p是一个转发引用
{
  return T{...};
}
```

即使T是从值调用中推导出的模板参数，当显式将模板参数指定为引用时，它也可能成为引用类型：

```cpp
template<typename T>
T retV (T p)
{
  return T{...};
}

int x;
retV<int&>(x);//retT()的实例化T为int&
```

我们处理这个问题，有以下策略

+ 使用类型特征`std::remove_reference<>`将类型T变成非引用类型

```cpp
template<typename T>
typename std::remove_reference<T>::type retV(T p)
{
  return T{...};//永远保持传递出来的值
}
```

+ 从C++14起，使用编译器进行类型推导`auto`进行返回值

```cpp
template<typename T>
auto retV(T p)
{
  return T{...};
}
```

### 推荐使用的模板参数声明

#### 声明按值传递参数：

这种方法很简单，它会分解字符串文字和原始数组，但不能为大型对象提供最佳性能（额外的拷贝）。调用者仍然可以决定使用`std::cref()`和`std::ref()`通过引用进行传递，但是调用者必须注意保证这样做是有效的

#### 声明按引用传递参数

这种方法能够为较大的物体提供更好的性能，尤其是通过：

+ 左值引用的现有对象（左值），
+ 临时对象（prvalues）或标记为右值引用可移动的对象（xvalue），
+ 或两者都用于转发引用

由于在所有这些情况下参数都不会衰减，因此在传递字符串文字和其他原始数组时可能需要格外小心。对于转发引用，您还必须提防，使用这种方法，模板参数可以隐式地推导出引用类型

### 通用推荐

1、默认情况下，声明要通过值传递的参数。这很简单，即使使用字符串文字也通常可以使用。对于较小的参数以及临时或可移动对象，该性能都很好。传递现有的大对象（左值）时，调用者有时可以使用`std::cref()`和`std::ref()`以避免昂贵的复制

2、如果需要out或inout参数，该参数返回一个新对象或允许为调用者修改参数，请将该参数作为非恒定引用传递（除非更喜欢通过指针传递）。但是，可以考虑禁用意外接受const对象。（`enable_if`操作）

如果提供了用于转发参数的模板，请使用完美转发。也就是说，声明参数为转发引用，并在适当的地方使用`std::forward<>()`。考虑使用`std::decay<>`或`std::common_type<>`来“协调”字符串文字和原始数组的不同类型。

如果性能是关键，并且预计复制参数很昂贵，请使用常量引用。当然，如果仍然需要本地副本，这种方法是不适合的。

3、不要对性能进行直观的假设，而应该进行试验测定

### 不要过度泛化

请注意，实际上，函数模板通常不适用于任意类型的参数。相反，存在一些约束。例如，您可能知道仅传递某种类型的向量。在这种情况下，最好不要过于笼统地声明这样的函数，因为如上所述，可能会出现令人惊讶的副作用：

```cpp
template<typename T>
void printVector(std::vector<T> const& v)
{...}
```

通过在`printVector()`中声明参数`v`，我们可以确保传递的`T`不会成为引用，因为vector不能将引用作为元素类型。同样，很显然，按值传递vector几乎总是很昂贵，因为`std::vector<>`的拷贝构造函数会创建元素的副本。因此，声明要通过值传递的vector参数可能永远没有用。如果我们将参数v声明为具有类型T决定，则按值调用和按引用调用之间的联系就不那么明显了

### `std::make_pair()`为例

在第一个C++标准C++ 98中，在名称空间std中声明了`make_pair<>()`，以使用按引用调用以避免不必要的复制

```cpp
//C++98
template<typename T1, typename T2>
pair<T1, T2> make_pair(T1 const& a, T2 const& b)
{
  return pair<T1, T2>(a, b);
}
```

但是，当使用成对的字符串文字或不同大小的原始数组时，这几乎立即导致了严重的问题，C++03把它改成了值传递

```cpp
//C++03
template<typename T1, typename T2>
pair<T1, T2> make_pair (T1 a, T2 b)
{
  return pair<T1, T2>(a, b);
}
```

但是性能损失，加上C++11支持了移动语义，所以参数必须成为转发引用：

```cpp
//C++11
template<typename T1, typename T2>
constexpr pair<typename decay<T1>::type, typename decay<T2>::type>
make_pair (T1&& a, T2&& b)
{
  return pair<typename decay<T1>::type, typename decay<T2>::type>(forward<T1>(a), forward<T2>(b));
}
```

完整的实现更加复杂：为了支持`std::ref()`和`std::cref()`，该函数还将`std::reference_wrapper`的实例解包装为实际引用

C++标准库现在可以在许多地方以相似的方式完美地传递的参数，通常结合使用`std::decay<>`