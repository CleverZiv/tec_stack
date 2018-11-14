---
title: Dubbo源码学习
date: 2018-11-11 17:03
tags: [Dubbo]
categories: [学习,Dubbo]
---
## JDK的spi机制 ##

#### 一、设计目标 ####
面向对象的设计里，模块之间是基于接口编程的，模块之间不对实现类进行硬编码。意思就是假如A模块要使用B模块的某个实现类serviceImpl，A不会直接使用serviceImpl，导致serviceImpl硬编码进A中，而是使用serviceImpl的接口service。
因为一旦代码中设计到具体的实现类，就违反了可拔插的原则，如果需要替换一种实现（相同接口的另一种实现类），就需要修改代码（将a实现改为b实现的类名称）。
为了实现在模块装配的时候，不在模块里面写死代码，这就需要一种服务发现机制，Java spi就是提供了这样一个机制：
为某个接口寻找服务实现的机制。有点类似IOC的思想，就是将装配的控制权转移到代码之外。

#### 二、具体约定 ####

当服务的提供者（provider）提供了一个接口的多种实现时，一般会在jar包的META-INF/service/目录下，创建该接口的同名文件。该文件里面的内容就是该服务接口的具体实现类的名称。而当外部加载这个模块的时候，就能通过该jar包一个接口的多种实现时，一般会在jar包的META-INF/service/里的配置文件得到具体的实现类名，并加载实例化，完成模块的装配。

1.	创建A模块，开发接口，并提供多个实现，然后打包为一个jar包。
项目结构：

![Alt Text](https://myblog-1258060977.cos.ap-beijing.myqcloud.com/JDKSPI1.png)

具体代码：
Developer.java

```java
package cn.edu.knowledge.spi;

public interface Developer {

	public String getPrograme();
}
```

JavaDeveloper.java：
```java
package cn.edu.knowledge.spi;

public class JavaDeveloper implements Developer {

	@Override
	public String getPrograme() {
		// TODO Auto-generated method stub
		return "==Java==";
	}

}
```

GoDeveloper.java：
```java
package cn.edu.knowledge.spi;

public class GoDeveloper implements Developer {

	@Override
	public String getPrograme() {
		// TODO Auto-generated method stub
		return "==Go==";
	}

}
```

2.	然后将“cn.edu.knowledge.spi”打包为jar包。
3.	新建一个项目，并将上面的jar包导入项目中，同时需要在项目的src目录下，创建META-INF/service目录，并创建对应接口的文件

![Alt Text](https://myblog-1258060977.cos.ap-beijing.myqcloud.com/JDKSPI2.png)

cn.edu.knowledge.spi.Developer
```java
cn.edu.knowledge.spi.JavaDeveloper
cn.edu.knowledge.spi.GoDeveloper
```

Client.java
```java
package com.edu.proandcon;

import java.util.Iterator;
import java.util.ServiceLoader;

import cn.edu.knowledge.spi.Developer;

public class Client {
	
	public ServiceLoader<Developer> serviceloader = ServiceLoader.load(Developer.class);

	public static void main(String[] args) {
		
		Client client = new Client();
		Developer dev = client.getDeveloper();
	}
	
	private Developer getDeveloper(){
		 Developer dev = null;
		 
		 Iterator<Developer> itr = serviceloader.iterator();
		 while(itr.hasNext()){
			 dev = itr.next();
			 System.out.println("out." + dev.getPrograme());
		 }
		 return dev;
	}

}
```
输出结果：
```java
out.==Java==
out.==Go==
```

注意：
*	这里在META-INF下的文件中指定了两个实现类，在实际中具体使用哪一个实现类，需要额外的手段控制。

#### 三、源码解析 ####

1.	获取ServiceLoader

```java
public ServiceLoader<Developer> serviceloader = ServiceLoader.load(Developer.class);
```

ServiceLoader注释：

```java
一个简单service-provider加载类。
一个service通常是一系列的接口或者类（通常是抽象类）。一个service-provider是一个service的具体实现。Provider中的类通常会实现service自身的接口并子类化这些类。Service-providers通过打包成jar可以被跨模块使用，运行在其他java扩展中。
要将一个模块中的service运行在其他模块中，第一步就是要实现加载。一个service可以被一个唯一的接口或抽象类代表。当然service可以很多种实现，在实际使用中也是根据每个实现的特点，去选择具体使用哪个service-provider。ServiceLoader对service-provider唯一的要求就是必须要有一个无参构造函数，以便在加载期间可以实例化它们。
Providers采用延迟定位和加载的方式，即在需要时才会被加载。一个service loader维护了一个至今为止加载的所有providers的缓存。迭代器方法的每次调用都会返回一个迭代器，该迭代器首先以实例化的顺序生成缓存的所有元素，然后懒惰地定位并实例化任何剩余的提供者，依次将每个提供者添加到缓存中。通过reload方法可以将缓存清空。
该类非线程安全
```

6个属性
```java
      private static final String PREFIX = "META-INF/services/";//定义实现类的接口文件所在的目录
      private final Class<S> service;//接口
      private final ClassLoader loader;//定位、加载、实例化实现类
      private final AccessControlContext acc;//权限控制上下文
      private LinkedHashMap<String,S> providers = new LinkedHashMap<>();//以初始化的顺序缓存<接口全名称, 实现类实例>
      private LazyIterator lookupIterator;//真正进行迭代的迭代器
```

核心方法：
创建实例：
```java
   /**
    * 获取当前线程的类加载器，然后调用load方法
	*/
   public static <S> ServiceLoader<S> load(Class<S> service) {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
    }
	
	/**
    * 调用ServiceLoader的私有构造函数，创建ServiceLoader实例
	*/
	   public static <S> ServiceLoader<S> load(Class<S> service,
                                            ClassLoader loader)
    {
        return new ServiceLoader<>(service, loader);
    }
	
	/**
    * ServiceLoader的私有构造函数，分别给service、loader、acc三个属性赋值，并清空缓存
	*/
	   private ServiceLoader(Class<S> svc, ClassLoader cl) {
        service = Objects.requireNonNull(svc, "Service interface cannot be null");
        loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
        acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
        reload();
    }
	
	/**
    * 清空缓存，即清空providers属性，并且新建LazyIterator迭代器
	*/
	    public void reload() {
        providers.clear();
        lookupIterator = new LazyIterator(service, loader);
    }
	
	//至此ServiceLoader的5个非静态属性全部初始化完毕，获得一个新的ServiceLoader实例
```

2.	获取迭代器并迭代

```java
Iterator<Developer> itr = serviceloader.iterator();
		 while(itr.hasNext()){
			 dev = itr.next();
			 System.out.println("out." + dev.getPrograme());
		 }
```

具体的源码不再贴出来，有兴趣再去看。
具体机制如下：
*	hasNext()：先从provider（缓存）中查找，如果有，直接返回true；如果没有，通过LazyIterator来进行查找。
*	next()：先从provider（缓存）中直接获取，如果有，直接返回实现类对象实例；如果没有，通过LazyIterator来进行获取。
hasNext()中调用了hasNextService()方法，其核心实现是：
*	首先使用loader加载配置文件，此时找到了META-INF/services/下的文件；
*	然后解析这个配置文件，并将各个实现类名称存储在pending的ArrayList中；
*	最后指定nextName;
Next()中调用了nextService()核心实现如下：
*	首先加载nextName代表的类Class
*	之后创建该类的实例，并转型为所需的接口类型
*	最后存储在provider中，供后续查找，最后返回转型后的实现类实例。

#### 三、缺点 ####
查找一个具体的实现需要遍历查找，耗时；-->此时就体现出Collection相较于Map差的地方，map可以直接根据key来获取具体的实现 （dubbo-spi实现了根据key获取具体实现的方式）。
