<h3>类型</h3>

类型的选择：
1. 明确知道非负数时选择无符号类型
2. 整数运算使用int，超过就选long long
3. 算术表达式中不要用bool或者char。因为char在不同机器上对于是否有符号实现是不同的，如果需要一个小整数，就明确使用unsigned char或者signed char
4. 浮点运算就使用double

类型转换：
1. 如果赋给无符号类型一个超过其范围的值时，就额外取模，如256赋予unsigned char，所得为256 % 255 = 1；同理，-1 为 255。
**此外需要额外小心的是：当赋给带符号的类型一个超出范围的值时是UB行为**
2. 易犯错误：当一个算术表达式中又有无符号数又有int时，会把int转化成无符号数，**切勿混用有符号类型与无符号类型**
```cpp
    unsigned u = 10;
    int i = -42;
    std::cout << u + i << std::endl;//int是32位的答案为4294967264
    std::cout << i - u << std::endl;//答案为4294967244
    std::cout << i * u << std::endl;//答案为4294966876
```

<h3>初始化：</h3>
C++中，初始化与赋值操作完全不同，需要严格区分

**C++11**：花括号初始化变量（列表初始化 list initialization）已经被广泛使用，**优点是当初始值存在信息丢失风险时编译器会报错**，以下内容都是正确的初始化：
```cpp
//C++ 11
int a{0};
int c = {0};
//传统：
int b(0);
int d = 0;
```
新旧初始化的区别：
```cpp
double pi = 3.1415926;
int a{pi},b = {pi}; //编译器抛出ERROR，存在信息丢失风险
int c(pi),d = pi;   //正确执行且的确丢失了部分信息
```
变量只能够被定义一次， `extern` 可以让我们只是声明而不是定义，但如果包含了显式初始化就转为了定义： `extern double pi = 3.1415;` 在函数体内部试图初始化一个`extern`标记的变量会引发错误

<h3>引用：</h3>
左值引用：为对象起另外一个名字（就是起一个别名），引用必须初始化，这个过程中引用的值会与原来的值**绑定**，而且无法令引用重新绑定到另一个对象；此外，引用不是对象，不能构建引用的引用（这部分内容的例外可以参考引用折叠，可以间接构建起引用的引用，前提必须是类型别名或者模板参数，见P608~612；此外，**You are forbidden from declaring references to references, but compilers may produce them inparticular contexts.**）

```cpp
int ival = 1024;
int &refval = ival;
int &refval;    //ERROR
```
右值引用**C++11**：为了支持move操作，必须绑定到右值的引用。通过`&&`来获得右值引用。它只能绑定到一个将要销毁的对象，所以可以自由地将一个右值引用的资源“移动”到另一个对象

左值：对象的身份（在内存中的位置）
右值：对象的值（对象的内容）
原则：需要右值的地方可以用左值来代替，但是不能把右值当成左值来使用

<h3>指针：</h3>

引用不是对象，没有实际地址，所以不能定义指向引用的指针`int &*p`这种，但是**指向指针的引用存在**，因为指针是对象（离变量名最近的符号对变量类型有最直接的影响（下文ptr1为符号&，因此ptr1是一个引用，其余部分确定引用的是什么））

```cpp
    int iVal=1024;
    int &refVal=iVal;
    int *ptr0 = &refVal;
    int *&ptr1 = ptr0;      //ptr1是对于指针ptr的引用
    int &*ptr2 = ptr0;      //ERROR: 'ptr2' declared as a pointer to a reference of type 'int &'
    int *&ptr3 = &refVal;   //ERROR: Non-const lvalue reference to type 'int *' cannot bind to a temporary of type 'int *'
```
解引用只能对确实指向了某个对象的指针进行，否则会引发段错误

<h3>空指针：</h3>

**C++11** `nullptr`特殊类型的字面值，可以被转化为任意其他的指针类型，这一点与传统的`NULL`不同，C++中，NULL被定义成0，显然这样会在某些情况下发生到底是普通整数还是指针的模糊，所以现在需要尽可能使用nullptr，抛弃NULL<br>
比较`nullptr`和`NULL`区别:万一有重载或者模板推导的时候，使用NULL编译器就无法给出正确结果了，比如

```cpp
void f(void *){}
void f(int){}
int main()
{
    f(0); // what function will be called?
}
```
而且用nullptr可以更好地进行异常的捕获与处理

<h3>const问题：</h3>

默认情况下，const对象设定为仅在文件内有效，当多个文件中出现了同名const变量，等同于不同文件中分别定义了独立的变量（编译时就直接发生替换）当真的需要跨文件进行使用共享，就把这个变量无论定义还是声明都加上`extern`，如`extern const int bufSize = 10;//a.c`和`extern const int bufSize;//a.h`，那么这两个变量就是一个的共享

可以把引用绑定到const对象上，但是要对这个引用也要有修饰词`const`，不然就可以通过这个引用修改`const`的值，这样显然是不合理的，但是`const`的引用可以绑定非`const`的对象

```cpp
const int iVal = 1024;
const int &ref1 = iVal;
int &ref2 = iVal;//ERROR: Binding value of type 'const int' to reference to type 'int' drops 'const' qualifier

int i = 12;
const int &ref3 = i;//Correct
/*可以这样理解：C++避免让你用别名的方式修改一个原来是只读的变量
* 但是如果你是想让这个别名只读，那随便你，而且这个别名也不会影响原来数的const与否
*/
```
需要理解的一个ERROR

```cpp
int iVal = 1024;
const int &ref1 = iVal;
const int &ref2 = ref1 * 2;
int &ref3 = ref1 * 2;//ERROR: Non-const lvalue reference to type 'int' cannot bind to a temporary of type 'int'
```
因为当const的引用`const int &ref = dval`编译器会进行这样的操作，也就是ref绑定了一个临时量而不是dval，那么当ref不是const时，只能改变绑定的temp的值，而对dval无能为力，这是不符合逻辑的，所以将其定义为非法

```cpp
const int temp = dval;//double dval = 3.14;也就是这句话进行类型转换
const int &ref = temp;//绑定的是temp，不是dval
```
**把`*`放在`const`之前与之后的区别**:放在后面是底层const，放在前面是顶层const

```cpp
int errNum = 0;
const int *curErr0 = &errNum;   //不能通过curErr0修改errNum的值，这是一个底层const (*curErr0)++;//ERROR
int *const curErr1 = &errNum;   //不能修改curErr1指向，这是一个顶层const int i =10; curErr1 = &i;//ERROR
```
**C++11**关键词`constexpr`：声明此类型可以让编译器来验证变量的值是不是一个常量表达式。声明为`constexor`的变量一定是常量，而且必须要用常量表达式（值不变且在编译器就计算完成的表达式）初始化。如果你认定一个变量是常量表达式，就用声明成`constexpr`类型，`constexpr指针`不能指向局部变量（不是固定地址）
```cpp
constexpr int size(){return 10;}    //constexpr函数

int main()
{
    const int sz0 = old_size();     //如果具体值要在运行时才知道，就不是常量表达式
    constexpr int mf = 10;
    constexpr int limit = mf + 10;
    constexpr int sz = size();      //只有size是constexpr函数才正确

    //指针和constexpr
    const int *p = nullptr;         // p是指向常量的指针
    constexpr int *q = nullptr;	    // q是常指针，constexpr仅对指针有效，直接置为顶层指针

}
```
`constexpr`函数：要求函数的返回类型与形参的类型都是字面值类型（包括算数类型、引用、指针、枚举等），且函数体中有且只有一条return语句，编译时隐式指定为内联函数，通常与内联函数一起定义在头文件中为了保证多个定义的一致性。 注意：constexpr函数不一定返回常量表达式，但是要确保编译时就能够确定下来

<h3>处理类型</h3>

类型别名：
传统：`typedef` 
**C++11：** 使用“别名声明”(alias declaration)`using SI = Sales_item;`，正常使用区别不大，优势主要在模板别名等上面，此外能够很好提升程序可读性，具体可见蓝色大大的专栏 https://zhuanlan.zhihu.com/p/21264013 在支持C++11上推荐`using`代替传统的`typedef`<br>
```cpp
//可读性提升，讲int[4]重命名成为int_array
using int_array = int[4];
typedef int int_array[4];
```

**警告：使用类型别名的语句，尝试把类型别名简单替换进行理解的方法是错误的！**
```cpp
//下面是上面typedef简单展开，但是这两者的语意是完全不同的
typedef char *pstring;
const pstring cstr = 0; //基本数据类型是指针，指向char的常量指针，即这个cstr的值不能改变 (*cstr)++;执行没有问题。
//静态检查出现如下提示：Clang-Tidy: 'cstr' declared with a const-qualified typedef type; results in the type being 'char *const' instead of 'const char *'

const char *cstr1 = 0;  //基本数据类型是char，cstr所指的位置的值不能改变， (*cstr1)++;报错
```

**C++11**： `auto`自动类型推导，对const忽略顶层，保留底层const
```cpp
int i = 0;
const int ci = i, &cr = ci;
auto b = ci, c = cr;    //auto推导得int
auto d = &ci;           //指向整数常量的指针，auto推导得到const int *，保留了底层const

const auto f = ci;      //若要保留顶层const，需要自己手动添加

//符号&和*只是辅助auto推导得到你想要的类型，只从属于某个声明符而非基本数据类型的一部分，auto那部分必须一行是同一个类型
auto &g = ci;           //可以将引用类型设为auto，这样初始值中的顶层const会保留，此处g推导出来为const int &
auto &h = 42;           //ERROR: 42是一个int，推导得到h为int &，显然可以想到不可能有 non-const reference bind the literal type，否则就可以通过h这个别名篡改42这个常量，改成const auto &j = 42;就对了
auto &n = i, *p2 = &ci; //ERROR: 前者推导得到类型是int，后者得到const int，不一致

```
**C++11**：`decltype关键词` 从表达式的类型推导出要定义的变量的类型，但不要进行初始化，对表达式不会进行计算。在用`decltype`进行推导时，是自动完全copy整个推导出来的类型（包括顶、底层const，引用等）

此外，引用在这里不是所指的对象的同义词（尽管在其他C++的地方都是如此）
```cpp
int i = 42, *p = &i, &r = i;
decltype(r + 0) b;  //正确，加法的结果是int，就推导出int
decltype(*p) c;     //ERROR: Declaration of reference variable 'c' requires an initializer
```
**这里\*是指针解引用的操作，那么解出来的就是一个引用&，所以`decltype`推导出来的结果是int&**<br>
此外，**`decltype`的括号一个与两个是完全不同的，**`decltype((variable))`得到的永远是引用，而`decltype(variable)`的结果只有当variable本身是引用才是引用

<h3>自定义数据结构</h3>

**C++11**可以为类的数据成员提供一个**类内初始值**用以初始化数据成员
```cpp
struct Sales_data
{
    std::string bookNo;     //空字符串
    unsigned units_sold = 0;//初始化为0
};
```
编写自己的头文件，类、const、constexpr变量应该包含在头文件中，而且类所在的头文件名字应该与类名相同以区分
```cpp
//不要偷懒，头文件都加上头文件保护符以避免反复包含
#ifndef XXX_H
#define XXX_H
...
#endif
```
