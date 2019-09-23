## C# in depth笔记2.1

### 委托

类似C中的函数指针，委托类型定于定义了一个方法的接口，委托实例等于实现了那个接口的对象。

比方：一份遗嘱，在之前写好，然后放到一个安全的地方，等到合适的时间，（希望）律师会去执行这些指令。

简单委托构成：

+ 声明委托类型
+ 必须有一个方法包含了要执行的代码
+ 必须创建一个委托实例
+ 必须调用委托实例

1、声明委托类型：

```c#
delegate void StringProcessor(string input);
```

如果要创建`StringProcessor`的一个实例，只需要携带一个参数的方法，这个方法范围为`void`

委托被误解往往是因为大家喜欢用委托描述委托类型与委托实例

2、为委托实例的操作找到一个恰当的方法

希望这个方法能够做我们想做的事情，并且具有和委托类型相同的签名。也就是说，**invoke一个委托实例时，参数要完全匹配，并且能以期望的方式使用返回值。**

```c#
void PrintString(string x)//符合需求
void PrintInteger(int x)//参数类型不匹配
void PrintTwoStrings(string x, string y)//参数数量不匹配
int GetStringLength(string x)//返回值不匹配
void PrintObject(object x)
```

第五个方法比较特别，因为string是object派生，所以任何时候调用StringProcessor实例，都可以调用具有相同参数的PrintObject方法。

3、创建委托实例

```c#
StringProcessor proc1, proc2;
proc1 = new StringProcessor(StaticMethods.PrintString);
InstanceMethods instance = new InstanceMethods();
proc2 = new StringProcessor(instance.PrintString);
```

注意区分静态方法与动态方法，动态方法需要先建立起一个类型的实例再调用

4、调用委托实例

调用一个委托实例的方法就可以了，这个方法本身称为Invoke

```c#
void Invoke(string input)
```

调用Invoke会执行委托实例的操作，向他传递Invoke时指定的任何参数。

`proc1("Hello");` 编译成=> `proc1.Invoke("Hello");` 执行时调用=> `PrintString("Hello");`

```c#
using System;
delegate void StringProcessor(string input);//声明委托类型

class Person
{
    private string name;

    public Person(string name)
    {
        this.name = name;
    }

    public void Say(string message)//声明兼容的实例方法
    {
        Console.WriteLine("{0} says: {1}", name, message);
    }
}

class Background
{
    public static void Note(string note)//声明兼容的静态方法
    {
        Console.WriteLine("({0})", note);
    }
}

class SimpleDelegateUse
{
    static void Main()
    {
        Person jon = new Person("Jon");
        Person tom = new Person("Tom");
        
        //创建3个实例
        StringProcessor jonsVoice, tomsVoice, backgound;
        jonsVoice = new StringProcessor(jon.Say);
        tomsVoice = new StringProcessor(tom.Say);
        backgound = new StringProcessor(Background.Note);
        
        //调用委托实例
        jonsVoice("Hello, son.");
        tomsVoice.Invoke("Hello, Daddy!");
        backgound("An airplane flies past.");
    }
}
```

为啥我们像这样的代码不使用直接调用而是使用委托？我们不能仅仅由于你希望某事发生，就一定会在正确的时间与地点出现，并且亲自发生。有时候我们需要给出一些指令去委托给别人。

委托是指定一部分代码在特定时间内执行，那时候也许已经无法更改要执行的代码了。比如说：单击一个按钮后发生某事，但是不想对按钮的代码进行更改，只是希望按钮可以调用我期望的那个方法。委托就是间接完成某种操作，增加了复杂性，提高了灵活性。

#### 合并与删除委托

委托实例实际有一个操作列表与之相关，称为委托实例的调用列表。

`System.Delegate`类型的静态方法Combine负责将两个委托实例的调用列表连接起来，Remove负责从一个委托实例中删除另一个实例的调用列表。

委托是不易变的——一旦创建了委托实例，那么关于它的一切都不能改变。这样我们就能大胆传递委托实例的引用，并且把它们和其他委托实例合并而不用担心一致性，线程安全性与会否有人尝试进行更改等情况。这里，可以把委托实例看成跟string一样，实例不易变，Delegate.Combine与String.Concat很像（合并现有实例形成一个新实例）如果null和委托实例合并，null将被视为带有空调用列表的一个委托

我们往往看不到Delegate.Combine的显式调用，而往往是通过使用`+`或者`+=`来完成合并；同理，我们往往使用`-`或者`-=`来代替Delegate.Remove

如果委托的签名是一个非void返回类型，则Invoke返回值是最后一个操作的返回值。

调用列表中的任何异常都会阻止后续操作，假定调用一个委托实例，操作列表如果为[a, b, c]，但是b抛出异常，那么异常会立刻传播，操作c不会执行

进行事件处理时，委托实例的合并与删除会格外有用

事件不是委托类型的字段！而应该看成一种属性。两者都声明为具有一种特定的类型，对于事件来说，必须是一个委托类型。

使用属性时候你直接对它的字段进行取值，赋值等，实际上是在调用方法。

在订阅/取消订阅一个事件的时候，看上去就像在通过`+=`或者`-=`运算符使用委托类型的字段，但事实上跟属性一样只是在调用方法。就像属性我们不希望外部代码能直接控制属性值一样，我们也不希望外部代码可以之间随意更改或调用一个事件的处理程序，至少需要通过所有者提供的API接口进行。

字段风格的事件让这些事件变得更易阅读。只要一个声明，编译器就会把声明转换成一个具有默认add/remove实现的事件和一个私有委托类型的字段。类内的代码看见字段，类外的代码只能看见事件。这样表面上看起来能调用一个事件，实际上是调用存储在字段中的委托实例。

总结：

+ 委托封装了包含特殊返回值类型和一组参数的行为，类似包含单一方法的接口
+ 委托类型声明中所描述的类型签名决定了哪个方法可以用于创建委托实例
+ 为了创建委托实例，我们需要一个方法以及调用方法的目标
+ 委托实例是不易变的
+ 每个委托实例都包含了一个调用列表
+ 委托实例可以合并与删除
+ 事件不是委托实例，而是成对的add/remove方法（类似属性的取值赋值方法）