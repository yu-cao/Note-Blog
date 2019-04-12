<h2>面向对象程序设计</h2>

继承：

虚函数：基类希望它的派生类各自定义适合自己的版本，基类就将这些函数声明为虚函数，任何构造函数之外的非静态函数都可以是虚函数，根据引用或指针所绑定的对象类型不同，可能调用基类的版本，可能调用派生类的版本，即为运行时绑定

派生类必须通过类派生列表明确指出它是从哪个（些）基类继承而来，每个基类前面可以有访问说明符声明是`public`/`protected`/`private`继承，告知从基类继承下来的成员是否对派生类可见；派生类必须在其内部对所有重新定义的虚函数进行声明

`protected`：类成员，友元和派生类这三者有权访问，其他用户禁止访问

**C++11**：`override`关键字：允许派生类显式地注明它使用某个成员函数覆盖了它继承的虚函数，在成员函数第一行末尾（大括号开始前）最后加上该关键字

基类部分与派生类自定义部分不一定连续存储，很多书上写的看上去连续的只是工作机理的概念模型，而不是物理内存布局（C++标准没有规定派生类对象怎么在内存中布局）

**每个类控制它自己的成员初始化过程**，基类部分由派生类通过初始化列表传入实参，由基类自己的构造函数去进行构造

```cpp
class Quote
{
public:
    Quote() = default;
    
    Quote(const string &book, double sales_price) : bookNo(book), price(sales_price)
    {}
    
    string isbn() const
    { return bookNo; }
    
    virtual double net_price(size_t n) const
    { return n * price; };
    
    virtual ~Quote() = default;
    
private:
    string bookNo;
protected:
    double price = 0.0;
};

class Bulk_quote : public Quote
{
public:
    Bulk_quote() = default;
    
    Bulk_quote(const string &book, double p, size_t qty, double disc) :
    Quote(book, p), min_qty(qty), discount(disc)
    {};
    
    double net_price(size_t cnt) const override;
    
private:
    size_t min_qty = 0;
    double discount = 0.0;
};

double Bulk_quote::net_price(size_t cnt) const
{
    if (cnt >= min_qty)
        return cnt * (1 - discount) * price;
    else
        return cnt * price;
}

```

派生类可以访问基类的`public`成员和`protected`成员，必须遵循基类的接口，即使这个对象是派生类的基类部分，所以务必需要使用基类自己的构造函数进行构造

如果在基类中定义了一个静态成员，则整个继承体系中将只存在该成员的唯一定义（即每个静态成员只有唯一的实例）

派生类的声明不应该包含它的派生列表，而只需要类名

**C++11**：我们不希望其他类对该类进行继承，在该类名后加上关键字`final`：

```cpp
class NoDerived final { /* */ };//NoDerived不能作为基类
class Last final : Base{ /* */ };//Last不能作为基类
```

**我们可以把基类的指针或引用绑定到派生类对象上**，这规则既支持内置指针，也支持智能指针

不存在从基类向派生类的隐式类型转换，因为一个基类对象可以是派生类的一部分，也可能不是；<br>而派生类可以向基类进行隐式类型转换，因为派生类必然包含一个基类部分

```cpp
Quote base;
Bulk_quote* bulkP = &base;//错误：不能把基类转换成派生类
Bulk_quote& bulkRef = base;//错误：不能把基类转换成派生类

Bulk_quote bulkBase;
Quote* quoteP = &bulkBase;//正确，派生类可以向基类进行转换
Quote& quoteRef = bulkBase;//正确，派生类可以向基类进行转换

//即使一个基类指针/引用指向派生类，也不能执行从基类向派生类的转换，因为编译器静态检查无法通过
Bulk_quote bulk;
Quote *itemP = &bulk;//正确，动态类型是Bulk_quote
Bulk_quote *bulkP = itemP;//错误：不能把基类转换成派生类
Bulk_quote *bulkP = dynamic_cast<Bulk_quote*>(itemP);//如果确认安全，可以使用动态转换覆盖编译器检查
```

当派生类对象为基类对象进行初始化或赋值时会出现**剪切**，以下代码中，构造item时运行Quote的拷贝构造函数，只负责拷贝bulk的Quote部分，其他部分将会直接忽略，赋值操作同理

```cpp
Bulk_quote bulk;//派生类对象
Quote item(bulk);//调用Quote::Quote(const Quote&)构造函数
item = bulk;//调用Quote::operator=(const Qutoe&)
```

虚函数

不管它是否被用到，每个虚函数都必须定义，因为编译器无法确定到底会使用哪个虚函数；一旦某个函数被声明为虚函数，则它的所有派生类中都是虚函数；**如果派生类函数覆盖了某个继承而来的虚函数，则它的形参必须与被覆盖的基类函数完全一致，返回类型也要与基类保持一致，除非是返回类本身的指针/引用**

多态性：具有继承关系的多个类型称为多态类型，因为我们能只用这些类型的“多种形式”而无需在意它们的差异。静态类型与动态类型在通过引用/指针调用虚函数时可以不一致

但是如果在派生类中定义了函数与基类的虚函数参数列表不同，一样合法，但是编译器会认为这是与原有的虚函数相互独立的（这很多时候与我们编程习惯不符，是我们自己的错误导致的，而且难以调试）

**C++11**：使用`override`标记，如果函数没有覆盖已标记的虚函数，将会报错

```cpp
struct B
{
    virtual void f1(int) const;
    virtual void f2();
    void f3();
};

struct D1 : B
{
    void f1(int) const override;//正确，f1与基类中f1匹配
    //也可以写尾置返回类型
    auto f1(int)const->void override;

    void f2(int) override;//错误，B没有形如f2(int)的函数
    void f3() override;//错误，B中f3不是虚函数
    void f4() override;//错误，B中没有命名为f4的虚函数
};

struct D2 : B
{
    //从B中继承f2()和f3()，覆盖f1(int)
    void f1(int) const final;//不允许后续的其他类进行覆盖f1
    
    //也可以写尾置返回类型
    auto f1(int)const->void final;
};

struct D3 : D2
{
	void f2();//正确，覆盖从间接基类B中继承而来的f2
	void f1(int) const;//错误，D2已经将f2声明成final
};
```

虚函数可以有默认实参，实参值由本次调用的静态类型决定，但是这种情况下基类和派生类中定义的默认实参最好要保持一致

回避虚函数：当我们对虚函数不要进行动态绑定而想要只执行某一版本时：使用作用域运算符

```cpp
double undiscounted = baseP->Quote::net_price(42);//强行调用基类定义的函数版本，无视baseP的动态类型
```

抽象基类：

含有（或未经过覆盖直接继承）纯虚函数的类是抽象基类，负责定义接口，后续的其他类可以覆盖该接口；我们不能直接创建一个抽象基类的实例，但是可以创建其派生类的实例

纯虚函数：无需定义，在声明语句的分号前书写=0即可将一个虚函数说明为纯虚函数

```cpp
//表示一本打折书籍的通用概念，而不是某种具体的折扣策略
class Disc_quote : public Quote
{
public:
    Disc_quote() = default;
    
    Disc_quote(const string &book = "", double price = 0.0, size_t qty = 0, double disc = 0.0) :
    Quote(book, price), quantity(qty), discount(disc)
    {}
    
    double net_price(size_t) const = 0;//纯虚函数
    
protected:
    size_t quantity = 0;
    double discount = 0.0;
};

class Bulk_quote : public Disc_quote
{
public:
    Bulk_quote() = default;
    
    Bulk_quote(const string &book = "", double sales_price = 0.0, size_t qty = 0, double disc_rate = 0) :
    Disc_quote(book, sales_price, qty, disc_rate)
    {}
    
    double net_price(size_t cnt) const override
    {
        if (cnt > quantity)
            return cnt * (1 - discount) * price;
        else
            return cnt * price;
    }
};
```

protected保护：受保护的成员对于派生类的成员和友元可以访问；但是这种访问只能通过派生类的对象才能访问到基类受保护的成员（否则只要定义一个形如下面Sneaky的新类就能绕过protected保护，显然是不合理的）

```cpp
class Base{
protected:
    int prot_mem;
};

class Sneaky : public Base{
    //仔细区别以下两种访问
    friend void clobber(Sneaky &s)
    {
        s.j = s.prot_mem = 0;//正确，通过派生类对象进行的访问
    }
    friend void clobber(Base &b)
    {
        b.prot_mem = 0;//错误，只能通过派生类的对象来访问基类protected成员
    }
    int j;
};
```

派生类说明符：控制派生类用户（包括派生类的派生类在内的实例）对于基类用户的访问权限，**记忆法：三种继承就是把基类变成派生类的哪个部分，`public`继承就是把基类变成派生类的`public`成员，`private`继承就是把基类变成派生类的`private`成员，以此类推**

友元关系不可传递，不可继承

```cpp
class Base{
public:
    void pub_mem();
protected:
    int prot_mem;
private:
    char priv_mem;
};

struct Pub_Derv : public Base
{
    int f() {return prot_mem;}
    char g() {return priv_mem;}//基类private对派生类不可访问
};

struct Priv_Derv : private Base
{
    int f1() const {return prot_mem;}
};

Pub_Derv d1;
Priv_Derv d2;
d1.pub_mem();//正确，因为pub_mem在派生类中是public
d2.pub_mem();//错误，因为pub_mem在派生类中是private

struct Derived_from_Public : public Pub_Derv
{
    int use_base(){return prot_mem;}
};

struct Derived_from_Private : public Priv_Derv
{
    int use_base(){return prot_mem;}//错误，Base::prot_mem在Priv_Derv中是private
};
```

using声明可以改变访问级别（**派生类只能为那些它能够访问到的名字提供using声明**，不然同样会出现权限问题）

```cpp
class Base{
public:
    size_t size() const {return n; }
protected:
    size_t n;
}

class Derived : private Base{//这里是private继承
public:
    using Base::size;//改变后，Derived用户可以使用size成员
protected:
    using Base::n;//Derived派生类能使用n
}
```

派生类将会隐藏基类的同名变量/成员，即使成员的形参列表不一致（名字查找优先于类型检查）但是我们可以通过作用域运算符使用一个被隐藏的变量/成员；所以，**除了覆盖继承而来的虚函数之外，派生类最好不要重用其他定义在基类中的名字**

函数调用的名字查找，如`p->mem()`，分为四个阶段（始终牢记：名字优先，再看类型）

+ 确定p的静态类型
+ 在p的静态类型中查找mem，如果找不到，依次在直接基类中继续查找，直至到达继承链顶端
+ 找到后，进行常规的类型检查，确认调用合法性
+ 如果调用合法，根据是否是虚函数产生不同代码
  + 如果mem是虚函数而且是我们通过引用/指针进行的调用，则在运行时确定函数入口
  + 如果mem不是虚函数或我们是通过对象实例进行的调用，编译器会产生一个常规的函数调用

虚析构函数

因为当delete指针指向继承体系的某一个类型，可能出现指针的静态类型与被删除对象的动态模型不匹配的情况，比如一个指向基类的指针，该指针可能已经指向了其派生类对象，编译器就需要正确调用派生类的析构函数才能够保证析构的安全性（如果基类的析构函数非虚，则delete一个指向派生类的基类指针是Ub）；一般在没有定义指针类成员的情况下使用默认版本即可，有指针类的情况下需要对指针成员进行适当清除

定义派生类的拷贝或移动构造函数时要在构造列表中显式调用

```cpp
class Base{ /* */};
class D : public Base{
public:
    D(const D& d): Base(d){/* */}//要在构造列表中显式调用该构造函数
    D(D&& d): Base(std::move(d)){/* */}
    
    D(const D& d){/* */}//基类部分是默认初始化，而不是拷贝，可能与预期不符
};
```

同理，赋值运算符也要显式调用

```cpp
D &D::operator=(const D &rhs)
{
    Base::operator=(rhs);//为基类部分赋值
    return *this;
}
```

派生类析构函数只需要析构派生类自己分配的资源即可，对基类的析构会被自动执行

容器与继承：当我们希望在容器中存放具有继承关系的对象的时候，实际应该存放的通常是基类的指针/智能指针
