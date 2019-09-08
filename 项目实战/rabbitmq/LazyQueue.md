Lazy Queues

什么时Lazy Queues:
> 尽可能早地将其内容移动到磁盘，并且只在消费者请求时将它们加载到RAM中

设计目标：避免队列消息过多，不断堆积超过限制造成丢失
造成消息堆积的场景：
1. 消费者离线/已经崩溃/正在维修
2. 生产能力大于消费能力

队列消息保存的几种方式：
1. 内存缓存
2. 持久性消息：先写入磁盘，在保存在RAM中

当Rabbitmq认为需要释放内存时，就会将缓存的消息分页保存到磁盘。写磁盘操作需要花费时间并阻止队列进程，此时无法接收新消息。



队列mode可选项：
* "default"
* "lazy"

声明：Lazy Queue

1. 在queue.declare中设置`x-queue-mode`参数 为`lazy`
2. 通过policy

约定：
1. 若同时配置了1和2 则 1的优先级高于2
2. 如果队列没有设置`x-queue-mode` 在默认使用`defalut`

### 实战：


一、在声明队列时，使用x-queue-mode设置mode



```java
  Map<String, Object> args = new HashMap<String, Object>();
  args.put("x-queue-mode", "lazy");
  channel.queueDeclare("myqueue", false, false, false, args);
```

二、通过应用policy，设置lazy Queue
```shell
rabbitmqctl set_policy Lazy "^lazy-queue$" '{"queue-mode":"lazy"}' --apply-to queues
```
三、在运行时改变Queue mode

1. 一个队列，它在声明的时候已经通过`x-queue-mode`设置了mode，如果它想要修改mode，必须先删除，然后再重新声明
2. 如果队列是通过应用policy指定mode的，那么可以在运行时重新指定
   
    ``` shell
    rabbitmqctl set_policy Lazy "^lazy-queue$" '{"queue-mode":"default"}' --apply-to queues
    ```

后续归纳....