<h2>条款4：查看类型推导</h2>

1、编译时编译器诊断（具体内容略，too naive）

2、运行时输出

```cpp
std::cout << typeid(x).name() << '\n';
```

有一个更加复杂的例子，涉及自定义类型，STL容器，`auto`变量：

```cpp
class Widget{};

template <typename T>
void f(const T& param)
{
	std::cout << "T = " << typeid(T).name() << '\n';
	std::cout << "param = " << typeid(param).name() << '\n';
}

std::vector<Widget> createVec()
{
	Widget a;
	return vector<Widget>(1,a);
}

const auto vw = createVec();

int main()
{
	if(!vw.empty())
	{
		f(&vw[0]);
		//....
	}
}

//Result:
T = PK6Widget
param = PK6Widget
```

其中，`PK`代表了“point to `const`”，而后面的6表示后面跟着的类的名字(`Widget`)的字符长度，即告诉我们类型都是`const Widget*`

但是这个返回结果是有问题的，显然我们分析可知`param`的类型是`const T&`，这两者不应该是相同的类型，也就是说`typeid`的结果是不可靠的，因为这里特化指定了类型会被当做它们被传给模板函数的时候的按值传递的参数，也就会如条款1所述丢失引用，`const`（和`volatile`）特性，同样的，IDE编辑器的类型推断也会不可信

使用boost库可能会有更好的效果，`boost::typeindex::type_id_with_cvr`接受一个类型参数来正常工作，且不会去除`const`，`volatile`和引用特性（即模板中的`cvr`三者），返回结果为`boost::typeindex::type_index`对象，使用`pretty_name`产生一个`std::string`进行展示输出

```cpp
template <typename T>
void f(const T& param)
{
	using boost::typeindex::type_id_with_cvr;

	std::cout << "T = " << type_id_with_cvr<T>().pretty_name() << '\n';
	std::cout << "param = " << type_id_with_cvr<decltype(param)>().pretty_name() << '\n';
}
```

但是这些都只是替代品而已，只是一个可能不是很可靠的提示，分析还是需要我们自己进行

<hr>

<h2>综上，我们要记住：</h2>

+ 类型推导的结果常常可以通过IDE的编辑器，编译器错误输出信息和Boost TypeIndex库的结果中得到
+ 一些工具的结果不一定有帮助性也不一定准确，所以对C++标准的类型推导法则加以理解是很有必要的