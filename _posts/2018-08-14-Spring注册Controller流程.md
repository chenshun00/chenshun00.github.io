### 讲述Spring处理controller以及处理http请求的过程

#### 使用工具

1. 使用工具
	*	`Spring` 版本： `5.0.7.release`，Spring地址:`git@github.com:spring-projects/spring-framework.git`
	*	包管理器:`maven`
	* 	采用的ide: `idea`
2. 看完本文的收获
	*	大致了解Spring解析Controller的时候，Spring的内部解析其实已经结束了。
	*	理解lamada表达式的执行过程
	*  了解 `InitializingBean` 的使用，分析 `Controller` 和`RequestMapping` 从它的实现方法切入。
	*	了解模版模式的使用

#### 处理controller

首先要明白Spring是要解析xml文件放在第一位，不论后续的什么注解，其实本质都是xml的解析过程，我们不是要分析Spring是如何解析xml或者annotation的，所有我们只要了解到，从Spring开始处理controller的时候，已经处理好了xml这一解析过程，我们从 `RequestMappingHandlerMapping` 这个类入手，它的整体实例化都是位于 `InitializingBean接口的 afterPropertiesSet() 当中`，如下代码所示

```text
@Override
	public void afterPropertiesSet() {
		# 实例化 RequestMappingInfo.BuilderConfiguration 对象，稍后处理ReuquestMappingInfo的是会使用到
		this.config = new RequestMappingInfo.BuilderConfiguration();
		this.config.setUrlPathHelper(getUrlPathHelper());
		this.config.setPathMatcher(getPathMatcher());
		this.config.setSuffixPatternMatch(this.useSuffixPatternMatch);
		this.config.setTrailingSlashMatch(this.useTrailingSlashMatch);
		this.config.setRegisteredSuffixPatternMatch(this.useRegisteredSuffixPatternMatch);
		this.config.setContentNegotiationManager(getContentNegotiationManager());
		## 重点位置，这里会调用父类的afterPropertiesSet方法
		super.afterPropertiesSet();
	}
```
父类的 `afterPropertiesSet` 方法，很简单，就是初始化，所有的 `handlerMethod`，而 `handlerMethod` 就是实际处理http请求的那个类对象

```java
	@Override
	public void afterPropertiesSet() {
		initHandlerMethods();//接着向后看
	}
	//为了简化篇幅，我将异常和日志信息全都移除了
	protected void initHandlerMethods() {
		//找到Spring容器当中所有的bean，obtainApplicationContext()是为了获取ApplicationContext
		//这里涉及到 ApplicationContextAware，Spring的Aware接口都是为了实现注入某个数据/实例的
		String[] beanNames = (this.detectHandlerMethodsInAncestorContexts ?
		BeanFactoryUtils.beanNamesForTypeIncludingAncestors(obtainApplicationContext(), Object.class) :		obtainApplicationContext().getBeanNamesForType(Object.class));

		for (String beanName : beanNames) {
			//beanName的名字不是以scopedTarget开头的，proxy可能会用到这个数据
			if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
				//从Spring容器中获取bean的类型
				Class<?> beanType = obtainApplicationContext().getType(beanName);
				//isHandler方法会去判断这个类有没有RequestMapping[或者]Controller注解，说明只要其中一个即可
				if (beanType != null && isHandler(beanType)) {
					//处理bean，一般说来就是controller类
					detectHandlerMethods(beanName);
				}
			}
		}
	handlerMethodsInitialized(getHandlerMethods());
}
```
从上述开始，我们就可以开始处理被 `@Controller` 注解的类了，使用for循环，会把所有的类都找到。下文开始分析是如何找到类的方法都，是如何将注解在类和方法级别的 `@RequestMapping` 联合起来的.

开始讲述处理Controller之前，我们先要了解到lambda表达式(匿名内部类)或者说回调函数的运行状况，我们一般写的lambda表达式都是先定义，后进行实际处理的，lambda表达式其实只是定义了回调的执行代码，具体的执行等到它被调用的时候才能去执行定义的回调代码，理解了这一点就能很好的理解下文了，具体的代码可以查看 [我的设计模式之回调](https://github.com/chenshun00/pattern/blob/master/src/main/java/com/callback/App.java)

先从总体上，看下 `Spring` 解析 `Controller` 的脉络. `final` 类型是因为在lambda中使用

```java
protected void detectHandlerMethods(final Object handler) {
		//如果是字符串，那么从容器中找到这个 beanName 对应的类，上文以ing讲述了 obtainApplicationContext() 其实就是ApplicationContext
		Class<?> handlerType = (handler instanceof String ? 
					obtainApplicationContext().getType((String) handler) : handler.getClass());
		if (handlerType != null) {
			//得到用户的class，其实我们在上边就得到了，多此一举是防止cglib的处理，我们需要的是用户类，而不是代理类
			final Class<?> userType = ClassUtils.getUserClass(handlerType);
			//这里涉及到了几个lambda表达式的运用，首先我们先不看这里的getMappingForMethod(method, userType))
			//等到后续用到的时候再去处理这里的源代码，从这里我们就跳转到了 MethodIntrospector.selectMethods了，这里会选择处Controller类中所有被@RequestMapping注解的方法
			//当然方法必须是public的
			Map<Method, T> methods = MethodIntrospector.selectMethods(userType, (MethodIntrospector.MetadataLookup<T>) method -> getMappingForMethod(method, userType));
			//经过上述的选择处理，我们已经得到了这个类里边的方法了
			methods.forEach((key, mapping) -> {
				//获取调用的方法
				Method invocableMethod = AopUtils.selectInvocableMethod(key, userType);
				//注册到Map当中，后续会根据http请求到uri去实际[转发]到那个Controller那里去
				registerHandlerMethod(handler, invocableMethod, mapping);
			});
		}
	}
```

#### 从Controller中获取method
`MethodIntrospector.selectMethods` 定义了如何获取类中的方法，主要的处理逻辑都在这里，只要明白了这里，后续的注册就可以一览无余了

*	方法过滤器，过滤桥接方法(编译器生产)，过滤Synthetic方法，过滤object的方法

```java
public static final MethodFilter USER_DECLARED_METHODS =
			(method -> (!method.isBridge() && !method.isSynthetic() && method.getDeclaringClass() != Object.class));
```

*	选择方法

```java
	public static <T> Map<Method, T> selectMethods(Class<?> targetType, final MetadataLookup<T> metadataLookup) {
		final Map<Method, T> methodMap = new LinkedHashMap<>();
		Set<Class<?>> handlerTypes = new LinkedHashSet<>();
		Class<?> specificHandlerType = null;
		//不是jdk代理
		if (!Proxy.isProxyClass(targetType)) {
			//处理cglib代理，上文有讲述
			specificHandlerType = ClassUtils.getUserClass(targetType);
			handlerTypes.add(specificHandlerType);
		}
		//添加该class实现的接口(包括父类实现的)，因为这个类有可能实现了其他的接口
		handlerTypes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetType));
		//开始处理
		for (Class<?> currentHandlerType : handlerTypes) {
			final Class<?> targetClass = (specificHandlerType != null ? specificHandlerType : currentHandlerType);
			//这里又涉及到了lambda的定义和调用的顺序问题了，现在我们先去看doWithMethods方法，稍后就能看到这个lambda的内在了
			ReflectionUtils.doWithMethods(currentHandlerType, method -> {
				//可先看下文，马上就会提到这里
				//这里就是mc.doWith(method)具体执行的代码了，很符合我们说的先定义后执行的说法
				//主要是接口和具体类之间的区别，意义不大，如果是我们Controller类的方法，那么直接放回
				//如果是Controller实现的接口方法，那么使用反射获取目标类的方法，这里反射可能会失败，失败了就使用原方法
				Method specificMethod = ClassUtils.getMostSpecificMethod(method, targetClass);
				//又一次回调，这次的回调就回到一开始全局脉络的地方了，找到这个方法对应的RequestMappingInfo，可先看下边的分析，再回过头看这里
				//看完下边的分析之后，我们能够得到这个RequestMapping了
				T result = metadataLookup.inspect(specificMethod);
				if (result != null) {
					//找到桥接方法，其实还是原来的方法
					Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);
					if (bridgedMethod == specificMethod || metadataLookup.inspect(bridgedMethod) == null) {
						//放到集合当中，后续直接返回
						methodMap.put(specificMethod, result);
					}
				}
			}, ReflectionUtils.USER_DECLARED_METHODS);
		}
		return methodMap;
	}

```

具体的实现细节，获取RequestMapping

```
protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {
		//对给定的[方法,方法级别的]创建一个RequestMappingInfo，第一步是找到RequestMapping注解，如果是null就返回
		//在这里会集中处理RequestMapping注解的信息，我们主要是value对应的uri，和 method，对应了这个
		//method能不能处理这个类型的请求，包括了get，post，put，delete，option，head等等
		//记住这里它获取到了uri，后续要利用这个uri来找到handler，也就是我们的Controller
		RequestMappingInfo info = createRequestMappingInfo(method);
		if (info != null) {
			//一样的处理，这次是类级别的，因为我们可能在method和class上都使用了RequestMapping这个注解
			RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType);
			if (typeInfo != null) {
				//将method 和 class 上获取的RequestMappingInfo 整合起来，uri就变成了class的uri + method的uri，后续会讨论需不需要加  /
				info = typeInfo.combine(info);
			}
		}
		//返回信息
		return info;
	}
```


从这里开始，我们先看看 `doWithMethods` 的内部干了些什么，对给定class和父类(接口或者父接口)并且匹配通过的method，执行给定义的回调方法，父类和子类名字相同的方法如果没有被  MethodFilter排除，就会执行2次
 
 ```java
 public static void doWithMethods(Class<?> clazz, MethodCallback mc, @Nullable MethodFilter mf) {
		// Keep backing up the inheritance hierarchy.  获取这个类的方法，包含了接口的方法
		Method[] methods = getDeclaredMethods(clazz);
		for (Method method : methods) {
			//如果是桥接或者object类的方法，那么继续下一个方法
			if (mf != null && !mf.matches(method)) {
				continue;
			}
			try {
				//执行回调方法，还记得lambda的先定义后调用吗？就在这里出现了，现在我们可以回过头看看它的实现了
				mc.doWith(method);
			}
			catch (IllegalAccessException ex) {
				throw new IllegalStateException("Not allowed to access method '" + method.getName() + "': " + ex);
			}
		}
		if (clazz.getSuperclass() != null) {
			doWithMethods(clazz.getSuperclass(), mc, mf);
		}
		else if (clazz.isInterface()) {
			for (Class<?> superIfc : clazz.getInterfaces()) {
				doWithMethods(superIfc, mc, mf);
			}
		}
	}
 ```

等到获取完method和mapping的映射之后，我们就可以对这个方法和这个方法对映射进行处理了，在内部将其注册成为一个map

```java
methods.forEach((key, mapping) -> {
				//获取调用方法，这里涉及到aop，暂时不做讨论
				Method invocableMethod = AopUtils.selectInvocableMethod(key, userType);
				//核心功能，注册每一个method和Mapping
				registerHandlerMethod(handler, invocableMethod, mapping);
			});
```

下面开始注册这个数据

```java
	protected void registerHandlerMethod(Object handler, Method method, T mapping) {
		//直接调用mappingRegistry这个内部类进行处理
		this.mappingRegistry.register(mapping, handler, method);
	}
```
后续跳转到了`内部类 MappingRegistry`

```java
public void register(T mapping, Object handler, Method method) {
	//注册的使用使用写锁，后期处理http请求的时候是读锁
	this.readWriteLock.writeLock().lock();
	try {
		//使用handler(这里的handler还是一个字符串)和method集合new一个handlerMethod
		HandlerMethod handlerMethod = createHandlerMethod(handler, method);
		//断言，每一个handlerMethod对应一个Mapping
		assertUniqueMethodMapping(handlerMethod, mapping);
		//存放mapping 和 handlermethod的映射，上述的断言使用了这个Map，由这里我们可以猜测必然重写了equals和hashcode方法
		this.mappingLookup.put(mapping, handlerMethod);
		//获取url,这个method使用@requestMapping注解写的value的值，后续处理http请求时可用到
		List<String> directUrls = getDirectUrls(mapping);
		for (String url : directUrls) {
			this.urlLookup.add(url, mapping);
		}
		//获取method + class 组成的名字
		String name = null;
		if (getNamingStrategy() != null) {
			name = getNamingStrategy().getName(handlerMethod, mapping);
			addMappingName(name, handlerMethod);
		}
		//跨越处理 判断method时不是使用CrossOrigin注解，基本上可以跳过，一般都是在FIlter中处理的
		CorsConfiguration corsConfig = initCorsConfiguration(handler, method, mapping);
		if (corsConfig != null) {
			this.corsLookup.put(handlerMethod, corsConfig);
		}
		//直接可以判断他是一个Map类型，因为是map类型的，可以推断mapping是一个【抽象】了某一个事物的类，并且重写equals和hashcode
		//MappingRegistration 其实和holder是一样的，就是和mapping绑定的数据太多，使用一个class集成这些数据
		this.registry.put(mapping, new MappingRegistration<>(mapping, handlerMethod, directUrls, name));
		}
		finally {
			this.readWriteLock.writeLock().unlock();
		}
	}
```

#### 模版模式
这里的模版模式应用主要是在这一部分 `protected abstract T getMappingForMethod(Method method, Class<?> handlerType);` 

模版模式就是在抽象中定义一些抽象方法(模版)，交给具体的子类去实现这个模版，然后返回数据，模版模式还是很简单的，其实难的是在模版如何定义，也就是说，如何将去抽象父类和子类之间的关联关系，我觉得这个才是最重要的，主要明白了这一点，模版模式不攻自破。

谈到模版模式，我认为必然涉及多态这个概念，多态的概念理解的人很好理解，不理解的人有些想不通呢？就是不懂，为什么可以去这么做呢，它觉得这个是不可思议的，(`面向接口编程我觉得也是多态`)，说的通用一点就是大家都是继承的同一个父类，抽象的方法都是通用的，你可以掉用，我也可以调用，举个例子，现在的数据库有很多，db2,mysql,pg,oracle,sqlserver 等等，但是每一家的数据库的内部实现肯定还是有所差别的，但是对于我们程序员来说，就不管你的区别了，用mysql，我在pom文件中加入mysql的依赖，用oracle，我加入oracle的驱动，你用什么我就加什么驱动，前提是你遵循了 `jdbc` 的规范就行，`jdbc` 规范是什么？其实就是一组接口，交给我们上层应用来用的，也就是是，不论是什么，我用DriverManager.getConnection()就可以获取一个Connection而不用管内部实现细节，以后换数据库了，我就换驱动，面向接口编程，jdbc规范不变，我的代码就不变，这个就是我理解的多态了。

模版模式代理-----> [模版模式](https://github.com/chenshun00/pattern/blob/master/src/main/java/com/template/App.java)

后续文章

*	SpringMVC是如何处理http请求的
* 	Spring事务是怎么管理的. 核心事务一览----> [threadLocal 和 事务](https://github.com/chenshun00/work-demo/blob/master/src/main/java/top/huzhurong/demo/transaction/App.java)
*  	Spring aop内部做了些什么？aop 代码可以先看我这里 -----> 代理模式 [cglib 和 jdk](https://git.io/fNNT4)

