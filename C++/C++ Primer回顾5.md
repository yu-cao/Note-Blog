#IO类

每个IO对象都在维护一组条件对象，以指出该对象上是否可以进行IO操作。如果遇到了错误，对象的状态变成失效，后续操作都不能执行直到错误被纠正。

加w代表读取宽字符，如wcin, wcout, wcerr等。

|头文件|类型|
|-----|----|
|iostream|istream, wistream 从流中读取数据|
||ostream, wostream 向流中写数据|
||iostream, wiostream 读写流|
|fstream|ifstream, wifstream 从文件中读取数据|
||ofstream, wofstream 向文件写入数据|
||fstream, wfstream 读写文件|
|sstream|istringstream, wistringstream 从string读取数据|
||ostringstream, wostringstream 向string写入数据|
||stringstream, wstringstream 读写string|

ifstream和istringstream都继承自istream，所以这三者操作都是类似的，不能对其拷贝或者赋值，所以通常用引用的方式传递和返回流，读写一个IO对象会改变其状态，所以传递与返回不能是const

查询流的状态`iostate`类型，包括4个iostate类型的constexpr值：

|类型|位模式|
|---|-----|
|badbit|系统级错误，流无法再使用|
|failbit|可恢复错误，如期望读数值却读到一个字符|
|eofbit|文件结束标志|
|goodbit|流未发生错误|

`endl`输出一个换行，然后刷新缓冲区；`flush`直接刷新缓冲区；`ends`输出一个空字符，然后刷新缓冲区<br>
程序异常崩溃，输出缓冲区不会被刷新，输出代码很可能已经执行但是没有被打印而已（debug的要点）

```cpp
int main()
{
    cout << "hi!" << flush;// flush刷新缓冲区
    cout << "hello";
    
    // 故意崩溃
    char *p = "j";
    *p = 'o';
}
/*不会输出下面的hello，Output如下：
hi!
Process finished with exit code 10*/
```

unitbuf操作符：

```cpp
cout << unitbuf;// 后续的所有输出操作都会立即刷新缓冲区
cout << nounitbuf;// 回到正常的缓冲方式
```

文件模式：只可以对ofstream和fstream对象设定out模式（以写的方式打开）；只可以对ifstream或fstream对象设定in模式（以读的方式打开）。ate模式（打开文件后立即定位到文件末尾）和binary模式（以二进制方式进行IO）可以用于任何文件流对象。

```cpp
ofstream out("file1"); // 隐含了out+truc模式
// 等同于
ofstream out2("file1", ofstream::out);// 隐含了trunc模式（截断文件or直接覆盖）
ofstream out3("file1", ofstream::out | ofstream::trunc);

// 保留文件内容的写入必须指定为app模式（append，写操作前定位到文件末尾）
ofstream app("file2", ofstream::app);// 隐含了out模式
ofstream app2("file2", ofstream::out | ofstream::app);
```

string流

```cpp
#include <iostream>
#include <sstream>

using namespace std;

istream &f(istream &in)
{
    int v;
    while(in >> v, !in.eof())
    {
        if(in.bad())
            throw runtime_error("IO流错误");
        if(in.fail())
        {
            cerr << "数据流错误，请重试： " << endl;
            in.clear();
            in.ignore(100,'\n');
            continue;
        }
        cout << v << endl;
    }
    in.clear();
    return in;
}

int main()
{
    ostringstream msg;
    msg << "C++ Primer 5th Edition" << endl;

    // sstream strm(s) strm是一个sstream对象，保存string s的一个拷贝，构造函数式explicit
    // strm.str() 返回strm所保存的string的拷贝
    // strm.str(s) 将string s拷贝入strm中，返回void
    // 如果原先in里面是之前读到eof等的failbit置位等情况，需要先进行in.clear()进行重新置位
    istringstream in(msg.str());
    f(in);
    return 0;
}
```

使用ostringstream：等验证完所有的电话号码后才进行输出（方法：先将输出内容写入到一个内存ostringstream中）：

```cpp
struct PersionInfo{
	string name;
	vector<string> phones;
};
vector<PersonInfo> people;
for (const auto &entry : people){
	ostringstream formatted, badNums;
	for(const auto &nums : entry.phone){
		if(!vaild(nums))
			badNums << " " << nums;// 将数的字符串形式写入badNums
		else
			formatted << " " << format(nums);// 将格式化的字符串写入formatted
	}
	if(badNums.str().empty())
		os << entry.name << " " << formatted.str() << endl;
	else
		cerr << "input error: " << entry.name << " invalid number(s) " << badNums.str() << endl;
}
```