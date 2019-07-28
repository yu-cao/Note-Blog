## Lua 学习笔记2

### 语言

#### 词汇约定

Lua是一种自由形式的语言。它忽略了空格（包括新行）和词法元素（标记）之间的注释，除了名称和关键字之间的分隔符。

Lua中的名称（也称为标识符）可以是任何字母，数字和下划线字符串，不以数字开头而不是保留字。标识符用于命名变量，表字段和标签。

```
     and       break     do        else      elseif    end
     false     for       function  goto      if        in
     local     nil       not       or        repeat    return
     then      true      until     while
```

上面的作为关键字，注意Lua是大小写敏感的语言。通常，用下划线开头并且后面跟着大写的都作为保留字，不要使用这样的名字

转义字符与C语言的类似，不再赘述。此外，Lua支持小数，浮点数，使用E/e的科学记数，16进制常量（0x开头），二进制指数P等。以下五个都表达相同的意思，可以帮助记忆：

```lua
     a = 'alo\n123"'
     a = "alo\n123\""
     a = '\97lo\10\04923"'
     a = [[alo
     123"]]
     a = [==[
     alo
     123"]==]
```

注释以字符串外的任何位置的双连字符（--）开头。如果紧跟在后面的文本不是一个开头的长括号，则注释是一个简短的注释，它会一直运行到行尾。否则，它是一个长注释，直到相应的结束长括号。长注释经常用于临时禁用代码。

#### 变量

变量是存储值的地方。 Lua中有三种变量：全局变量，局部变量和表字段。

单个名称可以表示全局变量或局部变量（或作为函数的形参，它是一种特殊的局部变量）：

```lua
var ::= Name
```

名称表示标识符，除非明确声明为本地变量名称，否则任何变量名称都被假定为全局变量名称。在第一次赋值之前，它的值为nil

方括号用于索引表，访问表字段的含义可以通过metatable进行改变：

```lua
var ::= prefixexp ‘[’ exp ‘]’
```

对于全局变量x的访问等同于`_ENV.x`，由于编译块的问题，`_ENV`永远不会是全局名称

### 声明

Lua支持几乎所有的常规语句集，类似于Pascal或C中的语句。该集包括赋值，控制结构，函数调用和变量声明。

#### Blocks

block是一系列的声明，它们将会被顺序执行。`block ::= {stat}`

Lua有空的声明，这允许你用分号分隔语句，用分号开始一个块或按顺序写两个分号：`stat ::= ';'`

函数调用和赋值可以以左括号开头。这种可能性导致了Lua语法的模糊性。请考虑以下片段：

```lua
a = b + c
(print or io.write)('done')
```

这里的语法出现了歧义：

```lua
a = b + c(print or io.write)('done')

a = b + c; (print or io.write)('done')
```

当前解析器总是以第一种方式看到这样的结构，将开括号解释为调用参数的开始。为避免这种歧义，最好始终在以括号开头的分号语句之前：

```lua
;(print or io.write)('done')
```

可以明确分隔块以生成单个语句：

```lua
stat ::= do block end
```

显式块对于控制变量声明的范围很有用。显式块有时也用于在另一个块的中间添加return语句

#### Chunks

Lua的编译单元称为块(chunk)。从语法上讲，块(chunk)只是一个块(block)：`chunk := block`

Lua将一个块作为具有可变数量参数的匿名函数的主体处理。因此，块可以定义局部变量，接收参数和返回值。此外，这种匿名函数被编译为名为`_ENV`的外部局部变量的范围。结果函数始终将`_ENV`作为其唯一的upvalue，即使它不使用该变量。

块可以存储在宿主程序内的文件或字符串中。为了执行一个块，Lua首先加载它，将块的代码预编译为虚拟机的指令，然后Lua用虚拟机的解释器执行编译的代码。

块也可以预编译成二进制形式;有关详细信息，请参阅program luac和函数string.dump。源代码和编译形式的程序是可以互换的; Lua自动检测文件类型并相应地执行操作。

#### Assignment

Lua允许多赋值。因此，赋值语法定义左侧的变量列表和右侧的表达式列表。两个列表中的元素用逗号分隔：

```lua
	stat ::= varlist ‘=’ explist
	varlist ::= var {‘,’ var}
    explist ::= exp {‘,’ exp}
```

在赋值之前，将值列表调整为变量列表的长度。如果值多于所需值，则会丢弃多余的值。如果值少于所需值，则列表将根据需要扩展为nil。如果表达式列表以函数调用结束，则该调用返回的所有值在调整之前输入值列表（除非将调用括在括号中）。赋值语句首先计算其所有表达式，然后才执行赋值（下面的语句中，将a[3]设为20，而不是a[4]）：

```lua
     i = 3
     i, a[i] = i+1, 20


-- 交换x y
    x, y = y, x
```

对全局名称x = val的赋值等同于赋值_ENV.x = val。表字段和全局变量（实际上也是表字段）的赋值的含义可以通过元表更改

#### Control Structures

控制结构主要有3个关键字：if，while，repeat，此外还有两种版本的for结构

```lua
	stat ::= while exp do block end
	stat ::= repeat block until exp
	stat ::= if exp then block {elseif exp then block} [else block] end
```

控制结构的条件表达式可以返回任何值。`false`和`nil`都被认为是假的。不同于`nil`和`false`的所有值都被认为是真的（特别是，数字0和空字符串也是真的）

在`repeat-until`循环中，内部块不以`until`关键字结束，而是仅在条件之后结束。因此，条件可以引用循环块内声明的局部变量。

`goto`语句将程序控制转移到标签。Lua中的标签也被视为语句：

```lua
	stat ::= goto Name
	stat ::= label
	label ::= ‘::’ Name ‘::’
```

标签在定义它的整个块中是可见的，除了嵌套块内，其中定义了具有相同名称的标签并在嵌套函数内。只要goto没有进入局部变量的范围，它就可以跳转到任何可见标签。

标签和空语句称为void语句，因为它们不执行任何操作。

break语句终止执行while，repeat或for循环，跳转到循环后的下一个语句：`stat ::= break`

`break`结束最里面的封闭循环。

`return`语句用于从函数或块（这是一个匿名函数）返回值。函数可以返回多个值，因此return语句的语法是`stat ::= return [explist] [‘;’]`

`return`语句只能写为块的最后一个语句。如果确实需要在块的中间返回，那么可以使用显式内部块，如在`do return end`，因为现在返回是其（内部）块中的最后一个语句。

#### for statement

for语句有两种形式：一种是数字形式，一种是通用形式。

数字形式的for循环重复一个block的代码，而控制变量则通过算术运算控制。它具有以下语法：`stat ::= for Name ‘=’ exp ‘,’ exp [‘,’ exp] do block end`

```lua
    for v = e1, e2, e3 do block end
    -- 等价于：
     do
       local var, limit, step = tonumber(e1), tonumber(e2), tonumber(e3)
       if not (var and limit and step) then error() end
       var = var - step
       while true do
         var = var + step
         if (step >= 0 and var > limit) or (step < 0 and var < limit) then
           break
         end
         local v = var
         block
       end
     end
```

泛型for语句适用于函数，称为迭代器。在每次迭代时，调用迭代器函数以生成新值，在此新值为nil时停止。泛型for循环具有以下语法：

```lua
stat ::= for namelist in explist do block end
	namelist ::= Name {‘,’ Name}
```

for语句类似如下：

```lua
    for var_1, ···, var_n in explist do block end
    -- 等价于
     do
       local f, s, var = explist
       while true do
         local var_1, ···, var_n = f(s, var)
         if var_1 == nil then break end
         var = var_1
         block
       end
     end
```

#### Function Calls as Statements

为了允许可能的副作用，函数调用可以作为语句执行：`stat ::= functioncall`。在这种情况下，所有返回的值都将被丢弃

#### Local Declarations

局部变量可以在块内的任何位置声明。声明可以包括初始分配：`stat ::= local namelist [‘=’ explist]`。如果存在，初始赋值具有与多赋值相同的语义。否则，所有变量都用nil初始化。

### Expressions

```lua
	exp ::= prefixexp
	exp ::= nil | false | true
	exp ::= Numeral
	exp ::= LiteralString
	exp ::= functiondef
	exp ::= tableconstructor
	exp ::= ‘...’
	exp ::= exp binop exp
	exp ::= unop exp
    prefixexp ::= var | functioncall | ‘(’ exp ‘)’
```

函数调用和vararg表达式都可能导致多个值。如果函数调用用作语句，则其返回列表将调整为零元素，从而丢弃所有返回的值。如果表达式用作表达式列表的最后一个（或唯一）元素，则不进行任何调整（除非表达式括在括号中）。在所有其他上下文中，Lua将结果列表调整为一个元素，要么丢弃除第一个之外的所有值，要么在没有值时添加单个nil。

```lua
     f()                -- adjusted to 0 results
     g(f(), x)          -- f() is adjusted to 1 result
     g(x, f())          -- g gets x plus all results from f()
     a,b,c = f(), x     -- f() is adjusted to 1 result (c gets nil) 即一一对应
     a,b = ...          -- a gets the first vararg argument, b gets
                        -- the second (both a and b can get nil if there
                        -- is no corresponding vararg argument)
     
     a,b,c = x, f()     -- f() is adjusted to 2 results
     a,b,c = f()        -- f() is adjusted to 3 results
     return f()         -- returns all results from f()
     return ...         -- returns all received vararg arguments
     return x,y,f()     -- returns x, y, and all results from f()
     {f()}              -- creates a list with all results from f()
     {...}              -- creates a list with all vararg arguments
     {f(), nil}         -- f() is adjusted to 1 result
```

在括号中的任何表达式始终只生成一个值。因此，`(f(x,y,z))`总是单个值，即使f返回多个值。

#### Arithmetic Operators

`+ - * / // % ^ -`

#### Bitwise Operators

`& | ~ >> << ~`(一个~是按位异或，一个~是取反)

#### Coercions and Conversions

Lua在运行时提供某些类型和表示之间的一些自动转换。按位运算符总是将浮点操作数转换为整数。 Exponentiation和float除法总是将整数操作数转换为浮点数。应用于混合数字（整数和浮点数）的所有其他算术运算将整数操作数转换为浮点数;这被称为通常的规则。根据需要，C API还将整数转换为浮点数并将浮点数转换为整数。此外，字符串连接除了字符串之外还接受数字作为参数。

每当有数字时，Lua也会将字符串转换为数字。

在从整数到浮点数的转换中，如果整数值具有精确的表示形式，那么就是结果。否则，转换得到最接近的较高或最接近的较低可表示值。这种转换永远不会失败。

从float到integer的转换检查float是否具有整数的精确表示（即，float具有整数值，并且它在整数表示的范围内）。如果是，则表示结果。否则，转换失败。

从字符串到数字的转换如下：首先，按照语法和Lua词法分析器的规则将字符串转换为整数或浮点数。（该字符串可能还有前导和尾随空格以及符号。）然后，结果数字（浮点数或整数）将转换为上下文所需的类型（浮点数或整数）（例如，强制转换的操作）。

从字符串到数字的所有转换都接受dot和当前区域设置标记作为基数字符（然而，Lua 文法只接受一个dot）

从数字到字符串的转换使用未指定的人类可读格式。如果要完全控制数字如何转换为字符串，请使用字符串库中的format函数

#### Relational Operators

`== ~= < > <= >=`返回值永远是true/false

`==`先比较类型，再比较值，类型不同，即为false（即"0" == 0一定为false）；table，userdata和thread比较只有当两个对象是同一个对象时才认为时相等的。而且我们可以通过使用`eq`的metamethod来更改tables和userdata的比较方式；`~=`为`==`的否定形式；后面的比较如果时两个数字，直接比，如果是两个string，它们的值比较就是根据当前的位置决定，否则，调用`lt`和`le`两个metamethod

注意，NaN不小于，不等于，不大于任何值（包括自身）

#### Logical Operators

`and`、`or`和`not`，所有的逻辑操作只有false和nil认为是false的，其他任何内容视为true。`not`运算符并不总是返回false或true。连接运算符`and`返回它的第一个参数如果它的值是false/nil（短路效应），否则返回第二个值，类似的`or`也具有类似的短路效应：

```lua
     10 or 20            --> 10
     10 or error()       --> 10
     nil or "a"          --> "a"
     nil and 10          --> nil
     false and error()   --> false
     false and nil       --> false
     false or nil        --> nil
     10 and 20           --> 20
```

#### Concatenation

Lua中的字符串连接运算符由两个点`..`表示。如果两个操作数都是字符串或数字，则它们将根据规则转换为字符串。否则，调用`__concat`元方法

#### The Length Operator

长度运算符由一元前缀运算符`#`表示。字符串长度就是它的字节数，这个操作应用于table时，返回的时该table的border。对于一个在表t中的border，我们给出如下定义：`(border == 0 or t[border] ~= nil) and t[border + 1] == nil`

board为1的表称为序列，如：`{10, 20, 30, 40, 50}`。而`{10, 20, 30, nil, 50}`就有两个border；`{nil, 20, 30, nil, nil, 60, nil}`有三个border，`{}`有0个border

当t是序列时，#t返回其唯一的边界，这对应于序列长度的直观概念。当t不是序列时，＃t可以返回其任何边界。（确切的一个取决于表的内部表示的细节，而这又取决于表的填充方式和非数字键的内存地址）

表的长度的计算具有保证的最差时间O（log n），其中n是表中最大的自然键。

程序可以通过__len元方法修改长度运算符对任何值的行为，除了字符串

#### Precedence（优先级） 略

#### Table Constructors

表构造函数是创建表的表达式。每次评估构造函数时，都会创建一个新表。构造函数可用于创建空表或创建表并初始化其某些字段。构造函数的一般语法是

```lua
	tableconstructor ::= ‘{’ [fieldlist] ‘}’
	fieldlist ::= field {fieldsep field} [fieldsep]
	field ::= ‘[’ exp ‘]’ ‘=’ exp | Name ‘=’ exp | exp
    fieldsep ::= ‘,’ | ‘;’
```

形如`[exp1] = exp2`的每个字段向新表添加一个键为`exp1`且值为`exp2`的条目。形如`name = exp`的字段等同于`[“name”] = exp`。最后，exp形式的字段等价于`[i] = exp`，其中i是从1开始的连续整数。其他格式的字段不影响此计数。例如，

```lua
     a = { [f(1)] = g; "x", "y"; x = 1, f(x), [30] = 23; 45 }

     -- 等价于
     do
       local t = {}
       t[f(1)] = g
       t[1] = "x"         -- 1st exp
       t[2] = "y"         -- 2nd exp
       t.x = 1            -- t["x"] = 1
       t[3] = f(x)        -- 3rd exp
       t[30] = 23
       t[4] = 45          -- 4th exp
       a = t
     end
```

构造函数中赋值的顺序是未定义的；如果列表中的最后一个字段的形式为exp，而表达式是函数调用或vararg表达式，则此表达式返回的所有值将连续输入列表。字段列表可以有一个可选的尾随分隔符，以方便机器生成的代码。

#### Function Calls

`functioncall ::= prefixexp args`在函数调用中，首先计算prefixexp和args。如果prefixexp的值具有type函数，则使用给定的参数调用此函数。否则，调用prefixexp“call”元方法，将prefixexp的值作为第一个参数，后跟原始调用参数

`functioncall ::= prefixexp ‘:’ Name args`这个形式可以被用来调用method。`v:name(args)`是`v.name(v, args)`的语法糖（除非v是第一次被计算）

参数需要符合以下语法：

```lua
	args ::= ‘(’ [explist] ‘)’
	args ::= tableconstructor
    args ::= LiteralString
```

在调用之前计算所有参数表达式。 f {fields}形式的调用是f（{fields}）的语法糖;也就是说，参数列表是一个新表。形式为f'字符串'（或f“字符串”或f [[string]]）的调用是f（'string'）的语法糖;也就是说，参数列表是单个文字字符串。

`return functioncall`形式的调用称为尾调用。 Lua实现了正确的尾调用（或正确的尾递归）：在尾调用中，被调用函数重用调用函数的堆栈条目。因此，程序可以执行的嵌套尾调用的数量没有限制。但是，尾调用会删除有关调用函数的所有调试信息。请注意，尾调用只发生在特定的语法中，其中return 只有一个函数调用 作为参数；这种语法使调用函数完全返回被调用函数的返回值。因此，以下示例都不是尾调用：

```lua
     return (f(x))        -- results adjusted to 1
     return 2 * f(x)
     return x, f(x)       -- additional results
     f(x); return         -- results discarded
     return x or f(x)     -- results adjusted to 1
```

#### Function Definitions

```lua
	functiondef ::= function funcbody
	funcbody ::= ‘(’ [parlist] ‘)’ block end

    --语法糖
    stat ::= function funcname funcbody
	stat ::= local function Name funcbody
	funcname ::= Name {‘.’ Name} [‘:’ Name]
```

```
The statement
     function f () body end
translates to
     f = function () body end

The statement
     function t.a.b.c.f () body end
translates to
     t.a.b.c.f = function () body end

The statement
     local function f () body end
translates to
     local f; f = function () body end
not to
     local f = function () body end
```

函数定义是可执行表达式，其值具有类型函数。当Lua预编译一个块时，它的所有函数体也都被预编译。然后，每当Lua执行函数定义时，该函数被实例化（或关闭）。此函数实例（或闭包）是表达式的最终值。

参数充当使用参数值初始化的局部变量：`parlist ::= namelist [‘,’ ‘...’] | ‘...’`

调用函数时，参数列表将调整为参数列表的长度，除非该函数是var arg函数，该参数由参数列表末尾的三个点（'...'）表示。 var arg函数不调整其参数列表;相反，它收集所有额外的参数并通过var arg表达式将它们提供给函数，该表达式也写为三个点。此表达式的值是所有实际额外参数的列表，类似于具有多个结果的函数。如果在另一个表达式内或表达式列表的中间使用vararg表达式，则将其返回列表调整为一个元素。如果表达式用作表达式列表的最后一个元素，则不进行任何调整（除非最后一个表达式括在括号中）。

```lua
     function f(a, b) end
     function g(a, b, ...) end
     function r() return 1,2,3 end

     CALL            PARAMETERS
     
     f(3)             a=3, b=nil
     f(3, 4)          a=3, b=4
     f(3, 4, 5)       a=3, b=4
     f(r(), 10)       a=1, b=10
     f(r())           a=1, b=2
     
     g(3)             a=3, b=nil, ... -->  (nothing)
     g(3, 4)          a=3, b=4,   ... -->  (nothing)
     g(3, 4, 5, 8)    a=3, b=4,   ... -->  5  8
     g(5, r())        a=5, b=1,   ... -->  2  3
```

使用return语句返回结果。如果控件到达函数的末尾而没有遇到return语句，则函数返回时没有结果。函数可能返回的值的数量存在系统相关限制。此限制保证大于1000。冒号语法用于定义方法，即具有隐式额外参数self的函数。即：

```lua
     function t.a.b.c:f (params) body end -- 下面函数的语法糖，两个函数等价

     t.a.b.c.f = function (self, params) body end
```

### Visibility Rules

Lua是一种lexically scoped的语言。局部变量的范围从其声明后的第一个语句开始，一直持续到包含声明的最内层块的最后一个非void语句：

```lua
     x = 10                -- global variable
     do                    -- new block
       local x = x         -- new 'x', with value 10
       print(x)            --> 10
       x = x+1
       do                  -- another block
         local x = x+1     -- another 'x'
         print(x)          --> 12
       end
       print(x)            --> 11
     end
     print(x)              --> 10  (the global one)
```

请注意，在像x = x这样的声明中，声明的新x尚未在范围内，因此第二个x引用外部变量。由于词法范围规则，局部变量可以由其范围内定义的函数自由访问。内部函数使用的局部变量在内部函数内称为upvalue或外部局部变量。注意，每次执行本地语句都会定义新的局部变量：

```lua
     a = {}
     local x = 20
     for i=1,10 do
       local y = 0
       a[i] = function () y=y+1; return x+y end
     end
```

循环创建十个闭包（即匿名函数的十个实例）。这些闭包中的每一个都使用不同的y变量，而它们都共享相同的x。
