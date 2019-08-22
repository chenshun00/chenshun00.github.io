### Servlet 路径映射和SpringMvc 路径

#### Servlet 路径映射规范

 *	以 / 结尾的字符串用于路径匹配   pathinfo 匹配
 *	以 *. 开始的字符串用于拓展名
 * 	空字符串是一种特殊的 URL 模式.其精确映射到应用的上下文根，即，http://host:port/<context-root>/请求形式。在这种情况下，路径信息是‘/’且 servlet 路径和上下文路径是空字符串("")。(映射到index页
 *  	只包含“/”字符的字符串表示应用的“default”servlet。在这种情况下，servlet 路径是请求 URL 减去上
下文路径且路径信息是 null。

Servlet 映射规则

*	精准匹配，/test/info ---> 直接匹配这个路径，拥有最高优先级
* 	路径映射(最长路径映射) ----> /* 和 /abc/* ，如果是abc首先就会被/abc/*处理
*  	后缀匹配，*.xxx 的匹配模式。*.json 和 *.do 等等都是一样的模式
*   默认匹配(缺省匹配) / 作为默认的匹配，如果上述都没有匹配命中，那么就会进入这里。

> Servlet 容器默认提供了2个 `Servlet` ，分别是 jspServlet 和 default


```xml
<servlet>
     <servlet-name>jsp</servlet-name>
     <servlet-class>org.apache.jasper.servlet.JspServlet</servlet-class>
     <load-on-startup>3</load-on-startup>
</servlet>
    
<servlet>
     <servlet-name>default</servlet-name>
     <servlet-class>org.apache.catalina.servlets.DefaultServlet</servlet-class>
     <load-on-startup>1</load-on-startup>
</servlet>
 
<!-- The mapping for the default servlet -->
<servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>

<!-- The mappings for the JSP servlet -->
<servlet-mapping>
    <servlet-name>jsp</servlet-name>
    <url-pattern>*.jsp</url-pattern>
    <url-pattern>*.jspx</url-pattern>
</servlet-mapping>    
```

#### SpringMVC 使用形式

  在使用 `SpringMVC` 都使用，我们会使用 `DispatcherServlet` 去作为映射处理，有些可能映射---> /* ，有些可能映射----> / 如果不明白其中的区别，就可能出现一些问题，例如访问jsp，但是页面却把所有的字符都打印出来了。这个时候我们还可能会引入 `<mvc annotation-driver/>` `default-servlet-handler` 这些形式，现在就从原理上理解这是怎么一回事，又是如何出现的。
	上边说道，/* 是路径匹配， / 是默认(缺省匹配) 但是 /* 的优先级比 / 高 ，又低于拓展映射的形式 -----> /xx/xx > /xx/* > /* > *.xxx  > / 
	
如果把`DispatcherServlet` 配置成 `/` ,那么就是默认匹配，这个时候 ，`Servlet 容器` 提供的 `JspServlet` 优先级高于 `DispatcherServlet` 所以jsp请求会被 `JspServlet` 处理，展示正常

如果把`DispatcherServlet` 配置成 `/*` , 那么就是路径匹配，优先级比容器提供的拓展映射要高，所以处理不了 `*.jsp` 的请求
	
从上述的分析就可以明白，他们应该怎么去配置了，那么 SpringMVC 的提供的 `annotation-driver` 和 `default-servlet-handler` 又是怎么一回事呢？
	
这里就不得不说，`handlerMapping` 和 `HandlerAdapter` 的关系，但是和本文关系不打，读者可以自行阅读`SpringMVC 处理流程` 这部分的源代码，都知道配置成 `/*` 就处理不了静态资源了，这个时候就出现  `default-servlet-handler` ，当我们在Springmvc.xml中注册着么一行之后，Spring会注册一个 defaultServlet，然后将请求转发给容器的静态资源处理器`defaultServlet`
	`Annotation-driver` 是什么呢？都知道我们可以使用 `@RequestMapping` 和 `@Controller` 去处理http请求，但是能这么处理的原因是什么呢？ 这个时候就可以引出`RequestMappingHandlerMapping` 这个类了，从名字就能看出他是处理http请求映射的，事实也是如此，他是用来完成  `@RequestMapping` 和 `@Controller`  之间关系的。同时使用`RequestMappingHandlerMapping`  有2中方法去使用
	
*	使用`annotation-dervier`
* 	配置`detectAllHandlerMappings` 为 `true`

> 具体的测试可以自己去试试

上述2种方法都是为了使用这个类，有了这个类再加上对应的`Adapter` 就达成了这个功能。

#### 总结
从这里看出来，其实所有的一切都是围绕 `Servlet 规范` 去实现的，正式因为他们之间处理的优先级，导致了这些问题的出现，而当我们理解完优先级之后就可以有的放矢去处理这些请求，碰到有时404这些诡异的问题，也能很好的分析。
