## Chapter 9 Using Templates in Practice

模板代码和普通代码有点不同。在某些方面，模板位于宏和普通（非模板）声明之间。尽管这可能过于简单化，但它不仅会影响我们使用模板编写算法和数据结构的方式，还会影响到表达和分析包含模板的程序

### 包容模式

有几种方法可以组织模板源代码。本节介绍最流行的方法：包含模型。

#### 链接错误

大多数C和C++程序员的非模板代码组织大体如下：

+ 类和其他类型完全放在header files中。通常，这是一个扩展名为.hpp（或.H，.h，.hh，.hxx）的文件
+ 对于全局（非内联）变量和（非内联）函数，只有声明放在头文件中，定义放在编译为自身翻译单元的文件中这种cpp文件通常是扩展名为.cpp（或.C、.c、.cc或.cxx）的文件

它使得所需的类型定义在整个程序中都很容易获得，并且避免了来自链接器的变量和函数的重复定义错误。

考虑到这些约定，下面的（错误的）小程序说明了一个常见的错误，初学者会抱怨这个错误。

与“普通代码”一样，我们在头文件中声明模板：

```cpp
#ifndef MYFIRST_HPP
#define MYFIRST_HPP

//declaration of template
template<typename T>
void printTypeof(T const&);

#endif
```

```cpp
#include <iostream>
#include <typeinfo>
#include "myfirst.hpp"

template<typename T>
void printTypeof(T const& x)
{
  std::cout << typeid(x).name() << '\n';
}
```

一个C++编译器很可能不需要任何问题就接受这个程序，但是链接器可能会报告一个错误，这意味着函数`PrrTyType()`没有定义。

此错误的原因是函数模板`printTypeOf()`的定义尚未实例化。为了实例化模板，编译器必须知道应该实例化哪个定义以及应该实例化哪个模板参数。不幸的是，在上一个例子中，这两条信息都在分别编译的文件中。因此，当我们的编译器看到对`printtypeof()`的调用，但看不到为double实例化此函数的定义时，它只是假设在其他地方提供了这样的定义，并创建对该定义的引用（供链接器解析）。另一方面，当编译器处理`myfirst.cpp`文件时，在该点上没有指示它必须为特定参数实例化它包含的模板定义。

#### 模板在头文件中

前一个问题的常见解决方案是使用与宏或内联函数相同的方法：我们在声明模板的头文件中包含模板的定义。

也就是说，我们不提供myfirst.cpp文件，而是重写myfirst.hpp，使其包含所有模板声明和模板定义：

```cpp
#ifndef MYFIRST_HPP
#define MYFIRST_HPP

#include <iostream>
#include <typeinfo>

//declaration of template
template<typename T>
void printTypeof(T const&);

template<typename T>
void printTypeof(T const& x)
{
  std::cout << typeid(x).name() << '\n';
}

#endif
```

这种组织模板的方式称为“包含模型”模式

这种方法大大增加了包含头文件myfirst.hpp的成本，这个成本主要出现在使用`<iostream>`等头文件上。这大大增加了编译时间，对此有预编译头和显式模板实例化等来处理。但是目前还是这种模式是最优的，当然也需要观察C++20的module特性

注意非内联函数模板，它们不会在调用点展开，而是在被实例化时创建一个新的函数副本。由于这个是一个自动过程，所以编译器可能会在不同文件中创建多个副本，而部分链接器会在发现同一个函数结果有多个定义而报错（很罕见的问题）

### 模板与内联

将程序声明为inline通常可以提高程序的运行时间，这只是一种提示，实现由编译器自己决定生成。我们很多时候不应该自作主张

### 预编译头

它依赖于这个前提：很多文件都已相同的代码开头。我们可以编译这些相同的代码并且让编译器在该点的完整状态保存在预编译投中，然后后续要用的话从第N+1行开始编译。所以，包含头的顺序也是非常重要的

有些程序员认为最好是包含一些额外的不必要的头，而不是传递一个使用预编译头加速文件翻译的机会。这一决定可以大大简化包容政策的管理。例如，创建一个名为std.hpp的头文件（包含所有标准头）通常比较简单：

```cpp
#include <iostream>
#include <string>
...
```

然后可以对该文件进行预编译，使用标准库的程序文件可以使用`#include <std.hpp>`来处理，这通常也是一种很好的解决方案

管理预编译头的一个有吸引力的方法是创建预编译头的层，这些层从最广泛使用和最稳定的头（例如，我们的std.hpp头）到预期不会一直更改的头，因此仍然值得预编译。但是，如果头部正在进行大量开发，则为其创建预编译头部所需的时间可能比重用它们所节省的时间要长。这种方法的一个关键概念是，可以重用用于更稳定层的预编译头，以提高不稳定头的预编译时间。

### 解读模板报错

一般的编译错误通常都是非常简明扼要的。例如，当编译器说“类X没有成员‘fun’”时，通常不太难找出代码中的错误（例如，我们可能将run错误地输入为fun）模板不是这样：

```cpp
#include <string>
#include <map>
#include <algorithm>
int main()
{
  std::map<std::string, double> coll;
  
  auto pos = std::find_if(coll.begin(), coll.end(), [](std::string const& s){return s != "";});
}
```

它包含一个相当小的错误：在用于查找集合中第一个匹配字符串的lambda中，我们检查给定的字符串但是，映射中的元素是键/值对，因此我们应该期望`std::pair<std::string const, double>`

然后给的报错却非常复杂，但是其实就是模板展开的时候不匹配，慢慢读就好了

#### 在某些编译器上丢失了const

```cpp
#include <string>
#include <unordered_set>

class Customer
{
  private:
    std::string name;
  public:
    Customer(std::string const& n) : name(n){}
    std::string getName() const { return name; }
};

int main()
{
  struct MyCustomerHash
  {
    std::size_t operator() (Customer const& c)
    {
      return std::hash<std::string>()(c.getName());
    }
  };
  
  std::unordered_set<Customer, MyCustomerHash> coll;
  ...
}
```

这里的问题是：

```cpp
std::unordered_set<Customer, MyCustromerHash> coll;
//与调用不匹配：
const main()::MyCustomerHash(const Customer&)
```

一个可能的“near match”候选是：

```cpp
std::size_t main()::MyCustomerHash::operator()(const Customer&)
```

此`std::unordered_set`类模板的实现要求哈希对象的函数调用运算符是常量成员函数。如果不是这样的话，算法的内部就会产生一个错误。所有其他错误消息从第一个开始层叠，并在将const限定符简单地添加到哈希函数运算符时消失。总之，这部分需要长时间的训练才能得到提升...速成不太可能