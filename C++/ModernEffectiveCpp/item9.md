## 优先使用声明别名而不是`typedef`

我们都不愿意重复地写一些类型名过于复杂的代码，所以引入了`typedef`，但是这是一种很老的，从C过来的传统，而自C++11开始，提供了声明别名(alias declarations)

```cpp
typedef std::unique_ptr<std::unordered_map<std::string, std::string>> UPtrMapSS;

using UptrMapSS = std::unique_ptr<std::unordered_map<std::string, std::string>>;
```

声明别名优越性：

+ 提升可读性，尤其是函数指针混合着模板的情况上（声明别名是可以模板化的，被称为模板别名(alias template)，而typedef做不到）

```cpp
//这个对比看上去区别不大
typedef void (*FP)(int, const std::string&);

using FP = void (*)(int, const std::string&);


//使用个性化分配器MyAlloc的链接表定义一个标识符
template<typename T>
using MyAllocList = std::list<T, MyAlloc<T>>;

MyAllocList<Widget> lw;//终端代码

//而使用typedef就不够优雅
template<typename T>
struct MyAllocList {
    typename std::list<T, MyAlloc<T>> type;
};

MyAllocListM<Widget>::type lw;//终端代码
```

这里`MyAllocList<T>::type`表示一个依赖于模板类型参数T的类型，因此`MyAllocList<T>::type`是一个依赖类型，名称前必须要冠以`typename`

但如果被定义成一个声明别名，就不用使用`typename`，它看似依赖模板参数`T`，但是编译器处理`Widget`遇到`MyAllocList<T>`是知道`MyAllocList<T>`是一个类型名称，因为`MyAllocList`是一个模板别名，显然就是一个类型。所以`MyAllocList<T>`在这里是非依赖类型，指定符`typename`不需要

```cpp
template<typename T>
using MyAllocList = std::list<T, MyAlloc<T>>;

template<typename T>
class Widget {
private:
    MyAllocList<T> list;
    ...
};
```

使用`typedef`这里甚至会有这样的代码，这时`MyAllocList<Wine>::type`不是一个类型，而是一个数据成员，这也是为什么前面编译器要求必须在类型名之前冠以`typename`前缀的原因

```cpp
class Wine{...};

template<>
class MyAllocList<Wine>//当T是Wine时的模板类偏特化
{
private:
	enum class WineType
	{ White, Red, Rose};

	WineType type;
	...
};
```

在模板元编程上，我们可能想剥夺`T`所包含的所有`const`或者引用的修饰符，即`const std::string`->`std::string`，可能想给类型增加`const`或者左值引用。C++给了我们`type_traits`以完成类型转换：

```cpp
std::remove_const<T>::type //const T -> T
std::remove_reference<T>::type //T& 或 T&& -> T
std::add_lvalue_reference<T>::type //T -> T&
```

上面体现了类型转换常常以`::type`作为每次使用的结尾，当一个模板中的类型参数使用时，必须在每次使用前冠以`typename`，因为C++11的类型特征就是从内嵌`typedef`到一个模板化的`struct`实现的，到C++14，我们对于上面的代码有了一个替代的统一别名`std::transformation_t<T>`来换掉`std::transformation<T>::type`

```cpp
std::remove_const_t<T> //const T -> T
std::remove_reference_t<T> //T&/T&& -> T
std::add_lvalue_reference_t<T> //T -> T&
```

## 要记住的东西

+ `typedef`不支持模板化，但是别名声明支持
+ 模板别名避免了`::type`后缀，在模板中，`typedef`还经常要求使用`typename`前缀
+ `C++14`为`C++11`中的类型特征转换提供了模板别名