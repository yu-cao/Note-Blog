## Object Lessons

C++在布局与存取时间上的负担主要是因为`virtual`特性带来的，主要为：

+ `virtual function`机制：支持一个有效率的”执行期绑定“
+ `virtual base class`机制：用来实现多次出现在继承体系中的base class，有单一而被共享的实例

### 简单对象模型

```cpp
class Point
{
public:
	Point(float xval);
	virtual ~Point();
	
	float x() const;
	static int PointCount();
	
protected:
	virtual ostream& print(ostream &os) const;
	
	float _x;
	static int _point_count;
};	
```

member本身并不放在object之中。只有“指向member的指针”才放在那个object中，这样可以避免“members有不同的类型，因而需要不同的存储空间”带来的问题。

Object中的members是以slot的索引值来寻址的。比如`_x`的索引是6，`_point_count`的索引是7。所以，**object大小 = 指针大小✖️member个数**

这个模型没有用在实际产品傻姑娘，但是这个观念被应用到C++的“指向成员的指针”观念之中。

### 表格驱动对象模型

把所有与members相关的信息抽出来，放在一个data member table和一个member function table之中

一个Point pt，有两个指针指向两个table：

+ Member Data Table：内含实际数据
+ Function Member Table：内含函数地址

这个模型也没有真正用在C++编译器上，但是member function table这个设计却成为virtual function的一个方案

### C++对象模型

nonstatic data members被配置在每一个class object之内，static data members被存放在个别的class object之外。static与nonstatic function members也被放在个别的class object之外。virtual function实现：

+ 每个class产生出一堆指向virtual functions的指针，放在virtual table中
+ 每个class object被安插一个指针，指向相关的virtual table。vptr的设定，重置都是由每个class的constructor、destructor和copy assignment运算符自动完成。每个class关联的type_info object（用来支持RTTI，runtime type identification）被放在table的第一个slot

也就是说：一个具体实例中的内存布局为存在一个vptr和非静态数据成员：

```cpp
|--------------|
|   float _x   |                        |-->|---------------|
|--------------|                        |   |               |
|__vptr __Point| --->|-------------|    |   |---------------|
|--------------|     |             |---/   type_info for Point
                     |-------------|
                     |             |------->|---------------|
                     |-------------|        |Point::~Point()|
                     |             |------  |---------------|
                     |-------------|      \
                  virtual table for Point   \->|----------------------|
                                               |Point::print(ostream&)|
                                               |----------------------|
---------------------------------------------------------------------------------------in/out
|------------------------------|     |------------------------------|
|static int Point::_point_count|     |static int Point::PointCount()|
|------------------------------|     |------------------------------|

|-------------------|     |----------------|
|Point::Point(float)|     |float Point::x()|
|-------------------|     |----------------|
```

#### 继承情况

单一继承、多重继承、虚继承

虚继承：base class不管在继承链中被派生多少次，永远只会存在一个实例。（比如`iostream`继承了`istream`和`ostream`，而这两者继承了`ios`，但是只有`ios`这一个base class实例）

```cpp
class iostream : public istream, public ostream {...};

class istream : virtual public ios {...};
class ostream : virtual public ios {...};
```

继承的两种设计：

1、每一个base class可以被derived class object内的一个slot指出，slot中内含base class subobject的地址。缺点：间接性导致空间与时间的浪费；优点：class object不会随base class改变而改变

2、base class table模型：当其被产生时，表格中的每一个slot内含一个相关的base class地址（与vtable类似）。优点：1、每个class object对于继承都有一致的表现：每个class object都应该在某个固定位置安放一个base table指针，与base classes大小与个数无关；2、无需改变class object本身就能更改base class table

最初的C++继承不使用任何间接性，base class object的member被直接放在derived class object中。但是这导致了如果base class member的任何改变会导致所有派生类都需要重新编译

```
|---------|
|   base  |
|---------|        原始的派生类的内部结构，就是在原来base的基础上缝合入派生的东西
| derived |
|---------|
```

C++2.0以后导入virtual base class。具体是：在class object中为每个有关联的virtual base class加上一个指针。其他演化而来的模型基本都是这个思路

### 关键词带来的差异

比如，struct与class有什么区别？——绝不在C++中使用struct，因为它真的，没用！

有时候，C语言的trick会在C++中存在陷阱（把单一元素的数组放在一个struct尾端，使得每个struct objects拥有可变大小的数组）：

```cpp
struct mumble
{
	char pc[1];
};

struct mumble *pmumb1 = (struct mumble*)malloc(sizeof(struct mumble) + strlen(string) + 1);

strcpy(&mumble.pc, string);
```

但是如果改成用C++声明，这个class是：

+ 指定多个access section，内含数据
+ 从另一个class派生而来
+ 定义了一个或多个virtual functions

但是也许转化会失败！

C++凡处于一个access section的数据，必须保证用声明方式排列内存布局。但是放置在多个access section中的数据，排列顺序就不定了。也就是说，需要考虑到底protected data member放在private data members前面或者后面；同理，base classes与derived classes的data members的布局也没有规定哪个在前，虚函数也会可能使得这个设计失效，所以：**不要自作聪明**

### 对象的差异

三种设计范式：

+ 程序模型（procedural model）：类似C语言
+ 抽象数据类型模型（ADT）：抽象成为一个类和一组表达式（public接口）一起提供
+ 面对对象模型（Object-Oriented model）：有一些彼此相关的类型，通过一个抽象的base class封装起来并进行继承

**建议保持一种设计范式来写程序，而不要混合**

C++用以下方法支持多态：

1、通过一组隐式转换，例如把derived class指针转化成为一个指向其public base type的指针

2、通过virtual function机制

3、通过`dynamic_cast`和`typeid`等运算符

<hr>

回过头来，我们讨论一个class object到底需要多少内存？

+ 其non-static data member的总和大小
+ 加上任何由于alignment的需求而padding上去的空间
+ 加上为了支持virtual而由内部产生的任何额外开销（比如vptr）

<hr>

### 指针的类型

一个指向对象的指针，到底和指向整数（如int）的指针之间有什么不同？

+ 从内存需求上看，没有什么不同！都需要一个word的内存来放置一个机器地址。
+ 具体不同在于其所寻址出来的object类型不同：“指针类型”会告知编译器如何解释某个特定地址中的内存内容及其大小，也就是说，指向地址1000的int的指针，在32位机器上，涵盖的地址空间为1000~1003

但是需要注意的是`void*`指针，它会涵盖多少地址空间？——不知道！所以`void*`指针只能持有一个地址，而不能通过它操作所指的object

所以，cast只是一个编译器命令，大部分情况下并不改变一个指针所含的真实地址，而只是改变解释这个指针的方式

<hr>

#### 加入多态以后的内存布局

在基类后面增加派生类的一系列非静态数据成员

如果指向基类的指针想要调用其基类派生出的派生类的函数，需要通过`static_cast`或者`dynamic_cast`，原因如上：解释这个指针所持有的空间不同带来的区别

<hr>

总之，在弹性（OO）与效率（OB）之间常常存在取舍，我们需要按照需求，合理决断
