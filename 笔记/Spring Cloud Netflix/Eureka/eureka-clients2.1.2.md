# 1. Eureka 服务发现 之 Eureka Clients




## 1.1. 在项目中引入 Eureka Client

```xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
```

```java

    @SpringBootApplication
    @EnableDiscoveryClient
    public class Application {
        public static void main(String[] args) {
            SpringApplication.run(Application.class, args);
        }
    }

```

## 1.2. 使用 Eureka 进行服务注册（Eureka Clients 配置）

引入`spring-cloud-starter-netflix-eureka-client`之后发生的变化：

  * 项目会变成一个Eureka “instance”，自动将自己注册到eureka server 上，默认的注册地址为 `http://localhost:8761/eureka/` 
  * 项目会变成一个 Eureka “client” （会自动查询已注册的服务清单，以便定位其他服务）

Eureka instance 行为由 `eureka.instance.*`配置项驱动。如果我们配置了`spring.application.name`,那么使用 `eureka.instance.*`默认值也很好。

从`Environment`中获取到的Service ID、virtual host、non-secure port 分别对应着`${spring.application.name}`, `${spring.application.name}` 和 `${server.port}` 

详细的配置项可以查看 `EurekaInstanceConfigBean` 和 `EurekaClientConfigBean`


我看看下配置示例：

```yml
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

如何禁用Eureka Client

`eureka.client.enabled: false`
或
`spring.cloud.discovery.enabled: false`


## 1.3. 在 Eureka Server 端进行身份验证

如果Eureka Server 启用了身份验证，Eureka Client 可以在defaultZone中使用带有身份信息的的url
`http://user:password@localhost:8761/eureka`进行认证。
如果需要更加的灵活，可以使用向Spring 容器中注入 `DiscoveryClientOptionalArgs `类型的实例，在该实例中添加`ClientFilter `,client调用server的时候会应用 `ClientFilter`中的信息

示例：

```java
import com.netflix.discovery.DiscoveryClient;
import com.sun.jersey.api.client.filter.ClientFilter;
import com.sun.jersey.api.client.filter.HTTPBasicAuthFilter;
import org.springframework.beans.factory.annotation.Configurable;
import org.springframework.context.annotation.Bean;

import java.util.ArrayList;
import java.util.Collection;

@Configurable
public class DiscoveryClientOptionalArgsConfig {

    @Bean
    public DiscoveryClient.DiscoveryClientOptionalArgs DiscoveryClientOptionalArgs() {
        DiscoveryClient.DiscoveryClientOptionalArgs discoveryClientOptionalArgs = new DiscoveryClient.DiscoveryClientOptionalArgs();
        HTTPBasicAuthFilter httpBasicAuthFilter = new HTTPBasicAuthFilter("root","123456");
        Collection<ClientFilter> list = new ArrayList<>(1);
        list.add(httpBasicAuthFilter);
        discoveryClientOptionalArgs.setAdditionalFilters(list);
        return discoveryClientOptionalArgs;
    }
}
```

需要显示引入 jersey-client 否则找不到 ClientFilter类
```xml
    <dependency>
        <groupId>com.sun.jersey</groupId>
        <artifactId>jersey-client</artifactId>
        <version>1.19.1</version>
    </dependency>
```
ClientFilter 类实现如下：

![1](./eureka-clients2.1.2.asserts/1.png)


## 1.4. Status 信息和健康指标

Eureka 实例通过`spring-boot-starter-actuator`提供了状态信息和健康指标访问的默认接口`/actuator/info` 和`/actuator/health`

如何修改默认访问接口：
```yml
eureka:
  instance:
    statusPageUrlPath: ${server.servletPath}/info
    healthCheckUrlPath: ${server.servletPath}/health
```

## 1.5. HTTPS 通信的 Application

一、 访问 HTTPS 的服务提供方

如果服务提供方是 基于 HTTPS 通信 需要在服务端设置`EurekaInstanceConfigBean`的两个配置项
```yml
eureka.instance.nonSecurePortEnabled=false
eureka.instance.securePortEnabled=true
```
这样的话，所有的client在调用服务方的时候都会将协议转换成 HTTPS。
我们查看一下相关源码
![1](./eureka-clients2.1.2.asserts/2.png)

二、因为Eureka是在内网工作的，所以Eureka 的Actuator 接口默认启用的是HTTP协议，如果使用HTTPS需要显式指定，如下：

```yml
eureka:
  instance:
    statusPageUrl: https://${eureka.hostname}/info
    healthCheckUrl: https://${eureka.hostname}/health
    homePageUrl: https://${eureka.hostname}/
```

## 1.6. Eureka的健康检测

Eureka默认使用的是客户端心跳检测包来判断Eureka 客户端是否启用。如果没有指定`eureka.client.healthcheck.enabled:true`客户端只在启动的时候会将自己注册到eureka server,后续不再发送心跳包。

application.yml
```yml
eureka:
  client:
    healthcheck:
      enabled: true
```
注意：`eureka.client.healthcheck.enabled=true`不能设置在`bootstrap.yml`文件中

如果需要细粒度的控制health checks可以实现`com.netflix.appinfo.HealthCheckHandler`接口

## 1.7. Eureka 客户端示例的 Metadata (元数据)

## 1.8. 使用 Eureka 客户端

## 1.9. Native Netflix EurekaClient 的替代品

## 1.10. 为什么注册服务这么慢？

## 1.11. Zones

## 1.12. 刷新 Eureka Clients

