### 问题
最近在学习SpringCloud , 以及将要在公司内部部署和推广的过程中，发现网关既需要支持 `http` ，同时也需要支持 `dubbo`，并且网关只需要支持http即可，那么在网关的内部就需要将http协议转换成dubbo协议，在内部做又有2个处理方式
    
  *   1、在网关层面处理
      * 优点
           *   直接利用dubbo的泛化功能
           *   服务提供者不需要进行额外的处理
       * 缺点
           *   在网关层需要进行dubbo的tcp连接，如果业务的网络环境比较特殊，那么这一套是较难维护的
           *   接口的交互较为复杂，泛化需要将参数类型，参数等等进行传递，而这些服务提供者本身其实是存在的。
    *   2、在dubbo#provider层面进行处理
        * 优点
           *   直接对接http协议，不必处理额外的网络环境
           *   仅需要传递dubbo服务需要的参数，不必传递额外的参数类型
        * 缺点
           *   如何让服务提供者支持http转dubbo.
    

通过上述的比较，以及公司业务上处理，我们选择了第二种进行处理.
    
### 开发

一开始我们的服务是通过tomcat或者内置容器的SpringBoot进行暴露的，如果访问dubbo的话，过程就是 `http --> nginx ---> tomcat ---> springmvc ---> dubbo` 这个过程，而现在我们需要做的就是将这个过程中的SpringMVC这一块进行移除，变成 `http --> nginx ---> tomcat ---> dubbo`，从而直接支持http被dubbo处理。于是我通过SpringMVC的处理机制将Controller这一块移除掉，达到了我们的目的，接下来看如何一步一步实现的.
    
> 假设url = `/dubbo/*`

   *    通过包装Servlet统一处理对接的http。

```text
    <servlet>
        <servlet-name>GatewayServlet</servlet-name>
        <servlet-class>com.xxx.gateway.dubbo.web.GatewayServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>GatewayServlet</servlet-name>
        <url-pattern>/dubbo/*</url-pattern>
    </servlet-mapping>    
```

   *    通过http参数获取dubbo服务的接口，版本等等
   
```text
     String inf = servletRequest.getParameter("serviceName");
     Assert.notNull(inf, "接口不能为空!");
     String parameter = servletRequest.getParameter("method");
     Assert.notNull(inf, "方法不能为空!");
     String uGroup = servletRequest.getParameter("group");
     String vVersion = servletRequest.getParameter("version");
     Assert.notNull(vVersion, "版本不能为空!");
```
    
   * 获取dubbo服务

```text
    String[] beanNamesForType = this.applicationContext.getBeanNamesForType(ServiceConfig.class);
    Object ref = null;
    for (String service : beanNamesForType) {
        ServiceConfig serviceConfig = (ServiceConfig) this.applicationContext.getBean(service);
        String version = serviceConfig.getVersion();
        String anInterface = serviceConfig.getInterface();
        String group = serviceConfig.getGroup();
        if (!inf.equalsIgnoreCase(anInterface)) {
            continue;
        }
        if (!vVersion.equalsIgnoreCase(version)) {
            continue;
        }
        if (uGroup != null && !group.equalsIgnoreCase(uGroup)) {
            continue;
        }
        ref = serviceConfig.getRef();
        break;
    }
```

  * 利用SpringMVC的的处理机制将Controller移除
   
```text
    try {
            HttpServletRequest req = (HttpServletRequest) servletRequest;
            HttpServletResponse resp = (HttpServletResponse) servletResponse;
            ServletInvocableHandlerMethod invocableMethod = new ServletInvocableHandlerMethod(handlerMethod);
            WebDataBinderFactory binderFactory = getDataBinderFactory(invocableMethod);

            invocableMethod.setDataBinderFactory(binderFactory);
            ServletWebRequest webRequest = new ServletWebRequest(req, resp);
            ModelAndViewContainer mavContainer = new ModelAndViewContainer();

            HandlerMethodArgumentResolverComposite handlerMethodArgumentResolverComposite = new HandlerMethodArgumentResolverComposite();
            handlerMethodArgumentResolverComposite.addResolvers(getDefaultArgumentResolvers());
            invocableMethod.setHandlerMethodArgumentResolvers(handlerMethodArgumentResolverComposite);

            List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
            HandlerMethodReturnValueHandlerComposite returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
            invocableMethod.setHandlerMethodReturnValueHandlers(returnValueHandlers);

            invocableMethod.invokeAndHandle(webRequest, mavContainer);
            methodMap.put(handlerMethod, invocableMethod);
        } catch (Exception e) {
            fail(servletResponse, e);
        }  
```

 * 如何突破传递参数的问题。
        利用SpringMVC的 `HandlerMethodArgumentResolver` 即可解析
    
    到这一步，http转dubbo就处理好了，测试发现SpringMVC在3，4，5的几个大版本中稍有变动，兼容花费一点时间。后续跟新会上传的github。 :)

### 结论

 Spring很强大.