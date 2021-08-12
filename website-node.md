# Rabbitmq Website Note

# 1. Queues

Queues may be named or not named (passing empty string).

## 1.1 Queues Properties

Following are properties that can be configured for a queue

1. name
2. durable
3. exclusive (to connection)
4. auto-delete (deleted when no consumers are connected)
5. arguments (plugins or broker specific features, e.g., TTL)

## 1.2 Declare New Queues

When we try to declare a new queue, the following scenario may happen:

1. If the queue doesn't exist previously, it will be created
2. If the queue is created already and the properties are the same, nothing happens.
3. If the queue is created already but with different properties, exceptions are thrown.

## 1.3 Queues' arguments

Arguments are optional, they are known as "x-arguments" according to AMQP, which is essentially a key-value map.

For example,

- **x-message-ttl** : time to live for the message
- **x-expires** : expiry time for queues that are not used
- **x-max-length** : the maximum numbers of messages for a queue, messages are discorded when such limit is met.
- **x-max-length-bytes** : the maximum size of a message in a queue

## 1.4 Message Ordering

- Messages in queues are ordered based on FIFO
- Ordering of messages delivery might be affected by
    - competing consumers
    - consumer priorities
    - message redeliveries

i.e., messages in the same queue are sent in order, but the order of their delivery is not.

## 1.5 Durability

- Queues can be:

    - durable:
        - metadata stored in disk
    - transient:
        - metadata stored in memory

- Messages in these queues can also be:
    - persistent:
        - if the queue is durable, and the message is persisitent, the message is recovered when the node restart
    - transient:
        - if the queue is durable, but the message is transient, the message will still be discarded after a restart

How to choose?

1. Durable queues are always preferred.
2. Throughput and latency is not affected by whether queue is durable or not.
3. Transient / Temporary queues are good for clients that are transient, and may be disconnnected after a while

## 1.6 Temporary Queues

1. It's good for some workload queus are supposed to be temporary.
2. It's not convenient for clients to delete the queue once they are done, nor is it a reliable option as well, so it's good for this scenario.

Three way to make queues deleted automatically:

- Exclusive Queues (to connection)
    - these queues are deleted when the TCP connection is lost or closed
- TTLs
    - these queues are deleted when TTL is reached
- Auto-delete queues
    - deletd when the last consumer is cancelled using `basic.cancel`
    - but it won't be deleted if it never has any arguments.

## 1.7 Message Acknowledgement

- automatic 
    - better throughput
- manual 
    - improved guarantees of message delivery

## 1.8 Message States (for enqueued messages)

1. ready for delivery
2. delivered but not yet acknowleged

# 2. Provider, Consumers, Exchange, Queue

In essence:

```
                     Exchange
                         |
publisher -> message -> queue -> message -> consumers
```

# 3. Recovery

There are a few types of recovery:

- Recover connections
- Recover channels
- Recover queues
- Recover exchanges
- Recover bindings
- Recover consumers

# 4. Message Properties & Delivery Metadata

- Delivery & Routing Details (set at routing time):
    - delivery tag
        - identifier of message
    - redelivered
        - true / false
    - exchange
        - the exchagne that routes the message
    - routing key
        - key used to route messag, depends on exchange type
    - consumer tags
        - identifier of consumer

- Message properties:
    - (except 'delivery mode', all the others are optional)
    - delivery mode (mandatory):
        - persistent / transient
    - type
        - app specific, e.g., "ordered created"
    - headers
        - key-value map metadata
    - content type
        - MIMIE e.g., 'application/json'
    - content encoding
        - used by app, e.g., 'grip'
    - messsage id
        - arbitrary ID
    - correlation id
        - used to correlate with responses
    - reply to
        - queue's name that the responses are routed to
    - expiration
        - message ttl
    - timestamp
        - provided by application
    - user id
        - validated if set
    - app id
        - application name

# 5. Pre-fetch

Pre-fetch is used to limit how many deliveries being transferred via the network.

# 6. Exclusivity

A consumer can be set to **exclusive**, so that only which can consume messages from the queue.

# 7. Single Active Consumer (SAC)

Single active consumer is set using **`x-single-active-consumer`**, and is an alternative to the **exclusivity** tag, and it's preferred.

- Multiple consumers can register to the same queue, but only one is active, and the other are ignored.
- Useful when messages must be processed in the same order as they are published.
- When the active consumer is disconnected to the broker for some reason, the remaining will fight for the only active consumer.

## 7.1 Without Single Active Consumer

Without SAC configured, messages are dipatched using **round-robin** to these consumers.

# 8. Consumer Priority

Consumers can be assigned different prioritie, one with higher priority will guarantee receiving messsages, but when multiple have same priority, messages are dispatched using round robin.

# 9. Concurrency Consideration

- Java & .Net clients guarantee that messages sent to a single channel will bedispatched in the same order as they were received regardless of degree of concurrency. However, the processing of these messages will be by nature under race conditions.

# 10. Publisher

In AMQP:

```
Publisher -> msg -> Exchange

Exchange -> routes -> Queue

                         bind
                          |
Publisher  ->  Exchange  <-> Queue
           |
        Channel
```

## 10.1 Common Errors For Publisher

- Channel error:
    - publishing to non-existing exchange.
- message cannot be routed due to undefined binding:
    - if property **mandatory** is set to false, this message may be discarded or republished to alternative one.
    - if property **mandatory** is set to true, this message is returned back to the publisher, thus a handler will need to be setup to handle the returned messages

# 11. AMQP Exchange Type

- Direct Exchange
    - based on routing key, and routed with a round robin style
- Fanout Exchange
    - all queues bound to it will receive the messages, it's broadcasting
- Topic Exchange
    - routed by both routing keys and topics
- Header Exchange
    - routed based on headers that are started with 'x-', i.e., custom headers.

# 12. Connection

AMQP uses TCP/TLS connections.

# 13. Channels

Channels can be seemed as a lightweight connections that share the same TCP connection.

# 14. Virtual Hosts

Virtual hosts are also called 'vhosts', that allow a single host to separate different environments (group of users, exhcanges, queues and so on).

# 15. Publisher Confirm

It's another strategy for keeping track of message delivery. It's similar to consumer acknowledgement. This is handled by the client library.

1. Publish message individually and handle confirm events asynchronously
2. Publish a batch of messages and wait for all confirms
3. Publish message individually and wait for confirmation synchronously

# 16. Metrics

- outgoing message rate
- publisher confirmation rate
- connection churn rata
- channel churn rate
- unroutable dropped message rate
- unroutable returned message rate
