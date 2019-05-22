<h2>泛型算法</h2>

在头文件\<algorithm>和\<numeric>中

只读算法：只会读取其输入范围内的元素，而从不改变元素。

+ find
+ count：统计给定值在序列中出现次数
+ accumulate：前两个迭代器给出求和范围，第三个给出和的初值
+ equal：`equal(roster1.cbegin(),roster1.cend(),roster2.cbegin());`即使是string和const char*也可以相互比较，只要能重载"=="运算符即可（假定第二个序列长度至少等于第一个序列，这是程序员的责任）

```cpp
string sum = accumulate(v.cbegin(),c.cend(),string(""));//必须显式定义空串使得满足"+"运算符
```

写容器元素的算法：把新值赋给序列中的元素**（算法不会执行容器操作，自身不可能改变容器大小）**

+ fill：`fill(vec.begin(),vec.end(),0);`把0赋给容器的每个元素
+ fill_n：`fill_n(dest,n,val);`：dest指向一个元素，从dest开始后n个元素赋值为val，至少保证后面n个元素都存在**（小心：千万不要在一个空容器中使用这个操作，对空容器插入元素需要如下代码所示：与back\_inserter组合使用）**
+ back_inserter：定义在\<iterator>头文件中，接收一个指向容器的引用，返回一个与该容器绑定的插入迭代器

```cpp
vector<int> vec;
auto it = back_inserter(vec);
*it = 42;//现在vec中有一个元素，值为42

//组合使用来添加元素
vector<int> vec;
fill_n(back_inserter(vec),10,0);//添加10个值为0的元素到vec中
```

拷贝算法：copy和replace

copy：前两个参数确定输入范围，第三个参数表示目的序列的起始位置<br>
replace：前两个参数确定输入范围，第三个参数表示要被替换的值，第四个参数表示替换成的值<br>
replace_copy：保留原序列的方法：相比较replace额外加入第三个参数back\_inserter(ivec)（新的vector）如：`replace_copy(ilst.cbegin(),ilst.cend(),back_inserter(ivec),0,42);`ilist并未改变，ivec包含了ilst的一份拷贝同时其中的val=0的元素在ivec中变成了42

排序算法：sort利用元素类型的<运算符实现排序

```cpp
int main(int argc, char *argv[])
{
    vector<string> words;
    string temp;
    while (in >> temp)
        words.push_back(temp);
    elimDups(words);
    for (auto &e : words)
        cout << e << " ";
    cout << endl;
}

void elimDups(vector<string> &words)
{
    sort(words.begin(), words.end());
    auto end_unique = unique(words.begin(), words.end());//unique使得不重复的部分排列在前部，返回一个指向最后一个不重复元素之后的位置
    words.erase(end_unique, words.end());
}
```

<h3>定制操作</h3>

可以向sort传递第三个参数：谓词(predicate)

```cpp
bool isShorter(const string &s1,const string &s2)
{
    return s1.size() < s2.size();
}
sort(words.begin(),words.end(),isShorter);//改成按长度排序单词
```

stable_sort：以升序排序范围 [first, last) 中的元素。保证保持等价元素的顺序。

partition(vec.begin(),vec.end(),predicate)：接收一个谓词，true的放在容器的前面，false放在容器的后面，返回一个指向最后一个使谓词为true的元素之后的位置

<h4>Lambda表达式</h4>

**C++11**：Lambda表达式表达一个可调用的代码单元，可以理解成未命名的内联函数。形式如下：

`[capture list](parameter list) -> return type {function body}`

capture list是一个lambda所在的函数中定义的局部变量的列表，通常为空，其他部分与普通函数类似，只是必须使用尾置返回的方式进行返回；捕获列表和函数体不可省略，省略参数列表默认为空参数列表，省略返回类型则lambda根据函数体内代码推断得出；调用与普通函数调用方式相同：

```cpp
auto f = []{ return 42;};
cout << f() << endl;
```

lambda不能有默认参数，lambda调用的形参与实参数量必须始终相等，类型必须匹配。lambda可以直接使用局部static变量和在它所在函数之外声明的名字。

```cpp
void biggies(vector<string> &words,vector<string>::size_type sz)
{
    elimDups(words);//参见上面几段的代码
    //按词长度排序，相同长度则维持字典序
    stable_sort(words.begin(),words.end(),
                [](const string &a,const string &b)
                { return a.size() < b.size();});
    //获取一个迭代器，指向第一个满足size()>=sz的元素
    auto wc = find_if(words.begin(),words.end(),
            [sz](const string &a)
            {return a.size() >= sz;});
    //计算满足size()>=sz的元素的数目
    auto count = words.end() - wc;
    cout<< count;
    //for_each算法：接收一个可调用对象，并对输入序列中每个元素调用此对象
    for_each(wc,words.end(),[](const string &s){cout << s << " ";});
}
```

```cpp
//两种不同的add
void add(const int a, const int b)
{
    auto sum = [a](const int c)->int
    { return a + c; };
    cout << sum(b) << endl;
}

int main()
{
    int a = 6, b = 7;
    auto f = [](const int a, const int b)->int
    { return a + b; };
    cout << f(a, b) << endl;
    add(a, b);
}
```

lambda可以捕获值与引用，当捕获引用时，要保证在lambda执行的时候变量依然存在。应该尽量减少捕获的数据量，尽量避免捕获指针或引用。

lambda隐式捕获：可以通过在捕获列表中写一个&或者=的方式帮助编译器推断，&为捕获引用方式，=为值捕获方式。

```cpp
//可以修改上面的代码如下：
    auto wc = find_if(words.begin(),words.end(),
            [=](const string &a)
            {return a.size() >= sz;});
//for_each部分可以修改如下：
//传入列表中ostream &os = cout,char c = ' '
    for_each(words.begin(), words.end(), [=,&os](const string &s)
    { os << s << c; });
```

可变lambda：希望能改变一个被“值捕获”的变量的值，必须在参数列表首加上关键字`mutable`

```cpp
void fcn1()
{
    size_t v1 = 42;
    //auto f = [v1](){return ++v1;};//ERROR: Cannot assign to a variable captured by copy in a non-mutable lambda
    auto f = [v1]()mutable{return ++v1;};
    v1 = 0;
    auto j = f();//j=43
    
    //或者是
    size_t v2 = 42;
    auto f2 = [&v2]{return ++v2;};
    v2 = 0;
    auto k = f2();//k=1
}
```

参数绑定：标准库函数：`bind`，定义在头文件`functional`中，接收一个可调用的对象，生成一个新的可调用对象“适应”原对象的参数列表。bind的参数：`auto newCallable = bind(callable,arg_list);`如果需要传入参数，就使用置位符`_n`，定义在`std::placeholders::_?`(?可以为1、2···)

```cpp
//统计长度小于等于6的单词数量的程序
using std::placeholders::_1;

bool check_size(const string &s, string::size_type sz)
{
    return s.size() >= sz;
}

void biggies(vector<string> &words, vector<string>::size_type sz)
{
    //bind中的第二个参数_1就是作为第一个参数传入check_size函数，第三个参数sz作为第二个参数传入check_size中
    auto bc = count_if(words.begin(),words.end(),bind(check_size,_1,sz));
    cout << bc;
}

//bind重排参数顺序
bool isShorter(const string &s1, const string &s2)
{
    return s1.size() < s2.size();
}
sort(words.begin(),words.end(),bind(isShorter,_1,_2);//调用isShorter(A,B)，交换_1和_2可以执行isShorter(B,A)

//绑定引用参数
for_each(words.begin(),words.end(),bind(print,os,_1,' ');//ERROR:不能拷贝os
for_each(words.begin(),words.end(),bind(print,ref(os),_1,' ');//标准库ref函数：传递给bind一个对象但不拷贝它
```

迭代器(Addition)：

+ 插入迭代器（只有在容器支持`push_back`或者`push_front`时候才能用各自配套的迭代器）
	+ `back_inserter`：创建一个使用`push_back`的迭代器
	+ `front_inserter`：创建一个使用`push_front`的迭代器
	+ `inserter`：创建一个使用`insert`的迭代器，接收第二个参数，必须指向给定容器的迭代器，插入后it还是指向它原来的元素（也即调用`inserter(c,iter)`时得到一个迭代器，接下来使用时会把元素插入到`iter`原来所指向的元素之前的位置：`it = c.insert(it,val); ++it;//使它指向原来元素`）

	例如，使用`unique_copy`拷贝不重复的元素的目的位置，将vector中不重复元素拷贝到一个初始为空的list中:`unique_copy(vec.begin(),vec.end(),back_inserter(lst));`
	
+ 流迭代器
	+ `istream_iterator`：允许懒惰求值，绑定到一个流之后不一定马上从流中读取，直到使用迭代器才真正读取。

	|istream\_iterator操作|含义|
	|:-:|:-:|
	|istream\_iterator<T> in(is);|in从输入流is读取类型为T的值|
	|istream\_iterator<T> end;|读取类型为T的值的istream\_iterator迭代器，表示尾后位置|
	|in1 == in2 或 !=|in1与in2必须类型相同，如果绑定到相同的输入或都是尾后迭代器，则相等|
	|*in|返回从流中读取的值|
	|in++ 或 ++in|使用元素类型所定义的>>运算符从输入流中读取下一个值，返回的值是否是递增过的参考++前置还是后置|	
	
	```cpp
	//计算从标准输入读取的值的累加值：
	istream_iterator<int> in(cin), eof;
	cout << accumulate(in, eof, 0) << endl;
	
	//从标准输入构造一个容器
	istream_iterator<int> in_iter(cin), eof;
	//方法1：
	vector<int> vec;
	while(in_iter != eof)
	    vec.push_back(*in_iter++);
	//方法2：
	vector<int> vec(in_iter, eof);//通过范围构造
	```
	
	+ `ostream_iterator`：可以对任何具有<<运算符的类型定义ostream\_iterator，可以选择第二个参数，在每个输出元素后都会打印这个字符串（C风格）

	|ostream\_iterator操作|含义|
	|:-:|:-:|
	|ostream\_iterator<T> out(os);|out将类型为T的值写到输出流os中|
	|ostream\_iterator<T> out(os,d);|out将类型为T的值写到输出流os中，每个值后面都跟一个d。d指向一个空字符结尾的字符数组|
	|out = val|用<<运算符把val写入到out绑定的ostream中。类型必须兼容|
	|out++,*out,++out|不对out做任何事情，都返回out|
	
	```cpp
	//输出vec，每个元素后面加一个空格
    ostream_iterator<int> out_iter(cout, " ");
    //方法1：
    for (auto e : vec)
        *out_iter++ = e;//也可以等价写成out_iter = e;但是可读性下降了，不建议
    //方法2：
    copy(vec.begin(),vec.end(),out_iter);
    cout << endl;
	```
		
+ 反向迭代器：带r，跟普通迭代器类似，略。
+ 移动迭代器