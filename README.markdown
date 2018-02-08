# RabbitMQ

A how-to about _publishing_, _routing_ and _consuming_ messages with RabbitMQ and the Advanced Message Queueing Protocol (AMQP). RabbitMQ features a full implementation of AMQP 0.9.1 as well as several custom additions over it.

While this how-to focuses on RabbitMQ's AMQP implementation only, it goes to mention that RabbitMQ also features implementations of a few other messaging protocols that can be used for special use cases.
* MQTT a lightweight protocol often used to implement pub-sub patterns with mobile devices
* STOMP a text-based protocol creates compatibility with ApacheMQ
* statelessd / Hare for high velocity fire and forget messaging

---

## Publishing messages

When publishing messages, RabbitMQ offers multiple methods to pick from. Each choice is a trade-off between speed and security that messages have really been delivered. All options can be combined to find the sweet spot for the type of messages being send on a specific queue.

This list skips the Transaction pattern RabbitMQ implements (AMQP TX), as the alternatives offer more lightweight and less complex methods to achieve the same goals.

### Fastest, no guarantees

* fire and forget, messages might get lost, without informing the publisher about it
* set `mandatory=true` to get messages returned when they are not routable, returned messages have to be handled

### Publisher confirmation

* lightweight and quick way to ensure a message can be routed and the broker has processed it
* messages are acknowledged when all queues to which the message has been routed have either delivered the message and received an acknowledgement (if required), or enqueued the message (and persisted it if required)
* the Publisher should make no assumptions about the exact time a message is acknowledged
* confirmation is send asynchronously
* if the message can't be routed, a `nack` is returned

### High availability (HA) queues (in clustered mode only)

* messages will be handled on all servers of the cluster that handle the HA queue
* if a node goes down, the message does still exist on the other nodes
* this can replace slow persistence to disk in most cases
* once it is consumed from any node in the cluster, all copies will be removed
* using a cluster is mandatory to have a redundant system
* clusters can be connected in different ways, this is a topic of its own

### Persisted, slowest but most secure

* set `delivery-mode=2` to persist every message before queueing it
* messages will survive restarts of the whole system
* define the queues and the exchanges as `durable`
* _lazy queues_ are similiar, but even slower, as they remove the message from RAM
* only use _lazy queues_ when you expect extremely long queues, or suffer from unpredictable performance
* when the system goes down, messages that have not been persisted yet, might still get lost
* I/O performance of the system must be high, to not slow message handling down

---

## Consuming messages

When consuming messages, RabbitMQ offers multiple methods to pick from. Each choice is a trade-off between speed and security that messages have really been consumed.

This list skips the Transaction pattern RabbitMQ implements (AMQP TX), as the alternatives offer more lightweight and less complex methods to achieve the same goals.  
It also skips the `get` pattern, as the performance is worse compared to `consuming` messages. At the same time it does not offer benefits over the available alternatives.

### Fastest, no guarantees

* messages are consumed, but `no_ack` is sent
* no guarantee if a message was delivered or successfully processed
* the consumer might be overloaded and crash with an unknown number of buffered messages

### Quality of Service (QoS) level, ack after prefetch count reached

* set a QoS level, to determine how many messages to prefetch, before sending an `ack`
* consumer counts the number of messages before sending an `ack` when the `prefetch_count` was reached
* when connection dies, all messages of the current batch are re-queued
* reject a batch of `multiple` messages by sending a `nack`, they will be re-queued
* lightweight system that allows for high throughput rates
* needs benchmarking to find the perfect prefect_count per queue

### Delivery confirmation, (manual) ack, slowest but most secure

* send an automatic `ack` when a message was received, this is the default setting
* change it to `manual_ack=true` to send the `ack` after processing the message
* reject a message with `requeue=true` to redeliver it

---

## Examples

Real world use cases and some thoughts about which **publishing** and which **consumption methods** to chose.

### Logging / Metrics

Typical fire and forget use case. Losing some of the many millions of messages won't affect the system, but slow message handling would. Choose the _fastest publishing and consumption methods_ without guarantees.

### SMS-TAN / Email notifications

_Publishing confirmation_ can be set to ensure the message has been routed. In case you are sending many notifications per client per day, this might be the only guarantee you opt in to. You expect the delivery to succeed in almost all cases. And you are aware that there will never be a guarantee that a recipient will really receive an email is his inbox or a text message on this phone.

If you need the guarantee that your SMS or email system is processing the messages, setting a _QoS level_ and receiving an `ack` after multiple messages have been processed is a reasonable choice.

If it's not about pure notifications, but information that has to reach the customer, _dead-letter queues_ should be used to handle failures.

### Data changes

A customer changes his personal data on one system and it has to inform other application about it. In this case the choice of methods depends heavily on how important it is for the other applications to have the latest data.

An invoicing tool that sends out emails, might be ok with eventual consistency of the real world address of the customer. You could send messages with _publishing confirmation_ and no further guarantees.  
However you will need a strategy to create consistency eventually. Maybe a job that sends messages every night for each customer whose data changed in the last 24h. As you might have less system load at night, using a _QoS level_ might give you the guarantees you need.
Nevertheless the same tool might expect to always be immediately informed about changes in the payment data or email address. In this case you would need to pick more secure messaging strategies like a _HA queue_ and _manual `ack`_ after message delivery.

### Money transfers

When a redundant cluster is available, using _HA queues_ with _delivery confirmation_ and _dead-letter queues_ could be good choice.

Some money transfers might ask for even more guarantees, so that you will have to add _persistence of messages to disk_ into the mix. This should ensure that all messages can be recovered even in the case of catastrophic failure.

---

## Exchange routing

* to connect publishers and consumers at least one exchange has to be declared
* exchange names consist of letters, digits, hyphen, underscore, period, or colon `[A-Za-z0-9-_.:]`
* a default _direct exchange_ with the name `''` (empty string) exists
* RabbitMQ allows to bind exchanges to exchanges for extra flexibility, which comes with extra complexity and overhead, you most probably won't need this feature

### Queues

* after declaring an exchange, `bind` at least one queue to it
* queue names consist of letters, digits, hyphen, underscore, period, or colon `[A-Za-z0-9-_.:]`
* the empty string `''` is a valid queue name
* has the queue name been omitted on declaration, a new queue with a generated unique name will be created
* queues can be declared to be `exclusive` and will be deleted when the connection closes
* to enhance performance, multiple consumers can subscribe to a queue, messages are then distributed in a _round robin_ way, so that each consumer will process a part of all messages, which helps to keep queues short


### Direct exchanges

* match the `routing_key` of messages to the queues bound to the same key
* `[A-Za-z0-9-_.:]` are the only valid characters for `routing_keys` (TODO: validate this!)
* the string is matched for equality, no pattern matching
* multiple queues can be bound to the same `routing_key`
* a queue can be bound to multiple `routing_keys`
* every queue bound to a `routing_key` will receive all messages sent with this `routing_key`
* _direct exchanges_ are a good choice for routing reply messages

### Fanout exchanges

* all messages published are delivered to all queues bound to the exchange
* no routing keys are necessary
* faster routing, as not matching has to be performed
* all applications consuming from a _fanout exchange_ need to handle all kinds of messages delivered through it

### Topic exchanges

* `routing.keys` are dot separated
* `[A-Za-z0-9]` are the only valid characters besides the dot `.`
* pattern matching is used to route messages to queues
* an asterisk `*` matches any characters up to the next dot `.`
* the pound `#` matches any characters including dots `.`
* emulate _direct exchange_ queues by binding to the exact `routing.key`
* emulate _fanout exchange_ queues by binding to any routing.key through `#`
* _topic exchanges_ are very flexible and a good future-proof choice

### Headers exchanges

* inspects the `headers` hash of the message `properties` to route the message
* no routing keys are necessary
* `bind` accepts an array of `key-value` pairs as parameter
* the `x-match` argument determines if `any` key-value pair has to be included in the properties headers, or if `all` have to match
* additional key-value pairs included in the headers property do not influence the routing


### Other exchanges

RabbitMQ provides other types of exchanges through plugins to enable special use cases.

### Handling failure

#### Alternate exchanges

When a published message can not be routed by the broker.

* define an `alternate-exchange` to route the message to, when the actual is unable to route it
* a `mandatory=true` message would be re-routed instead of being returned
* publishers won't have to handle returned messages
* declare an exchange and `bind` at least one queue to it
* messages stay in the `alternate-exchange` queue(s) for inspection, unless a consumer is defined

#### Dead-letter exchanges

When a queued message expires or is rejected by the consumer.

* define an `x-dead-letter-exchange` when declaring the queue
* the message headers will be changed
* messages stay in the dead-letter queue for inspection, unless it they expire or a consumer is defined
* handling processing failure with dead-letter queues is simpler than burdening the consumer client with it

---

## Routing examples

In a _fanout exchange_ all queues will receive all messages. This burdens the consumers of this queues with handling all kind of messages, when different kinds of messages are sent to such an exchange.

In a _direct exchange_ every queue bound to the exact `routing_key` of a message, will receive this message. When multiple queues are bound to the same `routing_key` the message will be routed to all of these queues. A message will only be removed from the system, when it was delivered in all bound queues.

A _headers exchange_ allows for more flexible routing, similar to a _topics exchange_. Instead of analyzing the routing key, it matches `key-value` pairs of the `headers` `property` of messages.

### Topic exchange

The most flexible routing is implemented through the **topic exchange.** It is even capable to emulate the _direct_ and the _fanout exchange_. Pattern matching allows queues to handle all, some or only very specific messages. Let's assume our routing keys have usually three parts: an _application_, an _entity_ and an _event_:

### Routing keys

| Applications    | Entities | Events     |
|:----------------|:---------|:-----------|
| registrations   | business | created    |
| identifications | person   | successful |
| banking         | account  | failed     |
| notifications   | transfer | updated    |
| admin           |          | deleted    |
| webui           |          | error      |

The above list would allow for `routing.keys` like the following:

* `registrations.business.created`
* `registrations.person.updated`
* `banking.account.created`
* `banking.transfer.error`
* `notifications.person.updated`
* `admin.person.deleted`

### Binding queues to routing keys

Queues are bound to a pattern or the exact string:

| Routing-key                    | Explanation                                            | Possible Consumers         |
|:-------------------------------|:-------------------------------------------------------|:---------------------------|
| `registrations.#`              | every message of the `registrations` app               | Customer care              |
| `identifications.*.successful` | matches `person` and `business` entities of this event | Download ID, Notifications |
| `banking.account.created`      | a message when this exact `routing.key` is used        | Audits, Notifications      |
| `banking.transfer.*`           | `banking.transfer` messages of any event type          | Audits, Banking provider   |
| `#.error`                      | all `error` events from all apps and of any entity     | Alerts                     |
| `#`                            | all messages                                           | Logs                       |

Messages can be delivered to multiple queues that can have one or more consumers each.  
In this example the consumers _Audits_ and _Notifications_ consume multiple queues.  

---

## Messages

When discussing publishing, routing and consumption of messages, parameters like the routing key have been mentioned. The routing information and arguments, are part of the _method frame._ Together with the _content header frame_ and the _body frame_ it represents a full AMQP message.

### Headers

The _content header frame_ contains the size and the properties of a message.

| Property           | Type      | Used by      | Use case                                                                  |
|:-------------------|:----------|:-------------|:--------------------------------------------------------------------------|
| `app_id`           | String    | Application  | use it to describe the publisher                                          |
| `content-encoding` | String    | Application  | in case compression is used `gzip`, `zlib` or encoded content `Base64`    |
| `content-type`     | String    | Application  | mime-type of the message body `application/json`                          |
| `type`             | String    | Application  | use it to describe message or payload                                     |
|                    |           |              |                                                                           |
| `headers`          | Hash      | Rabbit & App | key-value table to store any additional metadata, can be used for routing |
| `message_id`       | String    | Application  | typically a UUID                                                          |
| `correlation_id`   | String    | Application  | typically used to reference a message_id when sending a response          |
|                    |           |              |                                                                           |
| `delivery-mode`    | 1 or 2    | RabbitMQ     | 1 = keep message in memory, 2 = persist to disk                           |
| `timestamp`        | timestamp | Application  | Unix epoch timestamp in seconds, use `Time.now.utc.to_i`                  |
|                    |           |              |                                                                           |
| `expiration`       | String    | RabbitMQ     | one way to set an expiration date, a timestamp in seconds but as String   |
| `priority`         | 0..9      | RabbitMQ     | 0 has highest priority, don't set or manipulate the value                 |
| `user_id`          | String    | RabbitMQ     | identify message was sent by logged-in user, don't manipulate it          |

### Payload

They payload is contained in the _body frame._ When the size of the payload exceeds the maximum size of a message frame, it will be split over multiple _body frames_. The form of the payload is described by the `content-type` and `content-encoding` _header properties_.

Consumers should decode and deserialize the message body, to make it easier for the application to handle messages.

---

## Code Examples

### Setting up a topic exchange and binding queues to it

### Publishing messages with different settings

### Consuming messages

### Handling errors

---

## Cluster setup

TODO:

* setting up multipled nodes
* high availability (HA) queues
* federated exchanges
* benefits
* drawbacks
* configuration options

---

## Clients

To comfortably use AMQP with the RabbitMQ extensions, there are clients for basically all modern languages.

### Ruby: bunny

https://github.com/ruby-amqp/bunny

### Elixir: amqp

https://github.com/pma/amqp

### Go: amqp

https://github.com/streadway/amqp

---

## Links

* **AMQP 0-9-1 reference:** https://www.rabbitmq.com/amqp-0-9-1-reference.html (keep in mind that RabbitMQ implements more features on top of the official AMQP standard)

* **RabbitMQ download** https://www.rabbitmq.com/download.html

* **Managed hosting** https://www.cloudamqp.com/ (available in many different clouds and regions)

* **RabbitMQ in Depth** by Gavin M. Roy: https://www.manning.com/books/rabbitmq-in-depth (this book inspired me to summarize the information above - it's a great read, buy it yourself!)

---

Assembled by **Andreas Finger** in February 2018 in Barcelona :: [@mediafinger](http://mediafinger.com) on [Github](https://github.com/mediafinger) and [Twitter](https://twitter.com/mediafinger)
