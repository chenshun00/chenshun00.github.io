## ClassLoader 的一些问题2

最近在推动SpringBoot在内部的使用过程中，发生了一次 `ClassNotFoundException` 导致 `部分应用` 启动失败的一次总结.

### 背景
推动SpringBoot在技术人员中的应用，首先需要发布系统支持 `Spring Boot` 的部署和快速发布，为了兼容发布系统和监控系统的统一，必须悄无声息的将监控应用打入Spring Boot 开发的内部应用当中，在 `Tomcat + Spring` 的体系中是天然支持 `JSP` 的，但是在 `Spring Boot ` fat jar 中对 `JSP` 的支持不是很好，为了解决这个问题从而出现了本次 `Classloader` 的一些问题2. 

* 关于如何编写兼容 `Spring Boot 1.x ` 和 `Spring Boot 2.x` 可以看看 [兼容1.x和2.x的starter](https://github.com/chenshun00/springboot-Fragment/blob/master/springboot-starter/readme.md) 
* 关于如何将多个jar包合并成一个jar包可以 [用maven assembly插件打jar包实现依赖包归档](https://blog.csdn.net/e5945/article/details/7777286)
* 关于 `Spring Boot` 是如何支持 `JSP` 的可以看看 [Spring Boot加载指定目录下的JSP文件](https://github.com/chenshun00/springboot-Fragment/blob/master/test-inner-jsp/src/main/java/top/huzhurong/springboot/testinnerjsp/view/ResourceConfigurer.java)


### 事故过程与排查

Spring Boot 的兼容性测试通过之后便将 `starter jar` 放置到了 `tomcat/lib` 目录下，然后便有技术人员反馈应用启动失败，失败原因 `ClassNotFoundException` . 而且还是他们项目中没有应用的SpringBoot，立即开始排查，首先想到的是为什么 `Spring` 会加载我的starter，我没有什么地方引用到了这个`jar` ，这里想不通， 随后删除jar包，应用成功启动，将问题集中在`starter` 中，随后我将 `starter` 放置到 `pom.xml` 中直接依赖，应用正常启动. 开始考虑2者的差异，唯一的解释 `Spring` 以某种形式加载了我的starter，但是tomcat#webappClassLoader 仅能加载 `WEB-INF` 下的classes和jar包,所以委托给commonClassLoader去tomcat/lib目录下加载，找到并加载，但是在加载该类的过程中，又引用到了Spring中其他的类(我使用的是 `BeanPostProcessor` )，因为双亲委托模型的原因，加载该类中出现符号引用或者直接饮用必须有加载该类的加载器去进行loading(上下文加载器打破模型这里不涉及)，commonClassLoader 会去 appClassLoader 和 ext 以及boot中进行加载，如果都找不到那么就会出现本篇描述的问题 `ClassNotFoundException` ，想到这里立刻展开验证，将出现的类全部抹除之后重新运行，出现如下异常(篇幅所限，仅留下关键部分),到这一步问题很明显了，就是类加载器的问题。

```java
	at org.springframework.context.annotation.ConfigurationClassPostProcessor.enhanceConfigurationClasses(ConfigurationClassPostProcessor.java:414)
	at com.sun.jmx.interceptor.DefaultMBeanServerInterceptor.invoke(DefaultMBeanServerInterceptor.java:819)
Caused by: org.springframework.cglib.core.CodeGenerationException: java.lang.reflect.InvocationTargetException-->null
	at org.springframework.cglib.core.AbstractClassGenerator.generate(AbstractClassGenerator.java:345)
	at org.springframework.cglib.proxy.Enhancer.generate(Enhancer.java:492)
	at org.springframework.cglib.core.AbstractClassGenerator$ClassLoaderData$3.apply(AbstractClassGenerator.java:93)
	... 53 more
Caused by: java.lang.reflect.InvocationTargetException
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.cglib.core.ReflectUtils.defineClass(ReflectUtils.java:459)
	at org.springframework.cglib.core.AbstractClassGenerator.generate(AbstractClassGenerator.java:336)
	... 67 more
Caused by: java.lang.NoClassDefFoundError: org/springframework/context/annotation/ConfigurationClassEnhancer$EnhancedConfiguration
	at java.lang.ClassLoader.defineClass1(Native Method)
	at java.lang.ClassLoader.defineClass(ClassLoader.java:763)
	... 73 more
Caused by: java.lang.ClassNotFoundException: org.springframework.context.annotation.ConfigurationClassEnhancer$EnhancedConfiguration
	at java.net.URLClassLoader.findClass(URLClassLoader.java:381)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
	... 75 more
```

虽然问题找到了，但是为什么会被Spring加载却是没有想通，按照类加载器的规范，仅有在

* new，设置或读取类静态字段 ，调用静态方法 
* java.lang.reflect 进行反射会进行加载
* 初始化类时父类会被初始化
* 启动执行并包含main方法的类

等等少数场景下才会导致被加载，很显然我没有使用到这些场景，带着疑问显示打开trace日志，一运行，发现了问题，我的starter被加载了，随后进行debug调试，

```java
String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
					resolveBasePackage(basePackage) + '/' + this.resourcePattern;
			Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);//这里的扫描是 classpath*:com.example.**/*.class 所有的class文件 注意classpath 和 classpath* 的区别。
			boolean traceEnabled = logger.isTraceEnabled();
			boolean debugEnabled = logger.isDebugEnabled();
			for (Resource resource : resources) {
				if (traceEnabled) {
					logger.trace("Scanning " + resource); //扫描jar包
				}
				if (resource.isReadable()) {
					try {
					//asm 实例化，如果Spring的版本太低，那么就会出现兼容问题，
						MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
```

这里就知道了，大概率是开发配置的component-scan和我们的包名是一样的开头，并且starter是使用 `@Configuration` 进行配置的，一看就知道它是被 `componet` 注解. 随后修改包名，重启应用，问题解决.

### 如何重现

*  Spring Boot 2.0.3 版本
*  Tomcat 8.5.35 版本

编写一个包含了Spring#class (需要使用，类被@Service或其他的2个注解) 的jar包， `<scope>provide</scope>` ，放置到tocmat/lib目录下。 Spring应用配置到component-scan需要扫描这个包的路径，随后进行调试


### 总结

通过这里的配置，基本上可以明白了双亲委托模型的使用，以及Spring配置的重要性，需要明白Spring的黑盒性质.
