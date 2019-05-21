## 优先使用`delete`关键字删除函数而不是`private`却又不实现的函数

有些成员函数是`C++`自己生成的，比如复制构造函数和复制赋值操作。如果我们不想让用户调用这些函数，在`C++98`中阻止的策略是把这些函数声明为`private`且不进行定义。

比如在`C++`标准库中，IO流的基础是类模板`basic_ios`，所有的输入/输出流都继承（或间接继承）这个类。而拷贝输入/输出流显然是不正确的，而且我们不知道这时候该怎么做。比如我们有一个`istream`对象表示输入数值的流，一些已经读入内存，一些还没有，正在等待被读入。如果一个流被复制，那么已读入的数据和将来要读入的数据都会被复制吗？委员会的做法是定义这类问题不存在！

```cpp
template <class charT, class traits = std::char_traits<charT>>
class basic_ios : public ios_base
{
public:
	//...
	
private:
	basic_ios(const basic_ios&);
	basic_ios& operator(const basic_ios&;
};
```

在`C++11`中可以使用`=delete`标识拷贝复制函数和拷贝赋值函数是**删除的**，使用这种方式处理能够在编译期给出报错，而`C++98`的方式只能在链接时才会报错。此外，删除函数应该声明为公有，使得报错更为清晰

任何函数都可以是删除的，但是只有成员函数才能私有。比如

```cpp
bool isLucky(int);//只保留原来的函数，删去其他的函数
bool isLucky(char) = delete;
bool isLucky(bool) = delete;
bool isLucky(double) = delete;//同时拒绝double和float类型
```

此外，删除的函数可以阻止那些被禁用的模板实现。比如普通指针中，两种指针很特殊，`void*`不能被解引用，递增与递减；`char*`常常指向类`C`的字符串，而不是指向某个独立字符。这两种需要特殊处理：

```cpp
template<typename T>
void processPointer(T* ptr);

template<>
void processPointer<void>(void*) = delete;

template<>
void processPointer<char>(char*) = delete;

//你也许会还想对其中的const类型删除，const volatile类型删除
```

而这在`C++98`的私有化方式是无法实现的，因为赋予一个成员函数模板在特化情况的的特殊访问权限是不可能。

## 要记住的东西

+ 优先使用删除函数而不是私有而不定义的函数
+ 任何函数都可以被声明为删除，包括非成员函数和模板实现