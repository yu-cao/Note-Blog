```cpp
class A
{
public:
    A();
    A(A &);
    ~A();
};

A::A()
{
    cout << "执行构造函数创建一个对象\n" << endl;
}

A::A(A&)
{
    cout << "执行复制构造函数创建该对象的副本\n";
}

A::~A()
{
    cout << "执行析构函数删除该对象" << endl;
}

A func();

int main()
{
    A a;
    A q = func();
    return 0;
}

A func()
{
    A m;
    return m;
}
```

这段代码在g++或者clang++上都会报错（但是在Mac上的llvm和Windows下的Visual Studio都正常运行）:

```
main.cpp:35:7: error: no matching constructor for initialization of 'A'
    A q = func();
      ^   ~~~~~~
main.cpp:20:4: note: candidate constructor not viable: expects an l-value for 1st argument
A::A(A&)
   ^
main.cpp:15:4: note: candidate constructor not viable: requires 0 arguments, but 1 was provided
A::A()
   ^
1 error generated.
```

原因如下：func()返回一个rvalue，而一个rvalue并不能绑定在一个non-const的A&构造函数上，而可以是const A&。在类A中定义了个一个复制构造函数后，编译器不会再生成默认的const构造函数，所以找不到合适的复制构造函数了。修改成A::A(const &A)即可。

有点类似的问题《C++ Primer》P266 练习7.49(2)：**核心：非常量的左值不能与一个临时对象进行绑定（理解很简单，要是可以绑定那么你修改的不就是临时变量，这样的修改肯定毫无意义，所以编译器直接否决了这样的做法）**

```cpp
class Sales_data
{
public:
    
    /* Non-const lvalue reference to type 'Sales_data' cannot bind to a value of unrelated type 'std::__1::string' 
     * (aka 'basic_string<char, char_traits<char>, allocator<char> >')*/
    Sales_data &combine(Sales_data &);// ERROR
    
    // 正确写法
    Sales_data &combine(const Sales_data &);
};

int main()
{
    Sales_data i;
    string s;

    i.combine(s);
}
```