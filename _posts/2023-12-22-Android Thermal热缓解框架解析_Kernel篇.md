---
layout:     post
title:      Android Thermal热缓解框架解析_Kernel篇
subtitle:   
date:       2023-12-22
author:     WMY
header-img: img/ppost-bg-rwd.jpg
catalog: true
tags:
    - 工作
---


## Android Thermal热缓解框架解析_Kernel篇源码剖析


### 1. Kernel Thermal框架结构

thermal core：thermal主要的程序，驱动初始化程序，维系thermal zone、governor、cooling device三者的关系。

thermal zone device：创建thermal zone节点和连接thermal sensor，在/sys/class/thermal/目录下的thermal_zone*，通过dtsi文件进行配置生成。thermal sensor是温度传感器（即热敏电阻NTC），主要是给thermal提供温度感知。

thermal govnernor： 温度控制算法，解决温控触发时（即throttle），如何对相关的cooling device进行cooling的算法。目前主要的thermal governor有:step_wise,power_allocator,user_space等，这些算法网上有大量博客有介绍。

thermal cooling device：系统温控的执行者，实施冷却措施的驱动（cpufreq_cooling、cpuidle_cooling、 devfreq_cooling等）。cooling device根据governor计算出来的state，实施冷却操作，一般情况下，state越高表示系统的冷却需求越高。cooling device需要与trip point进行绑定，当 trip point 触发后，由相应的cooling device 去实施冷却措施。 如下时Kernel 层thermal 框架结构图：



