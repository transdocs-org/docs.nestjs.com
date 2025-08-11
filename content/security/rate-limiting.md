### 速率限制

保护应用程序免受暴力攻击的一种常见技术是**速率限制**。要开始使用，你需要安装 `@nestjs/throttler` 包。

```bash
$ npm i --save @nestjs/throttler
```

安装完成后，`ThrottlerModule` 可以像其他 Nest 包一样通过 `forRoot` 或 `forRootAsync` 方法进行配置。

```typescript
@@filename(app.module)
@Module({
  imports: [
     ThrottlerModule.forRoot({
      throttlers: [
        {
          ttl: 60000,
          limit: 10,
        },
      ],
    }),
  ],
})
export class AppModule {}
```

以上配置将为你的应用程序中受保护的路由设置全局的 `ttl`（以毫秒为单位的生存时间）和 `limit`（在 ttl 内的最大请求数）选项。

模块导入后，你可以选择如何绑定 `ThrottlerGuard`。可以按照[守卫](https://docs.nestjs.com/guards)部分中提到的任何一种绑定方式。例如，如果你想全局绑定守卫，可以通过向任意模块添加以下提供者来实现：

```typescript
{
  provide: APP_GUARD,
  useClass: ThrottlerGuard
}
```

#### 多个速率限制定义

有时你可能希望设置多个速率限制定义，例如每秒最多 3 次调用，10 秒内最多 20 次调用，1 分钟内最多 100 次调用。为此，你可以在数组中设置命名选项，之后可以在 `@SkipThrottle()` 和 `@Throttle()` 装饰器中引用这些选项以更改限制。

```typescript
@@filename(app.module)
@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        name: 'short',
        ttl: 1000,
        limit: 3,
      },
      {
        name: 'medium',
        ttl: 10000,
        limit: 20
      },
      {
        name: 'long',
        ttl: 60000,
        limit: 100
      }
    ]),
  ],
})
export class AppModule {}
```

#### 自定义

有时你可能希望将守卫绑定到控制器或全局，但希望为一个或多个端点禁用速率限制。为此，你可以使用 `@SkipThrottle()` 装饰器，来跳过整个类或单个路由的速率限制。`@SkipThrottle()` 装饰器也可以接受一个字符串键和布尔值的对象，用于在你希望排除一个控制器中的大多数路由但不是全部时进行配置，特别是在你设置了多个速率限制器的情况下。如果不传递对象，默认值是 `{{ '{' }} default: true {{ '}' }}`。

```typescript
@SkipThrottle()
@Controller('users')
export class UsersController {}
```

这个 `@SkipThrottle()` 装饰器可以用来跳过某个路由或类的速率限制，也可以用来在一个被跳过的类中不跳过某个路由。

```typescript
@SkipThrottle()
@Controller('users')
export class UsersController {
  // 此路由应用速率限制
  @SkipThrottle({ default: false })
  dontSkip() {
    return '列表用户功能启用了速率限制。';
  }
  // 此路由跳过速率限制
  doSkip() {
    return '列表用户功能未启用速率限制。';
  }
}
```

还有一个 `@Throttle()` 装饰器，可用于覆盖全局模块中设置的 `limit` 和 `ttl`，以提供更严格或更宽松的安全选项。这个装饰器也可以用在类或方法上。从版本 5 开始，该装饰器接受一个对象，其中字符串对应的是速率限制器集的名称，对象包含 `limit` 和 `ttl` 键和整数值，类似于传递给根模块的选项。如果你在原始选项中没有设置名称，请使用字符串 `default`。你可以这样配置：

```typescript
// 覆盖速率限制和持续时间的默认配置
@Throttle({ default: { limit: 3, ttl: 60000 } })
@Get()
findAll() {
  return "列表用户功能启用了自定义的速率限制。";
}
```

#### 代理

如果你的应用程序运行在代理服务器后面，必须配置 HTTP 适配器以信任该代理。你可以参考 [Express](http://expressjs.com/en/guide/behind-proxies.html) 和 [Fastify](https://www.fastify.io/docs/latest/Reference/Server/#trustproxy) 的特定 HTTP 适配器选项来启用 `trust proxy` 设置。

以下示例演示了如何为 Express 适配器启用 `trust proxy`：

```typescript
@@filename(main)
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { NestExpressApplication } from '@nestjs/platform-express';

async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(AppModule);
  app.set('trust proxy', 'loopback'); // 信任来自回环地址的请求
  await app.listen(3000);
}

bootstrap();
@@switch
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { NestExpressApplication } from '@nestjs/platform-express';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.set('trust proxy', 'loopback'); // 信任来自回环地址的请求
  await app.listen(3000);
}

bootstrap();
```

启用 `trust proxy` 后，你可以从 `X-Forwarded-For` 请求头中获取原始 IP 地址。你还可以通过覆盖 `getTracker()` 方法，从该请求头提取 IP 地址，而不是依赖 `req.ip`。以下示例展示了如何为 Express 和 Fastify 实现这一点：

```typescript
@@filename(throttler-behind-proxy.guard)
import { ThrottlerGuard } from '@nestjs/throttler';
import { Injectable } from '@nestjs/common';

@Injectable()
export class ThrottlerBehindProxyGuard extends ThrottlerGuard {
  protected async getTracker(req: Record<string, any>): Promise<string> {
    return req.ips.length ? req.ips[0] : req.ip; // 自定义 IP 提取逻辑
  }
}
```

> info **提示** 你可以在这里找到 Express 的 `req` 请求对象 API：[Express](https://expressjs.com/en/api.html#req.ips)，Fastify 的在这里：[Fastify](https://www.fastify.io/docs/latest/Reference/Request/)。

#### WebSocket

此模块可以与 WebSocket 一起使用，但需要一些类扩展。你可以扩展 `ThrottlerGuard` 并覆盖 `handleRequest` 方法，如下所示：

```typescript
@Injectable()
export class WsThrottlerGuard extends ThrottlerGuard {
  async handleRequest(requestProps: ThrottlerRequest): Promise<boolean> {
    const {
      context,
      limit,
      ttl,
      throttler,
      blockDuration,
      getTracker,
      generateKey,
    } = requestProps;

    const client = context.switchToWs().getClient();
    const tracker = client._socket.remoteAddress;
    const key = generateKey(context, tracker, throttler.name);
    const { totalHits, timeToExpire, isBlocked, timeToBlockExpire } =
      await this.storageService.increment(
        key,
        ttl,
        limit,
        blockDuration,
        throttler.name,
      );

    const getThrottlerSuffix = (name: string) =>
      name === 'default' ? '' : `-${name}`;

    // 当用户达到限制时抛出错误
    if (isBlocked) {
      await this.throwThrottlingException(context, {
        limit,
        ttl,
        key,
        tracker,
        totalHits,
        timeToExpire,
        isBlocked,
        timeToBlockExpire,
      });
    }

    return true;
  }
}
```

> info **提示** 如果你使用的是 `ws`，请将 `_socket` 替换为 `conn`。

在使用 WebSocket 时需要注意以下几点：

- 守卫不能通过 `APP_GUARD` 或 `app.useGlobalGuards()` 注册
- 当达到限制时，Nest 会触发一个 `exception` 事件，因此请确保有监听器准备处理该事件

> info **提示** 如果你使用的是 `@nestjs/platform-ws` 包，你可以使用 `client._socket.remoteAddress`。

#### GraphQL

`ThrottlerGuard` 也可以用于 GraphQL 请求。同样，你可以扩展守卫，但这次需要覆盖 `getRequestResponse` 方法：

```typescript
@Injectable()
export class GqlThrottlerGuard extends ThrottlerGuard {
  getRequestResponse(context: ExecutionContext) {
    const gqlCtx = GqlExecutionContext.create(context);
    const ctx = gqlCtx.getContext();
    return { req: ctx.req, res: ctx.res };
  }
}
```

#### 配置

以下选项适用于传递给 `ThrottlerModule` 配置数组的对象：

<table>
  <tr>
    <td><code>name</code></td>
    <td>用于内部跟踪使用的是哪个速率限制器集的名称。如果没有传递，默认为 <code>default</code></td>
  </tr>
  <tr>
    <td><code>ttl</code></td>
    <td>每个请求在存储中持续的毫秒数</td>
  </tr>
  <tr>
    <td><code>limit</code></td>
    <td>在 TTL 限制内的最大请求数</td>
  </tr>
  <tr>
    <td><code>blockDuration</code></td>
    <td>请求将被阻止的毫秒数</td>
  </tr>
  <tr>
    <td><code>ignoreUserAgents</code></td>
    <td>要忽略的用户代理的正则表达式数组，当进行请求速率限制时忽略这些代理</td>
  </tr>
  <tr>
    <td><code>skipIf</code></td>
    <td>一个函数，接受 <code>ExecutionContext</code> 并返回一个 <code>boolean</code>，用于短路速率限制逻辑。类似于 <code>@SkipThrottler()</code>，但基于请求</td>
  </tr>
</table>

如果你需要设置存储，或者希望将上述选项以更全局的方式应用到每个速率限制器集上，可以通过 `throttlers` 选项键传递这些选项，并参考以下表格：

<table>
  <tr>
    <td><code>storage</code></td>
    <td>用于跟踪速率限制的自定义存储服务。 <a href="/security/rate-limiting#storages">点击此处。</a></td>
  </tr>
  <tr>
    <td><code>ignoreUserAgents</code></td>
    <td>要忽略的用户代理的正则表达式数组，当进行请求速率限制时忽略这些代理</td>
  </tr>
  <tr>
    <td><code>skipIf</code></td>
    <td>一个函数，接受 <code>ExecutionContext</code> 并返回一个 <code>boolean</code>，用于短路速率限制逻辑。类似于 <code>@SkipThrottler()</code>，但基于请求</td>
  </tr>
  <tr>
    <td><code>throttlers</code></td>
    <td>使用上面的表格定义的速率限制器集数组</td>
  </tr>
  <tr>
    <td><code>errorMessage</code></td>
    <td>一个 <code>string</code> 或一个函数，该函数接受 <code>ExecutionContext</code> 和 <code>ThrottlerLimitDetail</code> 并返回一个 <code>string</code>，用于覆盖默认的速率限制错误消息</td>
  </tr>
  <tr>
    <td><code>getTracker</code></td>
    <td>一个函数，接受 <code>Request</code> 并返回一个 <code>string</code>，用于覆盖 <code>getTracker</code> 方法的默认逻辑</td>
  </tr>
  <tr>
    <td><code>generateKey</code></td>
    <td>一个函数，接受 <code>ExecutionContext</code>、追踪器 <code>string</code> 和速率限制器名称作为 <code>string</code>，并返回一个 <code>string</code>，用于覆盖用于存储速率限制值的最终键。这将覆盖 <code>generateKey</code> 方法的默认逻辑</td>
  </tr>
</table>

#### 异步配置

你可能希望异步获取速率限制配置，而不是同步方式。你可以使用 `forRootAsync()` 方法，它允许依赖注入和 `async` 方法。

一种方法是使用工厂函数：

```typescript
@Module({
  imports: [
    ThrottlerModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => [
        {
          ttl: config.get('THROTTLE_TTL'),
          limit: config.get('THROTTLE_LIMIT'),
        },
      ],
    }),
  ],
})
export class AppModule {}
```

你也可以使用 `useClass` 语法：

```typescript
@Module({
  imports: [
    ThrottlerModule.forRootAsync({
      imports: [ConfigModule],
      useClass: ThrottlerConfigService,
    }),
  ],
})
export class AppModule {}
```

只要 `ThrottlerConfigService` 实现了 `ThrottlerOptionsFactory` 接口，这是可行的。

#### 存储

内置的存储是一个内存缓存，它会跟踪所有请求，直到它们超过了全局选项中设置的 TTL。你可以将自己的存储选项插入到 `ThrottlerModule` 的 `storage` 选项中，只要该类实现了 `ThrottlerStorage` 接口。

对于分布式服务器，你可以使用社区提供的 [Redis](https://github.com/jmcdo29/nest-lab/tree/main/packages/throttler-storage-redis) 存储提供程序，以获得一个单一的真相源。

> info **注意** `ThrottlerStorage` 可以从 `@nestjs/throttler` 导入。

#### 时间辅助方法

如果你希望时间设置更易读，可以使用几个辅助方法。`@nestjs/throttler` 导出了五个不同的辅助方法：`seconds`、`minutes`、`hours`、`days` 和 `weeks`。使用时只需调用 `seconds(5)` 或其他辅助方法，即可返回正确的毫秒数。

#### 迁移指南

对于大多数人来说，将你的选项包装在数组中就足够了。

如果你使用了自定义存储，应将 `ttl` 和 `limit 包装在数组中，并将其赋值给选项对象的 `throttlers` 属性。

任何 `@SkipThrottle()` 装饰器都可以用于跳过特定路由或方法的速率限制。它接受一个可选的布尔参数，默认为 `true`。这在你希望跳过特定端点的速率限制时非常有用。

现在，任何 `@Throttle()` 装饰器也应接受一个对象，其字符串键对应于速率限制器上下文的名称（同样，如果没有名称则使用 `'default'`），其值是包含 `limit` 和 `ttl` 键的对象。

> 警告 **重要** 现在 `ttl` 是以**毫秒**为单位的。如果你希望以秒为单位提高可读性，可以使用此包中的 `seconds` 辅助方法。它只是将 `ttl` 乘以 1000 以转换为毫秒。

有关更多信息，请参见 [Changelog](https://github.com/nestjs/throttler/blob/master/CHANGELOG.md#500)