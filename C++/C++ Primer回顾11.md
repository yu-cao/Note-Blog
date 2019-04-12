<h2>重载运算与类型转换</h2>

建议：不要重载逻辑与`&&`，逻辑或`||`，逗号`,`，取地址`&`这四个运算符，因为无法保留下求值顺序和/或短路求值属性，后两者常常与类类型对象有关，对这些的重载可能对用户造成混淆（一直以来的求值规则被改变了）**程序员需要负责任地使用重载运算符，不应该强行曲解运算符常规含义以适合某个类型**

运算符定义为成员函数/非成员函数：

+ 赋值`=`，下标`[]`，调用`()`和成员访问箭头`->`运算符必须是成员函数
+ 复合赋值运算符一般来说应该是成员，但非必须
+ 改变对象状态的运算符或与给定类型密切相关的运算符，如`++`，`--`和解引用`*`运算符通常是成员
+ 具有对称性的运算符通常应该是非成员的，如算术，相等性，关系和位运算等

<h3>重载输入输出运算符</h3>

必须是非成员函数，一般被声明为友元

<h4>输出运算符<<</h4>

第一个形参：非常量ostream对象的引用；第二个形参：常量的引用，常量是我们想要打印的类类型；返回值：ostream形参。输出运算符尽量减少格式化操作（如换行）

```cpp
ostream &operator<<(ostream &os, const Sales_data &item)
{
    os << item.isbn() << " " << item.units_sold << " " << item.revenue << " " << item.avg_price();
    return os;
}
```

<h4>输入运算符>></h4>

第一个形参：将要读取的流的引用；第二个形参：要读到的对象的引用（非常量）；返回值：某个给定流的引用

```cpp
istream &operator>>(istream &is, Sales_data &item)
{
    double price;
    is >> item.bookNo >> item.units_sold >> price;
    if(is)//检查输入是否成功
        item.revenue = item.units_sold * price;
    else
        item = Sales_data();//输入失败，对象赋予默认状态
    return is;
}
```

输入运算符必须要处理输入时失败的情况：1、错误数据类型导致不匹配，无法构造（如int处输入了字符串），导致后续对流的其他使用都会失败 2、读取到达文件末尾或输入流出现其他错误

读取操作发生错误时，输入运算符应该负责从错误中恢复：设置failbit，eofbit，badbit等等标志进行标识

<h3>算术与关系运算符</h3>

除了赋值运算符外，其他的必须是非成员函数，一般被声明为友元<br>
赋值/复合赋值运算符应该在类内定义，返回左侧对象的引用

|算术符号|函数头|
|:-:|:-:|
|`+`/`-`/`*`/`/`|`Sales_data operator+(const Sales_data &lhs, const Sales_data &rhs)`|
|`==`/`!=`|`bool operator==(const Sales_data &lhs, const Sales_data &rhs)`|
|`<`/`<=`/`>`/`>=`|`bool operator<(const Sales_data &lhs, const Sales_data &rhs)`|
|`=`/`+=`/`-=`|`Sales_data& Sales_data::operator+=(const Sales_data &rhs)`|

<h3>下标运算符</h3>

必须是成员函数，通常以所访问元素的引用作为返回值，最好同时定义常量版本与非常量版本

```cpp
class StrVec{
public:
    std::string& operator[](std::size_t n)
        {return elements[n];}
    const std::string& operator[](std::size_t n) const//这里const含义为隐含的this底层指针是const的，也就是说*this这个对象内数据是只读的
        {return elements[n];}
private:
    std::string *elements;
};

StrVec svec;
const StrVec cvec = svec;//把svec元素拷贝到cvec

//如果svec中有元素，对一个元素运行string的empty函数
if(svec.size() && svec[0].empty()){
    svec[0] = "zero";//正确，下标运算符返回string的引用
    cvec[0] = "zip";//错误，对cvec取下标返回是常量引用
}
```

<h3>递增和递减运算符</h3>

建议将其设定为成员函数，因为它们改变了操作对象的状态，应该为类定义两个版本的递增/递减运算符（前置/后置）

后置版本接受一个额外的（不被使用的）int类型的形参进行区分，我们不会用到这个形参，所以无需对其命名

核心是保持这两者的设计操作与内置的尽可能一致

+ 前置：应该返回递增(/递减）后对象的**引用**
+ 后置：应该返回对象的原**值**（递增/递减前的值）

```cpp
class StrBlobPtr{
public:
    //前置
    StrBlobPtr& operator++();
    StrBlobPtr& operator--();
    
    //后置
    StrBlobPtr operator++(int);
    StrBlobPtr operator--(int);
};
```

<h3>成员访问运算符</h3>

解引用`*`和箭头运算符`->`，常常在迭代器类和智能指针类中使用，都应该是类的成员

重载的箭头运算符必须返回类的指针或自定义了箭头运算符的某个类的对象

```cpp
class StrBlobPtr{
public:
    std::string& operator*() const
    {
        auto p = check(curr, "dereference past end");//检查是否在作用范围内
        return (*p)[curr];//(*p)是对象所指的vector
    }
    std::string* operator->() const
    {
        return & this->operator*();//委托给解引用运算符，也就是把->变成(*p).的形式
    }
    //...
};
```

<h3>函数调用运算符</h3>

```cpp
struct absInt{
    int operator()(int val) const{
        return val < 0 ? -val : val;
    }
};

int i = -42;
absInt absObj;
int ui = absObj(i);
```

函数调用运算符必须是成员函数，一个类可以定义多个不同版本的调用运算符，只要在参数类型上区别出来就好了（类似重载）

类中定义了调用运算符，这个类的对象称为函数对象，因为可以调用这种对象

lambda表达式是当代码需要一个简单的函数且这个函数不会被在其他地方使用时的方法，类似于匿名函数对象<br>
当函数需要多次使用而且要保存一些状态时，最好使用普通函数对象

标准库函数对象，定义在头文件`<functional>`中

|算术|关系|逻辑|
|:-:|:-:|:-:|
|`plus<T>`|`equal_to<T>`|`logical_and<T>`|
|`minus<T>`|`not_equal_to<T>`|`logical_or<T>`|
|`multiplies<T>`|`greater<T>`|`logical_not<T>`|
|`divides<T>`|`greater_equal<T>`||
|`modulus<T>`（取余）|`less<T>`||
|`negate<T>`|`less_equal<T>`||

实例：

```cpp
std::sort(vec.begin(),vec.end(),std::greater<int>());//进行降序排列

//试图比较指针内存地址来sort指针的vector
std::vector<std::string *> nameTable;
std::sort(nameTable.begin(),nameTable.end(),[](std::string *a,std::string *b){return a<b;});//错误：nameTable中指针彼此没有关系，用<将产生未定义的行为
std::sort(nameTable.begin(),nameTable.end(),std::less<std::string*>());//正确，标准库规定指针的less是定义良好的
```

+ 统计大于1024的值有多少？
+ 找到第一个不等于pooh的字符串
+ 将所有的值乘以2
+ 一个给定的int能不能被int容器的所有元素整除

```cpp
count_if(vec.begin(),vec.end(),bind2nd(greater<int>(),1024));//也就是 ?（第一参数） > 1024（第二参数）进行遍历
find_if(vec.begin(),vec.end(),bind2nd(not_equal_to<string>(),"pooh"));
transform(vec.begin(),vec.end(),vec.begin(),bind2nd(multiplies<int>(),2));

bool divideByAll(vector<int> &ivec, int dividend)
{
    return count_if(ivec.begin(),ivec.end(),bind1st(modulus<int>,dividend)) == 0;//也就是 dividend（1st参数） % 每个ivec的值（2nd参数），所以bind1st
}
```

bind1st和bind2nd：绑定给定参数 x 到给定二元函数对象 f 的第一或第二参变量。即，在产生的包装器内存储 x ，若调用它，则将 x 传递为 f 的第一或第二参数。

1) 绑定 f 的第一参数到 x 。等效地调用`std::binder1st<F>(f, typename F::first_argument_type(x))`

2) 绑定 f 的第二参数到 x 。等效地调用`std::binder2nd<F>(f, typename F::second_argument_type(x)) `

函数表：以下函数共享同一种调用形式：`int(int,int)`

```cpp
int add(int i,int j){return i + j;}
auto mod = [](int i, int j){return i % j;};
struct divide{
    int operator()(int denominator, int divisor){
        return denominator / divisor;
    }
};
```

函数表用于存储指向这些可调用对象的“指针”，需要特定操作时直接查表即可

```cpp
map<string, int(*)(int,int)> binops;//运算符与函数指针之间的映射关系

binops.insert({"+",add});//正确
binops.insert({"%", mod});//错误，mod不是一个函数指针，lambda有它自己的类型，类型不匹配
```

function标准库：

|操作|含义|
|:-:|:-:|
|`function<T> f;`|f是一个用来存储可调用对象的空function，调用形式与函数类型T相同|
|`function<T> f(obj);`|在f中存储可调用对象obj的副本|
|`f`|将f作为条件：f含有一个可调用对象时为真，否则为假|
|`f(args)`|调用f中的对象，参数是args|
|**定义为`function<T>`的成员的类型**||
|`result_type`|该function类型的可调用对象返回的类型|
|`argument_type`<br>`first_argument_type`<br>`second_argument_type`|当T有一个或两个实参时定义的类型，如果只有一个实参，则`argument_type`是该类型同义词，如果T有两个实参，则`first_argument_type`和`second_argument_type`分别代表两个实参的类型|

比如上面可以有这样的指定类型：`function<int(int,int)>`

```cpp
function<int(int,int)> f1 = add;
function<int(int,int)> f2 = divide();
function<int(int,int)> f3 = [](int i,int j){return i*j;};

cout << f1(4,2) << endl;//6
cout << f2(4,2) << endl;//2
cout << f3(4,2) << endl;//8

map<string,function<int(int,int)>> binops = {
    {"+", add},
    {"-", std::minus<int>()},
    {"*", [](int i,int j){return i * j;}},
    {"/", divide()},
    {"%", mod}
};

//调用举例：
binops["+"](10,5);//调用add(10,5)
```

我们不能（直接）把重载函数的名字存入function类型的对象中，因为存在二义性问题

```cpp
//有多个add情况下，函数指针fp所指的add是调用两个int的版本
int (*fp)(int, int) = add;
binops.insert({"+", fp});//将fp指针存入

//或者使用lambda调用我们希望的版本
binops.insert({"+", [](int a,int b){return add(a,b);}});
```

类型转换运算符：将一个类类型的值转换成其他形式。**没有显式的返回类型，没有形参，必须定义成类的成员函数，类型转换函数通常应该是const**；形式：`operator type() const;`

```cpp
class SmallInt{
public:
    SmallInt(int i = 0):val(i)
    {
        if(i<0||i>255)
            throw std::out_of_range("Bad SmallInt value");
    }
    operator int() const {return val;}//把SmallInt转化成int，但是val要和int对应，可以隐式转换
    operator int*() const {return 42;}//错误，42不是一个指针，无法转换

private:
    size_t val;
};

SmallInt si;
si = 4;//首先，4隐式转换成SmallInt，然后调用SmallInt::operator=
si + 3;//首先，si隐式转换成int，然后执行整数的加法
```

上面的代码通过构造函数把算术类型值->SmallInt对象；通过类型转换运算符把SmallInt->int；**不应该过度使用或滥用曲解类型转换函数**

在实践中，类很少提供类型转换运算符，因为这往往给用户以惊吓而不是帮助

**C++11**：显式的类型转换运算符；**要求对类的类型转换必须显式执行，除非表达式被用作条件（这种情况下显式转换也会隐式执行）**

```cpp
class SmallInt{
public:
    explicit operator int() const {return val;}
    //其他同上保持一致
}

SmallInt si = 3;//正确，构造函数不是显式的
si + 3;//错误，这里需要隐式转换，但是运算符被定义为显式的
static_cast<int>(si) + 3;//正确，显式请求转换
```

这个在IO流中非常有用，如经典的输入语句：`while(std::cin >> value)`，`cin`被`istream operator bool`类型转换函数（一般定义成`explicit`的）隐式进行转换，如果`cin`的条件状态是`good`，就返回为`true`，否则为`false`

如果对IO流不定义成`explicit`可能会在旧标准下允许这样的情况：

```cpp
int i = 42;
cin << i;//cin的bool类型转换符把其转换成bool，然后bool提升成了int，用左移运算符左移42位，这显然是荒谬的
```

避免可能产生二义性的类型转换，不要为类定义相同的类型转换，也不要在类中定义两个以上的转换源或转换目标是算术类型的转换；也不要令两个类执行相同的类型转换：Foo类定义了接收Bar类构造函数，就不要再Bar类中定义转换目标是Foo类的类型转换运算符

总之，除了显式地向bool类型转换外，应该尽量避免定义类型转换函数而且尽可能限制那些“显然正确”的非显式构造函数

```cpp
struct A{
    A(int = 0);//不要创建转换源都是算术类型的类型转换
    A(double);
    
    operator int() const;//不要创建转换对象都是算术类型的类型转换
    operator double() const;
};
void f2(long double);
A a;
f2(a);//二义性错误，调用f(A::operator int())还是f(A::operator double())？

long lg;
A a2(lg);//二义性错误：调用A::A(int)还是A::A(double)？
```

此外，运算符重载也是一种重载函数，也需要判定应该使用内置运算符还是重载运算符，当这两者的优先级没有区别时，一样会引发二义性错误

```cpp
class SmallInt{
    friend SmallInt operator+(const SmallInt&, const SmallInt&);
public:
    SmallInt(int i = 0);
    operator int() const {return val;}
private:
    size_t val;
};

SmallInt s1,s2;
SmallInt s3 = s1 + s2;//使用重载的operator+
int i = s3 + 0;//二义性错误
```