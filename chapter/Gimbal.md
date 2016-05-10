方向的定义：
1.[摄像机的roll,yaw,pitch](https://segmentfault.com/a/1190000000408831)
里面有个不错的图
2.[什么是姿态角（Euler角）pitch yaw roll ](http://www.123kuai.com/index.php?a=show&c=index&catid=9&id=54&m=content)
<< 这里应该有一张不错的示意图

闭环系统：



## Gimbal
### 1.需求分析
实现对滚转角Φ(roll)、俯仰角 θ(pitch)、偏航角ψ(yaw)的精确控制。控制精度在 0.02° 以内。
提供自动限位功能，防止平台出现不合理移动。
提供停止移动功能。
提供每次运动到达指定位置或者是运动出错的信息回馈。

### 2.概要设计
#### 2.1.选取的技术路线
2.1.1.控制硬件选择
航模中用于稳定相机的无刷 Gimbal 云台，可以基本满足我们的需求，即对偏转角度的精确控制。另外，我们的应用场景更为稳定，所以原则上可以提供比较精确的控制。所以，从无刷云台着手选取控制方案。

2.1.1.1.Gimbal 控制技术备选方案与选择
可以直接控制无刷电机来达到目的，但是需要很多的硬件知识，而我们对这些了解有限，且会增添诸多不必要的工作量，最终效果也不见得会很好，所以，考虑整套的解决方案。

2.1.1.2.Gimbal 的整体控制方案有诸多开源与商业版本：
在开源固件方面，有 [BruGi](https://sourceforge.net/projects/brushless-gimbal-brugi/),[EvvGC](https://github.com/EvvGC/Firmware),[storm32bgc](https://github.com/olliw42/storm32bgc)等（在 github 上搜索 "3 Axis Gimbal" 会有很多开源项目）。其中开源固件的同时，绝大多数也是开源硬件的，特别是 STORM32 的项目。
在商业化固件方面，[BaseCam (AlexMos) SimpleBGC](http://www.basecamelectronics.com/simplebgc/) 的市场占有率很高。
控制硬件方面，主要是 Ardunio 和 STROM32 的板子。
市场占有率方面，AlexMos SimpleBGC 和 STROM32 的板子为主，前者更多些。

考虑到诸多因素：对 STORM32 这种 ARM 板编程的了解有限，有较大的风险存在；在之前的项目中已经有了 simple BGC 板子的使用，当时是对 2 轴的 BruGi 程序进行裁剪实现；BruGi 只适用于 2 轴；AlexMos SimpleBGC 官方发布了详细的 API 接口，并且 [Serial Protocol Specification (ver. 2.4x) ](http://www.basecamelectronics.com/serialapi/) 适用于我们的 8bit 的板子，且在 github 上有 ver. 2.5x 的实现和使用例子。
另外，AlexMos SimpleBGC 提供指令式操作，可以较少地使用线程编程，降低系统复杂度的同时，也更加可控。
我们最终决定采用 AlexMos SimpleBGC 8bit 的板子来控制无刷云台。

2.1.2.主控硬件与编程语言
主控硬件是 Raspberry(Model 2B)，选用原因是树莓派作为一个性能很不错的搭载了通用 linux 系统的硬件平台，编程较为方便，支持 C++、python、java 等开发。
编程语言选择，因为我们的应用属于嵌入式硬件编程，且要对 raspberry 的 GPIO 口进行编程，且对时序有比较高的要求，所以选用 C++，另外，为充分利用现代 C++ 的各种更加便捷的功能与诸多标准库（STL 等），选用 C++11。

2.1.3.编程环境
因为一直以来多在 windows 下进行开发，对 visual studio IDE 比较熟悉，另外 raspberry 直接编译大些的程序还是比较耗时的，而使用交叉编译可以明显地减少编译时间，所以，选用 visual studio 搭配合适的插件来进行 linux 嵌入式开发。比较知名的插件有 [WinGDB](http://wingdb.com/)、[VisualGDB](http://www.visualgdb.com/) 等，它们都提供直接在 windows 上调试 linux 程序，比较适合我们的应用场景。我们选用的是 VisualGDB，使用其他工具应当并无明显差别。

#### 2.2.整体实现思路
根据 API 接口 2.4x 版本适当更改官方提供的 C/C++ 库以支持我们 8bit 的控制板；
调用 simpleBGC API 实现 roll、pitch、yaw 的精确控制；
向上层软件提供合适的调用接口。

#### 2.3.关键技术难点
实验室之前的项目已经有对两轴（roll、pitch）控制的实现，但没有尝试过对 yaw 的精确控制；
操作板标配的 MPU6050 的 yaw 轴度数会有飘的问题（板子即使静止时度数也会一直减少或增加）；

也就是说对 yaw 轴的精确控制是存在风险的。

#### 2.4.术语定义
上面提到的 roll、pitch、yaw 是飞行动力学（Flight dynamics）中的专有名词，表示飞行器重心在三个维度的角度偏转，具体见附录；
PID 控制（proportional–integral–derivative control，比例-积分-微分控制），是一种控制系统中的循环反馈机制，在 Gimbal 的控制中起到很大的作用，详情见附录，Gimbal 的调参参见附录；

#### 2.5.模块概览

<< 这里来张图

#### 2.6.模块划分
1.SerialPort:参照 [libserial](https://github.com/jyksnw/libserial) 扩展 WiringPI 库中的 wiringSerial.c 提供类的包装，且增添适用于 linux 的端口枚举（port enumerate）功能；
2.SBGC_lib:根据 Alexmos SimpleBGC 的 2.4x 版本的 API 接口说明，对官方提供的库进行修改与增添方法，以适合我们的 8bit 的板子与提供更多的功能；
3.raspSBGC:调用 SerialPort 与 SBGC_lib，向外提供所有需要的控制角度偏转方法；
4.GimbalCommand:定义 Gimbal 移动的命令格式；
5.ServerController:解析/打包 Gimbal 数据，调用 raspSBGC 的方法；
6.AuSocketServer:Socket 库；
7.RaspberryServerMain:将程序部署到 raspberry 上，打开 Socket 通信与上层软件进行交互，接受到数据后，调用 SerialController 进行解析与运行，将返回值通过 Socket 返回到上层软件。

#### 2.7.测试计划
raspSBGC 的所有方法，可以直接写 cpp 文件（GimbalSBGC_Test.cpp）进行测试；
实现 communicateTest 工程（为简单且提供 GUI 使用 C# 开发）来进行经由 Socket 发送运动指令的测试；

### 3.详细设计
#### 3.1.SBGC_lib 库
使用：参考官方的 Ardunio 的程序与 lib 中的程序，要使用此库只需提供适用于自己平台的四个虚函数的具体实现即可([SBGC_parser.h](https://github.com/alexmos/sbgc-api-examples/blob/bb3fe16fea01fb68da6560240b160973b07879f4/libraries/SBGC_lib/include/SBGC_parser.h))：
``` C++
/* Need to be implemented in the main code */
class SBGC_ComObj {
public:
    // Return the number of bytes received in input buffer
    virtual uint16_t getBytesAvailable() = 0;
    // Read byte from the input stream
    virtual uint8_t readByte() = 0;
    // Write byte to the output stream
    virtual void writeByte(uint8_t b) = 0;
    // Return the space available in the output buffer. Return 0xFFFF if unknown.
    virtual uint16_t getOutEmptySpace() = 0;
};
```

而这些方法正是我们的 SerialPort 类提供的；

更改：因为官方库是根据 ver 2.5x 的 API 说明来写的，与我们正在使用的板子对应的 API 有些差异，需要修改一下。特别是 SBGC_cmd_realtime_data_t 有较大的差异，根据 2.4x 版本的说明我们进行修改，添加 SBGC_cmd_realtime_data_t_v24 的结构体，并且给出其解析方法；

添加：官方库提供的功能有限，根据 API 说明增添其他功能：
SBGC_direct_cmd_send 直接向板子发送不需要 data 的指令，比如 readRealtimeData, Motor on/off 等；
SBGC_direct_cmd_send_with_data 向板子发送已包装好简单 data 的指令，比如更改使用的 profile 文件等；
SGBC_realtime_angle:当前角度的结构体定义及相关 unpack 函数（SGBC_realtime_angle_unpack）；
serialFlush 与 okToRead:调用 SerialPort，在每次发送数据前清空接受/发送缓冲区，判断所有返回数据都达到，可以进一步处理。


#### 3.2.SerialPort 的设计
功能：提供 raspberry 与 simpleBGC 板子进行串口通信的方式；

findDevice:端口枚举，自动寻找已经连接的串口；

serialAllDataArrived:判断所有接收数据均到来，这个方法并不是通用的，只在在每个 request 后有一个 response 时有用。对应于我们的应用，向控制板发送不同请求得到不同长度响应时确保所有响应均收到，不然的话，仅接受部分数据 SBGC_lib 中对接收的数据的处理会返回错误。

提供了使用 SBGC_lib 所需的四个虚函数的实现；

提供简单错误处理：若接受到的数据量过大（在我们这个应用中，调用的方法 simpleBGC 控制板返回数据正常情况下数据小于 200）时直接关闭串口；（向板子的输入数据过滤在 raspSBGC 进行处理）

#### 3.3.raspSBGC 的设计
注：RTVector3_T 为一个模板三元组类，RTFLOAT 为 float 或 double 类型，因为 simpleBGC 内部将角度转化为整数便于进行其他处理。
功能：调用并封装 SBGC_lib 的方法及提供策略来实现对 Gimbal 平台的控制，并向外提供调用接口；

外部接口定义：
raspSBGC_setup:初始化 Gimbal 平台与相关设备；

raspSBGC_autoHorizonal:Gimbal 平台保持水平，这在相机拍照时一般是需要的；

raspSBGC_move_less_precise(RTVector3_T<RTFLOAT> targetVec)：Gimbal 移动一定角度，less_precise 是说我们采用的是 SBGC_lib 提供的三种移动模式 SBGC_CONTROL_MODE_SPEED、SBGC_CONTROL_MODE_ANGLE、SBGC_CONTROL_MODE_SPEED_ANGLE 中的角度模式，API 说明中说使用角度+速度模式可以得到更精确的控制，但是我们的应用场景是 Gimabl 在一个很稳定的台子上面，只需角度模式便足以达到要求。

内部接口定义（根据 API 接口格式向控制板发送数据）：
motor_on:电机上电，调用 SBGC_direct_cmd_send，命令种类是 SBGC_CMD_MOTORS_ON；
motor_off:电机断电，调用 SBGC_direct_cmd_send，命令种类是 SBGC_CMD_MOTORS_OFF；
targetAngleResonable(RTVector3_T<RTFLOAT> targetVec)：判断目标角度值是否合理；
setNowAngleInt(RTVector3_T<int> nowVecInt)、getNowAngleInt(RTVector3_T<int>& copyNowVecInt):当前角度线程安全的 get，set 方法；
updateNowAngleInt:更新当前角度；
raspSBGC_realtime_data_v24(SBGC_cmd_realtime_data_t_v24 &realtime_data):发送获取当前运行信息指令，获取数据后调用 SBGC_cmd_realtime_data_unpack_v24 解析返回数据；
raspSBGC_realtime_angle(SGBC_realtime_angle &realtime_angle):发送获取当前角度指令，获取数据后调用 SGBC_realtime_angle_unpack 解析返回数据；
delta_degree_to_target_degree(RTVector3_T<RTFLOAT> deltaVec, RTVector3_T<RTFLOAT>& targetVec):将增量角度转化为目标角度；
wait_move_succuss(...):轮询当前角度，当到达目标角度时进行相应处理，提供诸多参数进行配置，以更好地适应于我们的需求；
still_safe_to_move:移动的过程中判断是否到达限位，一般另开一个线程来运行，保证不会因为到达限位而对 Gimbal 平台造成损害；
raspSBGC_movingTowards(RTVector3_T<int> PositiveNegativeStill, float degreePerSec = 0.3):控制平台以指定速度（此速度由 simpleBGC 控制，不保证完全精确）向特定方向移动，直到下一指令来临。

### 4.编码实现及遇见的问题及解决方案
#### 4.1.simpleBGC API Specification ver2.4x 中的小错误
在我们尝试读取板子的 real_time_data 时总是无法正确解析数据，通过对比返回的字节数与 API 说明中的字节数发现说明书中有三个字节的缺失，通过在 github 上找到的[另一位使用 C# 实现了 2.4x 版本的程序](https://github.com/park53kr/SimpleBGC_GUI/blob/master/MousePointTracker/MousePointTracker/gimbal.cs) 中的 RealtimeDataStructure 数据结构定义发现是缺失了表示电机 power 的三个字节字段。更新我们自己的程序后，正常解析读取到的 real_time_data 数据。
#### 4.2.MPU6050 读取的 yaw 轴数据并不稳定
yaw 轴数据在上电时重置为 0，之后变向一个方向一直漂移。如此一来，在使用 simpleBGC 自己的策略时会保证 yaw 轴数据为 "0"，也就是会随着 yaw 轴的数据漂移而漂移。而这是我们所不能接受的，所以，需要对 yaw 轴重新设计控制策略。

备选方案：
使用 MPU9250 替代目前的 MPU6050，MPU9250 多出了磁强计（magnetometer），将磁强计与 MPU6050 的加速度计以及重力计数据进行融合，可以得到较为精确的 yaw 轴数据。

#### 4.3.API 接口中没有提供单独对某个轴
