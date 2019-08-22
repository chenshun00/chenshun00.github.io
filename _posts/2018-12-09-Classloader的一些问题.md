### Classloader的一些问题

#### 背景
最近在使用 `java agent` 开发的时候发现还能将 `agent` 的jar包attach到 `SystemClassLoader` ，感觉到自己对于 `Instrumentation` 和 `ClassLoader` 的一些点还是感到迷惑不解，在周末的时候又看了一些博客，看了下书加深了一下自己的

通过 `Instrumentation` 我们可以加入 `agent` 去监控运行的 java 应用，他的api主要有以下这些


*	void addTransformer(ClassFileTransformer transformer, boolean canRetransform);  
*    void addTransformer(ClassFileTransformer transformer);

通过 `void addTransformer(ClassFileTransformer transformer, boolean canRetransform);` 加上自定义 ClassFileTransformer 就可以在加载的时候修改类的字节码,然后达到 `Aop`,`热部署` 等不为人知的功能。

结论:这样的使用模式也带来了一定的问题，就是我们在 `agent` 中使用的时候，容易带来 `ClassNotFoundException` ， 原因就是确定一个类是由 `classLoader` + `class` 2者共同决定的，如果一个 `class` 同时被 `SystemClassLoader` 和 自定义的 `MyClassLoader` 加载的，这2个class对象是不一致的。这样带来的问题就是我们注入的字节码对象必须是 `加载agent的ClassLoader或者更高级的ClassLoader加载才行`.

#### 双亲委托模型
说到ClassLoader就不得不说到双亲委托模型，模型网上介绍的比较多，一般而言就是
	
*	boot 加载 `rt.jar`
* ext 加载 ext/目录下的
* app 加载classpath下的(未设置应用的classloader)
* custom 自定义的classloader

但是这些都有一个缺点，就是理论味十足，不能解决实际环境中出现的问题。

而在双亲委托模型的基础上，又出现了 `上下文类加载器`，主要用来解决 `jndi spi` 等等本应该由 `boot` 或者 `app` classloader加载的但是实际却不是由这2者加载的问题。

例子如下:

```java
 	ServiceLoader<Driver> load = ServiceLoader.load(Driver.class,Thread.currentThread().getContextClassLoader());
    Iterator<Driver> iterator = load.iterator();
    while (iterator.hasNext()){
        System.out.println(111);
        iterator.next();
    }
```
加载 `sql` 驱动的例子就可以说明问题了，`ServiceLoader` 位于 `rt.jar` 中，而 `驱动driver` 却是在厂商提供的，按照双亲委托模型的设计 `ServiceLoader` 中加载的类也应该由加载 `ServiceLoader` 的 `类加载器` 进行加载，这样一来就导致了厂商提供的驱动`ClassNotFound` 了，为了解决这个问题而引入了上下文加载器，主要用来解决夫加载器获取不到的类，交给子类的classloader去获取.

```java
    public static <S> ServiceLoader<S> load(Class<S> service) {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();//这里
        return ServiceLoader.load(service, cl);
    }
```
这也就造成了 `双亲委托模型` 被破坏了.


#### 遇到的问题
因为是使用 `-javaagent:path/xxx/xxx.jar` 的方式进行的，所以这个 `agent` jar包在jvm启动时就被 `boot` 或者 `app类加载` 加载完毕，取决于开发者. 正因为这个jar包是被父加载器加载的，所以他是拿不到 `classpath` 下的 `class` 的. 而我们开发 `agent` 的功能是监控应用，如果获取不到应用的 `jar 包，那么agent的本身的含义也就不存在了。思考良久，这个问题我还是没有解决.

agent 被AppClassLoader 加载，那么这个jar包已经在这个jar包引用的类都只能被app以及他的父加载器引用到，如果我们强行使用一个类，例如 `a.b.c.d.xxxService` 这个类，那么appClassLoader 首先去 `app` 的 `classpath` 下找，先boot找，找不到，app自己找，也找不到 ClassNotFoundException了。

> ps 使用Thread.currentThread().getContextClassLoader();目前是只能是反射.


#### tomcat的classLoader
按照java的规范，一个servlet容器是可以部署多个tomcat的，那么多个tomcat之间是如何隔离的呢？也是使用classloader进行隔离，每一个webapp都对应了一个webappClassLoader，这样一个容器就可以容纳多个应用，而不会互相影响。

同时tomcat的类加载也一定程度上影响了 `双亲委托模型的使用`，在tomcat中的加载顺序是按照如下顺序加载的

> 如果存在外部资源并且首先查找尾部资源则先委托父加载器进行加载否则先进行内部查找，如果没有找到，并且存在外部资源，那么委托个父加载寻找.还没有找到ClassNotFoundException
> tomcat默认的classpath是

```java
	/**
     * Code base to use for classes loaded from WEB-INF/classes.
     */
    private URL webInfClassesCodeBase = null;
```

按照 `模型`，应该先交给父加载器加载，而tomcat却是看条件给父加载加载.

#### 小结
classLoader 的问题比较麻烦，目前我也不清楚该如何去解决这个问题，但是classloader的基本使用模式还是较为简单的.
