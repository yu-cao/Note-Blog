<h2>用于大型程序的工具</h2>

异常处理

通过抛出一条表达式来引发一个异常；被抛出的表达式类型和调用链共同决定了哪段处理代码该处理这个异常

**执行一个throw时，跟在throw后面的语句不再被执行，控制权交给catch模块，沿着调用链创建的函数会提前退出，沿着调用链创建的对象将会被销毁**

当throw出现在try block中时，检查与该try向关联的catch子句。找到了匹配的catch，就处理这个异常，如果没有找到，就继续检查与外层try匹配的catch子句...这个过程被称为栈展开

在栈展开的过程中，依次退出某些块，编译器会负责将这些块中创建的对象进行正确的销毁，析构函数总是会被执行的，但是函数中负责释放资源的代码却可能被跳过**（使用类来控制资源的分配没有这个问题）**，析构函数不应该抛出异常，如果抛出且自身没有捕获，程序就被终止

异常对象是一种特殊的对象，编译器通过异常抛出表达式对于异常对象进行拷贝初始化；异常对象在编译器管理的空间内，所以无论最终调用哪个catch子句都可以访问到这个空间；处理完毕后，异常对象销毁

抛出表达式的时候，该表达式的静态编译时的类型决定了异常对象的类型

catch子句中的异常声明看似是只包含一个形参的函数形参列表。声明的类型决定了处理代码所就能捕获的异常类型，必须是完全类型，可以是左值引用，不能是右值引用；异常声明的静态类型决定了catch语句所能执行的操作，如果catch接收的异常与某个继承体系有关，最好把这个catch参数定义为引用类型

catch语句的顺序很重要，按照其出现顺序进行逐一匹配的，匹配规则比较严格，只允许非常量向常量转换，派生类向基类转换和数组与指针转换

应该catch子句从最低派生类型到最高派生类型排序，以便派生类处理代码出现在基类类型catch之前

重新抛出：当一个catch不能完整处理某个异常时，可以通过重新抛出的方式将异常传递给另外一个catch语句，只有当传入类型是引用时，才会改变其参数内容

```cpp
catch(my_error &eObj)//引用类型
{
	eObj.status = errCodes::severeErr;//修改了异常对象
	throw;//异常对象的status成员是servereErr
}
catch(other_error eObj)//非引用类型
{
	eObj.status = errCodes::severeErr;//只修改了异常对象的局部副本
	throw;//异常对象的status成员没有改变
}
```

一次性统一捕获所有异常，我们使用`catch(...)`（当与几个catch共同使用时必须放在最后，否则后续的catch将永远不会被匹配）

```cpp
int main()
{
	try{
		//...
	}
	catch(const exception &e)
	{
		cerr << e.what() << endl;
		abort();
	}
	return 0;
}
```

想要处理构造初始值抛出的异常，将构造函数写成函数try语句块的形式（这也是唯一能够处理构造函数初始值异常的方法）

```cpp
template<typename T>
Blob<T>::Blob(initializer_list<T> il) try : data(make_shared<vector<T>>(il))
{
	//空函数体
} catch (const bad_alloc &e)
{
	handle_out_of_memory(e);
}
```

**C++11**提供noexpect说明指定某个函数不会抛出异常，这个声明要么出现在所有声明函数与定义函数中，要么不要出现

`void recoup(int) noexcept;`：不会抛出异常（旧标准：`void recoup(int) throw();`)<br>
`void alloc(int);`：可能抛出异常

一个函数声明了noexcept，但是依然抛出了异常的话，编译器会确保遵守noexcept这个承诺不抛出。

`noexcept`说明符实参常常与`noexcept`运算符混合使用；`noexcept`运算符是一个一元运算符，返回一个bool类型的右值常量表达式，用于表示给定的表达式会否抛出异常

比如上面`void recoup(int)`声明使用了`noexcept`，所以这个表达式为`true`

```cpp
noexcept(recoup(i));//recoup不抛出异常则结果为true，否则结果为false

noexcept(e);//当e调用的所有函数都做了不抛出说明且e本身不含有throw语句，返回为true，否则为false
```

一般可以这两者联用方式如下：

```cpp
void f() noexcept(noexcept(g()));//f将与g的异常说明保持一致
```

异常说明对函数指针与给指针所指的函数的关系必须是一致的，也就是说：函数保证不抛出异常，那么函数指针可以声明为`noexcept`，也可以不声明；但是如果函数没有保证，函数指针就不能声明为`noexcept`。

```cpp
void (*pf1)(int) noexcept = recoup;
void (*pf2)(int) = recoup;//正确，recoup不会抛出异常，pf2可能抛出异常，二者互不干扰
pf1 = alloc;//错误，alloc可以抛出异常，pf1已经声明不会抛出异常
pf2 = alloc;//正确
```

一个虚函数保证了它不抛出异常，那么后续派生继承的虚函数(override)必须做出同样承诺；如果不保证它不抛出异常，那么后续派生的虚函数可以给出保证，也可以不给出（因为给出保证`noexcept`是一个更严格的限定，这是允许的）

<hr>

<h3>命名空间</h3>

大型程序往往会使用多个独立开发的库，而库之间变量名字可能会相互冲突

**始终牢记：每一个命名空间就是一个作用域，命名空间可以不连续，可以被重新打开进行写入，可以在多个文件中打开进行写入重新写入**

```cpp
namespace cpp
{
	class Sales_data
	{
		//...
	};

	Sales_data operator+(const Sales_data &, const Sales_data &);

	class Query
	{
		//...
	};

	class Query_base
	{
		//...
	};
}//与块类似，无需分号
```

**通常情况下不要把`#include`放在名字空间内部**，比如尝试把`<string>`头文件放入自己的名字空间，会导致编译器判定你尝试把命名空间`std`嵌套到自己的命名空间中，引发报错

在命名空间外进行定义必须要加前缀，模板特例化一定要在原始模板所属的命名空间中定义

全局作用域的名字定义在全局命名空间中，以隐式方式声明，也可以通过`::member_name`的方式表示全局命名空间的一个成员

<hr>

**C++11**内联命名空间

通过在关键字`namespace`之前添加关键字`inline`即可（必须出现在命名空间第一次定义的地方），内联空间中的名字可以被外层命名空间直接使用

当应用程序的代码在一次发布和另一次发布之间发生改变时，常常会用到内联命名空间（把现在的版本放在内联空间中，之前的版本放在非内联空间中）

下面的代码，使用test名字空间时，调用now的名字空间的A可以直接用`test::A`，而调用prev的名字空间中的A的话需要`test::prev::A`

```cpp
//prev.h
namespace prev
{
    class A{...}
    class B{...}
    //...
}

//now.h
inline namespace now
{
    class A{...}
    class B{...}
    //...
}

namespace test
{
    #include "now.h"
    #include "prev.h"
}
```

<hr>

**未命名的命名空间：其中的变量拥有静态的生命周期，第一次使用前创建，直到程序结束时销毁，可以在给定文件中不连续，但是不能跨越多个文件，只能在单文件中有效（用来取代C语言的静态全局声明）**

```cpp
int i;//i全局声明 i1
namespace
{
    int i;//未命名命名空间 i2
}
i = 10;//二义性错误
::i = 10;//正确，显式指出使用的是全局作用域的i1

//--------------
namespace
{
    int i;//未命名命名空间 i1
}
i = 10;//正确
::i = 10;//正确

//--------------
int i;//i全局声明
namespace local
{
    namespace
    {
        int i;
    }
}
local::i = 42;//正确
```

<hr>

命名空间名字很长，我们使用命名空间成员就会很艰难，有以下3种方法进行名字的简化

+ 命名空间别名：以关键字`namespace`开始，后面是别名名字 = 原来的命名空间名字; 也可以指向嵌套的命名空间

```cpp
namespace cplusplus_primer{ /*...*/ }
namespace primer = cplusplus_primer;//别名创建
```

+ `using`声明：每次引入命名空间的一个成员
+ `using`指示：将命名空间成员提升到包含命名空间本身+using指示的最近作用域的能力。

```cpp
namespace A
{
    int i, j;
}
void f()
{
    using namespace A;//将A中的名字注入到全局作用域中
    cout << i * j << endl;
}
```

using指示应该尽量避免使用，防止出现名字冲突，一般只有在命名空间本身的实现文件中使用using指示

给函数传递一个类类型对象时，除了在常规的作用域查找外还会查找实参类所属的命名空间；把这个和友元声明结合会有奇特的情况出现：

```cpp
namespace A
{
	class C
	{
		friend void f2();
		friend void f(const C&);
	};
}

int main()
{
	A::C obj;
	f(obj);//正确，查找了实参类所属的命名空间，通过友元声明找到了A::f()
	f2();//错误，A::f2()没有声明
}
```

同样的

```cpp
namespace NS
{
	class C{ /* */ };
	void display(const C&){ /* */ }
}

class A : public NS::C//要指出继承的类所在的命名空间
{
	//...
};

int main()
{
	A obj;
	display(obj);//正确，查找了实参类所属的命名空间及其基类
}
```

**注意：`using`声明只能是声明一个名字，而不是一个特定的函数，如果有重载，就把所有重载版本引入当前作用域，如果出现跟现在作用域中重名且形参列表相同的函数，则发生错误**

```cpp
using NS::print(int);//错误，不能有形参列表
using NS::print;
```

<hr>

<h3>多重继承与虚继承</h3>

多重继承：指从多个直接基类中产生派生类的能力，也就是继承多个（已经被定义过且非final的）基类：

```cpp
class Panda : public Bear, public Endangered { /* */ };
```

C++11允许派生类从它的一个基类或几个基类中继承构造函数，但是如果继承了形参列表完全相同的构造函数，则会发生错误

```cpp
struct Base1
{
	Base1() = default;
	Base1(const std::string&);
	Base1(std::shared_ptr<int>);
};

struct Base2
{
	Base2() = default;
	Base2(const std::string&);
	Base2(int);
};

struct D1 : public Base1, public Base2
{
	using Base1::Base1;
	using Base2::Base2;
};

int main()
{
    D1 obj("hello");//错误，构造函数D1具有二义性
}
```

可以用派生类的指针或引用自动转换成一个可访问基类的指针或引用（与单继承类似），但是编译器不会在派生类和基类中进行比较哪种转换更好：

```cpp
void print(const Bear&);
void print(const Endangered&);

Panda aa("aa");
print(aa);//二义性错误
```

当我们通过`Endangered`指针或引用访问`Panda`对象时，`Panda`接口中`Panda`特有部分与属于`Bear`部分都是不可见的（与单继承类似，单继承通过基类指针/引用访问派生类时，也只能访问只有定义在基类中的那些接口

当一个类拥有多个基类时，可能出现**派生类从两个或更多基类子树中继承了同名成员，调用时会引发二义性错误，如果使用，必须添加前缀限定符**；而且因为**编译器实行先名字检查，后类型检查，所以如果编译器在两个子树中同时发现了相同命名的东西，就直接报告二义性错误**（即使一个在public，一个在private；或者同名函数但是函数形参不同也会报错）

<hr>

虚继承：派生类可以多次继承同一类（菱形继承问题）；**C++要求让某个类做出声明，承诺愿意共享它的基类，共享的基类对象称为虚基类**，虚基类只会影响派生类进一步派生出来的类而不会影响派生类本身

```cpp
class ZooAnimal{};
class Endangered{};

class Raccoon : public virtual ZooAnimal
{/**/};

class Bear : public virtual ZooAnimal//共享了ZooAnimal基类
{/**/};

class Panda : public Bear, public Raccoon, public Endangered {/**/};//某个类指定了虚基类，该类的派生还是按常规进行
```

虚基类成员可见性：如果D1和D2都是虚继承B，B有一个变量x，D继承D1和D2，那么如果D1和D2中都有x的定义，直接在D中访问x将发生二义性错误，解决错误最好的方法是在派生类中为成员自定义一个实例

虚继承与构造函数：虚派生时，虚基类是由最底层的派生类初始化的，比如创造`Panda`类，那么`Panda`的构造函数肚子控制`ZooAnimal`的派生过程（否则如果按照普通规则，`ZooAnimal`会被`Raccoon`和`Bear`两个直接派生类重复初始化），所以我们在创建虚基类的派生对象时，构造函数就必须初始化它的虚基类

```cpp
Bear::Bear(std::string name, bool onExhitbit) : ZooAnimal(name, onExhitbit, "Bear") { }
Raccoon::Raccoon(std::string name, bool onExhitbit) : ZooAnimal(name, onExhitbit, "Raccoon") { }

Panda::Panda(std::string name, bool onExhitbit) 
      : ZooAnimal(name, onExhitbit, "Panda"),//即使不是直接基类也可以初始化
        Bear(name, onExhitbit),
        Raccoon(name, onExhitbit) { }
```

构造顺序：间接虚基类->直接虚基类->非虚基类的间接基类->非虚基类->本身（虚基类永远先于非虚基类构造）

<hr>

小思考：有个需求，汽车(B)过去都是内燃机车(D1)，后来有了电动车(D2)；但是电动车充电慢电池容量小，所以又有了混动车(D)。请问，当我的混动车同时从内燃机车和电动车多重继承后，你会不会自作主张把两个不同的动力基类合并？你要合并了，我这程序还怎么写？我的车上的的确确有两个不同的发动机！但倘若你不合并，菱形继承的二义性就又来了。

思路：基于class的想法下，继承的强耦合与思想包袱太重，而基于接口的思想可以解决这个问题：我们真正应该关心的应该是“对象可以提供什么样的服务（或者说，像XXX一样的服务）”：重要的是接口，而不是继承

声明自己支持某个“协议/接口/prototype”，然后想办法真的去支持这个协议就完了。——至于如何支持呢？你可以自己从头写；但也完全可以在自己的object中隐藏一个支持该协议的、来自系统或第三方的object，然后把相关调用转发给它，即使搞了一万个同样支持这个的接口进去，只要你自己头脑清醒、知道什么时候应该把调用转给这一万个object中的哪一个，它就是完全合法并且井井有条的。也就是说：放弃了自动从父类拿到祖传代码，也就抛弃了因为拿到祖传代码带来的弊端。**策略：优先使用组合而不是继承的想法**

在C++11完美转发的支持下，面向对象其实就是一组实现了特定协议（或者叫接口）的object，当我们处理内燃机动力单元时，就把参数完美转发到内燃机车的接口上，处理电动机车时完美转发到电动机车的接口上去处理就好

<hr>