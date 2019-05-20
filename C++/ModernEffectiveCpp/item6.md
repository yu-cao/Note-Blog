##当`auto`推导出非预期类型时应该使用显式类型初始化

```cpp
//这个函数接受一个Widget返回一个类型为bool的vector，每个bool表征Widget是否具有某个特性
std::vector<bool> feature(const Widget& w);

Widget w;
...
bool highPriority = feature(w)[5];//w的第6个特性是否具有高优先级
...
processWidget(w, highPriority);//配合优先级处理w
```

这份代码是符合我们预期的，**但是我们把`highPriority`的显式类型变换成`auto`后，`processWidget`会导致未定义行为，因为`auto`推导得到的不是`bool`！**而是一个`std::vector<bool>::reference`对象（这是`vector<bool>`所独有的问题，其他的`vector`都是返回引用）

为什么会这样？因为`bool`这个数据类型很特殊，每个`bool`对应一个bit，这给`std::vector::operator[]`带来了麻烦，而`std::vector<bool>::reference`就是对`bool`数据封装的模板特化。

换句话说：**原则上要求`std::vector<T>`的`operator[]`函数应该返回`T&`，但是C++又禁止bits的引用，这导致没有`bool&`这样的类型，于是使用了这样的一个行为上与`bool&`相似的对象。**这给返回的类型的对象必须能在`bool&`能存在的语境下使用，也就是`std::vector<bool>::reference`隐式转换成`bool`才能够使得其操作成立。

重回代码，`feature`返回了`std::vector<bool>`对象，调用`operator[]`，返回一个`vector<bool>::reference`，然后隐式转换成`bool`类型以对`bool`类型的`highPriority`变量进行初始化，也就是调用了`std::vector<bool>`的第6个bit的数值来初始化，符合我们的设计。

但是使用`auto`，我们的隐式转换因为`auto`的类型而消失了，这里`highPriority`压根就没有得到我们希望的第6个bit上的数值。

这时候，代码崩溃就在一线之间，这与`std::vector<bool>::reference`如何实现相关。一种实现是：这样的对象包含一个指向包含bit引用的的机器word指针，在word上进行偏移。这时调用`features`会返回一个临时的`std::vector<bool>`对象，这个对象是匿名的，我们在这里称其为`temp`，调用`operator[]`返回的`std::vector<bool>::reference`返回一个由`temp`管理的一个指向一个包含bits的数据结构的指针，在word上偏移5个bit。**而`auto`推导的类型也是`std::vector<bool>::reference`，这里发生的是直接的拷贝而没有任何隐式转换，也就是说`highPriority`的值是：`temp`中包含一个指向word的指针，加上偏移5个bit。但是`temp`作为返回值的右值在该句结束后销毁，此时自然而然`highPriority`内含了一个野指针，带来了未定义行为**

这就是一个代理类：某个类的存在是为了模拟和对外行为和另外一个类保持一致。**一个通用法则：“不可见”的代理类不能与`auto`配合使用。**因为这些代理类常常被设计为生命周期为单行语句，超过时被回收，导致未定义行为。

这时，我们的修复方法是使用当我们需要时，使用转换的策略

```cpp
auto highPriority = static_cast<bool>(features(w)[5]);
```