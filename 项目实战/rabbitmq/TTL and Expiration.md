TTL的设置维度：

    1. 设置消息的TTL
    2. 设置队列的TTL (队列多久后会被删除)

## 一、设置消息的TTL

设置方式：

    1. 通过在 operator policy 上设置 `message_ttl` argument
    2. 在声明队列的时候指定 `message_ttl` argument
    3. 在发布消息时指定`expiration `参数值

队列中的消息超过TTL后说明消息已经失效，会被发送到死信队列

TTL参数值约定

    1. 大于等于0
    2. 单位为毫秒



### 实战

方式一： 通过Policy将所有queue的TTL设置为 60s

```shell
rabbitmqctl set_policy TTL ".*" '{"message-ttl":60000}' --apply-to queues
```

方式二：在声明queue的时候指定x-argument参数将所有queue的TTL设置为 60s

```java
Map<String, Object> args = new HashMap<String, Object>();
args.put("x-message-ttl", 60000);
channel.queueDeclare("myqueue", false, false, false, args);
```

方式三：在发布消息时指定`expiration `参数值
```java
byte[] messageBodyBytes = "Hello, world!".getBytes();
AMQP.BasicProperties properties = new AMQP.BasicProperties.Builder()
                                   .expiration("60000")
                                   .build();
channel.basicPublish("my-exchange", "routing-key", properties, messageBodyBytes);
```


注意事项（2，3 待验证）：

1. 消息被requeue，原始的TTL参数值不会发生改变
2. 将TTL的参数值设置为0时，如果消息到达queue没有立即被消费，那么它就会过期并放入到死信队列中。
3. 通过publish Flag `immediate` 可以将消息设置为立即消费，规则为：当immediate标志位设置为true时，如果exchange在将消息route到queue(s)时发现对应的queue上没有消费者，那么这条消息不会放入队列中，并通过basic.return方法返还给生产者。
4. 通过设置`expiration `属性的Message上要到队列的头部的时候才会被丢弃或者时放到死信队列中。一个消息可能在其被写入socket但是到达costumer之前过期。
5. 在Queue上设置`message_ttl`属性时会定时清理队列中的消息


## 二、 设置队列TTL(expire)

设置方式：
1. 在`queue.declare` 协议上设置`x-expires`参数
2. 设置 `expire` policy

队列过期时间的计算规则
1. 队列没有消费者
2. 队列最近没有被重新声明
3. 从上次调用basic.get后进行计时

在经过至少一个`expire`时间周期后，rabbitmq保证，会将该queue删除。服务器重启时，持久化的队列的租赁也会重新计算（待验证）

约定：
1. 计时单位为毫秒

实战：

方式一、使用`policy`设置TTL(expire)
```shell
rabbitmqctl set_policy expiry ".*" '{"expires":1800000}' --apply-to queues
```

方式二、声明队列时通过`x-argument`参数设置
```java
Map<String, Object> args = new HashMap<String, Object>();
args.put("x-expires", 1800000);
channel.queueDeclare("myqueue", false, false, false, args);
```