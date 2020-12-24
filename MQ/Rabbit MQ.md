<center>Rabbit MQ</center>

- ==优势==

  异步、解耦、削峰（限流）

- ###### ==定义== 

  <center>第一章 RabbitMQ基础</center>

1.具有平台和供应商无关性的高级消息队列协议（ Advanced Message Queuing Protocol , AMQP ）。AMQP 规范不仅定义了一种网络协议，同时也定义了服务器端的服务和行为。。这些信息就是高级消息队列（Advanced Message Queuing, AMQ ）模型。针对代理服务器软件， AMQ模型在逻辑上定义了 种抽象组件用于指定消息的路由行为：

·交换器 Exchange ，消息代理服务器中用于把消息路由到队列的组件。

·队列 Queue ，用来存储消息的数据结构，位于硬盘或内存中。

·绑定 Binding 套规则，用于告诉交换器消息应该被存储到哪个队列

2.消息中间件（ Message-oriented middleware, MOM ）是 种软件或硬件基础设施，通过它可以在分布式系统中发送和接收消息. RabbitMQ 通过高级路由和消息分发功能巧妙地实现了这 角色，即使需要满足广域网（ Wide Area Network, ·WAN ）环境下实现可靠性所应达到的容错条件，分布式系统也可以很容易与其他系统进行互连。

3.RabbitMQ三大组件

交换器

交换器是 AMQ 模型定义的 种组件之一。 个交换器接收发送到 RabbitMQ 中的消息并决定把它们投递到何处。交换器定义消息的路由行为，通常这需要检查消息所携带的数据特性或者包含在消息体内的各种属性。RabbitMQ 拥有多种交换器类型，每 种类型具备不同的路由行为。另外，它还提供了种可用于自定义交换器的插件架构。图 1.9 展示了发布者发布消息到 RabbitMQ 中，然后通过交换器进行消息路由的逻辑视图。

![image-20200923141743228](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20200923141743228.png)

队列

队列负责存储接收到的消息，同时也可能包含如何处理消息的配置信息。队列可以把消息只存储在内存中，也可以存储在硬盘中，然后以先进先出 FIFO ）的顺序进行投递.

绑定

AMQ 模型使用绑定 bindi 来定义队列和交换器之间的关系。在 RabbitMQ ，绑定或绑定键 binding-key 即告知一个交换器应该将消息投递到明 些队列中。对于某些交换器类型，绑定同时告知交换器如何对消息进行过滤从而决定能够投递到队列的消息当发布一条消息到交换器时，应用程序使用路由键 routing key 属性。路由键可以是队列名称，也可以是一串用于描述消息、具有特定语法的字符串。当交换器对一条消息进行评估以决定路由到哪些合适的队列时，消息的路由键就会和绑定键进行比对.

<center>第二章    使用AMQ协议与Rabbit进行交互</center>

1.当与 AMQP 交互时，这个问候语就是协议头 protocol header ,由客户端发送给服务器。这个问候语不应该被认为是个请求，但是与其余的会话不同，这并不是个命令 RabbitMQ 通过 Connection Start 命令响应问候语来启动命令／响应序列， 客户端则使用 Connection .StartOk 响应帧来响应RPC 请求。

==AMQP 帧组件==

当使用命令与 RabbitMQ 进行交互时，执行这些命令所 的所有 数被封装在一个称为帧的数据结构中，中，帧对数据进行编码以便传输。帧为命令及其 数提供了 种有效的方式，用于在网络上进行编码和分隔 。

低层 AMQP 帧由五个不同的组件组成：

·帧类型

·信道编号

·以字节为单位的帧大小

·帧有效载荷

·结束字节标记（ SCII 206)

低层 AMQP 帧的头部是 个字段，这三个字段组合起来被称为帧头（frame header).第一个字段是指示帧类型的单个字节，第二个字段指定帧的信道，第三个字段携带帧有效载荷的字节大小。帧头和结束字节标记合在一起创建了帧的基本结构。在帧内部，位于头部之后和结束字节标记之前的内容就是帧的有效载荷。

![image-20200923144714716](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20200923144714716.png)

==帧类型==

AMQP 规范定义了五种类型的帧 协议头帧、方法帧、内容头帧、消息体帧及心跳帧。每种帧类型都有明确的目的，而有些帧的使用频率比其他的要高很多。

·协议头帧用于连接到 RabbitMQ ，仅使用一次。

·方法帧携带发送给 RabbitMQ 或从 RabbitMQ 接收到的 RPC 请求或响应。

·内容头帧包含一条消息的大小和属性

·消息体帧包含消息的内容。

·心跳帧在客户端与 RabbitMQ 之间进行传递，作为 种校验机制确保连接的两端都可用且在正常工作。

==将消息编组成帧==

我们使用方法帧、内 容头帧和消息体帧向 RabbitMQ 发布消息 发送的第一 个帧是携带命令和执行它所需参数（如交换器和路由键〉的方法帧。方法帧之后是内容帧，包含内容头和消息体。内容头帧包含消息属性以及消息体大小 AMQP 的帧大小有 个上限，如果消息体超过这个上限，消息内容将被拆分成多个消息体帧。这些 始终以相同的顺序发送方法帧、内容头帧以及 个或多个消息体帧。

当向 RabbitMQ 发送消息时，在方法帧中会发送 Basic.Publish命令，随后是一个带有消息属性的内容头帧，该内容头帧包括消息的内容 型和发送时间等。这些属性被封装在 AMQP 规范中所定义的 Basic.Properties 据结构中 。最后，消息内容被编组到一定数量的消息体帧中。

![image-20200923153551292](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20200923153551292.png)

==方法帧结构==

方法帧携带着构建RPC请求所需的类、方法以及相关参数。

方法帧有效载荷中的每个数据值都是按照数据类型采用特定的格式进行编码。这种编码格式旨在最大限度地减少网络传输中的字节大小，提供数据完整性，并确保数据编组和解组速度尽可能快。实际的格式根据数据类型的不同而不同，但通常表现为一个字节后跟数字数据，或者是一个字节后跟 个字节大小的宇段，其后则是文本数据。

![image-20200923161848700](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20200923161848700.png)

==内容头帧==

消息头帧还包含消息的各种属性，为 RabbitMQ 服务器和可能接收它的任何应用程序提供了对消息的描述。这些属性存储在 Basic .Properties 映射表中，可能包含描述消息内容的数据，也可能是完全空白。大多数客户端库将预先填充一小部分字段，比如内容类型和投递模式。

属性是编写消息的强大工具。它们可以用来在发布者和消费者之间就消息的内容创建契约，从而允许对消息进行大量的定制操作。

![image-20200923162512333](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20200923162512333.png)

==消息体帧==

消息的消息体帧与正在传输的数据类型无关，并且可能包含二进制或文本数据。无论你是发送如 JPEG 图片的二进制数据，或是序列化之后的 JSON、XML 格式数据，消息体帧都是消息中包含实际消息数据的结构。

消息属性和消息体组合在一起构成了数据的强大封装格式。将消息的描述性属性与内容无关的消息体结合起来，确保你可以使用 RabbitMQ 处理你认为合适的任何类型的数据。

![image-20200923162733939](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20200923162733939.png)

==声明交换器==

AMQP 规范中交换器和队列都有自己的类。使用 Exchange.Declare 命令可以创建交换器，该命令提供了定义交换器名称和类型的参数，以及用于消息处理的其他元数据。

一旦命令被发送 RabbitMQ 在创建了交换器之后将发送一个Exchange.DeclareOk方法帧作为响应。如果出于某种原因命令执行失败，则 RabbitMQ 将使用Channel.Close 令关 发送 Channel.Declare 命令的信道。该响应将包含一个数字回复编码和文本值，用于说明 Exchange.Declare 失败并关闭信道的原因。

![image-20200924083620347](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20200924083620347.png)

==声明队列==

一旦交换器 建成功，就可以通过发送 Queue.Declare 命令让 RabbitMQ 创建队列。像 Exchange.Declare 命令一样，该命令也会生成一个通信时序，如Queue.Declare 命令执行失败，信道将被关闭。

在声明一个队列时，多次发送同一个 Queue.Declare 命令并不会有任何副作用。RabbitMQ 不会处理后续的队列声明，只会返回队列相关的有用信息，比如队列中待处理消息的数量以及订阅该队列的消费者数量。

![image-20200924083735024](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20200924083735024.png)

==绑定队列到交换器==

一旦创建了交换器和队列，是时候将它 绑定在一起，如同 Queue.Declare命令，将队列绑定到交换器的 Queue.Bind 命令每次只能指定一个队列。与 Exchange.Declare和Queue.Declare 命令类似，在发出 Queue Bind 命令后，如果处理成功，你的程序会收到Queue .BindOk方法帧。

==发布消息到 RabbitMQ==

当发布消息到 RabbitMQ时，多个帧封装了发送到服务器的消息数据。在实际的消息内容到达 RabbitMQ之前，客户端应用程序发送 Basic.Publish方法帧、一个内容头帧和至少一个消息体帧。

当RabbitMQ 接收到一个消息的所有帧并确定下一步操作之前，它将检查方法帧以获取它所需要的信息。Basic.Publish方法帧携带消息的交换器名称和路由键。在检查这些数据时，RabbitMQ 会尝试将 Basic.Publish 帧中的交换器名称与配置交换器的数据库进行匹配。

当RabbitMQ 发现某个交换器与 Basic.Properties 方法帧中的交换器名称相匹配时，它将判断该交换器中的绑定信息，并通过路由键寻找匹配的队列。当消息与任一绑定的队列符合匹配标准时，RabbitMQ 服务器将以 FIFO 顺序将消息放入队列。放入队列数据结构中的并不是实际消息，而是消息的引用。当 RabbitMQ 准备投递消息时， 它将使用这个引用来编组消息并通过网络进行发送。这为发布到多个队列的消息提供了实质性的优化。当把消息发送到多个目标时，只保存消息的一个实例会占用较少的物理内存。某一个队列中的消息处理方式，无论是被消费、过期还是等待消费，都不会影响该消息在其他队列中的处理方式。一旦 RabbitMQ 不再需要这个消息，因为它的所有副本都已经被投递或者被删除了，消息数据将被从RabbitMQ的内存中移除。

默认情况下，只要没有消费者正在监听队列，消息就会被存储在队列中。当添加更多消息时，队列的 大小也会随之增加。 RabbitMQ将这些消息保存在内存中或写入磁盘，具体取决于消息 Basic.Properties 中指定的 delivery-mode 属性。

==RabbitMQ 消费消息==

要消费 RabbitMQ 队列中的消息，消费者应用程序通过发出 Basic.Cosume命令来订 RabbitMQ 中的队列。像其他同步命令一样 服务器将使用 Basic.ConsumeOK进行响应，让客户端知道它将 开闸门并释放一大堆消息，或者至少是一两条消息。

一旦发送完 Basic.Consume 命令，消费者将处于活跃状态，直到某些事件中的一个被触发。如果消费者想要停止接收消息，则可以发出 Basic.Cancel 命令。值得注意的是，这个命令是异步发出的，而 RabbitMQ 可能仍然在发送消息，所以消费者在接收Basic.CancelOk 响应帧之前仍然可以接收到 RabbitMQ 预分配给它的任意数量的消息。

消费消息时，有几个设置可以让 RabbitMQ 知道你想如何接收它们。其中的一个设置是Basic. Consume 命令中的 no_ack 参数。当设置该参数为 true 时， RabbitMQ 将连续发送消息直到消费者发送一个 Basic.Cancel 命令或消费者断开连接为止。如果 no_ack志被设置为 false ，则消费者必须通过发送 Basic.Ack RPC 请求来确认收到的每条消息.

当发送 Basic.Ack 响应帧时，消费者必须在 Basic.Deliver 方法帧中传递一个名为投递标签 （delivery tag） 的参数。 RabbitMQ 使用投递标签和信道作为唯一标识符来实现消息确认、拒绝和否定确认。

![image-20200924103608618](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20200924103608618.png)


<center>第三章   消息属性详解</center>

==合理使用属性==

![image-20200924134054986](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20200924134054986.png)

使用 RabbitMQ 发布消息时，消息由 AMQP 规范中的三个低层 类型组成 Basic.Publish 方法帧、内容头帧和消息体帧，这三种帧类型按顺序一起工作，以便将消息传递到它应该去的地方并确保它到达时完好无损。

包含在消息头帧中的消息属性是一组预定义的值，这些值通过 Basic.Properties数据结构进行指定。某些属性（如 delivery-mode ）在 AMQP 规范中具有明确的含义，而有些属（如type）没有明确规范。

在某些情况下， RabbitMQ 使用明确定义的属性来实现消息的特定行为。前面提到的delivery-mode就是一 例子。在将消息放入队列时， delivery-mode 值将告诉RabbitMQ 把消息保存在内存前是否必须先把它存储到磁盘中。

![image-20200924134738532](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20200924134738532.png)

==使用 contet-type 属性创建显式的消息契约==

contet-type：该属性用于传输消息体中的数据格式。比如：application/json

==通过 gzip和content-encoding属性压缩消息大小==

content- encoding 属性在消息体体上应用特殊的编码，如base64或gzip

==不要混淆 content-encoding和content-type。与 HTTP 规范一样，content encoding 用于指示 content-type之外的某种编码级别。它是一个修饰符字段，通常用于表明消息体的内容已经使用 gzip 或其他形式的压缩方式进行了压缩。某些 AMQP 客户端自动将 content-encoding 值设置为 UTF-8，但这是不正确的行为。 AMQP 规范规定 content-encoding 用于存储 MIME内容编码。==

==使用 messag -id和correlation-id 引用消息==

在AMQP 规范中， message-id和correlation-id 是“应用级别使用”的属性并没有提供正式的行为定义。这意味着就规范而言，你可以利用它们实现任何目的。这两个字段允许多达 255 节的 UTF-8 编码数据，并以未压缩的方式存储在 Basic.Properties 数据结构中.

==message-id==

某些消息类型（如登录事件〉并不需要与其关联的唯一标识，但我们很容易想象如销售订单或支持类请求等的消息需要具备这个唯一标识。当消息流经松耦合系统中的各个组件时， message-id 属性使得消息能够在消息头中携带数据， 数据可以唯一地识别该消息。

==Correlation-id==

虽然在 AMQP 规范中没有关于correlation-id的正式定义，但它的一个用途是指定该消息是另一个消息的响应，通过携带关联消息的 message-id 可以做到这一点。另一种选择是使用它来传送关联消息的 事务或其他类似数据。

###### ==创建时间：times tamp 属性==

==expiration消息自动过期==

如果消息没有被消费， expiration 属性告诉RabbitMQ 何时应该丢弃消息。另外， expiration属性的规范定义有点奇怪，它被指定为“用于实现，但没有正式的行为”，这 Rabbi MQ 以提供它认为合理的任何实现方式。最后一个奇怪之处是它的格式是一个短字符申，最多允许 255字符，而代表时间单位的另 属性 timestamp 则是一个整数值。

想要利用 expiration 属性来RabbitMQ消息的自动过期，它必须包含一个UNIX 纪元时间或整数时间戳，然后把它存储为字符串。必须将字符串值设置为类似“ 132969600”的格式，而不是存储像“2002 -0 2-20TOO : 00 : 00-0。”这样的 IS0-8601 格式的时间戳。

使用 expiration属性时，如果把一个已过期的消息发布到服务器，则该消息不会被路由到任何队列，而是直接被丢弃。