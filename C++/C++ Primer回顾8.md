<h2>关联容器</h2>

|按关键字有序保存元素|类型|
|:-:|:-:|
|map|关联数组，保存key-value对|
|set|key即为value，即只保存关键字的容器|
|multimap|关键字可以反复出现的map|
|multiset|关键字可以反复出现的set|
|**无序集合**||
|unordered_map|用hash函数组织的map|
|unordered_set|用hash函数组织的set|
|unordered_multimap|用hash函数组织的map，关键字可以反复出现|
|unordered_multiset|用hash函数组织的set，关键字可以反复出现|

```cpp
    map<string, size_t> word_count;
    set<string> exclude = {"The","But","And","Or"};
    string word;
    while (cin >> word)
        if(exclude.find(word) == exclude.end())//只统计不在exclude中的单词
            ++word_count[word];//未在map内就会额外创建一个新元素，key=word，val=0
    for (const auto &w : word_count)
    {
        cout << w.first << " occurs " << w.second
             << ((w.second > 1) ? " times" : "time") << endl;
    }
```

关联容器不支持顺序容器的位置相关操作，如push\_back或push\_front，也不支持构造函数或插入操作这些接收一个元素值和一个数量值的操作。

构造相关容器：

初始化时，map必须同时提供key和value，并且用`{key, value}`表示，key是不可以被修改的，默认为const，value是可以被修改的

```cpp
map<string, string> authors = {{"Joyce",   "James"},
                               {"Austen",  "Jane"},
                               {"Dickens", "Charles"}};
```

标准库用关键词类型的"<"运算符来比较两个关键词，我们可以重载元素的"<"，提供的操作必须要严格弱序("≤")

自定义的操作也是容器类型的一部分，指定使用自定义的操作，必须在定义关联容器类型时提供此操作的类型，而且必须在尖括号中紧接着元素类型给出。

```cpp
bool compareISBN(const Sale_data &lhs, const Sale_data &rhs)
{
    return lhs.isbn() < rhs.isbn();
}

int main()
{
    //compareISBN这个函数指针作为自定义操作传入到容器中，在向bookstore添加元素时，调用函数指针为这些元素排序
    //decltype指定出自定义操作的类型
    multiset<Sale_data, decltype(compareISBN) *> bookstore(compareISBN);
}
```

例题：定义一个map，将单词与行号关联，list保存单词出现的行号：

```cpp
    ifstream in(argv[1]);
    map<string, list<int>> word_lineno;
    string line, word;
    int lineno = 0;
    while (getline(in, line))
    {
        lineno++;
        istringstream l_in(line);//构造字符串流读取单词
        while (l_in >> word)
            word_lineno[word].push_back(lineno);//添加行号
    }
```

pair类型：定义于\<utility>中，保存两个public的数据成员，命名为first和second；运算符比较：只有当pair1.first < pair2.first && pair1.second < pair2.second时候才有pair1 < pair2为true

```cpp
pair<string, int> process(vector<string> &v)
{
    if(!v.empty())
        return {v.back(),v.back().size()};//C++11列表初始化方式
        //return pair<string,int>(v.back(),v.back().size());//早先的版本方式，显式构造返回值
    else
        return pair<string,int>();//隐式构造返回值
}
```

set和map的类型：

```cpp
set<string>::value_type v1;//v1是string，set中value_type与key_type一样
map<string,int>::value_type v2;//v2是pair<const string, int>，因为一个map中的元素关键字不可修改
map<string,int>::key_type v3;//v3是string
map<string,int>::mapped_type v4;//v4是int，mapped意义为“被关联的类型”
```

迭代器：对一个关联容器迭代器解引用得到其容器的value_type的值的引用，另外set的迭代器是const；遍历等操作类似之前的顺序容器，不要对关联容器使用泛型算法，因为它们的key是const会导致问题

插入：对于set和map只有在关键字不在c中插入才有效，否则什么都不做，返回pair中迭代器指向在map或set中的重复元素，bool为false；对于multi-类型则会插入每个元素

|命令|含义|
|:-:|:-:|
|c.insert(v)|v是value_type类型对象，函数返回一个pair，包含指向具有指定key的元素的迭代器和是否成功插入的bool，emplace返回值同|
|c.emplace(args)|args作为一个参数进行完美转发调用构造函数进行插入|
|c.insert(b,e)|b和c是两个迭代器，表示一个c::value_type类型的范围，返回void|
|c.insert(li)|li是值的花括号类型（列表初始化），返回void|
|c.insert(p,v)/emplace(p,args)|类似c.insert(v)（或emplace(args)），迭代器p作为提示指出从哪里开始搜索新元素应该存储的位置，返回一个迭代器指向具有给定key值的元素|

```cpp
//multimap使用
void add_child(multimap<string, string> &families, const string &family, const string &child)
{
    families.insert({family, child});
}

int main()
{
    multimap<string, string> families;

    add_child(families,"张","强");
    add_child(families,"张","刚");
    add_child(families,"王","五");

    for (const auto &item : families)
        cout << item.first << "家的孩子：" << item.second << endl;
}
//输出：
张家的孩子：强
张家的孩子：刚
王家的孩子：五
```

删除：除了顺序容器外还提供一个额外的erase操作：接收一个key_type参数，删除所有匹配该key的元素，返回实际删除的元素的数量，如果返回值为0，代表想要删除的元素不再容器中

下标操作：map和unordered_map提供下标运算符与对应的at函数，但是与其他的下标运算符不同的是**如果key不在map中，就会创建一个元素把它插入到map中**：

```cpp
    map<string, size_t> word_count;
    word_count["Anna"] = 1;//插入一个kay=Anna的元素，进行值初始化为0，然后把val=1赋给它
    //c.at(k)如果k不在c中会抛出Out_of_range异常，而不是插入
```

下标运算符返回的是左值，得到mapped\_type类型，与迭代器的类型value\_type是不同的

访问：

|命令|意义|
|:-:|:-:|
|c.find(k)|返回一个迭代器，指向第一个关键词为k的元素，如果不存在，返回尾后迭代器|
|c.count(k)|返回关键字等于k的元素的数量，对于不允许重复key的容器，返回值始终是0/1|
|c.lower_bound(k)|返回一个迭代器，指向第一个key不小于k的元素|
|c.upper_bound(k)|返回一个迭代器，指向第一个key大于k的元素|
|c.equal_range(k)|返回一个迭代器pair，表示key=k的元素的范围（第一个迭代器指向第一个元素，第二个迭代器指向最后一个元素）；若k不存在，则pair两个成员都为c.end()|

如果一个multimap或multiset中有多个元素具有相同的key，则这些元素会被相邻存储。因此，读取方法是用find找到指向等于key的第一个迭代器，然后依次向后：

```cpp
multimap<string,string> authors;
string search_item("Alain de Botton");
auto entries = authors.count(search_item);//元素的数量
auto iter = authors.find(search_item);//作者的第一本书
while(entries)
{
    cout << iter->second << endl;//打印书名
    ++iter;//前进到下一本书
    --entries;//记录已经打印了多少本书
}

//或者使用面向迭代器的方法：如果返回相等迭代器，代表key不在容器中
for(auto beg = authors.lower_bound(search_item), end = authors.upper_bound(search_item); beg != end; ++beg)
    cout << beg->second << endl;
    
//使用equal_range函数的方法：
for(auto pos = authors.equal_range(search_item); pos.first != pos.second; ++pos.first)
    cout << pos.first->second << endl;
```

**C++11**：无序关联容器

这些容器不是先上面一样使用<运算符组织元素，而是使用hash函数和==运算符完成比较。存储上组织为一个桶，每个桶保存0或多个元素，用hash函数把元素映射到桶中。不可以直接定义key类型是自定义类型的无序容器，必须提供我们自己的hash模板