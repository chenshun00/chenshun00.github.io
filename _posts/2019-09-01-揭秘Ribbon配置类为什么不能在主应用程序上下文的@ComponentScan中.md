### 揭秘Ribbon配置类为什么不能在主应用程序上下文的@ComponentScan中

`SpringCloud` 的中文分享还是比较少的，例如在自定义Ribbon配置中，网上很多文章都会说明 `Ribbon配置类不能在主应用程序上下文的@ComponentScan中` , 否则就会被声明为 `共享` , 但是为什么呢? 却少有人谈. 本文结合实际代码揭秘.

写之前先抛出如下问题

* 一个web应用可以用过多少个 `Spring Context(Spring上下文环境)`

#### 前情提要

1、写过 `SpringBoot` 应用的同学都知道，`SpringBoot` 会自动加载自己包及其子包下边的 `class`, 如果有其他的 `Bean` 不在这个里边，那么就需要通过 `@Configuration` 或者 增加 `@ComponentScan` 的扫描区域.


2、在通常的web应用中，会存在2个 `Spring Context` ， 分别是 

*  `org.springframework.web.servlet.FrameworkServlet.CONTEXT` web子容器.
*  `WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE` ROOT , parent 容器.

在这个基础上，可以针对他们进行拓展，每一个 `key` 单独拓展一个 `Spring Context` ，参考 ==> `AnnotationConfigApplicationContext`

#### 揭秘

前一阵弄 `注册中心时` 需要单独实现一些配置去操作 `Ribbon` 的配置时，必然会出现 [No qualifying bean of type 'com.netflix.client.config.IClientConfig' available](https://github.com/spring-cloud/spring-cloud-kubernetes/issues/81)  . 经过深入了解发现如下结论.

* 每一个 `ServiceId` 都有她专属的 `Spring Context` .

这样就可以解释为什么在 `XxxxxxRibbonClientConfiguration` 获取不到 `com.netflix.client.config.IClientConfig`. 同时也正是因为这个是 `子容器` 通过 `AnnotationConfigApplicationContext` 进行读取的，如果将 `配置放置到到主应用程序上下文的@ComponentScan中`，则会被主应用的上下文读取，最终被所有的 `上下文共享` .

代码配置如下: 分别是

* 通过 `org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.netflix.loadbalancer.RaycloudNacosAutoConfiguration` 引入配置文件， 使用注解的形式进行我失败了. 
* 加载自定义的配置.

```java
@Configuration
@EnableConfigurationProperties
@ConditionalOnBean({SpringClientFactory.class})
@ConditionalOnRibbonNacos
@ConditionalOnNacosDiscoveryEnabled
@AutoConfigureAfter({RibbonAutoConfiguration.class})
@RibbonClients(
        defaultConfiguration = {RaycloudRibbonClientConfiguration.class}
)
public class RaycloudNacosAutoConfiguration {
}
```

config:

```java
@Configuration
public class RaycloudRibbonClientConfiguration {

    @Bean
    @LoadBalanced
    public RestTemplate getRestTemplate() {
        return new RestTemplate();
    }

    @Bean
    public IRule rule(IClientConfig config) {
        RaycloudRule raycloudRule = new RaycloudRule();
        raycloudRule.initWithNiwsConfig(config);
        return raycloudRule;
    }

    @Bean
    @ConditionalOnMissingBean
    public ServerListUpdater ribbonServerListUpdater(IClientConfig config) {
        return new MyPoolUpdater(config);
    }

    /**
     * {@link RibbonClientConfiguration#ribbonLoadBalancer(IClientConfig, ServerList, ServerListFilter, IRule, IPing, ServerListUpdater)}  )}
     */
    @Bean
    @ConditionalOnMissingBean
    public ILoadBalancer ribbonLoadBalancer(IClientConfig config, ServerList serverList, ServerListFilter serverListFilter, IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
        if (this.propertiesFactory.isSet(ILoadBalancer.class, "client")) {
            return this.propertiesFactory.get(ILoadBalancer.class, config, "client");
        }
        return new MyLoad(config, rule, ping, serverList, serverListFilter, serverListUpdater);
    }

    @Autowired
    private PropertiesFactory propertiesFactory;
}
```

没有服务实例时销毁服务的Spring上下文

```java
public class MyPoolUpdater extends PollingServerListUpdater {

    public MyPoolUpdater(IClientConfig clientConfig) {
        super(init(clientConfig));
    }

    public static IClientConfig config = null;

    @Autowired
    private SpringClientFactory springClientFactory;

    private static IClientConfig init(IClientConfig clientConfig) {
        config = clientConfig;
        return clientConfig;
    }

    public MyPoolUpdater(long initialDelayMs, long refreshIntervalMs) {
        super(initialDelayMs, refreshIntervalMs);
    }

    private UpdateAction updateAction;

    @Override
    public synchronized void start(final UpdateAction updateAction) {
        super.start(updateAction);
        this.updateAction = updateAction;
    }

    @Override
    public synchronized void stop() {
        super.stop();
        if (springClientFactory != null) {
            try {
                Field contexts = springClientFactory.getClass().getSuperclass().getDeclaredField("contexts");
                contexts.setAccessible(true);
                Map<String, AnnotationConfigApplicationContext> contextMap = (Map<String, AnnotationConfigApplicationContext>) contexts.get(springClientFactory);
                AnnotationConfigApplicationContext annotationConfigApplicationContext = contextMap.remove(config.getClientName());
                if (annotationConfigApplicationContext != null) {
                    annotationConfigApplicationContext.close();
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    public UpdateAction getUpdateAction() {
        return updateAction;
    }
}
```

#### SpringClientFactory.

>  A factory that creates client, load balancer and client configuration instances. It creates a Spring ApplicationContext per client name, and extracts the beans that it needs from there.

创建 `Context` 的类，English就不解释了. 

为 `ServiceId` 单独创建上下文

```java
protected AnnotationConfigApplicationContext createContext(String name) {
	AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
	if (this.configurations.containsKey(name)) {
		for (Class<?> configuration : this.configurations.get(name)
				.getConfiguration()) {
			context.register(configuration);
		}
	}
	for (Map.Entry<String, C> entry : this.configurations.entrySet()) {
		if (entry.getKey().startsWith("default.")) {
			for (Class<?> configuration : entry.getValue().getConfiguration()) {
				context.register(configuration);
			}
		}
	}
	context.register(PropertyPlaceholderAutoConfiguration.class,
			this.defaultConfigType);
	context.getEnvironment().getPropertySources().addFirst(new MapPropertySource(
			this.propertySourceName,
			Collections.<String, Object> singletonMap(this.propertyName, name)));
	if (this.parent != null) {
		// Uses Environment from parent as well as beans
		context.setParent(this.parent);
	}
	context.setDisplayName(generateDisplayName(name));
	context.refresh();
	return context;
}
```

过程也不解释了， 这里反向列一下源代码的过程. (查阅时记得从下往上)

* RibbonAutoConfiguration#springClientFactory
* RibbonClientConfigurationRegistrar#registerBeanDefinitions
* RibbonClients

看完这部分就可以了解为为什么是通过 `@RibbonClient` 和 `@RibbonClients` 进入导入了. 

### 补充

```java
public class TestMoreContext {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        context.register(Hello.class);
        context.refresh();
        Object bean = context.getBean("good");
        System.out.println("bean:" + bean);

        AnnotationConfigApplicationContext lastContext = new AnnotationConfigApplicationContext();
        lastContext.register(CC.class);
        lastContext.setParent(context);
        lastContext.refresh();

        Date date = lastContext.getBean("last", Date.class);
        System.out.println(date);
        String good = context.getBean("good", String.class);
        System.out.println("good:" + good);
    }
}
```