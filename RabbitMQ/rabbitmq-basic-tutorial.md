# Basic RabbitMQ Tutorial with This Sample

This project is a small RabbitMQ example built with two .NET console apps:

- `SimpleTaskQueue.Producer` publishes messages.
- `SimpleTaskQueue.Consumer` receives and processes messages.

It demonstrates the core RabbitMQ ideas you usually start with:

1. a **producer** sends messages,
2. an **exchange** routes them,
3. a **queue** stores them,
4. a **consumer** processes them,
5. acknowledgements make delivery more reliable.

## What this sample does

The code uses:

- a **topic exchange** named `myExchange`
- a **routing key** of `my.routing.key`
- a **queue** named `myQueue`
- a RabbitMQ connection string of `amqp://admin:secret@localhost:5672`

The producer reads text from the console and publishes it to RabbitMQ. The consumer listens on `myQueue`, simulates one second of work, acknowledges the message, and then sends a small `"done"` reply back to the producer.

That last step is important: this is not only a queue example, it also shows a simple **request/reply acknowledgement pattern** on top of RabbitMQ.

## RabbitMQ concepts in plain English

### Producer

The producer is the app that sends a message.

In `SimpleTaskQueue.Producer\Program.cs`, the producer reads input and sends it:

```csharp
await producer.ProduceEvent(routingKey, input, 3, TimeSpan.FromSeconds(2));
```

### Exchange

An exchange decides how messages should be routed.

This sample declares a **topic exchange**:

```csharp
await channel.ExchangeDeclareAsync(exchange: exchangeName, type: ExchangeType.Topic, durable: true);
```

A topic exchange uses routing keys. Here the routing key is:

```csharp
var routingKey = "my.routing.key";
```

### Queue

A queue stores messages until a consumer is ready.

The consumer declares a durable queue:

```csharp
await channel.QueueDeclareAsync(queue: queueName, durable: true, exclusive: false, autoDelete: false);
```

Then it binds that queue to the exchange:

```csharp
await channel.QueueBindAsync(queue: queueName, exchange: exchangeName, routingKey: routingKey);
```

### Consumer

The consumer is the app that receives and handles messages.

In `SimpleTaskQueue.Consumer\EventConsumer.cs`, the consumer:

1. reads the message body,
2. simulates work with `Task.Delay(1000)`,
3. sends `BasicAck`,
4. optionally replies to the producer.

## How acknowledgements work here

This sample shows **two kinds of acknowledgement**.

### 1. Broker-level acknowledgement

After the consumer finishes processing, it tells RabbitMQ the message was handled:

```csharp
await channel.BasicAckAsync(ea.DeliveryTag, multiple: false);
```

If processing fails, it rejects and requeues the message:

```csharp
await channel.BasicNackAsync(ea.DeliveryTag, multiple: false, requeue: true);
```

This is standard RabbitMQ reliability behavior.

### 2. Application-level acknowledgement

The producer also creates a temporary reply queue and sends each message with:

- `CorrelationId`
- `ReplyTo`

That lets the consumer publish a `"done"` response back to the producer after work is complete.

So the producer does not just know that RabbitMQ accepted the message - it knows that a consumer finished the work.

## Why `prefetchCount = 1` matters

The consumer uses:

```csharp
await channel.BasicQosAsync(prefetchSize: 0, prefetchCount: 1, global: false);
```

This tells RabbitMQ to give each consumer one unacknowledged message at a time. That is the classic **fair dispatch** pattern for worker queues.

If you run multiple consumers, messages will be spread across them more evenly.

## How to run the sample

## 1. Start RabbitMQ

Make sure RabbitMQ is running locally and matches the credentials in the code:

```text
amqp://admin:secret@localhost:5672
```

## 2. Start a consumer

Open a terminal in the repo root and run:

```powershell
dotnet run --project .\SimpleTaskQueue.Consumer\SimpleTaskQueue.Consumer.csproj
```

You should see:

```text
[*] Waiting for messages. Press Ctrl+C to exit.
```

## 3. Start more consumers (optional)

To demonstrate worker distribution, run the helper script from the consumer project folder:

```powershell
.\start-consumers.ps1
```

That starts three consumer processes.

## 4. Start the producer

In another terminal, run:

```powershell
dotnet run --project .\SimpleTaskQueue.Producer\SimpleTaskQueue.Producer.csproj
```

You will be prompted for messages:

```text
Enter messages to enqueue. Empty line to exit.
```

Type a few lines and press Enter after each one.

## 5. Watch the flow

For each message:

1. the producer publishes to `myExchange`,
2. RabbitMQ routes it to `myQueue`,
3. one consumer receives it,
4. the consumer processes it and sends `BasicAck`,
5. the consumer publishes a `"done"` reply,
6. the producer prints that it received the consumer acknowledgement.

## What to learn from this sample

This code is a good introduction to RabbitMQ because it shows practical building blocks without much ceremony:

- **Exchange + routing key** for message routing
- **Durable queue** for work storage
- **Manual acknowledgement** for reliability
- **Nack + requeue** for failure handling
- **Prefetch** for fair dispatch
- **Reply queue + correlation ID** for application-level completion tracking

## A simple mental model

If you are new to RabbitMQ, think of this sample like this:

- the **producer** says, "please do this job"
- RabbitMQ holds and routes the job
- the **consumer** does the job
- `BasicAck` says, "RabbitMQ, I handled this delivery"
- the reply queue says, "Producer, the job is done"

That is the core idea behind many background processing systems.
