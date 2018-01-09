# 3.1背景

Spring框架的关键主题之一是控制反转。从最广泛的意义上讲，这意味着框架代表在其上下文中管理的组件处理责任。这些组件本身被简化，因为它们免除了这些责任。例如，依赖注入减轻了查找或创建依赖关系的责任。同样，面向切面编程通过将业务组件模块化为可重用的切面来减轻通用交叉问题的业务组件。在每种情况下，最终结果都是一个更容易测试、理解、维护和扩展的系统。

此外，Spring框架和产品组合为构建企业级应用程序提供了全面的编程模型。开发人员从这个模型的一致性中受益匪浅，特别是它基于已经建立的最佳实践，如编程接口和有利于继承的组合。 Spring的简化抽象和强大的支持库提高了开发人员的生产力，同时提高了可测试性和可移植性。

Spring Integration由这些相同的目标和原则驱动。它将Spring编程模型扩展到消息传递领域，并以Spring现有的企业集成支持为基础，提供更高级别的抽象。它支持消息驱动的体系结构，在这种体系结构中，控制反转适用于运行时问题，比如何时执行某些业务逻辑以及应该发送响应的位置。它支持消息的路由和转换，以便在不影响可测试性的情况下集成不同的传输和不同的数据格式。换言之，消息传递和集成问题由框架处理，因此业务组件与基础架构进一步隔离，开发人员免除了复杂的集成责任。

作为Spring编程模型的扩展，Spring Integration提供了各种配置选项，包括注释、带有命名空间支持的XML、带有通用“bean”元素的XML、当然还有直接使用底层API。该API基于明确定义的策略接口和非侵入式委托适配器。 Spring Integration的设计灵感来自于Spring内部常见模式与Gregor Hohpe和Bobby Woolf（Addison Wesley，2004）在同名书中描述的著名的“企业集成模式“。已经阅读过这本书的开发人员应该立即对Spring Integration概念和术语感到熟悉。

# 3.2 目标和原则

Spring Integration被以下目标驱动：

* 为实现复杂的企业集成解决方案提供简单的模型；

* 在基于Spring的应用程序中使用异步、消息驱动的行为；

* 促进现有Spring用户直观、渐进的使用。

Spring Integration由以下原则指导：

* 组件应该松耦合以实现模块化和可测试性；
* 框架应该强化业务逻辑和集成逻辑之间的分离；
* 扩展点本质上应该是抽象的，但是应该在明确的边界内，以提高重用性和可移植性。

# 3.3 主要组件

从垂直角度看，分层架构有助于分离关注点，基于接口的层之间的契约促进了松耦合。基于Spring的应用程序通常是以这种方式设计的，而Spring框架和产品组合为遵循这种针对整个企业应用程序的最佳实践提供了坚实的基础。消息驱动的体系结构增加了一个水平的角度，但是这些相同的目标仍然是相关的。就像“分层架构”是一个非常通用和抽象的范例一样，消息传递系统通常遵循类似抽象的“管道和过滤器”模型。 “过滤器”表示能够产生和/或消费消息的任何组件，“管道”在过滤器之间传输消息，使得组件本身保持松散耦合。需要指出的是，这两个高层次的范式并不是相互排斥的。支持“管道”的底层消息传递基础结构仍应被封装在将合约定义为接口的层中。同样，“过滤器”本身通常会在逻辑上位于应用程序服务层之上的层中进行管理，并通过与Web层大致相同的方式与这些服务进行交互。

## 3.3.1 消息

在Spring Integration中，消息是一个通用的包装器，用于包装任何Java对象和处理该对象时框架使用的元数据。它由有效载荷和标题组成。有效载荷可以是任何类型的，并且头部保存通常需要的信息，诸如id，时间戳，相关性id和返回地址。头也用于传递连接传输的值。例如，当从接收到的文件创建消息时，文件名可以存储在头中以供下游组件访问。同样，如果消息的内容最终将由出站邮件适配器发送，则上游组件可以将各种属性（来自cc，主题等）配置为消息头的值。开发人员也可以在头文件中存储任意的键值对。

![](https://docs.spring.io/spring-integration/docs/5.0.0.RELEASE/reference/html/images/message.jpg)

## 3.3.2 消息通道

消息通道表示管道和过滤器体系结构的“管道”。生产者将消息发送到通道，消费者从通道接收消息。因此，消息通道将消息组件解耦，并为消息的拦截和监视提供便利点。

![](https://docs.spring.io/spring-integration/docs/5.0.0.RELEASE/reference/html/images/channel.jpg)

消息通道可以遵循点对点或发布/订阅语义。使用点对点通道，至多一个消费者可以接收发送到该通道的每个消息。另一方面，发布/订阅通道将尝试把每个消息广播给其所有订阅者。 Spring集成对两者都支持。

尽管“点对点”和“发布/订阅”定义了有多少消费者将最终接收每条消息的两个选项，但还有一个重要的考虑因素：通道是否应该缓冲消息？在Spring Integration中，可轮询的通道能够缓冲队列中的消息。缓冲的好处是可以限制入站消息，从而防止消费者过载。然而，顾名思义，这也增加了一些复杂性，因为如果配置了轮询器，消费者只能从这样的频道接收消息。另一方面，连接到可订阅频道的消费者仅是消息驱动的。 Spring集成中可用的各种通道实现将在第4.1.2节“消息通道实现”中详细讨论。

## 3.3.3 消息端点

Spring Integration的主要目标之一是通过控制反转来简化企业集成解决方案的开发。这意味着开发者不必直接实现消费者和生产者，而且甚至不必在消息通道上构建消息并调用发送或接收操作。相反，开发者应该能够使用基于普通对象的实现来关注特定的域模型。然后，通过提供声明性配置，可以将特定于域的代码“连接”到Spring Integration提供的消息传递基础结构。负责这些连接的组件是消息端点。这并不意味着必须直接连接现有的应用程序代码。任何现实世界的企业集成解决方案都需要一定量的代码来集中关注路由和转换等集成问题。重要的是要实现这种集成逻辑和业务逻辑之间的关注点分离。换句话说，就像Web应用程序的Model-View-Controller范例一样，目标应该是提供一个精简但专用的层，将入站请求转换为服务层调用，然后将服务层返回值转换为出站应答。下一节将提供处理这些职责的消息端点类型的概述，在下面的章节中，将看到Spring Integration的声明性配置选项如何提供一种非侵入性的方式来使用其中的每一个类型。

# 3.4 消息端点

消息端点表示“管道和过滤器”体系结构的“过滤器”。如上所述，端点的主要作用是将应用程序代码连接到消息传递框架，并以非侵入的方式执行此操作。换句话说，应用程序代码应该理解为不知道消息对象或消息通道。这与MVC范例中的Controller的角色类似。就像Controller处理HTTP请求一样，Message Endpoint也处理消息。正如控制器被映射到URL模式一样，消息端点被映射到消息通道。在这两种情况下，目标是相同的：从基础设施中隔离应用程序代码。这些概念和下面所有的模式在“企业集成模式”一书中有详细讨论。在这里，我们只提供Spring Integration支持的主要端点类型及其角色的高级描述。下面的章节将详细阐述并提供示例代码以及配置示例。

## 3.4.1 转换器

消息转换器负责转换消息的内容或结构并返回修改后的消息。最常见的转换器可能是将消息的有效载荷从一种格式转换为另一种格式（例如，从XML文档转换为java.lang.String）。同样，可以使用转换器来添加、删除或修改消息头的值。

## 3.4.2 过滤器

消息过滤器决定是否应将消息传递到输出通道。这只需要一个布尔测试方法，可以检查特定的有效载荷内容类型、属性值和相关的头是否存在等。如果消息被接受，它被发送到输出通道，但是如果不是，它将被丢弃（或者对于更严格的实现，可能抛出异常）。消息过滤器通常与“发布预订”通道一起使用，其中多个消费者可能会收到相同的消息，并使用过滤器来根据某些条件缩小要处理的消息集。

_注意不要将“管道和过滤器”架构模式中的“过滤器”的通用用法与这种特定的端点类型相混淆，该类型可以选择性地缩小两个通道之间的消息流。 “过滤器和管道”概念中的过滤器概念与Spring Integration的消息端点更匹配：任何组件可以连接到消息通道以发送和/或接收消息。_

## 3.4.3 路由器

消息路由器负责决定接下来应该接收消息的通道（如果有的话）。通常，该决定基于消息的内容和/或消息头中可用的元数据。消息路由器通常用作服务激活器或其他能够发送回复消息的端点上静态配置的输出通道的动态替代方案。同样，消息路由器为多个订阅者使用的反应消息过滤器提供了主动的替代方案。

![](https://docs.spring.io/spring-integration/docs/5.0.0.RELEASE/reference/html/images/router.jpg)

## 3.4.4 分离器

分离器是另一种类型的消息端点，其职责是接受来自其输入通道的消息，将该消息拆分成多个消息，然后将每个消息发送到其输出通道。这通常用于将“组合的”有效载荷对象分成包含被细分的有效载荷的一组消息。

## 3.4.5 聚合器

基本上是分离器的镜像，聚合器是一种消息端点，它接收多个消息并将它们组合成单个消息。实际上，聚合器通常是包含分离器的管道的下游消费者。从技术上讲，聚合器比分离器更复杂，因为它需要维护状态（要聚合的消息），决定何时可以使用完整的消息组，并在必要时超时。此外，如果超时，聚合器需要知道是发送部分结果还是丢弃到单独的通道。 Spring Integration提供了CorrelationStrategy、ReleaseStrategy和可配置的设置：超时，是否发送超时部分结果，以及丢弃通道。

## 3.4.6 服务激活器

服务激活器是将服务实例连接到消息传递系统的通用端点。必须配置输入消息通道，并且如果需要调用的服务方法能够返回值，则也可以提供输出消息通道。

_输出通道是可选的，因为每个消息也可以提供它自己的返回地址头。同样的规则适用于所有消费者端点。_

服务激活器调用某个服务对象上的操作来处理请求消息，提取请求消息的有效负载并在必要时进行转换（如果该方法不期望消息类型的参数）。每当服务对象的方法返回一个值时，如果需要（如果它不是消息），那么返回值同样会被转换为回复消息。该回复消息被发送到输出通道。如果没有配置输出通道，则回复将被发送到消息“返回地址”中指定的通道（如果可用）。

请求回复“服务激活器通过”端点连接目标对象的方法来输入和输出消息通道。

![](https://docs.spring.io/spring-integration/docs/5.0.0.RELEASE/reference/html/images/handler-endpoint.jpg)

_如上面的消息通道所述，通道可以是可以轮询的或可订阅的；在该图中，由“时钟”符号和实线箭头（轮询）以及虚线箭头（订阅）来描绘。_

## 3.4.7 通道适配器

通道适配器是将消息通道连接到某个其他系统或传输的端点。通道适配器可以是入站或出站的。通常情况下，通道适配器将在消息和从其他系统（文件，HTTP请求，JMS消息等）接收或发送的任何对象或资源之间进行一些映射。根据传输，通道适配器也可以填充或提取消息头的值。 Spring Integration提供了许多通道适配器，将在以后的章节中对它们进行描述。

![](https://docs.spring.io/spring-integration/docs/5.0.0.RELEASE/reference/html/images/source-endpoint.jpg)

_消息源可以是可轮询的（例如POP3）或消息驱动的（例如，IMAP空闲）；在这个图中，由“时钟”符号和实线箭头（轮询）和虚线箭头（消息驱动）来描绘。_

![](https://docs.spring.io/spring-integration/docs/5.0.0.RELEASE/reference/html/images/target-endpoint.jpg)

_如上面的消息通道所述，通道可以是可以轮询的或可订阅的；在该图中，由“时钟”符号和实线箭头（轮询）以及虚线箭头（订阅）来描绘。_

# 3.5 配置和@EnableIntegration

在本文档中，将看到在Spring Integration流程中声明元素的XML名称空间支持的引用。这种支持是由一系列命名空间解析器提供的，它们生成适当的bean定义来实现特定的组件。例如，许多端点由一个MessageHandler bean和一个ConsumerEndpointFactoryBean组成，处理程序和输入通道名称被注入其中。

首次遇到Spring Integration命名空间元素时，框架会自动声明一些用于支持运行时环境（任务调度器，隐式通道创建器等）的bean。

_从版本4.0开始，引入了@EnableIntegration注解，以允许注册Spring Integration基础设施bean（请参阅JavaDocs）。仅当使用Java＆注解配置时，这个注解是必需的；例如，使用Spring Boot和/或Spring集成消息注解支持和Spring集成的Java DSL的无XML集成配置。_

当有一个没有Spring集成组件和两个或更多使用Spring集成的子上下文的父上下文时，@EnableIntegration注解也很有用。它使这些通用组件只在父上下文中声明一次。

@EnableIntegration注解将许多基础结构组件注册到应用程序上下文中：

* 注册一些内建的bean，例如errorChannel及其LoggingHandler、轮询器的taskScheduler、jsonPath SpEL函数等；
* 添加了几个BeanFactoryPostProcessor来增强用于全局和默认集成环境的BeanFactory；

* 添加几个BeanPostProcessor来增强和转换和包装特定的bean，以便集成;

* 添加注解处理器来解析消息注解，并为应用程序上下文注册它们的组件。

@IntegrationComponentScan注解也被引入来允许类路径扫描。这个注解和标准的Spring Framework @ComponentScan注解有着相似的作用，但是它仅限于Spring Integration特定的组件和注解，这是标准的Spring Framework组件扫描机制无法实现的。例如，第8.4.6节“@MessagingGateway注释”。

已引入@EnablePublisher注解用来注册PublisherAnnotationBeanPostProcessor bean，并为那些没有channel属性的@Publisher注解配置default-publisher-channel。如果找到多个@EnablePublisher注解，则它们必须都具有相同的默认通道值。有关更多信息，请参见第B.1.1节“通过@Publisher注解进行注解驱动的方法”。

引入了@GlobalChannelInterceptor注解来标记用于全局信道拦截的ChannelInterceptor bean。这个注解是&lt;int：channel-interceptor&gt; xml元素的一个模拟（参见“全局通道拦截器配置”一节）。 @GlobalChannelInterceptor注解可以放置在类级别（带有@Component构造型注解）或@Configuration类中的@Bean方法上。无论哪种情况，这个bean都必须是一个ChannelInterceptor。

已引入的@IntegrationConverter注解，将Converter，GenericConverter或ConverterFactory bean标记为integrationConversionService的候选转换器。这个注解是&lt;int：converter&gt; xml元素的模拟（参见第8.1.6节“有效载荷类型转换”）。可以将@IntegrationConverter注解放在类级别（使用@Component构造型标注）或@Configuration类中的@Bean方法。

有关消息注解的更多信息，另请参见第E.6节“注释支持”。

# 3.6 编程注意事项

通常建议尽可能使用普通的Java对象（PO​​JO），并且仅在绝对必要时代码中暴露框架。有关更多信息，请参见第3.8节“POJO方法调用”。

如果将框架暴露给类，则需要考虑一些注意事项，特别是在应用程序启动过程中；其中一些在这里列出。

* 如果组件是ApplicationContextAware，通常不应该在setApplicationContext（）方法中使用ApplicationContext。仅存储一个引用，并将这些用法推迟到上下文生命周期的后期

* 如果组件是一个InitializingBean或者使用@PostConstruct方法，不要从这些初始化方法发送消息——当这些方法被调用时，应用上下文还没有被初始化，并且发送这样的消息可能会失败。如果您需要在启动过程中发送消息，请实现ApplicationListener并等待ContextRefreshedEvent。或者，实现SmartLifecycle，把你的bean放到后期，然后从start\(\)方法发送消息。

# 3.7 编程技巧和窍门

在使用XML配置时，为了避免出现错误的模式验证错误，可以使用“Spring感知的”IDE，例如Spring Tool Suite（STS）（或使用Spring插件的eclipse）或IntelliJ IDEA。这些IDE知道如何从类路径中解析正确的XML模式（使用jar中的META-INF / spring.schemas文件）。在使用STS或带有插件的eclipse时，一定要在项目上启用Spring Project Nature。

在互联网上托管的某些传统模块（版本1.0中存在的模块）的模式是1.0版本兼容的；如果您的IDE使用这些模式，您可能会看到不正确的错误信息。

这些在线模式中的每一个都有类似的警告：

_这个模式适用于Spring Integration Core的1.0版本。我们无法将其更新到当前模式，因为这会破坏任何使用1.0.3或更低版本的应用程序。对于后续版本，未版本化的模式从类路径中解析并从jar中获得。请参阅github：https://github.com/spring-projects/spring-integration/tree/master/spring-integration-core/src/main/resources/org/springframework/integration/config。_

受影响的模块有：

* core（spring-integration.xsd）；
* file;
* http;
* jms;
* mail;
* rmi;
* security;
* stream;
* ws;
* xml。

## 3.7.2 为Java和DSL配置查找类名

借助XML配置和Spring Integration命名空间支持，XML解析器隐藏了如何声明和连接目标bean。对于Java和注解配置，了解目标最终用户应用程序的框架API非常重要。

EIP实现的头等公民是消息、通道和端点（参见上文第3.3节“主要组件”）。他们的实现（合同）是：

* org.springframework.messaging.Message——参阅5.1节“消息”；
* org.springframework.messaging.MessageChannel——参阅4.1节“消息通道”；
* org.springframework.integration.endpoint.AbstractEndpoint——参阅4.2节“轮询”。

前两个很简单，分别了解如何实现，配置和使用足够了；最后一个值得更多的检查。

AbstractEndpoint在整个框架中广泛用于不同的组件实现；其主要实现是：

* EventDrivenConsumer，当订阅SubscribableChannel来监听消息；
* PollingConsumer，当从PollableChannel轮询消息。

使用消息注解和/或Java DSL，不应该担心这些组件，因为框架通过适当的注解和BeanPostProcessor自动生成它们。在手动构建组件时，应使用ConsumerEndpointFactoryBean来帮助根据所提供的inputChannel属性确定要创建的目标AbstractEndpoint消费者实现。

另一方面，ConsumerEndpointFactoryBean在框架中委托给另一个一类公民——org.springframework.messaging.MessageHandler。这个接口实现的目标是处理被通道中终端消费的消息。 Spring Integration中的所有EIP组件都是MessageHandler的实现，例如。 AggregatingMessageHandler，MessageTransformingHandler，AbstractMessageSplitter等；并且目标协议出站适配器也是实现，例如， FileWritingMessageHandler，HttpRequestExecutingMessageHandler，AbstractMqttMessageHandler等。当您使用Java和注解配置开发Spring Integration应用程序时，您应该查看Spring Integration模块以找到适用于@ServiceActivator配置的MessageHandler实现。例如，要发送一个XMPP消息（见第38章，XMPP支持），我们应该配置如下所示：

```
@Bean
@ServiceActivator(inputChannel = "input")
public MessageHandler sendChatMessageHandler(XMPPConnection xmppConnection) {
    ChatMessageSendingMessageHandler handler = new ChatMessageSendingMessageHandler(xmppConnection);

    DefaultXmppHeaderMapper xmppHeaderMapper = new DefaultXmppHeaderMapper();
    xmppHeaderMapper.setRequestHeaderNames("*");
    handler.setHeaderMapper(xmppHeaderMapper);

    return handler;
}
```

MessageHandler实现代表消息流的出站和处理部分。

入站消息流侧有自己的组件，分为轮询和监听行为。监听（消息驱动）组件很简单并且通常只需要一个目标类实现就可以生成消息。 Listening组件可以是单向的MessageProducerSupport实现，例如AbstractMqttMessageDrivenChannelAdapter和ImapIdleChannelAdapter；也可以是请求回复 - MessagingGatewaySupport实现，例如AmqpInboundGateway和AbstractWebServiceInboundGateway。

轮询入站端点适用于那些不提供监听器API或不适用于此类行为的协议。例如，任何基于文件的协议，FTP，任何数据库（RDBMS或NoSQL）等。

这些入站端点包含两个组件：轮询器配置，定期启动轮询任务以及消息源类，以从目标协议读取数据，并为下游集成流程生成消息。轮询器配置的第一个类是SourcePollingChannelAdapter。这是另外一个AbstractEndpoint实现，特定用于轮询来启动一个集成流程。通常，使用Messaging Annotations或Java DSL，不必担心这个类，框架会根据@InboundChannelAdapter配置或Java DSL Builder规范为其生成一个bean。

消息源组件对于目标应用程序开发更为重要，它们都实现了MessageSource接口，例如MongoDbMessageSource和AbstractTwitterMessageSource。考虑到这一点，我们用JDBC从RDBMS表读取数据的配置可能如下所示：

```
@Bean
@InboundChannelAdapter(value = "fooChannel", poller = @Poller(fixedDelay="5000"))
public MessageSource<?> storedProc(DataSource dataSource) {
    return new JdbcPollingChannelAdapter(dataSource, "SELECT * FROM foo where status = 0");
}
```

目标协议的所有入站和出站类都可以在特定的Spring Integration模块中找到，大多数情况下在相应的包中。例如spring-integration-websocket适配器是：

* o.s.i.websocket.inbound.WebSocketInboundChannelAdapter —— 实现MessageProducerSupport，在套接字上侦听帧并产生消息到通道;

* o.s.i.websocket.outbound.WebSocketOutboundMessageHandler  - 单向AbstractMessageHandler实现，将传入的消息转换为适当的帧并通过websocket发送。

如果熟悉Spring Integration XML配置，从版本4.3开始，可以在XSD元素定义中提供有关哪些目标类用于为适配器或网关声明bean的信息，例如：

```
<xsd:element name="outbound-async-gateway">
    <xsd:annotation>
		<xsd:documentation>
Configures a Consumer Endpoint for the 'o.s.i.amqp.outbound.AsyncAmqpOutboundGateway'
that will publish an AMQP Message to the provided Exchange and expect a reply Message.
The sending thread returns immediately; the reply is sent asynchronously; uses 'AsyncRabbitTemplate.sendAndReceive()'.
       </xsd:documentation>
	</xsd:annotation>
```

# 3.8 POJO方法调用

如在3.6节“编程注意事项”中所述，推荐使用POJO编程风格。例如

```
@ServiceActivator
public String myService(String payload) { ... }
```

在这种情况下，框架将提取一个String有效载荷，调用方法，并将结果包装在一个消息中发送给流中的下一个组件（原始头将被复制到新消息中）。实际上，如果你使用XML配置，你甚至不需要@ServiceActivator注解：

```
<int:service-activator ... ref="myPojo" method="myService" />
```

```
public String myService(String payload) { ... }
```

只要在类的公共方法中没有歧义，就可以省略方法属性。

进一步信息：

可以POJO方法中获取头信息：

```
@ServiceActivator
public String myService(@Payload String payload, @Header("foo") String fooHeader) { ... }
```

可以提取消息中的属性：

```
@ServiceActivator
public String myService(@Payload("payload.foo") String foo, @Header("bar.baz") String barbaz) { ... }
```

由于有许多不同的POJO方法调用可用，5.0之前的版本使用SpEL来调用POJO方法。与通常在方法中进行的实际工作相比，SpEL（甚至被解释）对于这些操作通常“足够快”。但是，从版本5.0开始，默认情况下会使用org.springframework.messaging.handler.invocation.InvocableHandlerMethod。这种技术通常比解释的SpEL更快执行，并且与其他Spring消息传递项目一致。 InvocableHandlerMethod类似于用于在Spring MVC中调用控制器方法的技术。有一些方法仍然总是使用SpEL调用；例子包括具有如上所述的解除引用属性的注释参数。这是因为SpEL有能力导航属性路径。

可能有一些我们还没有考虑到的其他角落案例也不能使用InvocableHandlerMethod。出于这个原因，在这些情况下，我们会自动回退到使用SpEL。

如果愿意，也可以使用UseSpelInvoker注解设置你的POJO方法，使其始终使用SpEL：

```
@UseSpelInvoker(compilerMode = "IMMEDIATE")
public void bar(String bar) { ... }
```

如果省略了compilerMode属性，则spring.expression.compiler.mode系统属性将确定编译器模式 - 有关编译的SpEL的更多信息，请参阅SpEL编译。























