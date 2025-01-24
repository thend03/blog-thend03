---

draft: false
date: 2023-07-12T22:52:55+08:00
title: "dubbo spi"
slug: "dubbo-spi" 
tags: ["dubbo","spi"]
authors: ["since"]
description: "dubbo spi"
---

## SPI

spi的全称是service provider interface，是一种服务发现机制，动态选择接口的实现类。

Java中的spi是在META-INF/services提供一个名称为接口class的文件，里面配置具体的接口实现类。使用ServiceLoader加载spi。

Dubbo的spi是自己实现的，没有使用jdk自带的.Dubbo SPI扩展了很多的能力，包括kv配置，按需加载class和instance并且缓存了class和instance不会重复加载，也提供了更多高级的功能，包括扩展点自适应，扩展点自动激活，扩展点自动装配，扩展点包装等。可以说dubbo的spi功能还是非常丰富的，在dubbo源码里有很重要的地位。

## Java SPI

JDK提供的SPI,有一个亿点点的小缺点，就是每次都要遍历所有的实现，返回所有实现类的实例，无法按需加载，无法查找指定的实现类。每次想用的话只能进行遍历。

下面使用示例代码进行演示。

### 物料

一个简单的接口

```java
package com.fc.se.spi;

public interface Animal {
    void cout();
}

```

2个接口的实现类

```java
package com.fc.se.spi;

public class Cat implements Animal {
    @Override
    public void cout() {
        System.out.println("miao miao smh");
    }
}
```



```java
package com.fc.se.spi;

public class Dog implements Comparable<Dog>{
    int size;

    public Dog(int s) {
        size = s;
    }

    public String toString() {
        return size + "";
    }

    @Override
    public int compareTo(Dog o) {
        return size - o.size;
    }
}
```

一个测试类

```
public class SpiMain {
    public static void main(String[] args) {
         ServiceLoader<Animal> animals = ServiceLoader.load(Animal.class);
        for (Animal animal:animals) {
            animal.cout();
            System.out.println(animal.getClass());
            System.out.println(animal.hashCode());
        }
        ServiceLoader<Animal> load = ServiceLoader.load(Animal.class);
        for (Animal animal:load) {
            animal.cout();
            System.out.println(animal.getClass());
            System.out.println(animal.hashCode());
        }
    }
}
```

Resources下的spi配置文件

文件名为接口全限定名com.fc.se.spi.Animal，文件内容为所有的实现类的接口全限定名

```java
com.fc.se.spi.Cat
```

![image-20230712233205259](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202310101352755.png)

### 测试过程

首先执行下main方法

![image-20230712233855450](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202310101351741.png)

根据输出结果可以看到，每次调用ServiceLoader#load方法每次得到的都是新的实例，而且没有任何根据指定字段获取指定实例的方法。如果有的实例在实例化的时候存在比较耗时的操作，或者spi的实现非常多，会非常的消耗资源。

JDK的SPI方法比较简单，不看源码大概猜一下就是去指定目录读取实现类的class，然后反射创建对应的实例。

## Dubbo SPI

Dubbo自己实现了一套spi，解决了jdk spi的一些缺点，并且加上了很多的扩展功能，非常的强大。

读取如下几个目录的配置文件

- META-INF/dubbo/internal/
- META-INF/dubbo/external/
- META-INF/dubbo/
- META-INF/services/

dubbo兼容了jdk spi的目录，并扩展了自身的spi目录，其中internal和external是dubbo自己的扩展文件目录，META-INF/dubbo/这个目录可以放置我们的自定义扩展配置。

### 物料

添加dubbo依赖，我测试用的是dubbo-3.2.0-beta4，spi机制基本上一样，看的时候对应下自己的版本，可能会有差异。

```java
 <dependency>
     <groupId>org.apache.dubbo</groupId>
     <artifactId>dubbo</artifactId>
     <version>3.2.0</version>
</dependency>
```

一个添加了@SPI注解的接口，指定了默认的spi实现,有几个方法添加了@Adaptive注解，用于测试自适应扩展点。

```java
package com.fc.rpc.dubbo;

import org.apache.dubbo.common.URL;
import org.apache.dubbo.common.extension.Adaptive;
import org.apache.dubbo.common.extension.SPI;

@SPI("dog")
public interface SimpleExt {
    /**
     * yell
     *
     * @param url url
     * @param s   s
     * @return s
     */
    @Adaptive({"key1", "key2"})
    String yell(URL url, String s);

    /**
     * echo
     *
     * @param url url
     * @param s   s
     * @return s
     */
    @Adaptive
    String echo(URL url, String s);

    /**
     * no adaptive
     *
     * @param url url
     * @param i   i
     * @return i
     */
    String bang(URL url, int i);
}
```

2个实现了接口的普普通通实现类

```java
package com.fc.rpc.dubbo;

import org.apache.dubbo.common.URL;

public class CatExt implements SimpleExt {
    @Override
    public String yell(URL url, String s) {
        return "yell catExt: " + s;
    }

    @Override
    public String echo(URL url, String s) {
        return "echo catExt: " + s;
    }

    @Override
    public String bang(URL url, int i) {
        return "bang catExt: " + i;
}
```

```java
package com.fc.rpc.dubbo;

import org.apache.dubbo.common.URL;

public class DogExt implements SimpleExt {
    @Override
    public String yell(URL url, String s) {
        return "yell dogExt: " + s;
    }

    @Override
    public String echo(URL url, String s) {
        return "echo dogExt: " + s;
    }

    @Override
    public String bang(URL url, int i) {
        return "bang dogExt: " + i;
    }
}
```

一个普通的测试类

```java
package com.fc.rpc.dubbo;

import org.apache.dubbo.common.URL;
import org.apache.dubbo.common.extension.ExtensionLoader;
import org.apache.dubbo.common.url.component.URLAddress;
import org.apache.dubbo.common.url.component.URLParam;

import java.util.HashMap;

public class DubboSpi {
    public static void main(String[] args) {
        ExtensionLoader<SimpleExt> extensionLoader = ExtensionLoader.getExtensionLoader(SimpleExt.class);
        SimpleExt dog = extensionLoader.getExtension("dog");
        SimpleExt adaptiveExtension = extensionLoader.getAdaptiveExtension();
        dog.yell(null, "dogdog");
        URLAddress urlAddress = new URLAddress("127.0.0.1", 20880);
        URLParam parse = URLParam.parse(new HashMap<>());
        adaptiveExtension.echo(new URL(urlAddress, parse), "dogs");
        SimpleExt dog1 = extensionLoader.getExtension("dog");
        dog1.yell(null, "dog1");
    }
}
```

resources下在META-INF/dubbo下的spi配置文件，文件名为接口的全限定名com.fc.rpc.dubbo.SimpleExt，文件内容是kv配置，指定哪个key对应哪个实现类

```java
dog=com.fc.rpc.dubbo.DogExt
cat=com.fc.rpc.dubbo.CatExt
```



![image-20230714164535594](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202310101351450.png)

基本的准备工作做好了，下面先根据DubboSpi的main方法做一下测试

### ExtensionLoader

每一个接口类对应唯一一个ExtensionLoader实例

```java
public static void main(String[] args) {
        ExtensionLoader<SimpleExt> extensionLoader = ExtensionLoader.getExtensionLoader(SimpleExt.class);
        ExtensionLoader<SimpleExt> extensionLoader1 = ExtensionLoader.getExtensionLoader(SimpleExt.class);
        System.out.println(System.identityHashCode(extensionLoader));
        System.out.println(System.identityHashCode(extensionLoader1));  
}
```

控制台输出

```
1208203046
1208203046
```

![image-20230714153000173](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202310101351372.png)

根据debug和控制台输出，可以看出多次获取的是同一个extensinonLoader实例

### Extension

配置文件里配置了如下2个kv

```
dog=com.fc.rpc.dubbo.DogExt
cat=com.fc.rpc.dubbo.CatExt
```

我们获取spi实现的时候，可以指定key去获取

获取extension有如下几个api

- getExtension(String name)
- getExtension(String name, boolean wrap)
- getDefaultExtension()

第一个是获取指定key的扩展实现，示例中是dog

第二个是获取包装类，这个是dubbo中的自动包装的实现，使用warp实现了类似aop的功能，包装类持有spi扩展的实例

第三个是获取默认的扩展类，在spi注解上标记了默认扩展实现，类似@SPI("dog")，默认的扩展实现就是dog

接下来试验下这3个方法

```java
public static void main(String[] args) {
        ExtensionLoader<SimpleExt> extensionLoader = ExtensionLoader.getExtensionLoader(SimpleExt.class);

        SimpleExt defaultExtension = extensionLoader.getDefaultExtension();
        SimpleExt dog = extensionLoader.getExtension("dog");
        SimpleExt dog2 = extensionLoader.getExtension("dog", true);
        SimpleExt cat = extensionLoader.getExtension("cat");
}
```

![image-20230714164705658](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202310101351921.png)

根据结果可以看到defaultExtension/dog/dog2都是同一个实例，类型为com.fc.rpc.dubbo.DogExt

另一个是类型为com.fc.rpc.dubbo.CatExt的实例，同一个key只会生成唯一一个对应实现类的实例。

### Adaptive Extension

上面一小节讲的是普通的SPI的扩展实现，得到的就是对应的接口实现类的实例，dubbo还提供了另外一种扩展实现，叫做自适应扩展。

先看一下官网的描述: 

> 先说明下自适应扩展类的使用场景。比如我们有需求，在调用某一个方法时，基于参数选择调用到不同的实现类。和工厂方法有些类似，基于不同的参数，构造出不同的实例对象。 在 Dubbo 中实现的思路和这个差不多，不过 Dubbo 的实现更加灵活，它的实现和策略模式有些类似。每一种扩展类相当于一种策略，基于 URL 消息总线，将参数传递给 ExtensionLoader，通过 ExtensionLoader 基于参数加载对应的扩展类，实现运行时动态调用到目标实例上。

实际上就是根据url的参数动态选择具体的实现类

使用自适应扩展，需要在接口或者方法上添加@Adaptive注解

看下示例

```java
import org.apache.dubbo.common.URL;
import org.apache.dubbo.common.extension.Adaptive;
import org.apache.dubbo.common.extension.SPI;

/**
 * simle ext
 *
 * @author since
 * @date 2023-07-10 10:03
 **/
@SPI("dog")
public interface SimpleExt {
    /**
     * yell
     *
     * @param url url
     * @param s   s
     * @return s
     */
    @Adaptive({"key1", "key2"})
    String yell(URL url, String s);

    /**
     * echo
     *
     * @param url url
     * @param s   s
     * @return s
     */
    @Adaptive
    String echo(URL url, String s);

    /**
     * no adaptive
     *
     * @param url url
     * @param i   i
     * @return i
     */
    String bang(URL url, int i);

}
```

有2种用法，一种是取默认值的，默认的@Adaptive取的是spi的默认扩展实现，本例是dog，一种是根据key的值来获取对应扩展的，根据url中取到的extName获取对应的扩展实现。

获取adaptive extension使用的是`org.apache.dubbo.common.extension.ExtensionLoader#getAdaptiveExtension`，这个方法的得到的是一个interface$Adaptive的class实例,本例中的adaptive extension的class的`com.fc.rpc.dubbo.SimpleExt$Adaptive`

看下示例代码

```java
 public static void main(String[] args) {
        ExtensionLoader<SimpleExt> extensionLoader = ExtensionLoader.getExtensionLoader(SimpleExt.class);

        SimpleExt adaptiveExtension = extensionLoader.getAdaptiveExtension();
        dog.yell(null, "dogdog");
        URLAddress urlAddress = new URLAddress("127.0.0.1", 20880);
        URLParam parse = URLParam.parse(new HashMap<>());
        adaptiveExtension.echo(new URL(urlAddress, parse), "dogs");
        SimpleExt dog1 = extensionLoader.getExtension("dog");
        dog1.yell(null, "dog1");
}
```

![image-20230714172924754](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202310101351124.png)

echo方法使用的是默认的参数，所以会执行到spi的默认扩展dog，可以看一下下一步

![image-20230714173117260](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202310101351332.png)

在来看一下SimpleExt$Adaptive的class文件，这个是通过javaassist直接拼接出来的class文件，控制台会输出这个class文件

![image-20230714173302139](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202310101351832.png)

完整的class文件如下

```java
package com.fc.rpc.dubbo;

import org.apache.dubbo.rpc.model.ScopeModel;
import org.apache.dubbo.rpc.model.ScopeModelUtil;

/**
 * @author
 * @date 2023-07-10 13:11
 */
public class SimpleExt$Adaptive implements com.fc.rpc.dubbo.SimpleExt {
    public java.lang.String yell(org.apache.dubbo.common.URL arg0, java.lang.String arg1) {
        if (arg0 == null) {
            throw new IllegalArgumentException("url == null");
        }
        org.apache.dubbo.common.URL url = arg0;
        String extName = url.getParameter("key1", url.getParameter("key2", "dog"));
        if (extName == null) {
            throw new IllegalStateException("Failed to get extension (com.fc.rpc.dubbo.SimpleExt) name from url (" + url.toString() + ") use keys([key1, key2])");
        }
        ScopeModel scopeModel = ScopeModelUtil.getOrDefault(url.getScopeModel(), com.fc.rpc.dubbo.SimpleExt.class);
        com.fc.rpc.dubbo.SimpleExt extension = (com.fc.rpc.dubbo.SimpleExt) scopeModel.getExtensionLoader(com.fc.rpc.dubbo.SimpleExt.class).getExtension(extName);
        return extension.yell(arg0, arg1);
    }

    public java.lang.String echo(org.apache.dubbo.common.URL arg0, java.lang.String arg1) {
        if (arg0 == null) {
            throw new IllegalArgumentException("url == null");
        }
        org.apache.dubbo.common.URL url = arg0;
        String extName = url.getParameter("simple.ext", "dog");
        if (extName == null) {
            throw new IllegalStateException("Failed to get extension (com.fc.rpc.dubbo.SimpleExt) name from url (" + url.toString() + ") use keys([simple.ext])");
        }
        ScopeModel scopeModel = ScopeModelUtil.getOrDefault(url.getScopeModel(), com.fc.rpc.dubbo.SimpleExt.class);
        com.fc.rpc.dubbo.SimpleExt extension = (com.fc.rpc.dubbo.SimpleExt) scopeModel.getExtensionLoader(com.fc.rpc.dubbo.SimpleExt.class).getExtension(extName);
        return extension.echo(arg0, arg1);
    }

    public java.lang.String bang(org.apache.dubbo.common.URL arg0, int arg1) {
        throw new UnsupportedOperationException("The method public abstract java.lang.String com.fc.rpc.dubbo.SimpleExt.bang(org.apache.dubbo.common.URL,int) of interface com.fc.rpc.dubbo.SimpleExt is not adaptive method!");
    }
}
```

看下生成的class，接口里一共定义了3个方法，2个方法打上了@Adaptive注解，其中由于bang()没有打注解，所以通过代理类访问bang访问会直接报错

echo方法由于使用的是默认的值，不需要根据url里的取值去动态获取ext，所以是会走到默认的扩展点dog里，这个上面debug已经说明了。

具体获取ext是这行代码,@Adaptive没有设置具体的参数，会将接口类拆成simple.ext这样去获取extName,取不到的话会取默认值dog

```
String extName = url.getParameter("simple.ext", "dog");
```

yell方法由于配置了参数，所以实际获取extName的行为是这样

```
String extName = url.getParameter("key1", url.getParameter("key2", "dog"));
```

根据extName获取最终的扩展实现

自适应扩展先写到这，后面的案例再补充

### warpper

dubbo还有一种warper的用法，就是有一个spi的扩展类，有一个参数的构造函数，然后类型是扩展类的类型，就会实现warpper的效果，类似aop的功能，这么说可能有点晦涩，写代码看下效果吧

#### 物料

配置文件，照例我们还是先写配置文件，再META-INF/dubbo下新建一个文件命名为`com.fc.rpc.dubbo.WrapperExt`的配置文件，这个名称就是我们测试wrapper的接口全路径

配置文件里定义kv，前2行是目标实现，类似于我们上个例子的dogExt和catExt，后2行是包装实现，调用一个扩展，会按顺序执行tortoiseWrapper和rabbitWrapper，执行完扩展之后，会执行目标实现，比如tortoise和rabbit。

```
tortoise=com.fc.rpc.dubbo.TortoiseExt
rabbit=com.fc.rpc.dubbo.RabbitExt

tortoiseWrapper=com.fc.rpc.dubbo.TortoiseWrapperExt
rabbitWrapper=com.fc.rpc.dubbo.RabbitWrapperExt
```

SPI接口类

定义了一个WrapperExt接口，打上@SPI注解，这个和上面的SimpleExt一样

```java
package com.fc.rpc.dubbo;

import org.apache.dubbo.common.extension.SPI;

/**
 * warper ext
 *
 * @author since
 * @date 2023-08-03 08:37
 */
@SPI
public interface WrapperExt {
    /**
     * wrapper
     *
     * @param wrapper wrapper
     */
    void wrapper(String wrapper);
}
```

原始实现

有2个原始实现类，实现了WrapperExt,这2个和上面的DogExt和CatExt含义一样

```java
package com.fc.rpc.dubbo;

/**
 * @author
 * @date 2023-08-03 08:48
 */
public class TortoiseExt implements WrapperExt {
    @Override
    public void wrapper(String wrapper) {
        System.out.println("tortoise self: " + wrapper);
    }
}
```

```java
package com.fc.rpc.dubbo;

/**
 * @author
 * @date 2023-08-03 08:48
 */
public class RabbitExt implements WrapperExt {
    @Override
    public void wrapper(String wrapper) {
        System.out.println("rabbit self: {}");
    }
}
```

有2个包装实现类，这个和原始实现类不同的是里面有个类型为WrapperExt的成员变量，有一个含参构造函数，用于初始化成员变量

```java
package com.fc.rpc.dubbo;

/**
 * @author since
 * @date 2023-08-03 08:55
 */
public class TortoiseWrapperExt implements WrapperExt {
    private final WrapperExt wrapperExt;

    public TortoiseWrapperExt(WrapperExt wrapperExt) {
        this.wrapperExt = wrapperExt;
    }

    @Override
    public void wrapper(String wrapper) {
        System.out.println("tortoise wrapper: " + wrapper);
        wrapperExt.wrapper(wrapper);
    }
}
```

```java
package com.fc.rpc.dubbo;

/**
 * @author since
 * @date 2023-08-03 08:55
 */
public class RabbitWrapperExt implements WrapperExt {
    private final WrapperExt wrapperExt;

    public RabbitWrapperExt(WrapperExt wrapperExt) {
        this.wrapperExt = wrapperExt;
    }

    @Override
    public void wrapper(String wrapper) {
        System.out.println("rabbit wrapper: "+wrapper);
        wrapperExt.wrapper(wrapper);
    }
}

```

一个测试类，用于执行测试方法

```java
	package com.fc.rpc.dubbo;

import org.apache.dubbo.common.extension.ExtensionLoader;

/**
 * spi wrapper
 * @author since
 * @date 2023-08-05 08:50
 */
public class SpiWrapper {
    public static void main(String[] args) {
        ExtensionLoader<WrapperExt> extensionLoader = ExtensionLoader.getExtensionLoader(WrapperExt.class);
        WrapperExt tortoise = extensionLoader.getExtension("tortoise");
        tortoise.wrapper("abandon");
        System.out.println("###################");
        WrapperExt rabbit = extensionLoader.getExtension("rabbit");
        rabbit.wrapper("give up");
        
    }


}
```



#### 测试

先简单跑一下main函数,执行结果如下

```
rabbit wrapper: abandon
tortoise wrapper: abandon
tortoise self: abandon
###################
rabbit wrapper: give up
tortoise wrapper: give up
rabbit self: give up
```

可以看到先是按顺序执行了2个wrapper包装类，然后在执行到了自身实现类。

这个就是静态代理的用法，通过有参构造函数，反射构造包装类，先执行包装类，最终调用到的就是目标类。

#### 代码实现

通过org.apache.dubbo.common.extension.ExtensionLoader#getExtension(java.lang.String)进行调用，wrapper默认是true，所以获取到的构造参数都是会进行包装判断的，具体看下面的代码

```java
    private T createExtension(String name, boolean wrap) {
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null || unacceptableExceptions.contains(name)) {
            throw findException(name);
        }
        try {
            T instance = (T) extensionInstances.get(clazz);
            if (instance == null) {
                extensionInstances.putIfAbsent(clazz, createExtensionInstance(clazz));
                instance = (T) extensionInstances.get(clazz);
                instance = postProcessBeforeInitialization(instance, name);
                injectExtension(instance);
                instance = postProcessAfterInitialization(instance, name);
            }

            //看这里
            if (wrap) {
                List<Class<?>> wrapperClassesList = new ArrayList<>();
                if (cachedWrapperClasses != null) {
                    wrapperClassesList.addAll(cachedWrapperClasses);
                    wrapperClassesList.sort(WrapperComparator.COMPARATOR);
                    Collections.reverse(wrapperClassesList);
                }

                if (CollectionUtils.isNotEmpty(wrapperClassesList)) {
                    for (Class<?> wrapperClass : wrapperClassesList) {
                        Wrapper wrapper = wrapperClass.getAnnotation(Wrapper.class);
                        boolean match = (wrapper == null) || ((ArrayUtils.isEmpty(
                            wrapper.matches()) || ArrayUtils.contains(wrapper.matches(),
                            name)) && !ArrayUtils.contains(wrapper.mismatches(), name));
                        if (match) {
                            instance = injectExtension(
                                (T) wrapperClass.getConstructor(type).newInstance(instance));
                            instance = postProcessAfterInitialization(instance, name);
                        }
                    }
                }
            }

            // Warning: After an instance of Lifecycle is wrapped by cachedWrapperClasses, it may not still be Lifecycle instance, this application may not invoke the lifecycle.initialize hook.
            initExtension(instance);
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException(
                "Extension instance (name: " + name + ", class: " + type + ") couldn't be instantiated: " + t.getMessage(),
                t);
        }
    }
```

重点看if(wrap)这里，如果需要包装的话，会获取到对应的包装类的class列表，如果有对应的包装类class，则判断是否可以根据构造器创建对应类型的实例。

这一段代码会生成包装类，并将target设置进成员变量里

```java
if (match) {
     instance = injectExtension(
                                (T) wrapperClass.getConstructor(type).newInstance(instance)
                               );
     nstance = postProcessAfterInitialization(instance, name);
}
```



来看一下debug过程，以tortoise为例，断点断在tortoise实例上

![image-20230805095043419](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202310101350251.png)

可以看到tortoise实例的类型的RabbitWrapperExt，持有了一个TortoiseWrapperExt类型的成员变量，而TortoiseWrapperExt最终持有了我们的目标TortoiseExt类型的成员变量.



### IOC

dubbo的spi实现了ioc的功能，具体来说如果有个成员变量里实现了set方法，就会尝试为这个成员变量注入一个bean，以达到自动注入的目的。

本次没有提供spring环境，所以使用adaptiveExtension获取实例进行注入

#### 物料

配置文件，在META-INF/dubbo/下新增一个配置文件`com.fc.rpc.dubbo.IocExt`,文件内容如下

```
audi=com.fc.rpc.dubbo.AudiIocExt
```

以及之前测试用过的spi配额`com.fc.rpc.dubbo.SimpleExt`，拿过来用一下

```
dog=com.fc.rpc.dubbo.DogExt
cat=com.fc.rpc.dubbo.CatExt
```



SPI接口类

```java
package com.fc.rpc.dubbo;

import org.apache.dubbo.common.extension.SPI;

/**
 * ioc ext
 *
 * @author since
 * @date 2023-08-05 10:52
 **/
@SPI
public interface IocExt {
    /**
     * 汽车的价格
     */
    void carPrice();
}
```

SPI接口实现类

```java
package com.fc.rpc.dubbo;

import org.apache.dubbo.common.URL;
import org.apache.dubbo.common.url.component.URLAddress;
import org.apache.dubbo.common.url.component.URLParam;

import java.util.HashMap;

/**
 * audi ioc ext
 *
 * @author since
 * @date 2023-08-05 10:53
 */
public class AudiIocExt implements IocExt {
    private SimpleExt simpleExt;

    public void setSimpleExt(SimpleExt simpleExt) {
        this.simpleExt = simpleExt;
    }

    @Override
    public void carPrice() {
        System.out.println("audi's price is 299999");
        URLAddress urlAddress = new URLAddress("127.0.0.1", 20880);
        URLParam parse = URLParam.parse(new HashMap<>());
        String audi = simpleExt.echo(new URL(urlAddress, parse), "audi");
        System.out.println("ioc execute: " + audi);
    }
}
```

SPI实现里定义了一个成员变量SimpleExt，这个是我们本篇文章第一个测试类

一个测试类

```java
package com.fc.rpc.dubbo;

import org.apache.dubbo.common.extension.ExtensionLoader;

/**
 * @author
 * @date 2023-08-05 10:58
 */
public class SpiIoc {
    public static void main(String[] args) {
        ExtensionLoader<IocExt> extensionLoader = ExtensionLoader.getExtensionLoader(IocExt.class);
        IocExt audi = extensionLoader.getExtension("audi");
        audi.carPrice();
    }
}
```

#### 测试

看下执行结果,可以看到确实执行到了成员变量的方法，成员变量确实注入了一个实例

![image-20230808225444873](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202310101350254.png)

看下debug过程,simpleExt的类型是动态代理生成的一个类的实例，这个之前分析自适应扩展的时候件过，最终会执行到SimpleExt默认的spi实现那。

![image-20230808225724407](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202310101350550.png)

#### 代码实现

实现这个主要是在获取扩展时，做了一个依赖注入的功能，一共有3种类型的injector，本次测试用例用的是AdaptiveExtensionInjector

看下具体的代码

`org.apache.dubbo.common.extension.ExtensionLoader#createExtension`这里是创建扩展入口

```java
 private T createExtension(String name, boolean wrap) {
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null || unacceptableExceptions.contains(name)) {
            throw findException(name);
        }
        try {
            T instance = (T) extensionInstances.get(clazz);
            if (instance == null) {
                extensionInstances.putIfAbsent(clazz, createExtensionInstance(clazz));
                instance = (T) extensionInstances.get(clazz);
                instance = postProcessBeforeInitialization(instance, name);
                //主要看这里
                injectExtension(instance);
                instance = postProcessAfterInitialization(instance, name);
            }

            if (wrap) {
                List<Class<?>> wrapperClassesList = new ArrayList<>();
                if (cachedWrapperClasses != null) {
                    wrapperClassesList.addAll(cachedWrapperClasses);
                    wrapperClassesList.sort(WrapperComparator.COMPARATOR);
                    Collections.reverse(wrapperClassesList);
                }

                if (CollectionUtils.isNotEmpty(wrapperClassesList)) {
                    for (Class<?> wrapperClass : wrapperClassesList) {
                        Wrapper wrapper = wrapperClass.getAnnotation(Wrapper.class);
                        boolean match = (wrapper == null) || ((ArrayUtils.isEmpty(
                            wrapper.matches()) || ArrayUtils.contains(wrapper.matches(),
                            name)) && !ArrayUtils.contains(wrapper.mismatches(), name));
                        if (match) {
                            instance = injectExtension(
                                (T) wrapperClass.getConstructor(type).newInstance(instance));
                            instance = postProcessAfterInitialization(instance, name);
                        }
                    }
                }
            }

            // Warning: After an instance of Lifecycle is wrapped by cachedWrapperClasses, it may not still be Lifecycle instance, this application may not invoke the lifecycle.initialize hook.
            initExtension(instance);
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException(
                "Extension instance (name: " + name + ", class: " + type + ") couldn't be instantiated: " + t.getMessage(),
                t);
        }
    }
```

`org.apache.dubbo.common.extension.ExtensionLoader#injectExtension`这里是依赖注入实现,使用`org.apache.dubbo.common.extension.ExtensionInjector#getInstance`获取需要注入的实例

```java
private T injectExtension(T instance) {
        if (injector == null) {
            return instance;
        }

        try {
            for (Method method : instance.getClass().getMethods()) {
                if (!isSetter(method)) {
                    continue;
                }
                /**
                 * Check {@link DisableInject} to see if we need auto-injection for this property
                 */
                if (method.isAnnotationPresent(DisableInject.class)) {
                    continue;
                }

                // When spiXXX implements ScopeModelAware, ExtensionAccessorAware,
                // the setXXX of ScopeModelAware and ExtensionAccessorAware does not need to be injected
                if (method.getDeclaringClass() == ScopeModelAware.class) {
                    continue;
                }
                if (instance instanceof ScopeModelAware || instance instanceof ExtensionAccessorAware) {
                    if (ignoredInjectMethodsDesc.contains(ReflectUtils.getDesc(method))) {
                        continue;
                    }
                }

                Class<?> pt = method.getParameterTypes()[0];
                if (ReflectUtils.isPrimitives(pt)) {
                    continue;
                }

                try {
                    String property = getSetterProperty(method);
                    //主要看这里
                    Object object = injector.getInstance(pt, property);
                    if (object != null) {
                        method.invoke(instance, object);
                    }
                } catch (Exception e) {
                    logger.error(COMMON_ERROR_LOAD_EXTENSION, "", "",
                        "Failed to inject via method " + method.getName() + " of interface " + type.getName() + ": " + e.getMessage(),
                        e);
                }
            }
        } catch (Exception e) {
            logger.error(COMMON_ERROR_LOAD_EXTENSION, "", "", e.getMessage(), e);
        }
        return instance;
    }
```

使用反射调用set方法将获取到实例设置到对象的成员变量里

看下debug过程，`org.apache.dubbo.common.extension.ExtensionLoader#injectExtension`执行到获取instance时，injector类型是AdaptiveExtensionInjector,持有一个injector list，会遍历list获取实例

![image-20230808230814980](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202310101350398.png)



遍历3种类型的injector获取要注入的实例，找到一个就返回

![image-20230808231002006](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202310101350803.png)



## 其他

断断续续终于写完了，第一次写这种分析源码的文章，7月12号动笔的，8月12号才写完，真的墨迹。

中间也看了其他博客的文章，自己也动手Debug了代码，对dubbo的spi有了比较深入的了解了，也算学到了不少知识。

之前一阵子对dubbo的源码比较好奇，陆续看了dubbo consumer的超时重试策略，spi设计等模块，后续如果有机会的话会陆续写出来。

## 参考链接

- [dubbo spi官网介绍](https://cn.dubbo.apache.org/zh-cn/overview/mannual/java-sdk/reference-manual/spi/description/dubbo-spi/#1-dubbo-spi-%E6%89%A9%E5%B1%95%E7%AE%80%E4%BB%8B)

- [vivo互联网技术dubbo spi介绍](https://segmentfault.com/a/1190000040208433)
