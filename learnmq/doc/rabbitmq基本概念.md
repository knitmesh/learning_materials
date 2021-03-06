### AMQP(高级消息队列协议) ###

1. 消息通信概念
    * 消息: 消息包含两部分内容, 有效负载和标签

            有效负载就是传输的数据
            标签描述了有效负载, 并且RabbitMQ用它来决定谁将获得消息的拷贝
            其中标签包含了一个交换器的名称和可选的主题标记
    * 生产者: 生产者创建消息并设置标签, 然后发布到代理服务器(RabbitMQ)

    * 消费者: 生产者连接到代理服务器, 并订阅到队列(queue)上, 监听队列发送过来的消息, 但接收到的消息并不包含标签.
    因此, 如果消费者若想明确知道消息是由谁生产的, 就要看生产者是否把发送信息方的相关信息放入有效负载中

    * 信道: 信道是建立在真实的TCP连接内的虚拟连接. AMQP命令都是通过信道发送出去的. 不论是发布消息, 订阅队列或是接收消息.
    在一条TCP连接上创建多少消息是没有限制的. 因此, AMQP可以被看作加强版的传输层,
    使用信道, 可以减少TCP开销, 不被TCP连接约束所限制


2. AMQP消息路由
    1. 队列: 消息最终到达队列并等待消费, 队列为消息提供了处所

            如果至少有一个消费者订阅了队列, 消息会立即发送给这些订阅的消费者
            当队列拥有多个消费者时, 队列将以循环(round-robin)的方式将消息发送给消费者


        消费者通过以下两种方式从特定的队列中接收消息:

            (1)通过AMQP的basic.consume命令订阅: 这样会将信道置为接收模式.
               订阅了消息后, 消费者在空闲时, 消息一到达队列时就自动接收, 直到取消对队列的订阅为止

            (2)通过AMQP的basic.get命令订阅: 从队列获得单条消息而不是持续订阅

        消息的确认:

            消费者在接收到队列的每一条消息都必须进行确认, 或者在订阅到队列时就将auto_ack参数设置为True
            消息未被消费者确认时, Rabbitmq不会给该消费者发送下一条消息
            被确认的消息RabbitMQ将会把消息从队列中删除
            如果在消息确认之前与RabbitMQ断开连接(或取消订阅), Rabbitmq会认为这条消息没有分发, 然后重新分发给下一个订阅消费者

            注意: 直到消费者将消息处理完成之前, 绝不确认消息, 以免程序bug造成不可恢复错误, 导致消息丢失,
                  也可防止消息持续不断的涌向应用而导致过载

        只要消息未被确认, 则有以下两个选择拒绝消息:

            (1) 把消费者从RabbitMQ服务器断开连接. RabbitMq会把消息自动发送给下一个消费者
                优点: RabbitMq所有版本均支持
                缺点: 如果消费者在处理每条消息时都遇到错误, 这样连接/断开的连接方式会导致Rabbitmq潜在的重大负荷
            (2) 使用AMQP的basic.reject命令设置为true, 这将拒绝消息, Rabbitmq将把消息发送给下一个消费者
                前提是需要使用RabbitMQ2.0.0以上版本

                当basic.reject为false时, 将立即把消息在队列中删除, 使用这种方式丢弃消息的好处是, 消息将加入死信(dead letter)队列
                用来存放那些拒绝而不重新入队的消息. 可以通过死信队列检查拒绝/未送达消息来发现问题

                * 什么是死信:
                    消息被拒绝（basic.reject或basic.nack）并且requeue=false.
                    消息TTL过期
                    队列达到最大长度（队列满了，无法再添加数据到mq中）

         创建队列:

            消费者和生产者都能通过AMQP的queue.declare命令来创建队列.
            如果消费者在同一条信道上订阅了另一个队列的话就无法再声明队列了.必须先取消订阅, 将信道置为传输模式
            如果尝试声明一个已经存在的队列, 如果声明参数和已存在队列完全匹配, 将返回现存队列

            队列名称:
            当创建队列时, 我们可以指定队列名称, 消费者在订阅队列或创建绑定时需要指定队列名称.
            如果不指定队列名称的话, Rabbit会分配一个随机名称, 并在queue.declare命令响应中返回

         以下是队列设置中一些有用的参数:

            * exclusive --- 如果设置为true的话, 队列将变成私有的, 可限制一个队列只有一个消费者
            * auto-delete --- 当最后一个消费者取消订阅, 队列将自动删除, 结合exclusive参数可设置临时队列
            * queue.declare的passive选项置为true --- 检测队列是否存在

         队列应该由谁创建?

            消费者订阅队列, 且不能订阅一个不存在的队列
            生产者将消息发送出去的时候, 如果路由到了不存在的队列, Rabbit会忽略他们.
            因此, 如果不能承担得起消息丢失, 生产者和消费者都应该尝试去创建队列.

    2. 交换器（exchange）: 将发布的消息投递到队列, 使生产者无需关心队列和消费者

        绑定: 绑定决定了消息如何从路由器路由到特定的队列, 队列若想接收消息那么必须绑定一个交换器

        路由键(routing key): Rabbitmq通过一些确定的规则由交换器决定消息该投递到哪个队列, 这些规则被称作路由键.

            队列通过路由键绑定到交换器, 当消息被发送到代理服务器时, 消息将拥有一个路由键,
            即便是空的, Rabbitmq也会将队列绑定的路由键和其匹配,
            如果匹配成功, 那么消息会投递到该队列
            如果路由的消息不匹配任何绑定模式的话, 消息将进入黑洞

            如果对列与交换器绑定时不使用路由键, 那么所有发送给交换器没有路由键的消息都将发送至此队列

        交换器类型: 交换器是在RabbitMQ是一个实际存在的实体，不能被改变。只能删除之后，重新创建。

            * direct: 转发消息到routigKey指定的队列

                    服务器必须实现direct类型交换器, 包含一个空白字符串名称的默认交换器.
                    当声明一个队列时, 它会自动绑定到默认交换器, 并以队列名称作为路由键.
            * topic: 按规则转发消息（最灵活）

                    topic类型交换器通过模式匹配分析消息的routing-key属性。
                    它将routing-key和binding-key的字符串切分成单词。这些单词之间用点隔开。
                    支持表达式：
                    '.':  单个'.'把把路由键分割为了几个部分, 使用'*'匹配特定位置的任意文本
                          例如: *.1.geewu  //只要包含1.geewu就可以匹配相关信息

                    # : 匹配所有规则

            * fanout: 转发消息到所有绑定队列
            * headers: 不常用, 它允许匹配AMQP消息的header而非路由键, 除此之外和它和direct交换器完全一致


3. 多租户模式: 虚拟主机和隔离

        vhost: vhost本质上是一个mini班的Rabbitmq服务器, 他拥有自己的队列. 交换器和绑定. 以及独立的权限机制, vhost之间是绝对隔离的

4. 持久化消息:

    当服务器重启, 队列和交换器就都消失了(随同里面的消息). 原因在于每个队列和交换器的**durable**属性,
    该属性默认为false, 将他设置为true, 这样就不需要在服务器断电后重新创建队列和交换器了.

    所以, 队列和交换器的durable属性应该被设置为true, 但光这样做还不够.

    如果想让消息从rabbit崩溃中恢复, 那么消息必须:

        * 把他的投递模式设置为2 (持久)
        * 发送到持久化交换器
        * 发送到至就换的队列

    Rabbitmq确保持久性消息能从重启中恢复等方式是, 将它们写入磁盘上的一个持久化日志文件. 宕机后会根据日志重播.

    但需要强调的是必需保证以上三点, 否则消息到达非持久化队列/交换器时, rabbitmq会自动将此消息在日志中移除

    因持久化机制的写操作磁盘, 持久化消息开销很大, 且在内建集群环境下也难以保证消息的安全, 需要引入事务


5.  事务机制 VS 发送方确认模式(Publisher Confirm):

    如果采用标准的 AMQP 协议，则唯一能够保证消息不会丢失的方式是利用事务机制 -- 令 channel 处于 transactional 模式、

    向其 publish 消息、执行 commit 动作。在这种方式下，事务机制会带来大量的多余开销，并会导致吞吐量下降 250% 。



    为了补救事务带来的问题，引入了 confirmation 机制（即 Publisher Confirm）。

    为了使用 confirm 机制, 需要将信道设置成confirm模式, 首先要发送 confirm.select 方法帧.

    一旦在 channel 上使用 confirm.select方法，channel 就将处于 confirm 模式, 而且只能通过重新创建信道来关闭该模式.

    一旦信道进入confirm模式, 所有在信道上发布的消息都会指派一个唯一的ID号(从1开始)

    delivery-tag 域 的值标识了被 confirm 消息的序列号

    一旦消息被投递给所有匹配的队列后, 信道会发送一个发送方确认模式给生产者应用程序(包含ID)

    这使得生产者知晓消息已经安全到达目的队列了. 如果消息和队列是持久化的, 那么确认消息只会在队列将消息写入磁盘后才会发出

    发送方确认模式最大的好处是它们是异步的, 一旦发布了一跳消息, 生产者应用程序就可以在等待确认的同时继续发送下一条,

    当消息最终收到的时候, 生产者应用的回调方法就会触发来处理该确认消息.

    如果rabbir发生了内部错误导致了消息的丢失, rabbir会发送一条nack(not acknowledged 未确认)消息.

    在这种情况下，生产者应用可以选择将消息 re-publish .

    在信道被设置成 confirm 模式之后，所有被 publish 的后续消息都将被 confirm（即 ack） 或者被 nack 一次。

    但是没有对消息被 confirm 的快慢做任何保证，并且同一条消息不会既被 confirm 又被 nack 。

    由于没有事务回滚的概念, 因此发送方确认模式更加轻量级, 对Rabbit代理服务器的性能影响几乎可以忽略不计

6. 消息在什么时候确认

    broker 将在下面的情况中对消息进行 confirm ：

    broker 发现当前消息无法被路由到指定的 queues 中 （如果设置了  mandatory 属性，则broker会先发送  basic.return ）

    非持久属性的消息到达了其所应该到达的所有 queue 中（和镜像 queue 中）

    持久消息 到达了其所应该到达的所有 queue 中（和镜像 queue 中），并被持久化到了磁盘（被 fsync）

    持久消息从其所在的所有 queue 中被 consume 了 （如果必要则会被 acknowledge）

    broker 会丢失持久化消息，如果 broker 在将上述消息写入磁盘前异常。在一定条件下，这种情况会导致 broker 以一种奇怪的方式运行。 例如，考虑下述情景：

   1.  一个 client 将持久消息 publish 到持久 queue 中
   2.  另一个 client 从 queue 中 consume 消息（注意：该消息具有持久属性，并且 queue 是持久化的），当尚未对其进行 ack
   3.  broker 异常重启
   4.  client 重连并开始 consume 消息

   在上述情景下，client 有理由认为消息需要被（broker）重新投递

   但这并非事实：重启（有可能）会令 broker 丢失消息

   为了确保持久性，client 应该使用 confirm 机制

   如果 publisher 使用的 channel 被设置为 confirm 模式

   publisher 将不会收到已丢失消息的 ack（这是因为 consumer 没有对消息进行 ack ，同时该消息也未被写入磁盘）。