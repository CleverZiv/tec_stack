---
title: Dubbo源码学习
date: 2018-11-11 17:03
tags: [Dubbo]
categories: [学习,Dubbo]
---
## Dubbo架构初探 ##
Dubbo的角色有四种，分别是registry（注册中心）、provider（提供者）、consumer（消费者）、monitor（监控器）。
通过一个小Demo，分别启动consumer和provider，通过查看日志来探索dubbo的架构。
在此之前，我已经事先准备好了一个小项目以及在本机上利用虚拟机启动了一个zookeeper作为注册中心。

#### 一、Provider ####
启动provider，日志如下：

![Alt Text](https://myblog-1258060977.cos.ap-beijing.myqcloud.com/provider%E6%97%A5%E5%BF%97.png)
1.	启动项目；
2.	启动dubbo；
3.	Provider与远程注册中心（zookeeper作为注册中心）建立连接，此时zk需要提前启动；
4.	Provider与registry进行相关操作.
实际上，如果还有monitor，但是，因为此项目中没有开发这个模块，因此日志中没有显示出来。
对于第4步：

```java
[11/11/18 06:54:38:038 CST] main  INFO zookeeper.ZookeeperRegistry:  [DUBBO] Register: dubbo://192.168.248.1:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demotest-provider&dubbo=2.5.3&interface=com.alibaba.dubbo.demo.DemoService&methods=getPermissions&organization=dubbox&owner=programmer&pid=15740&side=provider&timestamp=1541933677025, dubbo version: 2.5.3, current host: 127.0.0.1
[11/11/18 06:54:38:038 CST] main  INFO zookeeper.ZookeeperRegistry:  [DUBBO] Subscribe: provider://192.168.248.1:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demotest-provider&category=configurators&check=false&dubbo=2.5.3&interface=com.alibaba.dubbo.demo.DemoService&methods=getPermissions&organization=dubbox&owner=programmer&pid=15740&side=provider&timestamp=1541933677025, dubbo version: 2.5.3, current host: 127.0.0.1
[11/11/18 06:54:38:038 CST] main  INFO zookeeper.ZookeeperRegistry:  [DUBBO] Notify urls for subscribe url provider://192.168.248.1:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demotest-provider&category=configurators&check=false&dubbo=2.5.3&interface=com.alibaba.dubbo.demo.DemoService&methods=getPermissions&organization=dubbox&owner=programmer&pid=15740&side=provider&timestamp=1541933677025, urls: [empty://192.168.248.1:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demotest-provider&category=configurators&check=false&dubbo=2.5.3&interface=com.alibaba.dubbo.demo.DemoService&methods=getPermissions&organization=dubbox&owner=programmer&pid=15740&side=provider&timestamp=1541933677025], dubbo version: 2.5.3, current host: 127.0.0.1
org.springframework.context.support.ClassPathXmlApplicationContext@48cf768c: here

```
*	[DUBBO] Register（注册）: dubbo的ip地址+要注册的接口+接口的方法+角色（side=provider）+时间戳，版本等。
*	[DUBBO] Subscribe（订阅）: provider的ip地址+要订阅的内容（categeory=configurator）+...。订阅完之后，只要订阅的对象有变化，注册中心就会向订阅者发送相应的数据。
*	[DUBBO] Notify urls for subscribe url: 监控的步骤，提示订阅后的角色刷新自己的缓存。


#### 二、Consumer ####

接着启动consumer，日志如下：

![Alt Text](https://myblog-1258060977.cos.ap-beijing.myqcloud.com/consumer%E6%97%A5%E5%BF%97.png)
1.	启动consumer项目；
2.	Consumer与远程注册中心连接；
3.	Consumer与registry进行相关的操作；
4.	Consumer连接provider，访问需要调用的相关服务。

```java
[11/11/18 07:05:36:036 CST] main  INFO zookeeper.ZookeeperRegistry:  [DUBBO] Register: consumer://192.168.248.1/com.alibaba.dubbo.demo.DemoService?application=demotest-consumer&category=consumers&check=false&dubbo=2.5.3&interface=com.alibaba.dubbo.demo.DemoService&methods=getPermissions&organization=dubbox&owner=programmer&pid=4148&side=consumer&timestamp=1541934335691, dubbo version: 2.5.3, current host: 192.168.248.1
[11/11/18 07:05:36:036 CST] main  INFO zookeeper.ZookeeperRegistry:  [DUBBO] Subscribe: consumer://192.168.248.1/com.alibaba.dubbo.demo.DemoService?application=demotest-consumer&category=providers,configurators,routers&dubbo=2.5.3&interface=com.alibaba.dubbo.demo.DemoService&methods=getPermissions&organization=dubbox&owner=programmer&pid=4148&side=consumer&timestamp=1541934335691, dubbo version: 2.5.3, current host: 192.168.248.1
[11/11/18 07:05:36:036 CST] main  INFO zookeeper.ZookeeperRegistry:  [DUBBO] Notify urls for subscribe url consumer://192.168.248.1/com.alibaba.dubbo.demo.DemoService?application=demotest-consumer&category=providers,configurators,routers&dubbo=2.5.3&interface=com.alibaba.dubbo.demo.DemoService&methods=getPermissions&organization=dubbox&owner=programmer&pid=4148&side=consumer&timestamp=1541934335691, urls: [dubbo://192.168.248.1:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demotest-provider&dubbo=2.5.3&interface=com.alibaba.dubbo.demo.DemoService&methods=getPermissions&organization=dubbox&owner=programmer&pid=15740&side=provider&timestamp=1541933677025, empty://192.168.248.1/com.alibaba.dubbo.demo.DemoService?application=demotest-consumer&category=configurators&dubbo=2.5.3&interface=com.alibaba.dubbo.demo.DemoService&methods=getPermissions&organization=dubbox&owner=programmer&pid=4148&side=consumer&timestamp=1541934335691, empty://192.168.248.1/com.alibaba.dubbo.demo.DemoService?application=demotest-consumer&category=routers&dubbo=2.5.3&interface=com.alibaba.dubbo.demo.DemoService&methods=getPermissions&organization=dubbox&owner=programmer&pid=4148&side=consumer&timestamp=1541934335691], dubbo version: 2.5.3, current host: 192.168.248.1
[11/11/18 07:05:37:037 CST] main  INFO transport.AbstractClient:  [DUBBO] Successed connect to server /192.168.248.1:20880 from NettyClient 192.168.248.1 using dubbo version 2.5.3, channel is NettyChannel [channel=[id: 0xc2270f63, /192.168.248.1:2475 => /192.168.248.1:20880]], dubbo version: 2.5.3, current host: 192.168.248.1
[11/11/18 07:05:37:037 CST] main  INFO transport.AbstractClient:  [DUBBO] Start NettyClient DESKTOP-RNUSDFQ/192.168.248.1 connect to the server /192.168.248.1:20880, dubbo version: 2.5.3, current host: 192.168.248.1
[11/11/18 07:05:37:037 CST] main  INFO config.AbstractConfig:  [DUBBO] Refer dubbo service com.alibaba.dubbo.demo.DemoService from url zookeeper://192.168.248.130:2181/com.alibaba.dubbo.registry.RegistryService?anyhost=true&application=demotest-consumer&check=false&dubbo=2.5.3&interface=com.alibaba.dubbo.demo.DemoService&methods=getPermissions&organization=dubbox&owner=programmer&pid=4148&side=consumer&timestamp=1541934335691, dubbo version: 2.5.3, current host: 192.168.248.1
consumer2
[Permission_0, Permission_1, Permission_2]
```
*	[DUBBO] Register（注册）: consumer的ip地址+注册角色（category=consumers）
*	[DUBBO] Subscribe（订阅）:consumer的ip地址+要订阅的内容（category=providers,configurators,routers）+...。订阅完之后，只要订阅的对象有变化，注册中心就会向订阅者发送相应的数据。
*	[DUBBO] Notify urls for subscribe url: 监控的步骤，提示订阅后的角色刷新自己的缓存。
*	后面三个[DUBBO]分别是：连接provider（192.168.248.1:20880是provider的端口）+通过Netty连接，因此需启动netty+访问注册中心，获取具体的接口信息

#### 三、Dubbo架构图 ####

![Alt Text](https://myblog-1258060977.cos.ap-beijing.myqcloud.com/Dubbo%E6%9E%B6%E6%9E%84.png)

