---
draft: false
date: 2024-09-11T15:40:13+08:00
title: "如何查看jvm运行时生成的动态代理class文件"
slug: "view-jvm-dynamic-class-detail" 
tags: ["java"]
categories: []
authors: ["since"]
description: ""
disableShare: true # 底部不显示分享栏
---



## 前言

最近看dubbo的时候，看到了讲spi的部分，其中提到了@Adaptive注解生成了代理类

![image-20240911160843943](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202409111608009.png)





我后面去debug了一下，打了断点，发现在运行时，只是显示了class的名称，由于**RegistryFactory$Adaptive** 是动态生成的，所以没有class文件结构可以看，而文章里贴了class文件。



所以比较好奇，如何拿到class文件。



经过一番搜索验证之后，有了如下的文章



## Java生成代理类的方式

首先呢, java生成代理类有如下几种方式。

在了解了动态代理的生成方式之后，可以分情况讨论如何拿到代理类的class文件



### JDK动态代理

这个是java自身支持的代理方式，自带的，无需引入额外组件，但是只能代理接口。

因为生成的代理类需要继承Proxy类，java是单继承的，所以只能代理接口。



### CGLIB动态代理

cglib可以为class创建代理类，类不必是接口，采用的是继承委托类的方式，因此不能代理final类，只能代理所有非final的public/protected类型的方法定义。



cglib是采用的asm字节码拼接技术，继承一个类，子类对方法进行拦截，织入横切逻辑。



### Javassist

使用javassist可以直接操纵字节码，生成代理类。

dubbo的代理类就是使用javassist拼接出来的



### ASM

asm相比javassist，更加的细粒度，且实现更加的复杂。





## Dubbo相关代理类的class文件

具体到dubbo的话，@Adaptive会生成相关的代理类，Dubbo的代理类是用javaasist生成的。



再延伸到调用过程，consumer调provider，生成的代理类是jdk动态代理

provider端调用过程中，会有一个warpper代理类，生成的代理类是使用javassist生成的，然后通过Warpper0.invokeMethod()根据具体的调用信息调用到具体的实现类。



**本次使用的dubbo版本是3.2.0**



去年写了一篇关于[dubbo spi](https://blog.thend03.com/posts/dubbo-spi/)的文章，感兴趣的可以看一下，介绍了spi的相关实现方式，以及如何使用



具体到Adaptive代理类，使用如下的测试代码(完整的过程可以参考上面的文章地址)

```java
public class DubboSpi {
    public static void main(String[] args) {
        ExtensionLoader<SimpleExt> extensionLoader = ExtensionLoader.getExtensionLoader(SimpleExt.class);

//        SimpleExt defaultExtension = extensionLoader.getDefaultExtension();
//        SimpleExt dog = extensionLoader.getExtension("dog");
//        SimpleExt dog2 = extensionLoader.getExtension("dog", true);
//        SimpleExt cat = extensionLoader.getExtension("cat");
//        dog.yell(null, "dogdog");
//        SimpleExt dog1 = extensionLoader.getExtension("dog");
//        dog1.yell(null, "dog1");

        SimpleExt adaptiveExtension = extensionLoader.getAdaptiveExtension();

        URLAddress urlAddress = new URLAddress("127.0.0.1", 20880);
        URLParam parse = URLParam.parse(new HashMap<>());
        adaptiveExtension.echo(new URL(urlAddress, parse), "dogs");
        adaptiveExtension.bang(new URL(urlAddress, parse),0);

    }
}
```



定位到如下生成代理类的方法,代码在`org.apache.dubbo.common.extension.ExtensionLoader#createAdaptiveExtensionClass`

```java
 private Class<?> createAdaptiveExtensionClass() {
        // Adaptive Classes' ClassLoader should be the same with Real SPI interface classes' ClassLoader
        ClassLoader classLoader = type.getClassLoader();
        try {
            if (NativeUtils.isNative()) {
                return classLoader.loadClass(type.getName() + "$Adaptive");
            }
        } catch (Throwable ignore) {

        }
        //这里生成class文件
        String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
        org.apache.dubbo.common.compiler.Compiler compiler = extensionDirector.getExtensionLoader(
            org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
        return compiler.compile(type, code, classLoader);
    }
```



我们debug看一下生成的code, 看类名，是SimpleExt$Adaptive, 通过手动拼接的方式，生成了adaptive代理类。



![image-20240918152509863](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202409181525928.png)





调用AdaptiveCompiler，最终compileer的类型为`JavassistCompiler`，即通过javassist生成的adaptive代理类

![image-20240918152734919](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202409181527968.png)





那么回归到我一开始的问题，如何查看动态生成的class的内容，在`String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();`这一行通过debug就可以拿到具体的class内容了



生成的代理类如下所示, 对接口的方法进行了拦截

```java
package com.fc.rpc.dubbo;
import org.apache.dubbo.rpc.model.ScopeModel;
import org.apache.dubbo.rpc.model.ScopeModelUtil;
public class SimpleExt$Adaptive implements com.fc.rpc.dubbo.SimpleExt {
  public java.lang.String yell(org.apache.dubbo.common.URL arg0, java.lang.String arg1) {
    if (arg0 == null) throw new IllegalArgumentException("url == null");
    org.apache.dubbo.common.URL url = arg0;
    String extName = url.getParameter("key1", url.getParameter("key2", "dog"));
    if (extName == null) throw new IllegalStateException("Failed to get extension (com.fc.rpc.dubbo.SimpleExt) name from url (" + url.toString() + ") use keys([key1, key2])");
    ScopeModel scopeModel = ScopeModelUtil.getOrDefault(url.getScopeModel(), com.fc.rpc.dubbo.SimpleExt.class);
    com.fc.rpc.dubbo.SimpleExt extension = (com.fc.rpc.dubbo.SimpleExt) scopeModel.getExtensionLoader(com.fc.rpc.dubbo.SimpleExt.class).getExtension(extName);
    return extension.yell(arg0, arg1);
  }
  
  
  public java.lang.String echo(org.apache.dubbo.common.URL arg0, java.lang.String arg1) {
    if (arg0 == null) throw new IllegalArgumentException("url == null");
    org.apache.dubbo.common.URL url = arg0;
    String extName = url.getParameter("simple.ext", "dog");
    if (extName == null) throw new IllegalStateException("Failed to get extension (com.fc.rpc.dubbo.SimpleExt) name from url (" + url.toString() + ") use keys([simple.ext])");
    ScopeModel scopeModel = ScopeModelUtil.getOrDefault(url.getScopeModel(), com.fc.rpc.dubbo.SimpleExt.class);
    com.fc.rpc.dubbo.SimpleExt extension = (com.fc.rpc.dubbo.SimpleExt) scopeModel.getExtensionLoader(com.fc.rpc.dubbo.SimpleExt.class).getExtension(extName);
    return extension.echo(arg0, arg1);
  }
  
  
  public java.lang.String bang(org.apache.dubbo.common.URL arg0, int arg1) {
    throw new UnsupportedOperationException("The method public abstract java.lang.String com.fc.rpc.dubbo.SimpleExt.bang(org.apache.dubbo.common.URL,int) of interface com.fc.rpc.dubbo.SimpleExt is not adaptive method!");
  }
}
```



另外debug级别的话，生成的class会输出到日志

![image-20240918155018371](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202409181550439.png)



## JDK动态代理的class文件

回归到jdk的动态代理，如何查看jdk动态代理生成的class文件



这里主要讲一下如何查看动态代理的class文件，动态代理的原理啥的不深入。



### 设置参数

在调用之前，添加`System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles","true");`

这个只适用于jdk动态代理





准备一下测试类

这是调用入口

```java
public class DynamicProxyMain {
    public static void main(String[] args) {
        //动态代理时生成class文件
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles","true");
        DynamicProxy dynamicProxy = new DynamicProxy(new HumenImpl());
        Humen humenProxy = dynamicProxy.getProxy();
        humenProxy.eat("rice");
    }
}
```



DynamicProxy实现了InvocationHandler，用于生成代理类

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;


public class DynamicProxy implements InvocationHandler {
    private Object target;

    public DynamicProxy(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        this.before();
        Object result = method.invoke(target, args);
        this.after();
        return result;
    }

    private void before() {
        System.out.println("吃东西之前先热身");
    }

    private void after() {
        System.out.println("吃完饭休息一下");
    }

    public <T> T getProxy() {
        return (T) Proxy.newProxyInstance(target.getClass().getClassLoader(),target.getClass().getInterfaces(),this);
    }


}
```



测试用的接口

```java
/**
 * 人类迷惑行为大赏
 **/
public interface Humen {
    /**
     * 吃东西
     *
     * @param food 食物
    */
    void eat(String food);
}
```

测试接口的实现类

```java
/**
 * 人类迷惑行为大赏具体案例
 **/
public class HumenImpl implements Humen {
    @Override
    public void eat(String food) {
        System.out.println("人类开始行为艺术表演：吃食物 "+food);
    }
}

```



那么主要看一下这个配置`System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles","true");`这个配置是用于保存生成的class的。



缺点是这个参数只适用于jdk动态代理





加上这个参数运行一下，会发现代理类保存到了项目路径(classpath)下

![image-20240918160242965](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202409181602018.png)



看一下$Proxy0的详细内容, 代理类继承了Proxy, 所以jdk动态代理只能代理接口，因为Java是单继承的

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//

package com.sun.proxy;

import com.fc.se.proxy.Humen;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0 extends Proxy implements Humen {
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void eat(String var1) throws  {
        try {
            super.h.invoke(this, m3, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("com.fc.se.proxy.Humen").getMethod("eat", Class.forName("java.lang.String"));
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```



构造函数传入了InvocationHandler，赋值给了成员变量h，所以方法的调用都是走的InvocationHandler。

本次示例中为DynamicProxy

```
public $Proxy0(InvocationHandler var1) throws  {
    super(var1);
}
```



在静态代码块中生成了对应的方法，类通用的hashCode()、toString()、equals(), 以及需要代理的方法eat()。



$Proxy0调用eat方法，然后执行到了`com.fc.se.proxy.DynamicProxy#invoke`, 

`method.invoke(target, args)`调用具体的实现类执行方法。





### 输出到文件

上一步中我们知道了生成的代理类的类名是$Proxy0, debug的时候也能看到类名，我们可以调用方法将这个类打印到文件里。

![image-20240918162848155](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202409181628208.png)



更新后的测试类入口如下

```java
import sun.misc.ProxyGenerator;

import java.io.FileOutputStream;
import java.io.IOException;

public class DynamicProxyMain {
    public static void main(String[] args) {
        //动态代理时生成class文件
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles","true");
        DynamicProxy dynamicProxy = new DynamicProxy(new HumenImpl());
        Humen humenProxy = dynamicProxy.getProxy();
        humenProxy.eat("rice");

        //保存$Proxy0到文件
        saveProxyFile();

    }

    private static void saveProxyFile() {
        FileOutputStream out = null;
        try {
            byte[] classFile = ProxyGenerator.generateProxyClass("$Proxy0", HumenImpl.class.getInterfaces());
            out = new FileOutputStream("/Users/since/git/vamei/out/class/" + "$Proxy0.class");
            out.write(classFile);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                if (out != null) {
                    out.flush();
                    out.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

}

```

执行之后可以发现已经生成了$Proxy0.class

![image-20240918163812572](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202409181638627.png)



所以知道类名，就可以以这种方式输出到文件里



### hsdb

另一种是hsdb



我这边使用hsdb失败了，具体是命令行执行的, attach process失败了，搜了一下，感觉执行起来挺麻烦的，就没深究。感兴趣的可以自行试下



打开hsdb有2种方式

一种是执行$JAVA_HOME/lib/sa-jdi.jar，命令行执行`java -classpath sa-jdi.jar "sun.jvm.hotspot.HSDB"`

另一种是使用jhsdb hsdb命令打开ui, 有的jdk版本没有sa-jdi.jar，只能使用$JAVA_HOME/bin/jhsdb命令





![image-20240918171627367](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202409181716411.png)





## CGLIB代理的class文件

另一种常用的动态生成代理类的方式是使用cglib库，底层是基于ASM的

cglib代理输出class文件，需要指定参数

```java
// 指定 CGLIB 将动态生成的代理类保存至指定的磁盘路径下
System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "/Users/since/git/vamei/out/cglib");
```

我们准备一个测试方法来测试一下，cglib依赖如下

```xml
<dependency>
   <groupId>cglib</groupId>
   <artifactId>cglib</artifactId>
   <version>3.3.0</version>
</dependency>
```



一个main函数用于执行，生成代理类，Enhancer对象的superClass设置为CglibBaseBean, CglibBaseBean为要代理的目标对象，然后使用`enhancer.create()`创建代理对象

```java
public class CglibProxy {
    public static void main(String[] args) {
        // 指定 CGLIB 将动态生成的代理类保存至指定的磁盘路径下
        System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "/Users/since/git/vamei/out/cglib");
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(CglibBaseBean.class);
        enhancer.setCallback(new CglibCustomizedMethodInterceptor());
        CglibBaseBean baseBean = (CglibBaseBean) enhancer.create();
        baseBean.say();
    }
}
```



需要一个实现了MethodInterceptor的类，这个和`InvocationHandler`比较相似。

在MethodInterceptor里添加拦截的逻辑

```java
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

/**
 * cglib CustomizedMethodInterceptor
 *
 * @author since
 * @date 2024-09-19 13:25
 */
public class CglibCustomizedMethodInterceptor implements MethodInterceptor {
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("invoke " + method.getName() + " before ! ");
        Object result = methodProxy.invokeSuper(o, objects);
        System.out.println("invoke " + method.getName() + " after ! ");
        return result;
    }
}

```



代理类持有了CglibCustomizedMethodInterceptor，真正执行的时候是通过CglibCustomizedMethodInterceptor的intercept方法执行到了被代理的目标类的。

![image-20240919141408249](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202409191414295.png)



然后看一下class文件输出结果，生成了3个代理类

![image-20240919142222166](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202409191422204.png)



CglibBaseBean$$EnhancerByCGLIB$$bf21c123.class是生成的代理类，其他2个是索引类



代理类class具体内容如下

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//

package com.fc.se.proxy.cglib;

import java.lang.reflect.Method;
import net.sf.cglib.core.ReflectUtils;
import net.sf.cglib.core.Signature;
import net.sf.cglib.proxy.Callback;
import net.sf.cglib.proxy.Factory;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

public class CglibBaseBean$$EnhancerByCGLIB$$bf21c123 extends CglibBaseBean implements Factory {
    private boolean CGLIB$BOUND;
    public static Object CGLIB$FACTORY_DATA;
    private static final ThreadLocal CGLIB$THREAD_CALLBACKS;
    private static final Callback[] CGLIB$STATIC_CALLBACKS;
    private MethodInterceptor CGLIB$CALLBACK_0;
    private static Object CGLIB$CALLBACK_FILTER;
    private static final Method CGLIB$say$0$Method;
    private static final MethodProxy CGLIB$say$0$Proxy;
    private static final Object[] CGLIB$emptyArgs;
    private static final Method CGLIB$equals$1$Method;
    private static final MethodProxy CGLIB$equals$1$Proxy;
    private static final Method CGLIB$toString$2$Method;
    private static final MethodProxy CGLIB$toString$2$Proxy;
    private static final Method CGLIB$hashCode$3$Method;
    private static final MethodProxy CGLIB$hashCode$3$Proxy;
    private static final Method CGLIB$clone$4$Method;
    private static final MethodProxy CGLIB$clone$4$Proxy;

    static void CGLIB$STATICHOOK1() {
        CGLIB$THREAD_CALLBACKS = new ThreadLocal();
        CGLIB$emptyArgs = new Object[0];
        Class var0 = Class.forName("com.fc.se.proxy.cglib.CglibBaseBean$$EnhancerByCGLIB$$bf21c123");
        Class var1;
        Method[] var10000 = ReflectUtils.findMethods(new String[]{"equals", "(Ljava/lang/Object;)Z", "toString", "()Ljava/lang/String;", "hashCode", "()I", "clone", "()Ljava/lang/Object;"}, (var1 = Class.forName("java.lang.Object")).getDeclaredMethods());
        CGLIB$equals$1$Method = var10000[0];
        CGLIB$equals$1$Proxy = MethodProxy.create(var1, var0, "(Ljava/lang/Object;)Z", "equals", "CGLIB$equals$1");
        CGLIB$toString$2$Method = var10000[1];
        CGLIB$toString$2$Proxy = MethodProxy.create(var1, var0, "()Ljava/lang/String;", "toString", "CGLIB$toString$2");
        CGLIB$hashCode$3$Method = var10000[2];
        CGLIB$hashCode$3$Proxy = MethodProxy.create(var1, var0, "()I", "hashCode", "CGLIB$hashCode$3");
        CGLIB$clone$4$Method = var10000[3];
        CGLIB$clone$4$Proxy = MethodProxy.create(var1, var0, "()Ljava/lang/Object;", "clone", "CGLIB$clone$4");
        CGLIB$say$0$Method = ReflectUtils.findMethods(new String[]{"say", "()V"}, (var1 = Class.forName("com.fc.se.proxy.cglib.CglibBaseBean")).getDeclaredMethods())[0];
        CGLIB$say$0$Proxy = MethodProxy.create(var1, var0, "()V", "say", "CGLIB$say$0");
    }

    final void CGLIB$say$0() {
        super.say();
    }

    public final void say() {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        if (var10000 != null) {
            var10000.intercept(this, CGLIB$say$0$Method, CGLIB$emptyArgs, CGLIB$say$0$Proxy);
        } else {
            super.say();
        }
    }

    final boolean CGLIB$equals$1(Object var1) {
        return super.equals(var1);
    }

    public final boolean equals(Object var1) {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        if (var10000 != null) {
            Object var2 = var10000.intercept(this, CGLIB$equals$1$Method, new Object[]{var1}, CGLIB$equals$1$Proxy);
            return var2 == null ? false : (Boolean)var2;
        } else {
            return super.equals(var1);
        }
    }

    final String CGLIB$toString$2() {
        return super.toString();
    }

    public final String toString() {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        return var10000 != null ? (String)var10000.intercept(this, CGLIB$toString$2$Method, CGLIB$emptyArgs, CGLIB$toString$2$Proxy) : super.toString();
    }

    final int CGLIB$hashCode$3() {
        return super.hashCode();
    }

    public final int hashCode() {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        if (var10000 != null) {
            Object var1 = var10000.intercept(this, CGLIB$hashCode$3$Method, CGLIB$emptyArgs, CGLIB$hashCode$3$Proxy);
            return var1 == null ? 0 : ((Number)var1).intValue();
        } else {
            return super.hashCode();
        }
    }

    final Object CGLIB$clone$4() throws CloneNotSupportedException {
        return super.clone();
    }

    protected final Object clone() throws CloneNotSupportedException {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        return var10000 != null ? var10000.intercept(this, CGLIB$clone$4$Method, CGLIB$emptyArgs, CGLIB$clone$4$Proxy) : super.clone();
    }

    public static MethodProxy CGLIB$findMethodProxy(Signature var0) {
        String var10000 = var0.toString();
        switch (var10000.hashCode()) {
            case -909388886:
                if (var10000.equals("say()V")) {
                    return CGLIB$say$0$Proxy;
                }
                break;
            case -508378822:
                if (var10000.equals("clone()Ljava/lang/Object;")) {
                    return CGLIB$clone$4$Proxy;
                }
                break;
            case 1826985398:
                if (var10000.equals("equals(Ljava/lang/Object;)Z")) {
                    return CGLIB$equals$1$Proxy;
                }
                break;
            case 1913648695:
                if (var10000.equals("toString()Ljava/lang/String;")) {
                    return CGLIB$toString$2$Proxy;
                }
                break;
            case 1984935277:
                if (var10000.equals("hashCode()I")) {
                    return CGLIB$hashCode$3$Proxy;
                }
        }

        return null;
    }

    public CglibBaseBean$$EnhancerByCGLIB$$bf21c123() {
        CGLIB$BIND_CALLBACKS(this);
    }

    public static void CGLIB$SET_THREAD_CALLBACKS(Callback[] var0) {
        CGLIB$THREAD_CALLBACKS.set(var0);
    }

    public static void CGLIB$SET_STATIC_CALLBACKS(Callback[] var0) {
        CGLIB$STATIC_CALLBACKS = var0;
    }

    private static final void CGLIB$BIND_CALLBACKS(Object var0) {
        CglibBaseBean$$EnhancerByCGLIB$$bf21c123 var1 = (CglibBaseBean$$EnhancerByCGLIB$$bf21c123)var0;
        if (!var1.CGLIB$BOUND) {
            var1.CGLIB$BOUND = true;
            Object var10000 = CGLIB$THREAD_CALLBACKS.get();
            if (var10000 == null) {
                var10000 = CGLIB$STATIC_CALLBACKS;
                if (var10000 == null) {
                    return;
                }
            }

            var1.CGLIB$CALLBACK_0 = (MethodInterceptor)((Callback[])var10000)[0];
        }

    }

    public Object newInstance(Callback[] var1) {
        CGLIB$SET_THREAD_CALLBACKS(var1);
        CglibBaseBean$$EnhancerByCGLIB$$bf21c123 var10000 = new CglibBaseBean$$EnhancerByCGLIB$$bf21c123();
        CGLIB$SET_THREAD_CALLBACKS((Callback[])null);
        return var10000;
    }

    public Object newInstance(Callback var1) {
        CGLIB$SET_THREAD_CALLBACKS(new Callback[]{var1});
        CglibBaseBean$$EnhancerByCGLIB$$bf21c123 var10000 = new CglibBaseBean$$EnhancerByCGLIB$$bf21c123();
        CGLIB$SET_THREAD_CALLBACKS((Callback[])null);
        return var10000;
    }

    public Object newInstance(Class[] var1, Object[] var2, Callback[] var3) {
        CGLIB$SET_THREAD_CALLBACKS(var3);
        CglibBaseBean$$EnhancerByCGLIB$$bf21c123 var10000 = new CglibBaseBean$$EnhancerByCGLIB$$bf21c123;
        switch (var1.length) {
            case 0:
                var10000.<init>();
                CGLIB$SET_THREAD_CALLBACKS((Callback[])null);
                return var10000;
            default:
                throw new IllegalArgumentException("Constructor not found");
        }
    }

    public Callback getCallback(int var1) {
        CGLIB$BIND_CALLBACKS(this);
        MethodInterceptor var10000;
        switch (var1) {
            case 0:
                var10000 = this.CGLIB$CALLBACK_0;
                break;
            default:
                var10000 = null;
        }

        return var10000;
    }

    public void setCallback(int var1, Callback var2) {
        switch (var1) {
            case 0:
                this.CGLIB$CALLBACK_0 = (MethodInterceptor)var2;
            default:
        }
    }

    public Callback[] getCallbacks() {
        CGLIB$BIND_CALLBACKS(this);
        return new Callback[]{this.CGLIB$CALLBACK_0};
    }

    public void setCallbacks(Callback[] var1) {
        this.CGLIB$CALLBACK_0 = (MethodInterceptor)var1[0];
    }

    static {
        CGLIB$STATICHOOK1();
    }
}
```

从上面的class文件可以看出，生成的代理类继承了CglibBaseBean，所以cglib相比于jdk动态代理，被代理类可以不用非得是接口





## 使用Agent输出class文件

再一种方式就是使用Instrument, 这是jdk5提供的能力。



关于instrument机制的介绍，可以看一下这一篇文章https://cnkirito.moe/instrument/



如何搭建一个demo进行测试，可以参考上面的文章，测试的代码可以在[git仓库](https://github.com/thend03/agent)查看



git clone之后，执行mvn clean package生成agent.jar， 在目标进程添加jvm参数`-javaagent:/Users/since/git/agent/target/agent.jar=-d=/Users/since/test/;-f=com/alibaba/dubbo/registry`

即可输出动态代理类的内容



- `-javaagent:/Users/since/git/agent/target/agent.jar `这个是agent的文件路径

- `-d=/Users/since/test/`是结果的输出路径

- `-f=com/alibaba/dubbo/registry`是要打印的类的前缀包路径



下面来简单说一下如何编写agent代码，实现class dump



agent项目结构如下

![image-20240918173752186](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202409181737237.png)



首先是要在META-INF下添加一个`MANIFEST.MF`，里面指定版本号，主类，是否可以重建类等属性

指定的premain-class是`com.fc.agent.GreetingAgent`，所以agent的入口就是GreetingAgent了。



来看一下`GreetingAgent`

类里有一个premain方法，和main方法比较像，options是传入的参数，以上面为例，运行时options为`-d=/Users/since/test/;-f=com/alibaba/dubbo/registry`。 



Instrumentation 便是 JAVA5 的 Instrument 机制的核心，它负责为类添加 ClassFileTransformer 的实现，从而对类进行装配增强修改



注意 premain 和它的两个参数不能随意修改， 规定是这样，不要修改。



我们为ins添加了一个ClazzDumpCustomTransformer， 对options进行解析，解析出文件输出路径，以及要输出的包的路径。

这里我想观察dubbo的adaptive代理，所以-f传的是dubbo的包路径

```java
import java.lang.instrument.Instrumentation;

/**
 *
 *
 * @author since
 * @date 2024-08-21 13:43
 */
public class GreetingAgent {
    public static void premain(String options, Instrumentation ins) {
        if (options != null) {
            System.out.printf("I've been called with options: \"%s\"\n", options);
        } else {
            System.out.println("I've been called with no options.");
        }
        ClazzDumpCustomTransformer transformer = getTransformer(options);
        //更换不同的transformer
        ins.addTransformer(transformer);
    }

    public static ClazzDumpCustomTransformer getTransformer(String options) {
        String exportDir = null;
        String filterStr = null;
        boolean recursiveDir = false;
        if (options != null) {
            if (options.contains(";")) {
                String[] args = options.split(";");
                for (String param1 : args) {
                    String[] kv = param1.split("=");
                    if ("-d".equalsIgnoreCase(kv[0])) {
                        exportDir = kv[1];
                    } else if ("-f".equalsIgnoreCase(kv[0])) {
                        filterStr = kv[1];
                    } else if ("-r".equalsIgnoreCase(kv[0])) {
                        recursiveDir = true;
                    }
                }
            } else {
                filterStr = options;
            }
        }
        System.out.println("getTransformer, exportDir: " + exportDir + ", filterStr: " + filterStr + ", recursiveDir: " + recursiveDir);
        return new ClazzDumpCustomTransformer(exportDir, filterStr, recursiveDir);

    }
}
```



ClazzDumpCustomTransformer负责dump class，对符合条件的class，获取文件内容，输出到指定目录下

```java
import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;
import java.util.Arrays;

/**
 * class dump
 *
 * @author since
 * @date 2024-09-10 13:36
 */
public class ClazzDumpCustomTransformer implements ClassFileTransformer {

    /**
     * 导出过滤表达式，此处为类名前缀， 以 -f 参数指定
     */
    private String filterStr;

    /**
     * 导出文件目录根目录, 以 -d 参数指定
     */
    private String exportBaseDir = "/tmp/";

    /**
     * 是否创建多级目录, 以 -r 参数指定
     */
    private boolean packageRecursive;

    public ClazzDumpCustomTransformer(String exportBaseDir, String filterStr) {
        this(exportBaseDir, filterStr, false);
    }

    public ClazzDumpCustomTransformer(String exportBaseDir, String filterStr, boolean packageRecursive) {
        if (exportBaseDir != null) {
            this.exportBaseDir = exportBaseDir;
        }
        this.packageRecursive = packageRecursive;
        this.filterStr = filterStr;
    }

    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined,
                            ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        if (needExportClass(className)) {
            System.out.println("needExportClass: " + className);
            int lastSeparatorIndex = className.lastIndexOf("/") + 1;
            String fileName = className.substring(lastSeparatorIndex) + ".class";
            String exportDir = exportBaseDir;
            if (packageRecursive) {
                exportDir += className.substring(0, lastSeparatorIndex);
            }
            exportClazzToFile(exportDir, fileName, classfileBuffer);         //"D:/server-tool/tmp/bytecode/exported/"
            System.out.println(className + " --> EXPORTED");
        }
        return classfileBuffer;
    }

    /**
     * 检测是否需要进行文件导出
     *
     * @param className class名,如 com.xx.abc.AooMock
     * @return y/n
     */
    private boolean needExportClass(String className) {
        if (filterStr != null) {
            return className.startsWith(filterStr);
        }
        return !className.startsWith("java") && !className.startsWith("sun");
    }

    /**
     * 执行文件导出写入
     *
     * @param dirPath 导出目录
     * @param fileName 导出文件名
     * @param data 字节流
     */
    private void exportClazzToFile(String dirPath, String fileName, byte[] data) {
        try {
            File dir = new File(dirPath);
            if (!dir.isDirectory()) {
                dir.mkdirs();
            }
            File file = new File(dirPath + fileName);
            if (!file.exists()) {
                System.out.println(dirPath + fileName + " is not exist, creating...");
                file.createNewFile();
            } else {

//                String os = System.getProperty("os.name");        // 主要针对windows文件不区分大小写问题
//                if(os.toLowerCase().startsWith("win")){
//                    // it's win
//                }
                try {
                    int maxLoop = 9999;
                    int renameSuffixId = 2;
                    String[] cc = fileName.split("\\.");
                    do {
                        long fileLen = file.length();
                        byte[] fileContent = new byte[(int) fileLen];
                        FileInputStream in = new FileInputStream(file);
                        in.read(fileContent);
                        in.close();
                        if (!Arrays.equals(fileContent, data)) {
                            fileName = cc[0] + "_" + renameSuffixId + "." + cc[1];
                            file = new File(dirPath + fileName);
                            if (!file.exists()) {
                                System.out.println("new create file: " + dirPath + fileName);
                                file.createNewFile();
                                break;
                            }
                        } else {
                            break;
                        }
                        renameSuffixId++;
                        maxLoop--;
                    } while (maxLoop > 0);
                } catch (Exception e) {
                    System.err.println("exception in read class file..., path: " + dirPath + fileName);
                    e.printStackTrace();
                }
            }
            FileOutputStream fos = new FileOutputStream(file);
            fos.write(data);
            fos.close();
        } catch (Exception e) {
            System.err.println("exception occur while export class.");
            e.printStackTrace();
        }
    }
}
```



执行mvn clean package打包之后，添加到目标进程，添加启动参数，查看执行结果，在控制台输出了要打印的class

![image-20240918174830197](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202409181748247.png)



进入输出路径，查看发现都生成了对应的class

![image-20240918175035451](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202409181750507.png)



将class文件拖进idea可以查看具体内容

从url里取相应的协议，注册默认是dubbo协议

![image-20240918175435124](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202409181754182.png)



## 使用arthas输出class文件

提到agent，那就不得不说到arthas了，arthas在运行时排障非常的好用，点个赞。



### 动态代理

当使用 `arthas-boot` 启动 Arthas 时，它会使用 Java 的 **Attach API** 来将自身动态注入到目标 JVM 进程中。通过附加 `agentmain` 方法，Arthas 可以对 JVM 进程进行监控和修改



那么我们就拿arthas来看一下，如何打印运行时的class文件，主要是使用jad命令

arthas的安装使用相关内容参考官方文档: https://arthas.aliyun.com/doc/



修改一下动态代理的测试类

```java
public class DynamicProxyMain {
    public static void main(String[] args) throws InterruptedException {
        //动态代理时生成class文件
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles","true");
        DynamicProxy dynamicProxy = new DynamicProxy(new HumenImpl());
        Humen humenProxy = dynamicProxy.getProxy();
        humenProxy.eat("rice");

        Thread.sleep(100*24*60*60*1000L);

        //保存$Proxy0到文件
//        saveProxyFile();

    }
```



让进程挂在这，然后启动arthas，选择这个进程进行attach



使用sc命令搜索动态代理生成的类: `sc *Proxy0*`, 得到了代理类的全路径，然后使用jad命令反编译class，得到动态生成的代理类的文件内容



![image-20240919112806799](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202409191128855.png)



再来看一下dubbo的示例，看看javassist生成的代理类如何

```java
public class DubboSpi {
    public static void main(String[] args) throws InterruptedException {
        ExtensionLoader<SimpleExt> extensionLoader = ExtensionLoader.getExtensionLoader(SimpleExt.class);

//        SimpleExt defaultExtension = extensionLoader.getDefaultExtension();
//        SimpleExt dog = extensionLoader.getExtension("dog");
//        SimpleExt dog2 = extensionLoader.getExtension("dog", true);
//        SimpleExt cat = extensionLoader.getExtension("cat");
//        dog.yell(null, "dogdog");
//        SimpleExt dog1 = extensionLoader.getExtension("dog");
//        dog1.yell(null, "dog1");

        SimpleExt adaptiveExtension = extensionLoader.getAdaptiveExtension();

        Thread.sleep(100*24*60*60*1000L);

        URLAddress urlAddress = new URLAddress("127.0.0.1", 20880);
        URLParam parse = URLParam.parse(new HashMap<>());
        adaptiveExtension.echo(new URL(urlAddress, parse), "dogs");
        adaptiveExtension.bang(new URL(urlAddress, parse),0);

    }
}
```



### javassist

仍然让进程挂在那，重新启动arthas进行attach

使用jad反编译class，正常

![image-20240919130747879](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202409191307947.png)



### cglib

让进程空跑，使用arthas进行attach

```java
import com.alibaba.excel.support.cglib.core.DebuggingClassWriter;
import net.sf.cglib.proxy.Enhancer;

/**
 * cglib proxy
 *
 * @author since
 * @date 2024-09-19 13:24
 */
public class CglibProxy {
    public static void main(String[] args) throws InterruptedException {
        // 指定 CGLIB 将动态生成的代理类保存至指定的磁盘路径下
        System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "/Users/since/git/vamei/out/cglib");
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(CglibBaseBean.class);
        enhancer.setCallback(new CglibCustomizedMethodInterceptor());
        CglibBaseBean baseBean = (CglibBaseBean) enhancer.create();
        baseBean.say();
        Thread.sleep(100*24*60*60*1000L);
    }
}
```





使用sc搜索相关的代理类，然后使用jad反编译, 也正常输出了

![image-20240919174244260](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202409191742338.png)



## 总结

本文介绍了一下java生成动态代理类的方式，像jdk动态代理，cglib动态代理，javassist在项目中都有广泛的应用。



对于查看动态代理class的内容，介绍了相关的方法进行查看，但是不够通用。



通过写agent，了解了agent的一些知识，如何使用instrument机制在启动时和运行时增加class



但是相比于自己写agent，还是arthas更加的方便通用一点，不用打agent包，也不用添加启动参数去重启进程。

arthas直接attach到目标进程上，可以模糊查询类，然后使用jad进行反编译，非常的方便。



arthas作为一个生产排障工具，还有其他更多强大的功能，值得去了解学习
