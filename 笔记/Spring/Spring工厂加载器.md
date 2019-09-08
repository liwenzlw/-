## SpringFactoriesLoader

从`META-INF/spring.factories`文件中加载工厂类

### 核心方法
* <a name="ac001">加载工厂类名称</a>
  > * 方法签名： static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader)
  > * 系统流程
  >> 1. cache中不存在类加载器为classLoader（如 AutoConfigurationImportSelector.beanClassLoader）的工厂类. AC1
  >> 2. 加载所有的`META-INF/spring.factories`属性文件
  >> 3. 提取所有的key：value 构建Map（value通过/分割成数组）
  >> 4. 将上一步的Map存储到cache中（以classLoader为索引）
  >> 5. 从上一步的map中提取key为factoryClass类型(如 EnableAutoConfiguration.class)的value
  > * 分支流程
  >> * AC1. cache已经存在
  >>> 跳转到 5

* 加载工厂
  > * 方法签名： static <T> List<T> loadFactories(Class<T> factoryClass, @Nullable ClassLoader classLoader)
    > * 系统流程
    >> 1. <a href="ac001">加载工厂类名称</a>
    >> 2. 根据类名称创建实例
    >> 3. 对上一步的结果进行排序（通过@Order）