## C# in depth笔记1

C# 1代码：

```c#
using System.Collections;
public class Product
{
    private string name;

    public string Name
    {
        get { return name;}
    }

    private decimal price;

    public decimal Price
    {
        get { return price;}
    }

    public Product(string name, decimal price)
    {
        this.name = name;
        this.price = price;
    }

    public static ArrayList GetSampleProducts()
    {
        ArrayList list = new ArrayList();
        list.Add(new Product("West Side Story", 9.99m));
        list.Add(new Product("Assassins", 14.99m));
        list.Add(new Product("Frogs", 13.99m));
        list.Add(new Product("Sweeney Todd", 10.99m));
        return list;
    }

    public override string ToString()
    {
        return string.Format("{0}: {1}", name, price);
    }
}
```

问题：

+ `ArrayList` 没有提供与其内部内容有关的编译时信息，我甚至可以添加一个字符串而不是一个Product
+ 代码中为属性提供了公共的取值方法，如果要添加对应的赋值方法，也需要是公共的
+ 用于创建属性和变量的代码太过复杂

C# 2：

```c#
public class Product
{
    private string name;

    public string Name
    {
        get { return name;}
        private set { name = value; }
    }

    private decimal price;

    public decimal Price
    {
        get { return price;}
        private set { price = value; }
    }

    public Product(string name, decimal price)
    {
        Name = name;
        Price = price;
    }

    public static List<Product> GetSampleProducts()
    {
        List<Product> list = new List<Product>();
        list.Add(new Product("West Side Story", 9.99m));
        list.Add(new Product("Assassins", 14.99m));
        list.Add(new Product("Frogs", 13.99m));
        list.Add(new Product("Sweeney Todd", 10.99m));
        return list;
    }

    public override string ToString()
    {
        return string.Format("{0}: {1}", name, price);
    }
}
```

现在，属性有了私有的赋值方法，而且能够推断出`List<Product>` 是告知编译器列表中只包含Product。试图将一个不同的类型加入列表会报错。

C# 3：

```c#
public class Product
{
    public string Name { get; private set; }
    public decimal Price { get; private set; }

    public Product() { }

    public Product(string name, decimal price)
    {
        Name = name;
        Price = price;
    }

    public static List<Product> GetSampleProducts()
    {
        return new List<Product>
        {
            new Product("West Side Story",9.99m),
            new Product("Assassins", 14.99m),
            new Product("Frogs", 13.99m),
            new Product("Sweeney Todd", 10.99m)
        };
    }

    public override string ToString()
    {
        return string.Format("{0}: {1}", Name, Price);
    }
}
```

不再有任何代码与属性关联。没有地方访问`name`和`price`变量，所以我们在类中处处要使用属性，增强了一致性。

C# 4

```c#
public class Product
{
    private readonly string name;
    public string Name { get { return name;}  }

    private readonly decimal price;
    public decimal Price { get { return price;} }

    public Product() { }

    public Product(string name, decimal price)
    {
        this.name = name;
        this.price = price;
    }

    public static List<Product> GetSampleProducts()
    {
        return new List<Product>
        {
            new Product(name: "West Side Story", price: 9.99m),
            new Product(name: "Assassins", price: 14.99m),
            new Product(name: "Frogs", price: 13.99m),
            new Product(name: "Sweeney Todd", price: 10.99m)
        };
    }

    public override string ToString()
    {
        return string.Format("{0}: {1}", Name, Price);
    }
}
```

C# 4在保证代码清晰度的同时，还额外移除了易变性。如果既不允许共有改变（私有赋值方法），也不允许私有改变（提供readonly属性）。用命名实参更加清晰地调用构造函数和方法。

<hr>

排序：

C# 1：

```c#
public class ProductNameComparer : IComparer
{
    public int Compare(object x, object y)
    {
        Product first = (Product) x;
        Product second = (Product) y;
        return first.Name.CompareTo(second.Name);
    }
}
...
ArrayList products = Product.GetSampleProducts();
products.Sort(new ProductNameComparer());
foreach(Product product in products)
{
    Console.WriteLine (product);
}
```

这里需要额外引入一个类型进行排序，而且需要强制转换，但是这样如果程序员本身犯错，之前ArrayList是可以插入单独一个字符串的，这里要是转成`Product`，会直接导致出错。（字符串->Product）代码的可读性也低，不够优雅。

C# 2：引入泛型

```c#
public class ProductNameComparer : IComparer<Product>
{
    public int Compare(Product x, Product y)
    {
        return x.Name.CompareTo(y.Name);
    }
}
...
List<Product> products = Product.GetSampleProducts();
products.Sort(new ProductNameComparer());
foreach(Product product in products)
{
    Console.WriteLine (product);
}
```

C# 2：使用委托实例（匿名方法）

```c#
 List<Product> products = Product.GetSampleProducts();
 products.Sort(delegate(Product x, Product y) { return x.Name.CompareTo(y.Name); });
 foreach (Product product in products)
 {
     Console.WriteLine(product);
 }
```

C# 3：Lambda表达式

```c#
 List<Product> products = Product.GetSampleProducts();
 products.Sort((x, y) =>x.Name.CompareTo(y.Name));
 foreach (Product product in products)
 {
     Console.WriteLine(product);
 }
```

C# 3：使用拓展方法：

```c#
 List<Product> products = Product.GetSampleProducts();
 foreach (Product product in products.OrderBy(p => p.Name))
 {
     Console.WriteLine(product);
 }
```

<hr>

C# 1：

```c#
 ArrayList products = Product.GetSampleProducts();
 foreach (Product product in products)
 {
     if (product.Price > 10m)
     {
         Console.WriteLine(product);
     }
 }
```

C# 2：

```c#
List<Product> products = Product.GetSampleProducts();

Predicate<Product> test = delegate(Product p) { return p.Price > 10m; };//匿名方法
List<Product> matches = products.FindAll(test);

Action<Product> print = Console.WriteLine;//方法组转换，从现有方法创建委托
matches.ForEach(print);
```

可以放到一条语句中：

```c#
List<Product> products = Product.GetSampleProducts();
products.FindAll(delegate(Product p) { return p.Price > 10m; }).ForEach(Console.WriteLine);
```

但是这个委托还是看上去很烦，我们在合理位置使用Lambda表达式：C# 3

```c#
List<Product> products = Product.GetSampleProducts();
foreach (Product product in products.Where(p => p.Price > 10m))
{
    Console.WriteLine(product);
}
```

<hr>

处理未知数据（这里表达了price可为null来表示未知的价格）

```c#
private decimal? price;

public decimal? Price
{
    get { return price; }
    private set { price = value; }
}
```

显示价格未知的产品：C# 3

```c#
List<Product> products = Product.GetSampleProducts();
foreach (Product product in products.Where(p => p.Price == null))
{
 Console.WriteLine(product);
}
```

C# 4：可选参数（类似C++的构造函数缺省值）

```c#
Product p = new Product("Unreleased product");
public Product(string name, decimal? price = null)
{
    this.name = name;
    this.price = price;
}
```

<hr>

LINQ：Language Integrated Query（语言集成查询）

```c#
List<Product> products = Product.GetSampleProducts();
var filtered = from Product p in products
               where p.Price > 10
               select p;
foreach(Product product in filtered)
{
    Console.WriteLine(product);
}
```

查询表达式不适合比较简单的任务，而在复杂情况下它具有比传统更好的可读性与优雅。

看上去很像SQL，但是我们不需要数据库就能使用它，可以从任意来源（如XML）中获取数据。也可以支持LINQ转换成为SQL，让数据库执行查询，而不是取出来用.NET进行操作

<hr>

COM与动态类型

让数据出现在Excel中，使用COM来进行控制。

```c#
var app = new Application {Visible = false};
Workbook workbook = app.Workbooks.Add();
WorkSheet workSheet = app.ActiveSheet;
int row = 1;
foreach (var product in Product.GetSampleProducts().Where(p => p.Price != null))
{
    workSheet.Cells[row, 1].Value = product.Name;
    workSheet.Cells[row, 2].Value = product.Price;
    row++;
}

workbook.SaveAs(Filename: "demo.xls", FileFormat: XlFileFormat.xlWorkbookNormal);
app.Application.Quit();
```

动态语言交互：

C#与Python等交互，可以通过dynamic类型来调用其方法，访问属性，将其作为方法的参数进行传递，而且基本都在执行时进行。

<hr>

轻松编写异步代码

C# 5特性：异步函数。中断代码的执行而不阻塞线程。

关键字`async`，`await`等。

<hr>

| C# 1                                                       | C# 2                                                         | C# 3                                     | C# 4                                   | C# 5     |
| ---------------------------------------------------------- | ------------------------------------------------------------ | ---------------------------------------- | -------------------------------------- | -------- |
| 只读属性，弱类型集合                                       | 私有属性赋值方法，强类型集合                                 | 自动实现的属性，增强的集合与对象初始化   | 用命名实参更加清晰地调用构造函数与方法 |          |
| 弱类型的比较功能，不支持委托排序                           | 强类型的比较功能，委托比较，匿名方法                         | 表达式，拓展方法，允许列表保持未排序状态 |                                        |          |
| 条件与操作紧密耦合，两者都是硬编码的                       | 条件与操作分开，匿名方法使委托变得简单                       | Lambda表达式使条件变得更容易阅读         |                                        |          |
| 处理未知数据时：                                           |                                                              |                                          |                                        |          |
| 要么维护一个标志，要么更改引用类型的语义，要么利用一个魔数 | 可空类型避免了采用C# 1的各种繁琐的方案。语法糖进一步简化了编程 | 同左                                     | 可选参数简化了默认设置                 |          |
|                                                            |                                                              |                                          |                                        | 异步函数 |

