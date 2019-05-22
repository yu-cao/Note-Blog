<h2>动态内存与智能指针</h2>

都是C++11的内容。

<h3>shared_ptr</h3>

shared_ptr:绝大部分雷同C的指针，比如p指向一个对象，p在条件判断中为true，*p得到它指向的对象，p->mem等于(\*p).mem

|命令|意义|
|:-:|:-:|
|`shared_ptr<T> sp` <br>`unique_ptr<T> up`|空智能指针，可以指向类型为T的对象|
|p.get()|返回p中保存的指针，但是如果智能指针释放了其对象，返回的指针对象也会消失|
|swap(p,q)<br>p.swap(q)|交换p和q的指针|
|**以下为shared_ptr独有**|
|`make_shared<T> (args)`|返回一个shared_ptr，指向一个动态分配的类型为T的对象，用args初始化该对象，args必须匹配T中合适的构造函数进行构造|
|`shared_ptr<T> p(q)`|p是shared_ptr q的拷贝，这个操作会递增q中的计数器，q中的指针必须能被转换成T*|
|p = q|p和q都是shared_ptr，保存的指针能相互转化，会递减p的引用计数，递增q的引用计数，若p引用计数=0，则将其管理的原内存释放|
|p.unique()|p.use_count = 1，返回ture；否则返回false|
|p.use_count()|返回与p共享对象的智能指针数量，很慢，主要用于调试|

```cpp
auto p1 = make_shared<vector<string>>();//p1类型为shared_ptr<vector<string>>
```

每当shared\_ptr进行拷贝或者赋值时，都会记录有多少个其他的shared\_ptr指向相同对象，当shared\_ptr初始化另一个shared\_ptr，或作为参数传递进函数，或作为函数返回值，计数器会递增；当shared\_ptr失效的时候（被赋值或离开作用域等），计数器会递减；一旦shared\_ptr计数器为0，就自动释放内存（使用T对象的析构函数将T析构）

```cpp
class StrBlob
{
public:
    using size_type = std::vector<std::string>::size_type;

    StrBlob() : data(make_shared<vector<string>>())
    {};

    //实参数量未知，可变参数模板
    StrBlob(std::initializer_list<std::string> il) : data(make_shared<vector<string>>())
    {};

    size_type size() const
    { return data->size(); }

    bool empty() const
    { return data->empty(); }

    //增删元素
    void push_back(const std::string &t)
    { data->push_back(t); }

    void pop_back()
    {
        check(0, "pop_back empty StrBlob");
        return data->pop_back();
    }

    //元素访问
    std::string &front()
    {
        check(0, "front on empty StrBlob");
        return data->front();
    }

    std::string &back()
    {
        check(0, "back on empty StrBlob");
        return data->back();
    }

private:
    std::shared_ptr<std::vector<std::string>> data;

    void check(size_type i, const std::string &msg) const//data[i]不合法就抛出异常
    {
        if (i >= data->size())
            throw out_of_range(msg);
    }
};
```

初始化：

提供了一个括号包围的初始化器，就可以使用auto，从这个初始化器来推断我们想要分配的类型（当且仅当只有一个初始化器）

```cpp
auto p1 = new auto(obj);//p指向一个与obj类型相同的对象，该对象用obj进行初始化
auto p2 = new auto{a,b,c};//ERROR，括号内只允许单个初始化器
```

内存耗尽：new表达式失败，抛出bad_alloc的异常；及时使用delete表达式归还动态申请的内存，销毁对象，但是**delete一块非new分配的内存或相同指针值重复释放的结果是未定义的**

```cpp
int *p1 = new int(30);//p1指向的对象的值为30，内存耗尽就抛出异常
int *p2 = new (nothrow) int;//内存耗尽就返回给p2空指针，为定位new
delete p1;//p1必须指向一个动态分配的对象或者空指针
```

**程序员对new出来的对象负责，必须显式释放**：由内置指针（而非智能指针）管理的动态内存在被显示释放前一直存在。所以，坚持只使用智能指针可以避免这些问题

以下两个是典型错误：

```cpp
StrBlob* factory(string &s)
{
    return new StrBlob({s});
}

void use_factory(string &s)
{
    auto p = factory(s);
    //忘记了delete...
}//离开了作用域，p失效，但是指向的内存却还在，发生泄漏
```

```cpp
int *p(new int(42));
auto q = p;
delete p;//p与q同时无效！对q操作很可能出现高危操作
p = nullptr;
```

智能指针替代内置指针提高动态内存安全性：

```cpp
vector<int> *new_vector(void)
{
    return new(nothrow) vector<int>;
}

//使用智能指针方式
shared_ptr<vector<int>> new_share_vector(void)
{
    return make_shared<vector<int>>();
}

void read_ints(vector<int> *pv)//使用智能指针方式(shared_ptr<vector<int>> pv)
{
    int v;

    while (cin >> v)
        pv->push_back(v);
}

void print_int(vector<int> *pv)//使用智能指针方式(shared_ptr<vector<int>> pv)
{
    for (const auto &item : *pv)
        cout << v << " ";
    cout << endl;
}

int main(int argc, char* argv[])
{
    vector<int> *pv = new_vector();//省略后面内存不足的检测
    auto spv = new_share_vector();//使用智能指针方式

    read_ints(pv);
    print_int(pv);

    //使用智能指针则不再需要以下两行：
    delete pv;
    pv = nullptr;
}
```

不能将一个内置的指针形式隐式转换成一个智能指针，因为智能指针构造是explicit的，必须使用直接初始化，初始化智能指针的普通指针一般要指向动态内存，否则就需要提供自己的操作去替代delete

```cpp
shared_ptr<int> p1(new int(1024));//正确
shared_ptr<int> p2 = new int(1024);//ERROR，必须使用直接初始化
//同理：
shared_ptr<int> clone(int p){return new int(p);}//ERROR，使用了隐式初始化
shared_ptr<int> clone(int p){return shared_ptr<int>(new int(p));}//正确

//凡是有第二个参数的都是去替代delete操作的
shared_ptr<T> p(u);//p从unique_ptr u那里接管了对象所有权，u=nullptr
shared_ptr<T> p(q,d);//p接管了q所指向对象的所有权。q要能转换成T*类型，p可以使用对象d来代替delete
p.reset(q,d);//令p指向q，会调用d而不是delete释放q
```

**不要混合使用普通指针与智能指针**：推荐使用make\_shared，因为shared\_ptr只能对于自身的拷贝之间可以协调对象的析构，否则有多个独立创建的shared\_ptr绑定到同一块内存的风险。**建议：一旦使用了智能指针，就不应该使用普通指针访问其所指向的内存**

观察以下代码：

```cpp
void process(shared_ptr<int> ptr)
{
    //使用ptr
}//离开作用域，被销毁

int main(int argc, char* argv[])
{
//以下代码安全：
    shared_ptr<int> p(new int(1024));//引用计数为1
    process(p);//传入时引用计数+1，跳出函数时引用计数-1，所以还是1
    int i = *p;//安全
//类似的，但以下代码不安全：
    int *x(new int(1024));
    //合法，但内存会被释放：
    //因为构建临时变量传入，原来是0，引用计数+1，跳出时引用计数-1，变成0，智能指针自动释放了内存
    process(shared_ptr<int>(x));
    int j = *x;//Ub,x是悬垂指针
}
```

**永远不要使用get初始化另一个智能指针或为智能指针赋值**：智能指针有get函数，它会返回一个普通指针指向其管理的内存，为的是向不能使用智能指针的代码传递一个普通指针。**使用get返回的指针代码永远不要delete这个指针**，否则会有如下效果：

```cpp
shared_ptr<int> p(new int(42));
int *q = p.get();
{
    shared_ptr<int>(q);
}//程序块结束，q被销毁，它指向的内存被释放
int i = *p;//Ub，p指向的内存已经释放，而且当p销毁时会出现二次delete
```

将一个新的指针赋给shared\_ptr，不能直接`p = new int(1024);`这种，因为int*无法隐式转换成shared\_ptr，可以使用`p.reset(new int(1024));`将p指向一个新对象，常与unique联用

```cpp
if(!p.unique())//检查是否是唯一用户
    p.reset(new string(*p));//不是唯一用户，分配新的拷贝
*p += newVal;//是唯一用户，可以改变对象的值
```

智能指针与异常：智能指针很好地控制了异常发生后资源的释放

```cpp
void f()
{
    shared_ptr<int> sp(new int(42));
    //抛出异常，在f中未捕获
}//结束时也会自动释放内存

void f()
{
    int *ip = new int(42);
    //中间抛出异常，在f中未捕获
    delete ip;
}//内存永远不会释放
```

智能指针与哑类：同时为C与C++设计的类很多时候没有定义析构函数，需要用户显式释放所使用的资源。定义自己的删除器来代替默认的delete：（也就是通过谓词的方式）

```cpp
struct destination{};
struct connection{};

connection connect(destination *)
{
	cout << "建立连接" << endl;
	return connection();
}

void disconnect(connection)
{
	cout << "关闭连接" << endl;
}

void end_connection(connection *p)
{
	disconnect(*p);
}

//未使用shared_ptr
void f(destination &d)
{
	connection c = connect(&d);
}

//使用shared_ptr
void f1(destination &d)
{
	connection c = connect(&d);
	
	shared_ptr<connection> p(&c,end_connection);
}

//使用shared_ptr和lambda表达式
void f2(destination &d)
{
	connection c = connect(&d);

	shared_ptr<connection> p(&c,[](connection *p){disconnect(*p);});
}

int main(int argc, char*argv[])
{
	destination d;
	f(d);
	f1(d);
	f2(d);
}
```

**总而言之，使用智能指针时要注意以下问题：**

  + 不使用相同的内置指针值初始化
  + 不delete get()返回的指针值
  + 不使用get()初始化或reset另一个智能指针
  + 使用get()返回的指针，最后一个对应的智能指针销毁后，指针就无效了
  + 当智能指针管理的资源不是new出来的东西时，需要传递给它一个删除器替代delete

<h3>unique_ptr</h3>

某个时刻只能有一个unique\_ptr指向一个给定对象；当其被销毁时，所指向的对象也同时被销毁；初始化unique\_ptr时必须采用直接初始化，不支持普通的拷贝或赋值操作（可以拷贝与赋值一个即将被销毁的unique\_ptr，比如从函数中返回的unique\_ptr），在STL中，将其拷贝构造函数和赋值函数声明为delete以阻止拷贝与赋值操作。`unique_ptr<T> u(p);//p是能转换成T*的普通指针是可以的，但是当u离开作用域后，内存就会自动回收，普通指针p就会变成悬垂指针，有风险`

|操作|含义|
|:-:|:-:|
|`unique_ptr<T> u1`|一个空的unique\_ptr，指向类型为T的对象|
|`unique_ptr<T, D> u2`|u2会使用类型为D的可调用对象来释放它的指针|
|`unique_ptr<T, D> u3(d)`|一个空的unique\_ptr，指向类型为T的对象，用类型为D的对象d来代替delete|
|u.release()|放弃对指针的控制权，返回指针（必须使用！通常用来初始化另一个智能指针或给另一个智能指针赋值），u=nullptr|
|u.reset()|释放u指向的对象|
|u.reset(q)|释放u指向的对象，如果提供了普通指针q，令u指向这个对象，q也可以是nullptr|

```cpp
auto p2 = p1.release();//正确
p2.reset(p3.release());//释放了p2原来指向的内存，所有权p3转移给p2
p2.release();//错误：p2不会释放内存而且我们丢失了指针（内存泄漏）
```

传递删除器：

```cpp
// unique_ptr<objT,delT> p(new objT,fcn);
void f(destination &d)
{
    connection c = connect(&d);
    unique_ptr<connection,decltype(end_connection)*> p(&c,end_connection);
    //...f退出时会正确关闭（即使是异常而退出）
}
```

<h3>weak_ptr</h3>

不控制所指向对象生存期的智能指针，指向一个由`shared_ptr`管理的对象，指向后引用计数不会更改

|操作|含义|
|:-:|:-:|
|`weak_ptr<T> w(sp)`|与`shared_ptr sp`指向相同对象的`weak_ptr`，T能够转成sp的指向的类型|
|w = p|p可以是`shared_ptr sp`或`weak_ptr`，赋值后共享对象|
|w.reset()|将w置为空|
|w.use_count()|与w共享对象的`shared_ptr`数量|
|w.expired()|若w.use_count==0，返回true，否则为false|
|w.lock()|如果expired为true，返回一个空的`shared_ptr`；否则返回一个指向w的对象的`shared_ptr`|

由于对象可能不存在，所以不能直接使用`weak_ptr`访问对象，而必须使用lock

```cpp
auto p = make_shared<int>(42);
weak_ptr<int> wp(p);//不改变p的引用计数
if (shared_ptr<int> np = wp.lock()//np不空，则条件成立
{//...在if中，np与p共享对象
}
```

`weak_ptr`作用：可以构建核查指针类，用w.lock()的方式防止出现越权访问。

<h3>动态数组</h3>

初始化动态分配对象的数组：`int *pia1 = new int[10]();`10个值初始化为0的int；也可以列表初始化`int *pia1 = new int[10]{1,2,3,4,5,6,7,8};`前8个用列表初始化，剩余的进行值初始化；当列表元素大于元素数目，则new失败，抛出异常；括号中不能给出初始化器，也就不能使用auto分配数组

```cpp
char arr[0];//错误，不能定义长度为0的数组
char *cp = new char[0];//正确，但cp不可解引用，可以视为尾后指针，其他操作都是合法的
```

释放动态数组：元素按逆序销毁，**释放时忽略了方括号，行为是未定义的；同理，对单一对象释放时使用了方括号，行为也是未定义的！**

```cpp
delete p;//p必须指向动态分配的“对象”或为空
delete[] pa;//pa必须指向动态分配的“数组”或为空
```

用`unique_ptr<T[]> up(new T[n]);`这种方式进行管理:

```cpp
unique_ptr<int[]> up(new int[10]);
up.release();//自动用delete[]销毁指针
```

指向数组的`unique_ptr`不支持成员运算符（`.`和`->`），`u[i]`返回u拥有的数组中位置i处的对象

而`shared_ptr`管理动态数组必须自己提供删除器，不提供将会导致Ub；也未定义下标运算符，也不支持指针算术运算（智能指针类型）

```cpp
shared_ptr<int> sp(new int[10],[](int *p){delete[] p;});
sp.reset();//lambda方法释放，使用delete[]

for(size_t i = 0; i!=10;i++)
    *(sp.get()+1) = i;//只能使用get获取一个内置指针来访问数组元素
```

<h3>allocator类</h3>

目的：把内存分配和对象构造分离开来（`new`是把两个整合在一起，对于构造单个对象有利，对于动态内存，按需构造来说浪费性能），分配得到的内存是原始的，未经过构造的（所以**直接使用未构造的内存是严重错误！Ub**）；该类定义在`<memory>`头文件中。

|操作|含义|
|:-:|:-:|
|`allocator<T> a`<br>`a.allocate(n)`|参考下面的代码即可|
|`a.deallocate(p,n)`|释放从`T*`指针p中地址开始的内存，这块内存保存了n个类型为T的对象；p必须是一个之前由`allocate`返回的指针，n必须是p创建时所要求的大小。调用前，用户必须对每个这块内存中创建的对象调用destroy|
|`a.construct(p,args)`|p必须是一个类型是`T*`的指针，指向一块原始内存；args被传递给类型为T的构造函数，用来在p指向的内存中构造一个对象|
|`a.destroy(p)`|p为T*类型的指针，对p指向的对象执行析构函数|

```cpp
allocator<string> alloc;
auto const p = alloc.allocate(n);//分配n个未初始化的string
auto q = p;
alloc.construct(q++,10,'c');//q指向最后构造元素之后的位置
cout << *p << endl;//*p为cccccccccc，输出*q是未定义的
while(q != p)
    alloc.destroy(--q);//释放我们实际构造出来的string（也只能对实际构造出来的对象进行destroy）
alloc.deallocate(p,n);//p不可为空，n必须相同
```

必须用construct构造对象，用完后必须对每个构造元素调用destroy进行销毁，被销毁后可以重新使用这部分内存来保存其他T类型对象（代码示例为string），也可将其归还系统

拷贝与填充未初始化内存的算法

|操作|含义|
|:-:|:-:|
|`uninitialized_copy(b,e,b2)`|从迭代器b到e的输入范围中拷贝元素到迭代器b2指定的未构造的原始内存中，b2指向的内存必须足够大以容纳|
|`uninitialized_copy_n(b,n,b2)`|从迭代器b指向的元素开始，拷贝n个元素到b2开始的内存中|
|`uninitialized_fill(b,e,t)`|从迭代器b到e的原始范围中创建对象，对象的值均为t的拷贝|
|`uninitialized_fill_n(b,n,t)`|从迭代器b指向的元素开始创建n个对象，b必须保证足够大的未构造原始内存以容纳|

```cpp
vector<int> vi(10,2);
allocator<int> alloc;
auto p = alloc.allocate(vi.size()*2);//分配比vi元素数量大一倍的动态内存
auto q = uninitialized_copy(vi.begin(),vi.end(),p);//拷贝vi中的元素来构造从p开始的元素，返回的q是最后一个构造元素之后的位置
uninitialized_fill_n(q,vi.size(),43);//剩余部分初始化为43
```