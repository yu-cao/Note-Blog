## 优先使用`const_iterator`而不是`iterator`

`const_iterator`在STL中等价于指向`const`的指针。被指向的数值是不可以修改的。即**尽可能在没有必要修改指针所指向的内容的地方使用`const_iterator**`

在C++98中，这样的代码看似合理，实则是不正确的：

```cpp
typedef std::vector<int>::iterator IterT;
typedef std::vector<int>::const_iterator ConstIterT;
std::vector<int> values;

ConstIterT ci = std::find(static_cast<ConstIterT>(values.begin()), static_cast<ConstIterT>(values.end()),1983);
values.insert(static_cast<IterT>(ci),1998);//编译错误
```

在C++98中，插入或删除元素的定位只能用`iterator`而不能是`const_iterator`，所以，我们只能再把它变回去才可以，但是这里会出现编译错误的情况，即使你已经使用了`static_cast`也不行(`const_iterator`就是不能转换成`iterator`)。

**总而言之，`const_iterator`在C++98中是一个非常麻烦的事情，是万恶之源。那时候，开发者在必要的地方并不使用 `const_iterator` ，在C++98中 `const_iterator` 是非常不实用的。**

到了现在，`const_iterator`既容易获取也容易使用，`cbegin`与`cend`可以直接产生`const_iterator`，甚至非`cosnt`的容器也可以。

现在，STL中的成员函数常用`cosnt_iterator`进行定位(`insert`/`erase`)，借此我们可以修改上面的代码：

```cpp
auto it = std::find(values.cbegin(),values.cend(),1983);
values.insert(it, 1998);
```

`const_iterator`的短处在C++11以后是在写最大化泛型库的代码的时候，需要考虑一些容器的数据结构提供`begin`/`end`等作为非成员函数而不是成员函数。