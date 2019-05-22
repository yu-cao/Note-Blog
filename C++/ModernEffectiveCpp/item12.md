## 使用`override`关键字声明覆盖的函数

如果使用覆盖（`override`）的函数，需要满足以下条件：

+ 基类中函数被声明是虚的
+ 基类中和派生出的函数名称，参数类型，常量特性必须完全一致
+ 基类中和派生出的函数的返回值类型与异常声明可以兼容
+ 函数的引用修饰符必须完全一致(`C++11`）

```cpp
class Widget
{
    public:
    ...
    void doWork() &;//只有当*this为左值时才会调用此版本
    void doWork() &&;//只有当*this为右值时才会调用此版本
};
...
Widget makeWidget();

Widget w;
...

w.doWork();//左值调用

makeWidget().doWork();//右值调用
```

下面的`Base`和`Derived`中没有一个函数是被覆盖的，因为他们都不满足开头的要求，但是都不会报错，甚至部分编译器都不会警告

```cpp
class Base {
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    void mf4() const;
};
class Derived: public Base {
public:
    virtual void mf1();
    virtual void mf2(unsigned int x);
    virtual void mf3() &&;
    void mf4() const;
};
```

正确的避免这种错误的方法是在对所有的覆盖函数声明为`override`，这样编译器就会在你想要覆盖但是没有覆盖的时候给与合适的警告