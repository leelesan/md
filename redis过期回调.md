在电商、支付等领域，往往会有这样的场景，用户下单后放弃支付了，那这笔订单会在指定的时间段后进行关闭操作，细心的你一定发现了像某宝、某东都有这样的逻辑，而且时间很准确，误差在1s内；那他们是怎么实现的呢？

**一般的做法有如下几种**

-   定时任务关闭订单
    
-   rocketmq延迟队列
    
-   rabbitmq死信队列
    
-   时间轮算法
    
-   redis过期监听
    

## 一、定时任务关闭订单（最low）

一般情况下，最不推荐的方式就是关单方式就是定时任务方式，原因我们可以看下面的图来说明

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6030ab5a1d39495792184607cb1de731~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

我们假设，关单时间为下单后10分钟，定时任务间隔也是10分钟；通过上图我们看出，如果在第1分钟下单，在第20分钟的时候才能被扫描到执行关单操作，这样误差达到10分钟，这在很多场景下是不可接受的，另外需要频繁扫描主订单号造成网络IO和磁盘IO的消耗，对实时交易造成一定的冲击，所以PASS

## 二、rocketmq延迟队列方式

**延迟消息** 生产者把消息发送到消息服务器后，并不希望被立即消费，而是等待指定时间后才可以被消费者消费，这类消息通常被称为延迟消息。 在RocketMQ开源版本中，支持延迟消息，但是不支持任意时间精度的延迟消息，只支持特定级别的延迟消息。 消息延迟级别分别为**1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h**，共18个级别。

**发送延迟消息（生产者）**

```java
/**
     * 推送延迟消息
     * @param topic 
     * @param body 
     * @param producerGroup 
     * @return boolean
     */
    public boolean sendMessage(String topic, String body, String producerGroup)
    {
        try
        {
            Message recordMsg = new Message(topic, body.getBytes());
            producer.setProducerGroup(producerGroup);

            //设置消息延迟级别，我这里设置14，对应就是延时10分钟
            // "1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h"
            recordMsg.setDelayTimeLevel(14);
            // 发送消息到一个Broker
            SendResult sendResult = producer.send(recordMsg);
            // 通过sendResult返回消息是否成功送达
            log.info("发送延迟消息结果：======sendResult：{}", sendResult);
            DateFormat format =new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            log.info("发送时间：{}", format.format(new Date()));

            return true;
        }
        catch (Exception e)
        {
            e.printStackTrace();
            log.error("延迟消息队列推送消息异常:{},推送内容:{}", e.getMessage(), body);
        }
        return false;
    }

```

**消费延迟消息（消费者）**

```java
/**
     * 接收延迟消息
     * 
     * @param topic
     * @param consumerGroup
     * @param messageHandler
     */
    public void messageListener(String topic, String consumerGroup, MessageListenerConcurrently messageHandler)
{
        ThreadPoolUtil.execute(() ->
        {
            try
            {
                DefaultMQPushConsumer consumer = new DefaultMQPushConsumer();
                consumer.setConsumerGroup(consumerGroup);
                consumer.setVipChannelEnabled(false);
                consumer.setNamesrvAddr(address);
                //设置消费者拉取消息的策略，*表示消费该topic下的所有消息，也可以指定tag进行消息过滤
                consumer.subscribe(topic, "*");
                //消费者端启动消息监听，一旦生产者发送消息被监听到，就打印消息，和rabbitmq中的handlerDelivery类似
                consumer.registerMessageListener(messageHandler);
                consumer.start();
                log.info("启动延迟消息队列监听成功:" + topic);
            }
            catch (MQClientException e)
            {
                log.error("启动延迟消息队列监听失败:{}", e.getErrorMessage());
                System.exit(1);
            }
        });
    }

```

**实现监听类，处理具体逻辑**

```java
/**
 * 延迟消息监听
 * 
 */
@Component
public class CourseOrderTimeoutListener implements ApplicationListener<ApplicationReadyEvent>
{

    @Resource
    private MQUtil mqUtil;

    @Resource
    private CourseOrderTimeoutHandler courseOrderTimeoutHandler;

    @Override
    public void onApplicationEvent(ApplicationReadyEvent applicationReadyEvent)
{
        // 订单超时监听
        mqUtil.messageListener(EnumTopic.ORDER_TIMEOUT, EnumGroup.ORDER_TIMEOUT_GROUP, courseOrderTimeoutHandler);
    }
}


```

```java
/**
 *  实现监听
 */
@Slf4j
@Component
public class CourseOrderTimeoutHandler implements MessageListenerConcurrently
{

    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
        for (MessageExt msg : list)
        {
            // 得到消息体
            String body = new String(msg.getBody());
            JSONObject userJson = JSONObject.parseObject(body);
            TCourseBuy courseBuyDetails = JSON.toJavaObject(userJson, TCourseBuy.class);

            // 处理具体的业务逻辑，，，，，

      DateFormat format =new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
           log.info("消费时间：{}", format.format(new Date()));
           
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
}

```

这种方式相比定时任务好了很多，但是有一个致命的缺点，就是延迟等级只有18种（商业版本支持自定义时间），如果我们想把关闭订单时间设置在15分钟该如何处理呢？显然不够灵活。

## 三、rabbitmq死信队列的方式

Rabbitmq本身是没有延迟队列的，只能通过Rabbitmq本身队列的特性来实现，想要Rabbitmq实现延迟队列，需要使用Rabbitmq的死信交换机（Exchange）和消息的存活时间TTL（Time To Live）

**死信交换机** 一个消息在满足如下条件下，会进死信交换机，记住这里是交换机而不是队列，一个交换机可以对应很多队列。

一个消息被Consumer拒收了，并且reject方法的参数里requeue是false。也就是说不会被再次放在队列里，被其他消费者使用。 上面的消息的TTL到了，消息过期了。

队列的长度限制满了。排在前面的消息会被丢弃或者扔到死信路由上。 **死信交换机就是普通的交换机**，只是因为我们把过期的消息扔进去，所以叫死信交换机，并不是说死信交换机是某种特定的交换机

**消息TTL（消息存活时间）** 消息的TTL就是消息的存活时间。RabbitMQ可以对队列和消息分别设置TTL。对队列设置就是队列没有消费者连着的保留时间，也可以对每一个单独的消息做单独的设置。超过了这个时间，我们认为这个消息就死了，称之为死信。如果队列设置了，消息也设置了，那么会取值较小的。所以一个消息如果被路由到不同的队列中，这个消息死亡的时间有可能不一样（不同的队列设置）。这里单讲单个消息的TTL，因为它才是实现延迟任务的关键。

```java
byte[] messageBodyBytes = "Hello, world!".getBytes();  
AMQP.BasicProperties properties = new AMQP.BasicProperties();  
properties.setExpiration("60000");  
channel.basicPublish("my-exchange", "queue-key", properties, messageBodyBytes);  

```

可以通过设置消息的expiration字段或者x-message-ttl属性来设置时间，两者是一样的效果。只是expiration字段是字符串参数，所以要写个int类型的字符串：当上面的消息扔到队列中后，过了60秒，如果没有被消费，它就死了。不会被消费者消费到。这个消息后面的，没有“死掉”的消息对顶上来，被消费者消费。死信在队列中并不会被删除和释放，它会被统计到队列的消息数中去

**处理流程图**

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/712a389820484c11bbc4808df3a5adb6~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

**创建交换机（Exchanges）和队列（Queues）**

**创建死信交换机**

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7540f409b0de41e994228a1fb01b7a0e~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

如图所示，就是创建一个普通的交换机，这里为了方便区分，把交换机的名字取为：delay

**创建自动过期消息队列** 这个队列的主要作用是让消息定时过期的，比如我们需要2小时候关闭订单，我们就需要把消息放进这个队列里面，把消息过期时间设置为2小时

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1083f903bd8c43a7ac0aa7716469247b~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

创建一个一个名为delay\_queue1的自动过期的队列，当然图片上面的参数并不会让消息自动过期，因为我们并没有设置x-message-ttl参数，如果整个队列的消息有消息都是相同的，可以设置，这里为了灵活，所以并没有设置，另外两个参数x-dead-letter-exchange代表消息过期后，消息要进入的交换机，这里配置的是delay，也就是死信交换机，x-dead-letter-routing-key是配置消息过期后，进入死信交换机的routing-key,跟发送消息的routing-key一个道理，根据这个key将消息放入不同的队列

**创建消息处理队列** 这个队列才是真正处理消息的队列，所有进入这个队列的消息都会被处理

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f3014f04496a484f818a1a3b4c7123c5~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

**消息队列的名字为delay\_queue2** 消息队列绑定到交换机 进入交换机详情页面，将创建的2个队列（delayqueue1和delayqueue2）绑定到交换机上面

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9090581453e14cbfb06e37634548586b~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp) 自动过期消息队列的routing key 设置为delay 绑定delayqueue2

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/963c59b5312b441cad1d2cc00abc4d63~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

delayqueue2 的key要设置为创建自动过期的队列的x-dead-letter-routing-key参数，这样当消息过期的时候就可以自动把消息放入delay\_queue2这个队列中了 绑定后的管理页面如下图：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/467ded5ffb244f24bfc247aa9ab9e1de~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

当然这个绑定也可以使用代码来实现，只是为了直观表现，所以本文使用的管理平台来操作 **发送消息**

```java
String msg = "hello word";  
MessageProperties messageProperties = newMessageProperties();  
messageProperties.setExpiration("6000");
messageProperties.setCorrelationId(UUID.randomUUID().toString().getBytes());
Message message = newMessage(msg.getBytes(), messageProperties);
rabbitTemplate.convertAndSend("delay", "delay",message);

```

设置了让消息6秒后过期 注意：因为要让消息自动过期，所以一定不能设置delay\_queue1的监听，不能让这个队列里面的消息被接受到，否则消息一旦被消费，就不存在过期了

**接收消息** 接收消息配置好delay\_queue2的监听就好了

```java
package wang.raye.rabbitmq.demo1;
import org.springframework.amqp.core.AcknowledgeMode;  
import org.springframework.amqp.core.Binding;  
import org.springframework.amqp.core.BindingBuilder;  
import org.springframework.amqp.core.DirectExchange;  
import org.springframework.amqp.core.Message;  
import org.springframework.amqp.core.Queue;  
import org.springframework.amqp.rabbit.connection.CachingConnectionFactory;  
import org.springframework.amqp.rabbit.connection.ConnectionFactory;  
import org.springframework.amqp.rabbit.core.ChannelAwareMessageListener;  
import org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;
@Configuration
publicclassDelayQueue{  
    /** 消息交换机的名字*/
    publicstaticfinalString EXCHANGE = "delay";
    /** 队列key1*/
    publicstaticfinalString ROUTINGKEY1 = "delay";
    /** 队列key2*/
    publicstaticfinalString ROUTINGKEY2 = "delay_key";
    /**
     * 配置链接信息
     * @return
     */
    @Bean
    publicConnectionFactory connectionFactory() {
        CachingConnectionFactory connectionFactory = newCachingConnectionFactory("120.76.237.8",5672);
        connectionFactory.setUsername("kberp");
        connectionFactory.setPassword("kberp");
        connectionFactory.setVirtualHost("/");
        connectionFactory.setPublisherConfirms(true); // 必须要设置
        return connectionFactory;
    }
    /**  
     * 配置消息交换机
     * 针对消费者配置  
        FanoutExchange: 将消息分发到所有的绑定队列，无routingkey的概念  
        HeadersExchange ：通过添加属性key-value匹配  
        DirectExchange:按照routingkey分发到指定队列  
        TopicExchange:多关键字匹配  
     */  
    @Bean  
    publicDirectExchange defaultExchange() {  
        returnnewDirectExchange(EXCHANGE, true, false);
    }
    /**
     * 配置消息队列2
     * 针对消费者配置  
     * @return
     */
    @Bean
    publicQueue queue() {  
       returnnewQueue("delay_queue2", true); //队列持久  
    }
    /**
     * 将消息队列2与交换机绑定
     * 针对消费者配置  
     * @return
     */
    @Bean  
    @Autowired
    publicBinding binding() {  
        returnBindingBuilder.bind(queue()).to(defaultExchange()).with(DelayQueue.ROUTINGKEY2);  
    }
    /**
     * 接受消息的监听，这个监听会接受消息队列1的消息
     * 针对消费者配置  
     * @return
     */
    @Bean  
    @Autowired
    publicSimpleMessageListenerContainer messageContainer2(ConnectionFactory connectionFactory) {  
        SimpleMessageListenerContainer container = newSimpleMessageListenerContainer(connectionFactory());  
        container.setQueues(queue());  
        container.setExposeListenerChannel(true);  
        container.setMaxConcurrentConsumers(1);  
        container.setConcurrentConsumers(1);  
        container.setAcknowledgeMode(AcknowledgeMode.MANUAL); //设置确认模式手工确认  
        container.setMessageListener(newChannelAwareMessageListener() {
            publicvoid onMessage(Message message, com.rabbitmq.client.Channel channel) throwsException{
                byte[] body = message.getBody();  
                System.out.println("delay_queue2 收到消息 : "+ newString(body));  
                channel.basicAck(message.getMessageProperties().getDeliveryTag(), false); //确认消息成功消费  
            }  
        });  
        return container;  
    }  
}

```

这种方式可以自定义进入死信队列的时间；是不是很完美，但是有的小伙伴的情况是消息中间件就是rocketmq，公司也不可能会用商业版，怎么办？那就进入下一节

## 四、时间轮算法

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/697829f92f6041de83c34d870c7adba4~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

(1)创建环形队列，例如可以创建一个包含3600个slot的环形队列(本质是个数组)

(2)任务集合，环上每一个slot是一个Set 同时，启动一个timer，这个timer每隔1s，在上述环形队列中移动一格，有一个Current Index指针来标识正在检测的slot。

Task结构中有两个很重要的属性： (1)Cycle-Num：当Current Index第几圈扫描到这个Slot时，执行任务 (2)订单号，要关闭的订单号（也可以是其他信息，比如：是一个基于某个订单号的任务）

假设当前Current Index指向第0格，例如在3610秒之后，有一个订单需要关闭，只需： (1)计算这个订单应该放在哪一个slot，当我们计算的时候现在指向1，3610秒之后，应该是第10格，所以这个Task应该放在第10个slot的Set中 (2)计算这个Task的Cycle-Num，由于环形队列是3600格(每秒移动一格，正好1小时)，这个任务是3610秒后执行，所以应该绕3610/3600=1圈之后再执行，于是Cycle-Num=1

Current Index不停的移动，每秒移动到一个新slot，这个slot中对应的Set，每个Task看Cycle-Num是不是0： (1)如果不是0，说明还需要多移动几圈，将Cycle-Num减1 (2)如果是0，说明马上要执行这个关单Task了，取出订单号执行关单(可以用单独的线程来执行Task)，并把这个订单信息从Set中删除即可。 (1)无需再轮询全部订单，效率高 (2)一个订单，任务只执行一次 (3)时效性好，精确到秒(控制timer移动频率可以控制精度)

## 五、redis过期监听

**1.修改redis.windows.conf配置文件中notify-keyspace-events的值** 默认配置notify-keyspace-events的值为 "" 修改为 notify-keyspace-events Ex 这样便开启了过期事件

**2\. 创建配置类RedisListenerConfig（配置RedisMessageListenerContainer这个Bean）**

```java
package com.zjt.shop.config;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.listener.RedisMessageListenerContainer;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;
 
 
@Configuration
public class RedisListenerConfig {
 
    @Autowired
    private RedisTemplate redisTemplate;
 
    /**
     * @return
     */
    @Bean
    public RedisTemplate redisTemplateInit() {
 
        // key序列化
        redisTemplate.setKeySerializer(new StringRedisSerializer());
 
        //val实例化
        redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());
 
        return redisTemplate;
    }
 
    @Bean
    RedisMessageListenerContainer container(RedisConnectionFactory connectionFactory) {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        return container;
    }
 
}

```

**3.继承KeyExpirationEventMessageListener创建redis过期事件的监听类**

```java
package com.zjt.shop.common.util;
import com.baomidou.mybatisplus.core.conditions.query.QueryWrapper;
import com.baomidou.mybatisplus.core.conditions.update.UpdateWrapper;
import com.zjt.shop.modules.order.service.OrderInfoService;
import com.zjt.shop.modules.product.entity.OrderInfoEntity;
import com.zjt.shop.modules.product.mapper.OrderInfoMapper;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.connection.Message;
import org.springframework.data.redis.listener.KeyExpirationEventMessageListener;
import org.springframework.data.redis.listener.RedisMessageListenerContainer;
import org.springframework.stereotype.Component;
 

@Slf4j
@Component
public class RedisKeyExpirationListener extends KeyExpirationEventMessageListener {
 
    public RedisKeyExpirationListener(RedisMessageListenerContainer listenerContainer) {
        super(listenerContainer);
    }
 
    @Autowired
    private OrderInfoMapper orderInfoMapper;
 
    /**
     * 针对redis数据失效事件，进行数据处理
     * @param message
     * @param pattern
     */
    @Override
    public void onMessage(Message message, byte[] pattern) {
      try {
          String key = message.toString();
          //从失效key中筛选代表订单失效的key
          if (key != null && key.startsWith("order_")) {
              //截取订单号，查询订单，如果是未支付状态则为-取消订单
              String orderNo = key.substring(6);
              QueryWrapper<OrderInfoEntity> queryWrapper = new QueryWrapper<>();
              queryWrapper.eq("order_no",orderNo);
              OrderInfoEntity orderInfo = orderInfoMapper.selectOne(queryWrapper);
              if (orderInfo != null) {
                  if (orderInfo.getOrderState() == 0) {   //待支付
                      orderInfo.setOrderState(4);         //已取消
                      orderInfoMapper.updateById(orderInfo);
                      log.info("订单号为【" + orderNo + "】超时未支付-自动修改为已取消状态");
                  }
              }
          }
      } catch (Exception e) {
          e.printStackTrace();
          log.error("【修改支付订单过期状态异常】：" + e.getMessage());
      }
    }
}

```

**4：测试** 通过redis客户端存一个有效时间为3s的订单：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d6c11699d7074bbda5f0c18e5e96d2fb~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

结果：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/239e0faac8d14b00b1c04d9c052c9e85~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

**总结：** 以上方法只是个人对于关单的一些想法，可能有些地方有疏漏，请在公众号直接留言进行指出，当然如果你有更好的关单方式也可以随时沟通交流

> 更多精彩内容，请关注我的公众号「程序员阿牛」
> 
> 我的个人博客站点：[www.kuya123.com](https://www.kuya123.com "https://www.kuya123.com")