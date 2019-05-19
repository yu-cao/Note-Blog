<h2>Haskell阅读笔记 第二章</h2>

配置环境：主要参考[intellij-haskell插件官方文档](https://github.com/rikvdkleij/intellij-haskell/blob/master/README.md)和[StackOverflow](https://stackoverflow.com/questions/37754956/how-do-i-set-up-intellij-to-build-haskell-projects-with-stack)解决了Haskell在IntelliJ IDEA下的环境配置（稍微有点麻烦，需要耐心找找资料）

REPL是什么？

REPL是read-eval-print loop的缩写。REPL是一个交互式编程环境，可以输入代码，进行计算，看到结果，如果使用stack ghci，那么会出现

```
Prelude>
```

等的字样，你可以在其中写入你的函数等，退出是`:q`

那么，Prelude又是什么呢？

Prelude是一个标准函数库。打开GHCi或Stack GHCi会自动加载这些功能，因此无需做任何特殊操作即可使用。你可以关闭Prelude，此外还有其他可选的Prelude。Prelude包含了Haskell的基本包(base package)，如果被提到某个东西是"is base"的，那就意味着这个东西是在基本包中

但是，我们都是希望把程序写成一个文件，然后逐步进行构建的方法编程，例如：

```haskell
sayHello :: String -> IO ()
sayHello x = putStrLn ("Hello, " ++ x ++ "!")
```

`::`是一种写类型签名的方式，可以把这个想象成“has the type”。所以第一行代码的意义是：`sayHello`是一个具有`String->IO()`类型的东西。然后把它装载进Prelude

```
Prelude> :load test.hs
Prelude> sayHello "Haskell"
Hello, Haskell!
Prelude>
```

可能你会发下装载进去后的代码头不再是"Prelude"了，变成我本地电脑中的一样了，如果需要跳出，可以使用`:m`指令：

```
Prelude> :load test.hs
[1 of 1] Compiling Main             ( test.hs, interpreted )
Ok, one module loaded.
*Main> sayHello "Haskell"
Hello, Haskell!
*Main> :m
Prelude>
```

在我上面配的环境中，如果需要达到同样的效果，代码如下

```haskell
-- Main.hs
module Main where

import Lib

main :: IO ()
main = sayHello "Haskell"


-- Lib.hs
module Lib
    ( sayHello
    ) where

sayHello :: String -> IO ()
sayHello x = putStrLn ("Hello, " ++ x ++ "!")
```

<hr>

理解表达式

Haskell中的所有内容都是表达式或声明。表达式可以是值，值的组合和/或应用于值的函数。表达式计算结果。在字面量的情况下，计算的结果就是其本身。在算术方程的情况下，计算过程是计算运算符及其参数的过程。但是，即使并非所有程序都是关于算术的，但是所有Haskell的表达式都以类似的方式工作，以可预测，透明的方式计算出结果。表达式是我们程序的构建块，程序本身就是由较小的表达式构成的一个重要表达。

计算的过程其实就是前面第一章所说的化简的过程，如1 + 1可以通过β-reduce变成2，而2是不可化简的，所以结果就是2

<hr>

函数

表达式是Haskell程序的最基本单元，函数是特定类型的表达式。 Haskell中的函数与数学中的函数类似，也就是说它们将输入或输入集映射到输出。函数是应用于参数的表达式，始终返回结果。因为它们纯粹是由表达式构建的，所以当给出相同的值时，它们总是会计算出相同的结果。

与lambda演算一样，**Haskell中的所有函数都接受一个参数并返回一个结果**。理解这一点，我们可以通过思考：在Haskell中，当我们将多个参数传递给函数时，我们实际上是应用了一系列嵌套函数，每个函数对应一个参数。这称为柯里化(currying)

至今为止，我们都直接使用字面量进行计算，而没有使用变量或者一个抽象的文字值。Haskell允许我们对其中的需要重用的部分进行抽象

定义函数

函数具有相同的格式：以函数名称开头，接下来是函数的形参，中间由空格分割，接下来是一个等号，表达相等性，然后是一个表达式作为函数体，计算出返回值。

定义函数在GHCi和Haskell源码中有所区别，在GHCi中，写法是这样的：

```
Prelude> let triple x = x * 3
Prelude> triple 1
3
Prelude> triple 3
9
```

而在Haskell源文件中，我们写作

```haskell
triple x = x * 3
```

代码风格问题：函数名称、变量都应该以小写字母开头！

<hr>

计算

当我们谈论计算表达式时，实质是我们进行化简，直到表达式达到最简单的形式。一旦一个term达到最简单的形式，我们就说它是不可简化的或完成评估。通常，我们称之为一个值。 Haskell使用非严格求值（有时称为“惰性求值”）策略，该策略推迟对term的计算，直到他们被引用它们的其他term强迫。

值(values)是不可简化的，但函数对参数的应用是可以减少的。减少表达式意味着计算这些term，直到最后留下一个值。与lambda演算一样，应用程序就是计算：将函数应用于参数允许求值或化简

值是表达式，但是是不能够被进一步化简。值是化简的终点。

Haskell不会对所有内容进行计算至规范或标准形式，而只是会默认情况下计算弱头标准形式(weak head normal form, WHNF)。也就是说，我们需要明确：Haskell的惰性求值意味着不是所有东西都会立即简化到不可简化的形式，例如：

```haskell
(\f -> (1, 2+f)) 2
-- 通过WHNF化简得到
(1, 2 + 2)
```

注意，上面的化简结果是`(2+2)`而不是`4`，除非在最后需要的时候，否则将不会对`2+2`进行计算，这就是惰性求值

<hr>

中缀操作符

Haskell中的函数默认都是前缀语法，意味着函数将会应用在表达式的开头而不是中间。正如上面的`triple`函数那样`triple 1`得到结果为`3`

但是我们使用的运算符，这其实也是一种函数，但是默认情况下显示在中缀位置以符合我们的习惯。当然我们有时候也可以使用函数的中缀操作，只是在语法上需要一些更改；除此之外，我们也可以使用运算符的前缀表达式（使用括号）：

```
-- div是执行整数除法，而`/`执行分数除法
Prelude> 10 `div` 4
2
Prelude> div 10 4
2


Prelude> 10 / 4
2.5
Prelude> (/) 10 4
2.5
```

相关性与优先权

我们可以通过`:info`来得到各个运算符之间的优先级顺序，其中`infixl`意味着是中缀表达式，满足左结合，`6`或者`7`是优先级，越高的优先级将会被越早执行，总共是`0~9`个级别，最后一个是中缀函数的名称，加法/减法...

```
Prelude> :info (+) (-) (*) (/)
class Num a where
  (+) :: a -> a -> a
  ...
  	-- Defined in ‘GHC.Num’
infixl 6 +
class Num a where
  ...
  (-) :: a -> a -> a
  ...
  	-- Defined in ‘GHC.Num’
infixl 6 -
class Num a where
  ...
  (*) :: a -> a -> a
  ...
  	-- Defined in ‘GHC.Num’
infixl 7 *
class Num a => Fractional a where
  (/) :: a -> a -> a
  ...
  	-- Defined in ‘GHC.Real’
infixl 7 /
```

稍微看个不一样的，`^`乘方，`infixr`代表是中缀表达式，右结合。`8`代表了优先级比前面的乘法等的`7`优先级更高，最后的`^`代表中缀函数名：乘方。

```
Prelude> :info (^)
(^) :: (Num a, Integral b) => a -> b -> a 	-- Defined in ‘GHC.Real’
infixr 8 ^

-- 右结合的意思
Prelude> 2 ^ 3 ^ 4
2417851639229258349412352
Prelude> 2 ^ (3 ^ 4)
2417851639229258349412352
Prelude> (2 ^ 3) ^ 4
4096
```

<hr>

声明变量(declaring values)

这里与传统的，命令式语言不同

```haskell
-- test.hs
module Learn where

x = 10 * 5 + y

myResult = x * 5

y = 10
```

可以得到：

```
Prelude> :load test.hs
[1 of 1] Compiling Learn            ( test.hs, interpreted )
Ok, one module loaded.
*Learn> x
60
*Learn> y
10
*Learn> myResult
300
```

找bug，以下是初学者非常容易犯一些错误的：

第一，要记住的一件事是**Haskell代码的缩进很重要，可以改变代码的含义**。代码缩进不正确也会破坏您的代码。提醒：使用空格而不是制表符来缩进源代码。

Haskell中，空格是函数调用的唯一标记，除非使用括号来修改函数的优先级，在行尾出现多余空格是不好的代码习惯

在源代码文件中，缩进通常会替换其他语言中大括号，分号和括号等语法标记。基本规则是作为表达式一部分的代码应该在该表达式的开头缩进，即使表达式的开头不在最左边缘。此外，分组的表达式部分应缩进到同一级别：

```haskell
-- 正确
let x = 3
    y = 4
-- 或者
let
  x = 3
  y = 4
  
-- 语法错误
let x = 3
 y = 4
-- 或者
let
 x = 3
  y = 4
  
-- 在多级表达式的情况下也必须要遵守这样的缩进规则
foo x =
    let y = x * 2
        z = x ^ 2
    in 2 * y * z
```

应用规则：作为表达式一部分的代码应该在该表达式的开头缩进，即使表达式的开头不在最左边缘。

```haskell
-- 错误：顶格写
x = 10
* 5 + y

-- 正确1
x = 10 * 5 + y
-- 正确2（至少空了1格）
x = 10
 * 5 + y
```

第二个容易出错的地方是没有在该开头进行声明：（注意下面的代码，x前面有一个空格）

```haskell
-- learn.hs
module Learn where
 x = 10 * 5 + y
myResult = x * 5
y = 10
```

注意：模块中的所有声明必须从同一列开始。在一个模块中的所有声明开始的列都必须与模块中的第一个声明对齐。

此外要小心，注释是`--`，是由两个`-`构成，错误输入注释符号也会导致奇怪的报错

<hr>

在Haskell中的算术函数

几个需要注意的地方，`div`和`quot`，`mod`和`rem`的区别

```
-- rounds down
Prelude> div 20 (-6)
-4

-- rounds toward zero（quot是向零取整的）
Prelude> quot 20 (-6)
-3
```

对于商(quotients）和余数（remainders）有以下定义（通过以下定义也明确了到底这两对的差别是什么地方）：

```
(quot x y)*y + (rem x y) == x
(div x y)*y + (mod x y) == x
```

例如`x = 10 y = -4`，有`quot 10 (-4)`结果为`-2`，`rem 10 (-4)`结果为2，有`(-2)*(-4) + (2) == 10`；同理有`div 10 (-4)`结果为`-3`，`mod 10 (-4)`结果为`-2`，所以计算结果是`(-3)*(-4) + (-2) == 10`

使用`mod`

