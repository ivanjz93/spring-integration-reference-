# 4.1 消息通道

Message扮演封装数据的关键角色，而MessageChannel使消息生产者与消息消费者解耦。

## 4.1.1 MessageChannel接口

Spring Integration的顶级MessageChannel接口定义如下。

```
public interface MessageChannel {

    boolean send(Message message);

    boolean send(Message message, long timeout);

}
```

当发送消息时，如果成功发送返回值是true。如果发送调用超时或者被中断，返回false。

### PollableChannel

由于Message Channel可能缓存也可能不缓存消息，所以有两个子接口定义了缓存（轮询）和不缓存（订阅）通道行为。下面是PollableChannel的定义。

```
public interface PollableChannel extends MessageChannel {

    Message<?> receive();

    Message<?> receive(long timeout);
}
```

与发送方法相似，当接收消息时，如果超时或被中断，返回值为null。

### SubscribableChannel

SubscribableChannel基础接口被直接发送消息给它们的订阅MessageHandler的通道实现。因此，它们不提供用于轮询的接收方法，但是定义了管理订阅者的方法：

```
public interface SubscribableChannel extends MessageChannel {

    boolean subscribe(MessageHandler handler);

    boolean unsubscribe(MessageHandler handler);

}
```

## 4.1.2 消息通道实现

Spring Integration提供了集中不同的消息通道实现。

### PublishSubscribeChannel

PublishSubscribeChannel实现将任何发送给它的消息广播给它的所有订阅处理器。它通常用于发送主要角色时通知的事件消息，而不是通常由单个处理器处理的文档消息。注意PublishSubscribeChannel仅用于发送。因为当调用send\(Message\)方法时它直接广播给订阅者，所以消费者不能轮询消息（它没有实现PollableChannel，因此没有receive\(\)方法）。相反，任何订阅者必须本身是一个MessageHandler，同时订阅者的handleMessage\(Message\)方法会被轮流调用。

在3.0版本之前，调用没有订阅者的PublishSubscribeChannel的发送方法会返回false。当与MessagingTemplate一起使用，会抛出MessageDeliveryException。从3.0版本开始，行为发生了变化，如果至少有最少用户（并成功处理消息），发送总是被视为成功。通过设置minSubscribers属性（默认值为0）可以改变这个行为。

_如果使用TaskExecutor，则只能根据正确数量的订阅者进行此确定，因为消息的实际处理是异步执行的。_

### QueueChannel

QueueChannel实现包装了一个队列。与PublishSubscribeChannel不同，QueueChannel有点对点的语义。即，即使通道有多个消费者，只有其中一个可以接收到发送给通道的消息。它提供了没有参数的默认构造方法（队列的容量是Integer.MAX\_VALUE）和接收队列容量的构造函数：

```
public QueueChannel(int capacity)
```

如果通道没有达到容量限制，它将会在内部队列中存储消息，并且send\(\)方法会立即方法，即便当时没有接受者准备处理消息。如果队列达到容量限制，那么发送着会阻塞直到有可用空间。或者，如果调用接收超时的发送方法，它会阻塞直到有可用空间或超时。相似的，如果队列中有可用的消息，接收方法调用立即返回；但是如果队列是空的，那么接收调用会阻塞直到有可用的消息或超时。在每种情况下，可以通过传递超时值为0，强制立即返回而不考虑队列状态。注意，调用没有参数的send\(\)和receive\(\)方法将会无限期的阻塞。

### PriorityChannel

QueueChannel强制先进先出（FIFO）顺序，PriorityChannel是允许消息在队列中基于优先级排序的一个替代实现。默认的优先级由每个消息的priority头决定。作为自定义的决定优先级逻辑，Comparator&lt;Message&lt;?&gt;类型的比较器可以被提供给PriorityChannel的构造函数。

### RendezvousChannel

RendezvousChannel使用了一个“直接切换”场景，发送着将阻塞，直到另一方调用该通道的receive\(\)方法，反之亦然。在内部，这个实现与QueueChanle使用SynchronousQueue（0容量的BlockingQueue实现）很相似。它在发送方和接收方在不同线程中但又不适合仅将消息简单的放入异步队列的场景很合适。换句话说，对于RendezvousChannel，发送者至少知道某个接受者已经接收了这个消息，而对于QueueChannel，这个消息将被存储到内部队列中，可能永远不会被接收到。

_所有这些基于队列的通道默认仅在内存中保存消息。当需要持久化时，可以将持久化MessageStore实现的引用提供给队列单元的message-store属性，或者可以将本地通道替换为持久化代理支持的通道，例如基于JMS的通道或通道适配器。第二种方法允许利用任何JMS提供的实现持久化消息，会在第21章“JMS支持”中讨论。当队列中不需要缓存，最简单的方法时使用下面要讨论的DirectChannel。_

RendezvousChannel对于实现请求-回复操作也很有用。发送者可以创建临时的、匿名RendezvousChannel实例，在构建消息时设置replyChannel头。在发送消息之后，发送者可以立即调用接收方法（可以提供超时值）用于等待回复消息时阻塞。这与Spring Integration的许多请求 - 回复组件内部使用的实现非常相似。

### DirectChannel

DirectChannel具有点对点的语义，但是与上述任何基于队列的通道实现相比，它更类似于PublishSubscribeChannel。它实现了SubscribableChannel接口而不是PollableChannel接口，所以它直接将消息分分给订阅者。但是作为一个点对点的通道，它不同于PublishSubscribeChannel，因为它只会将消息发送到单个订阅的MessageHandler。

除了作为最简单的点对点通道选择，它最重要的特性之一是它可以使单个线程作为通道的两端执行操作。例如，如果一个处理器订阅了DirectChannel，那么发送消息给该通道会在发送者的线程中触发处理器的handleMessage\(Message\)方法的直接调用，在send\(\)方法调用返回之前发生。

提供这种行为通道实现的关键动机是支持必须跨越通道的事务，同时仍然从通道提供的抽象和松散耦合中受益。如果在事务范围内调用发送方法，则处理程序调用（例如更新数据库记录）的结果将在确定该事务的最终结果（提交或回滚）中发挥作用。

_由于DirectChannel是最简单的选择，并且不会增加调度和管理轮询器线程所需的额外的开销，所以它是Spring Integration中的默认通道类型。总体思路是为应用程序定义通道，然后考虑哪些需要提供缓存或控制输入，然后修改这些为基于队列的PollableChannels。同样的，如果通道需要广播消息，它不应是DirectChannel而是PublishSubscribeChannel。下面将会看到如何配置它们。_

DirectChannel内部代理一个Message Dispatcher来调用它的订阅消息处理器，并且这个dispatcher可以通过load-balancer或load-balancer-ref属性持有负载均衡策略。负载均衡策略被Message Dispatcher用于在又多个Message处理器订阅相同通道的情况下帮助决定Message如果分发给Message处理器。load-balancer属性引用了LoadBalancingStrategy的预定义的实现值的枚举。round-robin（在处理程序中进行负载平衡）和none（想要显式的关闭负载均衡）是仅有的可用值。其它策略实现可能在未来的版本中添加。但是，从版本3.0开始，可以提供自定义的LoadBalancingStrategy实现，并使用load-balancer-ref属性注入它，该属性应该指向实现LoadBalancingStrategy的bean。

```
<int:channel id="lbRefChannel">
    <int:dispatcher load-balancer-ref="lib"/>
</int:channel>

<bean id="lb" class="foo.bar.SampleLoadBalancingStrategy"/>
```

注意，load-balancer或load-balancer-ref属性是互斥的。

负载均衡也与布尔属性failover一起工作。如果failover值为true（默认的），那么当前面的处理器抛出异常时，dispatcher会根据需要回退到任何后续处理程序。顺序由一个可选的order值确定，它定义在处理器本身；如果没有这个值，就由它们订阅的顺序决定。

如果某种情况需要一个dispatcher总是尝试调用第一个处理器，那么每次发生错误时都要以相同的固定顺序进行回退，所以不会提供负载均衡策略。换句话说，dispatcher也支持failover属性甚至当没有开启负载均衡时。但是，如果没有负载均衡，处理器的调用将始终按照它们的顺序从第一个开始。例如，如果对第一、第二、第三等有明确的定义，这个方法行得通。当使用命名空间支持，任何端点上的order属性会决定顺序。

_记住，负载均衡和failover属性仅当一个通道有多个订阅的消息处理器时应用。当使用命名空间支持，这意味着多个端点在input-channel属性中共享相同的通道引用。_

### ExecutorChannel

ExecutorChannel是支持与DirectChannel相同dispatcher配置的（负载均衡策略和failover布尔属性）点对点通道。两种分发通道类型的关键区别是ExecutorChannel代理一个TaskExecutor的实例来执行分发。这意味着第二个方法一般不会阻塞，但是这意味着处理器调用可能不会发生在发送者的线程中。因此它不支持跨发送者和接收处理器的事务。

_注意，有些场景下发送者可能会阻塞。例如，当使用TaskExecutor的情况下，拒绝策略会在客户端上进行限制（例如ThreadPoolExecutor.CallerRunsPolicy），发送线程会在线程池处于最大容量并且执行者的工作队列已满时直接执行该方法。由于这种情况只会以不可预测的方式发生，显然不能依赖事务。_

### Scoped Channel

Spring  Integration 1.0提供一个ThreadLocalChannel实现，但是从2.0开始被移除了。现在有一个更通用的方法来处理相同的需求——通过向通道添加一个scope属性。属性值可以是上下文中可用的任何范围名称。例如，在web环境中，某些范围可用，并且任何自定义范围实现都可以在上下文中注册。下面是一个基于ThreadLocal的范围应用于通道的例子，包括范围本身的注册。

```
<int:channel id="threadScopedChannel" scope="thread">
    <int:queue/>
</int:channel>

<bean class="org.springframework.beans.factory.config.CustomScopeConfigurer">
    <property name="scopes">
        <map>
            <entry key="thread" value="org.springframework.context.support.SimpleThreadScope"/>
        </map>
    </property>
</bean>
```

上面的通道内部也代理一个队列，但是通道被绑定到当前线程，也包括队列的内容。这样发送到通道的线程稍后将能够接收那些相同的消息，但是没有其它线程能够访问它们。尽管很少需要线程范围通道，但在使用DirectChannel来强制单线程操作而需要任何回复消息发送到终端通道的情况下，它们可能很有用。如果该终端通道是线程范围的，则原始发送线程可以从其收集回复。

现在，由于任何通道都可以被限定范围，所以除了Thread Local以外，还可以定义自己的范围。

## 4.1.3 通道拦截器

消息架构的一个优点是可以以非侵入的方式提供一致的行为并聚焦传递给系统的消息的有意义的信息。因为消息从MessageChannel发送和接收，这些通道提供了拦截发送和接收操作的机会。ChannelInterceptor策略接口为下面的每个操作提供方法：

```
public interface ChannelInterceptor {
    
    Message<?> preSend(Message<?> message, MessageChannel channel);
    
    void postSend(Message<?> message, MessageChannel channel, boolean sent);
    
    void afterSendCompletion(Message<?> message, MessageChannel channel, boolean sent, Exception ex);
    
    boolean preReceive(MessageChannel channel);
    
    Message<?> postReceive(Message<?> message, MessageChannel channel);
    
    void afterReceiveCompletion(Message<?> message, MessageChannel channel, Exception ex);
}
```

在实现接口之后，调用函数即可注册拦截器：

```
channel.addInterceptor(someChannelInterceptor);
```

返回Message实例的方法可用于转换Message，或者可以返回null以防止进一步处理（当然，任何方法都可以抛出RuntimeException）。同时，preReceive方法可以返回false来阻止接收的进一步操作。

_记住，receive\(\)调用仅与PollableChannel相关。事实上SubscribableChannel接口甚至不定义receive\(\)方法。这是因为当一个消息被发送到一个SubscribableChannel时，它将被直接发送给一个或多个订阅者，这取决于通道的类型（例如，PublishSubscribeChannel发送给所有订阅者）。因此，只有在拦截器应用于 PollableChannel时，才会调用preReceive\(..\)、postReceive\(..\)和afterReceiveCompletion\(..\)拦截器方法。_

Spring Integration也提供了Wire Tap模式的实现。它是一个简单的拦截器，将消息发送到另一个通道，而不会改变现有的流程。它对于调试和监测非常有用。“Wire Tap“一节展示例子。

由于很少需要实现所有的拦截器方法，所以一个ChannelInterceptorAdapter类也可以用于被扩展。它提供了没有操作的方法（void方法是空的，返回Message的方法返回Message本身，返回boolean的方法返回true）。因此，很易于扩展这个类并实现所需的方法。

```
public class CountingChannelInterceptor extends ChannelInterceptorAdapter {
    
    private final AtomicInteger sendCount = new AtomicInteger();
    
    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        sendCount.incrementAndGet();
        return message;
    }
}
```

_拦截器方法的调用顺序取决于通道。如上所述，基于队列的通道是仅有的接收方法首先拦截的。此外，发送拦截和接收拦截的关系取决于单独的发送者和接受者线程的时间。例如，如果接受者在等待消息时已经阻塞，那么顺序可以是preSend、preReceive、postReceive和postSend。但是如果发送者已经发送消息并返回之后接受者轮询，则顺序是：preSend、postSend、preReceive和postReceive。在这种情况下，时间取决于很多因此，因此通常是不可预测的（事实上，接收可能从不发生）。显然，队列的类型也扮演了重要的角色（例如rendezvous和proirty）。底线是不能超出preSend先于postSend和preReceive先于postReceive的事实。_

从Spring Framework 4.1和Spring Integration 4.1开始，ChannelInterceptor提供新的方法——afterSendCompletion\(\)和afterReceiveCompletion\(\)。它们在send\(\)/receive\(\)调用之后，不管是否有异常抛出，因此可用于资源清理。注意，通道按照出事preSend\(\)/preReceive\(\)调用的相反顺序在ChannelInterceptor列表上调用这些方法。















