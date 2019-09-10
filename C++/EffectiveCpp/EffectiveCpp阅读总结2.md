## Effective C++ 阅读总结2

<hr>

### 1、了解C++默默编写并调用了哪些函数

构造函数，拷贝函数，析构函数，复制操作符

```cpp
class Empty{ };
//等效于：
class Empty{
public:
	Empty(){...}
  Empty(const Empty& rhs){...}
  ~Empty(){...}
  
  Empty& operator=(const Empty& rhs) {...}
};
```

编译器给出的析构函数是non-virtual的，除非这个类的基类自身声明有virtual的析构函数

copy构造函数与copy assignment操作符，编译器就是将来源的每一个成员对象拷贝到目标对象里面（bitwise）

<hr>

### 2、如果你不想使用编译器自动生成的函数，请明确拒绝

将拷贝构造函数和copy assignment操作符的**声明**放到private中来明确拒绝或者继承一个不允许拷贝构造和copy assignment操作符的类（在C++11中可以直接使用`=delete`来拒绝，更加直观）

<hr>

### 3、为多态基类声明`virtual`析构函数

```cpp
class TimeKeeper {
public:
	TimeKeeper();
	~TimeKeeper();//错误，在多态下无法析构派生类
  virtual ~TimeKeeper();//OK
};

class AtomicClock : public TimeKeeper{};
class WaterClock : public TimeKeeper{};
class WristWatch : public TimeKeeper{};
```

这时，客户很可能不在乎时间怎么算的，而是只想在程序中使用时间

这时候，我们可以设计出factory函数，返回指针指向一个计时对象，通过指向基类的指针来处理多态的派生类，而这里的析构函数显然只有是virtual的才可以完成正确的析构调用（**对带有多态性质的base class，析构函数必须要virtual**）

```cpp
TimeKeeper* getTimeKeeper();
  
//具体使用：
TimeKeeper* ptk = getTimeKeeper();
  ...//运用它
delete ptk;//资源回收
```

也就是说，当class不含有virtual函数，一般也就代表着不应该拿它作为一个base class，**对不想作为base class的类的析构函数进行virtual，是不妥当的！**

想要实现出virtual函数，对象必须携带信息来判定在running time时哪个virtual函数会被调用。这份信息通常由虚函数表指针vptr给出，这个指针指向一个由函数指针构成的数组，称为虚函数表vtbl。调用时，编译器将会寻找这个实例的vptr指向的vtbl，并且选择合适的函数指针。

这里，我们丧失了C++的类C特性（C中没有vptr和vtbl），导致移植性下降；增加了每个实例的体积（白白多了一个指针）。因此，不应该无脑将所有析构函数设定为virtual

只有当class内含有至少一个virtual函数时，才为它声明virtual析构函数。

要小心STL里面的容器，这里想当然地使用将会导致一些危险的后果：比如在标准的string中不含有任何virtual函数，但是有时候程序员会想当然地把它作为base class：

```cpp
class SpecialString : public std::string{
  ...
};//bad idea!
```

只要你在后面将一个指针从`SpecialString*`变成`std::string*`时，然后进行delete，立刻内存就泄漏了

```cpp
SpecialString* pss = new SpecialString("Impending Doom");
std::string* ps;
...
ps = pss;//SpecialString* => std::string*
...
delete ps;//泄漏
```

这些情况也存在于所有不带virtual的STL容器中，有`vector`，`list`，`set`，`unordered_map`等等。（不像Java的`final`和C#的`seal`可以控制禁止派生）

如果你想要控制某个类成为抽象类，就需要类中至少有一个纯虚函数，这时候把析构函数设定为纯虚函数就是一个很好的方式

```cpp
class AWOV{
public:
  virtual ~AWOV() = 0;
}

AMOV::~AMOV(){ }//务必提供一个定义，不然将会在层级析构中这个底层抽象类因为没有定义只有声明导致链接错误
```

<hr>

### 4、避免异常离开析构函数

```cpp
class Widget{
public:
  ...
	~Widget(){...}
};

void doSomething()
{
	std::vector<Widget> v;
  ...
}
```

当销毁vector v时，它有责任回收其中所有Widget的内存等。当析构第一个Widget时，抛出了一个异常。其他的9个Widget应该依然要正常销毁，也就是依然调用对应的析构函数。但是，结果第二个又抛出了异常，那么这时对于C++来说，同时两个异常要么程序终止，要么就是UB。

```cpp
class DBConnection{
public:
  ...
  static DBConnection create();
  void close();
}

class DBConn{
public:
  ...
  ~DBConn()//确保数据库连接在结束时总是关闭
  {
    db.close();
  }
private:
  DBConnection db;
}

//也就允许这样的代码：
DBConn dbc(DBConnection::create());
...
```

这里看上去没啥问题，但是一旦close调用导致异常，DBConn的析构函数将会放任这个异常离开析构函数，这会造成麻烦。这时要进行及时catch或者直接终止程序，而不应该放任这种未定义行为。

此外，让客户来对于某个操作函数引发的异常进行反应，使用一个普通函数来执行操作，不要放到析构函数中

<hr>

### 5、绝不在构造和析构过程中调用`virtual`函数

```cpp
class Transaction
{
public:
	Transaction();

	virtual void logTransaction() const = 0;
};

Transaction::Transaction()
{
	logTransaction();
}

class BuyTransaction : public Transaction
{
public:
	virtual void logTransaction() const;
};

class SellTransaction : public Transaction
{
public:
	virtual void logTransaction() const;
};

//首先，这里会有BuyTransaction构造函数调用，但是base class的构造函数一定会更早调用
//构造函数会进行logTransaction，这个调用的是Transaction自己的版本！
//换句话说：在base class构造阶段，virtual函数不是virtual的！
	BuyTransaction b;//幸好这里是纯虚函数，程序将会直接终止
}
```

同样的，析构函数也存在这个情况，当析构开始后，派生类成员变量就是未定义的了，进入去析构base class时也是只认定当前类是base class，而不再认可派生类。

<hr>

### 6、令`operator=`返回一个ref to *this

来实现连续赋值等操作`Widget& operator=(const Widget& rhs){... return *this;}`（也包括`+=`等等赋值操作）

<hr>

### 7、在`operator=`中处理“自我赋值”

```cpp
class Bitmap {...};
class Widget {
  ...
private:
  Bitmap* pb;
};
Widget& Widget::operator=(const Widget& rhs)//不安全
{
  delete pb;
  pb = new Bitmap(*rhs.pb);
  return *this;
}
```

上面代码存在安全隐患，也就是`rhs`如果与`*this`就是同一个对象，第一步delete就干掉了`rhs`，第二步`*rhs`的pb就已经没了，这里就会出现严重的问题。

传统做法是进行“证同测试”，也就是在delete前加上一句：`if (this == &rhs) return *this;`。但是，还是在异常方面存在风险，比如new失败或者copy构造出现问题，还是会返回一个持有者被删除的Bitmap的Widget。但是这里你可以精心调整一下逻辑，得到一个安全的代码：

```cpp
Widget& Widget::operator=(const Widget& rhs)
{
  Bitmap* pOrig = pb;
  pb = new Bitmap(*rhs.pb);
  delete pOrig;
  return *this;
}
```

这里如果new引发异常，pb就会保持原状，同时观察发现，自我赋值也是可以的。当然还有一些方案，比如copy and swap：

```cpp
class Widget {
  ...
  void swap(Widget& rhs);//交换*this与rhs数据
};

Widget& Widget::operator=(const Widget& rhs)
{
  Widget temp(rhs);//制作一个副本
  swap(temp);//*this与副本数据进行交换
  return *this;
}
```

<hr>

### 8、复制对象时不要忘记其每一个成分

尤其要注意复制其base class成分，而那些数据往往是private，需要在派生类的copy函数中调用相应的base class函数

不要尝试用copy assignment操作符调用copy构造函数（就像：试图构造一个已经存在的对象），也不要用copy构造函数调用copy assignment操作符。当你发现两者代码很多很相近，可以复用时，建立一个新的private函数给两者调用即可，不要自作聪明！