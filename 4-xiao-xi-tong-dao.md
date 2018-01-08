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



























