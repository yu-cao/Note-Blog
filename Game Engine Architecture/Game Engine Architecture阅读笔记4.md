<h2>Game Engine Architecture笔记2</h2>


<h3>Chapter 7 游戏循环与实时模拟</h3>

渲染循环：当摄像机在三维场景中移动的时候，屏幕或者视窗上的一切内容都会不断改变，所以将使用渲染循环：给观众快速连续显示一连串的静止影像

```cpp
while(!quit)
{
    updateCamera();

    //更新场景中的所有动态元素的位置、方向和一系列视觉状态
    updateSceneElements();

    //把静止的场景渲染至屏幕外的帧缓冲中
    renderScene();

    //交换背景缓冲与前景缓冲，令最近渲染的影像显示于屏幕之上
    swapBuffers();
}
```

游戏循环：

* 视窗消息泵：先处理来自Windows的消息，无消息时才去执行引擎的任务，但是这里的前提条件是设定了任务的优先次序，当玩家在改变游戏视窗等操作时会导致游戏直接卡住不动

```cpp
while(true)
{
    //处理所有待处理的Windows消息

    MSG msg;

    while(PeekMessage(&msg,NULL,0,0) > 0)
    {
        TranslateMessage(&msg);
        DispatchMessage(&msg);
    }

    //无Windows消息需要处理时才去真正执行游戏循环
    RunOneIterationOfGameLoop();
}
```

* 回调驱动框架：很多游戏引擎子系统和第三方游戏中间套件都是以程序库的形式构成的：程序员正确使用程序库提供的各个函数和类；也有一些游戏引擎与游戏中间件是用框架构成的，程序员需要提供框架中空缺的自定义实现或者复写框架中的预设行为，但是这样相比前者对于应用软件的控制流程只有少量控制

* 基于事件的更新：多数引擎都会存在一个事件系统，让每个引擎子系统关注某类型的事件，当那些事件发生时就一一回应。有些游戏引擎会使用事件系统来对所有或部分子系统进行周期性更新，也就是说：事件可以先置于队列，稍后才取出处理。当游戏进行更新时，只要简单加入事件，在事件处理器里面代码就可以以任何所需的周期进行更新

抽象时间线：

* 真实时间：直接使用CPU的高分辨率计时寄存器来度量时间。原点定义为计算机上次启动时间。CPU经历的周期 * 频率 => 秒数
* 游戏时间：独立于真实时间的一条时间线，如果希望暂停游戏，就可以简单暂停对于游戏时间的更新，如果想慢动作，就把游戏时间更新速度慢于真实时间。**注意：游戏暂停时游戏循环是继续进行的，只不过是游戏时钟停止了！**
* 局部与全局时间线：每个动画片段与音频片段都有一个局部时间线，原点定义为片段的开始，局部时间线就按照原来的制作或录制的时间量度播放，我们可以加速/减速/反向播放动画或音频，因为这些效果都可以视为局部与全局时间线之间的映射。

测量与处理时间：
因为&Delta;x = x1 + v&Delta;t 所以玩家对于物体运动的感知取决于帧时间&Delta;t，所以&Delta;是游戏编程的核心问题之一

目标：开发与CPU速度脱钩的游戏：

  * 方法一：读取CPU的高分辨率计时器两次（一次在帧开始之前，一次在帧开始之后，取差值就能精确度量上一帧的&Delta;t，但是问题是使用第k帧去度量第k+1帧的时间不一定正确，而且可以会导致恶性循环（第k帧30ms，k+1帧40ms，然后度量k+2帧会估计为40ms，然后...）
  * 方法二：调控帧率。控制每帧都耗时相同（30FPS就33.3ms），如果本帧的耗时超过了这个限度，就白等到下一个目标时间，而本帧的耗时小于这个，就让主线程休眠。这样只能在游戏帧率比较稳定而且接近目标帧率时才比较有效。优点：稳定帧率对于游戏的数值积分等运算比可变帧率更有优势，如果更新的速率不配合屏幕刷新率会导致画面撕裂（电子枪等在扫描过程中执行了交换了背景缓冲区和前景缓冲区，让画面上半部分显示老的影像，下半部分显示新的影像），而这个方式可以避免画面撕裂的发生

时间单位与时钟变量:

* 64位整数时钟：同时支持高精度(3GHz CPU每周期位0.33ns)与很大的数值范围(3GHz CPU需要195年才会出现循环，远超使用寿命)是最好的方式
* 32位整数时钟：量取高精度与较短时间的策略（一般用来量取&Delta;t）
* 32位浮点时钟：把较小的持续时间以秒为单位储存为浮点数。必须避免使用浮点时钟变量储存很长的持续时间，最多能度量几分钟，而且需要定期重置为0，减少误差

应付断点：Debug会暂停游戏循环，但是时间依然前进，这样会导致巨大的增量时间，而如果这些时间被错误地传入了子系统中很可能导致整个游戏等的崩溃；解决策略：当量度到某帧的持续时间超过预设的时间，就假设游戏刚从断点中回复，人工地将增量时间设定为1/30s等而避免出现一个巨大的帧时间尖峰

简单的时钟类：通常含有一个变量，用来记录自时钟创建以来经过的绝对时间，一般选择以机器周期为单位的64位无符号整数最简单与方便

```cpp
typedef int64_t U64;
typedef float F32;

class Clock
{
    U64 m_timeCycles;
    F32 m_timeScale;
    bool m_isPaused;
    static F32 s_cyclesPerSecond;

    static inline U64 secondsToCycles(F32 timeSeconds)
    {
        return (U64)(timeSeconds * s_cyclesPerSecond);
    }

    static inline F32 cyclesToSeconds(U64 timeCycles)
    {
        return (F32)(timeCycles / s_cyclesPerSecond);
    }

public:
    static void init()
    {
        s_cyclesPerSecond = (F32)readHiResTimerFrequency();
    }

    explicit Clock(F32 startTimeSeconds = 0.0f) :
        m_timeCycles(secondsToCycles(startTimeSeconds)),
        m_timeScale(1.0f),
        m_isPaused(false)
    {}

    U64 getTimeCycles() const
    {
        return m_timeCycles;
    }

    F32 calcDeltaSecond(const Clock& other)
    {
        U64 dt = m_timeCycles - other.m_timeCycles;
        return cyclesToSeconds(dt);
    }

    void update(F32 dtRealSeconds)
    {
        if(!m_isPaused)
        {
            U64 dtScaledCycles = secondsToCycles(dtRealSeconds * m_timeCycles);
            m_timeCycles += dtScaledCycles;
        }
    }

    void setPaused(bool isPaused)
    {
        m_isPaused = isPaused;
    }

    bool isPaused() const
    {
        return m_isPaused;
    }

    void setTimeScale(F32 scale)
    {
        m_timeCycles = scale;
    }

    F32 getTimeScale() const
    {
        return m_timeCycles;
    }

    void singleStep()
    {
        if(m_isPaused)
        {
            U64 dtScaledCycles = secondsToCycles((1.0f / 30.0f) * m_timeCycles);
            m_timeCycles += dtScaledCycles;
        }
    }
};
```

多处理器的游戏循环：

* 分叉与汇合：把一个单位的工作分割成更小的子单位，再把这些工作量分配到多个核或者硬件线程之中（分叉），最后待所有工作完成后再合并结果（汇合）
* 每个子系统运行与独立线程：主控线程负责控制与同步这些子系统的次级子系统，并应付游戏的大部分高级逻辑（游戏主循环），子系统去重复执行那些比较有隔离性的功能，比如渲染引擎，物理模拟，动画管道，音频引擎等等

异步程序设计（为了多处理器的硬件而写的代码）：当操作请求发出之后，通常不会马上得到结果，很可能是在某帧的开始启动请求，下一帧才能够提取结果。调用的函数只会建立一个作业并把它加到队列之中，然后主线程看情况调用空闲的CPU或者核去处理这个作业

网络多人游戏循环：

* 主从式模式：大部分游戏逻辑运行中服务器上，客户端只是做渲染工作（单机游戏可以看成一个客户端且服务器与客户端运行在同一机器上）两者的代码可以以不同的频率循环
* 点对点模式：每一部机器既有服务器的特性，也有客户端的特性，机器对拥有管辖权的对象就像服务器，对无管辖权的对象就像客户端

<h3>Chapter 8 人体学接口设备</h3>

技术：

* 轮询：定期轮询来读取输入，查询设备状态（读取硬件寄存器/读取经内存映射的I/O端口）
* 中断：只有当状态出现变化之后才会把数据传送至游戏引擎（硬件中断方式通信）

输入类型

* 数字式按钮：按下/释放两种状态
* 模拟式轴与按钮：获得一个范围的值的模拟信号，然后把它数字化传入引擎（死区：因为存在电压噪声，所以需要引入一个小死区来控制输入稳定，死区需要容纳最大的噪声，也要足够小避免影响玩家操作体验）

输入事件检测：游戏需要检测事件——状态改变而不是每帧的当前状态
技巧：把上一帧与本帧的按钮状态字进行位异或，某个位为1代表两帧中出现了状态的改变，而测定这个事件是按下还是释放按钮，可以再通过后续审视每个按钮的状态