
## PlatformMoving
### 1.需求分析
实现对平移台三个运动维度（前后、左右、上下，以相机为参照系）的精确控制。控制精度在 0.01 mm 以内。
提供自动限位功能，防止平台出现不合理移动。
提供每次运动到达指定位置或者是运动出错的信息回馈。
### 2.概要设计
#### 2.1.硬件平台选择
使用专业的平移台（选用的是[联英精机](http://www.lyseiki.com/)）出品的水平滑台，提供两个维度（前后、左右）的运动；

电机控制器选取 Tb6560 步进电机驱动板，选取原因是此驱动板属于工业级驱动板，在诸多领域（比如 3D 打印机，雕刻机等）有广泛的使用，且市场占有率较高，有质量保证；

主控硬件选择 raspberry，原因在 Gimbal 处已有部分说明，另外，还是因为 raspberry Model 2B 拥有 40 个 GPIO 管脚，可以满足我们同时连接 3 个电机驱动板，并且兼顾电机驱动板的共阳极连接方式、接收限位的霍尔元件信号等要求，且树莓派 GPIO 的工作频率亦满足我们的控制需求。

#### 2.2.整体实现思路
程序将移动距离转化为发向电机控制板的脉冲数；
raspberry 通过 GPIO 口向电机控制板发送特定个数的脉冲，控制板控制步进电机移动。

#### 2.3.关键技术难点与可能存在风险
电机控制板与 raspberry GPIO 之间、电机控制板与水平滑台的步进电机之间的连线需要自己完成，其中不乏一些焊接工作（串口连接线、控制板接电线等），连线是有可能出现一些接触不良或者是连接出错（大概需要 30-40 根/对 线）的现象的，需要格外小心。

步进电机以及电机驱动板需要寻找许多参数的合理值，比如力矩较大时速度不可以太高，错误地参数可能会导致失步、空转等。

#### 2.4.术语定义
关于步进电机：
静态指标：相数、拍数、步距角、定位转矩、静转矩；
动态指标：步距角精度、失步、失调角、最大空载启动频率、最大空载运行频率、共振点等；
这些指标的详细介绍见附录，我们在编程控制时需要注意的有步距角、启动/运行频率等。

关于 Tb6560 步进电机驱动板：
工作电流、静止时半流、细分、衰减等
我们需要注意的是选择合适的配置与接线方式（共阳极或是共阴极），在编程时需要注意选择的细分。

步进电机矩频曲线（Pulse-torque characteristics），电机不同的脉冲频率（速度）对应的力矩有所不同，我们需要保证达到速度较快的同时要有足够的力矩。

#### 2.5.模块概览

<< 来张图

即：

#### 2.6.模块划分
1.stepMotor:步进电机类，将步进电机与电机驱动器当作一个整体，能够进行相应参数配置，主要是可以在给定移动距离后转化为脉冲数然后移动特定脉冲数
2.platformController:包含三个 stepMotor，调用 stepMotor 的移动方法，向外提供整个平移台三个维度的移动、平台归位（将水平两个方向移动到中间位置，保证平移台有较大行程）方法接口，并有错误处理（到达限位）信息返回。
3.ServerUtility:定义 platform 移动的命令格式；
4.AuSocketServer:Socket 库；
5.ServerController:解析/打包 platform 命令数据，调用 paltformController 的方法实现命令对应操作；
7.RaspberryServerMain:将程序部署到 raspberry 上，打开 Socket 通信与上层软件进行交互，接受到数据后，调用 ServerController 进行解析与运行，将返回值通过 Socket 返回到上层软件。

#### 2.7.测试计划
platformController/ServerController 的所有方法，可以直接写 cpp 文件（文件（GimbalSBGC_Test.cpp）进行测试；
实现 communicateTest 工程（为简单且提供 GUI 使用 C# 开发）来进行经由 Socket 发送运动指令的测试。

### 3.详细设计
#### 3.1.stepMotor 类
控制步进电机的类有许多参数，其中不乏需要手动配置的参数，参数列表如下（摘自源代码 stepMotor.h）：
``` CPP
/*参数定义*/    
public:
    char motorName;
    /*pin connection*/
    int dirPin, //控制方向管脚
        clkPin, //输出脉冲管脚
        outRangePinNegative, //限位信号1
        outRangePinPositive,    //限位信号2


    /*configurable parameter*/
        subdivision,        //细分数
        reductionRatio = 1; //减速比
    double  screwSpacingPerRound;   //每运动一周对应的丝杠间距(ms)
    bool outRangeSwitch = true;
    
    /*static speed parameter*/
    double  stepAngle = 1.8,    //步距角
        maxRps = 1.0 / 1,   //每秒最多运动圈数
        minRps = 1.0 / 1,   //每秒最少运动圈数
        maxMovingDistance = 0.0;    //最大行程

    /*calculate speed parameter*/
    double delayMSTable[speedupPulses]; //加速过程对应 delay
    double stepPerRound;    //每圈的脉冲数
```

dirPin 与 clkPin 是 raspberry GPIO 向电机驱动板发送信号的管脚，前者控制方向，后者发送不同频率的脉冲控制电机移动速度；
outRangePin 是内置于水平滑台成品中每个轴两端的两个霍尔元件，当其信号指示到达限位时水平台应及时停止移动，防止造成损坏；
screwSpacingPerRound:水平滑台内部是一个步进电机连接一高精度细螺距的金属丝杠，当电机运动时丝杠移动从而导致平台（准确地说是平台上的可移动块）移动，我们在转化移动距离时需要使用此间距；
maxMovingDistance:最大移动行程，在平台归位时需要用到，即先移动到一个边界，再反向移动最大行程的一半；
stepAngle:步距角，电机的固定参数，但当换用不同电机或者是电机的不同连线方式时有可能会有变化；
reductionRatio:减速比，电机固定参数，当需要更大的力矩时（我们使用的控制整台设备上下移动的电机），必须选择减速比较大的电机，以较小的速度换取足够的力矩；
subdivision :细分数，电机控制板的参数，可以提供更为精确的移动距离控制，

maxRps、minRps 可调的运动速度设置（每秒运动圈数），电机在上电时如果就要求较大速度的话很有可能出现失步现象，这会导致我们的控制失效且会对电机造成性能上的影响。所以需要在移动时提供加减速过程，即在移动过程时，先以一个较小速度开始移动，然后逐渐加大速度，快要移动结束时减少移动速度。
delayMSTable[speedupPulses]:上面说到的的加减速过程对应的脉冲间延时（我们通过控制脉冲间的延时来控制脉冲的频率），其中 speedupPulses 是一个加速过程的脉冲数，当适当增大该值时可以提供更平稳的加速过程。

加减速的示意图如下：

<< 来一个图

当然，程序中是以离散值的形式出现，当 speedupPulses 足够大时会非常平稳。

此后在给定总移动脉冲数时可以根据当前移动脉冲数来确定当前的 delay 值，策略：
1.总运行距离大于一个加减速过程（ 2 个加速）
    1.1.当前移动脉冲数小于一个加速脉冲数，继续加速
    1.2.否则，剩余脉冲数小于一个加速脉冲数，即应该减速
    1.3.否则（剩余脉冲数大于一个加速过程脉冲数）保持最高速
2.总移动距离小于一个加减速过程
    2.1.加速阶段（小于总移动距离的一半）
    2.2.减速阶段（大于总移动距离的一半）
具体翻译到对应代码已经很明晰了，就不再赘述。

外部接口定义：
setOrUpdateDelayMSTable:更新 delayMSTable，可以在运动过程中更新延时表，进而更新移动速度；
motorMove(double distance):电机移动特定毫米数，正负表示方向；
motorToHome：到达行程中间位置。

内部接口定义：
_dis2Pulses(double distance):将毫米数转化为电机的脉冲数
_motorMove(int clk_number, int maxOutRangePulse):提供了运动过程中对到达限位的处理。


#### 3.2.platformController 类
外部接口定义：
PiInit:初始化树莓派，将树莓派特定的 GPIO 口设置为 INPUT 或 OUTPUT 模式，且将接收霍尔元件信号的接口设置为上拉电阻模式；
plfMoveXandYandZ(int isInit, double x, double y, double z),仅向外提供这一个控制移动的接口，根据 isInit 取值来判断是正常移动特定距离还是平移台归位操作。

内部接口定义：
pltMove(int XorYorZ, double distance):调用 motorMove 方法单独移动一个轴；
plfInit：调用 motorToHome 使平移台归位。

#### 3.3.ServerUtility 类
定义了平移台移动指令的相关数据以及转化规则，这里用到的策略比较明晰，只需与上层调用软件规定好指令格式标准并严格执行即可。
需要特别注意的是，当返回错误信息（到达限位）时要注意返回数值，我们一开始时到达限位返回了 -1，但在与客户端经由 Socket 通信后无法正确解析，最终将错误返回值置为 2 而一切正常，当然，这是一种遇到了问题后进行规避，但可以采用类似方法。

#### 3.4.AuSocketServer 与 ServerController 类
在本地建立 Socket 服务器与客户端进行 Socket 通信，将指令信息解析后调用 platformController 相应方法进行移动并将移动返回信息返回到客户端。



### 4.编码实现及遇见的问题及解决方案
确定电机移动最大速度（或者合适速度），原本是准备完全根据电机的矩频曲线来确定，不过最终发现供应商提供的矩频曲线是空载时的，不适用于负重之后的电机。所以，手动尝试最大与最小速度（因为只需调整这两个参数，而我们的应用是尽可能的平稳且尽量快速，并不需要找到精确的速度最大值），最终找到了比较合适的值。水平两个方向（前后、左右）最小速度取 1.0 rps，最大取 3.0 rps；垂直方向（上下）最小速度取 1.0/30 rps，最大速度取 1.0/12 rps，当然，对应于不同电机和不同应用场景可以选择不同的最大最小速度，但程序是通用的。

在关闭程序很短的时间内重启程序在试图绑定 socket 时会出现 ": Address already in use" 错误，设置为允许 socket 地址重用即可解决问题。

### 5.测试
与 Gimbal 程序相似，我们的测试较为简单，对每个上述提到的方法进行单独测试，然后通过单独的 Socket 客户端与我们的程序进行 Socket 通信下的测试。在此赘述每次方法的测试有些过于浪费篇幅，详情可以参见 platformMoving_Test.cpp 文件以及 C# 的 communicationTest 工程。

#### 6.一些建议
对 raspberry GPIO 口进行编程时建议使用[webiopi 在 web 上查看 GPIO 口](http://webiopi.trouch.com/)，适用于 raspberry [Model 2B 的 补丁](https://github.com/doublebind/raspi)，在我们的应用中，特别是确定霍尔元件的信号时起到了极大的作用 -- 对一个维度（前后或左右）移动平移台到达一个限位位置，观察 GPIO 对应的两个霍尔元件的 GPIO 电平的高低，可以很轻易得得出每个限位对应的霍尔元件信号。另外，图形化的显示也让自己在试验 GPIO 口编程时更加直观可见。使用的说明与配置见附录。