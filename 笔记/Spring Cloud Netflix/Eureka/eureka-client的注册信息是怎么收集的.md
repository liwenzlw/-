我们看一下注册信息`EurekaRegistration`

```java
public class EurekaRegistration implements Registration {
    // ...
}

public interface Registration extends ServiceInstance {
    // 什么都没有
}

public interface ServiceInstance {

	default String getInstanceId() {return null;}
	String getServiceId();
	String getHost();
	int getPort();
	boolean isSecure();
	URI getUri();
	Map<String, String> getMetadata();
	default String getScheme() {return null;}
}
```

我们看到`ServiceInstance`里面定义了很多的get方法，那么这些方法的具体实现是从哪里获取的值呢，跟下源码：

```java
public class EurekaRegistration implements Registration {

	private final EurekaClient eurekaClient;

	private final AtomicReference<CloudEurekaClient> cloudEurekaClient = new AtomicReference<>();

	private final CloudEurekaInstanceConfig instanceConfig;

	private final ApplicationInfoManager applicationInfoManager;

	private ObjectProvider<HealthCheckHandler> healthCheckHandler;

	private EurekaRegistration(CloudEurekaInstanceConfig instanceConfig,
			EurekaClient eurekaClient, ApplicationInfoManager applicationInfoManager,
			ObjectProvider<HealthCheckHandler> healthCheckHandler) {
		this.eurekaClient = eurekaClient;
		this.instanceConfig = instanceConfig;
		this.applicationInfoManager = applicationInfoManager;
		this.healthCheckHandler = healthCheckHandler;
	}    
    // ...
}

```

我们先看下在哪里创建的`EurekaRegistration`

```java

public class EurekaClientAutoConfiguration {
    // ...

    @Configuration
	@ConditionalOnMissingRefreshScope
	protected static class EurekaClientConfiguration {
        // ...
		@Bean
		@ConditionalOnBean(AutoServiceRegistrationProperties.class)
		@ConditionalOnProperty(value = "spring.cloud.service-registry.auto-registration.enabled", matchIfMissing = true)
		public EurekaRegistration eurekaRegistration(EurekaClient eurekaClient,
				CloudEurekaInstanceConfig instanceConfig,
				ApplicationInfoManager applicationInfoManager,
				@Autowired(required = false) ObjectProvider<HealthCheckHandler> healthCheckHandler) {
			return EurekaRegistration.builder(instanceConfig).with(applicationInfoManager)
					.with(eurekaClient).with(healthCheckHandler).build();
		}        
    }
}
```
我们先看`eurekaClient`是如何创建的
```java

@ImplementedBy(DiscoveryClient.class)
public interface EurekaClient extends LookupService {
    // ...
}

public class DiscoveryClient implements EurekaClient{
    // ...
}

public class CloudEurekaClient extends DiscoveryClient {
    // ...
}
```
上面代码发现`EurekaClient`接口的最终实现是`CloudEurekaClient`,那么`CloudEurekaClient`又是如何创建的呢，我们找下源码

```java

public class EurekaClientAutoConfiguration {
    // ...

    @Configuration
	@ConditionalOnMissingRefreshScope
	protected static class EurekaClientConfiguration {
        // ...
		@Bean(destroyMethod = "shutdown")
		@ConditionalOnMissingBean(value = EurekaClient.class, search = SearchStrategy.CURRENT)
		public EurekaClient eurekaClient(ApplicationInfoManager manager,
				EurekaClientConfig config) {
			return new CloudEurekaClient(manager, config, this.optionalArgs,
					this.context);
		}
		@Bean
		@ConditionalOnMissingBean(value = ApplicationInfoManager.class, search = SearchStrategy.CURRENT)
		public ApplicationInfoManager eurekaApplicationInfoManager(
				EurekaInstanceConfig config) {
			InstanceInfo instanceInfo = new InstanceInfoFactory().create(config);
			return new ApplicationInfoManager(config, instanceInfo);
		}          
    }
}
```
我们发现`eurekaClient()`依赖于`ApplicationInfoManager`和`EurekaClientConfig`,`ApplicationInfoManager`又会依赖于`EurekaInstanceConfig`。所有创建`EurekaClient`实例最终要依赖于`EurekaClientConfig`和`EurekaInstanceConfig`。

我们先看下`EurekaClientConfig`是如何创建的
```java

public class EurekaClientAutoConfiguration {

    // EurekaClientConfigBean 是 EurekaClientConfig的实现类
	@Bean
	@ConditionalOnMissingBean(value = EurekaClientConfig.class, search = SearchStrategy.CURRENT)
	public EurekaClientConfigBean eurekaClientConfigBean(ConfigurableEnvironment env) {
		EurekaClientConfigBean client = new EurekaClientConfigBean();
		if ("bootstrap".equals(this.env.getProperty("spring.config.name"))) {
			// We don't register during bootstrap by default, but there will be another
			// chance later.
			client.setRegisterWithEureka(false);
		}
		return client;
	}
}
```
为什么只是new了一个对象什么都没有做呢？我们看下`EurekaClientConfigBean`源码

```java
@ConfigurationProperties(EurekaClientConfigBean.PREFIX)
public class EurekaClientConfigBean implements EurekaClientConfig, Ordered {
    // ...
}
```
原来是有`@ConfigurationProperties`注解，Spring会在`EurekaClientConfigBean`注入到容器后自动设置属性。
好了，`EurekaClientConfig`配置解读完了，我们再看下`EurekaInstanceConfig`

```java

public class EurekaClientAutoConfiguration {

	@Bean
	@ConditionalOnMissingBean(value = EurekaInstanceConfig.class, search = SearchStrategy.CURRENT)
	public EurekaInstanceConfigBean eurekaInstanceConfigBean(InetUtils inetUtils,
			ManagementMetadataProvider managementMetadataProvider) {
		String hostname = getProperty("eureka.instance.hostname");
		boolean preferIpAddress = Boolean
				.parseBoolean(getProperty("eureka.instance.prefer-ip-address"));
		String ipAddress = getProperty("eureka.instance.ip-address");
		boolean isSecurePortEnabled = Boolean
				.parseBoolean(getProperty("eureka.instance.secure-port-enabled"));

		String serverContextPath = env.getProperty("server.servlet.context-path", "/");
		int serverPort = Integer
				.valueOf(env.getProperty("server.port", env.getProperty("port", "8080")));

		Integer managementPort = env.getProperty("management.server.port", Integer.class); // nullable.
		// should
		// be
		// wrapped
		// into
		// optional
		String managementContextPath = env
				.getProperty("management.server.servlet.context-path"); // nullable.
																		// should
		// be wrapped into
		// optional
		Integer jmxPort = env.getProperty("com.sun.management.jmxremote.port",
				Integer.class); // nullable
		EurekaInstanceConfigBean instance = new EurekaInstanceConfigBean(inetUtils);

		instance.setNonSecurePort(serverPort);
		instance.setInstanceId(getDefaultInstanceId(env));
		instance.setPreferIpAddress(preferIpAddress);
		instance.setSecurePortEnabled(isSecurePortEnabled);
		if (StringUtils.hasText(ipAddress)) {
			instance.setIpAddress(ipAddress);
		}

		if (isSecurePortEnabled) {
			instance.setSecurePort(serverPort);
		}

		if (StringUtils.hasText(hostname)) {
			instance.setHostname(hostname);
		}
		String statusPageUrlPath = getProperty("eureka.instance.status-page-url-path");
		String healthCheckUrlPath = getProperty("eureka.instance.health-check-url-path");

		if (StringUtils.hasText(statusPageUrlPath)) {
			instance.setStatusPageUrlPath(statusPageUrlPath);
		}
		if (StringUtils.hasText(healthCheckUrlPath)) {
			instance.setHealthCheckUrlPath(healthCheckUrlPath);
		}

		ManagementMetadata metadata = managementMetadataProvider.get(instance, serverPort,
				serverContextPath, managementContextPath, managementPort);

		if (metadata != null) {
			instance.setStatusPageUrl(metadata.getStatusPageUrl());
			instance.setHealthCheckUrl(metadata.getHealthCheckUrl());
			if (instance.isSecurePortEnabled()) {
				instance.setSecureHealthCheckUrl(metadata.getSecureHealthCheckUrl());
			}
			Map<String, String> metadataMap = instance.getMetadataMap();
			metadataMap.computeIfAbsent("management.port",
					k -> String.valueOf(metadata.getManagementPort()));
		}
		else {
			// without the metadata the status and health check URLs will not be set
			// and the status page and health check url paths will not include the
			// context path so set them here
			if (StringUtils.hasText(managementContextPath)) {
				instance.setHealthCheckUrlPath(
						managementContextPath + instance.getHealthCheckUrlPath());
				instance.setStatusPageUrlPath(
						managementContextPath + instance.getStatusPageUrlPath());
			}
		}

		setupJmxPort(instance, jmxPort);
		return instance;
	}
}

@ConfigurationProperties("eureka.instance")
public class EurekaInstanceConfigBean implements CloudEurekaInstanceConfig, EnvironmentAware{
    // ...
}
```



我们先看一下`EurekaClientAutoConfiguration`的类声明

```java

@Configuration
@EnableConfigurationProperties
@ConditionalOnClass(EurekaClientConfig.class)
@Import(DiscoveryClientOptionalArgsConfiguration.class)
@ConditionalOnBean(EurekaDiscoveryClientConfiguration.Marker.class)
@ConditionalOnProperty(value = "eureka.client.enabled", matchIfMissing = true)
@ConditionalOnDiscoveryEnabled
@AutoConfigureBefore({ NoopDiscoveryClientAutoConfiguration.class,
		CommonsClientAutoConfiguration.class, ServiceRegistryAutoConfiguration.class })
@AutoConfigureAfter(name = {
		"org.springframework.cloud.autoconfigure.RefreshAutoConfiguration",
		"org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration",
		"org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationAutoConfiguration" })
public class EurekaClientAutoConfiguration {
    // ...
}
```

我们看下