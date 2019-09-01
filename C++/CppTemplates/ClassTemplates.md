## Chapter2 Class Templates

代码如下：

```cpp
#include <vector>
#include <cassert>

template<typename T>
class Stack
{
private:
   std::vector<T> elems;// elements
   
public:
   void push(T const &elem);// push element
   void pop();// pop element
   T const &top() const;// return top element
   bool empty() const
   {// return whether the stack is empty
      return elems.empty();
   }
};


template<typename T>
void Stack<T>::push(T const &elem)
{
   elems.push_back(elem);// append copy of passed elem}
}

template<typename T>
void Stack<T>::pop()
{
   assert(!elems.empty());
   elems.pop_back();// remove last element
}

template<typename T>
T const &Stack<T>::top() const
{
   assert(!elems.empty());
   return elems.back();// return copy of last element
}

int main()
{
	Stack<int> intStack;
	Stack<std::string> stringStack;

	intStack.push(7);
	std::cout << intStack.top() << '\n';

	stringStack.push("Hello");
	std::cout << stringStack.top() << '\n';
	stringStack.pop();
}
```

只有当被调用时，模板才会被实例化，实例化的模板类型可以是任何类型，包括`const`、`volatile`等等，也支持自定义类型，支持`using`或者`typedef`等：

```cpp
void foo(Stack<int> const& s)
{
  using IntStack = Stack<int>;
  Stack<int> istack[10];
  IntStack istack[2];
}
```

部分使用类型模板：模板参数只需要提供它们的必要操作，而不是提供给它们所有需要的操作，比如以下借用ostream的<<操作符来完成输出操作

```cpp
template<typename T>
class Stack
{
  ...
    void printOn()(std::ostream& strm) const
  {
    for(T const& elem : elems)
    {
      strm << elem << ' ';
    }
  }
};
```

但是如何知道哪些操作是模板能够被实例化所必须的呢？这通常借助模板库中需要重复一组约束，比如*random access iterator*、*default constructible*（默认可构造）等等，但是这个还是没有一个标准化的东西，但是我们至少可以通过`static_assert`和预定义类型等来进行检查，否则很可能出现一些可怕的错误

```cpp
template<typename T>
class C
{
  static_assert(std::is_default_constructible<T>::value, "Class C requires default-constructible elements")
};
```

没有这个检查，报错可能包含整个模板实例化历史记录，从实例化的初始原因到检测到错误的实际模板定义，排查bug可读性就会极差

如果不使用`printOn()`来打印，我们可以重载`<<`进行。通常情况下`<<`被实现作为非友元函数，它会内联调用`printOn()`

```cpp
template<typename T>
class Stack
{
  ...
    void printOn() {std::ostream& strm} const {...}
  
  friend std::ostream& operator<< (std::ostream& strm, Stack<T> const& s)
  {
    s.printOn(strm);
    return strm;
  }
};
```

###友元函数

但是我们尝试声明一个友元函数并且给出定义，事情就会变得复杂。我们要有以下2个观点：

1、我们可以隐式声明一个新的函数模板，它必须使用不同的模板参数。再次使用T或跳过模板参数声明都不会起作用（内部T隐藏外部T或我们在命名空间范围内声明非模板函数）：

```cpp
template<typename T>
class Stack
{
  ...
    template<typename U>
    friend std::ostream& operator<< (std::ostream&, Stack<U> const&);
};
```

2、我们可以将`Stack <T>`的输出运算符声明为模板，但这意味着我们首先必须提前声明`Stack <T>`

```cpp
template<typename T>
class Stack;
template<typename T>
std::ostream& operator<< (std::ostream&, Stack<T> const&)
```

这样我们可以这样声明友元函数，注意在函数名后面的`<T>`，因此，我们声明一个具体化的非成员函数模板作为友元：

```cpp
template<typename T>
class Stack
{
  ...
    friend std::ostream& operator<< <T> (std::ostream&, Stack<T> const&);
};
```

在任何情况下，我们仍然可以将此类用于没有定义`operator <<`的元素。仅对此堆栈调用`operator <<`会导致错误

```cpp
	Stack<std::pair<int, int>> ps;//pair没有<<的定义
	ps.push({4,5});
	ps.push({6,7});
	std::cout << ps.top().first << '\n';
	std::cout << ps.top().second << '\n';
	std::cout << ps << '\n';//ERROR: operator<< not supported for element type
```

###特例化（Specializtion）

可以为一些模板参数来特例化一个类的模板。与函数模板的重载类似，特例化的类模板允许优化某些类型的实现，或者修复某些类型的错误行为以用于类模板的实例化。 但是，如果特例化类模板，则还必须特例化所有成员函数。 虽然可以特例化类模板的单个成员函数，但是一旦完成，就不能再特例化专用成员所属的整个类模板实例。特例化：

```cpp
template<>
class Stack<std::string>
{
...
};
```

然后后面所有T的地方都变成特例化的描述，而且特例化可以用与原来截然不同的数据结构组织方式（比如原来是vector，你可以变成deque）

###偏特化（Partial Specialization）

类模板可以**部分专用**。针对特定情况提供特殊实现，但仍必须由用户定义某些模板参数：

```cpp
template<typename T>
class Stack<T*>
{
private:
	std::vector<T*> elems;

public:
	void push(T*);
	T* pop();
	T* top() const;
	bool empty() const {
		return elems.empty();
	}
};

template<typename T>
void Stack<T*>::push(T * elem)
{
	elems.push_back(elem);
}

template<typename T>
T* Stack<T*>::pop()
{
	assert(!elems.empty());
	T* p = elems.back();
	elems.pop_back();
	return p;
}

template<typename T>
T* Stack<T*>::top() const
{
	assert(!elems.empty());
	return elems.back();
}

int main()
{
  Stack<int*> ptrStack;
  ptrStack.push(new int{42});
  std::cout << *ptrStack.top() << '\n';
  delete ptrStack.pop();//比如这里，偏特化结果与原来的是有不同的，指针支持delete
}
```

多参数偏特化，注意，偏特化后的优先级与其他偏特化后的优先级一致，所以不小心可能会容易产生二义性调用error（比如下面的偏特化调用：`MyClass<int, int> m;`或者`MyClass<int*, int*>`）

```cpp
//原：
template<typename T1, typename T2>
class MyClass{...};

//偏特化：两个模板类型一致
template<typename T>
class MyClass<T,T>{...};

//偏特化，第二个参数是int
template<typename T>
class MyClass<T, int>{...};

//偏特化：两个都是指针
template<typename T1, typename T2>
class MyClass<T1*, T2*>{...};
```

缺省类模板参数：

```cpp
template<typename T, typename Cont = std::vector<T>>
class Stack{...};

template<typename T, typename Cont>
void Stack<T, Cont>::push(T const& elem){...}
```

###编辑器推导类模板参数（C++17）

C++17支持了类模板参数推导，如果我们从构造函数继承，允许我们跳过显式地给出模板参数，而使用编译器推导的结果：

```cpp
Stack<int> intStack1;                 // stack of strings
Stack<int> intStack2 = intStack1;     // OK in all versions
Stack intStack3 = intStack1;           // OK since C++17

template<typename T> 
class Stack { 
   private:
     std::vector<T> elems;        // elements 
   public:
     Stack () = default;//这里必须要支持默认构造函数可用
     Stack (T const& elem)//有元素要初始化时，使用初始化列表进行初始化，这里要加{}原因是默认vector会判定以为elem是要开空间大小而不是元素
       : elems({elem}) {
     }
     …
};

Stack intStack = 0;           // Stack<int> deduced since C++17
```

字符串文本进行类模板参数推导时，参数会因为没有衰减（decay）导致真的去得到了一个奇怪的模板参数：

```cpp
Stack stringStack = "bottom"; // Stack<char const[7]> deduced since C++17
```

这里就有大问题了。但是，当通过值传递模板类型T的参数时，参数就会像我们期望的一样衰减，这是将原始数组类型转换为相应的原始指针类型的机制。 也就是说，构造函数的调用参数T被推导为char const *，因此整个类被推导为`Stack <char const *>`，修改如下：

```cpp
template<typename T> 
class Stack { 
   private:
     std::vector<T> elems;        // elements 
   public:
     Stack (T elem)//通过值传递
       : elems({elem}) {//to decay on class tmpl arg deduction
     }
     …
};

//现在一切正常了
Stack stringStack = "bottom"; // Stack<char const*> deduced since C++17

//更好的方式应该是使用移动语义（针对上面的值传递）
Stack(T elem) : elems({std::move(elem)}){
}
```

但是这样搞真的也很麻烦（在容器中处理原始指针）

我们可以定义特定的deduction guides，以提供额外的或修复现有的类模板参数推导。 例如，可以定义每当传递字符串文字或C字符串时，都会为std :: string实例化堆栈，这个操作必须要跟类定义在同一个namespace下：

```cpp
Stack(char const*) -> Stack<std::string>;
...

Stack stringStack{"bottom"};        // OK: Stack<std::string> deduced since C++17
Stack stringStack = "bottom"; // Stack<std::string> deduced, but still not valid
```

###模板化的聚类

聚类（没有用户提供的，显式的或继承的构造函数的类/结构，没有私有或受保护的非静态数据成员，没有虚函数，也没有虚、私有或受保护的基类）也可以是模板。 例如：

```cpp
template<typename T>
struct ValueWithComment{
  T value;
  std::string comments;
}
```

为它所拥有的值val的类型定义参数化的聚合。 可以将对象声明为任何其他类模板，并仍将其用作聚合：

```cpp
ValueWithComment<int> vc;
vc.value = 42;
vc.comment = "initial value"
```

从C++17开始，我们可以为聚类模板定义推导法则

```cpp
ValueWithComment(char const*, char const*) -> ValueWithComment<std::string>;
ValueWithComment vc2 = {"hello", "initial value"};
```

这里没有提供推导法则的话，初始化是无法成功的，因为没有构造函数可以被用来推导。标准库类`std::array <>`也是一个聚合，为元素类型和大小参数化。 C++17标准库还为它定义了一个推导法则。