### 队列

队列是一种强大的设计模式，可帮助你应对常见的应用扩展与性能挑战。以下是队列能够帮助你解决的一些问题示例：

- 平滑处理峰值。例如，如果用户可以在任意时间启动资源密集型任务，你可以将这些任务加入队列，而不是同步执行。随后，你可以让工作进程以受控的方式从队列中拉取任务。随着应用规模扩大，你可以轻松添加新的队列消费者来扩展后端任务处理能力。
- 拆分可能阻塞 Node.js 事件循环的庞大任务。例如，如果用户请求需要进行 CPU 密集型的工作（如音频转码），你可以将此任务委托给其他进程，从而让面向用户的进程保持响应。
- 为各种服务提供可靠的通信通道。例如，你可以在一个进程或服务中排队任务（作业），然后在另一个进程或服务中消费它们。你可以监听状态事件，从而在作业生命周期的完成、错误或其他状态变化时收到通知。当队列生产者或消费者失败时，它们的状态会被保留，节点重启后任务处理可以自动重启。

Nest 提供了 `@nestjs/bullmq` 包用于 BullMQ 集成，以及 `@nestjs/bull` 包用于 Bull 集成。这两个包都是在各自库之上的抽象/封装，由同一团队开发。Bull 目前处于维护模式，团队主要专注于修复缺陷；而 BullMQ 正在积极开发，拥有现代的 TypeScript 实现和不同的功能集。如果 Bull 满足你的需求，它仍然是一个可靠且久经考验的选择。Nest 的这些包让你能够以友好的方式将 BullMQ 或 Bull 队列轻松集成到 Nest 应用中。

BullMQ 和 Bull 都使用 [Redis](https://redis.io/) 来持久化作业数据，因此你需要在系统中安装 Redis。由于它们基于 Redis，你的队列架构可以完全分布式且与平台无关。例如，你可以让部分队列 <a href="techniques/queues#producers">生产者</a>、<a href="techniques/queues#consumers">消费者</a> 和 <a href="techniques/queues#event-listeners">监听器</a> 在 Nest 的一个（或多个）节点上运行，而其他生产者、消费者和监听器则在其他网络节点的其他 Node.js 平台上运行。

本章涵盖 `@nestjs/bullmq` 和 `@nestjs/bull` 包。我们还推荐阅读 [BullMQ](https://docs.bullmq.io/) 和 [Bull](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md) 的文档，以获取更多背景信息和具体实现细节。

#### BullMQ 安装

要开始使用 BullMQ，我们首先安装所需的依赖项。
```bash
$ npm install --save @nestjs/bullmq bullmq
```
安装过程完成后，我们可以将 `BullModule` 导入到根模块 `AppModule` 中。
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
`forRoot()` 方法用于注册一个 `bullmq` 包配置对象，该对象将被应用程序中注册的所有队列使用（除非另有指定）。供你参考，配置对象中包含以下部分属性：

- `connection: ConnectionOptions` - 用于配置 Redis 连接的选项。更多信息请参见 [Connections](https://docs.bullmq.io/guide/connections)。可选。
- `prefix: string` - 所有队列键的前缀。可选。
- `defaultJobOptions: JobOpts` - 控制新作业默认设置的选项。更多信息请参见 [JobOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd)。可选。
- `settings: AdvancedSettings` - 高级队列配置设置。通常不应更改。更多信息请参见 [AdvancedSettings](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue)。可选。
- `extraOptions` - 模块初始化的额外选项。请参见 [手动注册](https://docs.nestjs.com/techniques/queues#manual-registration)。

所有选项均为可选，可对队列行为进行精细控制。这些选项将直接传递给 BullMQ 的 `Queue` 构造函数。阅读更多关于这些及其他选项的信息，请访问[此处](https://api.docs.bullmq.io/interfaces/v4.QueueOptions.html)。

要注册队列，请导入 `BullModule.registerQueue()` 动态模块，如下所示：
```typescript
BullModule.registerQueue({
  name: 'audio',
});
```
> info **提示** 通过向 `registerQueue()` 方法传递多个用逗号分隔的配置对象，可以创建多个队列。

`registerQueue()` 方法用于实例化和/或注册队列。连接到同一个底层 Redis 数据库并使用相同凭据的模块和进程之间会共享这些队列。每个队列通过其 `name` 属性唯一标识。队列名称既用作注入令牌（用于将队列注入到控制器/提供者中），也用作装饰器的参数，以便将消费者类和监听器与队列关联。

你还可以如下所示，为特定队列覆盖部分预配置选项：
```typescript
BullModule.registerQueue({
  name: 'audio',
  connection: {
    port: 6380,
  },
});
```
BullMQ 还支持作业之间的父子关系。该功能使你可以创建任意深度的树形流程，其中作业充当树的节点。要了解更多信息，请查看[这里](https://docs.bullmq.io/guide/flows)。

要添加一个流程，你可以这样做：
```typescript
BullModule.registerFlowProducer({
  name: 'flowProducerName',
});
```
由于任务会被持久化到 Redis 中，每当一个特定的命名队列被实例化（例如应用启动或重启时），它都会尝试处理上一次会话中尚未完成的旧任务。

每个队列可以拥有一个或多个生产者、消费者和监听器。消费者会按照特定顺序从队列中取出任务：FIFO（默认）、LIFO 或根据优先级。关于如何控制队列的处理顺序，请参见<a href="techniques/queues#consumers">此处</a>。

<app-banner-enterprise></app-banner-enterprise>

#### 命名配置

如果你的队列需要连接多个不同的 Redis 实例，可以使用一种叫做**命名配置**的技术。该功能允许你在指定的键名下注册多个配置，然后在队列选项中引用这些键名。

例如，假设除了默认的 Redis 实例外，你还有另一个额外的 Redis 实例，并且应用中有少数几个队列会用到它，你可以按如下方式注册该实例的配置：
```typescript
BullModule.forRoot('alternative-config', {
  connection: {
    port: 6381,
  },
});
```
在上面的示例中，`'alternative-config'` 只是一个配置键（可以是任意字符串）。

有了这个设置后，你现在可以在 `registerQueue()` 的选项对象中指向该配置：
```typescript
BullModule.registerQueue({
  configKey: 'alternative-config',
  name: 'video',
});
```
#### 生产者

任务生产者负责将任务添加到队列中。生产者通常是应用程序服务（Nest [提供者](/providers)）。要将任务添加到队列，首先按如下方式将队列注入到服务中：
```typescript
import { Injectable } from '@nestjs/common';
import { Queue } from 'bullmq';
import { InjectQueue } from '@nestjs/bullmq';

@Injectable()
export class AudioService {
  constructor(@InjectQueue('audio') private audioQueue: Queue) {}
}
```
> info **提示** `@InjectQueue()` 装饰器通过名称来标识队列，该名称与 `registerQueue()` 方法调用中提供的名称一致（例如 `'audio'`）。

现在，通过调用队列的 `add()` 方法并传入一个用户定义的作业对象来添加作业。作业以可序列化的 JavaScript 对象表示（因为它们就是这样存储在 Redis 数据库中的）。你传入的作业结构是任意的；用它来表示你的作业对象的语义。你还需要给它一个名称。这允许你创建专门的 <a href="techniques/queues#consumers">消费者</a>，它们只处理具有特定名称的作业。
```typescript
const job = await this.audioQueue.add('转码', {
  foo: 'bar',
});
```
#### 作业选项

作业可以附带额外的选项。在 `Queue.add()` 方法中，将选项对象作为 `job` 参数之后的参数传入。部分作业选项属性如下：

- `priority`: `number` - 可选的优先级值。范围从 1（最高优先级）到 MAX_INT（最低优先级）。注意，使用优先级会对性能产生轻微影响，因此请谨慎使用。
- `delay`: `number` - 等待该作业可被执行的时间（毫秒）。注意，为了延迟的准确性，服务器和客户端的时钟应保持同步。
- `attempts`: `number` - 作业完成前尝试的总次数。
- `repeat`: `RepeatOpts` - 根据 cron 规范重复作业。参见 [RepeatOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd)。
- `backoff`: `number | BackoffOpts` - 作业失败时自动重试的回退设置。参见 [BackoffOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd)。
- `lifo`: `boolean` - 如果为 true，则将作业添加到队列的右端而非左端（默认为 false）。
- `jobId`: `number` | `string` - 覆盖作业 ID - 默认情况下，作业 ID 是一个唯一的整数，但你可以使用此设置覆盖它。如果使用此选项，你需要自行确保 jobId 唯一。如果尝试添加已存在 ID 的作业，则不会添加该作业。
- `removeOnComplete`: `boolean | number` - 如果为 true，则在作业成功完成后移除作业。数字指定保留的作业数量。默认行为是保留在已完成集合中。
- `removeOnFail`: `boolean | number` - 如果为 true，则在所有尝试失败后移除作业。数字指定保留的作业数量。默认行为是保留在失败集合中。
- `stackTraceLimit`: `number` - 限制堆栈跟踪中记录的堆栈跟踪行数。

以下是一些使用作业选项自定义作业的示例。

要延迟作业的开始，请使用 `delay` 配置属性。
```typescript
const job = await this.audioQueue.add(
  'transcode',
  {
    foo: 'bar',
  },
  { delay: 3000 }, // 延迟 3 秒
);
```
要将任务添加到队列的右端（以 **LIFO**（后进先出）的方式处理任务），请将配置对象的 `lifo` 属性设置为 `true`。
```typescript
const job = await this.audioQueue.add(
  'transcode',
  {
    foo: 'bar',
  },
  { lifo: true },
);
```
要设置作业的优先级，请使用 `priority` 属性。
```typescript
const job = await this.audioQueue.add(
  'transcode',
  {
    foo: 'bar',
  },
  { priority: 2 },
);
```
如需查看完整的选项列表，请查阅 API 文档 [此处](https://api.docs.bullmq.io/types/v4.JobsOptions.html) 和 [此处](https://api.docs.bullmq.io/interfaces/v4.BaseJobOptions.html)。

#### 消费者

消费者是一个 **类**，定义了用于处理队列中添加的作业、监听队列事件或同时执行这两项操作的方法。使用 `@Processor()` 装饰器声明消费者类，如下所示：
```typescript
import { Processor } from '@nestjs/bullmq';

@Processor('audio')
export class AudioConsumer {}
```
> info **提示** 消费者必须注册为 `providers`，这样 `@nestjs/bullmq` 包才能识别它们。

装饰器的字符串参数（例如 `'audio'`）是要与类方法关联的队列名称。
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
每当工作线程空闲且队列中有待处理的任务时，process 方法就会被调用。该处理函数只接收一个参数：`job` 对象。处理函数返回的值会存储在 job 对象中，稍后可以访问，例如在监听 completed 事件时。

`Job` 对象提供了多种方法来与其状态交互。例如，上面的代码使用了 `updateProgress()` 方法来更新任务的进度。完整的 `Job` 对象 API 参考请见[此处](https://api.docs.bullmq.io/classes/v4.Job.html)。

在旧版 Bull 中，你可以通过向 `@Process()` 装饰器传入一个 `name`，来指定某个任务处理函数**仅**处理特定类型（具有特定 `name`）的任务，如下所示。

> warning **警告** 这在 BullMQ 中无效，请继续阅读。
```typescript
@Process('转码')
async transcode(job: Job<unknown>) { ... }
```
由于这种行为会造成混淆，BullMQ 不再支持。取而代之的是，你需要使用 switch 语句，根据每个作业名称调用不同的服务或逻辑：
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
这在 BullMQ 文档的 [named processor](https://docs.bullmq.io/patterns/named-processor) 章节中有说明。

#### 请求作用域的消费者

当消费者被标记为请求作用域时（可在此处[了解](/fundamentals/injection-scopes#provider-scope)有关注入作用域的更多信息），将为每个任务专门创建该类的一个新实例。任务完成后，该实例将被垃圾回收。
```typescript
@Processor({
  name: 'audio',
  scope: Scope.REQUEST,
})
```
由于请求作用域的消费者类是动态实例化的，并且作用域限定在单个作业中，因此你可以通过构造函数使用标准方式注入 `JOB_REF`。
```typescript
constructor(@Inject(JOB_REF) jobRef: Job) {
  console.log(jobRef);
}
```
> info **提示** `JOB_REF` 令牌从 `@nestjs/bullmq` 包中导入。

#### 事件监听器

当队列和/或作业状态发生变化时，BullMQ 会生成一组有用的事件。这些事件可以在 Worker 级别通过 `@OnWorkerEvent(event)` 装饰器订阅，或者在队列级别使用专门的监听器类并通过 `@OnQueueEvent(event)` 装饰器订阅。

Worker 事件必须在 <a href="techniques/queues#consumers">消费者</a> 类中声明（即在一个使用 `@Processor()` 装饰器修饰的类中）。要监听事件，请使用 `@OnWorkerEvent(event)` 装饰器并指定要处理的事件。例如，要监听 `audio` 队列中作业进入 active 状态时触发的事件，请使用以下结构：
```typescript
import { Processor, Process, OnWorkerEvent } from '@nestjs/bullmq';
import { Job } from 'bullmq';

@Processor('audio')
export class AudioConsumer {
  @OnWorkerEvent('active')
  onActive(job: Job) {
    console.log(
      `正在处理作业 ${job.id}，类型为 ${job.name}，数据为 ${job.data}...`,
    );
  }

  // ...
}
```
你可以[在这里](https://api.docs.bullmq.io/interfaces/v4.WorkerListener.html)查看完整的事件列表及其作为 WorkerListener 属性的参数。

QueueEvent 监听器必须使用 `@QueueEventsListener(queue)` 装饰器，并继承由 `@nestjs/bullmq` 提供的 `QueueEventsHost` 类。要监听某个事件，请使用 `@OnQueueEvent(event)` 装饰器并指定要处理的事件。例如，要监听 `audio` 队列中作业进入 active 状态时触发的事件，请使用以下结构：
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
    console.log(`正在处理任务 ${job.jobId}...`);
  }

  // ...
}
```
> info **提示** QueueEvent 监听器必须注册为 `providers`，这样 `@nestjs/bullmq` 包才能发现它们。

你可以在[这里](https://api.docs.bullmq.io/interfaces/v4.QueueEventsListener.html)查看完整的事件列表及其作为属性的参数。

#### 队列管理

队列提供了一套 API，允许你执行诸如暂停与恢复、获取各种状态下作业数量等管理功能，以及更多操作。你可以在[这里](https://api.docs.bullmq.io/classes/v4.Queue.html)找到完整的队列 API。直接在 `Queue` 对象上调用这些方法，如下面的暂停/恢复示例所示。

通过调用 `pause()` 方法暂停队列。被暂停的队列在恢复之前不会处理新的作业，但当前正在处理的作业会继续执行直至完成。
```typescript
await audioQueue.pause();
```
要恢复已暂停的队列，请使用 `resume()` 方法，如下所示：
```typescript
await audioQueue.resume();
```
#### 独立进程

作业处理器也可以在独立的（fork 出的）进程中运行（[来源](https://docs.bullmq.io/guide/workers/sandboxed-processors)）。这样做有以下几个优点：

- 进程被沙箱隔离，因此即使崩溃也不会影响工作线程。
- 可以运行阻塞代码而不会影响队列（作业不会卡住）。
- 更好地利用多核 CPU。
- 减少与 Redis 的连接数。
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
> warning **警告** 请注意，由于函数在 fork 进程中执行，依赖注入（以及 IoC 容器）将不可用。这意味着你的处理器函数需要包含（或创建）所有必需的外部依赖实例。

#### 异步配置

你可能希望以异步方式而非静态方式传递 `bullmq` 选项。此时请使用 `forRootAsync()` 方法，它提供了多种处理异步配置的方式。同样，如果想以异步方式传递队列选项，请使用 `registerQueueAsync()` 方法。

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
我们的工厂与任何其他[异步提供者](https://docs.nestjs.com/fundamentals/async-providers)行为一致（例如，它可以是 `async` 的，并且能够通过 `inject` 注入依赖）。
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
或者，你也可以使用 `useClass` 语法：
```typescript
BullModule.forRootAsync({
  useClass: BullConfigService,
});
```
上述构造过程将在 `BullModule` 内部实例化 `BullConfigService`，并通过调用 `createSharedConfiguration()` 来提供一个配置对象。请注意，这意味着 `BullConfigService` 必须实现 `SharedBullConfigurationFactory` 接口，如下所示：
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
为了防止在 `BullModule` 内部创建 `BullConfigService`，并改为使用从其他模块导入的提供者，你可以使用 `useExisting` 语法。
```typescript
BullModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```
此构造方式与 `useClass` 完全相同，但有一个关键区别：`BullModule` 会查找已导入的模块，以复用现有的 `ConfigService`，而不是实例化一个新的。

同样，如果你想异步传递队列选项，请使用 `registerQueueAsync()` 方法，只需注意将 `name` 属性放在工厂函数之外。
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

默认情况下，`BullModule` 会在 `onModuleInit` 生命周期钩子中自动注册 BullMQ 组件（队列、处理器以及事件监听器服务）。然而，在某些场景下，这种行为可能并不合适。若要阻止自动注册，请在 `BullModule` 中启用 `manualRegistration`，如下所示：
```typescript
BullModule.forRoot({
  extraOptions: {
    manualRegistration: true,
  },
});
```
要手动注册这些组件，请注入 `BullRegistrar` 并在 `OnModuleInit` 或 `OnApplicationBootstrap` 中调用 `register` 函数。
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
除非你调用 `BullRegistrar#register` 函数，否则任何 BullMQ 组件都无法正常工作——也就是说不会有任何作业被处理。

#### Bull 安装

> 警告 **注意** 如果你决定使用 BullMQ，请跳过本节及后续章节。

要开始使用 Bull，我们首先安装所需的依赖项。
```bash
$ npm install --save @nestjs/bull bull
```
安装过程完成后，我们就可以将 `BullModule` 导入到根 `AppModule` 中。
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
`forRoot()` 方法用于注册一个 `bull` 包的配置对象，该对象将被应用中所有队列使用（除非另行指定）。配置对象包含以下属性：

- `limiter: RateLimiter` - 控制队列作业处理速率的选项。更多信息请参见 [RateLimiter](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue)。可选。
- `redis: RedisOpts` - 配置 Redis 连接的选项。更多信息请参见 [RedisOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue)。可选。
- `prefix: string` - 所有队列键的前缀。可选。
- `defaultJobOptions: JobOpts` - 控制新作业默认设置的选项。更多信息请参见 [JobOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd)。可选。**注意：如果通过 FlowProducer 调度作业，这些设置不会生效。解释见 [bullmq#1034](https://github.com/taskforcesh/bullmq/issues/1034)。**
- `settings: AdvancedSettings` - 高级队列配置设置。通常不应更改。更多信息请参见 [AdvancedSettings](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue)。可选。

所有选项均为可选，可对队列行为进行精细控制。这些选项会直接传递给 Bull 的 `Queue` 构造函数。阅读更多关于这些选项的信息[点击这里](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue)。

要注册一个队列，请导入 `BullModule.registerQueue()` 动态模块，如下所示：
```typescript
BullModule.registerQueue({
  name: 'audio',
});
```
> info **提示** 通过向 `registerQueue()` 方法传递多个用逗号分隔的配置对象，可以创建多个队列。

`registerQueue()` 方法用于实例化和/或注册队列。队列在连接到同一底层 Redis 数据库并使用相同凭据的模块和进程之间共享。每个队列通过其 `name` 属性保持唯一。队列名称既用作注入令牌（用于将队列注入到控制器/提供者中），也用作装饰器的参数，以将消费者类和监听器与队列关联起来。

你还可以为特定队列覆盖某些预配置的选项，如下所示：
```typescript
BullModule.registerQueue({
  name: 'audio',
  redis: {
    port: 6380,
  },
});
```
由于任务会持久化到 Redis 中，每次实例化某个特定命名队列时（例如应用启动/重启时），都会尝试处理上一次未完成会话遗留下来的旧任务。

每个队列可以拥有一个或多个生产者、消费者和监听器。消费者会按照特定顺序从队列中取出任务：FIFO（默认）、LIFO 或按优先级排序。关于如何控制队列处理顺序的讨论请见<a href="techniques/queues#consumers">此处</a>。

<app-banner-enterprise></app-banner-enterprise>

#### 命名配置

如果你的队列需要连接多个 Redis 实例，可以使用一种称为**命名配置**的技术。该功能允许你将多个配置注册到指定的键名下，然后在队列选项中通过键名引用它们。

例如，假设除了默认的 Redis 实例外，你还有另一个 Redis 实例，且应用中的少数队列会使用它，那么你可以按如下方式注册该实例的配置：
```typescript
BullModule.forRoot('alternative-config', {
  redis: {
    port: 6381,
  },
});
```
在上面的示例中，`'alternative-config'` 只是一个配置键（可以是任意字符串）。

有了这一步之后，你现在就可以在 `registerQueue()` 的选项对象中指向这个配置：
```typescript
BullModule.registerQueue({
  configKey: 'alternative-config',
  name: 'video',
});
```
#### 生产者

作业生产者将作业添加到队列中。生产者通常是应用程序服务（Nest [提供者](/providers)）。要将作业添加到队列，首先按以下方式将队列注入到服务中：
```typescript
import { Injectable } from '@nestjs/common';
import { Queue } from 'bull';
import { InjectQueue } from '@nestjs/bull';

@Injectable()
export class AudioService {
  constructor(@InjectQueue('audio') private audioQueue: Queue) {}
}
```
> info **提示** `@InjectQueue()` 装饰器通过队列名称来标识队列，该名称在 `registerQueue()` 方法调用中提供（例如 `'audio'`）。

现在，通过调用队列的 `add()` 方法添加一个任务，并传入一个用户定义的任务对象。任务被表示为可序列化的 JavaScript 对象（因为它们就是这样存储在 Redis 数据库中的）。你传入的任务结构是任意的；用它来表示你的任务对象的语义。
```typescript
const job = await this.audioQueue.add({
  foo: 'bar',
});
```
#### 命名任务

任务可以拥有唯一的名称。这样你就可以创建专门的 <a href="techniques/queues#consumers">消费者</a>，仅处理具有指定名称的任务。
```typescript
const job = await this.audioQueue.add('转码', {
  foo: 'bar',
});
```
> 警告 **警告** 使用命名任务时，必须为队列中添加的每一个唯一名称创建对应的处理器，否则队列会报错提示缺少该任务的处理器。有关消费命名任务的更多信息，请参见<a href="techniques/queues#consumers">此处</a>。

#### 任务选项

任务可以附带额外的选项。在调用 `Queue.add()` 方法时，在 `job` 参数后传入一个选项对象即可。任务选项的属性如下：

- `priority`: `number` - 可选的优先级数值。范围从 1（最高优先级）到 MAX_INT（最低优先级）。注意使用优先级会对性能有轻微影响，请谨慎使用。
- `delay`: `number` - 延迟处理该任务的时间（毫秒）。注意，为了获得准确的延迟，服务器和客户端的时钟应同步。
- `attempts`: `number` - 任务失败时的总重试次数。
- `repeat`: `RepeatOpts` - 根据 cron 表达式重复执行任务。详见 [RepeatOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd)。
- `backoff`: `number | BackoffOpts` - 任务失败时自动重试的回退策略。详见 [BackoffOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd)。
- `lifo`: `boolean` - 若为 true，则将任务插入队列右侧而非左侧（默认为 false）。
- `timeout`: `number` - 任务超时时间（毫秒），超时后任务将失败并抛出超时错误。
- `jobId`: `number` | `string` - 覆盖任务 ID。默认情况下，任务 ID 是一个唯一的整数，但可通过此选项自定义。若使用此选项，请确保 jobId 唯一。如果尝试添加已存在的 jobId，该任务将不会被加入队列。
- `removeOnComplete`: `boolean | number` - 若为 true，任务成功完成后将被移除。若传入数字，则保留指定数量的已完成任务。默认行为是保留在已完成集合中。
- `removeOnFail`: `boolean | number` - 若为 true，任务在所有尝试失败后将被移除。若传入数字，则保留指定数量的失败任务。默认行为是保留在失败集合中。
- `stackTraceLimit`: `number` - 限制堆栈跟踪中记录的堆栈行数。

以下是使用任务选项自定义任务的一些示例。

如需延迟任务启动，可使用 `delay` 配置属性。
```typescript
const job = await this.audioQueue.add(
  {
    foo: 'bar',
  },
  { delay: 3000 }, // 延迟 3 秒
);
```
要将任务添加到队列的右端（以**LIFO**（后进先出）的方式处理任务），请将配置对象的 `lifo` 属性设置为 `true`。
```typescript
const job = await this.audioQueue.add(
  {
    foo: 'bar',
  },
  { lifo: true },
);
```
要优先处理某个作业，请使用 `priority` 属性。
```typescript
const job = await this.audioQueue.add(
  {
    foo: 'bar',
  },
  { priority: 2 },
);
```
#### 消费者

消费者是一个**类**，它定义了用于处理加入队列的任务、监听队列事件或两者兼有的方法。使用 `@Processor()` 装饰器声明一个消费者类，如下所示：
```typescript
import { Processor } from '@nestjs/bull';

@Processor('audio')
export class AudioConsumer {}
```
> info **提示** 消费者必须注册为 `providers`，这样 `@nestjs/bull` 包才能识别它们。

装饰器的字符串参数（例如 `'audio'`）是要与类方法关联的队列名称。

在消费者类中，通过使用 `@Process()` 装饰器来装饰处理程序方法，声明作业处理程序。
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
每当 worker 空闲且队列中有待处理的任务时，被装饰的方法（例如 `transcode()`）就会被调用。该处理器方法只接收一个参数，即 `job` 对象。处理器方法返回的值会保存在 job 对象中，并可在后续访问，例如在监听任务完成事件时。

`Job` 对象提供了多种方法，让你可以与其状态进行交互。例如，上面的代码使用了 `progress()` 方法来更新任务的进度。完整的 `Job` 对象 API 参考请参见[这里](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#job)。

你可以通过向 `@Process()` 装饰器传入一个 `name` 来指定某个任务处理器方法**仅**处理特定类型的任务（即具有特定 `name` 的任务），如下所示。在一个消费者类中，你可以拥有多个 `@Process()` 处理器，分别对应不同的任务类型（`name`）。当使用命名任务时，请确保为每个名称都提供对应的处理器。
```typescript
@Process('转码')
async transcode(job: Job<unknown>) { ... }
```
> warning **警告** 当为同一个队列定义多个消费者时，`@Process({{ '{' }} concurrency: 1 {{ '}' }})` 中的 `concurrency` 选项将不会生效。最小并发数将等于定义的消费者数量。即使 `@Process()` 处理程序使用不同的 `name` 来处理命名任务，这一规则也同样适用。

#### 请求作用域消费者

当消费者被标记为请求作用域时（点击[此处](/fundamentals/injection-scopes#provider-scope)了解更多关于注入作用域的信息），将为每个任务专门创建该类的一个新实例。该实例将在任务完成后被垃圾回收。
```typescript
@Processor({
  name: 'audio',
  scope: Scope.REQUEST,
})
```
由于请求作用域的消费者类是动态实例化的，并且作用域限定在单个任务中，因此你可以通过构造函数使用标准方式注入 `JOB_REF`。
```typescript
constructor(@Inject(JOB_REF) jobRef: Job) {
  console.log(jobRef);
}
```
> info **提示** `JOB_REF` 标记是从 `@nestjs/bull` 包中导入的。

#### 事件监听器

当队列和/或作业状态发生变化时，Bull 会生成一组有用的事件。Nest 提供了一组装饰器，允许订阅一组核心的标准事件。这些装饰器从 `@nestjs/bull` 包中导出。

事件监听器必须在 <a href="techniques/queues#consumers">消费者</a> 类中声明（即，在一个用 `@Processor()` 装饰器装饰的类中）。要监听某个事件，请使用下表中的某个装饰器来为该事件声明一个处理程序。例如，要监听当作业在 `audio` 队列中进入活动状态时发出的事件，请使用以下结构：
```typescript
import { Processor, Process, OnQueueActive } from '@nestjs/bull';
import { Job } from 'bull';

@Processor('audio')
export class AudioConsumer {

  @OnQueueActive()
  onActive(job: Job) {
    console.log(
      `正在处理任务 ${job.id}，类型为 ${job.name}，数据为 ${job.data}...`,
    );
  }
  ...
```
由于 Bull 运行在分布式（多节点）环境中，它引入了“事件本地性”的概念。该概念指出，事件可能完全在单个进程内触发，也可能由不同进程在共享队列上触发。**本地**事件指的是在本地进程的队列上发生动作或状态变更时产生的事件。换句话说，当事件的生产者和消费者都位于同一进程内时，队列上发生的所有事件都是本地的。

当队列在多个进程之间共享时，就可能出现**全局**事件。为了让一个进程中的监听器能够接收到由另一个进程触发的事件通知，它必须注册全局事件。

每当对应的事件被触发时，事件处理器就会被调用。处理器以下表所示的签名被调用，提供与事件相关的信息。下面我们讨论本地与全局事件处理器签名的一个关键区别。

<table>
  <tr>
    <th>本地事件监听器</th>
    <th>全局事件监听器</th>
    <th>处理器方法签名 / 触发时机</th>
  </tr>
  <tr>
    <td><code>@OnQueueError()</code></td><td><code>@OnGlobalQueueError()</code></td><td><code>handler(error: Error)</code> - 发生错误。<code>error</code> 包含触发错误的详细信息。</td>
  </tr>
  <tr>
    <td><code>@OnQueueWaiting()</code></td><td><code>@OnGlobalQueueWaiting()</code></td><td><code>handler(jobId: number | string)</code> - 某个作业正在等待，一旦有空闲 worker 就会立即处理。<code>jobId</code> 包含进入此状态的作业 ID。</td>
  </tr>
  <tr>
    <td><code>@OnQueueActive()</code></td><td><code>@OnGlobalQueueActive()</code></td><td><code>handler(job: Job)</code> - 作业 <code>job</code> 已开始。</td>
  </tr>
  <tr>
    <td><code>@OnQueueStalled()</code></td><td><code>@OnGlobalQueueStalled()</code></td><td><code>handler(job: Job)</code> - 作业 <code>job</code> 被标记为停滞。这对调试崩溃或阻塞事件循环的 job worker 很有帮助。</td>
  </tr>
  <tr>
    <td><code>@OnQueueProgress()</code></td><td><code>@OnGlobalQueueProgress()</code></td><td><code>handler(job: Job, progress: number)</code> - 作业 <code>job</code> 的进度已更新为值 <code>progress</code>。</td>
  </tr>
  <tr>
    <td><code>@OnQueueCompleted()</code></td><td><code>@OnGlobalQueueCompleted()</code></td><td><code>handler(job: Job, result: any)</code> 作业 <code>job</code> 成功完成，结果为 <code>result</code>。</td>
  </tr>
  <tr>
    <td><code>@OnQueueFailed()</code></td><td><code>@OnGlobalQueueFailed()</code></td><td><code>handler(job: Job, err: Error)</code> 作业 <code>job</code> 失败，原因为 <code>err</code>。</td>
  </tr>
  <tr>
    <td><code>@OnQueuePaused()</code></td><td><code>@OnGlobalQueuePaused()</code></td><td><code>handler()</code> 队列已暂停。</td>
  </tr>
  <tr>
    <td><code>@OnQueueResumed()</code></td><td><code>@OnGlobalQueueResumed()</code></td><td><code>handler(job: Job)</code> 队列已恢复。</td>
  </tr>
  <tr>
    <td><code>@OnQueueCleaned()</code></td><td><code>@OnGlobalQueueCleaned()</code></td><td><code>handler(jobs: Job[], type: string)</code> 旧作业已从队列中清理。<code>jobs</code> 是被清理的作业数组，<code>type</code> 是被清理的作业类型。</td>
  </tr>
  <tr>
    <td><code>@OnQueueDrained()</code></td><td><code>@OnGlobalQueueDrained()</code></td><td><code>handler()</code> 当队列已处理完所有等待中的作业时触发（即使仍有延迟作业尚未处理）。</td>
  </tr>
  <tr>
    <td><code>@OnQueueRemoved()</code></td><td><code>@OnGlobalQueueRemoved()</code></td><td><code>handler(job: Job)</code> 作业 <code>job</code> 已成功移除。</td>
  </tr>
</table>

在监听全局事件时，方法签名可能与本地版本略有不同。具体来说，本地版本中接收 `job` 对象的签名，在全局版本中改为接收 `jobId`（`number`）。若要在这种情况下获取实际的 `job` 对象，请使用 `Queue#getJob` 方法。该调用需要被 `await`，因此处理器应声明为 `async`。例如：
```typescript
@OnGlobalQueueCompleted()
async onGlobalCompleted(jobId: number, result: any) {
  const job = await this.immediateQueue.getJob(jobId);
  console.log('(全局) 任务完成：任务 ', job.id, ' -> 结果: ', result);
}
```
> info **提示** 要访问 `Queue` 对象（例如调用 `getJob()`），你必须先将其注入。此外，该 Queue 必须在你要注入它的模块中注册。

除了特定的事件监听器装饰器外，你还可以结合 `BullQueueEvents` 或 `BullQueueGlobalEvents` 枚举，使用通用的 `@OnQueueEvent()` 装饰器。阅读更多关于事件的信息[点击这里](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#events)。

#### 队列管理

队列提供了一个 API，允许你执行诸如暂停和恢复、获取各种状态下作业数量等管理功能。你可以在[此处](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue)找到完整的队列 API。直接在 `Queue` 对象上调用这些方法，如下面的暂停/恢复示例所示。

通过调用 `pause()` 方法暂停队列。被暂停的队列不会处理新作业，直到被恢复；但当前正在处理的作业会继续执行，直到完成。
```typescript
await audioQueue.pause();
```
要恢复已暂停的队列，请使用 `resume()` 方法，如下所示：
```typescript
await audioQueue.resume();
```
#### 独立进程

任务处理器也可以在单独的（fork 出的）进程中运行（[来源](https://github.com/OptimalBits/bull#separate-processes)）。这样做有以下几个优点：

- 进程被沙箱隔离，因此即使崩溃也不会影响工作进程。
- 可以运行阻塞代码而不会影响队列（任务不会停滞）。
- 更好地利用多核 CPU。
- 与 Redis 的连接更少。
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
请注意，由于你的函数在 fork 出的进程中执行，依赖注入（以及 IoC 容器）将不可用。这意味着你的处理器函数需要包含（或创建）其所需的所有外部依赖实例。
```ts
@@filename(processor)
import { Job, DoneCallback } from 'bull';

export default function (job: Job, cb: DoneCallback) {
  console.log(`[${process.pid}] ${JSON.stringify(job.data)}`);
  cb(null, 'It works');
}
```
#### 异步配置

你可能希望以异步方式而非静态方式传递 `bull` 选项。在这种情况下，请使用 `forRootAsync()` 方法，它提供了多种处理异步配置的方式。

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
我们的工厂表现得像任何其他[异步提供者](https://docs.nestjs.com/fundamentals/async-providers)一样（例如，它可以是 `async`，并且能够通过 `inject` 注入依赖）。
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
或者，你也可以使用 `useClass` 语法：
```typescript
BullModule.forRootAsync({
  useClass: BullConfigService,
});
```
上述构造过程将在 `BullModule` 内部实例化 `BullConfigService`，并通过调用 `createSharedConfiguration()` 来提供一个选项对象。请注意，这意味着 `BullConfigService` 必须实现 `SharedBullConfigurationFactory` 接口，如下所示：
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
为了防止在 `BullModule` 内部创建 `BullConfigService`，并使用从另一个模块导入的提供程序，你可以使用 `useExisting` 语法。
```typescript
BullModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```
此构造与 `useClass` 的工作方式相同，但有一个关键区别：`BullModule` 会查找已导入的模块，以复用现有的 `ConfigService`，而不是实例化一个新的。

同样，如果你想异步传递队列选项，请使用 `registerQueueAsync()` 方法，只需记住在工厂函数外部指定 `name` 属性。
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

一个可用的示例可在[此处](https://github.com/nestjs/nest/tree/master/sample/26-queues)查看。