# <a name="spring-cloud-stream-binder-rabbit-reference"></a>Spring Cloud Stream RabbitMQ Binder Reference Guide

**2.2.1.RELEASE**

This guide describes the RabbitMQ implementation of the Spring Cloud Stream Binder.
It contains information about its design, usage and configuration options, as well as information on how the Stream Cloud Stream concepts map into RabbitMQ specific constructs.

## Usage

To use the RabbitMQ binder, you can add it to your Spring Cloud Stream application, by using the following Maven coordinates:

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
</dependency>
```

Alternatively, you can use the Spring Cloud Stream RabbitMQ Starter, as follows:

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
```

## RabbitMQ Binder Overview

The following simplified diagram shows how the RabbitMQ binder operates:

![RabbitMQ Binder](./images/rabbit-binder.png)

By default, the RabbitMQ Binder implementation maps each destination to a `TopicExchange`.
For each consumer group, a `Queue` is bound to that `TopicExchange`.
Each consumer instance has a corresponding RabbitMQ `Consumer` instance for its group's `Queue`.
For partitioned producers and consumers, the queues are suffixed with the partition index and use the partition index as the routing key.
For anonymous consumers (those with no `group` property), an auto-delete queue (with a randomized unique name) is used.

By using the optional `autoBindDlq` option, you can configure the binder to create and configure dead-letter queues (DLQs) (and a dead-letter exchange `DLX`, as well as routing infrastructure).
By default, the dead letter queue has the name of the destination, appended with `.dlq`.
If retry is enabled (`maxAttempts > 1`), failed messages are delivered to the DLQ after retries are exhausted.
If retry is disabled (`maxAttempts = 1`), you should set `requeueRejected` to `false` (the default) so that failed messages are routed to the DLQ, instead of being re-queued.
In addition, `republishToDlq` causes the binder to publish a failed message to the DLQ (instead of rejecting it).
This feature lets additional information (such as the stack trace in the `x-exception-stacktrace` header) be added to the message in headers.
See the [`frameMaxHeadroom` property](#spring-cloud-stream-rabbit-frame-max-headroom) for information about truncated stack traces.
This option does not need retry enabled.
You can republish a failed message after just one attempt.
Starting with version 1.2, you can configure the delivery mode of republished messages.
See the [`republishDeliveryMode` property](#spring-cloud-stream-rabbit-republish-delivery-mode).

If the stream listener throws an `ImmediateAcknowledgeAmqpException`, the DLQ is bypassed and the message simply discarded.
Starting with version 2.1, this is true regardless of the setting of `republishToDlq`; previously it was only the case when `republishToDlq` was `false`.

**IMPORTANT:** Setting `requeueRejected` to `true` (with `republishToDlq=false` ) causes the message to be re-queued and redelivered continually, which is likely not what you want unless the reason for the failure is transient.
In general, you should enable retry within the binder by setting `maxAttempts` to greater than one or by setting `republishToDlq` to `true`.

See [RabbitMQ Binder Properties](#rabbit-binder-properties) for more information about these properties.

The framework does not provide any standard mechanism to consume dead-letter messages (or to re-route them back to the primary queue).
Some options are described in [Dead-Letter Queue Processing](#rabbit-dlq-processing).

**NOTE:** When multiple RabbitMQ binders are used in a Spring Cloud Stream application, it is important to disable 'RabbitAutoConfiguration' to avoid the same configuration from `RabbitAutoConfiguration` being applied to the two binders.
You can exclude the class by using the `@SpringBootApplication` annotation.

Starting with version 2.0, the `RabbitMessageChannelBinder` sets the `RabbitTemplate.userPublisherConnection` property to `true` so that the non-transactional producers avoid deadlocks on consumers, which can happen if cached connections are blocked because of a [memory alarm](https://www.rabbitmq.com/memory.html) on the broker.

**NOTE:** Currently, a `multiplex` consumer (a single consumer listening to multiple queues) is only supported for message-driven conssumers; polled consumers can only retrieve messages from a single queue.

## Configuration Options

This section contains settings specific to the RabbitMQ Binder and bound channels.

For general binding configuration options and properties, see the [Spring Cloud Stream core documentation](https://github.com/spring-cloud/spring-cloud-stream/blob/master/spring-cloud-stream-core-docs/src/main/asciidoc/spring-cloud-stream-overview.adoc#configuration-options).

### <a name="rabbit-binder-properties"></a>RabbitMQ Binder Properties

By default, the RabbitMQ binder uses Spring Boot's `ConnectionFactory`.
Conseuqently, it supports all Spring Boot configuration options for RabbitMQ.
(For reference, see the [Spring Boot documentation](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#common-application-properties)).
RabbitMQ configuration options use the `spring.rabbitmq` prefix.

In addition to Spring Boot options, the RabbitMQ binder supports the following properties:

**spring.cloud.stream.rabbit.binder.adminAddresses**
A comma-separated list of RabbitMQ management plugin URLs.
Only used when `nodes` contains more than one entry.
Each entry in this list must have a corresponding entry in `spring.rabbitmq.addresses`.
Only needed if you use a RabbitMQ cluster and wish to consume from the node that hosts the queue.
See [Queue Affinity and the LocalizedQueueConnectionFactory](https://docs.spring.io/spring-amqp/reference/html/_reference.html#queue-affinity) for more information.

  Default: empty.

**spring.cloud.stream.rabbit.binder.nodes**
A comma-separated list of RabbitMQ node names.
When more than one entry, used to locate the server address where a queue is located.
Each entry in this list must have a corresponding entry in `spring.rabbitmq.addresses`.
Only needed if you use a RabbitMQ cluster and wish to consume from the node that hosts the queue.
See [Queue Affinity and the LocalizedQueueConnectionFactory](https://docs.spring.io/spring-amqp/reference/html/_reference.html#queue-affinity) for more information.

  Default: empty.

**spring.cloud.stream.rabbit.binder.compressionLevel**
The compression level for compressed bindings.
See `java.util.zip.Deflater`.

  Default: `1` (BEST_LEVEL).

**spring.cloud.stream.binder.connection-name-prefix**
A connection name prefix used to name the connection(s) created by this binder.
The name is this prefix followed by `#n`, where `n` increments each time a new connection is opened.

  Default: none (Spring AMQP default).

### RabbitMQ Consumer Properties

**NOTE:** To avoid repetition, Spring Cloud Stream supports setting values for all channels, in the format of `spring.cloud.stream.default.<property>=<value>`.

The following properties are available for Rabbit consumers only and must be prefixed with `spring.cloud.stream.rabbit.bindings.<channelName>.consumer.`.

**acknowledgeMode**
The acknowledge mode.

  Default: `AUTO`.

**anonymousGroupPrefix**
When the binding has no `group` property, an anonymous, auto-delete queue is bound to the destination exchange.
The default naming stragegy for such queues results in a queue named `anonymous.<base64 representation of a UUID>`.
Set this property to change the prefix to something other than the default.

  Default: `anonymous.`.

**autoBindDlq**
Whether to automatically declare the DLQ and bind it to the binder DLX.

  Default: `false`.

**bindingRoutingKey**
The routing key with which to bind the queue to the exchange (if `bindQueue` is `true`).
For partitioned destinations, `-<instanceIndex>` is appended.

  Default: `#`.

**bindQueue**
Whether to declare the queue and bind it to the destination exchange.
Set it to `false` if you have set up your own infrastructure and have previously created and bound the queue.

  Default: `true`.

**consumerTagPrefix**
Used to create the consumer tag(s); will be appended by `#n` where `n` increments for each consumer created.
Example: `${spring.application.name}-${spring.cloud.stream.bindings.input.group}-${spring.cloud.stream.instance-index}`.

  Default: none - the broker will generate random consumer tags.

**containerType**
Select the type of listener container to be used.
See [Choosing a Container](https://docs.spring.io/spring-amqp/reference/html/_reference.html#choose-container) in the Spring AMQP documentation for more information.

  Default: `simple`

**deadLetterQueueName**
The name of the DLQ

  Default: `prefix+destination.dlq`

**deadLetterExchange**
A DLX to assign to the queue.
Relevant only if `autoBindDlq` is `true`.

  Default: 'prefix+DLX'

**deadLetterExchangeType**
The type of the DLX to assign to the queue.
Relevant only if `autoBindDlq` is `true`.

  Default: 'direct'

**deadLetterRoutingKey**
A dead letter routing key to assign to the queue.
Relevant only if `autoBindDlq` is `true`.

  Default: `destination`

**declareDlx**
Whether to declare the dead letter exchange for the destination.
Relevant only if `autoBindDlq` is `true`.
Set to `false` if you have a pre-configured DLX.

  Default: `true`.

**declareExchange**
Whether to declare the exchange for the destination.

  Default: `true`.

**delayedExchange**
Whether to declare the exchange as a `Delayed Message Exchange`.
Requires the delayed message exchange plugin on the broker.
The `x-delayed-type` argument is set to the `exchangeType`.

  Default: `false`.

**dlqDeadLetterExchange**
If a DLQ is declared, a DLX to assign to that queue.

  Default: `none`

**dlqDeadLetterRoutingKey**
If a DLQ is declared, a dead letter routing key to assign to that queue.

  Default: `none`

**dlqExpires**
How long before an unused dead letter queue is deleted (in milliseconds).

  Default: `no expiration`

**dlqLazy**
Declare the dead letter queue with the `x-queue-mode=lazy` argument.
See [Lazy Queues](https://www.rabbitmq.com/lazy-queues.html).
Consider using a policy instead of this setting, because using a policy allows changing the setting without deleting the queue.

  Default: `false`.

**dlqMaxLength**
Maximum number of messages in the dead letter queue.

  Default: `no limit`

**dlqMaxLengthBytes**
Maximum number of total bytes in the dead letter queue from all messages.

  Default: `no limit`

**dlqMaxPriority**
Maximum priority of messages in the dead letter queue (0-255).

  Default: `none`

**dlqOverflowBehavior**
Action to take when `dlqMaxLength` or `dlqMaxLengthBytes` is exceeded; currently `drop-head` or `reject-publish` but refer to the RabbitMQ documentation.

  Default: `none`

**dlqTtl**
Default time to live to apply to the dead letter queue when declared (in milliseconds).

  Default: `no limit`

**durableSubscription**
Whether the subscription should be durable.
Only effective if `group` is also set.

  Default: `true`.

**exchangeAutoDelete**
If `declareExchange` is true, whether the exchange should be auto-deleted (that is, removed after the last queue is removed).

  Default: `true`.

**exchangeDurable**
If `declareExchange` is true, whether the exchange should be durable (that is, it survives broker restart).

  Default: `true`.

**exchangeType**
The exchange type: `direct`, `fanout` or `topic` for non-partitioned destinations and `direct` or `topic` for partitioned destinations.

  Default: `topic`.

**exclusive**
Whether to create an exclusive consumer.
Concurrency should be 1 when this is `true`.
Often used when strict ordering is required but enabling a hot standby instance to take over after a failure.
See `recoveryInterval`, which controls how often a standby instance attempts to consume.

  Default: `false`.

**expires**
How long before an unused queue is deleted (in milliseconds).

  Default: `no expiration`

**failedDeclarationRetryInterval**
The interval (in milliseconds) between attempts to consume from a queue if it is missing.

  Default: 5000

<a name="spring-cloud-stream-rabbit-frame-max-headroom"></a>**frameMaxHeadroom**
The number of bytes to reserve for other headers when adding the stack trace to a DLQ message header.
All headers must fit within the `frame_max` size configured on the broker.
Stack traces can be large; if the size plus this property exceeds `frame_max` then the stack trace will be truncated.
A WARN log will be written; consider increasing the `frame_max` or reducing the stack trace by catching the exception and throwing one with a smaller stack trace.

  Default: 20000

**headerPatterns**
Patterns for headers to be mapped from inbound messages.

  Default: `['*']` (all headers).

**lazy**
Declare the queue with the `x-queue-mode=lazy` argument.
See [Lazy Queues](https://www.rabbitmq.com/lazy-queues.html).
Consider using a policy instead of this setting, because using a policy allows changing the setting without deleting the queue.

  Default: `false`.

**maxConcurrency**
The maximum number of consumers.
Not supported when the `containerType` is `direct`.

  Default: `1`.

**maxLength**
The maximum number of messages in the queue.

  Default: `no limit`

**maxLengthBytes**
The maximum number of total bytes in the queue from all messages.

  Default: `no limit`

**maxPriority**
The maximum priority of messages in the queue (0-255).

  Default: `none`

**missingQueuesFatal**
When the queue cannot be found, whether to treat the condition as fatal and stop the listener container.
Defaults to `false` so that the container keeps trying to consume from the queue -- for example, when using a cluster and the node hosting a non-HA queue is down.

  Default: `false`

**overflowBehavior**
Action to take when `maxLength` or `maxLengthBytes` is exceeded; currently `drop-head` or `reject-publish` but refer to the RabbitMQ documentation.

  Default: `none`

**prefetch**
Prefetch count.

  Default: `1`.

**prefix**
A prefix to be added to the name of the `destination` and queues.

  Default: "".

**queueDeclarationRetries**
The number of times to retry consuming from a queue if it is missing.
Relevant only when `missingQueuesFatal` is `true`.
Otherwise, the container keeps retrying indefinitely.
Not supported when the `containerType` is `direct`.

  Default: `3`

**queueNameGroupOnly**
When true, consume from a queue with a name equal to the `group`.
Otherwise the queue name is `destination.group`.
This is useful, for example, when using Spring Cloud Stream to consume from an existing RabbitMQ queue.

  Default: false.

**recoveryInterval**
The interval between connection recovery attempts, in milliseconds.

  Default: `5000`.

**requeueRejected**
Whether delivery failures should be re-queued when retry is disabled or `republishToDlq` is `false`.

  Default: `false`.

<a name="spring-cloud-stream-rabbit-republish-delivery-mode"></a>**republishDeliveryMode**
When `republishToDlq` is `true`, specifies the delivery mode of the republished message.

  Default: `DeliveryMode.PERSISTENT`

**republishToDlq**
By default, messages that fail after retries are exhausted are rejected.
If a dead-letter queue (DLQ) is configured, RabbitMQ routes the failed message (unchanged) to the DLQ.
If set to `true`, the binder republishs failed messages to the DLQ with additional headers, including the exception message and stack trace from the cause of the final failure.
Also see the [frameMaxHeadroom property](#spring-cloud-stream-rabbit-frame-max-headroom).

  Default: false

**transacted**
Whether to use transacted channels.

  Default: `false`.

**ttl**
Default time to live to apply to the queue when declared (in milliseconds).

  Default: `no limit`

**txSize**
The number of deliveries between acks.
Not supported when the `containerType` is `direct`.

  Default: `1`.

### Advanced Listener Container Configuration

To set listener container properties that are not exposed as binder or binding properties, add a single bean of type `ListenerContainerCustomizer` to the application context.
The binder and binding properties will be set and then the customizer will be called.
The customizer (`configure()` method) is provided with the queue name as well as the consumer group as arguments.

### Rabbit Producer Properties

**NOTE:** To avoid repetition, Spring Cloud Stream supports setting values for all channels, in the format of `spring.cloud.stream.default.<property>=<value>`.


The following properties are available for Rabbit producers only and
must be prefixed with `spring.cloud.stream.rabbit.bindings.<channelName>.producer.`.

**autoBindDlq**
Whether to automatically declare the DLQ and bind it to the binder DLX.

  Default: `false`.

**batchingEnabled**
Whether to enable message batching by producers.
Messages are batched into one message according to the following properties (described in the next three entries in this list): 'batchSize', `batchBufferLimit`, and `batchTimeout`.
See [Batching](https://docs.spring.io/spring-amqp//reference/html/_reference.html#template-batching) for more information.

  Default: `false`.

**batchSize**
The number of messages to buffer when batching is enabled.

  Default: `100`.

**batchBufferLimit**
The maximum buffer size when batching is enabled.

  Default: `10000`.

**batchTimeout**
The batch timeout when batching is enabled.

  Default: `5000`.

**bindingRoutingKey**
The routing key with which to bind the queue to the exchange (if `bindQueue` is `true`).
Only applies to non-partitioned destinations.
Only applies if `requiredGroups` are provided and then only to those groups.

  Default: `#`.

**bindQueue**
Whether to declare the queue and bind it to the destination exchange.
Set it to `false` if you have set up your own infrastructure and have previously created and bound the queue.
Only applies if `requiredGroups` are provided and then only to those groups.

  Default: `true`.

**compress**
Whether data should be compressed when sent.

  Default: `false`.

**confirmAckChannel**
When `errorChannelEnabled` is true, a channel to which to send positive delivery acknowledgments (aka publisher confirms).
If the channel does not exist, a `DirectChannel` is registered with this name.
The connection factory must be configured to enable publisher confirms.

  Default: `nullChannel` (acks are discarded).

**deadLetterQueueName**
The name of the DLQ
Only applies if `requiredGroups` are provided and then only to those groups.

  Default: `prefix+destination.dlq`

**deadLetterExchange**
A DLX to assign to the queue.
Relevant only when `autoBindDlq` is `true`.
Applies only when `requiredGroups` are provided and then only to those groups.

  Default: 'prefix+DLX'

**deadLetterExchangeType**
The type of the DLX to assign to the queue.
Relevant only if `autoBindDlq` is `true`.
Applies only when `requiredGroups` are provided and then only to those groups.

  Default: 'direct'

**deadLetterRoutingKey**
A dead letter routing key to assign to the queue.
Relevant only when `autoBindDlq` is `true`.
Applies only when `requiredGroups` are provided and then only to those groups.

  Default: `destination`

**declareDlx**
Whether to declare the dead letter exchange for the destination.
Relevant only if `autoBindDlq` is `true`.
Set to `false` if you have a pre-configured DLX.
Applies only when `requiredGroups` are provided and then only to those groups.

  Default: `true`.

**declareExchange**
Whether to declare the exchange for the destination.

  Default: `true`.

**delayExpression**
A SpEL expression to evaluate the delay to apply to the message (`x-delay` header).
It has no effect if the exchange is not a delayed message exchange.

  Default: No `x-delay` header is set.

**delayedExchange**
Whether to declare the exchange as a `Delayed Message Exchange`.
Requires the delayed message exchange plugin on the broker.
The `x-delayed-type` argument is set to the `exchangeType`.

  Default: `false`.

**deliveryMode**
The delivery mode.

  Default: `PERSISTENT`.

**dlqDeadLetterExchange**
When a DLQ is declared, a DLX to assign to that queue.
Applies only if `requiredGroups` are provided and then only to those groups.

  Default: `none`

**dlqDeadLetterRoutingKey**
When a DLQ is declared, a dead letter routing key to assign to that queue.
Applies only when `requiredGroups` are provided and then only to those groups.

  Default: `none`

**dlqExpires**
How long (in milliseconds) before an unused dead letter queue is deleted.
Applies only when `requiredGroups` are provided and then only to those groups.

  Default: `no expiration`

**dlqLazy**
Declare the dead letter queue with the `x-queue-mode=lazy` argument.
See [Lazy Queues](https://www.rabbitmq.com/lazy-queues.html).
Consider using a policy instead of this setting, because using a policy allows changing the setting without deleting the queue.
Applies only when `requiredGroups` are provided and then only to those groups.

**dlqMaxLength**
Maximum number of messages in the dead letter queue.
Applies only if `requiredGroups` are provided and then only to those groups.

  Default: `no limit`

**dlqMaxLengthBytes**
Maximum number of total bytes in the dead letter queue from all messages.
Applies only when `requiredGroups` are provided and then only to those groups.

  Default: `no limit`

**dlqMaxPriority**
Maximum priority of messages in the dead letter queue (0-255)
Applies only when `requiredGroups` are provided and then only to those groups.

  Default: `none`

**dlqTtl**
Default time (in milliseconds) to live to apply to the dead letter queue when declared.
Applies only when `requiredGroups` are provided and then only to those groups.

  Default: `no limit`

**exchangeAutoDelete**
If `declareExchange` is `true`, whether the exchange should be auto-delete (it is removed after the last queue is removed).

  Default: `true`.

**exchangeDurable**
If `declareExchange` is `true`, whether the exchange should be durable (survives broker restart).

  Default: `true`.

**exchangeType**
The exchange type: `direct`, `fanout` or `topic` for non-partitioned destinations and `direct` or `topic` for partitioned destinations.

  Default: `topic`.

**expires**
How long (in milliseconds) before an unused queue is deleted.
Applies only when `requiredGroups` are provided and then only to those groups.

  Default: `no expiration`

**headerPatterns**
Patterns for headers to be mapped to outbound messages.

  Default: `['*']` (all headers).

**lazy**
Declare the queue with the `x-queue-mode=lazy` argument.
See [Lazy Queues](https://www.rabbitmq.com/lazy-queues.html).
Consider using a policy instead of this setting, because using a policy allows changing the setting without deleting the queue.
Applies only when `requiredGroups` are provided and then only to those groups.

  Default: `false`.

**maxLength**
Maximum number of messages in the queue.
Applies only when `requiredGroups` are provided and then only to those groups.

  Default: `no limit`

**maxLengthBytes**
Maximum number of total bytes in the queue from all messages.
Only applies if `requiredGroups` are provided and then only to those groups.

  Default: `no limit`

**maxPriority**
Maximum priority of messages in the queue (0-255).
Only applies if `requiredGroups` are provided and then only to those groups.

  Default: `none`

**prefix**
A prefix to be added to the name of the `destination` exchange.

  Default: "".

**queueNameGroupOnly**
When `true`, consume from a queue with a name equal to the `group`.
Otherwise the queue name is `destination.group`.
This is useful, for example, when using Spring Cloud Stream to consume from an existing RabbitMQ queue.
Applies only when `requiredGroups` are provided and then only to those groups.

  Default: false.

**routingKeyExpression**
A SpEL expression to determine the routing key to use when publishing messages.
For a fixed routing key, use a literal expression, such as `routingKeyExpression='my.routingKey'` in a properties file or `routingKeyExpression: '''my.routingKey'''` in a YAML file.

  Default: `destination` or `destination-<partition>` for partitioned destinations.

**transacted**
Whether to use transacted channels.

  Default: `false`.

**ttl**
Default time (in milliseconds) to live to apply to the queue when declared.
Applies only when `requiredGroups` are provided and then only to those groups.

  Default: `no limit`

**NOTE:** In the case of RabbitMQ, content type headers can be set by external applications.
Spring Cloud Stream supports them as part of an extended internal protocol used for any type of transport -- including transports, such as Kafka (prior to 0.11), that do not natively support headers.

## Using Existing Queues/Exchanges

By default, the binder will automatically provision a topic exchange with the name being derived from the value of the destination binding property `<prefix><destination>`.
The destination defaults to the binding name, if not provided.
When binding a consumer, a queue will automatically be provisioned with the name `<prefix><destination>.<group>` (if a `group` binding property is specified), or an anonymous, auto-delete queue when there is no `group`.
The queue will be bound to the exchange with the "match-all" wildcard routing key (`#`) for a non-partitioned binding or `<destination>-<instanceIndex>` for a partitioned binding.
The prefix is an empty `String` by default.
If an output binding is specified with `requiredGroups`, a queue/binding will be provisioned for each group.

There are a number of rabbit-specific binding properties that allow you to modify this default behavior.

If you have an existing exchange/queue that you wish to use, you can completely disable automatic provisioning as follows, assuming the exchange is named `myExchange` and the queue is named `myQueue`:

* `spring.cloud.stream.binding.<binding name>.destination=myExhange`
* `spring.cloud.stream.binding.<binding name>.group=myQueue`
* `spring.cloud.stream.rabbit.bindings.<binding name>.consumer.bindQueue=false`
* `spring.cloud.stream.rabbit.bindings.<binding name>.consumer.declareExchange=false`
* `spring.cloud.stream.rabbit.bindings.<binding name>.consumer.queueNameGroupOnly=true`

If you want the binder to provision the queue/exchange, but you want to do it using something other than the defaults discussed here, use the following properties.
Refer to the property documentation above for more information.

* `spring.cloud.stream.rabbit.bindings.<binding name>.consumer.bindingRoutingKey=myRoutingKey`
* `spring.cloud.stream.rabbit.bindings.<binding name>.consumer.exchangeType=<type>`

* `spring.cloud.stream.rabbit.bindings.<binding name>.producer.routingKeyExpression='myRoutingKey'`

There are similar properties used when declaring a dead-letter exchange/queue, when `autoBindDlq` is `true`.

## Retry With the RabbitMQ Binder

When retry is enabled within the binder, the listener container thread is suspended for any back off periods that are configured.
This might be important when strict ordering is required with a single consumer. However, for other use cases, it prevents other messages from being processed on that thread.
An alternative to using binder retry is to set up dead lettering with time to live on the dead-letter queue (DLQ) as well as dead-letter configuration on the DLQ itself.
See [RabbitMQ Binder Properties](#rabbit-binder-properties) for more information about the properties discussed here.
You can use the following example configuration to enable this feature:

* Set `autoBindDlq` to `true`.
The binder create a DLQ.
Optionally, you can specify a name in `deadLetterQueueName`.
* Set `dlqTtl` to the back off time you want to wait between redeliveries.
* Set the `dlqDeadLetterExchange` to the default exchange.
Expired messages from the DLQ are routed to the original queue, because the default `deadLetterRoutingKey` is the queue name (`destination.group`).
Setting to the default exchange is achieved by setting the property with no value, as shown in the next example.

To force a message to be dead-lettered, either throw an `AmqpRejectAndDontRequeueException` or set `requeueRejected` to `true` (the default) and throw any exception.

The loop continue without end, which is fine for transient problems, but you may want to give up after some number of attempts.
Fortunately, RabbitMQ provides the `x-death` header, which lets you determine how many cycles have occurred.

To acknowledge a message after giving up, throw an `ImmediateAcknowledgeAmqpException`.

### Putting it All Together

The following configuration creates an exchange `myDestination` with queue `myDestination.consumerGroup` bound to a topic exchange with a wildcard routing key `#`:

```
spring.cloud.stream.bindings.input.destination=myDestination
spring.cloud.stream.bindings.input.group=consumerGroup
#disable binder retries
spring.cloud.stream.bindings.input.consumer.max-attempts=1
#dlx/dlq setup
spring.cloud.stream.rabbit.bindings.input.consumer.auto-bind-dlq=true
spring.cloud.stream.rabbit.bindings.input.consumer.dlq-ttl=5000
spring.cloud.stream.rabbit.bindings.input.consumer.dlq-dead-letter-exchange=
```

This configuration creates a DLQ bound to a direct exchange (`DLX`) with a routing key of `myDestination.consumerGroup`.
When messages are rejected, they are routed to the DLQ.
After 5 seconds, the message expires and is routed to the original queue by using the queue name as the routing key, as shown in the following example:

*Spring Boot application*

```java
@SpringBootApplication
@EnableBinding(Sink.class)
public class XDeathApplication {

    public static void main(String[] args) {
        SpringApplication.run(XDeathApplication.class, args);
    }

    @StreamListener(Sink.INPUT)
    public void listen(String in, @Header(name = "x-death", required = false) Map<?,?> death) {
        if (death != null && death.get("count").equals(3L)) {
            // giving up - don't send to DLX
            throw new ImmediateAcknowledgeAmqpException("Failed after 4 attempts");
        }
        throw new AmqpRejectAndDontRequeueException("failed");
    }

}
```

Notice that the count property in the `x-death` header is a `Long`.

## <a name="rabbit-error-channels"></a>Error Channels

Starting with version 1.3, the binder unconditionally sends exceptions to an error channel for each consumer destination and can also be configured to send async producer send failures to an error channel.
See [Error Handling](./index.html#spring-cloud-stream-overview-error-handling) for more information.

RabbitMQ has two types of send failures:

* Returned messages,
* Negatively acknowledged [Publisher Confirms](https://www.rabbitmq.com/confirms.html).

The latter is rare.
According to the RabbitMQ documentation "[A nack] will only be delivered if an internal error occurs in the Erlang process responsible for a queue.".

As well as enabling producer error channels (as described in [Error Handling](./index.html#spring-cloud-stream-overview-error-handling)), the RabbitMQ binder only sends messages to the channels if the connection factory is appropriately configured, as follows:

* `ccf.setPublisherConfirms(true);`
* `ccf.setPublisherReturns(true);`

When using Spring Boot configuration for the connection factory, set the following properties:

* `spring.rabbitmq.publisher-confirms`
* `spring.rabbitmq.publisher-returns`

The payload of the `ErrorMessage` for a returned message is a `ReturnedAmqpMessageException` with the following properties:

* `failedMessage`: The spring-messaging `Message<?>` that failed to be sent.
* `amqpMessage`: The raw spring-amqp `Message`.
* `replyCode`: An integer value indicating the reason for the failure (for example, 312 - No route).
* `replyText`: A text value indicating the reason for the failure (for example, `NO_ROUTE`).
* `exchange`: The exchange to which the message was published.
* `routingKey`: The routing key used when the message was published.

For negatively acknowledged confirmations, the payload is a `NackedAmqpMessageException` with the following properties:

* `failedMessage`: The spring-messaging `Message<?>` that failed to be sent.
* `nackReason`: A reason (if available -- you may need to examine the broker logs for more information).

There is no automatic handling of these exceptions (such as sending to a [dead-letter queue](#rabbit-dlq-processing)).
You can consume these exceptions with your own Spring Integration flow.

## <a name="rabbit-dlq-processing"></a>Dead-Letter Queue Processing

Because you cannot anticipate how users would want to dispose of dead-lettered messages, the framework does not provide any standard mechanism to handle them.
If the reason for the dead-lettering is transient, you may wish to route the messages back to the original queue.
However, if the problem is a permanent issue, that could cause an infinite loop.
The following Spring Boot application shows an example of how to route those messages back to the original queue but moves them to a third "`parking lot`" queue after three attempts.
The second example uses the [RabbitMQ Delayed Message Exchange](https://www.rabbitmq.com/blog/2015/04/16/scheduling-messages-with-rabbitmq/) to introduce a delay to the re-queued message.
In this example, the delay increases for each attempt.
These examples use a `@RabbitListener` to receive messages from the DLQ.
You could also use `RabbitTemplate.receive()` in a batch process.

The examples assume the original destination is `so8400in` and the consumer group is `so8400`.

### Non-Partitioned Destinations

The first two examples are for when the destination is *not* partitioned:

```java
@SpringBootApplication
public class ReRouteDlqApplication {

    private static final String ORIGINAL_QUEUE = "so8400in.so8400";

    private static final String DLQ = ORIGINAL_QUEUE + ".dlq";

    private static final String PARKING_LOT = ORIGINAL_QUEUE + ".parkingLot";

    private static final String X_RETRIES_HEADER = "x-retries";

    public static void main(String[] args) throws Exception {
        ConfigurableApplicationContext context = SpringApplication.run(ReRouteDlqApplication.class, args);
        System.out.println("Hit enter to terminate");
        System.in.read();
        context.close();
    }

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @RabbitListener(queues = DLQ)
    public void rePublish(Message failedMessage) {
        Integer retriesHeader = (Integer) failedMessage.getMessageProperties().getHeaders().get(X_RETRIES_HEADER);
        if (retriesHeader == null) {
            retriesHeader = Integer.valueOf(0);
        }
        if (retriesHeader < 3) {
            failedMessage.getMessageProperties().getHeaders().put(X_RETRIES_HEADER, retriesHeader + 1);
            this.rabbitTemplate.send(ORIGINAL_QUEUE, failedMessage);
        }
        else {
            this.rabbitTemplate.send(PARKING_LOT, failedMessage);
        }
    }

    @Bean
    public Queue parkingLot() {
        return new Queue(PARKING_LOT);
    }

}
```

```java
@SpringBootApplication
public class ReRouteDlqApplication {

    private static final String ORIGINAL_QUEUE = "so8400in.so8400";

    private static final String DLQ = ORIGINAL_QUEUE + ".dlq";

    private static final String PARKING_LOT = ORIGINAL_QUEUE + ".parkingLot";

    private static final String X_RETRIES_HEADER = "x-retries";

    private static final String DELAY_EXCHANGE = "dlqReRouter";

    public static void main(String[] args) throws Exception {
        ConfigurableApplicationContext context = SpringApplication.run(ReRouteDlqApplication.class, args);
        System.out.println("Hit enter to terminate");
        System.in.read();
        context.close();
    }

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @RabbitListener(queues = DLQ)
    public void rePublish(Message failedMessage) {
        Map<String, Object> headers = failedMessage.getMessageProperties().getHeaders();
        Integer retriesHeader = (Integer) headers.get(X_RETRIES_HEADER);
        if (retriesHeader == null) {
            retriesHeader = Integer.valueOf(0);
        }
        if (retriesHeader < 3) {
            headers.put(X_RETRIES_HEADER, retriesHeader + 1);
            headers.put("x-delay", 5000 * retriesHeader);
            this.rabbitTemplate.send(DELAY_EXCHANGE, ORIGINAL_QUEUE, failedMessage);
        }
        else {
            this.rabbitTemplate.send(PARKING_LOT, failedMessage);
        }
    }

    @Bean
    public DirectExchange delayExchange() {
        DirectExchange exchange = new DirectExchange(DELAY_EXCHANGE);
        exchange.setDelayed(true);
        return exchange;
    }

    @Bean
    public Binding bindOriginalToDelay() {
        return BindingBuilder.bind(new Queue(ORIGINAL_QUEUE)).to(delayExchange()).with(ORIGINAL_QUEUE);
    }

    @Bean
    public Queue parkingLot() {
        return new Queue(PARKING_LOT);
    }

}
```

### Partitioned Destinations

With partitioned destinations, there is one DLQ for all partitions. We determine the original queue from the headers.

#### `republishToDlq=false`

When `republishToDlq` is `false`, RabbitMQ publishes the message to the DLX/DLQ with an `x-death` header containing information about the original destination, as shown in the following example:

```java
@SpringBootApplication
public class ReRouteDlqApplication {

	private static final String ORIGINAL_QUEUE = "so8400in.so8400";

	private static final String DLQ = ORIGINAL_QUEUE + ".dlq";

	private static final String PARKING_LOT = ORIGINAL_QUEUE + ".parkingLot";

	private static final String X_DEATH_HEADER = "x-death";

	private static final String X_RETRIES_HEADER = "x-retries";

	public static void main(String[] args) throws Exception {
		ConfigurableApplicationContext context = SpringApplication.run(ReRouteDlqApplication.class, args);
		System.out.println("Hit enter to terminate");
		System.in.read();
		context.close();
	}

	@Autowired
	private RabbitTemplate rabbitTemplate;

	@SuppressWarnings("unchecked")
	@RabbitListener(queues = DLQ)
	public void rePublish(Message failedMessage) {
		Map<String, Object> headers = failedMessage.getMessageProperties().getHeaders();
		Integer retriesHeader = (Integer) headers.get(X_RETRIES_HEADER);
		if (retriesHeader == null) {
			retriesHeader = Integer.valueOf(0);
		}
		if (retriesHeader < 3) {
			headers.put(X_RETRIES_HEADER, retriesHeader + 1);
			List<Map<String, ?>> xDeath = (List<Map<String, ?>>) headers.get(X_DEATH_HEADER);
			String exchange = (String) xDeath.get(0).get("exchange");
			List<String> routingKeys = (List<String>) xDeath.get(0).get("routing-keys");
			this.rabbitTemplate.send(exchange, routingKeys.get(0), failedMessage);
		}
		else {
			this.rabbitTemplate.send(PARKING_LOT, failedMessage);
		}
	}

	@Bean
	public Queue parkingLot() {
		return new Queue(PARKING_LOT);
	}

}
```

#### `republishToDlq=true`

When `republishToDlq` is `true`, the republishing recoverer adds the original exchange and routing key to headers, as shown in the following example:

```java
@SpringBootApplication
public class ReRouteDlqApplication {

	private static final String ORIGINAL_QUEUE = "so8400in.so8400";

	private static final String DLQ = ORIGINAL_QUEUE + ".dlq";

	private static final String PARKING_LOT = ORIGINAL_QUEUE + ".parkingLot";

	private static final String X_RETRIES_HEADER = "x-retries";

	private static final String X_ORIGINAL_EXCHANGE_HEADER = RepublishMessageRecoverer.X_ORIGINAL_EXCHANGE;

	private static final String X_ORIGINAL_ROUTING_KEY_HEADER = RepublishMessageRecoverer.X_ORIGINAL_ROUTING_KEY;

	public static void main(String[] args) throws Exception {
		ConfigurableApplicationContext context = SpringApplication.run(ReRouteDlqApplication.class, args);
		System.out.println("Hit enter to terminate");
		System.in.read();
		context.close();
	}

	@Autowired
	private RabbitTemplate rabbitTemplate;

	@RabbitListener(queues = DLQ)
	public void rePublish(Message failedMessage) {
		Map<String, Object> headers = failedMessage.getMessageProperties().getHeaders();
		Integer retriesHeader = (Integer) headers.get(X_RETRIES_HEADER);
		if (retriesHeader == null) {
			retriesHeader = Integer.valueOf(0);
		}
		if (retriesHeader < 3) {
			headers.put(X_RETRIES_HEADER, retriesHeader + 1);
			String exchange = (String) headers.get(X_ORIGINAL_EXCHANGE_HEADER);
			String originalRoutingKey = (String) headers.get(X_ORIGINAL_ROUTING_KEY_HEADER);
			this.rabbitTemplate.send(exchange, originalRoutingKey, failedMessage);
		}
		else {
			this.rabbitTemplate.send(PARKING_LOT, failedMessage);
		}
	}

	@Bean
	public Queue parkingLot() {
		return new Queue(PARKING_LOT);
	}

}
```

## Partitioning with the RabbitMQ Binder

RabbitMQ does not support partitioning natively.

Sometimes, it is advantageous to send data to specific partitions -- for example, when you want to strictly order message processing, all messages for a particular customer should go to the same partition.

The `RabbitMessageChannelBinder` provides partitioning by binding a queue for each partition to the destination exchange.

The following Java and YAML examples show how to configure the producer:

*Producer*

```java
@SpringBootApplication
@EnableBinding(Source.class)
public class RabbitPartitionProducerApplication {

    private static final Random RANDOM = new Random(System.currentTimeMillis());

    private static final String[] data = new String[] {
            "abc1", "def1", "qux1",
            "abc2", "def2", "qux2",
            "abc3", "def3", "qux3",
            "abc4", "def4", "qux4",
            };

    public static void main(String[] args) {
        new SpringApplicationBuilder(RabbitPartitionProducerApplication.class)
            .web(false)
            .run(args);
    }

    @InboundChannelAdapter(channel = Source.OUTPUT, poller = @Poller(fixedRate = "5000"))
    public Message<?> generate() {
        String value = data[RANDOM.nextInt(data.length)];
        System.out.println("Sending: " + value);
        return MessageBuilder.withPayload(value)
                .setHeader("partitionKey", value)
                .build();
    }

}
```

*application.yml*

```yaml
    spring:
      cloud:
        stream:
          bindings:
            output:
              destination: partitioned.destination
              producer:
                partitioned: true
                partition-key-expression: headers['partitionKey']
                partition-count: 2
                required-groups:
                - myGroup
```

**NOTE:** The configuration in the preceding example uses the default partitioning (`key.hashCode() % partitionCount`).
This may or may not provide a suitably balanced algorithm, depending on the key values.
You can override this default by using the `partitionSelectorExpression` or `partitionSelectorClass` properties.
The `required-groups` property is required only if you need the consumer queues to be provisioned when the producer is deployed.
Otherwise, any messages sent to a partition are lost until the corresponding consumer is deployed.

The following configuration provisions a topic exchange:

![Partition Exchange](./images/part-exchange.png)

The following queues are bound to that exchange:

![Partition Queues](./images/part-queues.png)

The following bindings associate the queues to the exchange:

![Partition Bindings](./images/part-bindings.png)

The following Java and YAML examples continue the previous examples and show how to configure the consumer:

*Consumer*

```java
@SpringBootApplication
@EnableBinding(Sink.class)
public class RabbitPartitionConsumerApplication {

    public static void main(String[] args) {
        new SpringApplicationBuilder(RabbitPartitionConsumerApplication.class)
            .web(false)
            .run(args);
    }

    @StreamListener(Sink.INPUT)
    public void listen(@Payload String in, @Header(AmqpHeaders.CONSUMER_QUEUE) String queue) {
        System.out.println(in + " received from queue " + queue);
    }

}
```

*application.yml*

```yaml
    spring:
      cloud:
        stream:
          bindings:
            input:
              destination: partitioned.destination
              group: myGroup
              consumer:
                partitioned: true
                instance-index: 0
```

**IMPORTANT:** The `RabbitMessageChannelBinder` does not support dynamic scaling.
There must be at least one consumer per partition.
The consumer's `instanceIndex` is used to indicate which partition is consumed.
Platforms such as Cloud Foundry can have only one instance with an `instanceIndex`.

# Appendices

## <a name="building"></a>Building

### Basic Compile and Test

To build the source you will need to install JDK 1.8.

The build uses the Maven wrapper so you don't have to install a specific
version of Maven. To enable the tests, you should have RabbitMQ server running
on localhost and the default port (5672)
before building.

The main build command is

```sh
$ ./mvnw clean install
```

You can also add '-DskipTests' if you like, to avoid running the tests.

**NOTE:** You can also install Maven (>=3.3.3) yourself and run the `mvn` command
in place of `./mvnw` in the examples below. If you do that you also
might need to add `-P spring` if your local Maven settings do not
contain repository declarations for spring pre-release artifacts.

**NOTE:** Be aware that you might need to increase the amount of memory
available to Maven by setting a `MAVEN_OPTS` environment variable with
a value like `-Xmx512m -XX:MaxPermSize=128m`. We try to cover this in
the `.mvn` configuration, so if you find you have to do it to make a
build succeed, please raise a ticket to get the settings added to
source control.


The projects that require middleware generally include a
`docker-compose.yml`, so consider using
[Docker Compose](https://compose.docker.io/) to run the middeware servers
in Docker containers.

### Documentation

There is a "docs" profile that will generate documentation.

`./mvnw clean package -Pdocs -DskipTests`

The reference documentation can then be found in `docs/target/contents/reference`.

### Working with the code
If you don't have an IDE preference we would recommend that you use
[Spring Tools Suite](https://www.springsource.com/developer/sts) or
[Eclipse](https://eclipse.org) when working with the code. We use the
[m2eclipe](https://eclipse.org/m2e/) eclipse plugin for maven support. Other IDEs and tools
should also work without issue.

#### Importing into eclipse with m2eclipse
We recommend the [m2eclipe](https://eclipse.org/m2e/) eclipse plugin when working with
eclipse. If you don't already have m2eclipse installed it is available from the "eclipse
marketplace".

Unfortunately m2e does not yet support Maven 3.3, so once the projects
are imported into Eclipse you will also need to tell m2eclipse to use
the `.settings.xml` file for the projects.  If you do not do this you
may see many different errors related to the POMs in the
projects.  Open your Eclipse preferences, expand the Maven
preferences, and select User Settings.  In the User Settings field
click Browse and navigate to the Spring Cloud project you imported
selecting the `.settings.xml` file in that project.  Click Apply and
then OK to save the preference changes.

**NOTE:** Alternatively you can copy the repository settings from [`.settings.xml`](https://github.com/spring-cloud/spring-cloud-build/blob/master/.settings.xml) into your own `~/.m2/settings.xml`.

#### Importing into eclipse without m2eclipse
If you prefer not to use m2eclipse you can generate eclipse project metadata using the
following command:

```sh
$ ./mvnw eclipse:eclipse
```

The generated eclipse projects can be imported by selecting `import existing projects`
from the `file` menu.

## <a name="contributing"></a>Contributing

Spring Cloud is released under the non-restrictive Apache 2.0 license,
and follows a very standard Github development process, using Github
tracker for issues and merging pull requests into master. If you want
to contribute even something trivial please do not hesitate, but
follow the guidelines below.

### Sign the Contributor License Agreement
Before we accept a non-trivial patch or pull request we will need you to sign the
[contributor's agreement](https://support.springsource.com/spring_committer_signup).
Signing the contributor's agreement does not grant anyone commit rights to the main
repository, but it does mean that we can accept your contributions, and you will get an
author credit if we do.  Active contributors might be asked to join the core team, and
given the ability to merge pull requests.

### Code Conventions and Housekeeping
None of these is essential for a pull request, but they will all help.  They can also be
added after the original pull request but before a merge.

* Use the Spring Framework code format conventions. If you use Eclipse
  you can import formatter settings using the
  `eclipse-code-formatter.xml` file from the
  [Spring
  Cloud Build](https://github.com/spring-cloud/build/tree/master/eclipse-coding-conventions.xml) project. If using IntelliJ, you can use the
  [Eclipse Code Formatter
  Plugin](https://plugins.jetbrains.com/plugin/6546) to import the same file.
* Make sure all new `.java` files to have a simple Javadoc class comment with at least an
  `@author` tag identifying you, and preferably at least a paragraph on what the class is
  for.
* Add the ASF license header comment to all new `.java` files (copy from existing files
  in the project)
* Add yourself as an `@author` to the .java files that you modify substantially (more
  than cosmetic changes).
* Add some Javadocs and, if you change the namespace, some XSD doc elements.
* A few unit tests would help a lot as well -- someone has to do it.
* If no-one else is using your branch, please rebase it against the current master (or
  other target branch in the main project).
* When writing a commit message please follow [these conventions](https://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html),
  if you are fixing an existing issue please add `Fixes gh-XXXX` at the end of the commit
  message (where XXXX is the issue number).
