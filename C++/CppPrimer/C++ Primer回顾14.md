<h2>标准库特殊设施</h2>

**C++11** 

tuple类型

是类似`pair`的模板。`pair`每次都有两个成员，而`tuple`可以有任意数量的成员。

|命令|含义|
|:-:|:-:|
|`tuple<T1,T2,...,Tn> t(v1,v2,...vn);`|t是一个`tuple`，成员类型是T1...Tn，每个成员用对应的v1...vn进行初始化，这个构造是显式构造的|
|`make_tuple(v1,v2,...,vn)`|返回一个用于给定初始值初始化的tuple，类型由初始值推断得到|
|`t1 == t2`或`t1 != t2`或比较|两个具有相同数量成员且成员对应相等时，两个tuple才相等，只有每对成员的`<`运算符都合法，比较才合法|
|`get<i>(t)`|返回t的第i个数据成员的引用；如果t是左值，结果是左值引用；否则返回右值引用|
|`tuple_size<tupleType>::value`|一个类模板，可通过一个tuple类型进行初始化。有一个名为value的public constexpr static成员，类型size_t，表示给定tuple类型中成员的数量|
|`tuple_element<i,tupleType>::type`|一个类模板，可以通过一个整型常量和一个tuple类型进行初始化，有一个type的public成员，表示给定tuple类型中指定成员的的类型|

```cpp
tuple<size_t, size_t, size_t> threeD;//三个成员都默认初始化为0
tuple<string, vector<double>, int, list<int> > someVal("constants",{3.14,2.718},42,{0,1,2,3,4});//构造函数是显式的，必须直接初始化语法
auto item = make_tuple("0-999-78345-X",3, 20.00);//类型make_pair直接生成
```

对比pair，因为严格规定了只有两个成员，所以可以提供命名为`.first`和`.second`，而tuple类型就没有限制，这导致了我们必须给定显式模板实参告诉编译器我们要访问第几个成员，返回的是指定成员的引用

```cpp
auto item = make_tuple("0-999-78345-X",3, 20.00);
auto isbn = get<0>(item);
auto price = get<2>(item);
get<2>(item) *= 0.8;
```

如果不知道tuple准确类型，就可以使用`tuple_size`和`tuple_element`进行查询

```cpp
typedef decltype(item) trans;
size_t sz = tuple_size<trans>::value;
tuple_element<1,trans>::type cnt = get<1>(item);
```

一般用于使用tuple返回多个值

```cpp
class Sale_data;

typedef tuple<vector<Sale_data>::size_type,
		vector<Sale_data>::const_iterator,
		vector<Sale_data>::const_iterator> matches;

vector<matches>
findBook(const vector<vector<Sale_data>> &file, const string &book)
{
	//file代表每家书店的销售记录，
	//返回一个vector，每家销售了给定书籍的书店在其中都有一项
	//...
}
```

<hr>

bitset类

方便位运算，同时能够处理超过最长整形的类型大小的位集合

|命令|含义|
|:-:|:-:|
|`bitset<n> b;`|b有n位，每一位都是0|
|`bitset<n> b(u);`|b是一个unsigned long long(ull)值的u低n位的拷贝，若n大于ull的大小，则b中超出部分的高位为0|
|`bitset<n> b(s,pos,m,zero,one);`|b是string s从位置pos开始m个字符的拷贝，s只能包含字符zero或者one，若包含其他字符就抛出异常；b默认是string::npos，pos默认是0；zero默认是'0'，one默认是'1'|
|`bitset<n> b(cp,pos,m,zero,one);`|与上一个相同，但是从cp指向的字符数组中拷贝字符|

```cpp
bitset<13> bitvec1(0xbeef);//比初始值小，初始值中高位抛弃，得到1111011101111
bitset<20> bitvec2(0xbeef);//比初始值大，高位补0，得到00001011111011101111

string str("1111111000000011001101");
bitset<32> bitset3(str,5,4);//从str[5]开始取4个二进制位，为1100
bitset<32> bitset4(str,str.size()-4);//使用最后4个字符
```

注意string是从左向右读的，bitset的二进制是从右向左读的，所以string最右边的字符初始化的是bitset的低位

|部分命令|含义|
|:-:|:-:|
|b.set(pos,v)|将pos的位设置为bool值v，v默认为true，如果未传递实参，则将b中所有位置位|
|b.reset(pos)|将pos的位复位，如果未传递实参，则将b中所有位复位|
|b.flip(pos)|将pos的位翻转，如果未传递实参，则将b中所有位翻转|
|b[pos]|访问b中pos处的位|
|`b.to_ulong()`/`b.to_ullong()`|返回一个位模式与b相同的unsigned long或者unsigned long long，如果不能放入这个类型，抛出异常|
|b.to_string(zero, one)|返回一个string，表示b中的为模式，zero和one默认值为0和1|
|`os << b`/`is >> b`|将b中二进制位打印为0或1到输出流os中；从is中读取字符存入b，如果下一个字符不是1或0时，或者已经达到b.size()值的时候，读取过程停止|

<hr>

正则表达式

定义在头文件`<regex>`中

|命令|含义|
|:-:|:-:|
|regex|表示有一个正则表达式类|
|regex_match|将一个字符序列与一个正则表达式匹配|
|regex_search|寻找第一个与正则表达式匹配的子序列|
|regex_replace|使用给定的格式替换一个正则表达式|
|sregex_iterator|迭代器适配器，调用regex_search来遍历一个string中所有匹配的子串|
|smatch|容器类，保存在string中的搜索结果|
|ssub_match|string中匹配的子表达式的结果|

```cpp
string pattern("[^c]ei");//查找不在c之后的ei
//[[:alpha:]]匹配任何字母
pattern = "[[:alpha:]]*" + pattern + "[[:alpha:]]*";//需要包含pattern的整个单词
regex r(pattern);
smatch results;

string test_str = "receipt freind theif receive";//定义一个string的保存与模式匹配和不匹配的文本
if (regex_search(test_str, results, r))//用r在test_str中查找与pattern匹配的子串，找到第一个匹配就返回
	cout << results.str() << endl;
```

正则表达式在运行时编译，而且很慢，需要尽量不要创建不必要的regex以减小开销（例如：在循环中使用时，应该在循环外创建它，而不是每步迭代都编译它）

简单来说可以：

|命令|含义|实例|匹配结果|
|:-:|:-:|:-:|:-:|
|`.`|匹配除了换行符"\n"之外的字符|`a.c`|`abc`|
|`\`|转义|`a\.c`|`a.c`|
|`[...]`|字符集，可以对字符集单独列，也可以给范围，如`[abc]`也可以表示成`[a-c]`，`^`表示取反，如`[^abc]`表示`不是abc的其他字符`<br>在字符集中使用`]`或`-`或`^`需要加上反斜杠`\`|`a[b-d]e`|`abe,ace,ade`|
|
|`\d`|数字`[0-9]`|`a\dc`|`a1c`|
|`\D`|非数字`[^\d]`|`a\Dc`|`abc`|
|`\s`|空白字符`[<空格>\t\r\n\f\v]`|`a\sc`|`a c`|
|`\S`|空白字符`[^\s]`|`a\Sc`|`abc`|
|`\w`|单词字符`[A-Za-z0-9_]`|`a\wc`|`abc`|
|`\W`|非单词字符`[^\w]`|`a\Wc`|`a c`|
|`\f`/`\r`/`\n`|匹配换页符，回车符，换行符|||
|`*`|匹配前一个字符0次或无限次|`abc*`|`ab` `abccc`|
|`+`|匹配前一个字符1次或无限次|`abc+`|`abc` `abccc`|
|`?`|匹配前一个字符0次或1次|`abc?`|`ab` `abc`|
|`{m}`|匹配前一个字符m次|`ab{2}c`|`abbc`|
|`{m,n}`|匹配前一个字符m至n次，省略m，匹配0~n次，省略n，则匹配m至无限次|`ab{1,2}c`|`abc` `abbc`|
|`^`|匹配字符串开头，在多行模式中匹配每一行的开头|`^abc`|`abc`|
|`$`|匹配字符串末尾，在多行模式中匹配每一行的末尾|`abc$`|`abc`|
|`\A`|仅匹配字符串开头|`\Aabc`|`abc`|
|`\Z`|仅匹配字符串末尾|`abc\Z`|`abc`|
|`\b`|匹配`\w`和`\W`之间(单词字符与非单词字符之间)|`a\b!bc`|`a!bc`|
|`|`|匹配左右表达式中任意一个，短路原则|`abc|def`|`abc` `def`|
|`(...)`|分组|(abc){2}a(123|456)c|`abcabca123c`|
|`(?P<name>...)`|分组，同时提供别名|`(?P<id>abc){2}`|`abcabc`|
|`\<number>`|引用编号为`<number>`的分组匹配到的字符串|`(\d)abc\1`|`1abc1`|

ECMAScript正则表达式语言特性

  +  `\{d}`代表单个数字，`\{d}{n}`代表n个数字序列
  +  `[-. ]`代表匹配字符集`-`,`.`和` `的集合
  +  后接`?`的组件表示可选的（比如`\{d}{3}[-. ]?\{d}{4}`，匹配开始是3个数字，然后是短横或点或空格，后面又是4个数字（如555-0123）
  +  反斜线是C++特殊字符，所以模式中每次出现`\`的地方都要额外在前面加一个`\`告诉C++这里我们需要的是反斜线而不是一个特殊符号，也就是说，我们用`\\{d}{3}`来表达正则表达式`\{d}{3}`

使用`sregex_iterator`

```cpp
bool vaild(const smatch &m)
{
	//如果区号前有左括号
	if(m[1].matched)
		//区号后必须有右括号，之后紧跟剩余号码或者一个空格
		return m[3].matched && (m[4].matched == 0 || m[4].str() == " ");
	else
		//否则就不能有右括号，而且另外两个组成部分的分隔符必须匹配
		return !m[3].matched && m[4].str() == m[6].str();
}

	//查找所有违反“i在e之前，除非在c之后”规则的单词
	string file;
	string pattern("[^c]ei");
	pattern = "[[:alpha:]]*" + pattern + "[[:alpha:]]*";
	regex r(pattern, regex::icase);
	for (sregex_iterator it(file.begin(), file.end(), r), end_it; it != end_it; ++it)
		cout << it->str() << endl;
```

使用子表达式

```cpp
string phone = "(\\()?(\\d{3})(\\))?([-. ])?(\\d{3})([-. ]?)(\\d{4})";
regex r(phone);
smatch m;
string s;
while(getline(cin,s))
{
	for (sregex_iterator it(s.begin(), s.end(), r), end_it; it != end_it; it++)
	{
		if(vaild(*it))
			cout << "vaild: " << endl;
		else
			cout << "unvaild: " << endl;
	}
}
```

替换正则表达式`regex_replace`

```cpp
//把号码格式改成ddd.ddd.dddd
string phone = "(\\()?(\\d{3})(\\))?([-. ])?(\\d{3})([-. ]?)(\\d{4})";
string fmt = "$2.$5.$7";
regex r(phone);
string number = "(908) 555-1800";
cout << regex_replace(number, r, fmt) << endl;
```

<hr>

随机数

在头文件`<random>`中，随机数引擎和随机数分布类共同作用；随机数引擎是函数对象类，定义一个调用运算符，不接受参数并且返回一个随机的unsigned整数

```cpp
default_random_engine e;
for (size_t i = 0; i < 10; ++i)
	cout << e() << endl;
```

|命令|含义|
|:-:|:-:|
|Engine e(s);|使用整型值s作为种子|
|e.seed(s)|使用种子s重置引擎状态|
|e.min()/e.max()|这个引擎可生成的最大/最小值|
|Engine::result_type|这个引擎生成的unsigned整型类型|
|e.discard(u)|将引擎推进n步|

这些数字一般是不能直接使用的，称为原始随机数

为了得到一个指定范围的数，应该使用分布类型对象，下面生成0~9之间（包含）均匀分布的随机数

```cpp
uniform_int_distribution<unsigned> u(0, 9);
default_random_engine e;
for (auto i = 0; i < 10; i++)
	cout << u(e) << endl;//传递的是引擎本身，而不是引擎的值
```

注意特性：对于给定的种子，得到的随机数序列一定是相同的，所以以下的看似trivial的用法是严重错误的，因为这样你得到的vector每次都是相同的

```cpp
vector<unsigned> bad_randVec()
{
	default_random_engine e;
	uniform_int_distribution<unsigned> u(0,9);
	vector<unsigned> ret;
	for (auto i = 0; i < 100; i++)
		ret.push_back(u(e));
	return ret;
}
```

正确的方法是将引擎和关联的分布对象定义为static的来保持住状态，即

```cpp
static default_random_engine e;
static uniform_int_distribution<unsigned> u(0,9);
```

生成随机实数：`	uniform_real_distribution<double> u(0,1);`包含0,1的均匀分布，这里的模板参数默认为double

还可以生成非均匀分布的随机数，`	normal_distribution<> n(4, 1.5);`均值为4，标准差为1.5的正态分布

还可以进行伯努利分布，这个分布不接受模板参数，返回一个bool，这个概率默认值是0.5，`bernoulli_distribution b;`也可以通过`bernoulli_distribution b(0.55)`改变概率为55/45，使用方法如上面的例程所示