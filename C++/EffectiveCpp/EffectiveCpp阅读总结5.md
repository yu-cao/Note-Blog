## Effective C++ 阅读总结5

<hr>

### 1、尽量延后变量定义式的出现时间

应该尽量延后变量的定义**直到能够给他初始实参为止**。这样就避免构造（析构）非必要函数，更可以避免无意义的default构造行为，而在构造时直接指定初值。

<hr>

### 2、尽量少做转型动作

`const_cast`：常量性转除

`dynamic_cast`：安全向下转型。用来决定某对象是否归属继承类型中的某个类型

`reinterpret_cast`：低级转型，实际动作及结果可能取决于编译器

`static_cast`：强迫隐式转换：如non-const->const；int->double等等

尽可能在转型的时候使用以上四种新式转型操作，而不是原来的类C的强制转换

转型会生成运行期的代码，而不是简单更改代码的解释。

绝不应该做出“对象在C++中如何布局”的假设下进行转型，因为平台差异

派生类调用基类函数的时候，绝不应该先进行转换`*this`为基类对象进行调用。因为这个调用并不是当前对象上的函数，而是转型动作所建立的一个“`*this`对象的base class成分”的暂时副本身上的函数！注意，是副本而不是本身！

正确的做法是显式地告诉编译器你要调用的是基类的函数

要对`dynamic_cast`的开销保持警惕，绝对不要写出连串的`dynamic_cast`：

```cpp
class Window { ... };
...
typedef std::vector<std::shared_ptr<Window>> VPW;
VPW winPtrs;
...
for(VPW::iterator iter = winPtrs.begin(); iter != winPtrs.end(); iter++)
{
  if(SpecialWindow1* psw1 = dynamic_cast<SpecialWindow1*>(iter->get())) { ... }
if(SpecialWindow2* psw2 = dynamic_cast<SpecialWindow2*>(iter->get())) { ... }
if(SpecialWindow3* psw3 = dynamic_cast<SpecialWindow3*>(iter->get())) { ... }
  ...
}
```

这样的代码又大又慢，而且基础不稳。一点Window class的继承体系有变化，这里就需要再次check是否需要修改。这些代码应该通过某些基于virtual函数调用的东西进行取代。

<hr>

### 3、避免返回 “Handles” 指向对象的内部部分

也就是说，不要返回出可以直接操作对象内部数据的指针，引用，迭代器等东西。而且这样的传递可以会导致因为handle比所指的对象更加长寿引发指针悬垂的问题。总之，这些操作将会降低封装性，带来不必要的风险，如无必要，尽量规避。

<hr>

### 4、为“异常安全”的代码而努力

当异常被抛出时，异常安全的函数会：

+ 不泄露任何资源
+ 不允许数据破坏

```cpp
class PrettyMenu
{
public:
	void changeBackground(std::istream& imgSrc);

private:
	Mutex mutex;//互斥器
	Image* bgImage;//目前背景图像
	int imageChanges;//背景图像改变次数
};

void PrettyMenu::changeBackground(std::istream &imgSrc)
{
	lock(&mutex);
	delete bgImage;
	++imageChanges;
	bgImage = new Image(imgSrc);//抛出异常，直接死锁了，而且imageChange被累加，但是并没有数据
	unlock(&mutex);
}

//修改如下
void PrettyMenu::changeBackground(std::istream &imgSrc)
{
	Lock m1(&mutex);//可以保证它能够被稍后释放
	delete bgImage;
	++imageChanges;
	bgImage = new Image(imgSrc);
}
```

异常安全函数应该提供以下三个保证之一：

+ 基本承诺：如果异常被抛出，程序内的其他东西依然保持有效。比如上面的new出错，我们可以提供一个缺省的图像或者保持原来的图像以保证不会发生指针问题出错
+ 强烈保证：如果异常被抛出，程序状态不改变。要么成功，要么返回调用前的状态
+ 不抛出保证：承诺绝不抛出异常，因为他们能完成他们原来承诺的功能。

可能的话尽量选择nothrow保证，但是对于大部分函数而言，还是考虑基本保证与强烈保证之间。

对于上面的例子，提供强列保证不难。只需要我们把`Image*`变成智能指针即可良好管理资源（而且这样不再需要delete语句了）。其次重排句子，只有当更换image完成后再去累加。

其次，我们可以用copy and swap的策略，来保证对象状态是完整的。先完整地创建出一个副本，再在其上进行一系列修改，如果异常，原对象是不变的。当完成所有操作后进行swap

但是因为函数有时候有全局影响，或者copy and swap需要的额外的时间与空间是我们不愿承受的，就应该再退而求其次，追求基本保证。

<hr>

### 5、理解`inline`的里里外外

inline将会帮助编译器进行优化，但是这样会导致程序体积增加，产生额外的换页行为。其次，inline只是一个提示，一个申请，而不是强制。

inline函数通常一定被置于头文件中，因为bulid环境都是在编译器inline拓展，所以编译器需要知道函数到底长什么样。

所以，编译器会忽略过于复杂的函数的inline，会忽略virtual的inline（因为virtual需要运行期判定）通常不会对通过函数指针进行的调用实施inline

如果有个异常在对象构造时被抛出，那么这个对象已经构造好的部分将会被自动销毁...这些C++标准规定了会发生，但不规定怎么发生。所以看上去没有任何东西的派生类的构造函数，实际上可能是非常复杂的：

```cpp
class Base
{
public:
	...
private:
	std::string bm1, bm2;
};

class Derived : public Base
{
public:
	Derived(){}//表面上是空的
	...
private:
	std::string dm1, dm2, dm3;
};

//也许实际上的编译器实现：
Derived::Derived()
{
	Base::Base();
	try { dm1.std::string::string();}//尝试构建dm1
	catch (...) {//如果抛出异常就销毁已经建立的base class
		Base::~Base();
		throw;
	}
	try { dm2.std::string::string();}
	catch (...) {
		dm1.std::string::~string();
		Base::~Base();
		throw;
	}
	try { dm3.std::string::string();}
	catch (...) {
		dm2.std::string::~string();
		dm1.std::string::~string();
		Base::~Base();
		throw;
	}
}
```

在绝大部分调试器中，inline都不会发生作用。

我们应该一开始不设置任何inline，然后设定那些一定成为inline或非常简单的函数上。慎重使用inline为我们日后的调试带来帮助，也避免了手工最优化的优化陷阱。

<hr>

### 6、最小化文件之间的编译依赖

注意接口与实现的分离。而分离的关键在于“声明的依存性”替换“定义的依存性”。让头文件尽量可以自我满足，如果做不到，就让它与其他文件内的声明式（非定义式）相依。设计策略如下：

+ 如果使用object reference或者object pointer可以完成任务，就不要使用object
+ 如果能够，尽量以class声明式替换class定义式。
+ 为声明式和定义式提供不同的头文件

当然使用抽象基类来作为interface的策略也是可行的，同时也给予了类的弹性。