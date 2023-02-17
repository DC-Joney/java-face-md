# RabbitMQ

首先简单介绍 RabbitMQ 的运行原理，在 RabbitMQ 使用时，系统会先安装并启动 Broker Server，也就是 RabbitMQ 的服务中心。无论是生产者 （Producer），消费者（Consumer）都会通过连接池（Connection）使用 TCP/IP 协议（默认）来与 BrokerServer 进行连接。然后 Producer 会把 Exchange / Queue 的绑定信息发送到 Broker Server，Broker Server 根据 Exchange 的类型逻辑选择对应 Queue ，最后把信息发送到与 Queue 关联的对应 Consumer 。

![img](https://img2018.cnblogs.com/blog/64989/201907/64989-20190724180708501-1890880561.jpg)



## Exchange、Queue

### **2.2.1 交换器 Exchange**

Producer 建立连接后，并非直接将消息投递到队列 Queue 中，而是把消息发送到交换器 Exchange，由 Exchange 根据不同逻辑把消息发送到一个或多个对应的队列当中。目前 Exchange 提供了四种不同的常用类型：Fanout、Direct、Topic、Header。

- **Fanout类型**

此类型是最为常见的交换器，它会将消息转发给所有与之绑定的队列上。比如，有N个队列与 Fanout 交换器绑定，当产生一条消息时，Exchange 会将该消息的N个副本分别发给每个队列，类似于广播机制。

- **Direct类型**

此类型的 Exchange 会把消息发送到 Routing_Key 完全相等的队列当中。多个 Cousumer 可以使用相同的关键字进行绑定，类似于数据库的一对多关系。比如，Producer 以 Direct 类型的 Exchange 推送 Routing_Key 为 direct.key1 的队列，系统再指定多个 Cousumer 绑定 direct.key1。如此，消息就会被分发至多个不同的 Cousumer 当中。

- **Topic类型**

此类型是最灵活的一种方式配置方式，它可以使用模糊匹配，根据 Routing_Key 绑定到包含该关键字的不同队列中。比如，Producer 使用 Topic类型的 Exchange 分别推送 Routing_Key 设置为 topic.guangdong.guangzhou 、topic.guangdong.shenzhen 的不同队列，Cousumer 只需要把 Routing_Key 设置为 topic.guangdong.# ，就可以把所有消息接收处理。

- **Headers类型**

该类型的交换器与前面介绍的稍有不同，它不再是基于关键字 Routing_Key 进行路由，而是基于多个属性进行路由的，这些属性比路由关键字更容易表示为消息的头。也就是说，用于路由的属性是取自于消息 Header 属性，当消息 Header 的值与队列绑定时指定的值相同时，消息就会路由至相应的队列中。

### **2.2.2 Queue 队列**

Queue 队列是消息的载体，每个消息都会被投入到 Queue 当中，它包含 name，durable，arguments 等多个属性，name 用于定义它的名称，当 durable（持久化）为 true 时，队列将会持久化保存到硬盘上。反之为 false 时，一旦 Broker Server 被重启，对应的队列就会消失，后面还会有例子作详细介绍。

### **2.2.3 Channel 通道**

当 Broker Server 使用 Connection 连接 Producer / Cousumer 时会使用到信道（Channel），一个 Connection上可以建立多个 Channel，每个 Channel 都有一个会话任务，可以理解为逻辑上的连接。主要用作管理相关的参数定义，发送消息，获取消息，事务处理等。

### **2.2.4 Binding 绑定**

Binding 主要用于绑定交换器 Exchange 与 队列 Queue 之间的对应关系，并记录路由的 Routing-Key。Binding 信息会保存到系统当中，用于 Broker Server 信息的分发依据。



## Queue 参数

https://www.cnblogs.com/refuge/p/10354579.html

- x-message-ttl：Number ，发布的消息在队列中存在多长时间后被取消（单位毫秒） 可以对单个消息设置过期时间
- x-expires：Number

​		当Queue（队列）在指定的时间未被访问，则队列将被自动删除。

- x-max-length：Number

​	队列所能容下消息的最大长度。当超出长度后，新消息将会覆盖最前面的消息，类似于Redis的LRU算法。

- x-max-length-bytes：Number

限定队列的最大占用空间，当超出后也使用类似于Redis的LRU算法。

- x-overflow：String

设置队列溢出行为。这决定了当达到队列的最大长度时，消息会发生什么。有效值为Drop Head或Reject Publish。

- x-dead-letter-exchange：String
   如果消息被拒绝或过期或者超出max，将向其重新发布邮件的交换的可选名称
- x-dead-letter-routing-key：String

如果不定义，则默认为溢出队列的routing-key，因此，一般和6一起定义。

- x-max-priority：Number

如果将一个队列加上优先级参数，那么该队列为优先级队列。

1）、给队列加上优先级参数使其成为优先级队列

x-max-priority=10【值不要太大，本质是一个树结构】

2）、给消息加上优先级属性

- x-queue-mode：String

队列类型　　x-queue-mode=lazy　　懒队列，在磁盘上尽可能多地保留消息以减少RAM使用；如果未设置，则队列将保留内存缓存以尽可能快地传递消息。

- x-queue-master-locator：String

将队列设置为主位置模式，确定在节点集群上声明时队列主位置所依据的规则。





## Exchange 参数



## 死信队列的作用

https://mfrank2016.github.io/breeze-blog/2020/05/04/rabbitmq/rabbitmq-how-to-use-dead-letter-queue/

## 一、说明

RabbitMQ是流行的开源消息队列系统，使用erlang语言开发，由于其社区活跃度高，维护更新较快，性能稳定，深得很多企业的欢心（当然，也包括我现在所在公司【手动滑稽】）。

为了保证订单业务的消息数据不丢失，需要使用到RabbitMQ的死信队列机制，当消息消费发生异常时，将消息投入死信队列中。但由于对死信队列的概念及配置不熟悉，导致曾一度陷入百度的汪洋大海，无法自拔，很多文章都看起来可行，但是实际上却并不能帮我解决实际问题。最终，在官网文档中找到了我想要的答案，通过官网文档的学习，才发现对于死信队列存在一些误解，导致配置死信队列之路困难重重。

于是本着记录和分享的精神，将死信队列的概念和配置完整的写下来，以便帮助遇到同样问题的朋友。

## 二、本文大纲

以下是本文大纲：



![AG4T332}_NEUNPUAU(A_)U6.png](https://i.loli.net/2019/07/14/5d2af4176d1d483480.png)

**AG4T332}_NEUNPUAU(A_)U6.png**



本文阅读前，需要对RabbitMQ有一个简单的了解，偏向实战配置讲解。

## 三、死信队列是什么

死信，在官网中对应的单词为“Dead Letter”，可以看出翻译确实非常的简单粗暴。那么死信是个什么东西呢？

“死信”是RabbitMQ中的一种消息机制，当你在消费消息时，如果队列里的消息出现以下情况：

1. 消息被否定确认，使用 `channel.basicNack` 或 `channel.basicReject` ，并且此时`requeue` 属性被设置为`false`。
2. 消息在队列的存活时间超过设置的TTL时间。
3. 消息队列的消息数量已经超过最大队列长度。

那么该消息将成为“死信”。

“死信”消息会被RabbitMQ进行特殊处理，如果配置了死信队列信息，那么该消息将会被丢进死信队列中，如果没有配置，则该消息将会被丢弃。

## 四、如何配置死信队列

这一部分将是本文的关键，如何配置死信队列呢？其实很简单，大概可以分为以下步骤：

1. 配置业务队列，绑定到业务交换机上
2. 为业务队列配置死信交换机和路由key
3. 为死信交换机配置死信队列

注意，并不是直接声明一个公共的死信队列，然后所以死信消息就自己跑到死信队列里去了。而是为每个需要使用死信的业务队列配置一个死信交换机，这里同一个项目的死信交换机可以共用一个，然后为每个业务队列分配一个单独的路由key。

有了死信交换机和路由key后，接下来，就像配置业务队列一样，配置死信队列，然后绑定在死信交换机上。也就是说，死信队列并不是什么特殊的队列，只不过是绑定在死信交换机上的队列。死信交换机也不是什么特殊的交换机，只不过是用来接受死信的交换机，所以可以为任何类型【Direct、Fanout、Topic】。一般来说，会为每个业务队列分配一个独有的路由key，并对应的配置一个死信队列进行监听，也就是说，一般会为每个重要的业务队列配置一个死信队列。

有了前文这些陈述后，接下来就是惊险刺激的实战环节，这里省略了RabbitMQ环境的部署和搭建环节。

先创建一个Springboot项目。然后在pom文件中添加 `spring-boot-starter-amqp` 和 `spring-boot-starter-web` 的依赖，接下来创建一个Config类，这里是关键：



```java
@Configuration
public class RabbitMQConfig {

    public static final String BUSINESS_EXCHANGE_NAME = "dead.letter.demo.simple.business.exchange";
    public static final String BUSINESS_QUEUEA_NAME = "dead.letter.demo.simple.business.queuea";
    public static final String BUSINESS_QUEUEB_NAME = "dead.letter.demo.simple.business.queueb";
    public static final String DEAD_LETTER_EXCHANGE = "dead.letter.demo.simple.deadletter.exchange";
    public static final String DEAD_LETTER_QUEUEA_ROUTING_KEY = "dead.letter.demo.simple.deadletter.queuea.routingkey";
    public static final String DEAD_LETTER_QUEUEB_ROUTING_KEY = "dead.letter.demo.simple.deadletter.queueb.routingkey";
    public static final String DEAD_LETTER_QUEUEA_NAME = "dead.letter.demo.simple.deadletter.queuea";
    public static final String DEAD_LETTER_QUEUEB_NAME = "dead.letter.demo.simple.deadletter.queueb";

    // 声明业务Exchange
    @Bean("businessExchange")
    public FanoutExchange businessExchange(){
        return new FanoutExchange(BUSINESS_EXCHANGE_NAME);
    }

    // 声明死信Exchange
    @Bean("deadLetterExchange")
    public DirectExchange deadLetterExchange(){
        return new DirectExchange(DEAD_LETTER_EXCHANGE);
    }

    // 声明业务队列A
    @Bean("businessQueueA")
    public Queue businessQueueA(){
        Map<String, Object> args = new HashMap<>(2);
//       x-dead-letter-exchange    这里声明当前队列绑定的死信交换机
        args.put("x-dead-letter-exchange", DEAD_LETTER_EXCHANGE);
//       x-dead-letter-routing-key  这里声明当前队列的死信路由key
        args.put("x-dead-letter-routing-key", DEAD_LETTER_QUEUEA_ROUTING_KEY);
        return QueueBuilder.durable(BUSINESS_QUEUEA_NAME).withArguments(args).build();
    }

    // 声明业务队列B
    @Bean("businessQueueB")
    public Queue businessQueueB(){
        Map<String, Object> args = new HashMap<>(2);
//       x-dead-letter-exchange    这里声明当前队列绑定的死信交换机
        args.put("x-dead-letter-exchange", DEAD_LETTER_EXCHANGE);
//       x-dead-letter-routing-key  这里声明当前队列的死信路由key
        args.put("x-dead-letter-routing-key", DEAD_LETTER_QUEUEB_ROUTING_KEY);
        return QueueBuilder.durable(BUSINESS_QUEUEB_NAME).withArguments(args).build();
    }

    // 声明死信队列A
    @Bean("deadLetterQueueA")
    public Queue deadLetterQueueA(){
        return new Queue(DEAD_LETTER_QUEUEA_NAME);
    }

    // 声明死信队列B
    @Bean("deadLetterQueueB")
    public Queue deadLetterQueueB(){
        return new Queue(DEAD_LETTER_QUEUEB_NAME);
    }

    // 声明业务队列A绑定关系
    @Bean
    public Binding businessBindingA(@Qualifier("businessQueueA") Queue queue,
                                    @Qualifier("businessExchange") FanoutExchange exchange){
        return BindingBuilder.bind(queue).to(exchange);
    }

    // 声明业务队列B绑定关系
    @Bean
    public Binding businessBindingB(@Qualifier("businessQueueB") Queue queue,
                                    @Qualifier("businessExchange") FanoutExchange exchange){
        return BindingBuilder.bind(queue).to(exchange);
    }

    // 声明死信队列A绑定关系
    @Bean
    public Binding deadLetterBindingA(@Qualifier("deadLetterQueueA") Queue queue,
                                    @Qualifier("deadLetterExchange") DirectExchange exchange){
        return BindingBuilder.bind(queue).to(exchange).with(DEAD_LETTER_QUEUEA_ROUTING_KEY);
    }

    // 声明死信队列B绑定关系
    @Bean
    public Binding deadLetterBindingB(@Qualifier("deadLetterQueueB") Queue queue,
                                      @Qualifier("deadLetterExchange") DirectExchange exchange){
        return BindingBuilder.bind(queue).to(exchange).with(DEAD_LETTER_QUEUEB_ROUTING_KEY);
    }
}
```

这里声明了两个Exchange，一个是业务Exchange，另一个是死信Exchange，业务Exchange下绑定了两个业务队列，业务队列都配置了同一个死信Exchange，并分别配置了路由key，在死信Exchange下绑定了两个死信队列，设置的路由key分别为业务队列里配置的路由key。

下面是配置文件application.yml：

```yml
spring:
  rabbitmq:
    host: localhost
    password: guest
    username: guest
    listener:
      type: simple
      simple:
          default-requeue-rejected: false
          acknowledge-mode: manual
```

这里记得将`default-requeue-rejected`属性设置为false。

接下来，是业务队列的消费代码：

```java
@Slf4j
@Component
public class BusinessMessageReceiver {

    @RabbitListener(queues = BUSINESS_QUEUEA_NAME)
    public void receiveA(Message message, Channel channel) throws IOException {
        String msg = new String(message.getBody());
        log.info("收到业务消息A：{}", msg);
        boolean ack = true;
        Exception exception = null;
        try {
            if (msg.contains("deadletter")){
                throw new RuntimeException("dead letter exception");
            }
        } catch (Exception e){
            ack = false;
            exception = e;
        }
        if (!ack){
            log.error("消息消费发生异常，error msg:{}", exception.getMessage(), exception);
            channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, false);
        } else {
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        }
    }

    @RabbitListener(queues = BUSINESS_QUEUEB_NAME)
    public void receiveB(Message message, Channel channel) throws IOException {
        System.out.println("收到业务消息B：" + new String(message.getBody()));
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
    }
}
```

然后配置死信队列的消费者：

```java
@Component
public class DeadLetterMessageReceiver {


    @RabbitListener(queues = DEAD_LETTER_QUEUEA_NAME)
    public void receiveA(Message message, Channel channel) throws IOException {
        System.out.println("收到死信消息A：" + new String(message.getBody()));
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
    }

    @RabbitListener(queues = DEAD_LETTER_QUEUEB_NAME)
    public void receiveB(Message message, Channel channel) throws IOException {
        System.out.println("收到死信消息B：" + new String(message.getBody()));
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
    }
}
```

最后，为了方便测试，写一个简单的消息生产者，并通过controller层来生产消息。

```java
@Component
public class BusinessMessageSender {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void sendMsg(String msg){
        rabbitTemplate.convertSendAndReceive(BUSINESS_EXCHANGE_NAME, "", msg);
    }
}
```



```java
@RequestMapping("rabbitmq")
@RestController
public class RabbitMQMsgController {

    @Autowired
    private BusinessMessageSender sender;

    @RequestMapping("sendmsg")
    public void sendMsg(String msg){
        sender.sendMsg(msg);
    }
}
```

一切准备就绪，启动！

可以从RabbitMQ的管理后台中看到一共有四个队列，除默认的Exchange外还有声明的两个Exchange。

![8(%(3A_Y`_N8XX8W5XHZMWY.png](https://i.loli.net/2019/07/14/5d2ae72753a8e90481.png)



![123.png](https://i.loli.net/2019/07/14/5d2ae792598f781012.png)



接下来，访问一下url，来测试一下：

```
http://localhost:8080/rabbitmq/sendmsg?msg=msg
```

```java
收到业务消息A：msg
收到业务消息B：msg
```

表示两个Consumer都正常收到了消息。这代表正常消费的消息，ack后正常返回。然后我们再来测试nck的消息。

```
http://localhost:8080/rabbitmq/sendmsg?msg=deadletter
```

这将会触发业务队列A的NCK，按照预期，消息被NCK后，会抛到死信队列中，因此死信队列将会出现这个消息，日志如下：

```
收到业务消息A：deadletter
消息消费发生异常，error msg:dead letter exception
java.lang.RuntimeException: dead letter exception
...

收到死信消息A：deadletter
```

可以看到，死信队列的Consumer接受到了这个消息，所以流程到此为止就打通了。

## 五、死信消息的变化

那么“死信”被丢到死信队列中后，会发生什么变化呢？

如果队列配置了参数 `x-dead-letter-routing-key` 的话，“死信”的路由key将会被替换成该参数对应的值。如果没有设置，则保留该消息原有的路由key。

举个栗子：

如果原有消息的路由key是`testA`，被发送到业务Exchage中，然后被投递到业务队列QueueA中，如果该队列没有配置参数`x-dead-letter-routing-key`，则该消息成为死信后，将保留原有的路由key`testA`，如果配置了该参数，并且值设置为`testB`，那么该消息成为死信后，路由key将会被替换为`testB`，然后被抛到死信交换机中。

另外，由于被抛到了死信交换机，所以消息的Exchange Name也会被替换为死信交换机的名称。

消息的Header中，也会添加很多奇奇怪怪的字段，修改一下上面的代码，在死信队列的消费者中添加一行日志输出：

```java
log.info("死信消息properties：{}", message.getMessageProperties());
```

然后重新运行一次，即可得到死信消息Header中被添加的信息：

```
死信消息properties：MessageProperties [headers={x-first-death-exchange=dead.letter.demo.simple.business.exchange, x-death=[{reason=rejected, count=1, exchange=dead.letter.demo.simple.business.exchange, time=Sun Jul 14 16:48:16 CST 2019, routing-keys=[], queue=dead.letter.demo.simple.business.queuea}], x-first-death-reason=rejected, x-first-death-queue=dead.letter.demo.simple.business.queuea}, correlationId=1, replyTo=amq.rabbitmq.reply-to.g2dkABZyYWJiaXRAREVTS1RPUC1DUlZGUzBOAAAPQAAAAAAB.bLbsdR1DnuRSwiKKmtdOGw==, contentType=text/plain, contentEncoding=UTF-8, contentLength=0, receivedDeliveryMode=PERSISTENT, priority=0, redelivered=false, receivedExchange=dead.letter.demo.simple.deadletter.exchange, receivedRoutingKey=dead.letter.demo.simple.deadletter.queuea.routingkey, deliveryTag=1, consumerTag=amq.ctag-NSp18SUPoCNvQcoYoS2lPg, consumerQueue=dead.letter.demo.simple.deadletter.queuea]
```

Header中看起来有很多信息，实际上并不多，只是值比较长而已。下面就简单说明一下Header中的值：

| 字段名                 | 含义                                                         |
| :--------------------- | :----------------------------------------------------------- |
| x-first-death-exchange | 第一次被抛入的死信交换机的名称                               |
| x-first-death-reason   | 第一次成为死信的原因，`rejected`：消息在重新进入队列时被队列拒绝，由于`default-requeue-rejected` 参数被设置为`false`。`expired` ：消息过期。`maxlen` ： 队列内消息数量超过队列最大容量 |
| x-first-death-queue    | 第一次成为死信前所在队列名称                                 |
| x-death                | 历次被投入死信交换机的信息列表，同一个消息每次进入一个死信交换机，这个数组的信息就会被更新 |

## 六、死信队列应用场景

通过上面的信息，我们已经知道如何使用死信队列了，那么死信队列一般在什么场景下使用呢？

一般用在较为重要的业务队列中，确保未被正确消费的消息不被丢弃，一般发生消费异常可能原因主要有由于消息信息本身存在错误导致处理异常，处理过程中参数校验异常，或者因网络波动导致的查询异常等等，当发生异常时，当然不能每次通过日志来获取原消息，然后让运维帮忙重新投递消息（没错，以前就是这么干的= =）。通过配置死信队列，可以让未正确处理的消息暂存到另一个队列中，待后续排查清楚问题后，编写相应的处理代码来处理死信消息，这样比手工恢复数据要好太多了。

## 七、总结

死信队列其实并没有什么神秘的地方，不过是绑定在死信交换机上的普通队列，而死信交换机也只是一个普通的交换机，不过是用来专门处理死信的交换机。

总结一下死信消息的生命周期：

1. 业务消息被投入业务队列
2. 消费者消费业务队列的消息，由于处理过程中发生异常，于是进行了nck或者reject操作
3. 被nck或reject的消息由RabbitMQ投递到死信交换机中
4. 死信交换机将消息投入相应的死信队列
5. 死信队列的消费者消费死信消息

死信消息是RabbitMQ为我们做的一层保证，其实我们也可以不使用死信队列，而是在消息消费异常时，将消息主动投递到另一个交换机中，当你明白了这些之后，这些Exchange和Queue想怎样配合就能怎么配合。比如从死信队列拉取消息，然后发送邮件、短信、钉钉通知来通知开发人员关注。或者将消息重新投递到一个队列然后设置过期时间，来进行延时消费





## 如何实现延时队列

迟队列的使用场景：1.未按时支付的订单，30分钟过期之后取消订单；2.给活跃度比较低的用户间隔N天之后推送消息，提高活跃度；3.过1分钟给新注册会员的用户，发送注册邮件等。

实现延迟队列的方式有两种：

1. 通过消息过期后进入死信交换器，再由交换器转发到延迟消费队列，实现延迟功能；
2. 使用rabbitmq-delayed-message-exchange插件实现延迟功能；

**注意：** 延迟插件rabbitmq-delayed-message-exchange是在RabbitMQ 3.5.7及以上的版本才支持的，依赖Erlang/OPT 18.0及以上运行环境。

由于使用死信交换器相对曲折，本文重点介绍第二种方式，使用rabbitmq-delayed-message-exchange插件完成延迟队列的功能。

### 安装插件

#### 1.1 下载插件

打开官网下载：http://www.rabbitmq.com/community-plugins.html

选择相应的对应的版本“3.7.x”点击下载。

**注意：** 下载的是.zip的安装包，下载完之后需要手动解压。

#### 1.2 安装插件

拷贝插件到Docker：

> docker cp D:\rabbitmq_delayed_message_exchange-20171201-3.7.x.ez rabbit:/plugins

RabbitMQ在Docker的安装，请参照本系列的上一篇文章：http://www.apigo.cn/2018/09/11/springboot13/

#### 1.3 启动插件

进入docker内部：

> docker exec -it rabbit /bin/bash

开启插件：

> rabbitmq-plugins enable rabbitmq_delayed_message_exchange

查询安装的所有插件：

> rabbitmq-plugins list

安装正常，效果如下图：

![img](http://icdn.apigo.cn/blog/springboot-rabbitmq-4.png)

重启RabbitMQ，使插件生效

> docker restart rabbit



### 代码实现

#### 配置队列

```java
import com.example.rabbitmq.mq.DirectConfig;
import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.util.HashMap;
import java.util.Map;

@Configuration
public class DelayedConfig {
    final static String QUEUE_NAME = "delayed.goods.order";
    final static String EXCHANGE_NAME = "delayedec";
    @Bean
    public Queue queue() {
        return new Queue(DelayedConfig.QUEUE_NAME);
    }

    // 配置默认的交换机
    @Bean
    CustomExchange customExchange() {
        Map<String, Object> args = new HashMap<>();
        args.put("x-delayed-type", "direct");
        //参数二为类型：必须是x-delayed-message
        return new CustomExchange(DelayedConfig.EXCHANGE_NAME, "x-delayed-message", true, false, args);
    }
    // 绑定队列到交换器
    @Bean
    Binding binding(Queue queue, CustomExchange exchange) {
        return BindingBuilder.bind(queue).to(exchange).with(DelayedConfig.QUEUE_NAME).noargs();
    }
}
```



#### 发送消息

```java
import org.springframework.amqp.AmqpException;
import org.springframework.amqp.core.AmqpTemplate;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.core.MessagePostProcessor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import java.text.SimpleDateFormat;
import java.util.Date;

@Component
public class DelayedSender {
    @Autowired
    private AmqpTemplate rabbitTemplate;

    public void send(String msg) {
        SimpleDateFormat sf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        System.out.println("发送时间：" + sf.format(new Date()));

        rabbitTemplate.convertAndSend(DelayedConfig.EXCHANGE_NAME, DelayedConfig.QUEUE_NAME, msg, new MessagePostProcessor() {
            @Override
            public Message postProcessMessage(Message message) throws AmqpException {
                message.getMessageProperties().setHeader("x-delay", 3000);
                return message;
            }
        });
    }
}
```



#### 消费消息

```java
import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;
import java.text.SimpleDateFormat;
import java.util.Date;

@Component
@RabbitListener(queues = "delayed.goods.order")
public class DelayedReceiver {
    @RabbitHandler
    public void process(String msg) {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        System.out.println("接收时间:" + sdf.format(new Date()));
        System.out.println("消息内容：" + msg);
    }
}

```



#### 测试队列

```java
import com.example.rabbitmq.RabbitmqApplication;
import com.example.rabbitmq.mq.delayed.DelayedSender;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import java.text.SimpleDateFormat;
import java.util.Date;

@RunWith(SpringRunner.class)
@SpringBootTest
public class DelayedTest {

    @Autowired
    private DelayedSender sender;

    @Test
    public void Test() throws InterruptedException {
        SimpleDateFormat sf = new SimpleDateFormat("yyyy-MM-dd");
        sender.send("Hi Admin.");
        Thread.sleep(5 * 1000); //等待接收程序执行之后，再退出测试
    }
}
```



