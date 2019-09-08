### 一、 [Maven依赖](http://www.jasypt.org/maven.html)

```xml
<properties>
	<jasypt-version>1.9.2</jasypt-version>
</properties>
<dependency>
    <groupId>org.jasypt</groupId>
    <artifactId>jasypt</artifactId>
    <version>${jasypt-version}</version>
    <scope>compile</scope>
</dependency>
```



二、注入



1. 通过配置文件

   ```xml
     <bean id="strongEncryptor"
       class="org.jasypt.encryption.pbe.StandardPBEStringEncryptor">
       <property name="algorithm">
           <value>PBEWithMD5AndTripleDES</value>
       </property>
       <property name="password">
           <value>jasypt</value>
       </property>
     </bean>
   ```

   

2. 通过@Bean

   ```java
   @Configuration
   public class StandardPBEStringEncryptorConfig {
   
       @Bean
       @ConfigurationProperties("jasypt.encryptor")
       public StandardPBEStringEncryptor standardPBEStringEncryptor() {
           return new StandardPBEStringEncryptor();
       }
   }
   
   ```

   

三、应用



