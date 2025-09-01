### Kafka

[Kafka](https://kafka.apache.org/) 是一个开源的分布式流处理平台，具有以下三个核心能力：

- 发布和订阅记录流，类似于消息队列或企业级消息系统。
- 以容错且持久化的方式存储记录流。
- 在记录流发生时进行处理。

Kafka 项目旨在提供一个统一的、高吞吐量、低延迟的平台来处理实时数据流。它与 Apache Storm 和 Spark 集成良好，用于实时流数据分析。

#### 安装

要开始构建基于 Kafka 的微服务，请首先安装所需的包：

```bash
$ npm i --save kafkajs
```

#### 概述

与其他 Nest 微服务传输层实现一样，你可以通过向 `createMicroservice()` 方法传递的选项对象中的 `transport` 属性选择 Kafka 传输机制，并结合一个可选的 `options` 属性，如下所示：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      brokers: ['localhost:9092'],
    }
  }
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      brokers: ['localhost:9092'],
    }
  }
});
```

> info **提示** `Transport` 枚举是从 `@nestjs/microservices` 包中导入的。

#### 选项

`options` 属性是特定于所选传输机制的。**Kafka** 传输机制暴露了如下所述的属性。

<table>
  <tr>
    <td><code>client</code></td>
    <td>客户端配置选项（更多详情请 <a href="https://kafka.js.org/docs/configuration" rel="nofollow" target="blank">点击此处</a>）</td>
  </tr>
  <tr>
    <td><code>consumer</code></td>
    <td>消费者配置选项（更多详情请 <a href="https://kafka.js.org/docs/consuming#a-name-options-a-options" rel="nofollow" target="blank">点击此处</a>）</td>
  </tr>
  <tr>
    <td><code>run</code></td>
    <td>运行配置选项（更多详情请 <a href="https://kafka.js.org/docs/consuming" rel="nofollow" target="blank">点击此处</a>）</td>
  </tr>
  <tr>
    <td><code>subscribe</code></td>
    <td>订阅配置选项（更多详情请 <a href="https://kafka.js.org/docs/consuming#frombeginning" rel="nofollow" target="blank">点击此处</a>）</td>
  </tr>
  <tr>
    <td><code>producer</code></td>
    <td>生产者配置选项（更多详情请 <a href="https://kafka.js.org/docs/producing#options" rel="nofollow" target="blank">点击此处</a>）</td>
  </tr>
  <tr>
    <td><code>send</code></td>
    <td>发送配置选项（更多详情请 <a href="https://kafka.js.org/docs/producing#options" rel="nofollow" target="blank">点击此处</a>）</td>
  </tr>
  <tr>
    <td><code>producerOnlyMode</code></td>
    <td>功能标志，用于跳过消费者组注册并仅作为生产者运行（<code>boolean</code>）</td>
  </tr>
  <tr>
    <td><code>postfixId</code></td>
    <td>更改 clientId 值的后缀（<code>string</code>）</td>
  </tr>
</table>

#### 客户端

Kafka 与其他微服务传输机制有一个小差异：我们使用 `ClientKafkaProxy` 类而不是 `ClientProxy` 类。

与其他微服务传输机制一样，你有 <a href="https://docs.nestjs.com/microservices/basics#client">多种方式</a> 创建 `ClientKafkaProxy` 实例。

一种创建实例的方法是使用 `ClientsModule`。要使用 `ClientsModule` 创建客户端实例，请导入它并使用 `register()` 方法传递一个包含与上述 `createMicroservice()` 方法中相同的属性的选项对象，以及一个用作注入令牌的 `name` 属性。更多关于 `ClientsModule` 的信息请 <a href="https://docs.nestjs.com/microservices/basics#client">点击此处</a>。

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'HERO_SERVICE',
        transport: Transport.KAFKA,
        options: {
          client: {
            clientId: 'hero',
            brokers: ['localhost:9092'],
          },
          consumer: {
            groupId: 'hero-consumer'
          }
        }
      },
    ]),
  ]
  ...
})
```

你也可以使用其他创建客户端的方法（如 `ClientProxyFactory` 或 `@Client()`）。你可以在 <a href="https://docs.nestjs.com/microservices/basics#client">此处</a> 阅读相关内容。

使用 `@Client()` 装饰器如下所示：

```typescript
@Client({
  transport: Transport.KAFKA,
  options: {
    client: {
      clientId: 'hero',
      brokers: ['localhost:9092'],
    },
    consumer: {
      groupId: 'hero-consumer'
    }
  }
})
client: ClientKafkaProxy;
```

#### 消息模式

Kafka 微服务的消息模式使用两个主题（topic）分别用于请求和响应通道。`ClientKafkaProxy.send()` 方法通过将 [返回地址](https://www.enterpriseintegrationpatterns.com/patterns/messaging/ReturnAddress.html) 与请求消息关联来发送消息，该关联包括 [关联 ID](https://www.enterpriseintegrationpatterns.com/patterns/messaging/CorrelationIdentifier.html)、回复主题和回复分区。这要求 `ClientKafkaProxy` 实例在发送消息之前已订阅回复主题并分配了至少一个分区。

因此，你需要为每个运行的 Nest 应用程序至少保留一个回复主题的分区。例如，如果你运行了 4 个 Nest 应用程序但回复主题只有 3 个分区，那么其中一个 Nest 应用程序在尝试发送消息时会报错。

当新的 `ClientKafkaProxy` 实例启动时，它们会加入消费者组并订阅其对应的主题。这个过程会触发消费者组内消费者所分配主题分区的重新平衡。

通常，主题分区是通过轮询分区器进行分配的，该分区器根据在应用程序启动时随机设置的消费者名称对消费者进行排序，并将主题分区分配给这些消费者。然而，当一个新的消费者加入消费者组时，新消费者可以位于消费者集合中的任意位置。这会导致当已有消费者位于新消费者之后时，被重新分配不同的分区。结果，这些消费者将丢失在重新平衡之前发送的请求的响应消息。

为防止 `ClientKafkaProxy` 消费者丢失响应消息，Nest 使用了一个特定的内置自定义分区器。该自定义分区器通过使用应用程序启动时设置的高精度时间戳 (`process.hrtime()`) 对消费者进行排序来分配分区。

#### 消息响应订阅

> warning **注意** 此部分仅在你使用 [请求-响应](/microservices/basics#request-response) 消息风格（使用 `@MessagePattern` 装饰器和 `ClientKafkaProxy.send` 方法）时相关。对于基于事件的通信（`@EventPattern` 装饰器和 `ClientKafkaProxy.emit` 方法），无需订阅响应主题。

`ClientKafkaProxy` 类提供了 `subscribeToResponseOf()` 方法。该方法接受一个请求主题名称作为参数，并将派生的回复主题名称添加到回复主题集合中。在实现消息模式时必须使用此方法。

```typescript
@@filename(heroes.controller)
onModuleInit() {
  this.client.subscribeToResponseOf('hero.kill.dragon');
}
```

如果 `ClientKafkaProxy` 实例是异步创建的，则必须在调用 `connect()` 方法之前调用 `subscribeToResponseOf()` 方法。

```typescript
@@filename(heroes.controller)
async onModuleInit() {
  this.client.subscribeToResponseOf('hero.kill.dragon');
  await this.client.connect();
}
```

#### 入站消息

Nest 接收的 Kafka 入站消息是一个包含 `key`、`value` 和 `headers` 属性的对象，其值的类型为 `Buffer`。随后，Nest 会通过将缓冲区转换为字符串来解析这些值。如果字符串类似于对象，Nest 会尝试将其解析为 `JSON`。然后将 `value` 传递给对应的处理器。

#### 出站消息

Nest 在发布事件或发送消息时，在序列化之后发送出站 Kafka 消息。这发生在传递给 `ClientKafkaProxy` 的 `emit()` 和 `send()` 方法的参数上，或发生在 `@MessagePattern` 方法返回的值上。此序列化过程会将非字符串或缓冲区的对象使用 `JSON.stringify()` 或 `toString()` 方法进行“字符串化”。

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @MessagePattern('hero.kill.dragon')
  killDragon(@Payload() message: KillDragonMessage): any {
    const dragonId = message.dragonId;
    const items = [
      { id: 1, name: 'Mythical Sword' },
      { id: 2, name: 'Key to Dungeon' },
    ];
    return items;
  }
}
```

> info **提示** `@Payload()` 是从 `@nestjs/microservices` 包中导入的。

出站消息也可以通过传递一个包含 `key` 和 `value` 属性的对象进行键值配对。键值配对对于满足 [共分区要求](https://docs.confluent.io/current/ksql/docs/developer-guide/partition-data.html#co-partitioning-requirements) 是非常重要的。

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @MessagePattern('hero.kill.dragon')
  killDragon(@Payload() message: KillDragonMessage): any {
    const realm = 'Nest';
    const heroId = message.heroId;
    const dragonId = message.dragonId;

    const items = [
      { id: 1, name: 'Mythical Sword' },
      { id: 2, name: 'Key to Dungeon' },
    ];

    return {
      headers: {
        realm
      },
      key: heroId,
      value: items
    }
  }
}
```

此外，以这种格式传递的消息还可以在 `headers` 哈希属性中设置自定义头。头哈希属性的值必须是 `string` 或 `Buffer` 类型。

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @MessagePattern('hero.kill.dragon')
  killDragon(@Payload() message: KillDragonMessage): any {
    const realm = 'Nest';
    const heroId = message.heroId;
    const dragonId = message.dragonId;

    const items = [
      { id: 1, name: 'Mythical Sword' },
      { id: 2, name: 'Key to Dungeon' },
    ];

    return {
      headers: {
        kafka_nestRealm: realm
      },
      key: heroId,
      value: items
    }
  }
}
```

#### 基于事件的通信

虽然请求-响应方法适合在服务之间交换消息，但在你的消息风格是基于事件的情况下（这在 Kafka 中更合适）——即你只想发布事件 **而无需等待响应** 时，这种方法并不理想。在这种情况下，你不希望请求-响应机制因维护两个主题而带来的开销。

查看以下两节以了解更多信息：[概述：基于事件](/microservices/basics#event-based) 和 [概述：发布事件](/microservices/basics#publishing-events)。

#### 上下文

在更复杂的场景中，你可能需要访问有关入站请求的额外信息。当使用 Kafka 传输器时，你可以访问 `KafkaContext` 对象。

```typescript
@@filename()
@MessagePattern('hero.kill.dragon')
killDragon(@Payload() message: KillDragonMessage, @Ctx() context: KafkaContext) {
  console.log(`Topic: ${context.getTopic()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('hero.kill.dragon')
killDragon(message, context) {
  console.log(`Topic: ${context.getTopic()}`);
}
```

> info **提示** `@Payload()`、`@Ctx()` 和 `KafkaContext` 是从 `@nestjs/microservices` 包中导入的。

要访问原始的 Kafka `IncomingMessage` 对象，请使用 `KafkaContext` 对象的 `getMessage()` 方法，如下所示：

```typescript
@@filename()
@MessagePattern('hero.kill.dragon')
killDragon(@Payload() message: KillDragonMessage, @Ctx() context: KafkaContext) {
  const originalMessage = context.getMessage();
  const partition = context.getPartition();
  const { headers, timestamp } = originalMessage;
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('hero.kill.dragon')
killDragon(message, context) {
  const originalMessage = context.getMessage();
  const partition = context.getPartition();
  const { headers, timestamp } = originalMessage;
}
```

其中 `IncomingMessage` 满足以下接口：

```typescript
interface IncomingMessage {
  topic: string;
  partition: number;
  timestamp: string;
  size: number;
  attributes: number;
  offset: string;
  key: any;
  value: any;
  headers: Record<string, any>;
}
```

如果你的处理器在处理每个接收到的消息时需要较长时间，建议使用 `heartbeat` 回调。要获取 `heartbeat` 函数，请使用 `KafkaContext` 的 `getHeartbeat()` 方法，如下所示：

```typescript
@@filename()
@MessagePattern('hero.kill.dragon')
async killDragon(@Payload() message: KillDragonMessage, @Ctx() context: KafkaContext) {
  const heartbeat = context.getHeartbeat();

  // 执行一些耗时操作
  await doWorkPart1();

  // 发送心跳以避免超过 sessionTimeout
  await heartbeat();

  // 再次执行一些耗时操作
  await doWorkPart2();
}
```

#### 命名约定

Kafka 微服务组件会将各自角色的描述附加到 `client.clientId` 和 `consumer.groupId` 选项上，以防止 Nest 微服务客户端和服务器组件之间的名称冲突。默认情况下，`ClientKafkaProxy` 组件在这些选项后附加 `-client`，而 `ServerKafka` 组件则附加 `-server`。注意以下提供的值是如何以这种方式转换的（如注释所示）。

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      clientId: 'hero', // hero-server
      brokers: ['localhost:9092'],
    },
    consumer: {
      groupId: 'hero-consumer' // hero-consumer-server
    },
  }
});
```

而对于客户端：

```typescript
@@filename(heroes.controller)
@Client({
  transport: Transport.KAFKA,
  options: {
    client: {
      clientId: 'hero', // hero-client
      brokers: ['localhost:9092'],
    },
    consumer: {
      groupId: 'hero-consumer' // hero-consumer-client
    }
  }
})
client: ClientKafkaProxy;
```

> info **提示** 你可以通过在你自己的自定义提供者中扩展 `ClientKafkaProxy` 和 `KafkaServer` 并重写构造函数来自定义 Kafka 客户端和消费者的命名约定。

由于 Kafka 微服务消息模式使用两个主题分别用于请求和回复通道，因此回复模式应从请求主题派生。默认情况下，回复主题的名称是请求主题名称后加上 `.reply`。

```typescript
@@filename(heroes.controller)
onModuleInit() {
  this.client.subscribeToResponseOf('hero.get'); // hero.get.reply
}
```

> info **提示** 你可以通过在你自己的自定义提供者中扩展 `ClientKafkaProxy` 并重写 `getResponsePatternName` 方法来自定义 Kafka 回复主题的命名约定。

#### 可重试的异常

与其他传输器类似，所有未处理的异常都会自动包装在 `RpcException` 中，并转换为“用户友好”的格式。但是，有时你可能希望绕过此机制，让异常由 `kafkajs` 驱动程序处理。在处理消息时抛出异常会指示 `kafkajs` 对其进行 **重试**（重新投递），这意味着即使消息（或事件）处理器被触发，偏移量也不会提交到 Kafka。

> warning **警告** 对于事件处理器（基于事件的通信），默认情况下所有未处理的异常都被视为 **可重试异常**。

为此，你可以使用一个名为 `KafkaRetriableException` 的专用类，如下所示：

```typescript
throw new KafkaRetriableException('...');
```

> info **提示** `KafkaRetriableException` 类是从 `@nestjs/microservices` 包中导出的。

#### 自定义异常处理

除了默认的错误处理机制之外，你还可以为 Kafka 事件创建自定义的异常过滤器来管理重试逻辑。例如，下面的示例演示了如何在配置的重试次数之后跳过有问题的事件：

```typescript
import { Catch, ArgumentsHost, Logger } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';
import { KafkaContext } from '../ctx-host';

@Catch()
export class KafkaMaxRetryExceptionFilter extends BaseExceptionFilter {
  private readonly logger = new Logger(KafkaMaxRetryExceptionFilter.name);

  constructor(
    private readonly maxRetries: number,
    // 可选的自定义函数，在超过最大重试次数时执行
    private readonly skipHandler?: (message: any) => Promise<void>,
  ) {
    super();
  }

  async catch(exception: unknown, host: ArgumentsHost) {
    const kafkaContext = host.switchToRpc().getContext<KafkaContext>();
    const message = kafkaContext.getMessage();
    const currentRetryCount = this.getRetryCountFromContext(kafkaContext);

    if (currentRetryCount >= this.maxRetries) {
      this.logger.warn(
        `最大重试次数（${
          this.maxRetries
        }）已超出：${JSON.stringify(message)}`,
      );

      if (this.skipHandler) {
        try {
          await this.skipHandler(message);
        } catch (err) {
          this.logger.error('skipHandler 中发生错误:', err);
        }
      }

      try {
        await this.commitOffset(kafkaContext);
      } catch (commitError) {
        this.logger.error('提交偏移量失败:', commitError);
      }
      return; // 停止异常传播
    }

    // 如果重试次数未达到最大值，则继续使用默认的 Exception Filter 逻辑
    super.catch(exception, host);
  }

  private getRetryCountFromContext(context: KafkaContext): number {
    const headers = context.getMessage().headers || {};
    const retryHeader = headers['retryCount'] || headers['retry-count'];
    return retryHeader ? Number(retryHeader) : 0;
  }

  private async commitOffset(context: KafkaContext): Promise<void> {
    const consumer = context.getConsumer && context.getConsumer();
    if (!consumer) {
      throw new Error('无法从 KafkaContext 获取消费者实例。');
    }

    const topic = context.getTopic && context.getTopic();
    const partition = context.getPartition && context.getPartition();
    const message = context.getMessage();
    const offset = message.offset;

    if (!topic || partition === undefined || offset === undefined) {
      throw new Error('提交偏移量所需的 Kafka 消息上下文不完整。');
    }

    await consumer.commitOffsets([
      {
        topic,
        partition,
        // 提交偏移量时，提交下一个数字（即当前偏移量 + 1）
        offset: (Number(offset) + 1).toString(),
      },
    ]);
  }
}
```

此过滤器提供了一种最多可重试处理 Kafka 事件的机制。一旦达到最大重试次数，它将触发一个自定义的 `skipHandler`（如果提供），并提交偏移量，从而有效地跳过有问题的事件。这允许后续事件在不中断的情况下继续处理。

你可以通过将此过滤器添加到你的事件处理器中来集成它：

```typescript
@UseFilters(new KafkaMaxRetryExceptionFilter(5))
export class MyEventHandler {
  @EventPattern('your-topic')
  async handleEvent(@Payload() data: any, @Ctx() context: KafkaContext) {
    // 你的事件处理逻辑...
  }
}
```

#### 提交偏移量

在使用 Kafka 时，提交偏移量是至关重要的。默认情况下，消息会在特定时间后自动提交。更多信息请访问 [KafkaJS 文档](https://kafka.js.org/docs/consuming#autocommit)。`KafkaContext` 提供了一种访问活动消费者的途径，以便手动提交偏移量。该消费者是 KafkaJS 消费者，其行为与 [原生 KafkaJS 实现](https://kafka.js.org/docs/consuming#manual-committing) 一致。

```typescript
@@filename()
@EventPattern('user.created')
async handleUserCreated(@Payload() data: IncomingMessage, @Ctx() context: KafkaContext) {
  // 业务逻辑

  const { offset } = context.getMessage();
  const partition = context.getPartition();
  const topic = context.getTopic();
  const consumer = context.getConsumer();
  await consumer.commitOffsets([{ topic, partition, offset }])
}
@@switch
@Bind(Payload(), Ctx())
@EventPattern('user.created')
async handleUserCreated(data, context) {
  // 业务逻辑

  const { offset } = context.getMessage();
  const partition = context.getPartition();
  const topic = context.getTopic();
  const consumer = context.getConsumer();
  await consumer.commitOffsets([{ topic, partition, offset }])
}
```

要禁用消息的自动提交，请在 `run` 配置中设置 `autoCommit: false`，如下所示：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      brokers: ['localhost:9092'],
    },
    run: {
      autoCommit: false
    }
  }
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      brokers: ['localhost:9092'],
    },
    run: {
      autoCommit: false
    }
  }
});
```

#### 实例状态更新

要实时获取连接和底层驱动实例状态的更新，你可以订阅 `status` 流。此流提供特定于所选驱动的状态更新。对于 Kafka 驱动，`status` 流会发出 `connected`、`disconnected`、`rebalancing`、`crashed` 和 `stopped` 事件。

```typescript
this.client.status.subscribe((status: KafkaStatus) => {
  console.log(status);
});
```

> info **提示** `KafkaStatus` 类型是从 `@nestjs/microservices` 包中导入的。

同样，你可以订阅服务器的 `status` 流以接收服务器状态的通知。

```typescript
const server = app.connectMicroservice<MicroserviceOptions>(...);
server.status.subscribe((status: KafkaStatus) => {
  console.log(status);
});
```

#### 底层的生产者和消费者

对于更高级的用例，你可能需要访问底层的生产者和消费者实例。这在手动关闭连接或使用驱动特定的方法时非常有用。不过请注意，在大多数情况下，你 **不需要** 直接访问驱动。

为此，你可以使用 `ClientKafkaProxy` 实例暴露的 `producer` 和 `consumer` getter。

```typescript
const producer = this.client.producer;
const consumer = this.client.consumer;
```