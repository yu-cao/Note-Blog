<h2>顺序容器</h2>

|名称|功能|
|:-:|:-:|
|vector|可变大小数组，连续内存，随机访问，除尾部外其他位置插入或删除元素缓慢|
|deque|双端队列，随机访问，两端添加与删除速度快|
|list|双向链表，任何位置添加或删除速度都很快，但不支持随机访问，只能遍历整个容器|
|forward_list|单向链表，不支持sizeof操作；**C++11**|
|array|固定大小数组，大小固定，不支持对容器大小的改变；**C++**|
|string|与vector相似，但专门保存字符|

选择容器基本原则：

+ 通常，使用`vector`是最好的选择
+ 有很多小元素，且空间开销很重要，不要用`list`和`forward_list`
+ 要随机访问，用`vector`或`deque`
+ 要频繁在容器中间插入/删除元素，用`list`或`forward_list`
+ 要频繁在容器头尾插入/删除元素，用`deque`
+ 在读取输入时才需要在中间插入，随后要支持随机访问：向`vector`追加数据，再通过`sort()`进行排序；或者先存到`list`，再拷贝到`vector`中

迭代器左闭合区间:[begin,end)，当begin == end时，范围为空，end不能位于begin之前。

反向迭代器：`auto it = a.rbegin();`各种操作含义出现颠倒，例如：使用++就会得到上一个元素

`a.cbegin();`返回const\_iterator；
`a.crbeing()`返回const\_reverse\_iterator

在不需要“写”操作的时候，应使用cbegin与cend来保护

容器间的拷贝：两个容器的类型及其元素类型必须匹配（**除非当传递迭代器参数，拷贝一个范围（范围初始化**）

```cpp
list<string> authors  = {"Milton","Shakespeare","Austen"};
vector<const char*> articles = {"a","an","the"};

list<string> list2(authors);
deque<string> authList(authors);//错误：容器类型不匹配，list与deque类型不一致
vector<string> words(articles);//错误：元素类型不匹配，const char*与string类型不一致
forward_list<string> word(articles.begin(),articles.end());//正确，拷贝范围时不再严格要求类型，const char*可以转成string，vector可以转成forward_list
```

array：需要同时定义元素类型与大小，比如`array<int, 42>`，只要array的类型匹配（array的类型包括元素类型与大小），拷贝即是合法的

```cpp
array<int, 5> digits = {1,2,3,4,5};
array<int, 5> copy = digits;// 正确
array<int, 6> copyError = digits;// 不匹配
```

assign（仅顺序容器，array除外）：

```cpp
list<string> names;
vector<const char*> oldstyle;
names = oldstyle;//ERROR，容器类型不匹配
names.assign(oldstyle.cbegin(),oldstyle.cend());//用所指定的元素的拷贝替换左边容器中的所有元素
names.assign(10,"Hi");//assign重载版本，10个元素，每个都是"Hi"
```

swap：交换两个相同类型容器的内容（实质是交换了两个容器内部的数据结构，常数时间内完成；array除外，array将真正调换元素，O(n)）统一使用非成员版本

添加元素（array不支持）：

`push_back/push_front(t)`（在容器末尾/前面添加值为t的元素）<br>
**C++11**:`emplace_back/emplace_front(args)`（在容器末尾/前面添加由args创建的元素）<br>
在前面添加元素不支持vector和string两个容器，在后面添加元素不支持list容器；forward_list有专门的`insert`和`emplace`

insert（都支持，array除外）:添加到迭代器所指的**之前**的位置

```cpp
//后面三种若添加为空，则返回原先的迭代器p
c.insert(p,t);// 在迭代器p指向的元素之前插入一个值为t的元素，返回指向新添加的元素的迭代器
c.insert(p,n,t);// 在迭代器p指向的元素之前插入n个值为t的元素，返回指向新添加的第一个元素的迭代器
c.insert(p,b,e);// 在迭代器p指向的元素之前插入迭代器b和e指定的范围内的元素。（注意：b和e不能指向c中元素）返回指向新添加的第一个元素的迭代器
c.insert(p,il);// il是花括号包含的一个元素值列表。返回指向新添加的第一个元素的迭代器
```

**C++11**：emplace与push辨析：emplace是将参数传递给元素类型，不再构造临时对象，而是在容器管理的内存空间中直接创建对象（参数必须与元素类型的构造函数相匹配），而push是先构造一个临时局部对象再进行插入

```cpp
//c是一个元素类型是Sales_data的容器
c.emplace_back("978-0590353403",25,15.99);
c.push_back(Sales_data("978-0590353403",25,15.99));
c.emplace(iter,"999-999999999");//使用Sales_data(string)，iter指向c中一个元素，插入在iter之前，返回指向新添加的元素的迭代器（通过变参模板和完美转发实现）
```

back、front和at(n)：返回容器中尾元素，首元素和下标为n的元素的引用

删除元素（除array外）：必须确保删除元素是存在的，这是程序员的责任

`pop_front`：vector与string不支持
`pop_back`：`forward_list`不支持，同时`forward_list`有特殊版本的`erase`，pop操作的返回值为void，如果要使用，必须先保存

普通的erase:

```cpp
c.erase(p);//删除迭代器p所指元素，返回一个指向被删元素之后元素的迭代器，若p指向尾元素，则返回尾后迭代器，若p是尾后迭代器，行为未定义！
c.erase(b,e);//删除迭代器b和e所指定范围内的元素，返回一个指向最后一个被删元素后的元素的迭代器，若e是尾后迭代器，则函数返回尾后迭代器
c.clear();//删除c中的所有元素，返回void
```

forward_list的特殊插入与删除：

`insert_after`，`emplace_after`，`erase_after`如果要对一个位置进行插入，删除等，需要一个指向这个位置前面元素的迭代器执行内部函数，为了能够在链表头进行操作，定义了`before_begin`，返回一个首前迭代器，允许在首元素之前，不存在的元素“之后”进行插入/删除<br>
其余操作与前面的几乎类似，以下列举几个稍微有点不同的

```cpp
lst.insert_after(p,b,e);//在p之后（注意）插入表示范围的b和e这一对迭代器
lst.erase_after(b,e);//删除迭代器[b,e)所指定范围内的元素（注意：不包含e指向的元素）,返回一个被删元素之后元素的迭代器

//实例，删除forward_list<int>中的奇数元素
#include <iostream>
#include <forward_list>

using namespace std;

int main()
{
    forward_list<int> iflst = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9,};

    auto prev = iflst.before_begin();
    auto curr = iflst.begin();

    while (curr != iflst.end())
        if (*curr & 1)//奇数
            curr = iflst.erase_after(prev);
        else
        {
            prev = curr;
            curr++;
        }
    for(curr = iflst.begin(); curr != iflst.end(); curr++)
        cout << *curr << " ";
}
```

改变容器的大小，resize()（不适用于array）注意：不会改变容器的占用的内存大小，只改变元素的数目

```cpp
list<int> ilist(10,42);//10个int，每个值为42
ilist.resize(15);//增加5个值为0的元素加在末尾
ilist.resize(25,-1);//增加10个值为-1的元素加在末尾
ilist.resize(5);//从链表末尾删除20个元素
```

当容器大小出现改变后，必须要重新定位迭代器在string、deque和vector中；同时，不要保存end返回的迭代器，因为end迭代器在大小改变后总会失效，而应该不断重新调用

对容量管理的成员函数（无特别提醒，默认为适用于vector，string）

`c.shrink_to_fit()`：请求将capacity()减小为与size()相同大小（也可用于deque），标准库不保证退还内存<br>
`c.capacity()`：不重新分配内存空间的话，c可以保存多少元素<br>
`c.reserve(n)`：分配至少能容纳n个元素的内存空间，如果小于当前容量，reserve什么都不做，而不会退回空间

辨析：size与capacity，size是容器已经保存的元素的数目，capacity是不分配新的内存空间的前提下最多可以保存的元素

<h3>string的额外特性与处理：</h3>

构造：

```cpp
const char *cp = "Hello World!!!";
string s0(cp);
string s1(s0,6);//从s0[6]开始拷贝至s0末尾
string s2(s0,6,5);//从s0[6]开始拷贝5个字符，最长拷贝到s0.size()-6;
string s3(cp+6,5);//从cp[6]开始拷贝5个字符
string s4(s0,16);//抛出异常：out_of_range
```

`s.substr(pos,len)`或者`s.substr(pos)`，最多只能拷贝到s的末尾

string中额外的insert与erase：

```cpp
s.insert(s.size(),5,'!');//s末尾加入5个！
s.erase(s.size()-5,5);//从s末尾删掉5个字符
```

string中的append和replace：`append`是在末尾进行插入操作的简写形式，即`s.insert(s.size(),"Test");`等价于`s.append("Test");`;而`replace`是调用erase和insert的简写形式；replace(range,args)中range可以是位置+长度，也可以是一对迭代器

```cpp
s.erase(11,3);
s.insert(11,"test");
//等价于：
s.replace(11,3,"test");//删掉3个字符，插入字符test
```

搜索：

+ `find`：查找参数指定的字符串，找到返回地换一个匹配位置的下标，否则返回npos
+ `find_first_of`：查找第一个"与给定字符串中任何一个字符匹配的"字符的位置
+ `find_first_not_of`：查找第一个不在参数中的字符
+ 类似的还有`find_last_of`, `find_last_not_of`,`rfind`（查找s中args最后一次出现的位置）
+ args必须为`c,pos`(从s[pos]开始找字符串c，默认pos=0)或`s2,pos`(从s[pos]开始找字符串s2，默认pos=0)或`cp,pos`(从s[pos]开始查找指针cp指向的C风格字符串，默认pos=0)或`cp,pos,n`(从s[pos]开始查找指针cp指向的数组的前n个字符，无默认值)

```cpp
string numbers("0123456789"), name("r2d2");
auto pos = name.find_first_of(numbers);//返回为1，即name中第一个数字的下标（name[1]）

string dept("03714p3");
auto pos = dept.find_first_not_of(numbers);//返回5，即'p'，它是第一个不与numbers中任何元素匹配的
```

比较：

|s.compare参数类型|作用|
|:-:|:-:|
|s2|比较s1和s2|
|pos1,n1,s1|从s[pos1]开始的n1个字符与s2比较|
|pos1,n1,s2,pos2,n2|从s[pos1]开始的n1个字符与s2[pos2]开始的n2个字符进行比较|
|cp|比较s与cp指向的以\0结尾的字符数组|
|pos1,n1,cp|从s[pos1]开始的n1个字符与cp指向的以\0结尾的字符数组比较|
|pos1,n1,cp,n2|从s[pos1]开始的n1个字符与cp指向的字符数组的n2个字符比较|

数值与字符转换：

to_string(val)和stoX(s,p,b)（X可以是i，l，ul，ll，ull，f，d，ld等，p为保存s中第一个非数值字符的下标，默认为0，即不保存下标；b为base，浮点数不可选，默认是10）

```cpp
string s2 = "pi = 3.14";
double d = stod(s2.substr(s2.find_first_of("+-.0123456789)));//d = 3.14
```

容器适配器：栈适配器、队列适配器、优先队列适配器。

栈适配器：默认基于deque，但也可以基于list和vector实现

```cpp
//除了演示的3种外，还有s.emplace(args)这种
int main()
{
    stack<int> intStack;

    for(size_t ix = 0;ix!=10;++ix)
        intStack.push(ix);
    while(!intStack.empty())
    {
        int value = intStack.top();
        intStack.pop();
    }
}
```

队列适配器：queue提供FIFO的存储与访问策略，默认基于deque实现，也可基于list实现；priority\_queue允许为队列中元素建立优先级，默认基于vector实现，也可基于deque实现

|操作|意义|
|:-:|:-:|
|q.pop()|移除queue的首元素或priority_queue最高优先级元素|
|q.front()|返回首元素|
|q.back()|返回尾元素，只适用于queue|
|q.top()|返回最高优先级元素，只适用于priority_queue|
|q.push(item)|在queue末尾或priority_queue适当位置创建一个元素|
|q.emplace(args)|参数由args构造，效果同push|