<h2>模板与泛型编程</h2>

类型参数：`template<typename T>`，每个类型参数之前都需要加上关键字`typename`或`class`（这两者含义相同），如`template <typename T, class U> T calc (const T&, const U&);`

非类型参数：表示一个值而不是一个类型，实例化时被一个用户提供或编译器推断的值所代替，这些值必须是常量表达式

```cpp
template <unsigned N, unsigned M>
int compare(const char (&p1)[N], const char (&p2)[M])
{
    return strcmp(p1,p2);
}

compare("hi","mom");//会使用字面常量的大小替代N和M，实例化模板，N会为3，M为4
```

<hr>

`inline`和`constexpr`函数模板应该把说明符放在模板参数之后，返回类型之前

```cpp
template <typename T> 
inline T min(const T&, const T&);
```

<hr>

如果真的关心类型无关于可移植性，可以把原来的`<`变成`less`来定义函数

```cpp
template <typename T>
int compare(const T &v1, const T &v2)
{
    if(less<T>()(v1,v2)) return -1;
    if(less<T>()(v2,v1)) return 1;
    return 0;
};
```

<hr>

模板程序应该尽量减少对实参类型的要求；函数模板和类模板成员函数的定义通常放在头文件中，普通函数与类的成员函数定义放在源文件中

保证传递给模板的实参是支持模板所要求的操作，以及这些操作能够在模板中进行是调用者的责任

例子：打印数组，使用模板的代码如下：

```cpp
template <typename T, size_t N>
constexpr int arr_size(const T (&a)[N])
{
    return N;
|

template <typename T, size_t N>
void print(const T (&a)[N])
{
    for (int i = 0; i < arr_size(a); i++)
        cout << a[i] << " ";
    cout << endl;
}
```

<hr>

类模板：一个类模板的每一个实例都形成一个独立的类；对于一个实例化的类模板，其成员只有在使用时才被实例化

```cpp
template <typename T> class Blob{
public:
    typedef T value_type;
    typedef typename vector<T>::size_type size_type;
    
    Blob() : data(make_shared<vector<T>>()){};
    
    Blob(initializer_list<T> il) : data(make_shared<vector<T>>(il)){};
    
    size_type size() const{ return data->size(); }
    bool empty() const{ return data->empty(); }
    
    void push_back(const T &t){ data->push_back(t); }
    void push_back(T &&t){ data->push_back(std::move(t)); }
    void pop_back()
    {
        check(0, "pop_back on empty Blob");
        data->pop_back();
    }
    
    T& back()
    {
        check(0, "back on empty Blob");
        return data->back();
    }
    T& operator[](size_type i)
    {
        check(i, "subscript out of range");
        return (*data)[i];
    }
    
private:
    shared_ptr<vector<T>> data;
    void check(size_type i, const string &msg) const
    {
        if(i >= data->size())
            throw std::out_of_range(msg);
    }
};

//实例化Blob<int>和接收initializer_list<int>构造函数
Blob<int> squares = {0,1,2,3,4,5,6,7,8,9};
//实例化Blob<int>::size() const函数
for(size_t i = 0; i != squares.size(); ++i)
    squares = i * i;
```

只有在类模板的作用域中，才能够直接使用模板名而不必指定模板实参，其他情况使用一个类模板类型必须提供模板实参：

```cpp
template <typename T>
BlobPtr<T> BlobPtr<T>::operator++(int)
{
    BlobPtr ret = *this;//这里与BlobPtr<T> ret = *this;等价，作用域内无需指明模板实参
    ++*this;
    return ret;
};
```

<hr>

<h3>类模板与友元：</h3>

一对一友好：

```cpp
//前置声明
template <typename> class BlobPtr;
template <typename> class Blob;
template <typename T>
bool operator==(const Blob<T>&, const Blob<T>&);

template <typename T> class Blob{
    friend class BlobPtr<T>;
    friend bool operator==(const Blob<T>, const Blob<T>&);
    //其他成员定义同上
};

Blob<char> ca;//BlobPtr<char>和operator==<char>都是本对象的友元
```

通用与特定的友好关系

```cpp
//前置声明，将模板的一个特定实例声明为友元时使用
template <typename T> class Pal;

//C是非模板类
class C{
    friend class Pal<C>;//用C实例化的Pal是C的友元
    template <typename T> friend class Pal2;//Pal2的所有实例都是C的友元，这种情况无需前置声明
};

//C2是模板类
template <typename T> class C2{
    friend class Pal<T>;//C2的每个实例与相同实例化的Pal声明为友元
    template <typename X> friend class Pal2;//Pal2的所有实例都是C的友元，这种情况无需前置声明
    friend class Pal3;//Pal3是非模板类，是C2所有实例的友元，不需要Pal3的前置声明
};
```

**C++11**：模板类型参数可以声明为友元

```cpp
template <typename Type> class Bar{
    friend Type;//访问权限授予用来实例化Bar的类型
    //...
};

//对于某个类型名Foo，Foo将会成为Bar<Foo>的友元，如Sales_data将成为Bar<Sales_data>的友元
```

<hr>

<h3>模板类型别名</h3>

可以通过定义`typedef`来引用已经实例化的类（必须是已经实例化的，不能定义`typedef`引用一个模板，因为模板不是一个类型）；**C++11**：使用`using`允许我们为类模板定义一个别名

```cpp
typedef Blob<string> StrBlob;

template <typename T> using twin = pair<T, T>;
twin<string> authors;//authors是一个pair<string, string>类型
```

<hr>

<h3>模板参数</h3>

模板参数会隐藏外层作用域中声明的相同名字，一个特定文件所需要的所有模板声明一起放在文件开始处，出现于任何使用这些模板的代码之前

```cpp
typedef double A;
template <typename A, typename B> void f(A a,B b)
{
    A tmp = a;//这里tmp的类型是模板参数A的类型，由传入实参推断得到，而不是double
    double B;//错误，模板参数名不能重用（可以想象成B是int，你不能取一个变量名称叫做int）
}
```

**C++假定通过作用域运算符`::`访问的名字不是类型**，所以我们当使用某个模板类型参数的类型成员，就一定要显式地使用关键字`typename`告诉编译器这个名字是一个类型（否则编译器会混淆`T::size_type *p`到底是定义一个指针还是`size_type`的静态数据成员与名为`p`的变量相乘）

```cpp
template <typename T>
typename T::value_type top(const T& c)//使用关键字提示编译器是类型
{
    if (!c.empty())
        return c.back();
    else 
        return typename T::value_type();
}
```

默认模板实参

**C++11**：我们可以为函数与类模板提供默认实参，只有当它右侧的所有参数都有默认实参时，模板参数才可以有默认实参

```cpp
template<typename T, typename F = less<T>>//F表示可调用对象的类型
int compare(const T &v1, const T &v2, F f = F())//定义函数参数f绑定到可调用对象上
{
	if (f(v1, v2)) return -1;
	if (f(v2, v1)) return 1;
	return 0;
}

bool i = compare(0,42);//i = -1

Sales_data item1(cin), item2(cin);
bool j = compare(item1, item2, compareIsbn);//传入一个可调用对象，返回结果能转换成bool值，接收的实参能与前两者实参兼容，这里compare结果依赖于isbn
```

无论何时使用一个类模板，都必须在模板名之后接上尖括号

```cpp
template <class T = int> class Numbers{
public:
    Numbers(T v = 0): val(v){}
private:
    T val;
};

Numbers<long double> lots_of_precision;
Numbers<> average_precision;//空的<>表示我们希望使用默认类型
```

<hr>

<h3>成员模板</h3>

普通类的成员模板

```cpp
class DebugDelete{
public:
    DebugDelete(ostream &s = cerr) : os(s){ }
    template <typename T> void operator()(T *p) const
      { os << "delete unique_ptr" << endl; delete p;}
private:
    ostream &os;
};

double *p = new double;
DebugDelete d;
d(p);//调用DebugDelete::operator()(double*)释放p
DebugDelete()(p)//在一个临时对象DebugDelete对象上调用operator()(int*)
```

类模板的成员模板

```cpp
template <typename T> class Blob{
    template <typename It> Blob(It b,It e);
    //...
};

//在模板外为其定义成员模板时，需要同时提供模板参数列表
template <typename T>
template <typename It>
    Blob<T>::Blob(It b,It e): data(std::make_shared<vector<T>>(b,e)){ }
```

<hr>

<h3>控制实例化</h3>

**C++11**：显式实例化，避免多个文件中实例化相同模板引发的额外开销；实例化定义会实例化所有成员，所以所用类型必须能用于模板的所有成员函数

遇到`extern`模板声明时不会在本文件中生成实例化代码

```cpp
extern template class Blob<string>;//声明
template int compare(const int&, const int &);//定义

Blob<string> sa1, sa2;//实例化会出现在其他位置

//在本文件中实例化
Blob<int> a1 = {0,1,2,3,4};
Blob<int> a2(a1);

int i = compare(a1[0], a2[0]);//实例化会出现在其他位置
```

<hr>

<h3>模板实参推断</h3>

从函数实参来确定模板实参的过程；如果函数的形参的类型使用了模板类型参数，则只允许两种转换：

+ const转换：非const对对象的引用（指针）传递给一个const对象（指针）的形参
+ 数组/函数到指针的的转换：数组实参转换为数组首地址指针，函数实参与该函数类型的指针

```cpp
template <typename T> int compare(const T&, const T&);

compare("Hi", "world");//不合法，允许数组到指针的转换，但是如果是引用，数组不会转换成一个指针
```

提供显式模板实参：在函数名之后，实参列表之前，用尖括号给出（注意，显式模板实参从左至右顺序与对应的模板参数匹配，否则要我们自己提供）

```cpp
template <typename T1, typename T2, typename T3>
T1 sum(T2, T3);//因为没有任何实参供编译器推断T1，所以调用时必须为T1提供显式模板实参

auto val3 = sum<long long>(i, lng);//类型为long long sum(int, long)

//不好的设计
template <typename T1, typename T2, typename T3>
T3 bad_sum(T2, T1);

//在这种设计下，我们需要为每个参数指定实参类型
auto val2 = bad_sum<long long, int, long>(i, lng);
```

**C++11**：尾置返回类型与类型转换

显式指定模板实参有时会给用户带来不必要的麻烦，而且不会有什么好处；假如我们不知道返回结果的准确类型，但知道所需类型是所处理的序列的元素类型时，我们可以使用尾置返回类型与`decltype`配合进行类型推导

```cpp
template <typename T>
??? &fcn(It beg, It end)
{
    //处理序列
    return *beg;//返回其中一个元素的引用
}

vector<int> vi = {1,2,3,4,5};
Blob<string> ca = {"hi", "bye"};
auto &i = fcn(vi.begin(),vi.end());//fcn返回int&
auto &s = fcn(ca.begin(),ca.end());//fcn返回string&

//正确方法：
template <typename It>
auto fcn(It beg, It end) -> decltype(*beg)
{
    //处理序列
    return *beg;//返回其中一个元素的引用
}
```

我们这里存在一个问题：对于传递的参数的类型，我们一无所知，只可以使用迭代器，但这些迭代器只是引用，而不会生成元素

所以，当我们需要获得元素类型，可以使用标准库中的类型转换，这些模板定义在`<type_traits>`头文件下；

||标准类型转换模板||
|:-:|:-:|:-:|
|对`Mod<T>`，其中`Mod`为|若`T`为|则`Mod<T>::type`为|
|`remove_reference`|`X&`或`X&&`|`X`|
||否则|`T`|
|`add_const`|`X&`、`const X`或函数|`T`|
||否则|`const T`|
|`add_lvalue_reference`|`X&`|`T`|
||`X&&`|`X&`|
||否则|`T&`|
|`add_rvalue_reference`|`X&`或`X&&`|`T`|
||否则|`T&&`|
|`remove_pointer`|`X*`|`X`|
||否则|`T`|
|`add_pointer`|`X&`或`X&&`|`X*`|
||否则|`T*`|
|`make_signed`|`unsigned X`|`X`|
||否则|`T`|
|`make_unsigned`|带符号类型|`unsigned X`|
||否则|`T`|
|`remove_extent`|`X[n]`|`X`|
||否则|`T`|
|`remove_all_extents`|`X[n1][n2]...`|`X`|
||否则|`T`|

```cpp
//将会得到元素类型本身
template <typename It>
auto fcn2(It beg, It end)->typename remove_reference<decltype(*beg)>::type
{
    return *beg;
}

//保证容纳加法结果的sum（要求a+b还是要有某个内置的整数类型进行容纳）
template<typename T1, typename T2>
auto sum(T1 a, T2 b) -> decltype(a + b)
{
    return a + b;
}
```

<hr>

<h3>引用折叠</h3>

只能应用于间接创建的引用的引用，如类型别名或模板参数

```cpp
template <typename T> void f3(T&&);
f3(42);//正确，实参是int的右值，模板参数T是int

int i= 5;
f3(i);//绑定左值也是正确的，这里引发了引用折叠
```

**C++11**：通常我们不能把一个右值引用绑定到一个左值上，但是有两个例外：

+ 编译器会推断模板参数T的类型是左值引用，也就是`int&`，而不是`int`；现在f3的函数参数看上去像`int& &&`，我们一般不能定义一个引用的引用，但是通过类型别名或通过模板类型参数间接定义是可以的
+ 如果我们间接创建了一个引用的引用，则这些引用形成了“折叠”。除了类型`X&& &&`折叠形成`X&&`外，其他`X& &`、`X& &&`和`X&& &`都折叠形成`X&`（即普通左值引用）

这里暗示了：我们可以将任何类型的实参传递给`T&&`类型的函数参数

`std::move`：从一个左值通过`static_cast`到一个右值引用是允许的，将右值引用绑定到左值的特性允许它们截断左值

```cpp
template <typename T>
typename remove_reference<T>::type&& move(T&& t)
{
    return static_cast<typename remove_reference<T>::type&&>(t);
}
```

转发：在调用中使用`std::forward`保持类型信息（能够保持原始实参的类型），定义在头文件`<utility>`中

```cpp
template <typename F,typename T1, typename T2>
void flip2(F f, T1 &&t1, T2 &&t2)
{
    f(t2,t1);//错误，右值引用类型int不能绑定到左值的int（有名字的都是左值）
}

void f(int &&i,int &j)
{
    cout << i << " " << j << endl;
}

int main(int argc, char *argv[])
{
    int i = 5;
    flip2(f,i,42);
}
```

上面代码证明了简单转发是不可行的，`forward`实现了完美转发，必须通过显式模板实参来进行调用，`forward`返回该显式实参类型的右值引用。即`forward<T>`的返回类型是`T&&`

**通常情况下，我们使用`forward`传递那些定义为模板类型参数的右值引用的函数参数，通过引用折叠，`forward`保持给定实参的左值/右值属性**

```cpp
template <typename Type> intermediary(Type &&arg)
{
    finalFcn(std::forward<Type>(arg));
    // ...
}
```

使用`forward`之后，可以实现完美转发：

```cpp
template <typename F, typename T1, typename T2>
void flip(F f, T1 &&t1, T2 &&t2)
{
    f(std::forward<T2>(t2), std::forward<T1>(t1));
}
```

<hr>

重载与模板

函数模板可以被另一个模板或一个普通非模板函数重载，名字相同的函数必须具有不同数量和类型的参数

```cpp
//打印任何我们不能处理的类型
template<typename T>
string debug_rep(const T &t)//函数1
{
    ostringstream ret;
    ret << t;
    return ret.str();
}

template<typename T>
string debug_rep(T *p)//函数2
{
    ostringstream ret;
    ret << "pointer: " << p;
    if (p)
        ret << " " << debug_rep(*p);
    else
        ret << " null pointer";
    return ret.str();
}
```

当有多个重载模板对一个调用提供同样好的匹配时，应选择最特例化的版本（优先级：`普通非模板函数 > 特例化高的模板函数 > 特例化低的模板函数`）

```cpp
string s("hi");
cout << debug_rep(s) << endl;//只能匹配1
cout << debug_rep(&s) << endl;//两个函数都可以，函数1的参数：const string*&，函数2的参数：string*（第二个精确匹配）

const string *sp = &s;//多了一个const
cout << debug_rep(sp) << endl;//1：const string*& 2：const string *，两个都是精确匹配
//这里被解析成函数2（没有二义性），因为函数2更特例化，函数1可以用于任何类型，函数2只能用于指针类型

string debug_rep(const string &s)//函数3，普通非模板函数
{
    return '"' + s + '"';
}
cout << debug_rep(s) << endl;//函数1的模板为debug_rep<string>(const string&)，函数3也能匹配，使用函数3（函数3特例化最高）

cout << debug_rep("hi") << endl;//C风格字符串
//函数1：const T&，T绑定到char[3]
//函数2：T*，T绑定到const char（数组到指针的转化是许可的，所以是最特例化的，调用此函数）
//函数3：要进行一次const char*到string的转化（转化可行但是需要进行一次用户定义的转化，匹配度不如函数2）
```

另外注意：缺少函数声明会很可能会导致程序调用错误的模板函数进行实例化，而不是调用我们希望的非模板函数的情况，**最好在定义任何函数前，记得声明所有重载的函数版本**

<hr>

<h3>可变参数模板</h3>

**C++11**：接受一个【可变数目参数】（也称为【参数包】）的模板函数或模板类。

```cpp
template <typename T, typename ... Args>
void foo(const T &t, const Args& ... rest);

int i = 0;
double d = 3.14;
string s = "how now brown cow";
foo(i, s, 42, d);//包中有3个参数(const int&, const string&, const int&, const double&);
foo(s, 42, "hi");//包中有2个参数(const string&, const int&, const char[3]&);
foo(d, s);//包中有1个参数(const double&, const string&);
foo("hi");//空包(const char[3]&);
```

当我们想知道包中有多少元素时，可以使用`sizeof...`运算符

```cpp
template <typename ... Args> void g(Args ... args)
{
    cout << sizeof...(Args) << endl;
    cout << sizeof...(args) << endl;
}
```

编写可变参数函数模板：通常是**递归设计**的，第一步处理包中的第一个实参，然后用剩余实参调用自身

```cpp
template<typename T>
ostream &print(ostream &os, const T &t)//仅剩1个实参的时候，终结下面的循环调用
{
    return os << t;
}

template<typename T, typename ... Args>
ostream &print(ostream &os, const T &t, const Args &... rest)
{
    os << t << ", ";//打印第一个实参
    return print(os, rest...);//递归调用打印其他实参
}
```

此外可以对包进行拓展，需要提供用于每个拓展元素的模式；

拓展：把它分解为构成的元素，对于每个元素应用模式，获得拓展后的列表；在模式右边放`...`来触发拓展操作

转发参数包：自己重写`emplace_back`实现完美转发

```cpp
class StrVec
{
public:
    template<typename ... Args>
    void emplace_back(Args &&... args)//每个函数参数都是指向其对应实参的右值引用
    {
        chk_n_alloc();//如果需要重新分配StrVec的内存空间
        alloc.construct(first_free++, std::forward(Args)(args)...);
    }
};
```

`forward`保证了实参的原始类型，用`construct`在`first_free`指向位置处创建一个元素，`...`实现了参数包拓展（既拓展了模板参数包`Args`，也拓展了函数参数包`args`），形成`std::forward<Ti>(ti)`

```cpp
svec.emplace_back(10,'c');//将cccccccccc添加为新的尾元素

//会拓展出：
std::forward<int>(10),std::forward<char>(c)
```

这样能够保证转发的完美性，总的来说，可变参数函数通常将它们的参数转发给其他函数，由于`fun`的参数是右值引用，所以我们可以传递给它任何类型的实参，然后通过`forward`，它们的所有类型信息在调用`work`时得到保持

```cpp
template<typename ... Args>
void fun(Args &&... args)//将Args拓展为一个右值引用列表
{
    //work的实参既拓展Args又拓展args
    work(std::forward<Args>(args)...);
}
```

<hr>

<h3>模板特例化</h3>

模板的一个独立的定义，在其中一个或多个模板参数被指定为特定的类型；**本质：为原模板的一个特殊实例提供了定义**

在特例化一个模板时，必须为原来模板的每个模板参数提供实参，使用`template <>`空的尖括号对指出；

**我们必须为原模板的所有模板参数提供实参，即我们不能部分特例化函数模板，但是对于类模板，可以部分特例化**

```cpp
template <size_t N, size_t M>
int compare(const char (&)[N], const char (&)[M]);//函数1
//特例化以下模板
template <typename T> int compare(const T&, const T&);//函数2

//处理字符数组的指针的特例化
template <>
int compare(const char* const &p1, const char* const &p2)
{
    return strcmp(p1, p2);
}

//注意与上面特例化的区别，这里是普通非模板函数
int compare(const char* const &p1, const char* const &p2)//函数3
{
    return strcmp(p1, p2);
}

//以下比较函数1&2（参数是C风格字符串字面常量，应该是const char[]类型）
compare("hi", "mom");//1&2都可以，而且提供同样好的匹配，但是函数1只接受字符数组，更加实例化，选1
//我们定义的特例化是针对字符数组的指针的，这里不是指针，所以不会引发

//以下比较函数1&2&3，同样的参数
compare("hi", "mom");//1&2&3都是同样好的匹配，但是如果非模板与模板同样好时，选择非模板，选3
```

本质上，一个特例化的版本本质上是一个实例化的模板，而不是一个重载版本，所以不会影响函数匹配（等于我们部分接管了编译器的工作）

**模板及其特例化版本应该声明在同一个头文件中。所有同名模板的声明在前，这些模板的特例化版本在后。顺序非常重要！！否则会由原模板实例化生成代码，难以追踪这样的bug**

类模板特例化

必须要在原模板定义所在的命名空间中特例化它，下面例子展示全特例化`std::hash`类

```cpp
namespace std{//打开命名空间
template <>
struct hash<Sales_data>
{
    //散列无序容器要定义以下类型
    typedef size_t result_type;
    typedef Sales_data argument_type;
    size_t operator()(const Sales_data& s) const;
    //使用合成的拷贝控制成员和默认构造函数
};

//调用运算符为给定类型的值定义一个hash函数，三个从标准库生成得到的hash值进行异或
size_t hash<Sales_data>::operator()(const Sales_data& s) const{
    return hash<string>()(s.bookNo) ^
           hash<unsigned>()(s.units_sold) ^
           hash<double>()(s.revenue);//hash<string>这种是通过标准库来生成hash值
}
}//关闭命名空间（注意：没有分号）
```

由于其使用了`Sales_data`的私有成员，所以必须是`Sales_data`的友元类，而且为了让`Sales_data`用户能使用`hash`的特例化版本，应该在`Sales_data`的头文件中定义这个特例化版本：

```cpp
template <typename T> class std::hash;//友元声明必须
class Sales_data{
    friend class std::hash<Sales_data>;
};

//当将Sales_data作为容器关键字类型时，编译器自动调用这个特例化版本
unordered_multiset<Sales_data> SData;
```

部分特例化

例子：标准库中的`remove_reference`类型，通过一系列特例化模板完成其功能

```cpp
template<typename T>
struct remove_reference
{
    typedef T type;
};

template<typename T>
struct remove_reference<T &>
{
    typedef T type;
};

template<typename T>
struct remove_reference<T &&>
{
    typedef T type;
};
```

但是特例化可以只特例化成员而不是类，下面的代码，如果使用`int`以外的类型使用`Foo`，成员与往常一样进行实例化，用`int`使用`Foo`时，Bar以外成员与往常一样进行实例化，使用`Bar`时，会使用我们定义的特例化版本

```cpp
template <typename T> struct Foo{
    Foo(const T &t = T()) : mem(t){ }
    void Bar() {/* */}
    T mem;
    //...
}
template <>
void Foo<int>::Bar()//特例化Foo<int>的成员Bar
{
    //...
}
```
