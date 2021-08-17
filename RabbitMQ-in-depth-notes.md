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
3. Client responds with **`Connection.StartOk`**.


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
       | <---------------------  Basic.Cancel (method frame)
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

