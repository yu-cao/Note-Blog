<h2>Game Engine Architecture笔记2</h2>

<h3>Chapter 4 游戏所需的三维数学</h3>

因为这里的数学公式、矩阵之类的很多，使用Markdown有诸多不便，就省略一些，主要记录SSE和随机数生成

* SSE：单指令多数据拓展流(streaming SIMD extensions,SSE)，在引擎中多将4个32位的float值打包进单个128位寄存器中，单个指令就可以对4对浮点数进行并行运算

SSE在C/C++中提供了__m128这个内建的数据类型，它的运算可以通过汇编实现，也可以通过C/C++的内联汇编实现(inline assembly)。但是现在为了更好的编译器优化与跨平台，通常**建议使用编译器提供的内部函数**进行实现

**始终记住，将普通的float的加法和SIMD运算结合的混合代码无比愚蠢，必须规避或优化**
    
```cpp
#include<xmmintrin.h>//SSE内部函数与__m128这个数据类型必须要包括这个.h文件
#include <cstdio>

//内联汇编
__m128 addWithAssembly(__m128 a, __m128 b)
{
    __m128 r;
    __asm
    {
        movaps xmm0, xmmword ptr [a]
        movaps xmm1, xmmword ptr [b]
        addps xmm0, xmm1
        movaps xmmword ptr [r], xmm0
    }
    return r;
}

//内部函数
__m128 addWithIntrinsics(__m128 a,__m128 b)
{
    __m128 r = _mm_add_ps(a, b);
    return r;
}

int main()
{
    __declspec(align(16)) float A[] = { 2.0f,-1.0f,3.0f,4.0f };
    __declspec(align(16)) float B[] = { -1.0f,3.0f,4.0f,2.0f };
    __declspec(align(16)) float C[] = { 0.0f,0.0f,0.0f,0.0f };
    __declspec(align(16)) float D[] = { 0.0f,0.0f,0.0f,0.0f };

    __m128 a = _mm_load_ps(&A[0]);
    __m128 b = _mm_load_ps(&B[0]);

    __m128 c = addWithAssembly(a, b);
    __m128 d = addWithIntrinsics(a, b);

    _mm_store_ps(&C[0], c);
    _mm_store_ps(&D[0], d);

    printf("%g %g %g %g\n", C[0], C[1], C[2], C[3]);
    printf("------------------\n");
    printf("%g %g %g %g\n", D[0], D[1], D[2], D[3]);

    return 0;
}
```
    
也可以通过这个实现快速的矢量与矩阵乘法

**r** = **vM**

其中，v是1\*4矢量，M是4\*4矩阵
    
```cpp
//一定是通过Row进行计算，最后才能并行化；如果使用Col进行计算，最后会有4个类似于vxM11 + vyM21 + vzM31 + vwM41这样的普通浮点add运算，导致性能大幅度损失
#define SHUFFLE_PARAM(x,y,z,w) \
    ((x) | ((y) << 2) | ((z) << 4) | ((w) << 6))
#define _mm_replicate_x_ps(v) \
    _mm_shuffle_ps((v),(v),SHUFFLE_PARAM(0,0,0,0))

#define _mm_replicate_y_ps(v) \
    _mm_shuffle_ps((v),(v),SHUFFLE_PARAM(1,1,1,1))

#define _mm_replicate_z_ps(v) \
    _mm_shuffle_ps((v),(v),SHUFFLE_PARAM(2,2,2,2))

#define _mm_replicate_w_ps(v) \
    _mm_shuffle_ps((v),(v),SHUFFLE_PARAM(3,3,3,3))


__m128 mulVectorMatrixAttemp1(__m128 v,
    __m128 Mrow1,
    __m128 Mrow2,
    __m128 Mrow3,
    __m128 Mrow4)
{
    __m128 xMrow1 = _mm_mul_ps(_mm_replicate_x_ps(v), Mrow1);
    __m128 yMrow2 = _mm_mul_ps(_mm_replicate_y_ps(v), Mrow2);
    __m128 zMrow3 = _mm_mul_ps(_mm_replicate_z_ps(v), Mrow3);
    __m128 wMrow4 = _mm_mul_ps(_mm_replicate_w_ps(v), Mrow4);

    __m128 result = _mm_add_ps(xMrow1, yMrow2);
    result = _mm_add_ps(result, zMrow3);
    result = _mm_add_ps(result, wMrow4);

    return result;
}

//对上面的函数，我们也使用乘并加的指令进行简化，通常为madd：把前面两个参数相乘，与最后一个参数加和，但是SSE不支持madd，所以我们使用宏代替
#define _mm_madd_ps(a,b,c) \
    _mm_add_ps(_mm_mul_ps((a),(b)), (c))

__m128 mulVectorMatrixFinal(__m128 v,
    __m128 Mrow1,
    __m128 Mrow2,
    __m128 Mrow3,
    __m128 Mrow4)
{
    __m128 result;
    result = _mm_mul_ps(_mm_replicate_x_ps(v), Mrow1);
    result = _mm_madd_ps(_mm_replicate_y_ps(v), Mrow2, result);
    result = _mm_madd_ps(_mm_replicate_z_ps(v), Mrow3, result);
    result = _mm_madd_ps(_mm_replicate_w_ps(v), Mrow4, result);
    return result;
}
```

矩阵之间相乘也可以依靠上面的运算实现，具体针对Visual Studio上的提供的SSE内部函数，可参阅MSDN
    
* 产生随机数：因为确定性算法的原因，这些算法产生的都是伪随机数，仅仅只是非常复杂而已
    
    + 线性同余产生器(LCG)：不能产生特别高质量的伪随机序列，只要种子相同，序列就相同，很多时候不是很符合要求
    + 梅森旋转(MT)算法：通过了Diehard测试，其中一个采用SIMD矢量指令进行优化的SFMT很高效而有趣
    + Xorshift产生器/所有伪随机产生器之母(mother of all pseudo-random number generator)

<h3>Chapter 5 游戏支持系统</h3>

1. 游戏子系统

    游戏需要子系统进行一系列例行但是关键任务的处理，对于子系统的构建与终止，最简单也是最实用的方法是明确地为每个单例管理器类定义启动与终止函数，用来取代构造、析构函数的作用

```cpp
class RenderManager
{
    public:
    RenderManager(){/*不做事情*/}
    ~RenderManager(){/*不做事情*/}

    void startUp()
    {
        //启动管理器
    }
    void shutDown()
    {
        //终止管理器
    }
};
//...类似的一系列类的定义

RenderManager gRenderManager;
//...一系列实例化

int main(int argc, const char* argv)
{
    //正确方式启动引擎
    gMemoryManager.startUp();
    gFileSystemManager.startUp();
    //...

    //运行游戏
    gSimulationManager.run();

    //以反向方式关闭引擎
    //...
    gFileSystemManager.shutDown();
    gMemoryManager.shutDown();

    return 0;
}
```

这种方法既简单又实现方便，而且理解容易，易于后续的调试与维护

2. 内存管理

    在可行的情况下，代码中尽量避免使用**动态内存分配**，就像上面的代码，绝大多数的单例都是静态分配的对象

    此外，在现代CPU的局部性原理的作用下，数据尽量分配在**细小**（希望一次都能载入CPU或者Cache中而不需要多次处理）而又**连续**（一次性读入CPU或者Cache，减少找的时间）的内存块中，CPU的操作会高效得多
    
 * 堆分配

    通过malloc()/free()/new等方式获得动态内存——非常慢（原因：1. 这是一个通用函数，需要处理任何大小的请求，需要大量的管理开销 2. 在多数操作系统上，malloc()/free()会导致系统模式从user mode -> kernel mode -> user mode，这些转换需要大量的时间开销）<br>
    经验规则：**维持最低限度的堆分配，并且永不在紧凑循环中使用堆分配**
    
* 改进之道

    在一开始就通过malloc()或者new进行动态内存申请，之后的维护由程序员自己维护，不再使用malloc()或者new
    
    * 基于堆栈的分配器

    申请所得的连续内存安排一个指针指向堆栈的顶端，指针以下的是已经分配的，指针以上的是未分配的

    **注意：使用堆栈分配器时必须要以分配时相反的顺序释放内存** 可以通过在上面做标记(Marker)的方式，释放时释放从栈顶到Marker之间的所有内存
    
    * 双端堆栈分配器(原理类似,略)
    * 池分配器 通常为了分配实践中遇到的**大量同等尺寸的小块内存**这种情况

    简单来说，池内的每个元素会串成一个单链表，当需要的时候就取出，不用的时候放回去，每次都是 *O(1)* 的复杂度

    为了充分节约空间(节约掉链表中\*next这个指针，因为每个元素都要有，而每个指针一般都要4字节，太浪费)：当这些自由元素不用时，自由元素里面就储存下一个节点的地址（用内存本身来存储自由列表中的"next"指针）但是，如果元素尺寸\<指针时，那么就可以使用池元素的索引代替指针实现链表。
    
    * 含有对齐功能的分配器
    * 单帧和双缓冲内存分配器 在游戏循环中常常分配一些临时用数据，当循环迭代结束时丢弃
        + 单帧分配器
        先预留一块内存，每帧开始时就把堆栈顶端指针重置到内存块的底端，然后在该帧过程中不断向上生长<br>
        缺点：程序员要有清醒的认识与自制，**决不可把指向单帧内存块的指针去跨帧使用！**
        + 双缓冲分配器
        相比较上面的单帧分配器可以多保留一帧，允许在第(i+1)帧时调用第i帧的内存<br>
        实现策略：建立两个相同尺寸的单帧堆栈分配器，并且在每帧交替使用
        
* 内存碎片

因为开销等原因，很多设备都不支持可以规避这个问题的“虚拟内存”策略，一旦出现了很多内存碎片，可能会遇到尽管内存有空但是无法取用的尴尬境地

 + 堆栈/池分配器 不会产生内存碎片
 + 其他情况 要分配和释放的内存是不同大小的对象，并且是以随机次序进行的

    需要对堆进行定期**碎片整理**：把内存空洞（未分配内存）移动到高地址（冒泡法般地进行比较，复制，交换，形成“上浮”效果）

    但是，这样的操作影响了**已分配**的内存块，当程序中有指针指向这些内存块时，这些移动不可避免地会使指针失效。而指针重定位在C/C++中是很难实现的，通常使用智能指针或句柄实现（主要是使用句柄的方式以避免重定位指针）

    句柄：实现往往是索引，这些索引指向句柄表中的元素，每个元素储存指针。句柄表本身不能被重定位，但是当要移动某个已分配的内存时，只需要扫描句柄表，并自动修改相应的指针即可。
 + 成本分摊 当进行碎片整理时操作因为涉及复制，所以是很慢的，我们可以把碎片整理成本分摊到多个帧中，每帧进行N次内存移动，N一般很小如8或16，这样在多个帧中完成这样的操作能够不会对游戏帧率产生影响

* 缓存一致性 由于局部性原理，我们应该尽可能避免缓存命中失败的情况，以下是经验法则：
    + 高效能的代码体积越短越好，体积按机器码指令数目为单位
    + 在关键性能的代码段，避免调用函数
    + 要调用某函数，尽量把该函数置于最靠近调用函数的地方，不要把该函数置于另一个翻译单元
    + **审慎使用内联函数**：内联小型函数的确能提升性能，但是过多的内联会增大代码体积，而指令也是通过缓存载入CPU的，这样可能会导致性能关键的代码不再能完全装入缓存之中，如果有一个紧凑循环，那么每个循环都会产生两次miss产生大量时间浪费，可能需要重构这部分代码

* 容器
    * 链表 
+ 外露式表（extrusive list）：一个元素能够同时置于多个链表之中，但是必须要动态分配节点，通常可以用池分配器进行分配（32位机器上就是12个字节）

```cpp
template<typename ELEMENT>
struct Link
{
    Link<ELEMENT>* m_pPrev;
    Link<ELEMENT>* m_pNext;
    ELEMENT*       m_pElem;//指向元素具体位置
}
```

+ 侵入式表（intrusive list）：节点的数据结构已经被嵌入到目标元素之中，避免了动态内存分配，但是丧失了每个元素能够同时置于多个链表之中的弹性，比如像存储一些实例，而实例的类是第三方库的，就不能使用侵入式表

```cpp
template<typename ELEMENT>
struct Link
{
    Link<ELEMENT>* m_pPrev;
    Link<ELEMENT>* m_pNext;
    //由于继承的关系，无需ELEMENT* 指针
};

class SomeElement : public Link<SomeElement>
{
    //其他成员
};
```