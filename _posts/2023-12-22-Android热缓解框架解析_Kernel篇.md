---
layout:     post
title:      Android Thermal热缓解框架解析_Kernel篇
subtitle:   
date:       2023-12-22
author:     WMY
header-img: img/bg-me-2022.jpg
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

![](https://wwmmyy2023.github.io/img/thermal/thermal_framework.png)


### 2.Thermal Core的启动流程

从kernel6.1 之后thermal core启动流程框架做了一些更新，主要差别如下：

#### 2.1 旧架构流程如下(kernel5.15及之前版本)：

![](https://wwmmyy2023.github.io/img/thermal/thermal_core_old.png)

#### 2.2 新架构流程如下：

![](https://wwmmyy2023.github.io/img/thermal/thermal_core.png)

#### 2.3 Thermal Core的新旧架构优缺点分析

**(1) 旧架构的缺点：**

旧架构独立创建不同的thermal组件，这种方法导致代码重复，因为它首先创建了一个thermal zone，然后才会注册对应的thermal sensor。这样thermal zone在初始化时无条件地创建和使用假的thermal sensor，因此需要提供虚假操作并将所有thermal zone相关信息存储在重复的结构中。然后在初始化thermal sensor传感器时，并且代码查找设备树中的thermal zone名称 并将thermal sensor与thermal zone关联，并使用来自thermal zone操作的第二级间接调用thermal sensor的特定操作。

当thermal sensor被移除（通过模块卸载）时，阈值对应的thermal zone仍然带有虚假thermal sensor继续存在。与thermal zone和trip相关联的cooling device存储在列表中，使用设备树中的节点名称以匹配后续cooling device。也是重复信息

**(2) 新架构的优点：**

新架构更简单，它在注册thermal sensor时就创建对应thermal zone，并在thermal sensor移除时销毁相应的thermal zone。 cooling device、trip点和thermal zone之间的所有匹配都使用设备树以及绑定完成。操作不再是特定的，而是使用热thermal框架提供的通用操作。
从kernel6.1开始已经采用新的框架，旧的代码已经删除，使的thermal框架变的简单了 

**(3) 新架构中移除的函数：**

thermal_zone_of_sensor_register 在kernel6.1中已移除， 它代表注册一个实体的ntc 器件sensor，它通过thermal zone找到该sensor id，然后注册thermal sensor， 新架构中已不需要该函数。

**(4) 新架构中新增函数：**

kernel6.1开始，新增了 devm_thermal_of_zone_register 函数，主要功能点见上图。

thermal_zone_device_register 代表注册一个thermal zone节点，他的温度触发值依赖一个ntc sensor的， 它是在原先的thermal_init函数初始化，通过of_parse_thermal_zones解析设备树中的thermal zone节点时，调用了该接口，将设备数树中所有配置的thermal zone都给注册了


### 3. Kernel Thermal框架重点模块功能点源码剖析：

#### 3.1 Thermal Govnernor模块：

 (1) thermal govnernor的初始化流程
 
![](https://wwmmyy2023.github.io/img/thermal/governor.png)

通过如上流程可知：假如要新增一个governor，也可以仿照如上gov_step_wise.c等方式，通过调用THERMAL_GOVERNOR_DECLARE宏加入进来。
  
 (2) thermal govnernor 如何跟Thermal zone绑定
 
 ![](https://wwmmyy2023.github.io/img/thermal/thermal_bind_gov.png)
 
 
 除了初始化时thermal zone跟thermal bind之外，后续可以通过写sysfs节点来调整thermalzone对应的governor，对应源码如下：
 
  ![](https://wwmmyy2023.github.io/img/thermal/thermal_gov_pilicy.png)

#### 3.2 Thermal zone Temperature 更新原理：

3.2.1 方式1:主动读取对应sysfs温度节点获取thermal zone温度值：

 user_space层读取 /sys/class/thermal/thermal_zone*/temp时，会走到如下函数流程中：

  ![](https://wwmmyy2023.github.io/img/thermal/cat_temp.png)

3.2.2 方式2:系统定期轮询更新thermal zone温度值

对于非底层硬件中断触发而更新温度值的一些thermal zone，系统是通过给对应thermal zone 设定一个定时触发的轮询线程来更新温度值的。轮询的周期可以在设备树中进行配置，且可以配置触发设定温度阈值后的轮询周期（对应），及低于触发温度阈值的轮询周期（对应polling-delay值）。代码运行流程如下图：

  ![](https://wwmmyy2023.github.io/img/thermal/polling.png)

通过如上流程可知：

（1）若设定的polling-delay或polling-delay-passive值为0，轮询线程不会工作，也就不会定期轮询更新温度。如果没有底层硬件中断上报更新该thermal zone温度，同时userspace层也没有去主动读取thermal zone 温度值的话，该thermal zone的系统温度值就不会变化，也就是说即使设置了trip 值和对应的cooling_device策略，就算实际温度已经超过trip值也不会触发 对应的策略。

（2）若对应thermal zone状态为disabled，系统也不会再轮询更新温度。

3.2.3 方式3:通过硬件中断上报更新thermal zone温度值

如下以芯片的tsensor 为例，说明如何通过硬件中断方式更新温度的。

  ![](https://wwmmyy2023.github.io/img/thermal/thermalzone_interrupt.png)

#### 3.3 Thermal zone cooling Device工作流程

当thermal zone的温度值超过设定的trip值后，如果该thermal zone有配置对应trip值的cooling device策略，就会执行对应的cooling策略。如下是关键的运行流程源码：

  ![](https://wwmmyy2023.github.io/img/thermal/coolingdevice.png)

从上图可知如果thermal zone温度值超过了配置测critical温度阈值，则会调用到handle_critical_trips函数，走进高温关机保护流程，见下图：

  ![](https://wwmmyy2023.github.io/img/thermal/trip_power_off.png)


#### 3.4 典型的Thermal zone sensor 运行示例

下面以一个名称为”tboard1“ 的thermal zone初始化加载为例，展示下thermal zone的初始化流程。

第1步：设备树中配置对应的设备和thermal zone

第2步：kernel初始化启动时，会解析设备树，并加载执行对应的驱动逻辑，驱动初始化运行时会注册生成对应的thermal zone， 见下图：

  ![](https://wwmmyy2023.github.io/img/thermal/athermalzone_init.png)



参考链接：

(1) https://hqber.com/archives/427/

(2) https://cs.android.com/android/kernel


