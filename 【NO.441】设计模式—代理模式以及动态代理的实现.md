# 【NO.441】设计模式—代理模式以及动态代理的实现

代理模式（Proxy Design Pattern）是为一个对象提供一个替身，以控制对这个对象的访问。即通过代理对象访问目标对象。被代理的对象可以是远程对象、创建开销大的对象或需要安全控制的对象。

## **1.代理模式介绍**

在结束创建型模式的讲解后，从这一篇开始就进入到了结构型模式，结构型模式主要是总结一些类和或对象组合在一起的结构。代理模式在不改变原始代理类的情况下，通过引入代理类来给原始类附加功能。

代理模式的主要结构如下：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213160501_14222.jpg)

1. `Subject`：抽象主题类，通过接口或抽象类声明主题和代理对象实现的业务方法
2. `RealSubject`：真实主题类，实现`Subject`中的具体业务，是代理对象所代表的真实对象
3. `Proxy`：代理类，其内部含有对真实主题的引用，它可以访问、控制或扩展`RealSubject`的功能
4. `Client`：客户端，通过使用代理类来访问真实的主题类

按照上面的类图，可以实现如下代码：

```
//主题类接口
public interface Subject {
    void Request();
}
//真实的主题类
public class RealSubject implements Subject{
    @Override
    public void Request() {
        System.out.println("我是真实的主题类");
    }
}
//代理类
public class Proxy implements Subject{
    private RealSubject realSubject;
    @Override
    public void Request() {
        if (realSubject == null) {
            realSubject = new RealSubject();
        }
        realSubject.Request();
    }
}
//客户端
public class Client {
    public static void main(String[] args) {
        Proxy proxy = new Proxy();
        proxy.Request();
    }
}
```

代理模式有比较广泛的使用，比如`Spring AOP`、`RPC`、缓存等。在 Java 中，根据代理的创建时期，可以将代理模式分为静态代理和动态代理，下面就来分别阐述。

## **2.代理模式实现**

动态代理和静态代理的区分就是语言类型是在运行时检查还是在编译期检查。

### **2.1 .静态代理**

静态代理是指在编译期，也就是在JVM运行之前就已经获取到了代理类的字节码信息。即Java源码生成`.class`文件时期：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213160502_90225.jpg)

由于在JVM运行前代理类和真实主题类已经是确定的，因此也被称为静态代理。

在实际使用中，通常需要定义一个公共接口及其方法，被代理对象（目标对象）与代理对象一起实现相同的接口或继承相同的父类。其实现代码就是第一节中的代码。

### **2.2.动态代理**

动态代理，也就是在JVM运行时期动态构建对象和动态调用代理方法。

常用的实现方式是反射。**反射机制**是指程序在运行期间可以访问、检测和修改其本身状态或行为的一种能力，使用反射我们可以调用任意一个类对象，以及其中包含的属性及方法。比如JDK Proxy。

此外动态代理也可以通过ASM(Java 字节码操作框架)来实现。比如CGLib。

#### **2.2.1.JDK Proxy**

这种方式是JDK自身提供的一种方式，它的实现不需要引用第三方类，只需要实现`InvocationHandler`接口，重写`invoke()`方法即可。代码实现如下所示：

```
public class ProxyExample {
    static interface Car {
        void running();
    }
    static class Bus implements Car {
        @Override
        public void running() {
            System.out.println("bus is running");
        }
    }
    static class Taxi implements Car {
        @Override
        public void running() {
            System.out.println("taxi is runnig");
        }
    }
    //核心部分 JDK Proxy 代理类
    static class JDKProxy implements InvocationHandler {
        private Object target;
        public Object getInstance(Object target) {
            this.target = target;
            //获得代理对象
            return Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), this);
         }
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            Object result = method.invoke(target, args);
            return result;
        }
    }
    public static void main(String[] args) {
        JDKProxy jdkProxy = new JDKProxy();
        Car instance = (Car) jdkProxy.getInstance(new Taxi());
        instance.running();
    }
}
```

动态代理的核心是实现`Invocation`接口，我们再看看`Invocation`接口的源码：

```
public interface InvocationHandler {
    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}
```

实际上是通过`invoke()`方法来触发代理的执行方法。最终使得实现`Invocation`接口的类具有动态代理的能力。

动态代理的好处在于不需要和静态代理一样提前写好公共的代理接口，只需要实现`Invocation`接口就可拥有动态代理能力。下面我们再来看看 CGLib 是如何实现的

#### **2.2.2. CGLib**

CGLib 动态代理采取的是创建目标类的子类的方式，通过子类化，我们可以达到近似使用被调用者本身的效果。其实现代码如下所示：

```
public class CGLibExample {
    static class car {
        public void running() {
            System.out.println("car is running");
        }
    }
    static class CGLibProxy implements MethodInterceptor {
        private Object target;
        public Object getInstance(Object target) {
            this.target = target;
            Enhancer enhancer = new Enhancer();
            //设置父类为实例类
            enhancer.setSuperclass(this.target.getClass());
            //回调方法
            enhancer.setCallback(this);
            //创建代理对象
            return enhancer.create();
        }
        @Override
        public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
            Object result = methodProxy.invokeSuper(o, objects);
            return result;
        }
    }
    public static void main(String[] args) {
        CGLibProxy cgLibProxy = new CGLibProxy();
        car instance = (car) cgLibProxy.getInstance(new car());
        instance.running();
    }
}
```

从代码可以看出CGLib 也是通过实现代理器的接口，然后再调用某个方法完成动态代理，不同的是CGLib在初始化被代理类时，是通过`Enhancer`对象把代理对象设置为被代理类的子类来实现动态代理：

```
Enhancer enhancer = new Enhancer();
//设置父类为实例类
enhancer.setSuperclass(this.target.getClass());
//回调方法
enhancer.setCallback(this);
//创建代理对象
return enhancer.create();
```

#### **2.2.3. JDK Proxy 和 CGLib 的区别**

1. 来源：JDK Proxy 是JDK 自带的功能，CGLib 是第三方提供的工具
2. 实现：JDK Proxy 通过拦截器加反射的方式实现；CGLib 基于ASM实现，性能比较高
3. 接口：JDK Proxy 只能代理继承接口的类，CGLib 不需要通过接口来实现，它是通过实现子类的方式来完成调用

## **3.代理模式的应用场景**

### **3.1. MapperProxyFactory**

在MyBatis 中，也存在着代理模式的使用，比如`MapperProxyFactory`。其中的`newInstance()`方法就是生成一个具体的代理来实现功能，代码如下：

```
public class MapperProxyFactory<T> {
  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethodInvoker> methodCache = new ConcurrentHashMap<>();
  public MapperProxyFactory(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }
  public Class<T> getMapperInterface() {
    return mapperInterface;
  }
  public Map<Method, MapperMethodInvoker> getMethodCache() {
    return methodCache;
  }
  // 创建代理类
  @SuppressWarnings("unchecked")
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }
  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
}
```

### **3.2. Spring AOP**

代理模式最常使用的一个应用场景就是在业务系统中开发一些非功能性需求，比如监控、统计、鉴权、限流、事务、日志等。将这些附加功能与业务功能解耦，放在代理类中统一处理，让程序员只需要关注业务方面的开发。而Spring AOP 的切面实现原理就是基于动态代理

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213160502_63197.jpg)

Spring AOP 的底层通过上面提到的 JDK Proxy 和 CGLib动态代理机制，为目标对象执行横向织入。当Bean实现了接口时， Spring就会使用JDK Proxy，在没有实现接口时就会使用 CGLib。也可以在配置中强制使用 CGLib：

```
<aop:aspectj-autoproxy proxy-target-class="true"/>
```

### **3.3. RPC 框架的封装**

RPC 框架的实现可以看作成是一种代理模式，通过远程代理、将网络同步、数据编解码等细节隐藏起来，让客户端在使用 RPC 服务时，不必考虑这些细节。

原文链接：https://zhuanlan.zhihu.com/p/489003709

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)