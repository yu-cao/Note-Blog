##Effective C++ 阅读总结4

<hr>

### 1、让接口更易被正确使用而不易被误用

目标：调用者企图使用某个接口，但是没有获得预期的结果，那么就不应该通过编译；如果通过编译，那么作用一定是调用者想要的。

尽量保证接口的一致性，要与内置类型的行为兼容或逻辑类似。

阻止误用的办法包括建立新的类型，限制类型上的操作，束缚对象值，消除调用者的资源管理责任等

<hr>

### 2、设计class如同设计type

要考虑：新的type对象怎么创建，怎么销毁？初始化与赋值有什么区别？新的type对象pass-by-value将会意味着什么？这个type的合法值是什么？是否要配合某个继承图？需要支持转换吗？什么成员函数是合理的，什么成员函数是应该禁止的？真  的需要一个新的type吗？...

<hr>

### 3、最好用pass-by-reference-to-const来替换pass-by-value

往往可以实现开销降低，同时避免对象切割问题（当派生类对象以值传递并被函数视为基类对象时，会调用基类的拷贝构造函数，派生类的那些特化性质全部丢失）

如果是内置类型，pass-by-value效率更高；但是不代表对象小，copy构造函数就不昂贵，很多copy构造函数需要复制其中指针指向的内含物，那这个开销就是巨大的。

<hr>

### 4、必须返回对象时，不要尝试去返回一个引用

绝对不应该返回一个引用或者指针指向一个local stack对象，或者返回reference指向一个heap-allocated对象，或者使用静态对象等！必须老老实实返回回来就好了！永远考虑行为正确的那个，其他的交给编译器就好！

<hr>

### 5、将成员变量声明为private

protected成员变量的封装性并不高过public成员变量，因为它会导致其所有的使用它的派生类被破坏，这与public导致所有调用它的代码被破坏一样不可预知大量。

其实只有2中访问权限：private（提供封装）和其他（不提供封装）

<hr>

### 6、最好用non-member、non-friend替换member函数

```cpp
class WebBrowser{
public:
  ...
  void clearCache();
  void clearHistory();
  void removeCookies();
  ...
    
  void clearEverything();//方案1：使用member函数处理
};

//方案2：外部函数，调用适当的member函数
void clearBrowser(WebBrowser& wb)
{
  wb.clearCache();
  wb.clearHistory();
  wb.removeCookies();
}
```

方案2更好！使用member函数带来的封装性比非成员函数clearBrowser低。而且方案2带来的包裹弹性更好。通常情况下，比较自然的做法是让clearBrowser成为一个non-member函数并且位于WebBrowser所在的同一个namespace下。

越多的函数可以访问它，则数据的封装性越低（极端的就是public成员变量，所有函数都可访问）。能访问private数据的只有成员函数+友元函数。当可以选择使用非成员函数组合来实现效果时，就不要再实现一个成员函数，这样就保证了封装性。

<hr>

### 7、若所有参数都需要类型转换，请为此声明一个non-member函数

```cpp
class Rational{
public:
	Rational(int numerator = 0, int denominator = 1);
	int numerator() const;
	int denominator() const;
	const Rational operator* (const Rational& rhs) const;
};

Rational oneForth(1,4);
Rational result;
result = oneForth * 2;//正确，2被隐式转换成为Rational类型对象
result = 2 * oneForth;//错误！因为int的乘法无法把Rational隐式转换
```

如果上面的Rational构造函数是explicit，那么两个乘法都会是错误的（这时无法进行隐式转换）

如果还是想要支持混合式算数运算，方法就是：使用非成员函数的重载乘法：

```cpp
class Rational{
  ...
};

const Rational operator*(const Rational& lhs, const Rational& rhs)
{
  return Rational(lhs.numerator() * rhs.numerator(), lhs.denominator() * rhs.denominator());
}
```

接下来，问：这个重载是否应该成为Rational class 的一个友元函数？并不是！

不该成为member代表的是应该成为non-member，而不是friend！不能够只因函数不该成为member，就自动让它成为friend。**无论何时，尽量避免friend函数**

<hr>

### 8、考虑支持一个不抛出异常的swap

```cpp
namespace std
{
	template<typename T>
	void swap(T &a, T &b)
	{
		T temp(a);
		a = b;
		b = temp;
	}
}
```

只要T支持copy（通过拷贝构造函数与拷贝运算符完成），这个swap函数就是成立的。但是，这里带来的可是复制的巨大开销，假设我们实际上的交换只是需要交换指针值即可，这里就会额外产生其他的对象，还进行了复制，效率非常低：

```cpp
class WidgetImp1
{
public:
	//...
private:
	int a,b,c;
	std::vector<double> v;
	//...数据量很大
};

class Widget
{
public:
	Widget(const Widget& rhs);
	Widget& operator=(const Widget& rhs)
	{
		//...
		*pImp1 = *(rhs.pImp1);
		//...
	}
private:
	WidgetImp1* pImp1;
};
```

我们这里给出特化进行处理：

```cpp
namespace std
{
	template<>
	void swap<Widget>(Widget& a, Widget& b)
	{
		swap(a.pImp1, b.pImp1);//编译失败，尝试操作private变量
	}
}
```

将其变化成为一个类似友元函数的东西：（但并不是）

```cpp
class Widget
{
public:
	...
	void swap(Widget& other)
	{
		using std::swap;//令std::swap在这里可用，可以为对象找到兜底的swap版本
		swap(pImp1, other.pImp1);//若置换Widget就只需要置换pImp1指针
	}
	...
};

namespace std
{
	template<>
	void swap<Widget>(Widget& a, Widget& b)
	{
		a.swap(b);//要进行置换，只要调用其的swap成员函数
	}
}
```

但是注意，偏特化只能用在class上，而不能用在函数上，所以假设我们对于Widget和WidgetImp1进行参数化，再在Widget内以及WidgetImp1内放个swap成员函数，将会带来错误：

```cpp
template<typename T>
class WidgetImp1 {...};

template<typename T>
class Widget {...};

namespace std
{
  template<typename T>
  void swap<Widget<T>>(Widget<T>& a, Widget<T>& b)//错误，尝试对函数进行偏特化
  {
    a.swap(b);
  }
}

//正确做法：添加一个重载版本
namespace std
{
  template<typename T>
  void swap(Widget<T>& a, Widget<T>& b)
  {
    a.swap(b);
  }
}
```

也就是说，当需要重载swap来替换原来的效率不足时，尝试以下事情：

1、提供一个public swap成员函数，可以高效置换，这个函数绝不该抛出异常（你在直接对内置类型进行手动操作）

2、在自己的class/template所在的namespace中，提供一个non-member的swap，并让它调用上面的swap

3、如果你正在编写class（而非class template），为你的class特化std::swap，并且令它调用我们自己写的swap

4、如果调用swap，确定包含一个using 表达式来让std::swap在函数内可见，然后不要加域修饰符，直接赤裸裸调用swap

