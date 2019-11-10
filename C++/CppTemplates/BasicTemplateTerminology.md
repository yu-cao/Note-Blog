## Chapter 10 Basic Template Terminology

### “类模板”还是“模板类”？（Class Template or Template Class）

对于如何调用作为模板的类存在一些困惑：
类模板（class template）指出该类是模板。也就是说，它是一类类的参数化描述。
另一方面，模板类（template class）被用作：

+ 作为类模板的同义词
+ 引用从模板生成的类
+ 引用名称为template-id的类（模板名称后跟<和>之间指定的模板参数的组合）

后面两种很晦涩而且不是很重要，所以我们避免使用这个术语。

### 替代化，实例化和特例化（Substitution Instantiation and Specialization）

在处理使用模板的源代码时，C ++编译器必须在不同时间用具体的模板参数*替换（substitute）*模板中的模板参数。有时，这种替换只是暂时的：编译器可能需要检查该替换是否有效

通过用模板模板的具体参数替换模板来为模板实际创建常规类，类型别名，函数，成员函数或变量的定义的过程称为模板实例化（template instantiation）

目前尚无标准或公认的术语来表示创建声明（declaration）的过程，该声明不是通过模板参数替换定义的。我们已经看到一些团队使用的术语：部分实例化（partial instantiation）或声明实例化（instantiation of a declaration），但这绝不是通用的。也许更直观的术语是不完整的实例化（在类模板的情况下，它会生成不完整的类）

由实例化或不完整实例化产生的实体（即类，函数，成员函数或变量）通常称为特例化（Specialization）

### 声明 Vs 定义（Declaration versus Definition）

声明（declaration）是一种C++结构，为的是将名称引入或重新引入C++ scope。此简介始终包括该名称的部分分类，但是进行有效声明时不需要详细信息。

当知道其结构的详细信息时，或者对于变量，必须分配存储空间时，声明将成为定义（Definition）。对于类的类型定义，这意味着必须提供一个用括号括起来的主体。对于函数定义，这意味着必须提供一个用大括号括起来的主体（通常情况下），或者必须将函数指定为`= default`或`= delete`。对于变量，初始化或缺少`extern`说明符会使声明成为定义。

同时需要注意complete type 与 incomplete type区别（其实很trivial）

### 注意单次定义原则（The One-Definition Rule）

C ++语言定义对各种实体的重新声明施加了一些约束。这些约束的总和称为ODR(The One-Definition Rule)。该规则的细节有些复杂，涉及多种情况。一般来说，现在只要记住以下ODR基本知识即可：

+ 普通（即不是模板）非内联函数和成员函数，以及（非内联）全局变量和静态数据成员应在整个程序中只定义一次

+ 类类型（包括结构和联合），模板（包括偏特化但不完全特化）以及内联函数和变量应该每个翻译单元最多定义一次，并且所有这些定义应该相同

翻译单元（translation unit）是预处理源文件的结果；也就是说，它包含由`#include`指令命名并由宏扩展产生的内容

### 模板参数 Vs 模板参数（Template Argument versus Template Parameter）

一句话概括：Parameters are initialized by arguments.或者精确来说：

+ Template Parameter是那些在模板声明或定义中在关键字`template`之后列出的那些名称（就是尖括号里面的）
+ Template Argument是代替模板参数的item（也就是实例化尖括号里面的具体类型）。与模板参数不同，模板参数可以不仅仅是“名称”
