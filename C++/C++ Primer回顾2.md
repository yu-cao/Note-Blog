<h3>命名空间的using</h3>

语法：`using namespace::name`，比如使用string类型就
```cpp
#include <string>
using std::string;
```
合理使用标准库而不要轻易自己造轮子，标准库的性能也是有所保障的

<h3>标准库的string</h3>

`string::size_type` 它是`string::size()`的返回值，是一个与机器无关的无符号类型的值且足够放下任何string对象的大小，在C++11中可以通过auto或decltype来进行推断
```cpp
auto len = line.size();//len类型就是无符号的string::size_type
```
注意：无符号数与有符号数进行加减是高危的，必须要谨慎

单独处理string对象中的字符，可以选择cctype头文件来处理，建议尽量使用C++版本的C标准头文件，例如使用\<cstdio>,\<cctype>代替\<stdio.h>,\<ctype.h>

**C++11** 范围for语句：遍历给定序列中的每个元素并对序列中的每个值执行某中操作
```cpp
//统计整个字符串的标点
string s("Hello World!!!");
decltype(s.size()) punct_cnt = 0;
for(auto c : s)
    if(ispunct(c))
        ++punct_cnt;
cout << punct_cnt << " punctuation characters in " << s << endl;

//改变整个字符串中的字符
string s("Hello World!!!");
for(auto &c : s)    //这个引用c依次绑定到s的每个字符上
    c = toupper(c);
cout << s << endl;
```
注意不要使用越界的下标，这样的行为是未定义的！下标的值必须≥0 && ≤ size()，比较好的习惯是总是设下标的类型是`string::size_type`

<h3>标准库的vector</h3>

如果不清楚元素的确切个数，选择vector而不是数组。<br>
vector是一个类模板。引用不是对象，所以不存在包含引用的，vector永远在调用的时候是实例化成为一个具体的类，比如vector<int>::type_size才是正确的类型<br>
**C++11**：过去的vector元素还是vector需要`vector<vector<int> >`，C++11后只要`vector<vector<int>>`即可，不需要额外空格

初始化vector对象方法：
```cpp
//基本的默认初始化、拷贝构造等略
vector<T> v3(n,val) //v3包含了n个重复的元素，每个元素值都为val

vector<T> v4(n)     //v4包含了n个重复地执行了值初始化的对象
//例如vector<int> ivec(10);是10个元素，每个都被初始化为0
//vector<string> svec(10);是10个元素，每个都是空string对象
//也就是说，除非当元素是明确要求提供初始值且元素没有被设计有默认初始化的情况下时，否则这个工作都是奏效的

vector<T> v5{a,b,c...n} == vector<T> v5 = {a,b,c...n} //包含了n个元素，每个元素都被赋予了初始值(C++11特性：列表初始化)
```
注意，当使用花括号但是提供的值不能被用来初始化，编译器会尝试帮你把这样的值来构造vector对象，如果不行，就返回ERROR
```cpp
vector<string> v3(10);      //10个元素，每个都是0

vector<string> v4{10};      //10个元素，每个都是0
vector<string> v5{10,"hi"}; //10个元素，每个初始化为hi
```

vector对象能够高效增长，所以除非初始化成里面元素完全一样，否则更高效的做法是建立一个空的vector对象，然后往里面`.push_back()`具体值。但是注意：范围for循环不得改变其所遍历序列的大小，因为范围for语句会预存类似`.end()`的函数确认终点，而在其中改变会导致end的地方已经释放，这样的行为是UB的。

用数组可以初始化vector对象，只要指明拷贝区域的首尾指针即可
```cpp
int int_arr[] = {0, 1, 2, 3, 4, 5};
vector<int> ivec(begin(int_arr), end(int_arr));
```

<h3>迭代器</h3>

不必关心迭代器的类型，直接auto决定类型即可
```cpp
string s("Hello World!");
auto b = s.begin(), e = s.end();    //b，e就是迭代器
cout << *b;                         //返回迭代器所指元素的引用

//把Hello完全大写
for(auto it = s.begin();it != s.end() && !isspace(*it); ++it)
    *it = toupper(*it);
// 点评：for使用!=而不是<进行判断是为了标准库的泛型编程需要，这个操作在标准库的所有容器中都有效，
// 所有C++标准库都定义了!=和==，而且大多数却没有定义<，所以用C++要养成使用迭代器和!=的好习惯
```
begin()与end()的返回类型由对象是不是const决定，是则返回const_iterator，不是则返回iterator，但是如果我们希望对只需读操作的对象返回也为const_iterator<br>
**C++11**:`auto it3 = v.cbegin()`和`v.cend()`返回的一定是const_iterator，无论对象类型

**注意：但凡是使用了迭代器的循环体，都不要向迭代器所属的容器添加元素，这样会使得vector对象的迭代器失效，当修改了vector大小后，必须要重新定位迭代器**

迭代器可以进行`iter + n`这种操作使得其指向下第n个迭代器，也可以进行两个迭代器相减得到两者之间的距离，类型为`difference_type`（用auto直接推导比较好）

<h3>数组</h3>

标准库函数的begin与end<br>
尽管能够通过手算得到尾后指针，但是很容易出错，一旦出错就会出现缓冲区溢出的严重漏洞。为了让指针使用更简单，**C++11**:对数组引入begin与end函数（数组不是类，没有成员函数，这两个函数定义在\<iterator>的头文件中，具体操作如下）
```cpp    
int a[] = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
int *begin_a = begin(a);
int *end_a = end(a);
while (begin_a != end_a)
{
    cout << *begin_a << endl;
    begin_a++;
}
```

对于范围for语句处理的多维数组，除了最内层循环外，其他的所有循环控制变量都应该是引用类型（避免被auto推导成为指针）**注意：当没有特殊情况下，auto永远会把数组推导成指针**
```cpp
size_t cnt = 0;
int ia[3][4];
for (auto &row : ia)
    for (auto &col : row)
    {
        col = cnt;
        ++cnt;
    }
for (const auto &row : ia)
    for (auto col : row)
        cout << col << endl;

//第一层把row推导成为int*去分别指向第一个元素的地址，那么内层循环就不可能执行遍历操作，引发ERROR
for (auto row : ia)
    for (auto col : row)//ERROR: Invalid range expression of type 'int *'; no viable 'begin' function available
        cout << col << endl;

//使用begin与end
for (auto p = begin(ia); p != end(ia); ++p)//p类型：int(*)[]
    for (auto q = begin(*p); q != end(*p); ++q)//q类型：int *
        cout << *q << endl;
```
