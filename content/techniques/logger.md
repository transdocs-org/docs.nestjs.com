### 日志记录器（Logger）

Nest 自带了一个基于文本的内置日志记录器，用于应用程序启动过程中以及一些其他场景（例如显示捕获的异常，即系统日志）的日志记录。此功能通过 `@nestjs/common` 包中的 `Logger` 类提供。你可以完全控制日志系统的功能，包括以下任意一项：

- 完全禁用日志记录
- 指定日志的详细级别（例如，显示错误、警告、调试信息等）
- 配置日志消息的格式化方式（原始文本、JSON、带颜色等）
- 覆盖默认日志记录器中的时间戳（例如，使用 ISO8601 标准作为日期格式）
- 完全替换默认日志记录器
- 通过继承默认日志记录器来自定义它
- 利用依赖注入简化应用程序的构建和测试

你也可以使用内置的日志记录器，或创建你自己的自定义实现，以记录你自己的应用程序级事件和消息。

如果你的应用程序需要与外部日志系统集成、自动文件日志记录，或转发日志到集中式日志服务，你可以使用 Node.js 日志记录库实现一个完全自定义的日志解决方案。一个流行的选择是 [Pino](https://github.com/pinojs/pino)，它以高性能和灵活性著称。

#### 基本自定义

要禁用日志记录，请在传递给 `NestFactory.create()` 方法的（可选）Nest 应用程序选项对象中将 `logger` 属性设置为 `false`。

```typescript
const app = await NestFactory.create(AppModule, {
  logger: false,
});
await app.listen(process.env.PORT ?? 3000);
```

要启用特定的日志级别，请将 `logger` 属性设置为包含所需日志级别的字符串数组，如下所示：

```typescript
const app = await NestFactory.create(AppModule, {
  logger: ['error', 'warn'],
});
await app.listen(process.env.PORT ?? 3000);
```

数组中的值可以是 `'log'`、`'fatal'`、`'error'`、`'warn'`、`'debug'` 和 `'verbose'` 的任意组合。

要禁用彩色输出，请将 `ConsoleLogger` 对象的 `colors` 属性设置为 `false`，并将其作为 `logger` 属性的值传递：

```typescript
const app = await NestFactory.create(AppModule, {
  logger: new ConsoleLogger({
    colors: false,
  }),
});
```

要为每条日志消息配置前缀，请将 `ConsoleLogger` 对象的 `prefix` 属性设置如下：

```typescript
const app = await NestFactory.create(AppModule, {
  logger: new ConsoleLogger({
    prefix: 'MyApp', // 默认是 "Nest"
  }),
});
```

以下是所有可用选项的表格说明：

| 选项            | 描述                                                                                                                                                                                                                                                                                                                                          | 默认值                                        |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| `logLevels`       | 启用的日志级别。                                                                                                                                                                                                                                                                                                                                  | `['log', 'error', 'warn', 'debug', 'verbose']` |
| `timestamp`       | 如果启用，将打印当前日志与上一条日志之间的时间差。注意：当启用 `json` 时此选项无效。                                                                                                                                                                                                                                                                   | `false`                                        |
| `prefix`          | 每条日志消息使用的前缀。注意：当启用 `json` 时此选项无效。                                                                                                                                                                                                                                                      | `Nest`                                         |
| `json`            | 如果启用，将以 JSON 格式打印日志消息。                                                                                                                                                                                                                                                                                               | `false`                                        |
| `colors`          | 如果启用，将以彩色打印日志消息。默认情况下，如果未启用 `json` 则为 `true`，否则为 `false`。                                                                                                                                                                                                                                                  | `true`                                         |
| `context`         | 日志记录器的上下文。                                                                                                                                                                                                                                                                                                                           | `undefined`                                    |
| `compact`         | 如果启用，即使对象包含多个属性，也将日志消息打印为单行。若设置为数字，则最多 n 个内部元素将合并为一行，前提是所有属性适合 breakLength。短数组元素也会被分组。                                                                 | `true`                                         |
| `maxArrayLength`  | 指定格式化时包括的数组、类型化数组、Map、Set、WeakMap 和 WeakSet 元素的最大数量。设为 null 或 Infinity 以显示所有元素。设为 0 或负数则不显示任何元素。当启用 `json`、禁用颜色且 `compact` 设置为 true 时忽略此选项，因为它会产生可解析的 JSON 输出。             | `100`                                          |
| `maxStringLength` | 指定格式化时包括的字符最大数量。设为 null 或 Infinity 以显示所有元素。设为 0 或负数则不显示任何字符。当启用 `json`、禁用颜色且 `compact` 设置为 true 时忽略此选项，因为它会产生可解析的 JSON 输出。                                                           | `10000`                                        |
| `sorted`          | 如果启用，格式化对象时将对键进行排序。也可以是一个自定义排序函数。当启用 `json`、禁用颜色且 `compact` 设置为 true 时忽略此选项，因为它会产生可解析的 JSON 输出。                                                                                                                                | `false`                                        |
| `depth`           | 指定格式化对象时递归的次数。这对于检查大对象很有用。传入 Infinity 或 null 可递归到最大调用栈大小。当启用 `json`、禁用颜色且 `compact` 设置为 true 时忽略此选项，因为它会产生可解析的 JSON 输出。                                         | `5`                                            |
| `showHidden`      | 如果为 true，对象的不可枚举符号和属性将包含在格式化结果中。WeakMap 和 WeakSet 条目以及用户定义的原型属性也将包含在内                                                                                                                                                             | `false`                                        |
| `breakLength`     | 输入值被拆分为多行的长度。设为 Infinity 时，将输入格式化为单行（需配合 "compact" 设为 true）。当 "compact" 为 true 时默认为 Infinity，否则为 80。当启用 `json`、禁用颜色且 `compact` 设置为 true 时忽略此选项，因为它会产生可解析的 JSON 输出。 | `Infinity`                                     |

#### JSON 日志记录

JSON 日志记录对于现代应用程序的可观测性和与日志管理系统集成至关重要。要在你的 NestJS 应用程序中启用 JSON 日志记录，请将 `ConsoleLogger` 对象的 `json` 属性设置为 `true`，然后在创建应用程序实例时将此日志记录器配置作为 `logger` 属性的值提供。

```typescript
const app = await NestFactory.create(AppModule, {
  logger: new ConsoleLogger({
    json: true,
  }),
});
```

这种配置以结构化的 JSON 格式输出日志，使其更容易与诸如日志聚合器和云平台等外部系统集成。例如，像 **AWS ECS**（Elastic Container Service）这样的平台原生支持 JSON 日志，从而启用了以下高级功能：

- **日志过滤**：可以根据日志级别、时间戳或自定义元数据轻松缩小日志范围。
- **搜索与分析**：使用查询工具分析和跟踪应用程序行为的趋势。

此外，如果你使用 [NestJS Mau](https://mau.nestjs.com)，JSON 日志记录简化了以结构化格式查看日志的过程，这在调试和性能监控时尤其有用。

> info **注意** 当 `json` 设置为 `true` 时，`ConsoleLogger` 会自动将 `colors` 属性设置为 `false`，以禁用文本颜色化，确保输出为有效的 JSON，不含格式化干扰。不过，为了开发目的，你可以通过显式设置 `colors` 为 `true` 来覆盖此行为。这将添加带颜色的 JSON 日志，使本地调试时日志条目更易读。

当启用 JSON 日志记录时，日志输出将如下所示（单行）：

```json
{
  "level": "log",
  "pid": 19096,
  "timestamp": 1607370779834,
  "message": "Starting Nest application...",
  "context": "NestFactory"
}
```

你可以在 [Pull Request](https://github.com/nestjs/nest/pull/14121) 中看到不同的变体。

#### 使用日志记录器进行应用程序日志记录

我们可以结合上述几种技术，为 Nest 系统日志记录和我们自己的应用程序事件/消息日志记录提供一致的行为和格式。

一个良好的做法是在我们每个服务中实例化 `@nestjs/common` 中的 `Logger` 类。我们可以在 `Logger` 构造函数中将服务名称作为 `context` 参数传入，如下所示：

```typescript
import { Logger, Injectable } from '@nestjs/common';

@Injectable()
class MyService {
  private readonly logger = new Logger(MyService.name);

  doSomething() {
    this.logger.log('Doing something...');
  }
}
```

在默认的日志记录器实现中，`context` 将以方括号显示，如下面示例中的 `NestFactory`：

```bash
[Nest] 19096   - 12/08/2019, 7:12:59 AM   [NestFactory] Starting Nest application...
```

如果我们通过 `app.useLogger()` 提供了一个自定义日志记录器，它将被 Nest 内部使用。这意味着我们的代码仍然与实现无关，而我们可以通过调用 `app.useLogger()` 轻松地将默认日志记录器替换为我们的自定义日志记录器。

因此，如果我们按照上一节的步骤并调用 `app.useLogger(app.get(MyLogger))`，那么来自 `MyService` 的 `this.logger.log()` 调用将导致调用 `MyLogger` 实例的 `log` 方法。

这应该适用于大多数情况。但如果你需要更多自定义（如添加和调用自定义方法），请参阅下一节。

#### 带时间戳的日志记录

要为每条日志消息启用时间戳记录，可以在创建日志记录器实例时使用可选的 `timestamp: true` 设置。

```typescript
import { Logger, Injectable } from '@nestjs/common';

@Injectable()
class MyService {
  private readonly logger = new Logger(MyService.name, { timestamp: true });

  doSomething() {
    this.logger.log('Doing something with timestamp here ->');
  }
}
```

这将产生如下格式的输出：

```bash
[Nest] 19096   - 04/19/2024, 7:12:59 AM   [MyService] Doing something with timestamp here +5ms
```

请注意行末的 `+5ms`。对于每条日志语句，都会计算与上一条消息之间的时间差，并显示在行末。

#### 自定义实现

你可以提供一个自定义日志记录器实现，供 Nest 用于系统日志记录，方法是将 `logger` 属性的值设置为符合 `LoggerService` 接口的对象。例如，你可以告诉 Nest 使用内置的全局 JavaScript `console` 对象（该对象实现了 `LoggerService` 接口），如下所示：

```typescript
const app = await NestFactory.create(AppModule, {
  logger: console,
});
await app.listen(process.env.PORT ?? 3000);
```

实现你自己的自定义日志记录器非常简单。只需按如下方式实现 `LoggerService` 接口的每个方法即可：

```typescript
import { LoggerService, Injectable } from '@nestjs/common';

@Injectable()
export class MyLogger implements LoggerService {
  /**
   * 写入 'log' 级别日志。
   */
  log(message: any, ...optionalParams: any[]) {}

  /**
   * 写入 'fatal' 级别日志。
   */
  fatal(message: any, ...optionalParams: any[]) {}

  /**
   * 写入 'error' 级别日志。
   */
  error(message: any, ...optionalParams: any[]) {}

  /**
   * 写入 'warn' 级别日志。
   */
  warn(message: any, ...optionalParams: any[]) {}

  /**
   * 写入 'debug' 级别日志。
   */
  debug?(message: any, ...optionalParams: any[]) {}

  /**
   * 写入 'verbose' 级别日志。
   */
  verbose?(message: any, ...optionalParams: any[]) {}
}
```

然后，你可以通过 Nest 应用程序选项对象的 `logger` 属性提供 `MyLogger` 实例：

```typescript
const app = await NestFactory.create(AppModule, {
  logger: new MyLogger(),
});
await app.listen(process.env.PORT ?? 3000);
```

虽然这种技术很简单，但它并没有为 `MyLogger` 类使用依赖注入。这可能会带来一些挑战，特别是在测试时，也会限制 `MyLogger` 的可重用性。要获得更好的解决方案，请参阅下面的 <a href="techniques/logger#dependency-injection">依赖注入</a> 部分。

#### 扩展内置日志记录器

与其从头开始编写一个日志记录器，你可能通过扩展内置的 `ConsoleLogger` 类并覆盖默认实现的某些行为就能满足你的需求。

```typescript
import { ConsoleLogger } from '@nestjs/common';

export class MyLogger extends ConsoleLogger {
  error(message: any, stack?: string, context?: string) {
    // 在此处添加你的定制逻辑
    super.error(...arguments);
  }
}
```

你可以在你的功能模块中使用此类扩展的日志记录器，具体方法请参阅下面的 <a href="techniques/logger#using-the-logger-for-application-logging">使用日志记录器进行应用程序日志记录</a> 部分。

你可以通过将其实例作为应用程序选项对象的 `logger` 属性的值传递（如上面的 <a href="techniques/logger#custom-logger-implementation">自定义日志记录器实现</a> 部分所示），或使用下面 <a href="techniques/logger#dependency-injection">依赖注入</a> 部分中显示的技术，告诉 Nest 使用你的扩展日志记录器进行系统日志记录。如果是这样，请务必如上面示例代码中所示调用 `super`，以便将特定的日志方法调用委托给父类（内置类），从而使 Nest 能够依赖它期望的内置功能。

<app-banner-courses></app-banner-courses>

#### 依赖注入

对于更高级的日志功能，你将希望利用依赖注入。例如，你可能希望将 `ConfigService` 注入到你的日志记录器中以进行自定义，然后将你的自定义日志记录器注入到其他控制器和/或提供者中。要为你的自定义日志记录器启用依赖注入，请创建一个实现 `LoggerService` 接口的类，并在某个模块中将该类注册为提供者。例如，你可以：

1. 定义一个 `MyLogger` 类，该类可以继承内置的 `ConsoleLogger` 或完全重写它，如前面几节所示。请确保实现 `LoggerService` 接口。
2. 创建如下所示的 `LoggerModule`，并在该模块中提供 `MyLogger`。

```typescript
import { Module } from '@nestjs/common';
import { MyLogger } from './my-logger.service';

@Module({
  providers: [MyLogger],
  exports: [MyLogger],
})
export class LoggerModule {}
```

有了这个结构，你现在可以为其他任何模块提供你的自定义日志记录器。由于你的 `MyLogger` 类属于一个模块，它可以使用依赖注入（例如，注入一个 `ConfigService`）。还需要一个技术来为 Nest 提供用于系统日志记录（例如，启动和错误处理）的自定义日志记录器。

由于应用程序实例化（`NestFactory.create()`）发生在任何模块的上下文之外，因此它不参与初始化的正常依赖注入阶段。因此，我们必须确保至少有一个应用程序模块导入 `LoggerModule`，以触发 Nest 实例化我们的 `MyLogger` 类的单例实例。

然后，我们可以使用以下构造方式指示 Nest 使用相同的 `MyLogger` 单例实例：

```typescript
const app = await NestFactory.create(AppModule, {
  bufferLogs: true,
});
app.useLogger(app.get(MyLogger));
await app.listen(process.env.PORT ?? 3000);
```

> info **注意** 在上面的示例中，我们将 `bufferLogs` 设置为 `true`，以确保所有日志都被缓冲，直到附加了自定义日志记录器（在此例中为 `MyLogger`），并且应用程序初始化过程完成或失败。如果初始化过程失败，Nest 将回退到原始的 `ConsoleLogger` 以打印出报告的错误消息。此外，你可以将 `autoFlushLogs` 设置为 `false`（默认为 `true`）以手动刷新日志（使用 `Logger.flush()` 方法）。

在这里，我们使用 `NestApplication` 实例上的 `get()` 方法来检索 `MyLogger` 对象的单例实例。这种技术本质上是一种“注入”日志记录器实例供 Nest 使用的方法。`app.get()` 调用检索 `MyLogger` 的单例实例，这依赖于该实例首先在另一个模块中被注入，如上所述。

你还可以将这个 `MyLogger` 提供者注入到你的功能类中，从而确保 Nest 系统日志记录和应用程序日志记录之间具有一致的日志行为。更多信息请参见下面的 <a href="techniques/logger#using-the-logger-for-application-logging">使用日志记录器进行应用程序日志记录</a> 和 <a href="techniques/logger#injecting-a-custom-logger">注入自定义日志记录器</a> 部分。

#### 注入自定义日志记录器

首先，通过如下代码扩展内置日志记录器。我们为 `ConsoleLogger` 类提供 `scope` 选项作为配置元数据，指定一个 [transient](/fundamentals/injection-scopes) 作用域，以确保在每个功能模块中都有一个唯一的 `MyLogger` 实例。在这个示例中，我们没有扩展各个 `ConsoleLogger` 方法（如 `log()`、`warn()` 等），尽管你可以选择这样做。

```typescript
import { Injectable, Scope, ConsoleLogger } from '@nestjs/common';

@Injectable({ scope: Scope.TRANSIENT })
export class MyLogger extends ConsoleLogger {
  customLog() {
    this.log('Please feed the cat!');
  }
}
```

接下来，创建一个 `LoggerModule`，构造如下：

```typescript
import { Module } from '@nestjs/common';
import { MyLogger } from './my-logger.service';

@Module({
  providers: [MyLogger],
  exports: [MyLogger],
})
export class LoggerModule {}
```

接下来，将 `LoggerModule` 导入到你的功能模块中。由于我们扩展了默认的 `Logger`，我们可以方便地使用 `setContext` 方法。因此，我们可以开始使用上下文感知的自定义日志记录器，如下所示：

```typescript
import { Injectable } from '@nestjs/common';
import { MyLogger } from './my-logger.service';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  constructor(private myLogger: MyLogger) {
    // 由于是 transient 作用域，CatsService 拥有其自己的 MyLogger 唯一实例
    // 因此在此设置的 context 不会影响其他服务中的实例
    this.myLogger.setContext('CatsService');
  }

  findAll(): Cat[] {
    // 你可以调用所有默认方法
    this.myLogger.warn('About to return cats!');
    // 以及你的自定义方法
    this.myLogger.customLog();
    return this.cats;
  }
}
```

最后，指示 Nest 在你的 `main.ts` 文件中使用自定义日志记录器的实例，如下所示。当然，在这个示例中，我们实际上并没有自定义日志记录器的行为（通过扩展 `Logger` 方法如 `log()`、`warn()` 等），因此这一步实际上不是必需的。但是，如果你在这些方法中添加了自定义逻辑并希望 Nest 使用相同的实现，那么这一步就是必需的。

```typescript
const app = await NestFactory.create(AppModule, {
  bufferLogs: true,
});
app.useLogger(new MyLogger());
await app.listen(process.env.PORT ?? 3000);
```

> info **提示** 或者，你可以通过将 `logger: false` 指令传递给 `NestFactory.create` 来暂时禁用日志记录。请注意，如果你这样做了，在调用 `useLogger` 之前不会记录任何内容，因此你可能会错过一些重要的初始化错误。如果你不介意某些初始化消息将使用默认日志记录器记录，你可以省略 `logger: false` 选项。

#### 使用外部日志记录器

生产应用程序通常有特定的日志记录需求，包括高级过滤、格式化和集中式日志记录。Nest 的内置日志记录器用于监控 Nest 系统行为，在开发阶段用于功能模块的基本格式化文本日志记录也很有用，但生产应用程序通常会使用专用日志模块，如 [Winston](https://github.com/winstonjs/winston)。与任何标准的 Node.js 应用程序一样，你可以在 Nest 中充分利用此类模块。