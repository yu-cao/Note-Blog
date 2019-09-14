## Effective C++阅读总结3

<hr>

### 1、以对象管理资源

```cpp
class Investment {...};

//使用工厂模式提供特定的Investment对象
void f()
{
  Investment* pInv = createInvestment();
  ...
  delete pInv;
}
```

但是上面的代码存在隐患，在`...`中如果出现 了提前return，或者抛出异常等情况，将会使得直接略过`delete`，导致内存泄漏。我们无法保证：这里的`f`是一定会执行delete行为的。

正确方法：将资源放入对象内，就可以依赖C++的析构函数自动调用原则保证资源的释放：（建议C++11后使用`unique_ptr`来替换`auto_ptr`）

```cpp
void f()
{
  std::auto_ptr<Investment> pInv(createInvestment());
}
```

即：1、获得资源后立刻放入管理对象 2、管理对象运用析构函数确保资源被释放

注意，`auto_ptr`的复制将会导致原来指向的地方变成`null`，C++11后应该使用`unique_ptr`，必要时使用移动语义来替换这个操作。

<hr>

### 2、在资源管理类中小心Copying行为

资源取得时机就是初始化时机（Resource Acquisition Is Initialization，RAII）

当我们需要自己建立资源管理类时：（如下面确保不会忘记将一个被锁住的Mutex解锁）

```cpp
class Lock{
public:
  explicit Lock(Mutex* pm) : mutexPtr(pm)
  { lock(mutexPtr); }
  ~Lock() { unlock(mutexPtr); }
private:
  Mutex *mutexPtr;
}

//客户调用符合RAII
Mutex m;
...
{							//建立critical section
  Lock m1(&m);//锁定互斥器
  ...					//执行critical section内的操作
}							//区块结束，自动解除互斥器的锁定
```

但是如果Lock对象被复制：

```cpp
Lock m11(&m);//锁定m
Lock m12(m11);//复制m11
```

我们可以有两种策略：1、禁止复制 2、使用引用计数法

<hr>

### 3、在资源管理类中提供对原始资源的访问

比如提供get函数来获得内部的原始资源，穿透外面的封装

对于原始资源的访问可能经过显式转换或者隐式转换，显式比较安全，隐式对调用者比较便利，需要取舍

<hr>

### 4、成对使用new和delete时要采用相同形式

也就是说，被删除的那个指针，所指的是单一对象，还是一个对象数组？

单一对象：| Obj |

对象数组：| n | Obj | Obj |...

```cpp
std::string* stringPtr1 = new std::string;
std::string* stringPtr2 = new std::string[100];
...
delete stringPtr1;
delete[] stringPtr2;
```

当new表达式中使用[]，那么对应的delete也要有[]，如果在new表达式中不使用[]，那么对应的delete中也一定不要使用[]，错误使用都会导致未定义行为的发生。（尤其要注意千万不要对于数组进行typedef等操作，非常容易发生delete没有加[]的情况）

<hr>

### 5、以独立语句将newed对象存入智能指针

```cpp
int priority();
void processWidget(std::shared_ptr<Widget> pw, int priority);

//以前面：以对象管理资源 的思维，考虑调用
processWidget(new Widget, priority());//ERROR
//这里需要一个原始指针来完成，但是构造函数是显式的，无法完成隐式转换

processWidget(std::shared_ptr<Widget>(new Widget), priority());//OK
```

C++在调用processWidget之前，编译器需要完成3件事：

+ 调用priority

+ 执行“new Widget”

+ 调用std::shared_ptr构造函数

但事实上这三者在C++中的执行顺序是弹性的。如果执行顺序是2->1->3，在1调用中引发异常，结果就是new返回指针丢失，而我们对于资源的保护是在智能指针中完成的，提前失败导致的指针丢失将会导致资源直接泄漏。

修改策略：将无法明确先后顺序，而且先后顺序与安全很重要的语句进行拆分

```cpp
std::shared_ptr<Widget> pw(new Widget);
processWidget(pw, priority());
```

