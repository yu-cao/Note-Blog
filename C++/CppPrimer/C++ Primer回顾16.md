<h2>控制内存分配</h2>

重载new和delete

new的运行机理：

+ 调用一个`operator new`或`operator new[]`的标准库函数，分配一个足够大的原始未命名内存
+ 运行构造函数进行构造吗，传入初始值
+ 对象被分配了空间并且构造完成，返回指向对象的指针

delete的运行机理：

+ 对指向的对象进行析构函数
+ 调用`operator delete`或`operator delete[]`的标准库函数释放内存

我们可以自己定义与重载`operator new/new[]/delete/delete[]`，手动接管内存分配；与析构函数一样，`delete`不允许抛出异常：`void *operator delete(void *) noexcept;`

但是绝对不允许重载这个函数，这个形式只供标准库使用：

```cpp
void *operator new(size_t, void *);
```

一种简单方式：

```cpp
void *operator new(size_t size)
{
	if (void *mem = malloc(size))
		return mem;
	else
		throw bad_alloc();
}

void operator delete(void *mem) noexcept
{
	free(mem);
}
```

<hr>

<h3>运行时类型识别（RTTI）</h3>

+ typeid运算符：返回表达式类型
+ dynamic_cast运算符，将基类指针/引用安全转换为派生类的指针/引用

这两个运算符特别适合我们想使用基类对象的指针执行某个派生类操作而且这个操作不是虚函数，但是使用RTTI运算符蕴含更多的风险，需要加倍小心（最好使用虚函数而不是这种方式）

<hr>

<h4>dynamic_cast</h4>

```cpp
class A
{
public:
	virtual void print();
};

class B : public A
{
public:
	virtual void print();
	void pp();
};

int main()
{
	A* p = new B;
	p->pp();//错误，A中没有pp的函数
	dynamic_cast<B*>(p)->pp();//正确
}
```

dynamic_cast运算符：要求e是type的公有派生类，公有基类或就是其本身类型（或者是nullptr）才能转换，如果转换指针失败，`return 0`（也就是空指针）；转换引用失败，`throw bad_cast`（因为没有空引用，所以抛出异常）

```cpp
dynamic_cast<type *>(e)//e必须是有效的指针
dynamic_cast<type &>(e)//e必须是一个左值
dynamic_cast<type &&>(e)//e不能是左值
```

尽量确保类型转换和结果检查在同一个表达式中完成：

```cpp
if (Derived *dp = dynamic_cast<Derived*>(bp)) { /* 使用dp，而且dp在外部失效，安全 */ }
else { /* 使用bp */ }

void f(const Base &b)
{
    try
    {
        const Derived &d = dynamic_cast<const Derived&>(b);
        //...使用b引用的Derived对象
    } catch (bad_cast) {
        //...处理异常
    }
}
```

<hr>

<h4>typeid运算符</h4>

表达式形式为`typeid(e)`，e是任意表达式/类型的名字，返回对常量对象的引用，对象是标准库`type_info`或者其公有派生类型，这个过程中顶层const会被忽略，引用会返回所引用对象类型，**数组与指针的隐式转换不会执行(`typeid(char[]) == typeid(char*)`恒不成立)**

```cpp
Derived *dp = new Derived;
Base *bp = dp;
if (typeid(*bp) == typeid(*dp)){
    //...bp与dp指向同一类型对象
}
if (typeid(*bp) == typeid(Derived)) {
    //...bp实际指向Derived对象
}
```

<hr>

比如当我们要为具有继承关系的类实现`==`运算符时，RTTI就十分有用

```cpp
class Base {
	friend bool operator==(const Base&, const Base&);
public:
	//接口...
protected:
	virtual bool equal(const Base&) const;
};

class Derived : public Base {
public:
	//接口...
protected:
	virtual bool equal(const Base&)const;
};

bool operator==(const Base &lhs, const Base &rhs)
{
	return typeid(lhs) == typeid(rhs) && lhs.equal(rhs);
}

bool Derived::equal(const Base &rhs) const
{
	auto r = dynamic_cast<const Derived&>(rhs);
}

bool Base::equal(const Base &) const
{
	//执行比较两个base对象的操作
}
```

<hr>

<h3>枚举类型</h3>

C++11：限定作用域的枚举类型

```cpp
enum class Color { red, green = 20, blue };
Color r = Color::blue;//想要初始化一个enum对象，必须使用该enum类型的另一个对象或它的一个枚举成员，即使整型与其枚举成员值相同也不能初始化
switch(r)
{
    case Color::red  : std::cout << "red\n";   break;
    case Color::green: std::cout << "green\n"; break;
    case Color::blue : std::cout << "blue\n";  break;
}
// int n = r; // 错误：无有作用域枚举到 int 的转换
int n = static_cast<int>(r); // OK, n = 21
```

不限定作用域的枚举类型（省略了class/struct）：`enum color {red, green, blue};`

枚举值从0开始，依次是前一个枚举值加1

C++11：可以通过指定enum的中值的大小来避免枚举值越界，默认是int

```cpp
enum intValues : unsigned long long
{
	charTyp = 255,
	shortTyp = 65535,
	intTyp = 65535,
	longTyp = 4294967295UL,
	long_longTyp = 18446744073709551615ULL
};
```

C++11：前置声明，必须同时指出其成员的大小（即使是隐式也要写出是int）

```cpp
enum intValue : unsigned long long;//不限定作用域的，必须要指定成员类型
enum class open_mode;//限定作用的枚举类型可以使用默认的类型int
```

<hr>

<h3>类成员指针</h3>

指向类的非静态成员的指针

<h4>数据成员指针</h4>

相比较普通指针还需要包含成员所属的类

```cpp
const string Screen::*pdata;//一个指向Screen类的const string成员的指针
pdata = &Screen::contents;//指向某个非特定Screen对象的contents成员

//上面两句也可以合并为
auto pdata = &Screen::contents;
```

要注意：我们没有指向任何数据（因为这个指向并没有实例化），只有当绑定到一个实例之后，解引用成员指针时才能够提供对象的信息

```cpp
Screen myScreen, *pScreen = &myScreen;

auto s = myScreen.*pdata;//解引用pdata以得到myScreen对象的contents成员
s = pScreen->*pdata;
```

因为数据成员常常是private的，我们不能直接获得数据成员的指针。想要访问它的context成员，最好定义一个函数，使其返回值是指向该成员的指针

```cpp
class Screen {
public:
    //静态成员，返回一个成员指针
    static const std::string Screen::*data() { return &Screen::contents; }
    //其他成员保持原来一致
};

const string Screen::*pdata = Screen::data();
```

<hr>

<h4>成员函数指针</h4>

```cpp
auto pmf = &Screen::get_cursor;
```

如果有重载问题，必须显式声明函数类型以明确我们到底要哪个函数

```cpp
using Action = char (Screen::*) (Screen::pos, Screen::pos) const;//类型别名创建
Action pmf = &Screen::get;
```

而且与普通函数指针不同，这里必须要使用`&`运算符，不存在自动转换；可以使用类型别名增加可读性

下面的括号必不可少，因为函数调用优先级高

```cpp
Screen myScreen, *pScreen = &myScreen;

char c1 = (pScreen->*pmf)(0, 0);
```

成员函数用作可调用对象

使用标准库模板function

```cpp
function<bool (const string&)> fcn = &string::empty;
find_if(svec.begin(), svec.end(), fcn);

vector<string*> pvec;
function<bool (const string*)> fp = &string::empty;
find_if(pvec.begin(),pvec.end(),fp);
```

后面一些关于嵌套类，union，局部类的部分略。

<hr>

<h3>不可移植特性</h3>

位域：类可以将其非静态数据成员定义成位域，每个位域中含有一定数量的二进制位，通常用于程序为其他程序或硬件设备传递二进制数据时使用

类型必须是整型或枚举类型（符号位不同机器实现方式可能不同），取地址运算符`&`不可使用，因此任何指针都不能指向类的位域

```cpp
typedef unsigned int Bit;//最好要设为无符号类型
class File
{
	Bit mode : 2;//mode占2位，以此类推
	Bit modified : 1;
	Bit prot_owner : 3;
	Bit prot_group : 3;
	Bit prot_world : 3;
	//...
public:
	//文件以八进制方式表示
	enum modes{ READ = 01, WRITE = 02, EXECUTE = 03};
	File &open(modes);
	void close();
	void write();
	bool isRead() const;
	void setWrite();
};
```

访问位域的方式就跟一般访问类的方式很相似，通常使用位运算符操作超过1位的位域

<hr>

`volatile`限定符

值由程序直接控制之外的过程控制，如系统时钟等，对这种对象应该申明为`volatile`，告诉编译器不要优化这样的对象，用法与`const`比较类似，对类进行额外的修饰

此外，合成的拷贝/移动/赋值对`volatile`无效，必须要自定义拷贝或移动操作（把形参设定为const volatile引用）

<hr>

链接指示

调用其他语言编写的函数。前提：必须有权访问该语言的编译器，并且这个编译器与当前C++编译器兼容

```cpp
extern "C" size_t strlen(const chat *);//单行链接指示

extern "C" {//多行链接指示
	#include <stdlib.h>//多重声明的形式可以应用于整个头文件
	
	int strcmp(const char *, const char *);
	char *strcat(char *, const char *)
}
```

指向其他语言编写的函数的指针需要与函数本身使用同一的链接指示，而且指向C函数的指针与指向C++函数的指针是不一样的类型，比如下面的赋值操作是不可以的（虽然有些编译器可能会提供拓展优化）：

```cpp
extern "C" void (*pf) (int);

void (*pf1) (int);

pf1 = pf;//错误，类型不同
```

使用链接指示时，不仅对函数有效，还对 作为返回类型与形参类型的 函数指针也有效，所以如果我们想要给C++函数传入一个C函数指针，必须要用类型别名

```cpp
extern "C" void f1(void(*)(int));//传入的函数指针也必须是一个C函数的名字或指向C函数的指针

extern "C" typedef void FC(int);//FC是一个指向C函数的指针
void f2(FC *);//f2是一个C++的函数，形参是指向C函数的指针
```

导出C++到其他语言：使用链接指示对函数进行定义

```cpp
extern "C" double calc(double dparm) { /* */ }//calc函数可以被C程序调用
```

但是多种语言共享的函数返回类型和形参类型限制很多，比如C将无法理解C++中构造，析构等其他操作，要额外小心

链接指示与重载函数的相互作用依赖于目标语言，比如在C语言中就不能使用重载等操作