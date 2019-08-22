### Spring是如何处理http请求的

### 1、 SpingMVC处理流程

*	明确SptingMVC也是通过实现了 `Servlet` 来实现注解处理的
* 熟悉整个 `Servlet` 的处理过程


*	`SpringMVC` 大致处理流程

```text
	HttpServlet的 service方法(也被复写，请求主要是从这里开始) --> doXXX()方法 
	---> 1、doXXX()方法被子类FrameworkServlet复写 
	---> 2、processRequest(req,resp) 处理请求
	---> 3、Spring的尿性，所有实际处理都是以do开头，实际处理doService() 
	---> 4、子类 DispatcherServlet(所谓的前端处理器) 实现
	---> 5、内部doDispatch 处理
	---> 6、判断是不是文件上传 checkMultipart(request)
	---> 7、获取Handler getHandler(processedRequest);，如果handler不存在，404
	---> 8、根据handler获取handlerAdapter
	---> 9、处理Spring拦截器的前置方法
	---> 10、实际处理Controller的方法
	---> 11、处理Spring拦截器的后置方法
	---> 12、处理请求结果
	
	本次分析完毕
```
主要分析是从5开始；

为了方便下文的理解，先看下边的这个方法和 `DispatcherServlet.properties` 文件

```java
	protected void initStrategies(ApplicationContext context) {
		initMultipartResolver(context);//初始化文件上传
		initLocaleResolver(context); 
		initThemeResolver(context);
		initHandlerMappings(context);//初始化handlerMapping，重点
		initHandlerAdapters(context); //初始化handlerAdapter 反射执行invoke(Controller)方法
		initHandlerExceptionResolvers(context);//异常解析器
		initRequestToViewNameTranslator(context);
		initViewResolvers(context);
		initFlashMapManager(context);
	}
```
DispatcherServlet.properties，主要的几个
```java
	org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping
	
	org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
	org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter
	
	org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver
```
`Servlet init` 的时候首先会将 `properties` 里边的key和value读取进来，然后初始化；

这时，我们假设一个 `http 请求` 跨越千山万水，终于到达 `doDispatch()` 方法了。

```java
	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;
		//异步管理，servlet3.1的规范还不是很了解
		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
		ModelAndView mv = null;
		Exception dispatchException = null;
		//检查是不是文件上传
		processedRequest = checkMultipart(request);
		multipartRequestParsed = (processedRequest != request);
		//获取HandlerMapping 重点
		mappedHandler = getHandler(processedRequest);
		if (mappedHandler == null) {
			//如果能够处理该请求的处理器，返回一个404
			noHandlerFound(processedRequest, response);
			return;
		}
		//根据处理器返回一个adapter
		HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
		String method = request.getMethod();
		boolean isGet = "GET".equals(method);
		//处理last-modufied 请求头，还是要看 adapter 支不支持这个请求头
		if (isGet || "HEAD".equals(method)) {
			long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
			//使用缓存
			if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
				return;
			}
		}
		//处理Spring拦截器的前置方法，如果返回的是false，那么!false，不能反射调用Controller方法了
		if (!mappedHandler.applyPreHandle(processedRequest, response)) {
			return;
		}
		//实际处理
		mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
		applyDefaultViewName(processedRequest, mv);
		//拦截器的后置处理
		mappedHandler.applyPostHandle(processedRequest, response, mv);
		//处理结果
		processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		if (mappedHandler != null) {
			mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
		}
		else {
			//处理文件上传的
			if (multipartRequestParsed) {
				cleanupMultipart(processedRequest);
			}
		}
	}
```

1、文件上传处理，从 `DispatcherServlet.properties` 里边我们能知道 `Spring` 是没有字节去处理文件上传的，如果要使用文件上传，那么就需要加入其他的处理
具体的文件上传原理涉及到 input file 的处理，如果想进一步知道，可以看这里----> [http 抓包和解析](https://git.io/fAvvK)
文件上传，要求1、必须是 `POST` 请求；要求2、context-type必须是multipart/开头

2、获取 HandlerMapping
先看类图  [HandlerMapping 类图](/Users/chenshun/Desktop/学习/png/mapping.png)，可以看出，这是一个典型的多级抽象实现，很平常的模板模式，
HandlerMapping的定义很简单，就是获取一个能够处理这个http请求的handler
```
HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception; 核心方法，获取Handler
```
DispatcherServlet 获取 `Handlermapping`
```java
@Nullable
	protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		if (this.handlerMappings != null) {
			 //this.handlerMappings 就是我们上边些的properties里边的配置文件项，5.0.7只有2项了，更加老的版本应该是3个
			for (HandlerMapping hm : this.handlerMappings) {
				//更具http请求获取 HandlerExecutionChain 执行链，这里又涉及到了责任链模式的处理
				HandlerExecutionChain handler = hm.getHandler(request);
				if (handler != null) {
					return handler;
				}
			}
		}
		return null;
	}
```
`HandlerMapping` 的直接实现如下

```
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		Object handler = getHandlerInternal(request); // 重点，根据http请求获取到handler
		if (handler == null) {
			handler = getDefaultHandler();
		}
		if (handler == null) {
			return null;
		}
		//如果handler是字符串，那么从容器中找到这个Bean，obtainApplicationContext()还记得是ApplicationContext()把
		if (handler instanceof String) {
			String handlerName = (String) handler;
			handler = obtainApplicationContext().getBean(handlerName);
		}
		//组装 HandlerExecutionChain ，chain内部就是我们定义的拦截器
		HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
		//跨域处理
		if (CorsUtils.isCorsRequest(request)) {
			CorsConfiguration globalConfig = this.globalCorsConfigSource.getCorsConfiguration(request);
			CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
			CorsConfiguration config = (globalConfig != null ? globalConfig.combine(handlerConfig) : handlerConfig);
			executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
		}
		//返回执行链，后续先执行拦截器的前置方法，再执行具体的http请求方法，再执行后置方法，完成方法
		return executionChain;
	}
```
如何根据http请求获取到handler，看代码我们可以知道它这里是一个抽象方法，交给子类去实现了，子类也很简单，一个是根据方法，一个是根据url，根据我们平常使用的，我们只看前者，如果你喜欢，你都可以自定义自己的handler实现

```
@Override
	protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
		//核心方法，根据http请求得到请求路径 ，  getUrlPathHelper() 是一个UrlPathHelper，路径处理的
		//涉及到Servlet-path，context-path，path-info的处理，不懂这几个，可以回头看看前一篇文章
		String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
		//获取读锁，这里是读写锁，写锁是在写入的时候出现的，不熟悉的可以再去看看前文
		this.mappingRegistry.acquireReadLock();
		//根据请求路径，http请求获取到执行方法HandlerMethod，如果还记得Controller类的处理，那么就应该知道这是什么
		HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
		this.mappingRegistry.releaseReadLock();// 移除了日志，将finally代码块移上来了
		//返回
		return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
	}
```
解析路径，首先是根据path-info处理，如何为null，再根据 servlet路径处理

```
public String getLookupPathForRequest(HttpServletRequest request) {
		// Always use full path within current servlet context? 这里默认是false，可以修改成true，true是根据servlet-path
		if (this.alwaysUseFullPath) {
			return getPathWithinApplication(request);
		}
		//返回servlet映射下的请求路径，使用pathinfo
		String rest = getPathWithinServletMapping(request);
		if (!"".equals(rest)) {
			return rest;
		}
		else {
			return getPathWithinApplication(request);
		}
	}
```
这里就是servlet-path和pathinfo，以及context-path的处理了

```
	public String getPathWithinServletMapping(HttpServletRequest request) {
		//获取pathinfo
		String pathWithinApp = getPathWithinApplication(request);
		//获取servlet路径
		String servletPath = getServletPath(request);
		//移除分号[;]以后的路径，因为http请求是可以带分号的
		String sanitizedPathWithinApp = getSanitizedPath(pathWithinApp);
		String path;
		
		//
		if (servletPath.contains(sanitizedPathWithinApp)) {
			path = getRemainingPath(sanitizedPathWithinApp, servletPath, false);
		}
		else {
			path = getRemainingPath(pathWithinApp, servletPath, false);
		}

		if (path != null) {
			// Normal case: URI contains servlet path.
			return path;
		}
		else {
			String pathInfo = request.getPathInfo();
			if (pathInfo != null) {
				return pathInfo;
			}
			if (!this.urlDecode) 「
				path = getRemainingPath(decodeInternal(request, pathWithinApp), servletPath, false);
				if (path != null) {
					return pathWithinApp;
				}
			}
			return servletPath;
		}
	}
```
 getPathWithinApplication(request);
 
 ```
 	public String getPathWithinApplication(HttpServletRequest request) {
		//不懂可以看我上篇路径懂讲解
		String contextPath = getContextPath(request);
		String requestUri = getRequestUri(request);
		//返回 uri - context 部分
		String path = getRemainingPath(requestUri, contextPath, true);
		if (path != null) {
			return (StringUtils.hasText(path) ? path : "/");
		}
		else {
			return requestUri;
		}
	}
 ```
到这一步，请求路径获取完毕，开始根据路径获取 `handlerMethod`
HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);

```
		protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
		List<Match> matches = new ArrayList<>();
		//根据url获取requestMappingInfo，不懂看controller注册部分
		List<T> directPathMatches = this.mappingRegistry.getMappingsByUrl(lookupPath);
		//添加到matches
		if (directPathMatches != null) {
			//路径匹配
			addMatchingMappings(directPathMatches, matches, request);
		}
		//如果没有匹配中，让全部路径匹配一次
		if (matches.isEmpty()) {
			addMatchingMappings(this.mappingRegistry.getMappings().keySet(), matches, request);
		}
		//如果还是没有匹配中404
		if (!matches.isEmpty()) {
			//排序，选择最佳处理方法,比较Mapping的几个参数，Head，路径，参数等等
			Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));
			matches.sort(comparator);
			Match bestMatch = matches.get(0);
			if (matches.size() > 1) {
				//是不是跨域
				if (CorsUtils.isPreFlightRequest(request)) {
					return PREFLIGHT_AMBIGUOUS_MATCH;
				}
			Match secondBestMatch = matches.get(1);
			//两个方法一致，如果你写了2个相同的方法，启动不会报错，但是执行具体的情况会报错，不知道给谁处理
			if (comparator.compare(bestMatch, secondBestMatch) == 0) {
				Method m1 = bestMatch.handlerMethod.getMethod();
				Method m2 = secondBestMatch.handlerMethod.getMethod();
					throw new IllegalStateException("Ambiguous handler methods mapped for HTTP path '" +request.getRequestURL() + "': {" + m1 + ", " + m2 + "}");
				}
			}
			//
			handleMatch(bestMatch.mapping, lookupPath, request);
			//返回HandlerMethod
			return bestMatch.handlerMethod;
		}
		else {
			return handleNoMatch(this.mappingRegistry.getMappings().keySet(), lookupPath, request);
		}
	}
```
这里 handler 有了，执行的方法也有了，去找一个adapter就可以了，原则也很简单，只要你支持 support 即可，查看adapter的实现结构，最后找到了 `AbstractHandlerMethodAdapter` 也就只有 `RequestMappingHandlerAdapter` 的父类实现了这个接口，其余的要么是不符合，要么是跟不上时代了，例如继承 `controller`

```
@Override
	public final boolean supports(Object handler) {
		return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
	}
```
到这一步，啥都有了，处理缓存的请求头，这个我们不是重点，于是接下来执行拦截器的前置方法

```
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HandlerInterceptor[] interceptors = getInterceptors();
		if (!ObjectUtils.isEmpty(interceptors)) {
			for (int i = 0; i < interceptors.length; i++) {
				HandlerInterceptor interceptor = interceptors[i];
				//一个一个的执行前置方法，看到责任链模式没有，同志们
				if (!interceptor.preHandle(request, response, this.handler)) {
					//提前结束了，触发请求完毕
					triggerAfterCompletion(request, response, null);
					//false
					return false;
				}
				this.interceptorIndex = i;
			}
		}
		return true;
	}
```

这个时候我们可以说说 `Spring` 的拦截器和 `Filter` 的区别了
	
	*	Filter一个是在Servlet前执行，是Servlet的规范
	* 	拦截器是在Servlet里边执行，是Spring的实现

接下来就是实际处理了,内在原理是反射，`method.invoke(object,args)`，后续介绍 `Spring 声明式事务也会讲这里`

```
	mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```
到这一步Spring的处理基本上算上结束了，接下来的就是一些扫尾操作了。

### 总结

结合 `controller` 的注册和 `request` 的处理，我们基本上明白了，其实内在就是路径和handler的映射，只是Spring做到了大而全，就加大了我们理解的难度，不过只要我们抓住它的七寸，理解那至于实现一次都不是问题。

一开始，我很迷惑他的路由`uri --> handler` ，等到我真的理解了的时候，其实本质还是一个Map映射，uri是key，handler是value

责任链模式------> [代码在这里](https://git.io/fAUU8)
