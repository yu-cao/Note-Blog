<h3>求值</h3>

求值顺序：除了`&& || ?: ,`这四种运算符规定了求值顺序之外，大多数运算符不会明确指定求值顺序以供编译器优化<br>
例如：`f() + g() * h() + j()`这样的表达式，优先级和结合律都是确定的，但是调用顺序没有定义，可按任意顺序进行，如果其中某几个函数同时影响某个对象，这条语句将会成为一个UB

短路求值：<br>
`&&`左侧运算对象为**真**才对右侧运算对象进行求值<br>
`||`左侧运算对象为**假**才对右侧运算对象进行求值<br>
`,`首先对左侧的表达式求值，然后将求值结果丢弃

除法运算：**C++11**规定商一律向0取整（即直接切除小数部分）

位运算：位运算对于符号位的处理由机器决定，所以建议**仅将位运算符用于处理无符号类型**

显式转换：
`static_cast dynamic_cast const_cast reinterpret_cast`
+ `static_cast`：任何具有明确定义的类型转换，只要不包含底层const，就都可以使用static_cast
+ `dynamic_cast`：运行时类型识别
+ `const_cast`：只能改变运算对象的底层const
+ `reinterpret_cast`：从位模式上进行强制改变类型

**C++11**:范围for语句，遍历容器或其他序列的所有元素(在范围for语句中预存了end()的值，如果增加/删除元素可能导致end()的结果无效，程序进入未定义情况)

```cpp
    vector<int> v = {0,1,2,3,4,5,6,7,8,9};
    for(auto &r : v)
        r *= 2;
```

异常：stdexcept定义了几种常见的异常类


|\<stdexcept\>定义的异常类|不包括全部|
|---|---|
|exception|最常见的问题|
|runtime_error|运行时才能检测出的问题|
|range_error|生成的结果超出了有意义的值域范围|
|overflow_error|计算上溢|
|invaild_argument|逻辑错误：无效参数|
|domain_error|逻辑错误：参数对应的结果值不存在|

只能以默认初始化方式初始化exception、bad_alloc和bad_cast对象；只能用string对象或C风格字符串初始化除此以外的其他异常类型对象，不能用默认初始化

```cpp
try
{
    //...
    throw runtime_error("...");
}catch
{
    cout << err.what() << endl;// what()成员函数范围const char*，提供关于异常的一些文本信息
    //...
}
```

函数传值：除了形参是引用类型，它会绑定到对应的实参上去，否则都是拷贝赋值给形参。**在C++中，建议使用引用类型代替C语言的通过指针访问函数外部对象的习惯。如果某类型不支持拷贝操作(如IO)，就只能通过引用形参访问该类型的对象。** 如果函数无需改变引用形参的值，最好声明为常量引用。

含有可变形参的函数：**C++11** `initializer_list<T>`（如果实参数量未知但所有实参类型都相同）要求其对象中元素必须是常量值

```cpp
void error_msg(initializer_list<string> il)
{
    for(auto beg = il.begin(); beg != il.end(; ++beg))
        cout << *beg << " ";
    cout << endl;
}
// 想向initializer_list形参中传递一个值的序列，必须把序列放到一对花括号内
// expected 和 actual 是string对象
if (expected != actual)
    error_msg({"fuctionX", expected, actual});
else
    error_msg({"fuctionX", "okay"});
```

返回值**C++11** :函数可以返回花括号包围的值的列表

```cpp
vector<string> process()
{
    // ... expected 和 actual 是string对象
    if(expected.empty())
        return {};// 返回空的vector对象
    else if(expected == actual)
        return {"functionX", "okay"};// 返回列表初始化的vector对象
    else
        return {"functionX", excepted, actual};
}
```

**C++11**:尾置返回类型

```cpp
// func接收一个int的实参，返回一个指针，指向含有10个整数的数组
auto func(int i) -> int(*)[10];
// 等价于下面这个，但是上面的尾置类型更为清晰易读
int (*func(int i))[10];

//同样的，对于函数指针使用尾置返回类型,以下两者等价
int (*f1(int))(int*, int);
auto f1(int) -> int (*)(int*, int);
// 也可以使用以下方式等价
using pF = int (*)(int *, int);// function pointer
pF f1(int);
auto f1(int)->pF;

// 以下是错误的
using F = int(int *, int);// function
auto f(int)->F;// error，F是函数类型，函数返回值不能是函数
```

函数重载：函数入口参数必须有区别。顶层const不影响传入，无法进行区分，底层const是可以区分的

```cpp
Record lookup(Phone);
Record lookup(const Phone);// 重复声明，并没有重载

Record lookup(Account&);
Record lookup(const Account&);// 新函数，作用于常量引用
```

const_cast与重载：

```cpp
// 函数1
const string &shorterString(const string &s1, const string &s2)
{
    return s1.size() <= s2.size() ? s1 : s2;
}

// 函数2
string &shorterString(string &s1, string &s2)
{
    auto &r = shorterString(const_cast<const string&>(s1), const_cast<const string&>(s2));// 通过const_cast使得符合要求，调用函数1
    return const_cast<string&>(r);
}
```

默认实参：一旦某个形参被赋予了默认值，它后面的所有形参都必须要有默认值。使用默认实参调用函数时，只能省略尾部的实参。通常，应该在函数声明中指定默认实参，并且将其放到合适的头文件中

```cpp
typedef string::size_type sz;
string screen(sz ht = 24, sz wid = 80, char backgroud = ' ');
string window = screen(, , '?');// 错误，只能省略尾部的实参，这里必须要提供ht，wid的实参
```

内联函数：只是提供一个请求，编译器可以忽视这个请求；不要对递归函数进行内联，应该去优化规模小，流程直接，调用频繁的函数。

**C++11** constexpr函数：返回类型与所有形参类型都是字面值类型，而且函数体中有且只有一条return语句</br>
该函数体重可以包含运行时不执行任何操作的语句，比如空语句，类型别名，using声明

```cpp
constexpr int new_sz() { return 42;}
constexpr size_t scale(size_t cnt) { return new_sz() * cnt;}// 正确，返回值不一定是一个常量
constexpr int foo = new_sz();
```

把函数名作为一个指针使用时，该函数自动转换成为指针
```cpp
//以下两句语句是等价的
pf = lengthCompare;
pf = &lengthCompare;
//还可以在指向后直接调用，无需提前解引用指针：以下两者等价
bool b1 = pf("hello","goodbye");
bool b1 = (*pf)("hello","goodbye");
```
