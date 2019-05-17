<h2>auto关键字</h2>

我们要尽量引导`auto`得到正确的，符合我们预期的结果，尽管回到手动声明类型是一个万能解决方案，但是应该尽量避免

<h2>条款5：优先使用auto而非显式类型声明</h2>

这种命名是极度的sb：

```cpp
template <typename It>
void dwim(It b, It e)
{
	while(b != e)
	{
		typename std::iterator_traits<It>::value_type currValue = *b;
		//...
		
		//C++11
		//auto currValue = *b;
	}
}
```

我们在`C++11`之后使用`auto`完成了一些解脱，但是根据条款2所述，`auto`使用类型推导表达仅仅被编译器知晓的类型，直到C++14：

```cpp
auto dereUpLess = [](const std::unique_ptr<Widget> &p1, const std::unique_ptr<Widget> &p2)
{ return *p1 < *p2; };

//C++14 允许lambda参数可以包含auto
auto derefLess = [](const auto &p1, const auto &p2)
{ return *p1 < *p2; };
```

我们是否不需要使用`auto`来声明一个持有封装体的变量？因为我们有`std::function`啊！它可以使得函数指针普通化，函数指针只能指向一个函数，但是`std::function`对象可以应用任何可以被调用的对象，就像函数。

我们可以进行这样的操作，

```cpp
std::function<bool(const std::unique_ptr<Widget>&, const std::unique_ptr<Widget>&)> func;
```

而lambda表达式可以得到一个可调用对象，所以封装体存储在`std::function`中，意味着可以声明不使用`auto`的`dereUpLess`

```cpp
std::function<bool(const std::unique_ptr<Widget> &, const std::unique_ptr<Widget> &)>
		derefUPLess = [](const std::unique_ptr<Widget> &p1, const std::unique_ptr<Widget> &p2)
{ return *p1 < *p2; };
```

要注意，这里使用`std::function`和`auto`并不一样：

+ 使用`auto`声明持有一个封装的变量和封装体之间有相同类型也仅适用于封装体一样大小的内存。
+ 持有一个封装体的被`std::function`声明的变量类型是`std::function`这个模板的一个实例。对任何类型只有一个固定大小，当这个内存比封装体小，无法满足封装体需求时，`std::function`会调用堆空间进行封装体存储。

这使得`std::function`比`auto`消耗内存更大，调用慢，甚至出现内存不足的异常。

**`auto`优点除了可以避免未初始化的变量，变量声明引起的歧义，直接持有封装体的能力。还有一个就是可以避免“类型截断”问题：**

```cpp
std::vector<int> v;
...
unsigned sz = v.size();//返回值是std::vector<int>::size_type，长度随操作系统平台变化，而unsigned始终是32bit，可能发生异常

//正确做法
auto sz = v.size();
```

更显然的例子证明`auto`的优势：

```cpp
std::unordered_map<std::string, int> m;
...
for (const std::pair<std::string, int>& p : m)
{
    ... // do something with p
}
```

这里`unordered_map`中的`key`是`const`的，也就是说，hash表中`std::pair`类型是`std::pair<const std::string, int>`，但是这不是`p`的声明类型。这使得编译器尝试寻找把`std::pair<const std::string, int>`对象变成`std::pair<std::string, int>`的对象（即`p`的对象），这过程会创建m的一个元素的复制到一个临时对象，然后将临时对象与`p`绑定完成，在每个循环结束时销毁这个临时对象。这与我们的本意大相径庭，而使用`auto`可以很好处理这个问题：

```cpp
for (const auto& p : m)
{
    ... // as before
}
```

而且当我们要取得`p`的地址时，我们的确得到了一个指向`m`中的某个元素的指针，而不使用`auto`，我们将只能得到一个指向临时对象的指针，这个临时对象将会在循环结束销毁，这可能会引发悬空指针等问题。

`auto`不是银弹！`auto`变量的类型是从初始化表达式进行推导的，而有一些推导出来的结果与我们的预期是不符的，这里我们可以选择手动处理等方式进行。

事实是显式地写出类型可能会引入一些难以察觉的错误，导致正确性或者效率问题，或者两
者兼而有之。除此之外，`auto`类型会自动的改变如果初始化它的表达式改变后，这意味着通过使用`auto`可以使代码重构变得更简单。举个例子，如果一个函数被声明为返回`int`，但是你稍后决定返回`long`可能更好一些，如果你把这个函数的返回结果存储在一个`auto`变量中，在下次编译的时候，调用代码将会自动的更新。

<hr>

<h2>综上，我们要记住：</h2>

`auto`变量一定要被初始化，并且对由于类型不匹配引起的兼容和效率问题有免疫力，可
以简单化代码重构，一般会比显式的声明类型敲击更少的键盘