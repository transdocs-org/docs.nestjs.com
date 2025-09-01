### Redis

[Redis](https://redis.io/) 传输器实现了发布/订阅消息传递模型，并利用了 Redis 的 [Pub/Sub](https://redis.io/topics/pubsub) 功能。发布的消息被分类在频道中，而无需知道哪些订阅者（如果有的话）最终会接收到该消息。每个微服务可以订阅任意数量的频道，同时也可以一次订阅多个频道。通过频道交换的消息是**即发即忘（fire-and-forget）**的，这意味着如果一条消息被发布时没有订阅者感兴趣，该消息将被删除且无法恢复。因此，你无法保证消息或事件至少会被一个服务处理。一条消息可以被多个订阅者订阅（并接收）。

<figure><img class="illustrative-image" src="/assets/Redis_1.png" /></figure>

#### 安装

要开始构建基于 Redis 的微服务，首先安装所需的包：

```bash
$ npm i --save ioredis
```

#### 概述

要使用 Redis 传输器，请将以下选项对象传递给 `createMicroservice()` 方法：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.REDIS,
  options: {
    host: 'localhost',
    port: 6379,
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.REDIS,
  options: {
    host: 'localhost',
    port: 6379,
  },
});
```

> info **提示** `Transport` 枚举是从 `@nestjs/microservices` 包中导入的。

#### 选项

`options` 属性特定于所选的传输器。<strong>Redis</strong> 传输器公开了以下描述的属性。

<table>
  <tr>
    <td><code>host</code></td>
    <td>连接地址</td>
  </tr>
  <tr>
    <td><code>port</code></td>
    <td>连接端口</td>
  </tr>
  <tr>
    <td><code>retryAttempts</code></td>
    <td>消息重试次数（默认值：<code>0</code>）</td>
  </tr>
  <tr>
    <td><code>retryDelay</code></td>
    <td>消息重试之间的延迟（毫秒）（默认值：<code>0</code>）</td>
  </tr>
  <tr>
    <td><code>wildcards</code></td>
    <td>启用 Redis 通配符订阅，指示传输器在底层使用 <code>psubscribe</code>/<code>pmessage</code>。（默认值：<code>false</code>）</td>
  </tr>
</table>

官方 [ioredis](https://redis.github.io/ioredis/index.html#RedisOptions) 客户端支持的所有属性，此传输器也都支持。

#### 客户端

与其他微服务传输器一样，你有 <a href="https://docs.nestjs.com/microservices/basics#client">多种选项</a> 来创建 `ClientProxy` 实例。

一种创建实例的方法是使用 `ClientsModule`。要使用 `ClientsModule` 创建客户端实例，请导入它并使用 `register()` 方法传递一个选项对象，该对象包含与上面 `createMicroservice()` 方法中显示的相同属性，以及一个用作注入令牌的 `name` 属性。更多关于 `ClientsModule` 的内容请 <a href="https://docs.nestjs.com/microservices/basics#client">点击此处</a> 阅读。

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.REDIS,
        options: {
          host: 'localhost',
          port: 6379,
        }
      },
    ]),
  ]
  ...
})
```

也可以使用其他创建客户端的方法（例如 `ClientProxyFactory` 或 `@Client()`）。你可以 <a href="https://docs.nestjs.com/microservices/basics#client">点击此处</a> 了解更多。

#### 上下文

在更复杂的场景中，你可能需要访问有关传入请求的额外信息。当使用 Redis 传输器时，你可以访问 `RedisContext` 对象。

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RedisContext) {
  console.log(`频道: ${context.getChannel()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(`频道: ${context.getChannel()}`);
}
```

> info **提示** `@Payload()`、`@Ctx()` 和 `RedisContext` 是从 `@nestjs/microservices` 包中导入的。

#### 通配符

要启用通配符支持，请将 `wildcards` 选项设置为 `true`。这会指示传输器在底层使用 `psubscribe` 和 `pmessage`。

```typescript
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.REDIS,
  options: {
    // 其他选项
    wildcards: true,
  },
});
```

创建客户端实例时也请确保传递 `wildcards` 选项。

启用此选项后，你可以在消息和事件模式中使用通配符。例如，要订阅所有以 `notifications` 开头的频道，可以使用以下模式：

```typescript
@EventPattern('notifications.*')
```

#### 实例状态更新

要实时获取底层驱动程序实例的连接和状态更新，可以订阅 `status` 流。此流提供特定于所选驱动程序的状态更新。对于 Redis 驱动程序，`status` 流会发出 `connected`、`disconnected` 和 `reconnecting` 事件。

```typescript
this.client.status.subscribe((status: RedisStatus) => {
  console.log(status);
});
```

> info **提示** `RedisStatus` 类型是从 `@nestjs/microservices` 包中导入的。

同样，你可以订阅服务器的 `status` 流以接收服务器状态的通知。

```typescript
const server = app.connectMicroservice<MicroserviceOptions>(...);
server.status.subscribe((status: RedisStatus) => {
  console.log(status);
});
```

#### 监听 Redis 事件

在某些情况下，你可能希望监听微服务发出的内部事件。例如，你可以监听 `error` 事件，在发生错误时触发额外操作。为此，请使用 `on()` 方法，如下所示：

```typescript
this.client.on('error', (err) => {
  console.error(err);
});
```

同样，你可以监听服务器的内部事件：

```typescript
server.on<RedisEvents>('error', (err) => {
  console.error(err);
});
```

> info **提示** `RedisEvents` 类型是从 `@nestjs/microservices` 包中导入的。

#### 访 访问底层驱动程序

对于更高级的用例，你可能需要访问底层驱动程序实例。这在手动关闭连接或使用特定于驱动程序的方法等场景中非常有用。但请注意，对于大多数情况，你**不需要**直接访问驱动程序。

为此，你可以使用 `unwrap()` 方法，该方法返回底层驱动程序实例。泛型类型参数应指定你期望的驱动程序实例类型。

```typescript
const [pub, sub] =
  this.client.unwrap<[import('ioredis').Redis, import('ioredis').Redis]>();
```

同样，你可以访问服务器的底层驱动程序实例：

```typescript
const [pub, sub] =
  server.unwrap<[import('ioredis').Redis, import('ioredis').Redis]>();
```

请注意，与其他传输器不同，Redis 传输器返回的是两个 `ioredis` 实例的元组：第一个用于发布消息，第二个用于订阅消息。