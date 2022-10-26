# JVM发展历程


## Sun Classic VM
- 早在1996年Java1.0版本的时候，Sun公司发布了一款名为Sun Classic VM的java虚拟机，它同时也是世界上第一款商用Java虚拟机，JDK1.4时被完全淘汰
- 虚拟机内部提供解释器
- 如果使用JIT编译期，就需要进行外挂。但是一旦使用了JIT编译器，JIT就会接管虚拟机的执行系统。解释器就不再工作


## Exact VM
- 为了解决虚拟机问题，jdk1.2时，Sun提供了此虚拟机
- 提供了准确内存管理（Exact Memory Management）
  - 也可以叫Non-Conservative/Accurate Memory Management
  - 虚拟机可以知道内存中某个位置的数据具体是什么类型
- 具备现代高性能虚拟机的雏形
  - 热点探测
  - 编译期和解释器混合工作模式
- 只在Solaris平台（Sun公司自己的服务器）短暂使用，其他平台上还是Classic VM
  - 英雄气短，最终被Hotspot替代
