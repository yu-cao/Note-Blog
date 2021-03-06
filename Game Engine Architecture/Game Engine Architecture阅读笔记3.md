<h2>Game Engine Architecture笔记2</h2>

<h3>Chapter 6 资源与文件系统</h3>

游戏的本质就是多媒体与玩家的交互体验，所以游戏引擎必须要有载入与管理多种媒体的能力

1. 文件系统 

    * 在Unix下，使用正斜线符(/)的方式作为路径分隔符，而Windows下为反斜线(\\)
    * Windows下是支持以卷为单位分开目录层次，而Unix下一切资源都是挂接在主层次下的某一个子树上
    * 注意，每个操作系统都会禁止一些字符出现在文件与目录名称中，有些特殊情况需要(\\)这种转义或者双引号才能够有效
  
  搜索路径是含有一串路径的字符串，各路径之间由冒号或分号进行分割，找文件是就会按照这些路径去找例如Windows就会先找到当前目录下的可执行文件，再找PATH环境变量下提供的路径中的文件

2. 文件I/O：

   * C标准程序中，含缓冲功能的与不含缓冲功能的API表（具体使用可以查询各种官方文档）：
  
    |操作|有缓冲|无缓冲|
    |:-:|:-:|:-:|
    |开启文件|fopen()|open()|
    |关闭文件|fclose()|close()|
    |读取文件|fread()|read()|
    |写入文件|fwrite()|write()|
    |移动访问位置|fseek()|seek()|
    |返回当前位置|ftell()|tell()|
    |读入单行|fgets()|无|
    |写入单行|fputs()|无|
    |格式化读取|fscanf()|无|
    |格式化写入|fprintf()|无|
    |查询文件状态|fstat()|stat()|

	具体上的使用通常会把这些API包装成为自己的自定义I/O函数，这样可以保证API能够自主可控地在所有目标平台上保证同样的效果，而且API也可以根据引擎的需要进行一部分简化，提供可延伸功能

  * 同步文件I/O C标准库中的两种文件I/O库都是同步的，即程序发出I/O请求后必须要等到整个读/写完毕才能继续运行

```cpp
#include<iostream>
typedef int8_t U8;
bool syncReadFile(const char* filePath,
    U8* buffer, size_t bufferSize, size_t& rBytesRead)
{
    FILE* handle = fopen(filePath, "rb");
    if(handle)
    {
        //阻塞直至所有数据读取完毕
        size_t bytesRead = fread(buffer, 1, bufferSize, handle);
        int err = ferror(handle);//如果过程出错，取得错误码
        fclose(handle);
        if(0==err)
        {
            rBytesRead = bytesRead;
            return true;
        }
    }
    return false;
}
void main(int argc,const char* argv[])
{
    U8 testBuffer[512];
    size_t bytesRead = 0;

    if(syncReadFile("C:\\test.bin",testBuffer,sizeof(testBuffer),bytesRead))
    {
        printf("SUCCESS: read %u bytes\n", bytesRead);
        //可以使用缓冲内容了
    }
}
```
    
  * 异步文件I/O 串流(streaming)指在背景载入数据，而主程序同时继续运行，为了支持串流，就要使用异步(asynchronous)文件I/O库，如果目标平台不提供这样的库，也可以自己开发一个（前提是平台能提供线程或类似服务）

```cpp
#include<iostream>
typedef int8_t U8;

AsyncRequestHandle g_hRequest;//异步I/O请求的句柄
U8 g_asyncBuffer[512];

static void asyncReadComplete(AsyncRequestHandle g_hRequest);

void main(int argc,const char* argv[])
{
    //注意：这里调用asyncOpen()可能本身就是异步的，在这里暂时忽略
    //假设该函数是阻塞的
    AsyncRequestHandle hFile = asyncOpen("C:\\test.bin");

    if(hFile)
    {
        //此函数做出读取请求，然后立即返回（非阻塞）
        g_hRequest = asyncReadFile(
            hFile,                  //文件句柄
            g_asyncBuffer,          //输入缓冲
            sizeof(g_asyncBuffer),  //缓冲大小
            asyncReadComplete);     //回调函数
    }

    for(;;)
    {
        OutputDebugString("zzz...\n");
        Sleep(50);
    }
}

//当数据都读入后，才会调用此函数
void asyncReadComplete(AsyncRequestHandle hRequest)
{
    if(hRequest == g_hRequest && asyncWasSuccessful(hRequest))
    {
        //现在数据已经读入g_asyncBuffer[]。查询实际读入的字节数量
        size_t bytes = asyncGetBytesReadOrWritten(hRequest);

        char msg[256];
        sprintf(msg, "async success, read %u bytes\n", bytes);
        OutputDebugString(msg);
    }
}
```
    
  需要谨记文件I/O是一个实时系统，要有时限与优先级，异步I/O系统需要有暂停低优先级的请求的能力，才能够让高优先级的请求在合理的时限中完成

  异步I/O工作原理：主线程调用异步函数，把请求放在一个队列中并立即传回；同时I/O线程会从队列中拉取请求并以阻塞I/O函数如read()或者fread()等处理这些请求；完成后就会调用主线程之前的提供的回调函数，告知这个操作已完成

  因此，任何的同步操作都可以通过代码置于另一个线程上变成异步操作

3. 资源管理器

    每个游戏都是由各种各样的资源构成，都需要一个资源管理器进行调度，每个资源管理器都由两个组件构成：
    
    * 离线资源管理：用来创建资产并且把他们转化成为引擎可用的形式<br>
    因为游戏中有大量而且内容庞大的美术、音乐等资源，对于大部分资产来说，这些东西都是由先进的数字内容创作（DCC）工具完成的，如Maya，ZBrush，Photoshop，Houdini，但是这些格式一般不适合游戏引擎直接使用，需要经过一些资产调节管道将资产转化为可支持的格式。我们不可能每次都用人工的方式进行处理每个资产，而需要一个半自动化的资源管道，而管道所需要的数据储存在某种数据库中（例如在虚幻中，资源数据库为UnrealEd，因为这个组件被设计为引擎的一部分，所以可以使得UnrealEd能够在创建资产后立即看到资产的模样，得知资产是否有效与正确配置，但是缺点是所以资源都存在少量但巨大的二进制包文件中，很难版本控制）
    
    + 导出器 把DCC的原生格式变成我们能处理的格式，方法是为DCC工具撰写自定义插件，变成某种中间格式供后续管道继续使用
    + 资源编译器 为由DCC导出的数据以不同方式做一些处理（类似对数据的重新编译）才能让引擎使用
    + 资源链接器 把多个资源先结合成一个有用的包再载入引擎，比如一个三维模型，需要绑定多个网格文件，材质文件，骨骼文件，动画文件等结合生成再导入资源依赖，因为资源之间存在相互依存关系构成一张有向图，在处理时我们修改了一个资源本身或者其数据格式之后，其他资源也可能会受到影响，需要重新生成（还是要以正确的顺序进行生成）

* 运行时资源管理：确保资源在使用之前已经载入内存，在不需要的时候把它们从内存中取下

所有的资源都必须要有某个全局唯一标识符（Global Unique Identifier，GUID）常见的GUID选项就是资源的文件系统路径（储存为字符串或者hash值）而虚幻是通过包地址+包内地址串接组合而成。
    
以下是设计的要求与实现思路：

+ 确保在任何时刻，同一个资源在内存中只有一个副本：使用**资源注册表**的方式记录已载入的资源。实现方法：使用字典方式储存，键是资源唯一标识符，值是指向内存中资源的指针；当注册表中找不到时，就进行资源载入，这时可以使用loading界面给玩家或使用较难实现的异步方式加载（玩家在玩A时，后台慢慢加载后面一个场景B）
+ 管理一个资源的生命周期：玩家一直能听到或者看到的资源，都应该归于LSR(load-and-stay-resident)资源，生命周期无限，其余的资源的归还都是比较困扰的问题，需要仔细考量<br>
解决方案：对资源进行使用计数（斜体代表该资源不在内存中，正体代表该资源已经载入内存，有括号表示该资源正要载入或卸下）此外，在具体分配方式考量中可以考虑使用之前介绍的几种资源分配方式进行分配（基于堆、堆栈、池等）

|事件|A|B|C|D|E|
|:-:|:-:|:-:|:-:|:-:|:-:|
|起始状态|*0*|*0*|*0*|*0*|*0*|
|关卡1的引用计数加1|*1*|*1*|*1*|*0*|*0*|
|载入关卡1|(1)|(1)|(1)|*0*|*0*|
|玩关卡1|1|1|1|*0*|*0*|
|关卡2的引用计数加1|1|2|2|*1*|*1*|
|关卡1的引用计数减1|0|1|1|*1*|*1*|
|卸下关卡1，载入关卡2|*(0)*|1|1|(1)|(1)|
|玩关卡2|*0*|1|1|1|1|
    
+ 保持引用完整性
    + 处理交叉引用等问题：指针只是内存地址，值离开运行的程序之后就没有意义，所以储存数据至文件时，不能使用指针表示对象之间的依赖性，而应该使用全局唯一标识符(GUID)进行交叉引用（把交叉引用储存为字符串或者散列码，内涵被引用的对象的唯一标识符）
    + 指针修正表的方式：储存时把**指针**变成**文件偏移值**（要注意32位与64位的区别，开发编译平台与目标平台的兼容）在载入文件到内存之后把偏移值转回指针；通过一个简单列表的方式储存每个类的中指针，当进入内存之后就能够进行修正
    + 外部(非单个资源文件内的对象)引用处理：除了要指明偏移值和GUID外，还需要加上资源对象所属文件的路径
+ 管理载入后的合理内存用量
    载入后的初始化：希望每个资源都能够经过离线工具完全处理，载入后内存能够立即使用。在C++中可以实现为两个虚函数的方式Init()和Destroy()进行（因为构造函数不可虚，所以无法对基类进行操作，因此选择自己写一个虚函数进行）
+ 通常保证单一统一接口处理多种资源类型
+ 支持串流操作（异步资源载入）