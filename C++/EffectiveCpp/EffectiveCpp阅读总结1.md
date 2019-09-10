## Effective C++ 阅读总结1

<hr>

### 1、视C++作为一个语言联邦，主要分为4个部分：C、Object-Oriented C++、Template C++、STL

<hr>

###2、以`const`，`enum`，`inline`来替换`#define`

**宁愿用编译器而不要使用预处理器**，用一个常量来替换宏。此外define不能提供任何的封装性，而const等可以被封装。

以常量来替换`#define`的两种特殊情况：

1、定义常量指针：

```cpp
const char* const authorName = "Scott Meyers";
const std::string authorName("Scott Meyers");//better
```

2、class专属常量（无法用define创建，因为define是无作用域的）

```cpp
class GamePlayer{
private:
  static const int NumTurns = 5;//只是一个声明式，而不是定义式
  int scores[NumTurns];
};
```

如果不取地址，且这个class专属常量是`static`且为整数型的，则可以声明使用，无需定义式。要是你一定要看到它的地址，需要额外提供定义式，将其放入实现文件而非头文件：

```cpp
const int GamePlayer::NumTurns;//声明时已获得初值，定义时不可再次给初值
```

**Enum hack**：

```cpp
class GamePlayer{
private:
  enum {NumTurns = 5};//用Enum来阻止Caller来通过pointer/ref来指向某个整数常量
  //而且这个方法在模板元编程上非常重要
  int scores[NumTurns];
}
```

**不要使用define来定义函数，可以用template inline函数代替**

```cpp
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))

//即使每个地方都加上了括号，也存在严重风险！
int a = 5, b = 0;
CALL_WITH_MAX(++a, b);//a被累加2次
CALL_WITH_MAX(++a, b + 10);//a被累加一次

template<typename T>
inline void callWithMax(const T& a, const T& b)
{
  f(a > b ? a : b);
}
```

<hr>

### 3、尽可能用`const`

只要这个值不变是一个事实，就应该告诉给编译器，来让它一起帮你管理

`const`在星号左边，表示被指物是只读的；在星号右边，表示指针本身是只读的

STL迭代器的作用就类似于`T*`，声明迭代器是const也就等于`T* const`，不可指向其他东西；如果想要指向的东西是不可改动的（类似`const T*`），需要使用`const_iterator`

成员函数只有在不改变对象的任何non-static成员变量时才可以说是`const`，也就是说，const成员函数不会对已经实例出来的对象有哪怕1bit的更改（编译器的逻辑）

但是我们有时候有也想要在const函数中操作变量，那么就应该使用`mutable`关键字处理

```cpp
class CTextBlock{
public:
	std::size_t length() const;
	char& operator[](std::size_t position) const//不符合bitwise-const，如果调用进行赋值，还是会隐性对对象进行修改
	{
		return pText[position];
	}
private:
	char* pText;
	mutable std::size_t textLength;//使用关键字mutable
	mutable bool lengthIsValid;
};

std::size_t CTextBlock::length() const
{
	if(!lengthIsValid)
	{
		textLength = std::strlen(pText);//mutable后，成员变量即使是在const成员函数中也会被更改
		lengthIsValid = true;
	}
	return textLength;
}
```

当代码开始复杂时，我们可能需要维护两个功能极其相近的代码，但是他们的区别可能只是一个是const，另一个不是，这时候可以通过合理的转换来优化重复代码：（注意，应该是non-const调用const，不要反向调用，因为const已经代表你对于对象内部不变的承诺，尽量不要去违背）

```cpp
class TextBlock{
public:
  const char& operator[](std::size_t position) const
  {
    ...//包括边界检查，数据完整性校验etc
    return text[position];
  }
  char& operator[](std::size_t position)
  {
    return const_cast<char&>(
      static_cast<const TextBlock&>(*this)//这里我们必须指明我们要调用const operator[]，避免发生无穷递归，因为没有这个语法，所以通过将类转换成const来替代
    [position]);
  }
};
```

<hr>

### 4、确定对象使用前已经被初始化

永远使用成员初始化列表，它比赋值更加高效，也绝对非常有必要。（当类存在多个构造函数时，如果某些赋值与初始化效率相近，而且在各个构造函数中都需要一样被调用时，可以写成一个private函数，通过赋值方式进行“初始化”

初始化列表顺序：基类优先，成员变量按**声明顺序**初始化

local static对象：函数内的静态对象

Non-local static对象：global对象，在namespace作用域内的对象等等

那么，**不同编译单元的non-local static对象初始化，如果需要相互依赖，则初始化顺序非常重要，不幸的是，C++对于这个顺序没有规定**，这里有一个trick来解决：将每个non-local static搬到自己的专属函数中（该对象在这个函数中声明为static），这些函数返回一个ref指向这个对象，函数去调用这些函数，而不是直接使用这些对象（也就是Singleton模式的一个常见实现方法）