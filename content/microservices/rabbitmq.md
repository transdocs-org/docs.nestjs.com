### RabbitMQ

[RabbitMQ](https://www.rabbitmq.com/) 是一个开源且轻量级的消息代理，支持多种消息协议。它可以以分布式和联邦配置进行部署，以满足高规模和高可用性的需求。此外，它是目前使用最广泛的消息代理，被全球范围内的小型初创公司和大型企业所采用。

#### 安装

要开始构建基于 RabbitMQ 的微服务，首先安装所需的包：

```bash
$ npm i --save amqplib amqp-connection-manager
```

#### 概述

要使用 RabbitMQ 传输器，请将以下选项对象传递给 `createMicroservice()` 方法：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.RMQ,
  options: {
    urls: ['amqp://localhost:5672'],
    queue: 'cats_queue',
    queueOptions: {
      durable: false
    },
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.RMQ,
  options: {
    urls: ['amqp://localhost:5672'],
    queue: 'cats_queue',
    queueOptions: {
      durable: false
    },
  },
});
```

> info **提示** `Transport` 枚举是从 `@nestjs/microservices` 包中导入的。

#### 选项

`options` 属性特定于所选的传输器。**RabbitMQ** 传输器暴露了如下所述的属性。

<table>
  <tr>
    <td><code>urls</code></td>
    <td>尝试连接的 URL 数组，按顺序尝试</td>
  </tr>
  <tr>
    <td><code>queue</code></td>
    <td>服务器将监听的队列名称</td>
  </tr>
  <tr>
    <td><code>prefetchCount</code></td>
    <td>设置通道的预取数量</td>
  </tr>
  <tr>
    <td><code>isGlobalPrefetchCount</code></td>
    <td>启用每个通道的预取</td>
  </tr>
  <tr>
    <td><code>noAck</code></td>
    <td>如果为 <code>false</code>，则启用手动确认模式</td>
  </tr>
  <tr>
    <td><code>consumerTag</code></td>
    <td>服务器用于区分消费者消息传递的名称；在通道上不能重复使用；通常可以省略此参数，此时服务器将创建一个随机名称并在响应中提供（详见 <a href="https://amqp-node.github.io/amqplib/channel_api.html#channel_consume" rel="nofollow" target="_blank">此处</a>）</td>
  </tr>
  <tr>
    <td><code>queueOptions</code></td>
    <td>附加队列选项（详见 <a href="https://amqp-node.github.io/amqplib/channel_api.html#channel_assertQueue" rel="nofollow" target="_blank">此处</a>）</td>
  </tr>
  <tr>
    <td><code>socketOptions</code></td>
    <td>附加的 socket 选项（详见 <a href="https://amqp-node.github.io/amqplib/channel_api.html#connect" rel="nofollow" target="_blank">此处</a>）</td>
  </tr>
  <tr>
    <td><code>headers</code></td>
    <td>每条消息都要发送的头部信息</td>
  </tr>
  <tr>
    <td><code>replyQueue</code></td>
    <td>生产者的回复队列。默认值为 <code>amq.rabbitmq.reply-to</code></td>
  </tr>
  <tr>
    <td><code>persistent</code></td>
    <td>如果为真值，则只要队列在重启后也存在，消息就能在代理重启后保留</td>
  </tr>
  <tr>
    <td><code>noAssert</code></td>
    <td>如果为 false，在消费前不会断言队列是否存在</td>
  </tr>
  <tr>
    <td><code>wildcards</code></td>
    <td>仅当您希望使用主题交换器来路由消息到队列时才设置为 true。启用此选项后，您可以使用通配符（*, #）作为消息和事件模式</td>
  </tr>
  <tr>
    <td><code>exchange</code></td>
    <td>交换器的名称。当 "wildcards" 设置为 true 时，默认值为队列名称</td>
  </tr>
  <tr>
    <td><code>exchangeType</code></td>
    <td>交换器类型。默认值为 <code>topic</code>。有效值为 <code>direct</code>、<code>fanout</code>、<code>topic</code> 和 <code>headers</code></td>
  </tr>
  <tr>
    <td><code>routingKey</code></td>
    <td>主题交换器的附加路由键</td>
  </tr>
  <tr>
    <td><code>maxConnectionAttempts</code></td>
    <td>最大连接尝试次数。仅适用于消费者配置。-1 表示无限次</td>
  </tr>
</table>

#### 客户端

与其他微服务传输器一样，您有 <a href="https://docs.nestjs.com/microservices/basics#client">多种选项</a> 可用于创建 RabbitMQ 的 `ClientProxy` 实例。

一种创建实例的方法是使用 `ClientsModule`。要使用 `ClientsModule` 创建客户端实例，请导入它并使用 `register()` 方法传递一个选项对象，该对象具有与上面 `createMicroservice()` 方法中相同的属性，以及一个用作注入令牌的 `name` 属性。有关 `ClientsModule` 的更多内容，请参见 <a href="https://docs.nestjs.com/microservices/basics#client">此处</a>。

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.RMQ,
        options: {
          urls: ['amqp://localhost:5672'],
          queue: 'cats_queue',
          queueOptions: {
            durable: false
          },
        },
      },
    ]),
  ]
  ...
})
```

也可以使用其他客户端创建方式（如 `ClientProxyFactory` 或 `@Client()`）。您可以在 <a href="https://docs.nestjs.com/microservices/basics#client">此处</a> 阅读相关内容。

#### 上下文

在更复杂的场景中，您可能需要访问有关传入请求的额外信息。当使用 RabbitMQ 传输器时，您可以访问 `RmqContext` 对象。

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  console.log(`Pattern: ${context.getPattern()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(`Pattern: ${context.getPattern()}`);
}
```

> info **提示** `@Payload()`、`@Ctx()` 和 `RmqContext` 是从 `@nestjs/microservices` 包中导入的。

要访问原始的 RabbitMQ 消息（包含 `properties`、`fields` 和 `content`），请使用 `RmqContext` 对象的 `getMessage()` 方法，如下所示：

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  console.log(context.getMessage());
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(context.getMessage());
}
```

要获取对 RabbitMQ [通道](https://www.rabbitmq.com/channels.html) 的引用，请使用 `RmqContext` 对象的 `getChannelRef` 方法，如下所示：

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  console.log(context.getChannelRef());
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(context.getChannelRef());
}
```

#### 消息确认

为确保消息不会丢失，RabbitMQ 支持[消息确认](https://www.rabbitmq.com/confirms.html)。消费者会发送一个确认回执，告诉 RabbitMQ 某条特定消息已被接收、处理，并且 RabbitMQ 可以安全地将其删除。如果消费者在未发送确认的情况下死亡（其通道关闭、连接关闭或 TCP 连接中断），RabbitMQ 将理解为消息未完全处理，并将其重新入队。

要启用手动确认模式，请将 `noAck` 属性设置为 `false`：

```typescript
options: {
  urls: ['amqp://localhost:5672'],
  queue: 'cats_queue',
  noAck: false,
  queueOptions: {
    durable: false
  },
},
```

当启用手动消费者确认后，我们必须从工作者发送适当的确认，以表明任务已完成。

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  const channel = context.getChannelRef();
  const originalMsg = context.getMessage();

  channel.ack(originalMsg);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  const channel = context.getChannelRef();
  const originalMsg = context.getMessage();

  channel.ack(originalMsg);
}
```

#### 消息构建器

要配置消息选项，可以使用 `RmqRecordBuilder` 类（注意：事件驱动的流程也适用）。例如，要设置 `headers` 和 `priority` 属性，请使用 `setOptions` 方法，如下所示：

```typescript
const message = ':cat:';
const record = new RmqRecordBuilder(message)
  .setOptions({
    headers: {
      ['x-version']: '1.0.0',
    },
    priority: 3,
  })
  .build();

this.client.send('replace-emoji', record).subscribe(...);
```

> info **提示** `RmqRecordBuilder` 类是从 `@nestjs/microservices` 包中导出的。

您还可以在服务器端读取这些值，方法是访问 `RmqContext`，如下所示：

```typescript
@@filename()
@MessagePattern('replace-emoji')
replaceEmoji(@Payload() data: string, @Ctx() context: RmqContext): string {
  const { properties: { headers } } = context.getMessage();
  return headers['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('replace-emoji')
replaceEmoji(data, context) {
  const { properties: { headers } } = context.getMessage();
  return headers['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
```

#### 实例状态更新

要实时获取连接和底层驱动实例状态的更新，您可以订阅 `status` 流。此流提供特定于所选驱动的状态更新。对于 RMQ 驱动，`status` 流会发出 `connected` 和 `disconnected` 事件。

```typescript
this.client.status.subscribe((status: RmqStatus) => {
  console.log(status);
});
```

> info **提示** `RmqStatus` 类型是从 `@nestjs/microservices` 包中导入的。

同样，您可以订阅服务器的 `status` 流以接收有关服务器状态的通知。

```typescript
const server = app.connectMicroservice<MicroserviceOptions>(...);
server.status.subscribe((status: RmqStatus) => {
  console.log(status);
});
```

#### 监听 RabbitMQ 事件

在某些情况下，您可能希望监听微服务发出的内部事件。例如，您可以监听 `error` 事件，在发生错误时触发额外的操作。为此，请使用 `on()` 方法，如下所示：

```typescript
this.client.on('error', (err) => {
  console.error(err);
});
```

同样，您可以监听服务器的内部事件：

```typescript
server.on<RmqEvents>('error', (err) => {
  console.error(err);
});
```

> info **提示** `RmqEvents` 类型是从 `@nestjs/microservices` 包中导入的。

#### 访问底层驱动

对于更高级的用例，您可能需要访问底层驱动实例。这在需要手动关闭连接或使用驱动特定方法的场景中非常有用。但请注意，对于大多数情况，您**不需要**直接访问驱动。

为此，您可以使用 `unwrap()` 方法，该方法返回底层驱动实例。泛型类型参数应指定您期望的驱动实例类型。

```typescript
const managerRef =
  this.client.unwrap<import('amqp-connection-manager').AmqpConnectionManager>();
```

同样，您可以访问服务器的底层驱动实例：

```typescript
const managerRef =
  server.unwrap<import('amqp-connection-manager').AmqpConnectionManager>();
```

#### 通配符

RabbitMQ 支持在路由键中使用通配符，以实现灵活的消息路由。`#` 通配符匹配零个或多个词，而 `*` 通配符匹配恰好一个词。

例如，路由键 `cats.#` 匹配 `cats`、`cats.meow` 和 `cats.meow.purr`。路由键 `cats.*` 匹配 `cats.meow`，但不匹配 `cats.meow.purr`。

要在 RabbitMQ 微服务中启用通配符支持，请将选项对象中的 `wildcards` 配置选项设置为 `true`：

```typescript
const app = await NestFactory.createMicroservice<MicroserviceOptions>(
  AppModule,
  {
    transport: Transport.RMQ,
    options: {
      urls: ['amqp://localhost:5672'],
      queue: 'cats_queue',
      wildcards: true,
    },
  },
);
```

配置完成后，您可以在订阅事件/消息时使用路由键中的通配符。例如，要监听路由键为 `cats.#` 的消息，请使用以下代码：

```typescript
@MessagePattern('cats.#')
getCats(@Payload() data: { message: string }, @Ctx() context: RmqContext) {
  console.log(`Received message with routing key: ${context.getPattern()}`);

  return {
    message: 'Hello from the cats service!',
  }
}
```

要发送具有特定路由键的消息，您可以使用 `ClientProxy` 实例的 `send()` 方法：

```typescript
this.client.send('cats.meow', { message: 'Meow!' }).subscribe((response) => {
  console.log(response);
});
```