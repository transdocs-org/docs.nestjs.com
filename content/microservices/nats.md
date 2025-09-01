### NATS

[NATS](https://nats.io) 是一个简单、安全且高性能的开源消息系统，适用于云原生应用、物联网消息传递和微服务架构。NATS 服务器使用 Go 编程语言编写，但与服务器交互的客户端库支持数十种主流编程语言。NATS 支持 **至多一次** 和 **至少一次** 的消息传递语义。它可以运行在大型服务器、云实例、边缘网关甚至物联网设备上。

#### 安装

要开始构建基于 NATS 的微服务，首先需要安装所需的包：

```bash
$ npm i --save nats
```

#### 概述

要使用 NATS 传输器，请将以下选项对象传递给 `createMicroservice()` 方法：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.NATS,
  options: {
    servers: ['nats://localhost:4222'],
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.NATS,
  options: {
    servers: ['nats://localhost:4222'],
  },
});
```

> info **提示** `Transport` 枚举是从 `@nestjs/microservices` 包中导入的。

#### 选项

`options` 对象是特定于所选传输器的。**NATS** 传输器公开了 [此处](https://github.com/nats-io/node-nats#connection-options) 描述的属性，以及以下属性：

<table>
  <tr>
    <td><code>queue</code></td>
    <td>你的服务器应订阅的队列（留空为 <code>undefined</code> 以忽略此设置）。关于 NATS 队列组的更多信息，请参见下方 <a href="#queue-groups">队列组</a> 部分。
    </td> 
  </tr>
  <tr>
    <td><code>gracefulShutdown</code></td>
    <td>启用优雅关闭。启用后，服务器首先会取消订阅所有通道，然后关闭连接。默认为 <code>false</code>。
  </td>
  </tr>
  <tr>
    <td><code>gracePeriod</code></td>
    <td>取消订阅所有通道后等待服务器的毫秒数。默认为 <code>10000</code> 毫秒。
  </td>
  </tr>
</table>

#### 客户端

与其它微服务传输器一样，你有 <a href="https://docs.nestjs.com/microservices/basics#client">多种方式</a> 可以创建一个 NATS `ClientProxy` 实例。

一种创建实例的方法是使用 `ClientsModule`。要使用 `ClientsModule` 创建客户端实例，请导入它并使用 `register()` 方法传递一个选项对象，该对象包含上面 `createMicroservice()` 方法中所示的相同属性，以及一个作为注入令牌使用的 `name` 属性。关于 `ClientsModule` 的更多内容，请参见 <a href="https://docs.nestjs.com/microservices/basics#client">此处</a>。

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.NATS,
        options: {
          servers: ['nats://localhost:4222'],
        }
      },
    ]),
  ]
  ...
})
```

其它创建客户端的方法（如 `ClientProxyFactory` 或 `@Client()`）也可以使用。你可以 <a href="https://docs.nestjs.com/microservices/basics#client">此处</a> 阅读相关内容。

#### 请求-响应

对于 **请求-响应** 消息样式（[了解更多](https://docs.nestjs.com/microservices/basics#request-response)），NATS 传输器不使用 NATS 内置的 [请求-回复](https://docs.nats.io/nats-concepts/reqreply) 机制。相反，"请求" 是使用 `publish()` 方法在特定主题上发布的，并附带唯一的回复主题名称。响应者监听该主题并发送响应到回复主题。无论通信双方位于何处，回复主题都会动态地返回给请求者。

#### 事件驱动

对于 **事件驱动** 消息样式（[了解更多](https://docs.nestjs.com/microservices/basics#event-based)），NATS 传输器使用 NATS 内置的 [发布-订阅](https://docs.nats.io/nats-concepts/pubsub) 机制。发布者在某个主题上发送消息，任何正在监听该主题的活动订阅者都会接收到该消息。订阅者还可以注册对通配符主题的兴趣，其行为类似于正则表达式。这种一对多的模式有时称为扇出（fan-out）。

#### 队列组

NATS 提供了一个名为 [分布式队列](https://docs.nats.io/nats-concepts/queue) 的内置负载均衡功能。要创建队列订阅，请使用 `queue` 属性，如下所示：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.NATS,
  options: {
    servers: ['nats://localhost:4222'],
    queue: 'cats_queue',
  },
});
```

#### 上下文

在更复杂的场景中，你可能需要访问有关传入请求的附加信息。当使用 NATS 传输器时，你可以访问 `NatsContext` 对象。

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: NatsContext) {
  console.log(`主题: ${context.getSubject()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(`主题: ${context.getSubject()}`);
}
```

> info **提示** `@Payload()`、`@Ctx()` 和 `NatsContext` 是从 `@nestjs/microservices` 包中导入的。

#### 通配符

订阅可以是明确的主题，也可以包含通配符。

```typescript
@@filename()
@MessagePattern('time.us.*')
getDate(@Payload() data: number[], @Ctx() context: NatsContext) {
  console.log(`主题: ${context.getSubject()}`); // 例如 "time.us.east"
  return new Date().toLocaleTimeString(...);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('time.us.*')
getDate(data, context) {
  console.log(`主题: ${context.getSubject()}`); // 例如 "time.us.east"
  return new Date().toLocaleTimeString(...);
}
```

#### 记录构建器

要配置消息选项，可以使用 `NatsRecordBuilder` 类（注意：此功能也可用于事件驱动流程）。例如，要添加 `x-version` 头，请使用 `setHeaders` 方法，如下所示：

```typescript
import * as nats from 'nats';

// 在你的代码某处
const headers = nats.headers();
headers.set('x-version', '1.0.0');

const record = new NatsRecordBuilder(':cat:').setHeaders(headers).build();
this.client.send('replace-emoji', record).subscribe(...);
```

> info **提示** `NatsRecordBuilder` 类是从 `@nestjs/microservices` 包中导出的。

你也可以在服务端读取这些头信息，通过访问 `NatsContext`，如下所示：

```typescript
@@filename()
@MessagePattern('replace-emoji')
replaceEmoji(@Payload() data: string, @Ctx() context: NatsContext): string {
  const headers = context.getHeaders();
  return headers['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('replace-emoji')
replaceEmoji(data, context) {
  const headers = context.getHeaders();
  return headers['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
```

在某些情况下，你可能希望为多个请求配置相同的头信息，你可以将这些头信息作为选项传递给 `ClientProxyFactory`：

```typescript
import { Module } from '@nestjs/common';
import { ClientProxyFactory, Transport } from '@nestjs/microservices';

@Module({
  providers: [
    {
      provide: 'API_v1',
      useFactory: () =>
        ClientProxyFactory.create({
          transport: Transport.NATS,
          options: {
            servers: ['nats://localhost:4222'],
            headers: { 'x-version': '1.0.0' },
          },
        }),
    },
  ],
})
export class ApiModule {}
```

#### 实例状态更新

要实时获取连接和底层驱动实例状态的更新信息，你可以订阅 `status` 流。此流提供与所选驱动器相关的状态更新。对于 NATS 驱动器，`status` 流会发出 `connected`、`disconnected` 和 `reconnecting` 事件。

```typescript
this.client.status.subscribe((status: NatsStatus) => {
  console.log(status);
});
```

> info **提示** `NatsStatus` 类型是从 `@nestjs/microservices` 包中导入的。

同样地，你也可以订阅服务器的 `status` 流以接收服务器状态的通知。

```typescript
const server = app.connectMicroservice<MicroserviceOptions>(...);
server.status.subscribe((status: NatsStatus) => {
  console.log(status);
});
```

#### 监听 NATS 事件

在某些情况下，你可能想要监听微服务发出的内部事件。例如，你可以监听 `error` 事件，在发生错误时触发额外操作。为此，可以使用 `on()` 方法，如下所示：

```typescript
this.client.on('error', (err) => {
  console.error(err);
});
```

同样地，你也可以监听服务器的内部事件：

```typescript
server.on<NatsEvents>('error', (err) => {
  console.error(err);
});
```

> info **提示** `NatsEvents` 类型是从 `@nestjs/microservices` 包中导入的。

#### 访ing底层驱动

对于更高级的使用场景，你可能需要访问底层驱动实例。这在需要手动关闭连接或使用特定驱动方法的场景中非常有用。但请注意，在大多数情况下，你 **不需要** 直接访问驱动。

为此，你可以使用 `unwrap()` 方法，它将返回底层驱动实例。泛型类型参数应指定你期望的驱动实例类型。

```typescript
const natsConnection = this.client.unwrap<import('nats').NatsConnection>();
```

同样地，你也可以访问服务器的底层驱动实例：

```typescript
const natsConnection = server.unwrap<import('nats').NatsConnection>();
```