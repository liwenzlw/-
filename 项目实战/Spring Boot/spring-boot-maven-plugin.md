Spring Boot Maven plugin的5个Goals

spring-boot:repackage，默认goal。在mvn package之后，再次打包可执行的jar/war，同时保留mvn package生成的jar/war为.origin
spring-boot:run，运行Spring Boot应用
spring-boot:start，在mvn integration-test阶段，进行Spring Boot应用生命周期的管理
spring-boot:stop，在mvn integration-test阶段，进行Spring Boot应用生命周期的管理

--------------------- 