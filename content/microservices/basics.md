### 概述

除了传统的（有时称为单体式）应用架构之外，Nest 原生支持微服务架构风格的开发。本文档中讨论的大多数概念，例如依赖注入、装饰器、异常过滤器、管道、守卫和拦截器，同样适用于微服务。Nest 尽可能地对实现细节进行抽象，以使相同的组件可以在基于 HTTP 的平台、WebSockets 和微服务之间运行。本节介绍的是 Nest 中与微服务相关的一些特定内容。

在 Nest 中，微服务本质上是一种使用不同于 HTTP 的**传输**层的应用程序。

<figure><img class="illustrative-image" src="/assets/Microservices_1.png" /></figure>

Nest 支持几种内置的传输层实现，称为**传输器（transporters）**，它们负责在不同的微服务实例之间传输消息。大多数传输器原生支持**请求-响应**和**事件驱动**两种消息风格。Nest 通过统一的接口对每种传输器的请求-响应和事件驱动消息进行抽象，这使得在不同传输层之间切换变得简单——例如，可以利用某种特定传输层的可靠性或性能特性——而不会影响你的应用程序代码。

#### 安装

要开始构建微服务，首先安装所需的包：

```bash
$ npm i --save @nestjs/microservices
```

#### 快速开始

要实例化一个微服务，请使用 `NestFactory` 类的 `createMicroservice()` 方法：

```typescript
@@filename(main)
import { NestFactory } from '@nestjs/core';
import { Transport, MicroserviceOptions } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.TCP,
    },
  );
  await app.listen();
}
bootstrap();
@@switch
import { NestFactory } from '@nestjs/core';
import { Transport } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice(AppModule, {
    transport: Transport.TCP,
  });
  await app.listen();
}
bootstrap();
```

> info **提示** 微服务默认使用 **TCP** 传输层。

`createMicroservice()` 方法的第二个参数是一个 `options` 对象。该对象可能包含两个成员：

<table>
  <tr>
    <td><code>transport</code></td>
    <td>指定传输器（例如 <code>Transport.NATS</code>）</td>
  </tr>
  <tr>
    <td><code>options</code></td>
    <td>特定于传输器的行为配置选项对象</td>
  </tr>
</table>
<p>
  <code>options</code> 对象与所选传输器有关。例如 <strong>TCP</strong> 传输器暴露了如下属性。对于其他传输器（如 Redis、MQTT 等），请参考相应章节了解可用选项。
</p>
<table>
  <tr>
    <td><code>host</code></td>
    <td>连接主机名</td>
  </tr>
  <tr>
    <td><code>port</code></td>
    <td>连接端口</td>
  </tr>
  <tr>
    <td><code>retryAttempts</code></td>
    <td>消息重试次数（默认值： <code>0</code>）</td>
  </tr>
  <tr>
    <td><code>retryDelay</code></td>
    <td>消息重试之间的延迟（毫秒）（默认值： <code>0</code>）</td>
  </tr>
  <tr>
    <td><code>serializer</code></td>
    <td>用于传出消息的自定义 <a href="https://github.com/nestjs/nest/blob/master/packages/microservices/interfaces/serializer.interface.ts" target="_blank">序列化器</a></td>
  </tr>
  <tr>
    <td><code>deserializer</code></td>
    <td>用于传入消息的自定义 <a href="https://github.com/nestjs/nest/blob/master/packages/microservices/interfaces/deserializer.interface.ts" target="_blank">反序列化器</a></td>
  </tr>
  <tr>
    <td><code>socketClass</code></td>
    <td>扩展 <code>TcpSocket</code> 的自定义 Socket（默认值： <code>JsonSocket</code>）</td>
  </tr>
  <tr>
    <td><code>tlsOptions</code></td>
    <td>用于配置 tls 协议的选项</td>
  </tr>
</table>

> info **提示** 上述属性仅适用于 TCP 传输器。有关其他传输器的可用选项，请参阅相应章节。

#### 消息和事件模式

微服务通过**模式**识别消息和事件。模式是一个普通值，例如一个字面量对象或字符串。模式会自动序列化，并与消息的数据部分一起通过网络发送。这样，消息的发送方和消费者就可以协调哪些请求由哪些处理器处理。

#### 请求-响应

请求-响应的消息风格适用于需要在不同外部服务之间**交换**消息的场景。这种模式确保服务实际接收到了消息（无需手动实现确认协议）。然而，请求-响应方式并不总是最佳选择。例如，像 [Kafka](https://docs.confluent.io/3.0.0/streams/) 或 [NATS streaming](https://github.com/nats-io/node-nats-streaming) 这样使用日志持久化的流式传输器更适合处理事件消息范式的问题（有关更多细节，请参阅 [事件驱动消息](https://docs.nestjs.com/microservices/basics#event-based)）。

为了启用请求-响应消息类型，Nest 会创建两个逻辑通道：一个用于传输数据，另一个用于等待响应。对于某些底层传输器（如 [NATS](https://nats.io/)），这种双通道支持是开箱即用的。对于其他传输器，Nest 会手动创建独立通道。虽然这有效，但可能会带来一些开销。因此，如果你不需要请求-响应消息风格，可以考虑使用事件驱动方式。

要创建基于请求-响应范式的处理器，请使用 `@MessagePattern()` 装饰器，该装饰器从 `@nestjs/microservices` 包中导入。该装饰器只能在 [控制器](https://docs.nestjs.com/controllers) 类中使用，因为它们是应用程序的入口点。在提供者中使用它将无效，因为 Nest 运行时会忽略它们。

```typescript
@@filename(math.controller)
import { Controller } from '@nestjs/common';
import { MessagePattern } from '@nestjs/microservices';

@Controller()
export class MathController {
  @MessagePattern({ cmd: 'sum' })
  accumulate(data: number[]): number {
    return (data || []).reduce((a, b) => a + b);
  }
}
@@switch
import { Controller } from '@nestjs/common';
import { MessagePattern } from '@nestjs/microservices';

@Controller()
export class MathController {
  @MessagePattern({ cmd: 'sum' })
  accumulate(data) {
    return (data || []).reduce((a, b) => a + b);
  }
}
```

在上面的代码中，`accumulate()` **消息处理器** 监听匹配 `{{ '{' }} cmd: 'sum' {{ '}' }}` 消息模式的消息。消息处理器接收一个参数，即客户端传递的 `data`。在这种情况下，数据是需要累加的数字数组。

#### 异步响应

消息处理器可以同步或**异步**响应，这意味着支持 `async` 方法。

```typescript
@@filename()
@MessagePattern({ cmd: 'sum' })
async accumulate(data: number[]): Promise<number> {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@MessagePattern({ cmd: 'sum' })
async accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```

消息处理器也可以返回一个 `Observable`，在这种情况下，结果值将在流完成之前依次发出。

```typescript
@@filename()
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): Observable<number> {
  return from([1, 2, 3]);
}
@@switch
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): Observable<number> {
  return from([1, 2, 3]);
}
```

在上面的示例中，消息处理器将**三次**响应，每次对应数组中的一个元素。

#### 事件驱动

虽然请求-响应方法适用于服务之间的消息交换，但它不太适合事件驱动的消息传递——当你只是想发布**事件**而无需等待响应时。在这种情况下，维护两个通道用于请求-响应是不必要的。

例如，如果你想通知另一个服务系统中某部分发生了特定条件，事件驱动的消息风格是理想的选择。

要创建事件处理器，可以使用 `@EventPattern()` 装饰器，该装饰器从 `@nestjs/microservices` 包中导入。

```typescript
@@filename()
@EventPattern('user_created')
async handleUserCreated(data: Record<string, unknown>) {
  // 业务逻辑
}
@@switch
@EventPattern('user_created')
async handleUserCreated(data) {
  // 业务逻辑
}
```

> info **提示** 你可以为**单个**事件模式注册多个事件处理器，所有处理器都会自动并行触发。

`handleUserCreated()` **事件处理器** 监听 `'user_created'` 事件。事件处理器接收一个参数，即客户端传递的 `data`（在这种情况下，是通过网络发送的事件负载）。

#### 额外的请求细节

在更高级的场景中，你可能需要访问关于传入请求的额外信息。例如，当使用带有通配符订阅的 NATS 时，你可能希望获取生产者发送消息的目标主题。同样，使用 Kafka 时，你可能需要访问消息头。为此，可以使用如下所示的内置装饰器：

```typescript
@@filename()
@MessagePattern('time.us.*')
getDate(@Payload() data: number[], @Ctx() context: NatsContext) {
  console.log(`Subject: ${context.getSubject()}`); // 例如 "time.us.east"
  return new Date().toLocaleTimeString(...);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('time.us.*')
getDate(data, context) {
  console.log(`Subject: ${context.getSubject()}`); // 例如 "time.us.east"
  return new Date().toLocaleTimeString(...);
}
```

> info **提示** `@Payload()`、`@Ctx()` 和 `NatsContext` 从 `@nestjs/microservices` 导入。

> info **提示** 你也可以将属性键传递给 `@Payload()` 装饰器，以从传入的负载对象中提取特定属性，例如 `@Payload('id')`。

#### 客户端（生产者类）

客户端 Nest 应用程序可以使用 `ClientProxy` 类与 Nest 微服务交换消息或发布事件。此类提供了几种方法，例如 `send()`（用于请求-响应消息）和 `emit()`（用于事件驱动消息），使与远程微服务通信成为可能。你可以通过以下方式获取此类的实例：

一种方法是导入 `ClientsModule`，它暴露了静态的 `register()` 方法。此方法接收一个表示微服务传输器的对象数组。每个对象必须包含一个 `name` 属性，以及可选的 `transport` 属性（默认为 `Transport.TCP`）和可选的 `options` 属性。

`name` 属性作为**注入令牌**，你可以在任何需要的地方使用它来注入 `ClientProxy` 实例。该 `name` 属性的值可以是任意字符串或 JavaScript 符号，如 [此处](https://docs.nestjs.com/fundamentals/custom-providers#non-class-based-provider-tokens) 所述。

`options` 属性是一个对象，包含我们在前面的 `createMicroservice()` 方法中看到的相同属性。

```typescript
@Module({
  imports: [
    ClientsModule.register([
      { name: 'MATH_SERVICE', transport: Transport.TCP },
    ]),
  ],
})
```

如果你需要在设置过程中提供配置或执行任何其他异步操作，也可以使用 `registerAsync()` 方法。

```typescript
@Module({
  imports: [
    ClientsModule.registerAsync([
      {
        imports: [ConfigModule],
        name: 'MATH_SERVICE',
        useFactory: async (configService: ConfigService) => ({
          transport: Transport.TCP,
          options: {
            url: configService.get('URL'),
          },
        }),
        inject: [ConfigService],
      },
    ]),
  ],
})
```

一旦模块被导入，你就可以使用 `@Inject()` 装饰器注入一个为 `'MATH_SERVICE'` 传输器配置的 `ClientProxy` 实例。

```typescript
constructor(
  @Inject('MATH_SERVICE') private client: ClientProxy,
) {}
```

> info **提示** `ClientsModule` 和 `ClientProxy` 类从 `@nestjs/microservices` 包中导入。

有时，你可能需要从其他服务（例如 `ConfigService`）获取传输器配置，而不是硬编码在客户端应用程序中。为此，你可以使用 `ClientProxyFactory` 类注册一个 [自定义提供者](/fundamentals/custom-providers)。此类提供了一个静态 `create()` 方法，接受一个传输器选项对象并返回一个定制的 `ClientProxy` 实例。

```typescript
@Module({
  providers: [
    {
      provide: 'MATH_SERVICE',
      useFactory: (configService: ConfigService) => {
        const mathSvcOptions = configService.getMathSvcOptions();
        return ClientProxyFactory.create(mathSvcOptions);
      },
      inject: [ConfigService],
    }
  ]
  ...
})
```

> info **提示** `ClientProxyFactory` 从 `@nestjs/microservices` 包中导入。

另一种方法是使用 `@Client()` 属性装饰器。

```typescript
@Client({ transport: Transport.TCP })
client: ClientProxy;
```

> info **提示** `@Client()` 装饰器从 `@nestjs/microservices` 包中导入。

使用 `@Client()` 装饰器并不是首选技术，因为它难以测试且难以共享客户端实例。

`ClientProxy` 是**惰性的**。它不会立即建立连接。相反，它会在第一次微服务调用之前建立连接，并在后续调用中重复使用。不过，如果你希望延迟应用程序的引导过程，直到建立连接，可以在 `OnApplicationBootstrap` 生命周期钩子中使用 `ClientProxy` 对象的 `connect()` 方法手动建立连接。

```typescript
@@filename()
async onApplicationBootstrap() {
  await this.client.connect();
}
```

如果无法创建连接，`connect()` 方法将拒绝并返回相应的错误对象。

#### 发送消息

`ClientProxy` 暴露了一个 `send()` 方法。该方法用于调用微服务并返回一个包含其响应的 `Observable`。因此，我们可以轻松订阅发出的值。

```typescript
@@filename()
accumulate(): Observable<number> {
  const pattern = { cmd: 'sum' };
  const payload = [1, 2, 3];
  return this.client.send<number>(pattern, payload);
}
@@switch
accumulate() {
  const pattern = { cmd: 'sum' };
  const payload = [1, 2, 3];
  return this.client.send(pattern, payload);
}
```

`send()` 方法接收两个参数：`pattern` 和 `payload`。`pattern` 应与 `@MessagePattern()` 装饰器中定义的一个匹配。`payload` 是我们要传输到远程微服务的消息。此方法返回一个**冷 `Observable`**，这意味着你必须显式订阅它，消息才会被发送。

#### 发布事件

要发送事件，请使用 `ClientProxy` 对象的 `emit()` 方法。此方法将事件发布到消息代理。

```typescript
@@filename()
async publish() {
  this.client.emit<number>('user_created', new UserCreatedEvent());
}
@@switch
async publish() {
  this.client.emit('user_created', new UserCreatedEvent());
}
```

`emit()` 方法接收两个参数：`pattern` 和 `payload`。`pattern` 应与 `@EventPattern()` 装饰器中定义的一个匹配。`payload` 表示你要传输到远程微服务的事件数据。此方法返回一个**热 `Observable`**（与 `send()` 返回的冷 `Observable` 相比），这意味着无论你是否显式订阅该 observable，代理都会立即尝试发送事件。

#### 请求作用域

对于来自不同编程语言背景的人来说，可能会惊讶地发现，在 Nest 中，大多数内容在传入请求之间是共享的。这包括数据库连接池、具有全局状态的单例服务等。请记住，Node.js 并不遵循请求/响应多线程无状态模型，每个请求由单独的线程处理。因此，使用单例实例对我们的应用程序来说是**安全的**。

然而，也有一些边缘情况，你可能希望处理器具有基于请求的生命周期。这可能包括 GraphQL 应用程序中的每个请求缓存、请求跟踪或多租户等场景。你可以通过 [这里](/fundamentals/injection-scopes) 了解如何控制作用域的更多信息。

可以使用 `@Inject()` 装饰器结合 `CONTEXT` 令牌注入 `RequestContext` 来创建基于请求的作用域处理器和提供者：

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { CONTEXT, RequestContext } from '@nestjs/microservices';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(CONTEXT) private ctx: RequestContext) {}
}
```

这提供了对 `RequestContext` 对象的访问，该对象有两个属性：

```typescript
export interface RequestContext<T = any> {
  pattern: string | Record<string, any>;
  data: T;
}
```

`data` 属性是消息生产者发送的消息负载。`pattern` 属性是用于识别适当处理器以处理传入消息的模式。

#### 实例状态更新

要实时获取连接和底层驱动程序实例的状态更新，你可以订阅 `status` 流。此流提供特定于所选驱动程序的状态更新。例如，如果你使用 TCP 传输器（默认），`status` 流会发出 `connected` 和 `disconnected` 事件。

```typescript
this.client.status.subscribe((status: TcpStatus) => {
  console.log(status);
});
```

> info **提示** `TcpStatus` 类型从 `@nestjs/microservices` 包中导入。

同样，你可以订阅服务器的 `status` 流以接收服务器状态的通知。

```typescript
const server = app.connectMicroservice<MicroserviceOptions>(...);
server.status.subscribe((status: TcpStatus) => {
  console.log(status);
});
```

#### 监听内部事件

在某些情况下，你可能想要监听微服务发出的内部事件。例如，你可以在发生错误时监听 `error` 事件以触发额外的操作。为此，可以使用 `on()` 方法，如下所示：

```typescript
this.client.on('error', (err) => {
  console.error(err);
});
```

同样，你可以监听服务器的内部事件：

```typescript
server.on<TcpEvents>('error', (err) => {
  console.error(err);
});
```

> info **提示** `TcpEvents` 类型从 `@nestjs/microservices` 包中导入。

#### 底层驱动器访问

对于更高级的用例，你可能需要访问底层驱动程序实例。这在手动关闭连接或使用驱动程序特定方法的场景中非常有用。但请注意，大多数情况下，你**不需要**直接访问驱动程序。

为此，可以使用 `unwrap()` 方法，该方法返回底层驱动程序实例。泛型类型参数应指定你期望的驱动程序实例类型。

```typescript
const netServer = this.client.unwrap<Server>();
```

这里，`Server` 是从 `net` 模块导入的类型。

同样，你可以访问服务器的底层驱动程序实例：

```typescript
const netServer = server.unwrap<Server>();
```

#### 处理超时

在分布式系统中，微服务有时可能宕机或不可用。为了防止无限期等待，你可以使用超时。在与其他服务通信时，超时是一个非常有用的功能。要将超时应用于你的微服务调用，可以使用 [RxJS](https://rxjs.dev) 的 `timeout` 操作符。如果微服务在指定时间内没有响应，则会抛出异常，你可以捕获并适当处理。

为此，你需要使用 [`rxjs`](https://github.com/ReactiveX/rxjs) 包。只需在管道中使用 `timeout` 操作符：

```typescript
@@filename()
this.client
  .send<TResult, TInput>(pattern, data)
  .pipe(timeout(5000));
@@switch
this.client
  .send(pattern, data)
  .pipe(timeout(5000));
```

> info **提示** `timeout` 操作符从 `rxjs/operators` 包中导入。

5 秒后，如果微服务没有响应，它将抛出错误。

#### TLS 支持

在非私有网络中通信时，加密流量以确保安全性非常重要。在 NestJS 中，这可以通过 Node 内置的 [TLS](https://nodejs.org/api/tls.html) 模块在 TCP 上实现。Nest 在其 TCP 传输中提供了对 TLS 的内置支持，使我们可以在微服务或客户端之间加密通信。

要为 TCP 服务器启用 TLS，你需要 PEM 格式的私钥和证书。这些文件通过设置 `tlsOptions` 并指定 key 和 cert 文件添加到服务器选项中，如下所示：

```typescript
import * as fs from 'fs';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';

async function bootstrap() {
  const key = fs.readFileSync('<pathToKeyFile>', 'utf8').toString();
  const cert = fs.readFileSync('<pathToCertFile>', 'utf8').toString();

  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.TCP,
      options: {
        tlsOptions: {
          key,
          cert,
        },
      },
    },
  );

  await app.listen();
}
bootstrap();
```

对于客户端通过 TLS 安全通信，我们也定义 `tlsOptions` 对象，但这次使用 CA 证书。这是签署服务器证书的权威机构的证书。这确保了客户端信任服务器的证书，并可以建立安全连接。

```typescript
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.TCP,
        options: {
          tlsOptions: {
            ca: [fs.readFileSync('<pathToCaFile>', 'utf-8').toString()],
          },
        },
      },
    ]),
  ],
})
export class AppModule {}
```

如果设置涉及多个受信任的权威机构，你也可以传递一个 CA 数组。

一切设置完成后，你可以像往常一样使用 `@Inject()` 装饰器注入 `ClientProxy`，以便在你的服务中使用客户端。这确保了 Node 的 `TLS` 模块处理加密细节，使你的 NestJS 微服务之间进行加密通信。

有关更多信息，请参阅 Node 的 [TLS 文档](https://nodejs.org/api/tls.html)。

#### 动态配置

当微服务需要使用 `ConfigService`（来自 `@nestjs/config` 包）进行配置，但注入上下文仅在微服务实例创建后才可用时，`AsyncMicroserviceOptions` 提供了解决方案。这种方法允许动态配置，确保与 `ConfigService` 的无缝集成。

```typescript
import { ConfigService } from '@nestjs/config';
import { AsyncMicroserviceOptions, Transport } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<AsyncMicroserviceOptions>(
    AppModule,
    {
      useFactory: (configService: ConfigService) => ({
        transport: Transport.TCP,
        options: {
          host: configService.get<string>('HOST'),
          port: configService.get<number>('PORT'),
        },
      }),
      inject: [ConfigService],
    },
  );

  await app.listen();
}
bootstrap();
```