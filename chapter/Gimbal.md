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
2.1.选取的技术路线
2.1.1.航模中用于稳定相机的无刷 Gimbal 云台，可以基本满足我们的需求，即对偏转角度的精确控制。另外，我们的应用场景更为稳定，所以原则上可以提供比较精确的控制。
2.1.2.技术备选方案与选择
Gimbal 的控制方案有诸多开源与商业版本：
在开源固件方面，有 [BruGi](https://sourceforge.net/projects/brushless-gimbal-brugi/),[EvvGC](https://github.com/EvvGC/Firmware),[storm32bgc](https://github.com/olliw42/storm32bgc)等（在 github 上搜索 "3 Axis Gimbal" 会有很多开源项目）。其中开源固件的同时，绝大多数也是开源硬件的，特别是 STORM32 的项目。
在商业化固件方面，[BaseCam (AlexMos) SimpleBGC](http://www.basecamelectronics.com/simplebgc/) 的市场占有率很高。

硬件方面，主要是 Ardunio 和 STROM32。

市场占有率方面，AlexMos SimpleBGC 和 STROM32 的板子为主，前者更多些。

考虑到诸多因素：对 STORM32 这种 ARM 板编程的了解有限，有较大的风险存在；在之前的项目中已经有了 simple BGC 板子的使用，当时是对 2 轴的 BruGi 程序进行裁剪实现；BruGi 只适用于 2 轴；AlexMos SimpleBGC 官方发布了详细的 API 接口，并且 [Serial Protocol Specification (ver. 2.4x) ](http://www.basecamelectronics.com/serialapi/) 适用于我们的 8bit 的板子，且在 github 上有 ver. 2.5x 的实现和使用例子。
另外，AlexMos SimpleBGC 提供指令式操作，可以较少地使用线程编程，降低系统复杂度的同时，也更加可控。
我们最终决定采用 AlexMos SimpleBGC 8bit 的板子来控制无刷云台。

2.2.主控硬件
主控硬件是