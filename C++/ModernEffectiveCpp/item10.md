##优先使用作用域限制的`enum`而不是无作用域的`enum`

一般而言，在花括号里面声明的变量名会限制在括号外的可见性。但是这对于`C++98`风格的`enum`中的枚举元素并不成立；事实就是枚举元素泄露到包含它的枚举类型所在的作用域中，对于这种类型的`enum`官方称作无作用域的（unscoped）。在`C++11`中对应的使用作用域的`enum`（scoped enums）（也就是所谓枚举类）不会造成这种泄露：

```cpp
//old-style
enum Color {black, white, red};
auto white = false;//错误，white在本作用域中已经有了声明


//std=c++11，使用有作用域的枚举
enum class Color {black, white, red};

auto white = false;//正确，这个作用域中没有其他的white

Color c = white;//错误，这个作用域中没有叫"white"的枚举元素

Color c = Color::white;//正确
auto c = Color::white;//正确
```

带有限制作用域的`enum`可以减少命名空间污染，此外它们的枚举元素类型会更加丰富，而不像无作用域`enum`的隐式转换为整数类型

以下代码合法：

```cpp
enum Color {black, white, red};//无作用域限制的enum
std::vector<std::size_t> primeFactors(std::size_t x);//返回质因子函数

Color c = red;

if(c < 14.5)//将Color与double进行比较
{
    auto factors = primeFactors(c);//计算Color变量的质因子
}
```

而增加一个`class`变成一个枚举类之后，就会有

```cpp
enum class Color {black, white, red};

Color c = Color::red;

if(c < 14.5)//出错，不能把Color类型与double比较
{
    auto factors = primeFactors(c);//出错，不能把Color类型传递给参数类型是size_t的函数
}

//使用强制类型转换可以满足我们的需求
if(static_cast<double>(c) < 14.5)
{
    auto factors = primeFactors(static_cast<std::size_t>(c));
    ...
}
```

而且枚举类可以不指定枚举元素进行声明：

```cpp
enum Color;//Error
enum class Color;//Correct
```

而且原来的版本（C++98）不能事先声明枚举类型，这就会增加编译依赖性，如果在原来的版本中需要引入一个新的状态，整个系统可能都会被重新编译。而在C++11中，这个问题得到解决，如果Status定义被修改，但是修改部分没有被用到的地方，实现就不用重新编译

```cpp
enum class Status;//前置声明

void continueProcessing(Status s);//使用前置声明的枚举体
```

此外对于有作用域的枚举体，默认的潜在类型是`int`，如果默认不适合，可以手动重载

```cpp
enum class Status;//潜在类型int
enum class Status : std::uint32_t;//改变潜在类型

//在定义处重载
enum class Status : std::uint32_t{
    good = 0, indeterminate = 0xFFFFFFFF};
```

但是原来的`enum`并不是一无是处：

假设我们的`tuple`包含了姓名，email和影响力，我们在这里想要email（即第一个域的值），但是这样做的可读性非常差，需要程序员自己记忆每个域是什么：

```cpp
using UserInfo = std::tuple<std::string, std::string, std::size_t>;

UserInfo uInfo;
	
auto val = std::get<1>(uInfo);//得到元祖类型的第一个域的值

//改进方法就是在前面加入
enum UserInfoFIEID {uiName, uiEmail, uiReputation};
//然后调用时1用uiEmail取代
auto val = std::get<uiEmail>(uInfo);

//如果用枚举类就显得冗余的代码
enum class UserInfoFields { uiName, uiEmail, uiReputation};
//调用时需要类型转换
auto val = std::get<static_cast<std::size_t>(UserInfoFields::uiEmail)>(uInfo);
```

这里`std::get`是一个模板，我们需要提供一个模板参数，所以需要在编译前就要把枚举元素转换成`std::size_t`，也就是说这个转换函数必须是`constexpr`的函数，模板是`constexpr`函数模板。

我们需要让它对任何类型的枚举体有效，我们就需要返回一般化的值类型，而不是单单的`std::size_t`，这通过`std::underlying_type`来实现，最后这个声明必须是要`noexcept`来保证这个函数永不会导致异常，它的功能就是接受任何的枚举元素，返回这个元素在编译阶段的常数值：

```cpp
template<typename E>
constexpr typename std::underlying_type<E>::type toUType(E enumerator) noexcept
{
    return static_cast<typename std::underlying_type<E>::type>(enumerator);
}

//C++14中引入返回值类型推导后更加优雅
template<typename E>
constexpr auto toUType(E enumerator) noexcept
{
    return static_cast<typename std::underlying_type<E>::type>(enumerator);
}

//具体使用
auto val = std::get<toUType(UserInfoFields::uiEmail)>(uInfo);
```

虽然上面的使用还是比没有作用域的`enum`要复杂，但是可以避免命名空间污染和枚举元素隐式转换等问题，所以值得开发者斟酌

## 要记住的东西

+ C++98 风格的`enum`是没有作用域的`enum`
+ 有作用域的枚举体的枚举元素仅仅对枚举体内部可见。只能通过类型转换（ cast ）转换
  为其他类型
+ 有作用域和没有作用域的`enum`都支持指定潜在类型。有作用域的`enum`的默认潜在类型
  ` int`。没有作用域的`enum`没有默认的潜在类型。
+ 有作用域的`enum`总是可以前置声明的。没有作用域的`enum`只有当指定潜在类型时才可
  以前置声明。