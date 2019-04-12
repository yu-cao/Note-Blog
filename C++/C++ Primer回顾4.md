**C++11**：如果我们需要默认的构造行为，可以通过在参数列表后面加上`= default`来要求编译器生成构造函数。如果作为定义在类的内部，那么默认构造函数是内联的，定义在类的外部默认情况下是不内联的

```cpp
struct Sales_data
{
    Sales_data() = default;// 默认构造函数
    Sales_data(const string &s) : bookNo(s)
    {}

private:
    string bookNo;
};
```

友元的声明最好在类的开始或者结束处集中声明，友元只是声明了访问权限，并不是通常意义的函数声明，所以类的用户要调用某个友元函数，必须要在友元声明外再专门对函数进行一次声明

可变数据成员(mutable data member)：能修改该成员，即使是在一个const成员函数里。在变量声明中加入mutable关键字即可。'mutable' and 'const' cannot be mixed，即可变数据成员永远不会是const。

```cpp
class Screen{
public:
    void some_member() const;

private:
    mutable size_t access_ctr;
};

void Screen::some_member() const
{
    ++access_ctr;
}
```

基于const的重载：区分成员函数是否是const可以进行重载（因为非常量版本的函数对常量对象不可用，而非常量对象尽管两周都可用，但是非常量->非常量是best match，所以成功避免二义性）

```cpp
class Screen{
public:
    Screen &display(ostream &os)
    {
        do_display(os);
        return *this;
    }
    const Screen &display(ostream &os) const
    {
        do_display(os);
        return *this;
    }

private:
    string content;
    void do_display(ostream &os) const 
    {
        os << content;
    }
};

Screen myScreen(5,3);
const Screen blank(5, 3);
myScreen.set('#').display(cout);// 调用非常量版本
blank.display(cout);// 调用常量版本
```

友元类：一个类指定了友元类，友元类的成员函数可以访问此类包括非公有成员在内的所有成员。但是注意，友元关系没有传递性，我朋友的朋友不是我的朋友。此外，除了令整个类作为友元，也可以选择某个类的成员函数提供访问权限。

**C++11**：委托构造函数。一个构造函数委托给其他构造函数后，不能再进行成员列表初始化，而只能在函数体内进行初始化其他成员变量。(否则会静态检查错误：An initializer for a delegating constructor must appear alone)作用：可以减少重复代码

```cpp
class cook
{
public:
    cook(const char *rice, int amount) : rice(rice), amount(amount)
    {
        cout << "taomi" << endl;
    }// 淘米
    cook(const char *rice, int amount, const char *vegetable1) : cook(rice, amount)
    {
        vegetable = vegetable1;
        cout << "xicai" << endl;
    }// 淘米+洗菜（重用了淘米的构造函数）
    ~cook(){}
private:
    const char *rice;
    int amount;
    const char *vegetable;
};

int main()
{
    const char *rice = "china rice";
    const char *veg = "apple";
    cook c(rice, 2, veg);
}

// Output:
taomi
xicai
```

**C++11**：constexpr构造函数：必须初始化所有数据成员，初始值或使用constexpr构造函数，或者是一条常量表达式（回忆：constexpr函数只能包含return语句，不允许执行其他任务）

```cpp
class Debug//字面值常量类
{
public:
    constexpr Debug(bool b = true) : hw(b), io(b), other(b)
    {}

    constexpr Debug(bool h, bool i, bool o) : hw(h), io(i), other(o)
    {}

    constexpr bool any()
    { return hw || io || other; }

    void set_io(bool b)
    { io = b; }
    
    void set_hw(bool b)
    { hw = b; }

    void set_other(bool b)
    { other = b; }

private:
    bool hw;
    bool io;
    bool other;
};
```

静态类成员：该成员与类的本身相关，而不是与每个对象实例相关，对象中也不包含任何与静态数据成员有关的数据。static关键字只是用来在类内进行声明，在外部定义时，不能重复static关键字

```cpp
void Account::rate(double newRate)
{
	interestRate = newRate;
}
```

静态成员原则上不应该在类内进行初始化（除非静态成员是constexpr类型而且提供的值是const的类内初始值，例如在类内`static constexpr int period = 30;`），而必须在类的外部定义和初始化每个静态成员（与其他的类似，定义有且仅能有一次）。

```cpp
double Account::interestRate = initRate();//Accont::之后作用域就在类内了，即使initRate()是Account私有函数也是可以的
```