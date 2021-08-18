# Notes for RabbitMQ In Depth by Grain M. Roy

# 1. Chap 1 - RabbitMQ & Application Architecture

## 1.1 AMQP Advanced Message Queuing Protocol

Three abstract components:

1. Exchange 
    - Component that routes messages to queues
2. Queue
    - Data structures on disk or in memory that stores messages
3. Binding
    - Rules for exchange to route messages (A defined relationship between queues and exchanges)
    - In rabbitmq, for direct-exchange, it's simply binding keys (or, routing keys)

```
                        |  Inside RabbitMQ        |
                        |                         |
Publisher -> Message -> |  Exchange -> Queues ->  | -> Message -> Consumer
                        |                         |
                        |   (through bindings)    |
                        |                         |
```

# 2. Chap 2 - AMQP Protocol

## 2.1 Starts Conversation 

Conversation Initialization:

1. Client starts the conversation by sending a protocol header, which is neither a request nor a command.  
2. Server replies with a **`Connection.Start`**.
3. lient responds with **`Connection.StartOk`**.


```
    Client                         Server
       |                             |
       |                             |
 Protocol Header ---------------->   |
       |                             |
       | <---------------------  Connection.Start
       |                             |
 Connection.StartOk ------------->   |
       |                             |
       |                             |
```

## 2.2 Channels Over Connection

AMQP specifies Channels for communcation, which is essentially an unique id for each channel assigned to all messages, such that these conversations are isolated from each other within the same connection. This is sometime called **multiplexing**. RabbitMQ indeed sets up structures for each channel to manage the message flows for different conversation, however, it's important not to over-complicate things by opening lots of channels if it's not really needed.

## 2.3 AMQP's RPC Frame Structure

### 2.3.1 Command Structure

In AMQP, a command is composed of two components:  

- AMQP class
- AMQP method

E.g., for command **`Connection.Start`**, **`Connection`** is the class, and **`Start`** is the method. 

When commands are sent to and from RabbitMQ, all the arguments involved are encapsulated in data structures called **`frame`**. 

### 2.3.2 Frame

A AMQP Frame is composed of five components:

1. Frame type
2. Channel number
3. Frame size in bytes
4. Frame payload
5. End-byte marker (ASCII value 206)

The anatomy of a frame:

|Frame Type | Channel Number | Frame Size | Payload | End-byte marker |
--- | --- | --- | --- | ---
1 |0 | 335 |010101001110 | 0xce

### 2.3.3 Frame Header

The first three components are collectively called **Frame Header**:

- Frame Type 
    - single byte indicating frame type
- Channel number
    - field that specifies the channel 
- Frame size in bytes
    - byte size payload

### 2.3.4 Types of Frames

AMQP specifies five types of frames:

1. Protocol header frame
    - only used once for connection 
2. Method frame
    - carries RPC request or responses
3. Content header frame
    - size and properties for messages
4. Body frame
    - content of messages
5. Heartbeat frame
    - for heartbeat

I.e., when clients send messages, the method frame, content header frame and body frame are used. Protocol header frame, however, is only used for opening connection (not the one for TCP), and heartbeat frame is used to check if the opposite side is still operating. 

### 2.3.5 Marshelling Messages in Frames 

Below is what happens when client sends a message to RabbitMq: 

Let's say we are using `channel 1`.

1. Client sends a **Method Frame** to server
    - wherein the `frame_type=1` (for method frame), `channel=1`, and the payload contains the method that client is going to use, in this case, it's **``Basic.Publish`**, 
2. Client sends a **Content Header Frame** to server
    - wherein the `frame_type=2` (for content header frame), but channel is still the same. Payload now contains the properties of the message, including the size of it, so that RabbitMQ can properly handle the following one or more body frames. 
3. Client sends one or more **Body Frame** to server
    - wherein the `frame_type=3` (for body frame), but channel is still the same. Payload now contains the actual data of the message.

### 2.3.6 Anatomy of Method Frame

Method frames specifies the **class**, **method** as well as other arguments that the request is going to use. For **`Basic.Publish`**, the class will be **`Basic`** and the method will be **`Publish`**, the other arguments include **`Exchange Name`**, **`Routing Key`**, and **`Mendatory Flag`**. These are all encoded inside the method frame.

E.g., for **`Basic.Publish`** method frame:

|Class id |Method id | First argument | Second argument  | More argument |
---|---|---|---|---
Basic | Publish | Exchange Name | Routing key | mandatory flag for routing 

### 2.3.7 Anatomy of Content Header Frame

Content header frame contains properties of a message (a **Basic.Properties** table), the size of message is also included, but it's not considered as a property. 

The anatomy of a content header frame is as follows:

|Size of message | Which properties are set | Content type | app_id | timestamp | delivery mode | 
---|---|---|---|---|---
55 | 144, 200 | application/json | TestApp |  1014206980 | 1 

### 2.3.8 Anatomy of Body Frame

In body frame, all it takes care of is message data. The message body is not decoded, inspected or evaluated by RabbitMQ, it's merely a payload.

## 2.4 Protocol Use 

### 2.4.1 Declare An Exchange

Exchanges are created using **`Exchange.Declare`** command, server replies **`Exchange.DeclareOk`** when it was successful, otherwise, it replies **`Channel.Close`**.

```
If it was successful:

    Client                         Server
       |                             |
       |                             |
 Exchange.Declare --------------->   |
       |                             |
       |                             |
       | <---------------------  Exchange.DeclareOk
       |                             |
       |                             |

If it failed:

    Client                         Server
       |                             |
       |                             |
 Exchange.Declare --------------->   |
       |                             |
       |                             |
       | <---------------------  Channel.Close
       |                             |
       |                             |
```

### 2.4.2 Declaring A Queue

When exchange is created, client creates queue by sending **`Queue.Declare`** command, similiarly, server replies **`Queue.DeclareOk`** when it was successful, other, it replies **`Channel.Close`** to close the channel.

```
If it was successful:

    Client                         Server
       |                             |
       |                             |
 Queue.Declare  ----------------->   |
       |                             |
       |                             |
       | <---------------------  Queue.DeclareOk   
       |                             |
       |                             |

If it failed:

    Client                         Server
       |                             |
       |                             |
 Queue.Declare  ----------------->   |
       |                             |
       |                             |
       | <---------------------  Channel.Close     
       |                             |
       |                             |
```

### 2.4.3 Binding A Queue to An Exchange

To bind a queue to an exchange, client sends **`Queue.Bind`** command to server, this command can only bind one queue at a time. Server replies **`Queue.BindOk`** when it was successful.


```
If it was successful:

    Client                         Server
       |                             |
       |                             |
 Queue.Bind  -------------------->   |
       |                             |
       |                             |
       | <---------------------  Queue.BindOk      
       |                             |
       |                             |
```

### 2.4.4 Publishing Messages   

When client is about to publish a message, it will at least send three frames, 1) a `Basic.Publish` method frame to specify the method it will use, 2) a content header frame and 3) one or more body frames. 

When RabbitMQ receives the message, it will try to match the *exchange name* in the `Basic.Publish` method frame to verify if such an exchange actually exists. If the exchange doesn't exist, the mandatory flag is not set, this message is sliently dropped. RabbitMQ then evaluates the queue and routing keys, when such evaluation succeeds, it **enqueues the reference to the message instead of the actual message body**. The messages inside queue are stored in memory or disk depending on the **delivery-mode** property specified in content header frame.

```
    Client                         Server
       |                             |
       |                             |
 Basic.Publish (method frame) --->   |
       |                             |
       |                             |
 Content-Header Frame ----------->   |
       |                             |
       |                             |
 Body Frame --------------------->   |
       |                             |
```

### 2.4.5 Consuming Messages 

When a client wants to consume messages from a specified queue or topics, it sends a **`Basic.Consume`** command to the server, the server replies **`Basic.ConsumeOk`** to tell the client that, messages will be sent to the client to consume, if there is any. This method frame is followed by a **`Basic.Deliver`** method for actual message delivery, similar to **`Basic.Publish`**, this method frame will also be followed by content header frame and one or more body frames. 

```
    Client                         Server
       |                             |
       |                             |
 Basic.Consume (method frame) --->   |
       |                             |
       |                             |
       | <---------------------  Basic.ConsumeOk (method frame)
       |                             |
       |                             |
       | <---------------------  Basic.Deliver (method frame)
       |                             |
       |                             |
       | <---------------------  Content Header Frame 
       |                             |
       |                             |
       | <---------------------  Body Frame(s)
       |                             |
       |                             |
```

When the client wants to stop receiving messages, it may sends a **`Basic.Cancel`** command to the server, for which, the server will replies a **`Basic.CancelOk`** method frame.

```
    Client                         Server
       |                             |
       |                             |
 Basic.Cancel  (method frame) --->   |
       |                             |
       |                             |
       | <---------------------  Basic.CancelOk (method frame)
       |                             |
       |                             |
```

***no_ack flag***:

**`no_ack`** is an argument for **`Basic.Consume`**, When this argument is set to 'true', server will keep sending messages to client continuously until it requests a cancel. However, when this argument is set to 'false', server requires client to reply a **`Basic.Ack`** to acknowlege the receival of each message. The **`Basic.Ack`** will contain a identifier for the message that comes from **`Basic.Deliver`**, it's called the **Delivery Tag**. RabbitMQ uses the **delivery tag** and the **channel number** to uniquely identify each message delivery.

```
    Client                         Server
       |                             |
       |                             |
       | <---------------------  Basic.Deliver (delivery tag)
       |                             |
 Basic.Ack (delivery tag) ------->   |
       |                             |
```

# 3. Chap 3 - In-Depth Tour of Message Properties

## 3.1 Basic.Properties

Basic.Properties is a predefined data structure used to describe a message, it's sent as part of the *Content Header Frame*. Some properties have well-defined meanings in AMQP.

- content-type
    - e.g., json or xml
- content-encoding
    - encoding or compression, e.g., using gzip
- expiration
    - a string timestamp specifies when current message should be expire when it's not yet consumed, think of it as 'expire-at'
- reply-to
    - for application use.
- delivery-mode
    - a byte field that tells the broker, whether messages should be persisted in disk or stored in memory.
- message-id & correlation-id
    - for application use, no formaly defined behaviour. These two are often used to track messages that flow in/out the system. E.g., a reply `Mr` to a message `Mx` with message id `x'` may contain a correlation-id the value `x'`. Such that the receiver of message `Mr` will know that this message correlates to the message `Mx`.
- app-id & user-id
    - user-id is validated by RabbitMQ for authentication. These two are used to validate the source of the messages or to gather statistical data.
- priority
    - priority of message with a integer value 0-9, 0 is the most prioritised value.
- timestamp 
    - for application use. It's useful for evaluating performance. 
- type
    - for application use.
- cluster-id 
    -  removed in AMQP 0-9-1
- headers
    - key-value map for custom properties. Messages may be routed based on the key-value pairs in this 'headers'.

# 4. Chap 4 - Performance Trade-Offs in Publishing

## 4.1 Balacing Delivery Speed with Guaranteed Delivery

There is a trade-off between reliable message delivery and performance.  

*From performant to slow but reliable mechanisms:*

No Guarantees|Failure Notification|Publisher Confirms|Alternate Exchanges|
--- | --- |  --- | --- 
Very Fast |  Fast  | <- | <- |

HA Queues|Transactions|HA Queues with Transactions|Persisted Messages|
---  | ---   | ---   | ---  
-> | -> | Slow | Very Slow 


### 4.1.1 No Guarantees

The broker "fires and forgets" the messages, if the consumers are disconnected, they simply reconnect later and messages may not even arrive.

### 4.1.2 Reject Non-Routable Messages (Notification of Failure)

When **`mendatory flag`** is set on **Basic.Publish** method frame, RabbitMQ checks if the published message is routable (e.g., check specified exchange and binding). If it's found that the message is not routable, and it's mendatory, it returns the message back to the publisher via **`Basic.Return`** method frame. This is a kind of notification of failure. However, it this flag is not set at all, the messages will be silently dropped, no notification of failure is ever received by the publisher.

### 4.1.3 Publisher Confirms 

**Publisher Confirms** is a lightweight alternative to transactions, it's an extension to AMQP. It's used to assure the *message delivery between publisher and RabbitMQ server*. Prior to message publishing, publisher sends a **`Confirm.Select`** to server, server replies **`Confirm.SelectOk`** to tell that delivery confirmation is enabled between publisher and server. At this point, for every messages that publisher sends to server, RabbitMQ will respond an acknowledgement **`Basic.Ack`** or a negative acknowledgement **`Basic.Nack`**. E.g., if message can't be routed, RabbitMQ will respond a `Basic.Nack`.

```
    Client                         Server
       |                             |
 Confirm.Select ----------------->   |
       |                             |
       | <---------------------  Confirm.SelectOk      
       |                             |
 Basic.Publish 1) --------------->   |
       |                             |
       | <---------------------  Basic.Ack 1)        
       |                             |
 Basic.Publish 2) --------------->   |
       |                             |
       | <---------------------  Basic.Ack 2)
       |                             |
```

### 4.1.4 Alternate Exchanges

It's another extension to AMQP. An alternate exchange is specified when declaring an exchange (primary exchange) for the first time, the alternate exchange is a pre-existing exchange, that when the primary exchange is unable to route the published messages, the alternate exchagne is responsible for these messages. An alternate exchange is just like any other exchanges, if the alternate exchange is also unable to route the messages, the messages are lost. 

The alternate exchange is declared in the argument **`alternate-exchange`** in **`Exchange.Declare`**.

### 4.1.5 Batch Processing with Transactions

Transaction provides a way to notify publisher the successful delivery of a message to a queue on RabbitMQ. To begin a transaction, client sends a **`TX.Select`** request, and server will respond a **`TX.SelectOk`** indicating that the transaction is started. Then publisher can start sending one or more messages, and server may repond **`Basic.Return`** if a message is unroutable, then the publisher will decide what to do for this transaction. 

Transactions are explicitly committed or rollbacked using **`TX.Commit`** and **`TX.Rollback`**. Such transactions in RabbitMQ are atomic, which means that unless all messages are processed, client won't receive **`TX.CommitOk`** which indicates that a transaction is committed. Transaction in RabbitMQ only impact messages delivered to the same queue, and it may decrease performance of publisher, since publishers will have to wait a bit longer for the transaction to commit or rollback.

***"If you’re considering transactions as a method of delivery confirmation, consider using Publisher Confirms as a lightweight alternative—it'sfaster and can provide both positive and negative confirmation."*** (p.70)

### 4.1.6 Surviving Node Failures with HA Queues

HA Queues or **Highly Available Queues**, is uses to guarantee that messages wont't be lost inside the queue. HA requires clustered RabbitMQ, when a publisher sends message to the cluster, the message enqueues into random node, and these nodes inside the cluster synchronize these messages (making copies), this is also called **Mirroring**. Once a message (a copy) is consumed from any node, all copies are immediately removed.

Publisher can also explicitly specify nodes that this message should be mirrored on, using **`x-ha-policy: rabbit@node1, rabbit@node2, rabbit@node3`**, it's by default **`x-ha-policy: all`**.   

### 4.1.7 HA Queues with Transactions

HA Queues with Transactions is almost the same, except that the transaction won't commit until al nodes have a copy of the message, which is much slower than pure transactions.

### 4.1.8 Persisting Messages to Disk via Delivery-Mode 2

**`Delivery-Mode`** specifies whether message should be stored in memory-based queues or disk-based queues. By default the `Delivery-Mode` is set to 1, the messages are stored in memory, if the RabbitMq restarts, these messages are lost. However, if the `Delivery-Mode` is set to 2, RabbitMQ will ensure that the messages are stored to disk. *Additionally, for messages to truly survive after a restart, queues need to be durable as well*. Message persistence requires the servers to have sufficient I/O performance.

### 4.1.9 RabbitMQ Server Pushes Back

When publishers keep publishing data to server, under heavy load, server will eventually get overwhelmed. RabbitMQ server implements a mechanism called **TCP Backpressure**, which essentially stops socket to accept client's TCP incoming data. RabbitMQ uses the notion of **credits**, when a new connection is made, credits are assigned to the connection, for each RPC command is received by server, the credits decrement. When the conncection is out of credits, it's blocked, and RabbitMQ sends a **`Connection.Blocked`** and **`Connection.Unblocked`** to notify that the connection is now blocked or unblocked. This can be monitored as an indicator of whether the server is not properly sized, etc.

# 5. Chap 5 - Don't get messages, consume them

## 5.1 Basic.Get Vs. Basic.Consume

**`Basic.Get`** is not the ideal way to retrieve messages from server, **`Basic.Get`** is a polling model, and **`Basic.Consume`**` is a push model.

**Basic.Get** (sync)

```
    Client                         Server
       |                             |
 Basic.Get ---------------------->   |
       |                             |
       |                             |
       | <---------------------  Basic.GetOk           
       |                             |
       |                             |
       | <---------------------  Message 
       |                             |
       |                             |
 Basic.Ack ---------------------->   |
       |                             |
       |                             |
```

**Basic.Consume** (async)

```
    Client                         Server
       |                             |
 Basic.Consume ------------------>   |
       |                             |
       |                             |
       | <---------------------  Basic.ConsumeOk       
       |                             |
       |                             |
       | <---------------------  Message 1 
       |                             |
       |                             |
 Basic.Ack ---------------------->   |
       |                             |
       |                             |
       | <---------------------  Message 2
       |                             |
       |                             |
 Basic.Ack ---------------------->   |
       |                             |
       |                             |
```

### 5.1.1 Consumer Tag 

When client issues a **`Basic.Consume`**, a string (**consumer tag**) is created to uniquely identifies the consumer. This *consumer tag* is sent to the client every time the client receives a message. This *consumer tag* is also used to cancel consuming messages by including it in **`Basic.Cancel`**.

## 5.2 Performance-Tunning Consumers

```
csm: Basic.Consume 
get: Basic.Get
ack: Basck.Ack
```

csm + ack & QoS > 1|csm + no-ack|csm + ack|csm + transactions|get|
--- | --- |  --- | --- | ---
Fast | <- | - | -> | Slow

### 5.2.1 Using No-Ack Mode for Faster Throughput

Consumers can enable **no-ack** mode by setting **`no-ack`** flag in **`Basic.Consume`** method frame. When messages are not acknowledged at all, the throughput is maximized, because server doesn't wait at all to send the next message to consumer. 

### 5.2.2 Prefeching & Quality of Service (QoS)

QoS or Pre-fetching is essentially a setting by which consumers can specify *the number of messages to be received before consumer acknowledging the received messages*. These messages are acknowledged in batches (not using `Basic.Ack`), if the consumer crashes before it can acknowledges the messages, all the pre-fetched messages will return to the queue. This setting is configured through sending **`Basic.QoS`**. Based on it's characterisics, the value of QoS should be adjusted to a optimal level according to the architecture being used, the business need and so on.

### 5.2.3 Consuming Messages in Transactions

Transactions don't work for consumers with acknowledgements disabled. Just like message publishing with transaction enabled, message consuming are committed or rollback. It has negative performance impact, when transaction is used with QoS (value greater than 1), because transactions are only comitted when all pre-fetched messages are committed.

## 5.3 Rejecting Messages

Consumer can reject messages using **`Basic.Reject`** or **`Basic.Nack`**, both of them tell RabbitMQ that messages are not properly processed. `Basic.Reject` only works for single message, but `Basic.Nack` works for multiple messages. `Basic.Reject` is an AMQP command, and `Basic.Nack` is a RabbitMQ-specific command. Similarly, they both work by carring the **delivery tag** (from `Basic.Deliver`) to the server. 

### 5.3.1 Redelivered Attribute

If a message is rejected/'nacked', and it's mandatory, broker will try to redeliver it, we can check **`redelivered`** flag for this. For some cases, we may want to drop the message if it fails after a re-delivery.

### 5.3.2 Dead Letter Exchanges

**Dead letter exchange** or **DLX** is an extension to AMQP. A DLX has nothing different from normal exchange, it's *used to store messages that are expired or rejected*, thus it's helpful when trying to diagnose problems. This is not the same as the alternate exchange, tho they are quite similar, alternate exchanges are used for unroutable messages. To specify a dead letter exchange for a normal exchange, we use argument **`x-dead-letter-exchange`** in **`Queue.Declare`**, we can also override the routing key by setting argument **`x-dead-letter-routing-key`**.

## 5.4 Controlling Queues

### 5.4.1 Temporary Queue

- Automatically deleting queues
    - queues that will delete themselves when no more consumers are listening to it 
    - by setting **`auto_delete`** flag to true in `Queue.Declare`
- Allowing only a single consumer
    - queues that only allow a single consumer at any time
    - when no consumer is listening to it, it's deleted automatically
    - by setting **`exclusive`** flag to true in `Queue.Declare`
- Automatically expiring queues
    - queues that will delete themselves if it's unused for some time (exceeding specified TTL)
    - by setting **`x-expire`** with the queue's TTL in milliseconds

### 5.4.2 Permanent Queue

- Queue durability
    - queues that will persist after server restarts
    - by seting **`durable`** flag to true
- Auto-expiration of messages in queue
    - queues that expire messages if which are not consumed over a long time 
    - by setting **`x-message-ttl`** that enforces a maximum age for all messages in the queue 
- Maximum length queue
    - queues with maximum size, once it reaches the limit, it will drop messages from the front (earliest messages) 
    - by setting **`x-max-length`** that enforces a maximum length for the queue