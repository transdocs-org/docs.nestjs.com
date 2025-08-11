### 迁移指南

本文提供了一个从 NestJS 10 版本迁移到 11 版本的完整指南。要了解 v11 中引入的新功能，请查看 [这篇文章](https://trilon.io/blog/announcing-nestjs-11-whats-new)。尽管此次更新包含一些小的破坏性变更，但它们不太可能影响大多数用户。您可以在此查看完整的变更列表：[v11.0.0 版本发布说明](https://github.com/nestjs/nest/releases/tag/v11.0.0)。

#### 升级包

虽然您可以手动升级您的包，但我们推荐使用 [npm-check-updates (ncu)](https://npmjs.com/package/npm-check-updates) 来获得更流畅的升级过程。

#### Express v5

经过数年的开发，Express v5 于 2024 年正式发布，并在 2025 年成为稳定版本。随着 NestJS 11 的发布，Express v5 现在成为框架中默认集成的版本。尽管此次更新对大多数用户来说是无缝的，但需要注意的是，Express v5 引入了一些破坏性变更。详细指导请参考 [Express v5 迁移指南](https://expressjs.com/en/guide/migrating-5.html)。

Express v5 中最显著的更新之一是修订了路径路由匹配算法。以下是对路径字符串如何与传入请求匹配的一些变更：

- 通配符 `*` 必须具有名称，与参数的行为一致：使用 `/*splat` 或 `/{{ '{' }}*splat&#125;` 代替 `/*`。`splat` 只是通配符参数的名称，没有特殊含义。您可以将其命名为任何名称，例如 `*wildcard`。
- 不再支持可选字符 `?`，请改用花括号语法：`/:file{{ '{' }}.:ext&#125;`。
- 不再支持正则表达式字符。
- 为避免升级时的混淆，部分字符已被保留，需要使用 `\` 进行转义：`(()[]?+!)`。
- 参数名称现在支持有效的 JavaScript 标识符，或者可以使用引号包裹，例如 `:"this"`。

也就是说，之前在 Express v4 中正常工作的路由可能在 Express v5 中不再适用。例如：

```typescript
@Get('users/*')
findAll() {
  // 在 NestJS 11 中，这将自动转换为有效的 Express v5 路由。
  // 尽管它可能仍然有效，但建议不要再使用 Express v5 中的这种通配符语法。
  return '此路由在 Express v5 中不应起作用';
}
```

要修复此问题，可以将路由更新为使用命名的通配符：

```typescript
@Get('users/*splat')
findAll() {
  return '此路由将在 Express v5 中起作用';
}
```

> warning **警告** 注意，`*splat` 是一个命名的通配符，匹配除根路径外的任何路径。如果您还需要匹配根路径（例如 `/users`），可以使用 `/users/{{ '{' }}*splat&#125;`，将通配符包裹在花括号中（可选组）。注意，`splat` 只是通配符参数的名称，没有特殊含义。您可以将其命名为任何名称，例如 `*wildcard`。

同样，如果您有一个中间件应用于所有路由，您可能需要将路径更新为使用命名的通配符：

```typescript
// 在 NestJS 11 中，这将自动转换为有效的 Express v5 路由。
// 尽管它可能仍然有效，但建议不要再使用 Express v5 中的这种通配符语法。
forRoutes('*'); // <-- 此路由在 Express v5 中不应起作用
```

相反，您可以将路径更新为使用命名的通配符：

```typescript
forRoutes('{*splat}'); // <-- 此路由将在 Express v5 中起作用
```

注意，`{{ '{' }}*splat&#125;` 是一个命名的通配符，匹配包括根路径在内的任何路径。外层的花括号使路径变为可选。

#### 查询参数解析

> info **提示** 此变更仅适用于 Express v5。

在 Express v5 中，默认情况下不再使用 `qs` 库解析查询参数。取而代之的是使用 `simple` 解析器，它不支持嵌套对象或数组。

因此，类似以下的查询字符串：

```plaintext
?filter[where][name]=John&filter[where][age]=30
?item[]=1&item[]=2
```

将不再按预期解析。要恢复到之前的行为，可以通过将 `query parser` 选项设置为 `extended` 来配置 Express 使用 `extended` 解析器（这是 Express v4 中的默认值）：

```typescript
import { NestFactory } from '@nestjs/core';
import { NestExpressApplication } from '@nestjs/platform-express';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(AppModule); // <-- 确保使用 <NestExpressApplication>
  app.set('query parser', 'extended'); // <-- 添加此行
  await app.listen(3000);
}
bootstrap();
```

#### Fastify v5

`@nestjs/platform-fastify` v11 现在终于支持 Fastify v5。对于大多数用户来说，此次更新应该是无缝的；不过，Fastify v5 引入了一些破坏性变更，尽管这些变更不太可能影响大多数 NestJS 用户。更多详细信息，请参考 [Fastify v5 迁移指南](https://fastify.dev/docs/v5.1.x/Guides/Migration-Guide-V5/)。

> info **提示** Fastify v5 中的路径匹配没有变化（中间件除外，请参见下文相关章节），因此您可以继续使用之前使用的通配符语法。行为保持不变，使用通配符（如 `*`）定义的路由仍将按预期工作。

#### Fastify CORS

默认情况下，只允许 [CORS 安全列表中的方法](https://fetch.spec.whatwg.org/#methods)。如果您需要启用其他方法（如 `PUT`、`PATCH` 或 `DELETE`），则必须在 `methods` 选项中显式定义它们。

```typescript
const methods = ['GET', 'POST', 'PUT', 'PATCH', 'DELETE']; // 或逗号分隔的字符串 'GET,POST,PUT,PATH,DELETE'

const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter(),
  { cors: { methods } },
);

// 或者，您可以使用 `enableCors` 方法
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter(),
);
app.enableCors({ methods });
```

#### Fastify 中间件注册

NestJS 11 现在使用最新版本的 [path-to-regexp](https://www.npmjs.com/package/path-to-regexp) 包来匹配 `@nestjs/platform-fastify` 中的 **中间件路径**。因此，用于匹配所有路径的 `(.*)` 语法不再受支持。相反，您应该使用命名的通配符。

例如，如果您有一个应用于所有路由的中间件：

```typescript
// 在 NestJS 11 中，即使您不更新它，这也会自动转换为有效的路由。
.forRoutes('(.*)');
```

您需要将其更新为使用命名的通配符：

```typescript
.forRoutes('*splat');
```

其中 `splat` 只是通配符参数的任意名称。您可以将其命名为任何名称。

#### 模块解析算法

从 NestJS 11 开始，模块解析算法得到了改进，以提高大多数应用程序的性能并减少内存使用。此更改不需要任何手动干预，但在某些边缘情况下，行为可能与以前版本有所不同。

在 NestJS v10 及更早版本中，动态模块会被分配一个基于模块动态元数据生成的唯一不透明密钥。该密钥用于在模块注册表中标识模块。例如，如果您在多个模块中包含 `TypeOrmModule.forFeature([User])`，NestJS 会去重这些模块，并将它们视为注册表中的单个模块节点。这个过程称为节点去重。

随着 NestJS v11 的发布，我们不再为动态模块生成可预测的哈希。相反，现在使用对象引用来确定一个模块是否等同于另一个模块。要在多个模块之间共享同一个动态模块，只需将其赋值给一个变量并在需要的地方导入即可。这种新方法提供了更大的灵活性，并确保动态模块的处理更加高效。

如果您的集成测试中使用了大量动态模块，此新算法可能会影响您的测试，因为在没有上述手动去重的情况下，您的 `TestingModule` 可能会包含多个依赖项实例。这使得存根方法变得稍微复杂，因为您需要定位到正确的实例。您有以下几种选择：

- 对您想要存根的动态模块进行去重；
- 使用 `module.select(ParentModule).get(Target)` 来找到正确的实例；
- 使用 `module.get(Target, {{ '{' }} each: true &#125;)` 存根所有实例；
- 或者通过使用 `Test.createTestingModule({{ '{' }}&#125;, {{ '{' }} moduleIdGeneratorAlgorithm: 'deep-hash' &#125;)` 将测试切换回旧算法。

#### Reflector 类型推断

NestJS 11 对 `Reflector` 类进行了多项改进，增强了其元数据值的功能和类型推断能力。这些更新提供了在处理元数据时更加直观和稳健的体验。

1. `getAllAndMerge` 现在在只有一个元数据条目且 `value` 是 `object` 类型时返回一个对象，而不是包含单个元素的数组。这一变更在处理基于对象的元数据时提高了一致性。
2. `getAllAndOverride` 的返回类型已更新为 `T | undefined` 而不是 `T`。此更新更好地反映了未找到元数据的可能性，并确保正确处理未定义的情况。
3. `ReflectableDecorator` 的转换类型参数现在在所有方法中都能正确推断。

这些增强功能通过提供更好的类型安全性和元数据处理方式，改善了 NestJS 11 的整体开发者体验。

#### 生命周期钩子执行顺序

终止生命周期钩子现在以与其初始化对应的相反顺序执行。也就是说，像 `OnModuleDestroy`、`BeforeApplicationShutdown` 和 `OnApplicationShutdown` 这样的钩子现在将以相反的顺序执行。

设想以下场景：

```plaintext
// 其中 A、B 和 C 是模块，"->" 表示模块依赖关系。
A -> B -> C
```

在这种情况下，`OnModuleInit` 钩子的执行顺序如下：

```plaintext
C -> B -> A
```

而 `OnModuleDestroy` 钩子则以相反的顺序执行：

```plaintext
A -> B -> C
```

> info **提示** 全局模块被视为所有其他模块的依赖项。这意味着全局模块首先被初始化，最后被销毁。

#### 中间件注册顺序

在 NestJS v11 中，中间件注册的行为已更新。以前，中间件注册的顺序由模块依赖图的拓扑排序决定，无论中间件是在全局模块还是普通模块中注册，根模块的距离定义了中间件注册的顺序。在这方面，全局模块被视为普通模块，这导致了不一致的行为，尤其是在与其他框架功能比较时。

从 v11 开始，无论其在模块依赖图中的位置如何，**全局模块中注册的中间件现在始终首先执行**。此更改确保了全局中间件始终在导入模块中的中间件之前运行，从而保持一致且可预测的顺序。

#### 缓存模块

`CacheModule`（来自 `@nestjs/cache-manager` 包）已更新，以支持最新版本的 `cache-manager` 包。此次更新带来了一些破坏性变更，包括迁移到 [Keyv](https://keyv.org/)，Keyv 为通过存储适配器实现多个后端存储的键值存储提供了一个统一的接口。

新旧版本之间的主要区别在于外部存储的配置方式。在旧版本中，要注册 Redis 存储，您可能像这样配置：

```ts
// 旧版本 - 不再支持
CacheModule.registerAsync({
  useFactory: async () => {
    const store = await redisStore({
      socket: {
        host: 'localhost',
        port: 6379,
      },
    });

    return {
      store,
    };
  },
}),
```

在新版本中，您应该使用 `Keyv` 适配器来配置存储：

```ts
// 新版本 - 支持
CacheModule.registerAsync({
  useFactory: async () => {
    return {
      stores: [
        new KeyvRedis('redis://localhost:6379'),
      ],
    };
  },
}),
```

其中 `KeyvRedis` 从 `@keyv/redis` 包导入。请参阅 [缓存文档](/techniques/caching) 以了解更多信息。

> warning **警告** 在此次更新中，Keyv 库处理的缓存数据现在以包含 `value` 和 `expires` 字段的对象形式存储，例如：`{{ '{' }}"value": "yourData", "expires": 1678901234567{{ '}' }}`。虽然 Keyv 在通过其 API 访问数据时会自动检索 `value` 字段，但如果直接与缓存数据交互（例如，在 cache-manager API 之外）或需要支持使用旧版 `@nestjs/cache-manager` 写入的数据，请注意此变更。

#### 配置模块

如果您使用的是 `@nestjs/config` 包中的 `ConfigModule`，请注意 `@nestjs/config@4.0.0` 引入的几个破坏性变更。最值得注意的是，`ConfigService#get` 方法读取配置变量的顺序已更新。新的顺序如下：

- 内部配置（配置命名空间和自定义配置文件）；
- 已验证的环境变量（如果启用了验证并提供了模式）；
- `process.env` 对象。

以前，已验证的环境变量和 `process.env` 对象优先读取，这阻止了它们被内部配置覆盖。此次更新后，内部配置现在始终优先于环境变量。

此外，之前允许禁用 `process.env` 对象验证的 `ignoreEnvVars` 配置选项已被弃用。请改用 `validatePredefined` 选项（设置为 `false` 以禁用对预定义环境变量的验证）。预定义环境变量是指在模块导入之前设置的 `process.env` 变量。例如，如果您使用 `PORT=3000 node main.js` 启动应用程序，则 `PORT` 变量被视为预定义变量。然而，从 `.env` 文件加载的 `ConfigModule` 变量不被视为预定义变量。

还引入了一个新的 `skipProcessEnv` 选项。该选项允许您完全阻止 `ConfigService#get` 方法访问 `process.env` 对象，这在您希望限制服务直接读取环境变量时非常有用。

#### Terminus 模块

如果您正在使用 `TerminusModule` 并构建了自己的自定义健康检查指示器，v11 中引入了一个新的 API。新的 `HealthIndicatorService` 旨在增强自定义健康检查指示器的可读性和可测试性。

在 v11 之前，健康检查指示器可能如下所示：

```typescript
@Injectable()
export class DogHealthIndicator extends HealthIndicator {
  constructor(private readonly httpService: HttpService) {
    super();
  }

  async isHealthy(key: string) {
    try {
      const badboys = await this.getBadboys();
      const isHealthy = badboys.length === 0;

      const result = this.getStatus(key, isHealthy, {
        badboys: badboys.length,
      });

      if (!isHealthy) {
        throw new HealthCheckError('Dog check failed', result);
      }

      return result;
    } catch (error) {
      const result = this.getStatus(key, isHealthy);
      throw new HealthCheckError('Dog check failed', result);
    }
  }

  private getBadboys() {
    return firstValueFrom(
      this.httpService.get<Dog[]>('https://example.com/dog').pipe(
        map((response) => response.data),
        map((dogs) => dogs.filter((dog) => dog.state === DogState.BAD_BOY)),
      ),
    );
  }
}
```

从 v11 开始，建议使用新的 `HealthIndicatorService` API，它简化了实现过程。以下是实现相同健康检查指示器的新方式：

```typescript
@Injectable()
export class DogHealthIndicator {
  constructor(
    private readonly httpService: HttpService,
    // 注入由 `TerminusModule` 提供的 `HealthIndicatorService`
    private readonly healthIndicatorService: HealthIndicatorService,
  ) {}

  async isHealthy(key: string) {
    // 为给定键启动健康检查
    const indicator = this.healthIndicatorService.check(key);

    try {
      const badboys = await this.getBadboys();
      const isHealthy = badboys.length === 0;

      if (!isHealthy) {
        // 标记指示器为“down”并向响应中添加附加信息
        return indicator.down({ badboys: badboys.length });
      }

      // 标记健康检查指示器为“up”
      return indicator.up();
    } catch (error) {
      return indicator.down('Unable to retrieve dogs');
    }
  }

  private getBadboys() {
    // ...
  }
}
```

主要变更：

- `HealthIndicatorService` 取代了旧的 `HealthIndicator` 和 `HealthCheckError` 类，提供了更简洁的健康检查 API；
- `check` 方法允许轻松地跟踪状态（`up` 或 `down`），同时支持在健康检查响应中包含附加元数据。

> info **提示** 请注意，`HealthIndicator` 和 `HealthCheckError` 类已被标记为弃用，并计划在下一个主要版本中移除。

#### Node.js v16 和 v18 不再支持

从 NestJS 11 开始，Node.js v16 不再受支持，因为其生命周期已于 2023 年 9 月 11 日结束。同样，Node.js v18 的安全支持也将在 2025 年 4 月 30 日结束，因此我们提前放弃了对其的支持。

NestJS 11 现在要求使用 **Node.js v20 或更高版本**。

为了获得最佳体验，我们强烈建议使用最新的 Node.js LTS 版本。

#### Mau 官方部署平台

如果您错过了公告，我们在 2024 年推出了官方部署平台 [Mau](https://www.mau.nestjs.com/)。

Mau 是一个完全托管平台，简化了 NestJS 应用程序的部署流程。通过 Mau，您可以使用单条命令将应用程序部署到云端（**AWS**），管理环境变量，并实时监控应用程序性能。

Mau 使配置和维护基础设施变得简单，只需点击几下按钮即可。Mau 设计简单直观，因此您可以专注于构建应用程序，而无需担心底层基础设施。在底层，我们使用 Amazon Web Services 来为您提供强大可靠的平台，同时抽象掉了所有 AWS 的复杂性。我们为您承担了所有繁重的工作，因此您可以专注于构建应用程序并推动业务增长。

```bash
$ npm install -g @nestjs/mau
$ mau deploy
```

您可以在 [本章节](/deployment#easy-deployment-with-mau) 了解更多关于 Mau 的信息。