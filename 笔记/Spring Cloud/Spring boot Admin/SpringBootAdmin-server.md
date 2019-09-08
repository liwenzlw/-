Spring Boot Admin Server应用程序

依赖：pom.xml

```xml
    <dependencies>
        <!-- @Start Spring boot Admin -->
        <dependency>
            <groupId>de.codecentric</groupId>
            <artifactId>spring-boot-admin-starter-server</artifactId>
            <version>2.0.2</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- @End Spring boot Admin -->
    </dependencies>
```

在启动类上添加`@EnableAdminServer`注解

```java
    @SpringBootApplication
    @EnableAdminServer
    public class ServerApplication {
        public static void main(String[] args) {
            SpringApplication.run(ServerApplication.class, args);
        }
    }
```