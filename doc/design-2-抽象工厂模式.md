### 抽象工厂方法模式

#### 1.项目简介

| 工程 | 描述                                            |
| ---- | ----------------------------------------------- |
| 2-00 | 场景模拟工厂，模拟出使用Redis升级为集群时类改造 |
| 2-01 | 使用一坨代码实现业务需求，也是对ifelse的使用    |
| 2-02 | 通过设计模式优化改造代码，产生对比性从而学习    |

#### 2.业务解析

从单机Redis升级，可以预见的问题会有：

1. 很多服务用到Redis需要一起升级到集群。
2. 需要兼容集群A和集群B，便于后续的宰备。
3. 两套集群提供的接口和方法各有差异，需要做适配。
4. 不能影响到目前正常运行的系统。

#### 3.抽象工厂模式的代码实现

这里抽象工厂的创建和获取方式，会采用代理类的方式进行实现。所被代理的类就是目前的Redis操作方法类。让这个类不需要任何修改下，就可以实现调用集群A和集群B的数据服务。

因为集群A与集群B的部分方法提供上是不同的，因此需要做一个接口适配，而这个适配类就相当于工厂中的工厂，用于创建把不同的服务抽象为同一的接口做相同的业务。

##### 3.1定义适配接口

```java
package org.itstack.demo.desgin.factory;

import java.util.concurrent.TimeUnit;

public interface ICacheAdapter {

    String get(String key);

    void set(String key, String value);

    void set(String key, String value, long timeout, TimeUnit timeUnit);

    void del(String key);

}
```

- 定义了适配接口，分别包装两个集群中差异化的接口名。
- 主要作用是让所有集群的提供方，能在统一的方法名称下进行操作。也方便后续的拓展。

##### 3.2实现集群使用服务

**EGMCacheAdapter**

```java
public class EGMCacheAdapter implements ICacheAdapter {

    private EGM egm = new EGM();

    public String get(String key) {
        return egm.gain(key);
    }

    public void set(String key, String value) {
        egm.set(key, value);
    }

    public void set(String key, String value, long timeout, TimeUnit timeUnit) {
        egm.setEx(key, value, timeout, timeUnit);
    }

    public void del(String key) {
        egm.delete(key);
    }
}
```

**IIRCacheAdapter**

```java
public class IIRCacheAdapter implements ICacheAdapter {

    private IIR iir = new IIR();

    public String get(String key) {
        return iir.get(key);
    }

    public void set(String key, String value) {
        iir.set(key, value);
    }

    public void set(String key, String value, long timeout, TimeUnit timeUnit) {
        iir.setExpire(key, value, timeout, timeUnit);
    }

    public void del(String key) {
        iir.del(key);
    }

}
```

- 以上两个实现都非常容易，在统一方法名下进行包装。

##### 3.3定义抽象工程代理类和实现

**JDKProxy**

```java
public class JDKProxy {

    public static <T> T getProxy(Class<T> interfaceClass, ICacheAdapter cacheAdapter) throws Exception {
        InvocationHandler handler = new JDKInvocationHandler(cacheAdapter);
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
        Class<?>[] classes = interfaceClass.getInterfaces();
        return (T) Proxy.newProxyInstance(classLoader, new Class[]{classes[0]}, handler);
    }
}
```

- 这⾥主要作用就是完成代理类，同事对于使用哪个集群由外部通过入参进行传递。

**JDKInvocationHandler**

```java
public class JDKInvocationHandler implements InvocationHandler {

    private ICacheAdapter cacheAdapter;

    public JDKInvocationHandler(ICacheAdapter cacheAdapter) {
        this.cacheAdapter = cacheAdapter;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        return ICacheAdapter.class.getMethod(method.getName(), ClassLoaderUtils.getClazzByArgs(args)).invoke(cacheAdapter, args);
    }

}
```

- 在代理类的实现中也非常简单，通过穿透进来的集群服务进行方法操作。
- 另外在invoke中通过使用获取方法名称反射方式，调用对应的方法功能，也就简化了整体的使用。
- 到这我们就已经将整体的功能实现完成了，关于抽象工厂这部分也可以使用非代理的方式进行实现。

#### 4.测试验证

**编写测试类**

```java
public class ApiTest {

    @Test
    public void test_CacheService() throws Exception {

        CacheService proxy_EGM = JDKProxy.getProxy(CacheServiceImpl.class, new EGMCacheAdapter());
        proxy_EGM.set("user_name_01", "小傅哥");
        String val01 = proxy_EGM.get("user_name_01");
        System.out.println("测试结果：" + val01);

        CacheService proxy_IIR = JDKProxy.getProxy(CacheServiceImpl.class, new IIRCacheAdapter());
        proxy_IIR.set("user_name_01", "小傅哥");
        String val02 = proxy_IIR.get("user_name_01");
        System.out.println("测试结果：" + val02);

    }
}
```

- 如果后续有扩展的需求，可以按照这样的类型方式进行补充，同时对于改造上来说，并没有改动原来的方法，降低了修改成本。

#### 5.总结

- 抽象工厂模式，所要解决的问题就是在一个产品族里，存在很多个不同类型的产品（Redis集群、操作系统）情况下，接口选择的问题。而这种场景在业务开发中也是非常多见的。只不过可能有时候没有将他们抽象化出来。
- 这个设计模式满足了单一职责、开闭原则、解耦等优点。

