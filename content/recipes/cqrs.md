### CQRS

简单 [CRUD](https://zh.wikipedia.org/wiki/增删改查)（创建、读取、更新和删除）应用程序的流程可以描述如下：

1. 控制器层处理 HTTP 请求，并将任务委托给服务层。
2. 服务层包含大部分业务逻辑。
3. 服务使用仓库 / 数据访问对象（DAO）来更改 / 持久化实体。
4. 实体作为值的容器，包含设置器和获取器。

虽然该模式通常足以满足小型和中型应用程序的需求，但对于更大、更复杂的应用程序来说，它可能不是最佳选择。在这种情况下，**CQRS**（命令和查询职责分离）模型可能更加合适且可扩展（取决于应用程序的需求）。该模型的优点包括：

- **职责分离**。该模型将读操作和写操作分别放入不同的模型中。
- **可扩展性**。读操作和写操作可以独立扩展。
- **灵活性**。该模型允许为读操作和写操作使用不同的数据存储。
- **性能** 。该模型允许使用为读操作和写操作优化的不同数据存储。

为了支持该模型，Nest 提供了一个轻量级的 [CQRS 模块](https://github.com/nestjs/cqrs)。本章将介绍如何使用它。

#### 安装

首先安装所需的包：

```bash
$ npm install --save @nestjs/cqrs
```

安装完成后，导航到应用程序的根模块（通常是 `AppModule`），并导入 `CqrsModule.forRoot()`：

```typescript
import { Module } from '@nestjs/common';
import { CqrsModule } from '@nestjs/cqrs';

@Module({
  imports: [CqrsModule.forRoot()],
})
export class AppModule {}
```

该模块接受一个可选的配置对象。以下是可用的选项：

| 属性                     | 描述                                                                                                                  | 默认值                           |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| `commandPublisher`       | 负责将命令分派到系统的发布者。                                                                                         | `DefaultCommandPubSub`           |
| `eventPublisher`         | 用于发布事件的发布者，允许广播或处理事件。                                                                            | `DefaultPubSub`                  |
| `queryPublisher`         | 用于发布查询的发布者，可以触发数据检索操作。                                                                          | `DefaultQueryPubSub`             |
| `unhandledExceptionPublisher` | 负责处理未处理异常的发布者，确保异常被跟踪和报告。                                                                  | `DefaultUnhandledExceptionPubSub`|
| `eventIdProvider`        | 提供唯一事件 ID 的服务，通过生成或从事件实例中检索。                                                                  | `DefaultEventIdProvider`         |
| `rethrowUnhandled`       | 确定在处理未处理异常后是否重新抛出，有助于调试和错误管理。                                                            | `false`                          |

#### 命令

命令用于更改应用程序状态。它们应该是基于任务的，而不是以数据为中心的。当命令被分派时，它由相应的 **命令处理器** 处理。处理器负责更新应用程序状态。

```typescript
@@filename(heroes-game.service)
@Injectable()
export class HeroesGameService {
  constructor(private commandBus: CommandBus) {}

  async killDragon(heroId: string, killDragonDto: KillDragonDto) {
    return this.commandBus.execute(
      new KillDragonCommand(heroId, killDragonDto.dragonId)
    );
  }
}
@@switch
@Injectable()
@Dependencies(CommandBus)
export class HeroesGameService {
  constructor(commandBus) {
    this.commandBus = commandBus;
  }

  async killDragon(heroId, killDragonDto) {
    return this.commandBus.execute(
      new KillDragonCommand(heroId, killDragonDto.dragonId)
    );
  }
}
```

在上面的代码片段中，我们实例化了 `KillDragonCommand` 类并将其传递给 `CommandBus` 的 `execute()` 方法。这是展示的命令类：

```typescript
@@filename(kill-dragon.command)
export class KillDragonCommand extends Command<{
  actionId: string // 该类型表示命令执行的结果
}> {
  constructor(
    public readonly heroId: string,
    public readonly dragonId: string,
  ) {
    super();
  }
}
@@switch
export class KillDragonCommand extends Command {
  constructor(heroId, dragonId) {
    this.heroId = heroId;
    this.dragonId = dragonId;
  }
}
```

如你所见，`KillDragonCommand` 类继承了 `Command` 类。`Command` 是一个简单的工具类，从 `@nestjs/cqrs` 包中导出，它允许你定义命令的返回类型。在这种情况下，返回类型是一个包含 `actionId` 属性的对象。现在，每当分派 `KillDragonCommand` 命令时，`CommandBus#execute()` 方法的返回类型将被推断为 `Promise<{{ '{' }} actionId: string {{ '}' }}>`。当你希望从命令处理器返回某些数据时，这非常有用。

> info **提示** 继承 `Command` 类是可选的。只有当你想定义命令的返回类型时才需要这样做。

`CommandBus` 表示一个命令流。它负责将命令分派到适当的处理器。`execute()` 方法返回一个 Promise，它解析为处理器返回的值。

让我们为 `KillDragonCommand` 命令创建一个处理器。

```typescript
@@filename(kill-dragon.handler)
@CommandHandler(KillDragonCommand)
export class KillDragonHandler implements ICommandHandler<KillDragonCommand> {
  constructor(private repository: HeroesRepository) {}

  async execute(command: KillDragonCommand) {
    const { heroId, dragonId } = command;
    const hero = this.repository.findOneById(+heroId);

    hero.killEnemy(dragonId);
    await this.repository.persist(hero);

    // "ICommandHandler<KillDragonCommand>" 强制你返回一个与命令返回类型匹配的值
    return {
      actionId: crypto.randomUUID(), // 该值将返回给调用者
    }
  }
}
@@switch
@CommandHandler(KillDragonCommand)
@Dependencies(HeroesRepository)
export class KillDragonHandler {
  constructor(repository) {
    this.repository = repository;
  }

  async execute(command) {
    const { heroId, dragonId } = command;
    const hero = this.repository.findOneById(+heroId);

    hero.killEnemy(dragonId);
    await this.repository.persist(hero);

    // "ICommandHandler<KillDragonCommand>" 强制你返回一个与命令返回类型匹配的值
    return {
      actionId: crypto.randomUUID(), // 该值将返回给调用者
    }
  }
}
```

此处理器从仓库中检索 `Hero` 实体，调用 `killEnemy()` 方法，然后持久化更改。`KillDragonHandler` 类实现了 `ICommandHandler` 接口，该接口要求实现 `execute()` 方法。`execute()` 方法接收命令对象作为参数。

请注意，`ICommandHandler<KillDragonCommand>` 强制你返回一个与命令返回类型匹配的值。在这种情况下，返回类型是一个包含 `actionId` 属性的对象。这仅适用于继承自 `Command` 类的命令。否则，你可以返回任何内容。

最后，请确保在模块中将 `KillDragonHandler` 注册为提供者：

```typescript
providers: [KillDragonHandler];
```

#### 查询

查询用于从应用程序状态中检索数据。它们应该是以数据为中心的，而不是基于任务的。当查询被分派时，它由相应的 **查询处理器** 处理。处理器负责检索数据。

`QueryBus` 遵循与 `CommandBus` 相同的模式。查询处理器应实现 `IQueryHandler` 接口并使用 `@QueryHandler()` 装饰器进行注解。请参见以下示例：

```typescript
export class GetHeroQuery extends Query<Hero> {
  constructor(public readonly heroId: string) {}
}
```

类似于 `Command` 类，`Query` 类是一个简单的工具类，从 `@nestjs/cqrs` 包中导出，它允许你定义查询的返回类型。在这种情况下，返回类型是一个 `Hero` 对象。现在，每当分派 `GetHeroQuery` 查询时，`QueryBus#execute()` 方法的返回类型将被推断为 `Promise<Hero>`。

要检索英雄，我们需要创建一个查询处理器：

```typescript
@@filename(get-hero.handler)
@QueryHandler(GetHeroQuery)
export class GetHeroHandler implements IQueryHandler<GetHeroQuery> {
  constructor(private repository: HeroesRepository) {}

  async execute(query: GetHeroQuery) {
    return this.repository.findOneById(query.heroId);
  }
}
@@switch
@QueryHandler(GetHeroQuery)
@Dependencies(HeroesRepository)
export class GetHeroHandler {
  constructor(repository) {
    this.repository = repository;
  }

  async execute(query) {
    return this.repository.findOneById(query.heroId);
  }
}
```

`GetHeroHandler` 类实现了 `IQueryHandler` 接口，该接口要求实现 `execute()` 方法。`execute()` 方法接收查询对象作为参数，并必须返回与查询返回类型匹配的数据（在本例中为一个 `Hero` 对象）。

最后，请确保在模块中将 `GetHeroHandler` 注册为提供者：

```typescript
providers: [GetHeroHandler];
```

现在，要分派查询，请使用 `QueryBus`：

```typescript
const hero = await this.queryBus.execute(new GetHeroQuery(heroId)); // "hero" 将自动推断为 "Hero" 类型
```

#### 事件

事件用于通知应用程序其他部分应用程序状态的变化。它们由 **模型** 或直接使用 `EventBus` 分派。当事件被分派时，它由相应的 **事件处理器** 处理。处理器可以例如更新读模型。

为了演示目的，让我们创建一个事件类：

```typescript
@@filename(hero-killed-dragon.event)
export class HeroKilledDragonEvent {
  constructor(
    public readonly heroId: string,
    public readonly dragonId: string,
  ) {}
}
@@switch
export class HeroKilledDragonEvent {
  constructor(heroId, dragonId) {
    this.heroId = heroId;
    this.dragonId = dragonId;
  }
}
```

现在，虽然可以使用 `EventBus.publish()` 方法直接分派事件，我们也可以从模型中分派它们。让我们更新 `Hero` 模型，在调用 `killEnemy()` 方法时分派 `HeroKilledDragonEvent` 事件。

```typescript
@@filename(hero.model)
export class Hero extends AggregateRoot {
  constructor(private id: string) {
    super();
  }

  killEnemy(enemyId: string) {
    // 业务逻辑
    this.apply(new HeroKilledDragonEvent(this.id, enemyId));
  }
}
@@switch
export class Hero extends AggregateRoot {
  constructor(id) {
    super();
    this.id = id;
  }

  killEnemy(enemyId) {
    // 业务逻辑
    this.apply(new HeroKilledDragonEvent(this.id, enemyId));
  }
}
```

`apply()` 方法用于分派事件。它接受一个事件对象作为参数。然而，由于我们的模型不知道 `EventBus`，我们需要将它与模型关联起来。我们可以使用 `EventPublisher` 类来实现这一点。

```typescript
@@filename(kill-dragon.handler)
@CommandHandler(KillDragonCommand)
export class KillDragonHandler implements ICommandHandler<KillDragonCommand> {
  constructor(
    private repository: HeroesRepository,
    private publisher: EventPublisher,
  ) {}

  async execute(command: KillDragonCommand) {
    const { heroId, dragonId } = command;
    const hero = this.publisher.mergeObjectContext(
      await this.repository.findOneById(+heroId),
    );
    hero.killEnemy(dragonId);
    hero.commit();
  }
}
@@switch
@CommandHandler(KillDragonCommand)
@Dependencies(HeroesRepository, EventPublisher)
export class KillDragonHandler {
  constructor(repository, publisher) {
    this.repository = repository;
    this.publisher = publisher;
  }

  async execute(command) {
    const { heroId, dragonId } = command;
    const hero = this.publisher.mergeObjectContext(
      await this.repository.findOneById(+heroId),
    );
    hero.killEnemy(dragonId);
    hero.commit();
  }
}
```

`EventPublisher#mergeObjectContext` 方法将事件发布者合并到提供的对象中，这意味着该对象现在能够向事件流发布事件。

请注意，在此示例中我们也在模型上调用了 `commit()` 方法。该方法用于分派任何未处理的事件。要自动分派事件，我们可以将 `autoCommit` 属性设置为 `true`：

```typescript
export class Hero extends AggregateRoot {
  constructor(private id: string) {
    super();
    this.autoCommit = true;
  }
}
```

如果我们想将事件发布者合并到一个不存在的对象中，而是合并到一个类中，我们可以使用 `EventPublisher#mergeClassContext` 方法：

```typescript
const HeroModel = this.publisher.mergeClassContext(Hero);
const hero = new HeroModel('id'); // <-- HeroModel 是一个类
```

现在 `HeroModel` 类的每个实例都可以发布事件，而无需使用 `mergeObjectContext()` 方法。

此外，我们还可以使用 `EventBus` 手动发出事件：

```typescript
this.eventBus.publish(new HeroKilledDragonEvent());
```

> info **提示** `EventBus` 是一个可注入的类。

每个事件可以有多个 **事件处理器**。

```typescript
@@filename(hero-killed-dragon.handler)
@EventsHandler(HeroKilledDragonEvent)
export class HeroKilledDragonHandler implements IEventHandler<HeroKilledDragonEvent> {
  constructor(private repository: HeroesRepository) {}

  handle(event: HeroKilledDragonEvent) {
    // 业务逻辑
  }
}
```

> info **提示** 请注意，当你开始使用事件处理器时，你将脱离传统的 HTTP Web 上下文。
>
> - `CommandHandlers` 中的错误仍可被内置的 [异常过滤器](/exception-filters) 捕获。
> - `EventHandlers` 中的错误无法被异常过滤器捕获：你必须手动处理它们。可以通过简单的 `try/catch`，使用 [Sagas](/recipes/cqrs#sagas) 触发补偿事件，或选择其他解决方案。
> - `CommandHandlers` 中的 HTTP 响应仍可发送回客户端。
> - `EventHandlers` 中的 HTTP 响应则不能。如果你想向客户端发送信息，可以使用 [WebSocket](/websockets/gateways)、[SSE](/techniques/server-sent-events)，或选择其他解决方案。

与命令和查询一样，请确保在模块中将 `HeroKilledDragonHandler` 注册为提供者：

```typescript
providers: [HeroKilledDragonHandler];
```

#### Saga

Saga 是一个长时间运行的进程，它监听事件并可能触发新命令。它通常用于管理应用程序中的复杂工作流。例如，当用户注册时，一个 Saga 可能会监听 `UserRegisteredEvent` 并向用户发送欢迎邮件。

Sagas 是一个极其强大的功能。一个 Saga 可以监听 1..\* 个事件。使用 [RxJS](https://github.com/ReactiveX/rxjs) 库，我们可以过滤、映射、分叉和合并事件流以创建复杂的工作流。每个 Saga 返回一个 Observable，它产生一个命令实例。然后，该命令由 `CommandBus` **异步** 分派。

让我们创建一个 Saga，监听 `HeroKilledDragonEvent` 并分派 `DropAncientItemCommand` 命令。

```typescript
@@filename(heroes-game.saga)
@Injectable()
export class HeroesGameSagas {
  @Saga()
  dragonKilled = (events$: Observable<any>): Observable<ICommand> => {
    return events$.pipe(
      ofType(HeroKilledDragonEvent),
      map((event) => new DropAncientItemCommand(event.heroId, fakeItemID)),
    );
  }
}
@@switch
@Injectable()
export class HeroesGameSagas {
  @Saga()
  dragonKilled = (events$) => {
    return events$.pipe(
      ofType(HeroKilledDragonEvent),
      map((event) => new DropAncientItemCommand(event.heroId, fakeItemID)),
    );
  }
}
```

> info **提示** `ofType` 运算符和 `@Saga()` 装饰器从 `@nestjs/cqrs` 包中导出。

`@Saga()` 装饰器将方法标记为 Saga。`events$` 参数是所有事件的 Observable 流。`ofType` 运算符按指定的事件类型过滤流。`map` 运算符将事件映射到新的命令实例。

在此示例中，我们将 `HeroKilledDragonEvent` 映射到 `DropAncientItemCommand` 命令。然后 `CommandBus` 会自动分派该 `DropAncientItemCommand` 命令。

与查询、命令和事件处理器一样，请确保在模块中将 `HeroesGameSagas` 注册为提供者：

```typescript
providers: [HeroesGameSagas];
```

#### 未处理的异常

事件处理器是异步执行的，因此它们必须始终正确处理异常，以防止应用程序进入不一致状态。如果异常未被处理，`EventBus` 将创建一个 `UnhandledExceptionInfo` 对象并将其推送到 `UnhandledExceptionBus` 流。该流是一个 `Observable`，可用于处理未处理的异常。

```typescript
private destroy$ = new Subject<void>();

constructor(private unhandledExceptionsBus: UnhandledExceptionBus) {
  this.unhandledExceptionsBus
    .pipe(takeUntil(this.destroy$))
    .subscribe((exceptionInfo) => {
      // 在此处处理异常
      // 例如发送到外部服务、终止进程或发布新事件
    });
}

onModuleDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

要过滤异常，我们可以使用 `ofType` 运算符，如下所示：

```typescript
this.unhandledExceptionsBus
  .pipe(
    takeUntil(this.destroy$),
    UnhandledExceptionBus.ofType(TransactionNotAllowedException),
  )
  .subscribe((exceptionInfo) => {
    // 在此处处理异常
  });
```

其中 `TransactionNotAllowedException` 是我们想要过滤的异常。

`UnhandledExceptionInfo` 对象包含以下属性：

```typescript
export interface UnhandledExceptionInfo<
  Cause = IEvent | ICommand,
  Exception = any,
> {
  /**
   * 抛出的异常。
   */
  exception: Exception;
  /**
   * 异常的原因（事件或命令引用）。
   */
  cause: Cause;
}
```

#### 订阅所有事件

`CommandBus`、`QueryBus` 和 `EventBus` 都是 **Observables**。这意味着我们可以订阅整个流并例如处理所有事件。例如，我们可以将所有事件记录到控制台，或保存到事件存储中。

```typescript
private destroy$ = new Subject<void>();

constructor(private eventBus: EventBus) {
  this.eventBus
    .pipe(takeUntil(this.destroy$))
    .subscribe((event) => {
      // 将事件保存到数据库
    });
}

onModuleDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

#### 请求作用域

对于来自不同编程语言背景的人来说，可能会感到惊讶的是，在 Nest 中，大多数东西在传入请求之间是共享的。这包括数据库连接池、具有全局状态的单例服务等。请记住，Node.js 不遵循每个请求由单独线程处理的请求/响应多线程无状态模型。因此，使用单例实例对我们的应用程序来说是 **安全的**。

然而，有些边缘情况中，处理器的请求作用域可能是可取的。这可能包括 GraphQL 应用程序中的每个请求缓存、请求跟踪或多租户。你可以在此处了解更多关于如何控制作用域的信息 [here](/fundamentals/injection-scopes)。

在 CQRS 中使用请求作用域的提供者可能会比较复杂，因为 `CommandBus`、`QueryBus` 和 `EventBus` 是单例的。幸运的是，`@nestjs/cqrs` 包通过为每个处理的命令、查询或事件自动创建请求作用域处理器的新实例来简化了这一过程。

要使处理器具有请求作用域，你可以：

1. 依赖一个请求作用域的提供者。
2. 使用 `@CommandHandler`、`@QueryHandler` 或 `@EventsHandler` 装饰器显式将其作用域设置为 `REQUEST`，如下所示：

```typescript
@CommandHandler(KillDragonCommand, {
  scope: Scope.REQUEST,
})
export class KillDragonHandler {
  // 实现代码
}
```

要将请求负载注入任何请求作用域的提供者中，你可以使用 `@Inject(REQUEST)` 装饰器。然而，CQRS 中请求负载的性质取决于上下文——它可能是 HTTP 请求、计划任务，或其他触发命令的操作。

负载必须是一个继承自 `AsyncContext`（由 `@nestjs/cqrs` 提供）的类的实例，它作为请求上下文并持有在整个请求生命周期中可访问的数据。

```typescript
import { AsyncContext } from '@nestjs/cqrs';

export class MyRequest extends AsyncContext {
  constructor(public readonly user: User) {
    super();
  }
}
```

执行命令时，将自定义请求上下文作为第二个参数传递给 `CommandBus#execute` 方法：

```typescript
const myRequest = new MyRequest(user);
await this.commandBus.execute(
  new KillDragonCommand(heroId, killDragonDto.dragonId),
  myRequest,
);
```

这使得 `MyRequest` 实例作为 `REQUEST` 提供者对相应的处理器可用：

```typescript
@CommandHandler(KillDragonCommand, {
  scope: Scope.REQUEST,
})
export class KillDragonHandler {
  constructor(
    @Inject(REQUEST) private request: MyRequest, // 注入请求上下文
  ) {}

  // 处理器实现代码
}
```

你可以对查询采用相同的方法：

```typescript
const myRequest = new MyRequest(user);
const hero = await this.queryBus.execute(new GetHeroQuery(heroId), myRequest);
```

以及在查询处理器中：

```typescript
@QueryHandler(GetHeroQuery, {
  scope: Scope.REQUEST,
})
export class GetHeroHandler {
  constructor(
    @Inject(REQUEST) private request: MyRequest, // 注入请求上下文
  ) {}

  // 处理器实现代码
}
```

对于事件，虽然你可以将请求提供者传递给 `EventBus#publish`，但这不太常见。相反，使用 `EventPublisher` 将请求提供者合并到模型中：

```typescript
const hero = this.publisher.mergeObjectContext(
  await this.repository.findOneById(+heroId),
  this.request, // 在此处注入请求上下文
);
```

订阅这些事件的请求作用域事件处理器将能够访问请求提供者。

Sagas 始终是单例实例，因为它们管理长期运行的进程。然而，你可以从事件对象中检索请求提供者：

```typescript
@Saga()
dragonKilled = (events$: Observable<any>): Observable<ICommand> => {
  return events$.pipe(
    ofType(HeroKilledDragonEvent),
    map((event) => {
      const request = AsyncContext.of(event); // 检索请求上下文
      const command = new DropAncientItemCommand(event.heroId, fakeItemID);

      AsyncContext.merge(request, command); // 将请求上下文合并到命令中
      return command;
    }),
  );
}
```

或者，使用 `request.attachTo(command)` 方法将请求上下文绑定到命令。

#### 示例

一个可用的示例可在 [这里](https://github.com/kamilmysliwiec/nest-cqrs-example) 找到。