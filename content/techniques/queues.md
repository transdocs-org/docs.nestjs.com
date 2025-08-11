### 队列

队列是一种强大的设计模式，有助于解决常见的应用程序扩展和性能挑战。队列可以协助解决的一些问题示例如下：

- **平滑处理峰值**。例如，如果用户可以在任意时间触发资源密集型任务，您可以将这些任务添加到队列中，而不是同步执行。然后，您可以使用工作进程以受控的方式从队列中拉取任务。随着应用程序的扩展，您可以轻松地添加新的队列消费者来提升后端任务的处理能力。
- **分解可能阻塞 Node.js 事件循环的单体任务**。例如，如果一个用户请求需要执行像音频转码这样 CPU 密集型的工作，您可以将此任务委托给其他进程，从而释放面向用户的进程以保持响应性。
- **在各种服务之间提供可靠的通信通道**。例如，您可以在一个进程或服务中排队任务（作业），然后在另一个进程中消费它们。您可以监听作业生命周期中的完成、错误或其他状态变更事件来获得通知。当队列生产者或消费者发生故障时，其状态将被保留，并且在节点重启后任务处理可以自动恢复。

Nest 提供了 `@nestjs/bullmq` 包用于集成 BullMQ，以及 `@nestjs/bull` 包用于集成 Bull。这两个包都是对各自库的封装/抽象，这些库由同一团队开发。Bull 目前处于维护模式，团队主要致力于修复错误，而 BullMQ 正在积极开发中，具有现代的 TypeScript 实现和不同的功能集。如果 Bull 满足您的需求，它仍然是一个可靠且经过验证的选择。Nest 的这些包使得将 BullMQ 或 Bull 队列以友好的方式集成到 Nest 应用程序中变得简单。

BullMQ 和 Bull 都使用 [Redis](https://redis.io/) 来持久化作业数据，因此您需要在系统上安装 Redis。由于它们是基于 Redis 的，所以您的队列架构可以完全分布式且与平台无关。例如，您可以在一个（或多个）节点上的 Nest 中运行一些队列 <a href="techniques/queues#producers">生产者</a>、<a href="techniques/queues#consumers">消费者</a> 和 <a href="techniques/queues#event-listeners">监听器</a>，而其他生产者、消费者和监听器可以在其他网络节点上的其他 Node.js 平台上运行。

本章涵盖 `@nestjs/bullmq` 和 `@nestjs/bull` 包的内容。我们还建议阅读 [BullMQ](https://docs.bullmq.io/) 和 [Bull](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md) 的文档以获取更多背景知识和具体实现细节。

#### BullMQ 安装

要开始使用 BullMQ，我们首先安装所需的依赖项。

```bash
$ npm install --save @nestjs/bullmq bullmq
```

安装完成后，我们可以将 `BullModule` 导入到根模块 `AppModule` 中。

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bullmq';

@Module({
  imports: [
    BullModule.forRoot({
      connection: {
        host: 'localhost',
        port: 6379,
      },
    }),
  ],
})
export class AppModule {}
```

`forRoot()` 方法用于注册一个 `bullmq` 包的配置对象，该对象将被应用程序中注册的所有队列使用（除非另有指定）。以下是一些配置对象中的属性，供您参考：

- `connection: ConnectionOptions` - 用于配置 Redis 连接的选项。更多信息请参见 [Connections](https://docs.bullmq.io/guide/connections)。可选。
- `prefix: string` - 所有队列键的前缀。可选。
- `defaultJobOptions: JobOpts` - 控制新作业默认设置的选项。更多信息请参见 [JobOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd)。可选。
- `settings: AdvancedSettings` - 高级队列配置设置。通常不应更改。更多信息请参见 [AdvancedSettings](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue)。可选。
- `extraOptions` - 模块初始化的额外选项。更多信息请参见 [Manual Registration](https://docs.nestjs.com/techniques/queues#manual-registration)

所有选项都是可选的，提供对队列行为的详细控制。这些选项直接传递给 BullMQ 的 `Queue` 构造函数。更多关于这些选项和其他选项的信息请参见 [这里](https://api.docs.bullmq.io/interfaces/v4.QueueOptions.html)。

要注册一个队列，请导入 `BullModule.registerQueue()` 动态模块，如下所示：

```typescript
BullModule.registerQueue({
  name: 'audio',
});
```

> info **提示** 通过向 `registerQueue()` 方法传递多个逗号分隔的配置对象，可以创建多个队列。

`registerQueue()` 方法用于实例化和/或注册队列。队列在连接到相同底层 Redis 数据库并使用相同凭据的模块和进程之间共享。每个队列通过其 `name` 属性唯一标识。队列名称既用作注入令牌（用于将队列注入到控制器/提供者中），也用作装饰器的参数，以将消费者类和监听器与队列关联。

您还可以为特定队列覆盖某些预配置的选项，如下所示：

```typescript
BullModule.registerQueue({
  name: 'audio',
  connection: {
    port: 6380,
  },
});
```

BullMQ 还支持作业之间的父子关系。此功能允许创建作业树，这些树可以具有任意深度。要了解更多，请参见 [这里](https://docs.bullmq.io/guide/flows)。

要添加一个流程，可以执行以下操作：

```typescript
BullModule.registerFlowProducer({
  name: 'flowProducerName',
});
```

由于作业被持久化在 Redis 中，每次实例化特定命名的队列（例如，当应用程序启动/重启时），它都会尝试处理可能存在的旧作业（来自之前未完成的会话）。

每个队列可以有多个生产者、消费者和监听器。消费者以特定顺序从队列中检索作业：FIFO（默认）、LIFO 或根据优先级。控制队列处理顺序的内容在 <a href="techniques/queues#consumers">此处</a> 讨论。

<app-banner-enterprise></app-banner-enterprise>

#### 命名配置

如果您的队列连接到多个不同的 Redis 实例，则可以使用一种称为 **命名配置** 的技术。此功能允许您在指定的键下注册多个配置，然后可以在队列选项中引用这些配置。

例如，假设您有一个额外的 Redis 实例（除了默认的实例）被应用程序中注册的一些队列使用，您可以如下注册其配置：

```typescript
BullModule.forRoot('alternative-config', {
  connection: {
    port: 6381,
  },
});
```

在上面的示例中，`'alternative-config'` 只是一个配置键（可以是任意字符串）。

有了这个配置后，您现在可以在 `registerQueue()` 的选项对象中指向此配置：

```typescript
BullModule.registerQueue({
  configKey: 'alternative-config',
  name: 'video',
});
```

#### 生产者

作业生产者将作业添加到队列中。生产者通常是应用程序服务（Nest [提供者](/providers)）。要将作业添加到队列中，请首先将队列注入到服务中，如下所示：

```typescript
import { Injectable } from '@nestjs/common';
import { Queue } from 'bullmq';
import { InjectQueue } from '@nestjs/bullmq';

@Injectable()
export class AudioService {
  constructor(@InjectQueue('audio') private audioQueue: Queue) {}
}
```

> info **提示** `@InjectQueue()` 装饰器通过其名称标识队列，该名称是在 `registerQueue()` 方法调用中提供的（例如 `'audio'`）。

现在，通过调用队列的 `add()` 方法并传递一个用户定义的作业对象来添加作业。作业表示为可序列化的 JavaScript 对象（因为它们以这种方式存储在 Redis 数据库中）。您传递的作业形状是任意的；请使用它来表示作业对象的语义。您还需要为其指定一个名称。这允许您创建专门的 <a href="techniques/queues#consumers">消费者</a>，这些消费者只会处理具有给定名称的作业。

```typescript
const job = await this.audioQueue.add('transcode', {
  foo: 'bar',
});
```

#### 作业选项

作业可以具有与其关联的额外选项。在 `Queue.add()` 方法中，在 `job` 参数之后传递一个选项对象。一些作业选项属性如下：

- `priority`: `number` - 可选的优先级值。范围从 1（最高优先级）到 MAX_INT（最低优先级）。请注意，使用优先级会对性能产生轻微影响，因此请谨慎使用。
- `delay`: `number` - 等待作业可以处理的时间量（以毫秒为单位）。请注意，为了准确的延迟，服务器和客户端的时钟应保持同步。
- `attempts`: `number` - 尝试作业直到完成的总次数。
- `repeat`: `RepeatOpts` - 根据 cron 规范重复作业。更多信息请参见 [RepeatOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd)。
- `backoff`: `number | BackoffOpts` - 如果作业失败，自动重试的退避设置。更多信息请参见 [BackoffOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd)。
- `lifo`: `boolean` - 如果为 true，则将作业添加到队列的右端而不是左端（默认为 false）。
- `jobId`: `number` | `string` - 覆盖作业 ID - 默认情况下，作业 ID 是一个唯一的整数，但您可以使用此设置进行覆盖。如果您使用此选项，则由您确保 jobId 是唯一的。如果您尝试添加一个已存在的 ID 的作业，则不会被添加。
- `removeOnComplete`: `boolean | number` - 如果为 true，则在作业成功完成后将其移除。数字指定要保留的作业数量。默认行为是将作业保留在已完成集合中。
- `removeOnFail`: `boolean | number` - 如果为 true，则在作业失败后所有尝试后将其移除。数字指定要保留的作业数量。默认行为是将作业保留在失败集合中。
- `stackTraceLimit`: `number` - 限制记录在堆栈跟踪中的堆栈跟踪行数。

以下是使用作业选项自定义作业的一些示例。

要延迟作业的启动，请使用 `delay` 配置属性。

```typescript
const job = await this.audioQueue.add(
  'transcode',
  {
    foo: 'bar',
  },
  { delay: 3000 }, // 延迟 3 秒
);
```

要将作业添加到队列的右端（以 **LIFO**（后进先出）的方式处理作业），请将配置对象的 `lifo` 属性设置为 `true`。

```typescript
const job = await this.audioQueue.add(
  'transcode',
  {
    foo: 'bar',
  },
  { lifo: true },
);
```

要优先处理作业，请使用 `priority` 属性。

```typescript
const job = await this.audioQueue.add(
  'transcode',
  {
    foo: 'bar',
  },
  { priority: 2 },
);
```

有关选项的完整列表，请参见 API 文档 [这里](https://api.docs.bullmq.io/types/v4.JobsOptions.html) 和 [这里](https://api.docs.bullmq.io/interfaces/v4.BaseJobOptions.html)。

#### 消费者

消费者是一个 **类**，该类定义的方法用于处理添加到队列中的作业，或监听队列上的事件，或两者兼具。使用 `@Processor()` 装饰器声明消费者类，如下所示：

```typescript
import { Processor } from '@nestjs/bullmq';

@Processor('audio')
export class AudioConsumer {}
```

> info **提示** 消费者必须注册为 `providers`，以便 `@nestjs/bullmq` 包能够识别它们。

其中，装饰器的字符串参数（例如 `'audio'`）是与类方法关联的队列名称。

```typescript
import { Processor, WorkerHost } from '@nestjs/bullmq';
import { Job } from 'bullmq';

@Processor('audio')
export class AudioConsumer extends WorkerHost {
  async process(job: Job<any, any, string>): Promise<any> {
    let progress = 0;
    for (let i = 0; i < 100; i++) {
      await doSomething(job.data);
      progress += 1;
      await job.updateProgress(progress);
    }
    return {};
  }
}
```

每当工作进程空闲且队列中有作业需要处理时，都会调用 process 方法。此处理方法接收 `job` 对象作为其唯一参数。处理方法返回的值将存储在作业对象中，并可在稍后访问，例如在完成事件的监听器中。

`Job` 对象具有多个方法，允许您与其状态进行交互。例如，上述代码使用 `progress()` 方法更新作业的进度。有关完整的 `Job` 对象 API 参考，请参见 [这里](https://api.docs.bullmq.io/classes/v4.Job.html)。

在旧版本 Bull 中，您可以通过向 `@Process()` 装饰器传递 `name` 来指定某个作业处理方法 **仅** 处理特定类型的作业（具有特定 `name` 的作业），如下所示。

> warning **警告** 这在 BullMQ 中不起作用，请继续阅读。

```typescript
@Process('transcode')
async transcode(job: Job<unknown>) { ... }
```

由于由此产生的混淆，BullMQ 中不支持这种行为。相反，您需要使用 switch case 来为每个作业名称调用不同的服务或逻辑：

```typescript
import { Processor, WorkerHost } from '@nestjs/bullmq';
import { Job } from 'bullmq';

@Processor('audio')
export class AudioConsumer extends WorkerHost {
  async process(job: Job<any, any, string>): Promise<any> {
    switch (job.name) {
      case 'transcode': {
        let progress = 0;
        for (i = 0; i < 100; i++) {
          await doSomething(job.data);
          progress += 1;
          await job.progress(progress);
        }
        return {};
      }
      case 'concatenate': {
        await doSomeLogic2();
        break;
      }
    }
  }
}
```

这在 BullMQ 文档的 [命名处理器](https://docs.bullmq.io/patterns/named-processor) 部分有更详细的说明。

#### 请求作用域消费者

当消费者被标记为请求作用域时（关于注入作用域的更多信息请参见 [这里](/fundamentals/injection-scopes#provider-scope)），将为每个作业专门创建一个类的新实例。作业完成后，该实例将被垃圾回收。

```typescript
@Processor({
  name: 'audio',
  scope: Scope.REQUEST,
})
```

由于请求作用域的消费者类是动态实例化的并作用域到单个作业，因此您可以使用标准方法通过构造函数注入 `JOB_REF`。

```typescript
constructor(@Inject(JOB_REF) jobRef: Job) {
  console.log(jobRef);
}
```

> info **提示** `JOB_REF` 令牌是从 `@nestjs/bullmq` 包中导入的。

#### 事件监听器

当队列和/或作业状态发生变化时，BullMQ 会生成一组有用的事件。这些事件可以通过 `@OnWorkerEvent(event)` 装饰器在 Worker 级别订阅，或者通过专用的监听器类和 `@OnQueueEvent(event)` 装饰器在队列级别订阅。

Worker 事件必须在 <a href="techniques/queues#consumers">消费者</a> 类（即，使用 `@Processor()` 装饰器装饰的类）中声明。要监听事件，请使用带有要处理的事件的 `@OnWorkerEvent(event)` 装饰器。例如，要监听在 `audio` 队列中作业进入激活状态时发出的事件，请使用以下构造：

```typescript
import { Processor, Process, OnWorkerEvent } from '@nestjs/bullmq';
import { Job } from 'bullmq';

@Processor('audio')
export class AudioConsumer {
  @OnWorkerEvent('active')
  onActive(job: Job) {
    console.log(
      `Processing job ${job.id} of type ${job.name} with data ${job.data}...`,
    );
  }

  // ...
}
```

您可以在 WorkerListener 的属性中看到事件及其参数的完整列表 [这里](https://api.docs.bullmq.io/interfaces/v4.WorkerListener.html)。

QueueEvent 监听器必须使用 `@QueueEventsListener(queue)` 装饰器，并继承 `@nestjs/bullmq` 提供的 `QueueEventsHost` 类。要监听事件，请使用带有要处理的事件的 `@OnQueueEvent(event)` 装饰器。例如，要监听在 `audio` 队列中作业进入激活状态时发出的事件，请使用以下构造：

```typescript
import {
  QueueEventsHost,
  QueueEventsListener,
  OnQueueEvent,
} from '@nestjs/bullmq';

@QueueEventsListener('audio')
export class AudioEventsListener extends QueueEventsHost {
  @OnQueueEvent('active')
  onActive(job: { jobId: string; prev?: string }) {
    console.log(`Processing job ${job.jobId}...`);
  }

  // ...
}
```

> info **提示** QueueEvent 监听器必须注册为 `providers`，以便 `@nestjs/bullmq` 包能够识别它们。

您可以在 QueueEventsListener 的属性中看到事件及其参数的完整列表 [这里](https://api.docs.bullmq.io/interfaces/v4.QueueEventsListener.html)。

#### 队列管理

队列有一个 API，允许您执行管理功能，如暂停和恢复、检索处于各种状态的作业计数等。您可以在 [这里](https://api.docs.bullmq.io/classes/v4.Queue.html) 找到完整的队列 API。直接在 `Queue` 对象上调用这些方法中的任何一个，如下所示使用暂停/恢复示例。

使用 `pause()` 方法调用暂停队列。暂停的队列在恢复之前不会处理新作业，但当前正在处理的作业将继续执行直到完成。

```typescript
await audioQueue.pause();
```

要恢复暂停的队列，请使用 `resume()` 方法，如下所示：

```typescript
await audioQueue.resume();
```

#### 单独的进程

作业处理程序也可以在单独的（分叉的）进程中运行 ([源](https://docs.bullmq.io/guide/workers/sandboxed-processors))。这有几个优点：

- 进程是沙箱化的，因此如果它崩溃不会影响工作进程。
- 您可以运行阻塞代码而不影响队列（作业不会挂起）。
- 更好地利用多核 CPU。
- 减少对 redis 的连接。

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bullmq';
import { join } from 'path';

@Module({
  imports: [
    BullModule.registerQueue({
      name: 'audio',
      processors: [join(__dirname, 'processor.js')],
    }),
  ],
})
export class AppModule {}
```

> warning **警告** 请注意，由于您的函数是在分叉的进程中执行的，因此依赖注入（和 IoC 容器）将不可用。这意味着您的处理器函数需要包含（或创建）所需的所有外部依赖项实例。

#### 异步配置

您可能希望异步传递 `bullmq` 选项而不是静态传递。在这种情况下，请使用 `forRootAsync()` 方法，它提供了几种处理异步配置的方法。同样，如果您希望异步传递队列选项，请使用 `registerQueueAsync()` 方法。

一种方法是使用工厂函数：

```typescript
BullModule.forRootAsync({
  useFactory: () => ({
    connection: {
      host: 'localhost',
      port: 6379,
    },
  }),
});
```

我们的工厂行为像任何其他 [异步提供者](https://docs.nestjs.com/fundamentals/async-providers) 一样（例如，它可以是 `async` 的，并且可以通过 `inject` 注入依赖项）。

```typescript
BullModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    connection: {
      host: configService.get('QUEUE_HOST'),
      port: configService.get('QUEUE_PORT'),
    },
  }),
  inject: [ConfigService],
});
```

或者，您可以使用 `useClass` 语法：

```typescript
BullModule.forRootAsync({
  useClass: BullConfigService,
});
```

上面的构造将在 `BullModule` 内部实例化 `BullConfigService` 并使用它通过调用 `createSharedConfiguration()` 提供一个选项对象。请注意，这意味着 `BullConfigService` 必须实现 `SharedBullConfigurationFactory` 接口，如下所示：

```typescript
@Injectable()
class BullConfigService implements SharedBullConfigurationFactory {
  createSharedConfiguration(): BullModuleOptions {
    return {
      connection: {
        host: 'localhost',
        port: 6379,
      },
    };
  }
}
```

为了防止在 `BullModule` 内部创建 `BullConfigService` 并使用从其他模块导入的提供者，您可以使用 `useExisting` 语法。

```typescript
BullModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

这种构造与 `useClass` 的工作方式相同，只有一个关键区别 - `BullModule` 将查找导入的模块以重用现有的 `ConfigService`，而不是实例化一个新的。

同样，如果您希望异步传递队列选项，请使用 `registerQueueAsync()` 方法，只需记住在工厂函数外部指定 `name` 属性。

```typescript
BullModule.registerQueueAsync({
  name: 'audio',
  useFactory: () => ({
    redis: {
      host: 'localhost',
      port: 6379,
    },
  }),
});
```

#### 手动注册

默认情况下，`BullModule` 在 `onModuleInit` 生命周期函数中自动注册 BullMQ 组件（队列、处理器和事件监听服务）。然而，在某些情况下，这种行为可能并不理想。要防止自动注册，请在 `BullModule` 中启用 `manualRegistration`，如下所示：

```typescript
BullModule.forRoot({
  extraOptions: {
    manualRegistration: true,
  },
});
```

要手动注册这些组件，请注入 `BullRegistrar` 并调用 `register` 函数，理想情况下在 `OnModuleInit` 或 `OnApplicationBootstrap` 中调用。

```typescript
import { Injectable, OnModuleInit } from '@nestjs/common';
import { BullRegistrar } from '@nestjs/bullmq';

@Injectable()
export class AudioService implements OnModuleInit {
  constructor(private bullRegistrar: BullRegistrar) {}

  onModuleInit() {
    if (yourConditionHere) {
      this.bullRegistrar.register();
    }
  }
}
```

除非调用 `BullRegistrar#register` 函数，否则任何 BullMQ 组件都不会工作 - 这意味着不会处理任何作业。

#### Bull 安装

> warning **注意** 如果您决定使用 BullMQ，请跳过本节和后续章节。

要开始使用 Bull，我们首先安装所需的依赖项。

```bash
$ npm install --save @nestjs/bull bull
```

安装完成后，我们可以将 `BullModule` 导入到根模块 `AppModule` 中。

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bull';

@Module({
  imports: [
    BullModule.forRoot({
      redis: {
        host: 'localhost',
        port: 6379,
      },
    }),
  ],
})
export class AppModule {}
```

`forRoot()` 方法用于注册一个 `bull` 包的配置对象，该对象将被应用程序中注册的所有队列使用（除非另有指定）。配置对象包含以下属性：

- `limiter: RateLimiter` - 控制队列作业处理速率的选项。更多信息请参见 [RateLimiter](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue)。可选。
- `redis: RedisOpts` - 配置 Redis 连接的选项。更多信息请参见 [RedisOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue)。可选。
- `prefix: string` - 所有队列键的前缀。可选。
- `defaultJobOptions: JobOpts` - 控制新作业默认设置的选项。更多信息请参见 [JobOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd)。可选。**注意：如果您通过 FlowProducer 调度作业，这些设置将不起作用。有关解释，请参见 [bullmq#1034](https://github.com/taskforcesh/bullmq/issues/1034)。**
- `settings: AdvancedSettings` - 高级队列配置设置。这些通常不应更改。更多信息请参见 [AdvancedSettings](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue)。可选。

所有选项都是可选的，提供对队列行为的详细控制。这些选项直接传递给 Bull 的 `Queue` 构造函数。有关这些选项的更多信息，请参见 [这里](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue)。

要注册一个队列，请导入 `BullModule.registerQueue()` 动态模块，如下所示：

```typescript
BullModule.registerQueue({
  name: 'audio',
});
```

> info **提示** 通过向 `registerQueue()` 方法传递多个逗号分隔的配置对象，可以创建多个队列。

`registerQueue()` 方法用于实例化和/或注册队列。队列在连接到相同底层 Redis 数据库并使用相同凭据的模块和进程之间共享。每个队列通过其 `name` 属性唯一标识。队列名称既用作注入令牌（用于将队列注入到控制器/提供者中），也用作装饰器的参数，以将消费者类和监听器与队列关联。

您还可以覆盖特定队列的某些预配置选项，如下所示：

```typescript
BullModule.registerQueue({
  name: 'audio',
  redis: {
    port: 6380,
  },
});
```

由于作业在 Redis 中持久化，每次实例化特定命名的队列（例如，当应用程序启动/重启时），它都会尝试处理可能存在的旧作业（来自之前未完成的会话）。

每个队列可以有一个或多个生产者、消费者和监听器。消费者以特定顺序从队列中检索作业：FIFO（默认）、LIFO 或根据优先级。控制队列处理顺序的内容在 <a href="techniques/queues#consumers">此处</a> 讨论。

<app-banner-enterprise></app-banner-enterprise>

#### 命名配置

如果您的队列连接到多个 Redis 实例，则可以使用一种称为 **命名配置** 的技术。此功能允许您在指定的键下注册多个配置，然后可以在队列选项中引用这些配置。

例如，假设您有一个额外的 Redis 实例（除了默认实例）被应用程序中注册的一些队列使用，您可以如下注册其配置：

```typescript
BullModule.forRoot('alternative-config', {
  redis: {
    port: 6381,
  },
});
```

在上面的示例中，`'alternative-config'` 只是一个配置键（可以是任意字符串）。

有了这个配置后，您现在可以在 `registerQueue()` 的选项对象中指向此配置：

```typescript
BullModule.registerQueue({
  configKey: 'alternative-config',
  name: 'video',
});
```

#### 生产者

作业生产者将作业添加到队列中。生产者通常是应用程序服务（Nest [提供者](/providers)）。要将作业添加到队列中，请首先将队列注入到服务中，如下所示：

```typescript
import { Injectable } from '@nestjs/common';
import { Queue } from 'bull';
import { InjectQueue } from '@nestjs/bull';

@Injectable()
export class AudioService {
  constructor(@InjectQueue('audio') private audioQueue: Queue) {}
}
```

> info **提示** `@InjectQueue()` 装饰器通过其名称标识队列，该名称是在 `registerQueue()` 方法调用中提供的（例如 `'audio'`）。

现在，通过调用队列的 `add()` 方法并传递一个用户定义的作业对象来添加作业。作业表示为可序列化的 JavaScript 对象（因为它们以这种方式存储在 Redis 数据库中）。您传递的作业形状是任意的；请使用它来表示作业对象的语义。

```typescript
const job = await this.audioQueue.add({
  foo: 'bar',
});
```

#### 命名作业

作业可以具有唯一的名称。这允许您创建专门的 <a href="techniques/queues#consumers">消费者</a>，这些消费者只会处理具有给定名称的作业。

```typescript
const job = await this.audioQueue.add('transcode', {
  foo: 'bar',
});
```

> 警告 **警告** 使用命名作业时，您必须为队列中添加的每个唯一名称创建处理器，否则队列会报错缺少给定作业的处理器。有关消费命名作业的更多信息，请参见 <a href="techniques/queues#consumers">此处</a>。

#### 作业选项

作业可以具有与其关联的额外选项。在 `Queue.add()` 方法中，在 `job` 参数之后传递一个选项对象。作业选项属性如下：

- `priority`: `number` - 可选的优先级值。范围从 1（最高优先级）到 MAX_INT（最低优先级）。请注意，使用优先级会对性能产生轻微影响，因此请谨慎使用。
- `delay`: `number` - 等待作业可以处理的时间量（以毫秒为单位）。请注意，为了准确的延迟，服务器和客户端的时钟应保持同步。
- `attempts`: `number` - 尝试作业直到完成的总次数。
- `repeat`: `RepeatOpts` - 根据 cron 规范重复作业。更多信息请参见 [RepeatOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd)。
- `backoff`: `number | BackoffOpts` - 如果作业失败，自动重试的退避设置。更多信息请参见 [BackoffOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd)。
- `lifo`: `boolean` - 如果为 true，则将作业添加到队列的右端而不是左端（默认为 false）。
- `timeout`: `number` - 作业应在多少毫秒后因超时错误而失败。
- `jobId`: `number` | `string` - 覆盖作业 ID - 默认情况下，作业 ID 是一个唯一的整数，但您可以使用此设置进行覆盖。如果您使用此选项，则由您确保 jobId 是唯一的。如果您尝试添加一个已存在的 ID 的作业，则不会被添加。
- `removeOnComplete`: `boolean | number` - 如果为 true，则在作业成功完成后将其移除。数字指定要保留的作业数量。默认行为是将作业保留在已完成集合中。
- `removeOnFail`: `boolean | number` - 如果为 true，则在作业失败后所有尝试后将其移除。数字指定要保留的作业数量。默认行为是将作业保留在失败集合中。
- `stackTraceLimit`: `number` - 限制记录在堆栈跟踪中的堆栈跟踪行数。

以下是使用作业选项自定义作业的一些示例。

要延迟作业的启动，请使用 `delay` 配置属性。

```typescript
const job = await this.audioQueue.add(
  {
    foo: 'bar',
  },
  { delay: 3000 }, // 延迟 3 秒
);
```

要将作业添加到队列的右端（以 **LIFO**（后进先出）的方式处理作业），请将配置对象的 `lifo` 属性设置为 `true`。

```typescript
const job = await this.audioQueue.add(
  {
    foo: 'bar',
  },
  { lifo: true },
);
```

要优先处理作业，请使用 `priority` 属性。

```typescript
const job = await this.audioQueue.add(
  {
    foo: 'bar',
  },
  { priority: 2 },
);
```

#### 消费者

消费者是一个 **类**，该类定义的方法用于处理添加到队列中的作业，或监听队列上的事件，或两者兼具。使用 `@Processor()` 装饰器声明消费者类，如下所示：

```typescript
import { Processor } from '@nestjs/bull';

@Processor('audio')
export class AudioConsumer {}
```

> info **提示** 消费者必须注册为 `providers`，以便 `@nestjs/bull` 包能够识别它们。

其中，装饰器的字符串参数（例如 `'audio'`）是与类方法关联的队列名称。

在消费者类中，通过使用 `@Process()` 装饰器装饰处理方法来声明作业处理程序。

```typescript
import { Processor, Process } from '@nestjs/bull';
import { Job } from 'bull';

@Processor('audio')
export class AudioConsumer {
  @Process()
  async transcode(job: Job<unknown>) {
    let progress = 0;
    for (let i = 0; i < 100; i++) {
      await doSomething(job.data);
      progress += 1;
      await job.progress(progress);
    }
    return {};
  }
}
```

每当工作进程空闲且队列中有作业需要处理时，都会调用装饰的方法（例如 `transcode()`）。此处理方法接收 `job` 对象作为其唯一参数。处理方法返回的值将存储在作业对象中，并可在稍后访问，例如在完成事件的监听器中。

`Job` 对象具有多个方法，允许您与其状态进行交互。例如，上述代码使用 `progress()` 方法更新作业的进度。有关完整的 `Job` 对象 API 参考，请参见 [这里](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#job)。

您可以通过向 `@Process()` 装饰器传递 `name` 来指定某个作业处理方法 **仅** 处理特定类型的作业（具有特定 `name` 的作业），如下所示。您可以在给定的消费者类中拥有多个 `@Process()` 处理程序，对应每个作业类型（`name`）。当使用命名作业时，请确保每个名称都有对应的处理程序。

```typescript
@Process('transcode')
async transcode(job: Job<unknown>) { ... }
```

> warning **警告** 在为同一队列定义多个消费者时，`@Process({{ '{' }} concurrency: 1 {{ '}' }})` 中的 `concurrency` 选项将不起作用。最小的 `concurrency` 将匹配定义的消费者数量。即使 `@Process()` 处理程序使用不同的 `name` 来处理命名作业，也是如此。

#### 请求作用域消费者

当消费者被标记为请求作用域时（关于注入作用域的更多信息请参见 [这里](/fundamentals/injection-scopes#provider-scope)），将为每个作业专门创建一个类的新实例。作业完成后，该实例将被垃圾回收。

```typescript
@Processor({
  name: 'audio',
  scope: Scope.REQUEST,
})
```

由于请求作用域的消费者类是动态实例化的并作用域到单个作业，因此您可以使用标准方法通过构造函数注入 `JOB_REF`。

```typescript
constructor(@Inject(JOB_REF) jobRef: Job) {
  console.log(jobRef);
}
```

> info **提示** `JOB_REF` 令牌是从 `@nestjs/bull` 包中导入的。

#### 事件监听器

当队列和/或作业状态发生变化时，Bull 会生成一组有用的事件。Nest 提供了一组装饰器，允许订阅一组核心标准事件。这些装饰器导自 `@nestjs/bull` 包。

事件监听器必须在 <a href="techniques/queues#consumers">消费者</a> 类中声明（即，在使用 `@Processor()` 装饰器装饰的类中）。要监听事件，请使用下表中的装饰器之一来声明事件的处理程序。例如，要监听在 `audio` 队列中作业进入激活状态时发出的事件，请使用以下构造：

```typescript
import { Processor, Process, OnQueueActive } from '@nestjs/bull';
import { Job } from 'bull';

@Processor('audio')
export class AudioConsumer {

  @OnQueueActive()
  onActive(job: Job) {
    console.log(
      `Processing job ${job.id} of type ${job.name} with data ${job.data}...`,
    );
  }
  ...
```

由于 Bull 在分布式（多节点）环境中运行，它定义了事件局部性的概念。这个概念认识到事件可能完全在单个进程中触发，也可能在不同进程的共享队列上触发。**本地** 事件是指当在本地进程的队列上触发操作或状态变更时生成的事件。换句话说，当您的事件生产者和消费者位于单个进程中时，队列上发生的所有事件都是本地的。

当一个队列在多个进程中共享时，我们可能会遇到 **全局** 事件的可能性。为了使一个进程中的监听器接收到另一个进程触发的事件通知，它必须注册一个全局事件。

每当触发相应的事件时，事件处理程序就会被调用。处理程序使用下表中显示的签名调用，提供对与事件相关的信息的访问。我们将在下面讨论本地和全局事件处理程序签名的一个关键区别。

<table>
  <tr>
    <th>本地事件监听器</th>
    <th>全局事件监听器</th>
    <th>处理方法签名 / 触发时机</th>
  </tr>
  <tr>
    <td><code>@OnQueueError()</code></td><td><code>@OnGlobalQueueError()</code></td><td><code>handler(error: Error)</code> - 发生错误。<code>error</code> 包含触发错误。</td>
  </tr>
  <tr>
    <td><code>@OnQueueWaiting()</code></td><td><code>@OnGlobalQueueWaiting()</code></td><td><code>handler(jobId: number | string)</code> - 作业正在等待，一旦工作进程空闲就会被处理。<code>jobId</code> 包含进入此状态的作业的 ID。</td>
  </tr>
  <tr>
    <td><code>@OnQueueActive()</code></td><td><code>@OnGlobalQueueActive()</code></td><td><code>handler(job: Job)</code> - 作业 <code>job</code> 已开始。</td>
  </tr>
  <tr>
    <td><code>@OnQueueStalled()</code></td><td><code>@OnGlobalQueueStalled()</code></td><td><code>handler(job: Job)</code> - 作业 <code>job</code> 被标记为停滞。这对于调试崩溃或暂停事件循环的作业工作进程非常有用。</td>
  </tr>
  <tr>
    <td><code>@OnQueueProgress()</code></td><td><code>@OnGlobalQueueProgress()</code></td><td><code>handler(job: Job, progress: number)</code> - 作业 <code>job</code> 的进度已更新为值 <code>progress</code>。</td>
  </tr>
  <tr>
    <td><code>@OnQueueCompleted()</code></td><td><code>@OnGlobalQueueCompleted()</code></td><td><code>handler(job: Job, result: any)</code> 作业 <code>job</code> 成功完成，结果为 <code>result</code>。</td>
  </tr>
  <tr>
    <td><code>@OnQueueFailed()</code></td><td><code>@OnGlobalQueueFailed()</code></td><td><code>handler(job: Job, err: Error)</code> 作业 <code>job</code> 因原因 <code>err</code> 失败。</td>
  </tr>
  <tr>
    <td><code>@OnQueuePaused()</code></td><td><code>@OnGlobalQueuePaused()</code></td><td><code>handler()</code> 队列已暂停。</td>
  </tr>
  <tr>
    <td><code>@OnQueueResumed()</code></td><td><code>@OnGlobalQueueResumed()</code></td><td><code>handler(job: Job)</code> 队列已恢复。</td>
  </tr>
  <tr>
    <td><code>@OnQueueCleaned()</code></td><td><code>@OnGlobalQueueCleaned()</code></td><td><code>handler(jobs: Job[], type: string)</code> - 旧作业已从队列中清理。<code>jobs</code> 是清理作业的数组，<code>type</code> 是清理作业的类型。</td>
  </tr>
  <tr>
    <td><code>@OnQueueDrained()</code></td><td><code>@OnGlobalQueueDrained()</code></td><td><code>handler()</code> - 每当队列处理完所有等待的作业时触发（即使可能有一些尚未处理的延迟作业）。</td>
  </tr>
  <tr>
    <td><code>@OnQueueRemoved()</code></td><td><code>@OnGlobalQueueRemoved()</code></td><td><code>handler(job: Job)</code> - 作业 <code>job</code> 已成功移除。</td>
  </tr>
</table>

在监听全局事件时，方法签名可能与其本地对应项略有不同。具体来说，本地版本中接收 `job` 对象的任何方法签名，在全局版本中改为接收 `jobId`（`number`）。在这种情况下，要获取实际 `job` 对象的引用，请使用 `Queue#getJob` 方法。此调用应使用 `await`，因此处理程序应声明为 `async`。例如：

```typescript
@OnGlobalQueueCompleted()
async onGlobalCompleted(jobId: number, result: any) {
  const job = await this.immediateQueue.getJob(jobId);
  console.log('(Global) on completed: job ', job.id, ' -> result: ', result);
}
```

> info **提示** 要访问 `Queue` 对象（以进行 `getJob()` 调用），当然必须注入它。此外，队列必须在您注入它的模块中注册。

除了特定的事件监听器装饰器外，您还可以将通用的 `@OnQueueEvent()` 装饰器与 `BullQueueEvents` 或 `BullQueueGlobalEvents` 枚举结合使用。有关事件的更多信息，请参见 [这里](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#events)。

#### 队列管理

队列具有一个 API，允许您执行管理功能，如暂停和恢复、检索处于各种状态的作业计数等。您可以在 [这里](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue) 找到完整的队列 API。直接在 `Queue` 对象上调用这些方法中的任何一个，如下所示使用暂停/恢复示例。

使用 `pause()` 方法调用暂停队列。暂停的队列在恢复之前不会处理新作业，但当前正在处理的作业将继续执行直到完成。

```typescript
await audioQueue.pause();
```

要恢复暂停的队列，请使用 `resume()` 方法，如下所示：

```typescript
await audioQueue.resume();
```

#### 单独的进程

作业处理程序也可以在单独的（分叉的）进程中运行 ([源](https://github.com/OptimalBits/bull#separate-processes))。这有几个优点：

- 进程是沙箱化的，因此如果它崩溃不会影响工作进程。
- 您可以运行阻塞代码而不影响队列（作业不会挂起）。
- 更好地利用多核 CPU。
- 减少对 redis 的连接。

```ts
@@filename(app.module)
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bull';
import { join } from 'path';

@Module({
  imports: [
    BullModule.registerQueue({
      name: 'audio',
      processors: [join(__dirname, 'processor.js')],
    }),
  ],
})
export class AppModule {}
```

请注意，由于您的函数在分叉的进程中执行，因此依赖注入（和 IoC 容器）将不可用。这意味着您的处理器函数需要包含（或创建）所需的所有外部依赖项实例。

```ts
@@filename(processor)
import { Job, DoneCallback } from 'bull';

export default function (job: Job, cb: DoneCallback) {
  console.log(`[${process.pid}] ${JSON.stringify(job.data)}`);
  cb(null, 'It works');
}
```

#### 异步配置

您可能希望异步传递 `bull` 选项而不是静态传递。在这种情况下，请使用 `forRootAsync()` 方法，它提供了几种处理异步配置的方法。

一种方法是使用工厂函数：

```typescript
BullModule.forRootAsync({
  useFactory: () => ({
    redis: {
      host: 'localhost',
      port: 6379,
    },
  }),
});
```

我们的工厂行为像任何其他 [异步提供者](https://docs.nestjs.com/fundamentals/async-providers) 一样（例如，它可以是 `async` 的，并且可以通过 `inject` 注入依赖项）。

```typescript
BullModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    redis: {
      host: configService.get('QUEUE_HOST'),
      port: configService.get('QUEUE_PORT'),
    },
  }),
  inject: [ConfigService],
});
```

或者，您可以使用 `useClass` 语法：

```typescript
BullModule.forRootAsync({
  useClass: BullConfigService,
});
```

上面的构造将在 `BullModule` 内部实例化 `BullConfigService` 并使用它通过调用 `createSharedConfiguration()` 提供一个选项对象。请注意，这意味着 `BullConfigService` 必须实现 `SharedBullConfigurationFactory` 接口，如下所示：

```typescript
@Injectable()
class BullConfigService implements SharedBullConfigurationFactory {
  createSharedConfiguration(): BullModuleOptions {
    return {
      redis: {
        host: 'localhost',
        port: 6379,
      },
    };
  }
}
```

为了防止在 `BullModule` 内部创建 `BullConfigService` 并使用从其他模块导入的提供者，您可以使用 `useExisting` 语法。

```typescript
BullModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

这种构造与 `useClass` 的工作方式相同，只有一个关键区别 - `BullModule` 将查找导入的模块以重用现有的 `ConfigService` 而不是实例化新的。

同样，如果您希望异步传递队列选项，请使用 `registerQueueAsync()` 方法，只需记住在工厂函数外部指定 `name` 属性。

```typescript
BullModule.registerQueueAsync({
  name: 'audio',
  useFactory: () => ({
    redis: {
      host: 'localhost',
      port: 6379,
    },
  }),
});
```

#### 示例

一个可用的示例可在 [这里](https://github.com/nestjs/nest/tree/master/sample/26-queues) 找到。