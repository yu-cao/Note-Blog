##创建对象时区分()和{}

初始化方法：C++11开始推广列表初始化

```cpp
int x(0);
int y = 0;
int z{0};
int z = {0};
```

建议：一定要区分初始化与赋值这两个操作

```cpp
Widget w1;//缺省构造
Widget w2 = w1;//拷贝构造
w1 = w2;//赋值
```

大括号还可以用于为非静态数据成员指定默认初始化值，这个与C++11的新功能与“=”初始化语法共享，但不与括号共享

```cpp
class Widget{
    ...
    private:
    int x{0};
    int y = 0;
    int z(0);//Error!
};
```

另一方面，不可复制的对象（例如，`std::atomic`）可以使用大括号或括号初始化，但不能使用“=”：

```cpp
std::atomic<int> ai1{0};
std::atomic<int> ai2(0)l
std::atomic<int> ai3 = 0;//Error!
```

这也就证明了大括号初始化为什么被称为统一的(uniform)

**大括号初始化的一个特性是禁止内置类型之间的隐式缩小转换**，如果一个在大括号中的表达式的值不能保证被对象类型进行描述表达，则直接无法编译（但是使用括号可以进行隐式转换）：

```cpp
double x,y,z;

int sum{x + y + z};//编译错误，int无法表达double
```

花括号初始化的另外一个值得一谈的特性是它能避免C++最令人恼火的解析。一方面，C++的规则中，所有能被解释为声明的东西肯定会被解释为声明，最令人恼火的解析常常折磨开发者，当开发者想用默认构造函数构造一个对象时，他常常会不小心声明了一个函数。问题的根本就是你想使用一个参数调用一个构造函数，你能这么做：

```cpp
Widget w1(10);//使用参数10调用构造函数

//痛点
Widget w2();//这里会被解析成构造函数，而不是声明...

//函数不能使用花括号作为参数列表来声明
//所以使用花括号调用默认构造函数构造一个对象不会有这样的问题：
Widget w3{};//不带参数调用的Widget的默认构造函数
```

花括号初始化这种语法能用在最宽广的范围，它能阻止隐式收缩转换（narrowing convertions），并且它能避免C++最让人恼火的解析。

但是它的缺点也很明显：它与`std::initializer_list`之间的关系很复杂，以及在函数重载中的问题，这导致代码看上去本意与实际的执行出现偏差。比如`auto`声明的变量使用花括号时，类型会被推导为`std::initializer_list`，而其他的类型会产生更符合正确的类型。

而且**花括号初始化语法会强烈地偏向调用`std::initializer_list`的参数的函数，只要有任何机会能调用带`std::initializer_list`参数的构造函数，编译器就会采用这种解释。**（这一点非常致命，很可能会出现奇怪的错误）

```cpp
class Widget {
public:
    Widget(int i, bool b);  
    Widget(int i, double d);
    Widget(std::initializer_list<long double> il);

    operator float() const;//转换到float
    ...
};

Widget w1(10, true);//调用第一个构造函数

Widget w2{10, true};//调用第三个函数而不是第一个，10和true都被转化了

Widget w3(10, 5.0);//调用第二个构造函数

Widget w4{10, 5.0};//调用第三个函数而不是第二个，10和5.0都被转化了

Widget w5(w4);//使用圆括号，调用拷贝构造函数

Widget w6{w4};//使用花括号，调用std::initializer_list构造函数（w4 转换到float，然后float转换到long double）

Widget w7(std::move(w4));//使用圆括号，调用移动构造函数

Widget w8{std::move(w4)};//使用花括号，调用std::initializer_list构造函数
```

甚至会有这样的报错（即使`std::initializer_list`压根不匹配）而忽视最佳匹配：

```cpp
class Widget {
public:
    Widget(int i, bool b);  
    Widget(int i, double d);
    Widget(std::initializer_list<bool> il);//元素类型现在是bool

    ...
};  

Widget w{10, 5.0};//错误！需要收缩转换（narrowing conversion）
```

只有当这里没有办法转换参数的类型为`std::initializer_list`时，编译器才会回到正常的重载解析（这里压根没有转换能把10，5.0这种变成string类型）

```cpp
class Widget {
public:
    Widget(int i, bool b);  
    Widget(int i, double d);

    //元素类型现在是std::string
    Widget(std::initializer_list<std::string> il);  

    ...
};  

Widget w1(10, true);    //使用圆括号，和以前一样，调用第一个构造函数

Widget w2{10, true};    //使用花括号，现在调用第一个构造函数

Widget w3(10, 5.0);     //使用圆括号，和以前一样，调用第二个构造函数

Widget w4{10, 5.0};     //使用花括号，现在调用第二个构造函数
```

如果想使用元素为空的`std::initializer_list`来调用一个`std::initializer_list`函数，需要用空的花括号作为参数来调用---把空的花括号放在圆括号或者花括号中间：

```cpp
Widget w4({});      //使用空的list调用std::initializer_list构造函数
//或者
Widget w5{{}};
```

第一，作为一个class的作者，你需要意识到如果你设置了一个或多个`std::initializer_list`构造函数，客户代码使用花括号初始化时将会只看见`std::initializer_list`版本的重载函数。所以，**最好在设计你的构造函数时考虑到，客户使用圆括号或者花括号都不会影响到构造函数的调用。**

`std::initializer_list`构造函数重载的不同之处在于它不同别的版本的函数竞争，它直接占据领先地位以至于其它重载函数几乎不会被考虑。所以只有经过仔细考虑过后你才能增加这样的重载（`std::initializer_list`构造函数）。

如果是template的开发者，在创建对象时选择使用圆括号还是花括号时会尤其沮丧，因为通常来说，这里有没办法知道哪一种会被使用。

```cpp
template<typename T,        //对象的类型是T
         typename...Ts>     //参数的类型
void doSomeWork(TS&&... params)
{

    使用params创建局部T对象...
    ...
}

//两种形式把伪代码转换成真正的代码
T localObject(std::forward<Ts>(params)...); //使用圆括号

T localObject{std::forward<Ts>(params)...}; //使用花括号

//考虑这个代码：
std::vector<int> v;
...
doSomeWork<std::vector<int>>(10, 20);
```

如果doSomeWork使用圆括号来创建localObject，最后的`std::vector`将有10个元素。如果doSomeWork使用花括号，最后的`std:vector`将有两个元素。哪一种是对的？doSomeWork的作者不知道，只有调用者知道。(如我的一篇[blog](https://github.com/yu-cao/Note-Blog/blob/master/C%2B%2B/InterestingErrorInCpp/%E5%88%97%E8%A1%A8%E5%88%9D%E5%A7%8B%E5%8C%96%E5%92%8CCpp17%E7%89%B9%E6%80%A7%E6%B7%B7%E5%90%88%E5%AF%BC%E8%87%B4%E7%9A%84%E6%97%A0%E7%A9%B7%E7%BC%96%E8%AF%91.md))

##我们要记住的要点

+ 花括号初始化是用途最广的初始化语法，它阻止收缩转换，能避免C++最令人恼火的解析
+ 在构造函数重载解析中，只要有可能，花括号初始化都对应`std::initializer_list`构造函数，就算其他的构造函数看起来更好
+ 选择圆括号或花括号创建对象能起到重大影响的例子是用两个参数创建一个`std::vector`的对象
+ 在template内部，创建对象时选择圆括号或花括号是一个难题，需要小心斟酌