
首先我们看一下 `EurekaAutoServiceRegistration`的类声明

```java
public class EurekaAutoServiceRegistration implements AutoServiceRegistration, SmartLifecycle, Ordered, SmartApplicationListener{
    //...
}
```

先看一下`AutoServiceRegistration`

```java
public interface AutoServiceRegistration {

}
```

这个接口很奇怪，里面一个方法都木有，那么它的作用到底是干什么呢，我们跟一下源码发现再`AutoServiceRegistrationAutoConfiguration`类中有引用到，原来，它的作用是在自动配置的时候用来检测是否注入了`AutoServiceRegistration`对象实例，看下源码

```java
@Configuration
@Import(AutoServiceRegistrationConfiguration.class)
@ConditionalOnProperty(value = "spring.cloud.service-registry.auto-registration.enabled", matchIfMissing = true)
public class AutoServiceRegistrationAutoConfiguration {

	@Autowired(required = false)
	private AutoServiceRegistration autoServiceRegistration;

	@Autowired
	private AutoServiceRegistrationProperties properties;

	@PostConstruct
	protected void init() {
		if (this.autoServiceRegistration == null && this.properties.isFailFast()) {
			throw new IllegalStateException("Auto Service Registration has "
					+ "been requested, but there is no AutoServiceRegistration bean");
		}
	}

}
```

再看下`SmartLifecycle`,既然是生命周期，自然就有start, stop, isRunning方法咯

我们看下`EurekaAutoServiceRegistration`是如何实现的

```java

public class EurekaAutoServiceRegistration implements AutoServiceRegistration, SmartLifecycle, Ordered, SmartApplicationListener{
    //...
	@Override
	public void start() {
		// only set the port if the nonSecurePort or securePort is 0 and this.port != 0
		if (this.port.get() != 0) {
			if (this.registration.getNonSecurePort() == 0) {
				this.registration.setNonSecurePort(this.port.get());
			}

			if (this.registration.getSecurePort() == 0 && this.registration.isSecure()) {
				this.registration.setSecurePort(this.port.get());
			}
		}

		// only initialize if nonSecurePort is greater than 0 and it isn't already running
		// because of containerPortInitializer below
		if (!this.running.get() && this.registration.getNonSecurePort() > 0) {

			this.serviceRegistry.register(this.registration); // 开始注册（注册信息是怎么来的呢）

			this.context.publishEvent(new InstanceRegisteredEvent<>(this,
					this.registration.getInstanceConfig())); // 发布事件
			this.running.set(true);
		}
	}

	@Override
	public void stop() {
		this.serviceRegistry.deregister(this.registration); // 取消注册
		this.running.set(false);
	}

	@Override
	public boolean isRunning() {
		return this.running.get();
	}

	@Override
	public void stop(Runnable callback) {
		stop();
		callback.run();
	}
    //...
}
```

那我们的`start` 和 `stop`方法是如何调用的呢，聪明的小伙伴已经发现了还有一个`SmartApplicationListener`。既然是监听器，肯定就会有事件触发咯。我们看下`EurekaAutoServiceRegistration`会被哪些事件触发

```java
public class EurekaAutoServiceRegistration implements AutoServiceRegistration, SmartLifecycle, Ordered, SmartApplicationListener{
    //...
    	@Override
	public boolean supportsEventType(Class<? extends ApplicationEvent> eventType) {
		return WebServerInitializedEvent.class.isAssignableFrom(eventType) // WebServerInitializedEvent 和 ContextClosedEvent事件
				|| ContextClosedEvent.class.isAssignableFrom(eventType);
	}

	@Override
	public void onApplicationEvent(ApplicationEvent event) {
		if (event instanceof WebServerInitializedEvent) {
			onApplicationEvent((WebServerInitializedEvent) event);
		}
		else if (event instanceof ContextClosedEvent) {
			onApplicationEvent((ContextClosedEvent) event);
		}
	}

	public void onApplicationEvent(WebServerInitializedEvent event) {
		// TODO: take SSL into account
		String contextName = event.getApplicationContext().getServerNamespace();
		if (contextName == null || !contextName.equals("management")) {
			int localPort = event.getWebServer().getPort();
			if (this.port.get() == 0) {
				log.info("Updating port to " + localPort);
				this.port.compareAndSet(0, localPort);
				start(); // 开始注册
			}
		}
	}

	public void onApplicationEvent(ContextClosedEvent event) {
		if (event.getApplicationContext() == context) {
			stop(); // 停止注册
		}
	}
    // ...
}
```

思考：客户端注册到Eureka Server上的信息是从哪里获取的呢