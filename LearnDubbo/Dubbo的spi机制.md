---
title: Dubbo源码学习
date: 2018-11-11 17:03
tags: [Dubbo]
categories: [学习,Dubbo]
---
## Dubbo的spi机制 ##

#### 一、为什么dubbo不直接用JDK的spi ####

1.	JDK标准的SPI会一次性实例化扩展点（模块）的所有实现，导致初始化阶段很耗时。
2.	并且即使没有用上也会被加载，浪费资源
Dubbo的spi机制增加了对扩展点IoC和AOP的支持，一个扩展点可以直接setter注入其它扩展点；采用键值对的形式存储，不再需要全部加载所有的实现。

#### 二、Dubbo spi的约定 ####

1.	Spi文件的存储路径在“META-INF/dubbo/internal”目录下，并且文件名为接口的全路径名，就是“接口包名+接口名”，命名与JDK中是一样的。
2.	每个spi文件里面的格式定义为：扩展名=具体的类名。例如：
dubbo=com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtoco

#### 三、Dubbo spi的思路 ####

1.	Dubbo spi的目的：获取一个接口的实现类的对象。
2.	途径：ExtensionLoader.getExtension(String name)
3.	实现路径：
*	getExtensionLoader(Class<T> type) 就是为该接口创建一个ExtensionLoader，然后缓存起来。
*	getAdaptiveExtension() 获取一个扩展装饰类的对象，这个类有一个规则，如果它没有一个@Adaptive注解，就动态创建一个装饰类，例如Protocol$Adaptive对象。
*	getExtension(String name) 获取一个对象。

#### 四、源码分析 ####

##### 实现路径1： getExtensionLoader #####
首先找到
```java
E:\PRO\SourceCode\dubbo-dubbo-2.5.4\dubbo-demo\dubbo-demo-provider\src\test\java\com\alibaba\dubbo\demo\provider\DemoProvider.java
```
代码如下：
```java
package com.alibaba.dubbo.demo.provider;

public class DemoProvider {

    public static void main(String[] args) {
        com.alibaba.dubbo.container.Main.main(args);
    }

}
```
进入到该Main.java文件，看到第四行代码：

```java
private static final ExtensionLoader<Container> loader = ExtensionLoader.getExtensionLoader(Container.class);
```
重点就是分析这个getExtensionLoader（Class<T> type）函数。此函数获取到的ExtensionLoader可以类比为JDK-SPI中的ServiceLoader。

ExtensionLoader的属性：

```java
/** SPI接口Class */
    private final Class<?> type;
    /** SPI实现类对象实例的创建工厂 */
    private final ExtensionFactory objectFactory;
    /** key: ExtensionClass的Class value: SPI实现类的key */
    private final ConcurrentMap<Class<?>, String> cachedNames = new ConcurrentHashMap<Class<?>, String>();
    /** 存放所有的extensionClass */
    private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<Map<String, Class<?>>>();

    private final Map<String, Activate> cachedActivates = new ConcurrentHashMap<String, Activate>();
    /** 缓存创建好的extensionClass实例 */
    private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<String, Holder<Object>>();
    /** 缓存创建好的适配类实例 */
    private final Holder<Object> cachedAdaptiveInstance = new Holder<Object>();
    /** 存储类上带有@Adaptive注解的Class */
    private volatile Class<?> cachedAdaptiveClass = null;
    /** 默认的SPI文件中的key */
    private String cachedDefaultName;
    /** 存储在创建适配类实例这个过程中发生的错误 */
    private volatile Throwable createAdaptiveInstanceError;
    /** 存放具有一个type入参的构造器的实现类的Class对象 */
    private Set<Class<?>> cachedWrapperClasses;
    /** key :实现类的全类名  value: exception, 防止真正的异常被吞掉 */
    private Map<String, IllegalStateException> exceptions = new ConcurrentHashMap<String, IllegalStateException>();
```

getExtensionLoader（Class<T> type）
```java
/**
     * 1 校验入参type：非空 + 接口 + 含有@SPI注解
     * 2 根据type接口从全局缓存EXTENSION_LOADERS中获取ExtensionLoader,如果有直接返回;如果没有,则先创建,之后放入缓存,最后返回
     */
    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        if (type == null)
            throw new IllegalArgumentException("Extension type == null");
        if (!type.isInterface()) {
            throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
        }
        if (!withExtensionAnnotation(type)) {
            throw new IllegalArgumentException("Extension type(" + type +
                    ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
        }

        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }

```

初始化时全局缓存EXTENSION_LOADERS中没有对应的type（此时是Container）.因此会先进行创建。即执行：

```java
EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
```
创建ExtensionLoader
```java
private ExtensionLoader(Class<?> type) {
        this.type = type;
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }

```

这是第一个ExtensionLoader对象。
此时type=Container.class，于是执行这条语句：
```java
ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension()
```
可以看到，这条语句的前半句，与最开始获取ExtensionLoader是一样的，于是再次进入getExtensionLoader，继续执行一次，然后又来到构造器中。但此时的type= ExtensionFactory.class，于是此次的ExtensionLoader对象中的objectFactory=null。接着程序继续执行，将“loader”存入全局缓存中，并返回。
因此整个过程执行完毕之后，产生了两个ExtensionLoader。流程简化如下：

```java
-----------------------ExtensionLoader.getExtensionLoader(Class<T> type)
ExtensionLoader.getExtensionLoader(Container.class)
  -->this.type = type;
  -->objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
  //ExtensionLoader对象1：type=Container.class, objectFactory=ExtensionLoader对象2.getAdaptiveExtension();objectFactory是ExtensionFactory对象
  //对象1的objectFactory的属性依赖对象2
     -->ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension()
       -->this.type = type;
       -->objectFactory =null;
	   //ExtensionLoader对象2：type=ExtensionFactory.class,objectFactory =null
	   //产生的两个ExtensionLoader对象都会放在全局缓存EXTENSION_LOADERS中，因此最后缓存中是这样的{{ExtensionFactory,ExtensionLoader[ExtensionFactory]}, {Container, ExtensionLoader[Container]}}
       
执行以上代码完成了2个属性的初始化
1.每个一个ExtensionLoader都包含了2个值 type 和 objectFactory
  Class<?> type；//构造器  初始化时要得到的接口名
  ExtensionFactory objectFactory//构造器  初始化时 AdaptiveExtensionFactory[SpiExtensionFactory,SpringExtensionFactory]
2.new 一个ExtensionLoader 存储在ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS

关于这个objectFactory的一些细节：
1.objectFactory就是ExtensionFactory，它也是通过ExtensionLoader.getExtensionLoader(ExtensionFactory.class)来实现的，但是它的objectFactory=null
2.objectFactory作用，它就是为dubbo的IOC提供所有对象。

```
以上就是路径1的分析。

##### 路径2：getAdaptiveExtension() #####
找到ServiceConfig.java
```java
E:\PRO\SourceCode\dubbo-dubbo-2.5.4\dubbo-config\dubbo-config-api\src\main\java\com\alibaba\dubbo\config\ServiceConfig.java
```
第二行代码：
```java
private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).
            getAdaptiveExtension();

```

代码的前面部分上面已经详细分析过，接下来主要看getAdaptiveExtension()
```java
public T getAdaptiveExtension() {
        Object instance = cachedAdaptiveInstance.get(); //A
        if (instance == null) {
            if (createAdaptiveInstanceError == null) {
                synchronized (cachedAdaptiveInstance) {
                    instance = cachedAdaptiveInstance.get();
                    if (instance == null) {
                        try {
                            instance = createAdaptiveExtension(); //B
                            cachedAdaptiveInstance.set(instance); //C
                        } catch (Throwable t) {
                            createAdaptiveInstanceError = t;
                            throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
                        }
                    }
                }
            } else {
                throw new IllegalStateException("fail to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
            }
        }

        return (T) instance;
    }
```
关注代码中标注的A、B、C综合可知，该方法的作用是：获取cachedAdaptiveInstance变量的值，如果该值为空，则调用createAdaptiveExtension()方法，并对其进行赋值再返回该值。
cachedAdaptiveInstance类变量定义如下：

```java
private final Holder<Object> cachedAdaptiveInstance = new Holder<Object>();
```
Holder是工具包中的一个类，作用是帮助Class保存一个对象（值）

```java
package com.alibaba.dubbo.common.utils;

/**
 * Helper Class for hold a value.
 *
 * @author william.liangf
 */
public class Holder<T> {

    private volatile T value;

    public void set(T value) {
        this.value = value;
    }

    public T get() {
        return value;
    }

}

```
createAdaptiveExtension()：

```java
private T createAdaptiveExtension() {
        try {
            return injectExtension((T) getAdaptiveExtensionClass().newInstance());
        } catch (Exception e) {
            throw new IllegalStateException("Can not create adaptive extenstion " + type + ", cause: " + e.getMessage(), e);
        }
    }

```
getAdaptiveExtensionClass()：

```java
private Map<String, Class<?>> getExtensionClasses() {
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                    classes = loadExtensionClasses();
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
    }

```
getExtensionClasses()方法的作用是，获取cachedClasses变量的值，如果没有就调用loadExtensionClasses()进行创建。
再往下分析loadExtensionClasses方法，下面还有loadFile()方法，由于代码量过多，不再贴出完整方法，毕竟也没必要（最后会整理一下整个方法调用路径），关注的是整个思路。

```java
loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
loadFile(extensionClasses, DUBBO_DIRECTORY);
loadFile(extensionClasses, SERVICES_DIRECTORY);
```

loadFile作用是加载配置文件信息，将配置文件中的配置信息缓存在变量中，这个方法还是比较复杂。

```java
关于loadfile的一些细节
目的：通过把配置文件META-INF/dubbo/internal/com.alibaba.dubbo.rpc.Protocol的内容，存储在缓存变量里面。
cachedAdaptiveClass//如果这个class含有adative注解就赋值，例如ExtensionFactory，而例如Protocol在这个环节是没有的。
cachedWrapperClasses//只有当该class无adative注解，并且构造函数包含目标接口（type）类型,
                    //例如protocol里面的spi就只有ProtocolFilterWrapper和ProtocolListenerWrapper能命中
cachedActivates//剩下的类，包含Activate注解
cachedNames//剩下的类就存储在这里。
```

整个路径2的总结：

```java

1-->getAdaptiveExtension()//为cachedAdaptiveInstance赋值
  2-->createAdaptiveExtension()
    3-->getAdaptiveExtensionClass()
      4-->getExtensionClasses()//为cachedClasses 赋值
        5-->loadExtensionClasses()
          6-->loadFile
      4-->createAdaptiveExtensionClass()//自动生成和编译一个动态的adpative类，这个类是一个代理类，运用的是模板方法。模板方法见adative-code-templet文件
        5-->ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension()
        5-->compiler.compile(code, classLoader)
    3-->injectExtension()//作用：进入IOC的反转控制模式，实现了动态注入
1-->最后回到getAdaptiveExtension()

为什么要设计adaptive？注解在类上和注解在方法上的区别？
adaptive设计的目的是为了识别固定已知类和扩展未知类。
1.注解在类上：代表人工实现，实现一个装饰类（设计模式中的装饰模式），它主要作用于固定已知类，
  目前整个系统只有2个，AdaptiveCompiler、AdaptiveExtensionFactory。
  a.为什么AdaptiveCompiler这个类是固定已知的？因为整个框架仅支持Javassist和JdkCompiler。
  b.为什么AdaptiveExtensionFactory这个类是固定已知的？因为整个框架仅支持2个objFactory,一个是spi,另一个是spring
2.注解在方法上：代表自动生成和编译一个动态的Adpative类，它主要是用于SPI，因为spi的类是不固定、未知的扩展类，所以设计了动态$Adaptive类.
例如 Protocol的spi类有 injvm dubbo registry filter listener等等 很多扩展未知类，
它设计了Protocol$Adaptive的类，通过ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(spi类);来提取对象

```

adative-code-templet文件：

```java
package <扩展点接口所在包>;
 
public class <扩展点接口名>$Adpative implements <扩展点接口> {
    public <有@Adaptive注解的接口方法>(<方法参数>) {
        if(是否有URL类型方法参数?) 使用该URL参数
        else if(是否有方法类型上有URL属性) 使用该URL属性
        # <else 在加载扩展点生成自适应扩展点类时抛异常，即加载扩展点失败！>
         
        if(获取的URL == null) {
            throw new IllegalArgumentException("url == null");
        }
 
              根据@Adaptive注解上声明的Key的顺序，从URL获致Value，作为实际扩展点名。
               如URL没有Value，则使用缺省扩展点实现。如没有扩展点， throw new IllegalStateException("Fail to get extension");
 
               在扩展点实现调用该方法，并返回结果。
    }
 
    public <有@Adaptive注解的接口方法>(<方法参数>) {
        throw new UnsupportedOperationException("is not adaptive method!");
    }
}

```

##### 实现路径3： getExtension(String name) #####

```java
/**
     * 从cachedInstances缓存中获取name对应的实例,如果没有,通过createExtension(name)创建,之后放入缓存
     * getExtension(String name)
     * --createExtension(String name)
     * ----injectExtension(T instance)
     */
    public T getExtension(String name) {
        if (name == null || name.length() == 0)
            throw new IllegalArgumentException("Extension name == null");
        if ("true".equals(name)) {
            return getDefaultExtension();
        }
        Holder<Object> holder = cachedInstances.get(name);
        if (holder == null) {
            cachedInstances.putIfAbsent(name, new Holder<Object>());
            holder = cachedInstances.get(name);
        }
        Object instance = holder.get();
        if (instance == null) {
            synchronized (holder) {
                instance = holder.get();
                if (instance == null) {
                    instance = createExtension(name);
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
    }

```
来看一下创建createExtension(name)

```java
private T createExtension(String name) {
        /** 从cachedClasses缓存中获取所有的实现类map,之后通过name获取到对应的实现类的Class对象 */
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
            /** 从EXTENSION_INSTANCES缓存中获取对应的实现类的Class对象,如果没有,直接创建,之后放入缓存 */
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                EXTENSION_INSTANCES.putIfAbsent(clazz, (T) clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            injectExtension(instance);//ioc
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (wrapperClasses != null && wrapperClasses.size() > 0) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                    type + ")  could not be instantiated: " + t.getMessage(), t);
        }
    }

```

这里，就体现出来了dubbo-SPI比JDK-SPI的好处：dubbo-SPI不需要遍历所有的实现类来获取想要的实现类，可以直接通过name来获取。
injectExtension(instance)和wrapper包装功能后续再说。
```java
getExtension(String name) //指定对象缓存在cachedInstances；get出来的对象wrapper对象，例如protocol就是ProtocolFilterWrapper和ProtocolListenerWrapper其中一个。
  1-->createExtension(String name)
    2-->getExtensionClasses()
    2-->injectExtension(T instance)//dubbo的IOC反转控制，就是从spi和spring里面提取对象赋值。
      3-->objectFactory.getExtension(pt, property)
        4-->SpiExtensionFactory.getExtension(type, name)
          5-->ExtensionLoader.getExtensionLoader(type)
          5-->loader.getAdaptiveExtension()
        4-->SpringExtensionFactory.getExtension(type, name)
          5-->context.getBean(name)
    2-->injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance))//AOP的简单设计
    
```
最终返回的是创建好的具体实现类，例如SpringExtensionFactory实例。

参考博文：A [Link](https://www.cnblogs.com/java-zhao/category/1090034.html "dubbo源码解析")
