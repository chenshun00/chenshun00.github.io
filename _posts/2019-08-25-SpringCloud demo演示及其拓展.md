### 从零搭建SpringCloud Demo及其拓展

使用 `SpringCloud Alibaba` + `Zuul` 搭建一个基本的 `SpringCloud` 应用.

目标:

* 1、跑起来
* 2、拓展 `Zuul` 的各种规则
* 3、魔改 `Nacos` 减少 `Http` 调用量.

版本信息:

* 1、SpringBoot 2.0.4.RELEASE
* 2、SpringCloud Finchley.RELEASE
* 3、SpringCloud Alibaba 2.1.0.RELEASE

前提:

* 1、搭建 `Nacos` 集群环境
* 2、配置 `Nacos VIP` 环境

### 搭建demo

首先使用 `Idea` 搭建一个基本的 `Maven` 多模块应用. 可以是 `SpringBoot` , `Spring+Tomcat` , `Spring` 的版本也没有限制.如下所示

```xml
	<modules>
        <module>gateway-spring3</module>
        <module>gateway-spring4</module>
        <module>gateway-spring5</module>
        <module>gateway-springboot-1.x</module>
        <module>gateway-springboot-2.x</module>
        <module>gateway-zuul</module>
        <module>gateway-spring-dubbo</module>
        <module>gateway-springboot-dubbo</module>
        <module>gateway-dubbo-common</module>
    </modules>
```

注册`Spring + Tomcat`,以 `Spring3` 为例子，其他版本同理. 加入Spring3的依赖，加入单独的 `Nacos` 依赖，注册bean. `Nacos` 配置如下

```text
	<dependency>
		<groupId>com.alibaba.nacos</groupId>
		<artifactId>nacos-client</artifactId>
		<version>1.0.1</version>
	</dependency>
```

```text
	<nacos:global-properties endpoint="${nacos.endpoint:127.0.0.1}"/>
```
后配置Conponent,分别添加 `@PostConstruct` 进行服务注册，`@PreDestroy` 进行服务注销，同理可使用 `Spring` 提供的接口进行注册和注销

注册 `SpringBoot 2.x` , 加入 `SpringCloud Alibaba discovery` 依赖,配置 `bootstrap.properties`

```text
	<dependency>
		<groupId>com.alibaba.cloud</groupId>
		<artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
	</dependency>
```

```text
	server.port=8088
	spring.application.name=gateway.springboot.2.x
	spring.cloud.nacos.discovery.endpoint=127.0.0.1
	spring.cloud.nacos.discovery.metadata.path=gateway.springboot.2.x
```

> 因为starter会自动注册服务，所以无须手动调用 `Nacos` 进行注册

注册 `Zuul` ,加入 `SpringCloud Alibaba nacos-config 和 nacos-discovery`, `spring-cloud-starter-netflix-zuul` 依赖，增加 `SpringCloud和Zuul` 启动注解

```java
	@SpringBootApplication
	@EnableDiscoveryClient
	@EnableZuulProxy
	public class ZuulGatewayApplication {
   		public static void main(String[] args) {
       	SpringApplication.run(ZuulGatewayApplication.class);
	  	}
	}
```

为了获取注册到 `Nacos` 中的服务(在 `zuul` 中被称之为`路由(route)`, 需要定时从 `Nacos` 集群中进行全量 `service list` 查询. 然后遍历每一个服务获取到 `服务实例` 返回给 `zuul` 使用.

> 讲述 `zuul` 会称之为 `路由route` , 讲述其他部分会称之为 `服务service`.

参考文章1，通过自定义 `NewZuulRouteLocator` 实现 `RefreshableRouteLocator` 达到动态刷新 `route` 的效果. 加载完 `Nacos` 中的实例数据之后，再将其转换成 `ZuulProperties.ZuulRoute`.

```java
	 // 通过http 请求获取全部的ServiceInstance，如果服务多，那么产生的http请求会更多
    private List<ZuulProperties.ZuulRoute> listenerNacos(List<ServiceInstance> serviceInstanceList) {
        List<ZuulProperties.ZuulRoute> entities = new ArrayList<>();
        for (ServiceInstance serviceInstance : serviceInstanceList) {
            ZuulProperties.ZuulRoute zuulRoute = new ZuulProperties.ZuulRoute();
            zuulRoute.setId(serviceInstance.getServiceId());
            zuulRoute.setServiceId(serviceInstance.getServiceId());
            zuulRoute.setPath("/" + serviceInstance.getServiceId() + "/**");
            ConfigurationManager.getConfigInstance().setProperty("hystrix.command." + zuulRoute.getServiceId() + ".execution.isolation.thread.timeoutInMilliseconds", serviceInstance.getMetadata().getOrDefault("timeout", "3000"));
        }
        return entities;
    }
```

分别启动 `Nacos` 集群. `Spring+Tomcat` 应用, `SpringBoot` 应用 , 执行以下 `curl` 确定服务是否已经注册上去.(⚠️注意替换服务名)

```bash
curl -X GET 127.0.0.1:8848/nacos/v1/ns/instance/list?serviceName=nacos.test.1 
```

观察到如下结果显示注册成功, 注册失败可查看启动日志是否正常.

```text
➜ curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/service/list?pageNo=1&pageSize=100'
{"count":1,"doms":["gateway.spring3"]}%

```

验证网关功能是否可用
分别执行如下 `curl` 操作即可验证

```bash
➜  ~ curl 'http://127.0.0.1:9912/zuul/gateway.spring3/gateway/spring3.json'
```

#### 拓展 `zuul` 的配置

`Zuul` 的配置是通过 [RibbonClientConfiguration]() 配置类进行引入的，通过该配置，每一个服务都可以拥有一个 `IClientConfig` `IRule` 等等。 这一步通过 `@RibbonClients` 对这些配置类进行覆盖.

* 定义配置类

```java
	@Configuration
	@EnableConfigurationProperties
	@ConditionalOnBean({SpringClientFactory.class})
	@ConditionalOnRibbonNacos
	@ConditionalOnNacosDiscoveryEnabled
	@AutoConfigureAfter({RibbonAutoConfiguration.class})
	@RibbonClients(defaultConfiguration = {DemoRibbonClientConfiguration.class})
	public class DemoNacosAutoConfiguration {}
```
	
实际配置类
	
```java
	@Configuration
	public class DemoRibbonClientConfiguration {

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
这里分别覆盖了 `IRule` `ServerListUpdater` `ILoadBalancer` ， 可以根据自己的需求分别进行处理
	
* 引入配置文件 META-INF/spring.factories

```text
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.netflix.loadbalancer.DemoNacosAutoConfiguration
```
* 自定义 `Bean` 进行覆盖

这里介绍下为什么引入 `ServerListUpdater` , 先看代码.

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

核心就在 `stop` 中，在 `stop` 的时候，将该服务对应 `Spring Context` 进行销毁.

### 魔改 `Nacos` 减少 `Http` 调用量

在上述代码中，可以看到，为了获取全部的 `ServiceInstance` 需要对 `Nacos` 的服务进行 `遍历`. 如果服务有上千个，这样会生成上千个http请求. 因此我们可以通过 `Memcache` `Redis` 进行同步，两端比较 `Service` 的 `checksum` , 前提是 `Zuul` 和 `Nacos` 共享一个 `缓存集群` .

* Nacos端进行改造，在 `Service#onChange` 时进行 `checksum` 比较，如果 `checksum` 不一致，那么可以可以将 `checksum` 存入 `缓存`, 同时将服务的 `实例明细` 存入 `缓存` 即可.
  
```java
try {
	String result;
	try {
		MessageDigest md5 = MessageDigest.getInstance("MD5");
		result = new BigInteger(1, md5.digest((ipsString.toString()).getBytes(Charset.forName("UTF-8")))).toString(16);
	} catch (Exception e) {
		Loggers.SRV_LOG.error("[NACOS-DOM] error while calculating checksum(md5)", e);
		result = RandomStringUtils.randomAscii(32);
	}
	checksum = result;
	//魔改之后的代码
	if (!oldCheckSum.equalsIgnoreCase(result)) {
		MemcachedClient memcachedClient = memcachedClient();
		memcachedClient.set(KeyBuilder.buildServiceMetaKey(getNamespaceId(), getName().split("@@")[1]), 3600, checksum);
		try {
			System.out.println("写入缓冲明细:" + JSONObject.toJSONString(covertTo(ips)));
			memcachedClient.set(KeyBuilder.buildServiceMetaKey(getNamespaceId(), getName().split("@@")[1]) + "_detail", 3600, JSONObject.toJSONString(covertTo(ips)));
		} catch (Throwable e) {
			e.printStackTrace();
		}
	}
} catch (Exception e) {
	Loggers.SRV_LOG.error("[NACOS-DOM] error while calculating checksum(md5)", e);
	checksum = RandomStringUtils.randomAscii(32);
}
```
  
* `Zuul` 端对 `NewZuulRouteLocator` 进行改造，启动一个本地定时任务 . 定时做 `Nacos` 全量同步(可以事件跨度放大一点，例如1分钟)，定时比较 `checksum` ，发现不一致进行替换. 代码如下:

```java
@Service
@Slf4j
public class NacosTask implements InitializingBean {
    /**
     * namespace <===> 服务 <===> namespace很少
     */
    public static Map<String, Set<String>> NAMESPACE_SERVER_SET = new ConcurrentHashMap<>(8);
    /**
     * namespace <===>服务 <===> checksum
     */
    public static Map<String, Map<String, String>> CHECKSUM_MAP = new ConcurrentHashMap<>(8);

    /**
     * namespace <===> 服务 <===> 实例集合
     */
    public static Map<String, Map<String, Instances>> INSTANCES_MAP = new ConcurrentHashMap<>(8);

    private final static String DEFAULT_NAMESPACE = "public";

    static {
        NAMESPACE_SERVER_SET.put(DEFAULT_NAMESPACE, new CopyOnWriteArraySet<>());
        CHECKSUM_MAP.put(DEFAULT_NAMESPACE, new ConcurrentHashMap<>());
        INSTANCES_MAP.put(DEFAULT_NAMESPACE, new ConcurrentHashMap<>());
    }

    @Resource
    private MemcachedClient memcachedClient;
    @Resource
    private DiscoveryClient compositeDiscoveryClient;
    @Resource
    private NacosDiscoveryProperties nacosProperties;

    private NamingProxy namingProxy;

    public NamingProxy getNamingProxy() {
        if (namingProxy != null) return namingProxy;
        NamingProxy namingProxy;
        try {
            NacosNamingService namingService = (NacosNamingService) nacosProperties.namingServiceInstance();
            Field serverProxy = namingService.getClass().getDeclaredField("serverProxy");
            serverProxy.setAccessible(true);
            namingProxy = (NamingProxy) serverProxy.get(namingService);
        } catch (NoSuchFieldException | IllegalAccessException e) {
            throw new RuntimeException(e);
        }
        this.namingProxy = namingProxy;
        return this.namingProxy;
    }

    @Override
    public void afterPropertiesSet() {
        executorService.scheduleWithFixedDelay(() -> {
            //获取服务集合===>ServiceController(/nacos/v1/ns/service/list)==>ketSet
            List<String> services = compositeDiscoveryClient.getServices();
            log.debug("[start handle sche task][services:{}][services:{}]", services.size(), JSONArray.toJSON(services));
            NAMESPACE_SERVER_SET.get(DEFAULT_NAMESPACE).addAll(services);
            List<String> removeKey = new ArrayList<>();
            //s===>gateway.spring5 coreKey===> com.alibaba.nacos.naming.domains.meta.public##gateway.spring5
            for (String s : NAMESPACE_SERVER_SET.get(DEFAULT_NAMESPACE)) {
                try {
                    String coreKey = KeyBuilder.buildServiceMetaKey(DEFAULT_NAMESPACE, s);
                    String remoteCheckSum = memcachedClient.get(coreKey);
                    log.debug("[start handle][key:{}][remoteCheckSum:{}]", coreKey, remoteCheckSum);
                    //如果不为空
                    if (!StringUtils.isEmpty(remoteCheckSum)) {
                        log.info("[checksum changed][service:{}]", s);
                        //比较是否一致
                        Map<String, String> checksumMap = CHECKSUM_MAP.getOrDefault(coreKey, new ConcurrentHashMap<>());
                        String localCheckSum = checksumMap.getOrDefault(coreKey, "d");
                        if (!localCheckSum.equalsIgnoreCase(remoteCheckSum)) {
                            checksumMap.put(coreKey, remoteCheckSum);
                            //拉明细,解析明细直接覆盖
                            String detail = memcachedClient.get(coreKey + "_detail");
                            if (detail != null) {
                                INSTANCES_MAP.getOrDefault(DEFAULT_NAMESPACE, new ConcurrentHashMap<>()).put(coreKey, new Instances(JSONArray.parseArray(detail, Instance.class)));
                            }
                        }
                    } else {
                        removeKey.add(s);
                        removeKey.add(coreKey);
                    }
                } catch (Throwable e) {
                    e.printStackTrace();
                }
            }
            //memcache挂掉不会跑到这里来
            for (String s : removeKey) {
                NAMESPACE_SERVER_SET.getOrDefault(DEFAULT_NAMESPACE, new CopyOnWriteArraySet<>()).remove(s);
                INSTANCES_MAP.getOrDefault(DEFAULT_NAMESPACE, new ConcurrentHashMap<>()).remove(s);
                CHECKSUM_MAP.getOrDefault(DEFAULT_NAMESPACE, new ConcurrentHashMap<>()).remove(s);
            }
        }, 0, 5, TimeUnit.SECONDS);
        ALL_SERVICE.scheduleWithFixedDelay(this::handleAllData, 0, 1, TimeUnit.MINUTES);
    }

    private void handleAllData() {
        try {
            String allFromNacos = getAllFromNacos();
            Map<String, Datum> stringDatumMap = PropertiesAssemble.deserializeMap(allFromNacos);
            if (stringDatumMap != null && stringDatumMap.size() > 0) {
                log.debug("[handle all sync data from nacos][size:{}]", stringDatumMap.size());
                stringDatumMap.forEach((key, value) -> {
                    String serviceName = KeyBuilder.getServiceName(key);
                    String namespaceId = KeyBuilder.getNamespace(key);
                    Instances instances = value.instances;
                    String coreKey = KeyBuilder.buildServiceMetaKey(namespaceId, serviceName.split("@@")[1]);
                    INSTANCES_MAP.getOrDefault(namespaceId, new ConcurrentHashMap<>()).put(coreKey, instances);
                });
            } else {
                log.warn("[sync data from nacos is empty]");
            }
        } catch (Exception e) {
            log.error("[sync nacos all data occur exception][error:{}]", e.getMessage(), e);
        }
    }

    private String getAllFromNacos() throws Exception {
        NamingProxy namingProxy = getNamingProxy();
        Assert.notNull(namingProxy, "namingProxy can't be null");
        Exception e = null;
        List<String> serverListFromEndpoint = namingProxy.getServerListFromEndpoint();
        for (String server : serverListFromEndpoint) {
            try {
                return getAllData(server);
            } catch (Exception ex) {
                e = ex;
            }
        }
        throw e == null ? new RuntimeException("获取nacos全量数据失败") : e;
    }

    private static String getAllData(String server) throws Exception {
        HttpClient.HttpResult result = HttpClient.httpGet(String.format("http://%s/nacos/v1/ns/distro/datums", server),
                new ArrayList<>(), new HashMap<>(8), "utf-8");

        if (HttpURLConnection.HTTP_OK == result.code) {
            return result.content;
        }
        throw new IOException("failed to req API: " + String.format("http://%s/nacos/v1/ns/distro/datums", server) + ". code: " + result.code + " msg: " + result.content);
    }

    private ScheduledExecutorService executorService = Executors.newSingleThreadScheduledExecutor(r -> {
        Thread thread = new Thread(r);
        thread.setName("scan-nacos");
        thread.setDaemon(true);
        return thread;
    });


    private ScheduledExecutorService ALL_SERVICE = Executors.newSingleThreadScheduledExecutor(r -> {
        Thread thread = new Thread(r);
        thread.setName("all_in_scan");
        thread.setDaemon(true);
        return thread;
    });
```

参考文章:

[spring cloud zuul使用记录（2）路由接入流程以及并发刷新问题](https://www.jianshu.com/p/eab9e2f9fbb6)
