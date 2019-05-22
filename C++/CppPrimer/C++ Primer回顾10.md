<h2>拷贝控制</h2>

编译器会为我们合成一个拷贝构造函数，无论我们是否定义了其他构造函数

拷贝构造函数必须参数是引用：因为自身的参数是非引用类型，为了调用它，必须拷贝其实参，而为了拷贝实参，又要调用拷贝构造函数···出现了无限递归导致错误

赋值运算符通常应该返回一个指向其左侧运算对象的引用（否则无法进行从右向左连续赋值）

```cpp
class Foo{
public:
    Foo &operator=(const Foo &);
    //...
};
```

由编译器生成的“合成拷贝构造函数”与“合成拷贝赋值运算符”都是用来完成拷贝，**部分情况（类中有数据成员不能默认构造、拷贝、赋值(比如有const)或销毁）下是阻止拷贝或赋值该类型对象**；**C++11**：我们也可以自己写这样的函数来进行阻止：`=delete`通知编译器，我们不希望定义与使用这些成员，且必须出现在函数第一次声明的时候（我们不能删除析构函数，如果类中定义了这样的析构函数，编译器将不允许我们将该类实例化）：

```cpp
struct NoCopy{
    NoCopy() = default;
    //新标准之前是把这两个函数放到private中进行阻止，并且只声明，不定义：
    NoCopy(const NoCopy&) = delete;//阻止拷贝
    NoCopy &operator=(const NoCopy&) = delete;//阻止赋值
    
    ~NoCopy() = default;
    //...
};
```

构造函数：构造函数初始化对象的非static成员<br>
析构函数：析构函数释放对象使用的资源，销毁对象的非static数据成员，不接受参数；析构函数自身并不直接销毁成员，成员是析构函数体之后的隐含的析构阶段被销毁；当没有定义析构函数时，编译器会为它生成“合成析构函数”，函数体为空，成员在这个空函数体执行完后逐个销毁；当类中分配动态内存时，需要自己定义一个析构函数，在函数体中对构造函数分配的内存进行delete处理

三/五法则：当类需要一个析构函数时，那么它同时也需要一个拷贝构造函数与拷贝赋值运算符，因为编译器提供的往往是基于bit的全拷贝；当需要拷贝操作的类也需要赋值操作，反之亦然，但不一定需要析构函数。<br>一个类如果定义了任何一个拷贝操作，他就应该定义所有5个操作，当拷贝资源导致性能开销，当拷贝没有必要时，定义了移动构造函数与移动赋值运算符的类就可以避免这种开销（要么3，要么5）

```cpp
class HasPtr{
public:
    HasPtr(const string &s = string()):ps(new string(s)),i(0){}
    ~HasPtr(){delete ps;}
    //没有定义拷贝构造函数与拷贝赋值运算符，将使用编译器默认提供的...
private:
    string *ps;
    int i;
};

//上面的类定义因为没有遵循三/五法则导致错误
HasPtr f(HasPtr hp)//值传递，拷贝
{
    HasPtr ret = hp;//拷贝给定的HasPtr（基于bit的拷贝，指针指向同一块内存）
    return ret;//ret和hp都被销毁
}

HasPtr p("test");
f(p);//f结束时，p.ps指向的内存已经被释放
HasPtr q(p);//现在p与q都指向无效内存！
```

**C++11**：移动构造函数：避免原来的在分配新的内存后需要拷贝原来的数据然后销毁原来的数据（值拷贝）的庞大的性能开销，而是只需要将资源从给定对象“移动”到正在创建的对象。

move标准库函数：定义在`<utility>`头文件中，需要注意的是，通常不为move提供一个using声明，而是直接调用`std::move`

右值引用：必须绑定到右值的引用（通过`&&`而不是`&`来获得右值引用），而且只能绑定到一个将要销毁的对象

比较：变量是左值，但是经过运算的后的返回是右值；字面常量都是右值；我们不能把一个右值引用绑定到一个变量上，即使这个变量是右值引用类型也不行

```cpp
int &&rr1 = 42;//正确，字面常量是右值
int &&rr2 = rr1;//错误，表达式rr1是左值
```

||左值|右值|
|:-:|:-:|:-:|
|属性|对象的身份|对象的值|
|时间|持久|短暂|

```cpp
    int i = 42;
    int &r = i;//正确
    int &&rr = i;//错误：不能把一个右值引用绑定到左值上
    int &r2 = i * 42;//错误：i*42得到的是一个右值
    const int &r3 = i * 42;//正确，可以把一个const的引用绑定到一个右值上
    int &&rr2 = i * 42;//正确
```

右值引用所被引用的对象：1. 即将被销毁 2. 该对象没有其他用户；所以右值引用的代码可以自由接管所引用对象的资源

不能把一个右值引用直接绑定到一个左值上，但是我们可以显式地将一个左值转换为对应的右值引用类型进行绑定：

```cpp
int &&rr3 = std::move(rr1);//必须直接调用std::move而不是using声明
```

**一旦调用了move，我们就应该保证：除了对rr1赋值或销毁外，我们不再使用它**（不能使用移动后原对象的值）移动操作”窃取“了资源，通常不分配资源，不抛出异常（不抛出异常的移动构造函数和移动赋值运算符都必须标记为`noexcept`，无论在定义还是声明中），不抛出异常必须显式定义，否则有可能使用拷贝构造函数而不是移动构造函数。

移动赋值运算符：必须确保移后源对象处于可析构状态，移动后对源对象要求保证其依然有效，但是对其中留下的值没有要求，我们的程序不能依赖移动后源对象中的数据，可以给移动后源对象赋予新值。

```cpp
StrVec &StrVec::operator=(StrVec &&rhs) noexcept
{
    //直接检测自赋值
    if (this != &rhs){
        free();//释放已有元素
        elements = rhs.elements;//从rhs接管资源
        first_free = rhs.first_free;
        cap = rhs.cap;
        rhs.elements = rhs.first_free = rhs.cap = nullptr;//将rhs置于可析构状态
    }
    return *this;
}
```

只有当一个类没有定义任何自己版本的拷贝控制成员且类中每个非static数据都可以移动时，编译器才会为它合成移动构造函数或移动赋值运算符；移动操作永远不会隐式定义成删除的函数，除非我们显式要求编译器生成(`=default`)但编译器无法移动所有成员时才会定义为删除的函数

```cpp
struct X{
    int i;//内置类型可以移动
    std::string s;//string有自己定义的移动操作
};

struct hasX{
    X mem;//X有合成的移动操作
};

class Y{
    int j;
    //定义了自己的拷贝构造函数，但未定义自己的移动构造函数
    Y(const Y &y):j(y.j){}
};

struct hasY{
    hasY() = default;
    hasY(hasY &&) = default;//无法默认合成
    Y mem;
};

int main(int argc, char*argv[])
{
    X x,x2 = std::move(x);//使用合成的移动构造函数
    hasX hx,hx2 = std::move(hx);//使用合成的移动构造函数
    
    hasY hy, hy2 = std::move(hy);//错误，移动构造函数是删除的
}
```

一个定义了移动构造函数或移动赋值运算符的类也定义自己的拷贝操作，否则，这些成员默认定义为删除的。

如果既有拷贝，又有移动的情况下，移动右值，拷贝左值（类似函数调用匹配），如果没有移动构造函数，右值也会被拷贝（即使调用move，也是跟之前一样进行值拷贝）

移动迭代器：通过改变给定迭代器的解引用运算符的行为来适配该迭代器。该迭代器的解引用返回生成了一个右值引用；标准库中`make_move_iterator`函数将一个普通迭代器转换为一个移动迭代器，接收一个迭代器，返回一个移动迭代器

```cpp
void StrVec::reallocate()
{
    auto newcapacity  = size() ? 2 * size() : 1;//分配2倍大小的内存
    auto first = alloc.allocate(newcapacity);
    
    //移动元素
    //对输入序列的每个元素调用construct“拷贝”到目标位置，因为使用的是右值引用，所以construct使用移动构造函数来进行构造
    auto last = uninitiallized_copy(make_move_iterator(begin()),make_move_iterator(end()),first);
    
    free();//释放旧空间
    elements = first;//更新指针
    first_free = last;
    cap = elements + newcapacity;
}
```

在类的代码中小心使用move可以大幅提升性能，只有当你确信需要进行移动而且移动安全时，才可以使用`std::move`，否则很可能导致一些莫名其妙而且难以定位的错误

```cpp
void StrVec::push_back(const string& s)
{
    chk_n_alloc();//确保有空间容纳新元素
    alloc.construct(first_free++,s);//在first_free指向的元素中构造s的一个副本
}

void StrVec::push_back(string&& s)
{
    chk_n_alloc();//如果需要的话为StrVec重新分配内存
    alloc.construct(first_free++,std::move(s));
}

StrVec vec;//空StrVec
string s = "some string";
vec.push_back(s);//调用push_back(const string&)
vec.push_back("done");//调用push_back(string&&)
```

由于历史原因，右值是可以进行赋值的，但是我们可以在自己写的类中进行阻止：强制左侧运算对象（即this所指的对象）是一个左值。方法：在参数列表后面放置一个引用限定符，可以是`&`或`&&`，可以限定this指向的是左值还是右值（定义与声明必须都要出现），而且const限定必须在前

```cpp
class Foo{
public:
    Foo &operator=(const Foo&) &;//只能向可修改的左值赋值
    
    Foo someMem() & const;//错误：const限定符必须在前
    Foo anotherMem() const &;//正确：const在前
};

Foo &Foo::operator=(const Foo &rhs) &
{
    return *this;
}
```

如果一个成员函数有引用限定符，则具有相同参数列表的所有版本都要有引用限定符：

```cpp
class Foo{
public:
    Foo sorted() &&;
    Foo sorted() const;//错误：必须加上引用限定符
    
    using Comp = bool(const int&, const int&);
    Foo sorted(Comp*);//正确，不同的参数列表
    Foo sorted(Comp*) const;//正确，两个版本都没有引用限定符
```

是否匹配右值：（下面两者的核心：ret是个左值，Foo(*this)是个右值，所以调用不同）

```cpp
Foo Foo::sorted() const &{
    Foo ret(*this);
    return ret.sorted();//不是函数返回语句，因为其先执行了sorted再试图返回
    //编译器会认定其为左值，仍然用左值引用版本，导致递归循环...
}

Foo Foo::sorted() const &{
    return Foo(*this).sorted();//可以正确利用右值引用版本进行排序，编译器认为Foo(*this)是一个“无主”右值，对它调用sorted会匹配右值引用版本
}
```