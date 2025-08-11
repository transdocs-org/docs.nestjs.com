### 版本控制

> info **提示** 本章节仅适用于基于 HTTP 的应用程序。

版本控制允许你在同一个应用程序中运行具有**不同版本**的控制器或单个路由。应用程序经常发生变化，有时需要进行破坏性更改，同时还需要支持旧版本的应用程序接口。

目前支持以下 4 种版本控制方式：

<table>
  <tr>
    <td><a href='techniques/versioning#uri-versioning-type'><code>URI 版本控制</code></a></td>
    <td>版本信息将包含在请求的 URI 中（默认方式）</td>
  </tr>
  <tr>
    <td><a href='techniques/versioning#header-versioning-type'><code>Header 版本控制</code></a></td>
    <td>使用自定义请求头来指定版本</td>
  </tr>
  <tr>
    <td><a href='techniques/versioning#media-type-versioning-type'><code>媒体类型版本控制</code></a></td>
    <td>通过请求的 <code>Accept</code> 请求头来指定版本</td>
  </tr>
  <tr>
    <td><a href='techniques/versioning#custom-versioning-type'><code>自定义版本控制</code></a></td>
    <td>可以使用请求中的任意部分来指定版本。你需要提供一个自定义函数来提取版本信息。</td>
  </tr>
</table>

#### URI 版本控制

URI 版本控制使用请求 URI 中的版本信息，例如：`https://example.com/v1/route` 和 `https://example.com/v2/route`。

> warning **注意** 使用 URI 版本控制时，版本号会自动添加到 <a href="faq/global-prefix">全局路径前缀</a>（如果存在）之后，并在控制器或路由路径之前。

要为你的应用程序启用 URI 版本控制，请执行以下操作：

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
// 或 "app.enableVersioning()"
app.enableVersioning({
  type: VersioningType.URI,
});
await app.listen(process.env.PORT ?? 3000);
```

> warning **注意** 默认情况下，URI 中的版本号会自动以 `v` 开头，但你可以通过设置 `prefix` 键来更改前缀，或者设置为 `false` 来禁用该功能。

> info **提示** `VersioningType` 枚举用于 `type` 属性，它可以从 `@nestjs/common` 包中导入。

#### Header 版本控制

Header 版本控制使用一个用户指定的自定义请求头来指定版本信息，请求头的值即为版本号。

Header 版本控制的 HTTP 请求示例：

要为你的应用程序启用 **Header 版本控制**，请执行以下操作：

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.HEADER,
  header: 'Custom-Header',
});
await app.listen(process.env.PORT ?? 3000);
```

`header` 属性应该指定包含版本信息的请求头名称。

> info **提示** `VersioningType` 枚举用于 `type` 属性，它可以从 `@nestjs/common` 包中导入。

#### 媒体类型版本控制

媒体类型版本控制使用请求的 `Accept` 请求头来指定版本信息。

在 `Accept` 请求头中，版本号会通过分号 `;` 与媒体类型分隔。然后应包含一个键值对来表示请求所使用的版本，例如：`Accept: application/json;v=2`。在确定版本时，键会被视为前缀，配置时需要包含键和分隔符。

要为你的应用程序启用 **媒体类型版本控制**，请执行以下操作：

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.MEDIA_TYPE,
  key: 'v=',
});
await app.listen(process.env.PORT ?? 3000);
```

`key` 属性应该指定包含版本号的键值对的键和分隔符。例如，对于 `Accept: application/json;v=2`，`key` 应设置为 `v=`。

> info **提示** `VersioningType` 枚举用于 `type` 属性，它可以从 `@nestjs/common` 包中导入。

#### 自定义版本控制

自定义版本控制允许你使用请求中的任意部分来指定一个或多个版本。通过一个 `extractor` 函数来分析传入请求，并返回一个字符串或字符串数组。

如果请求者提供了多个版本号，`extractor` 函数可以返回一个字符串数组，按从高到低的顺序排序。版本号会按照从高到低的顺序与路由进行匹配。

如果 `extractor` 返回空字符串或空数组，则不会匹配任何路由，并返回 404。

例如，如果一个请求中指定了支持版本 `1`、`2` 和 `3`，那么 `extractor` 函数**必须**返回 `[3, 2, 1]`。这确保了首先选择最高的可用路由版本。

如果提取的版本是 `[3, 2, 1]`，但只有版本 `2` 和 `1` 的路由存在，则会选择版本 `2` 的路由（忽略版本 `3`）。

> warning **注意** 由于 Express 适配器的设计限制，根据 `extractor` 返回的数组选择最高匹配版本**不能可靠工作**。对于 Express，单个版本（字符串或长度为 1 的数组）没有问题。Fastify 则正确支持最高版本匹配和单个版本匹配。

要为你的应用程序启用 **自定义版本控制**，请创建一个 `extractor` 函数，并像下面这样传入：

```typescript
@@filename(main)
// 示例 extractor 函数，从自定义请求头中提取版本列表并排序。
// 本例使用 Fastify，但 Express 的请求也可以类似处理。
const extractor = (request: FastifyRequest): string | string[] =>
  [request.headers['custom-versioning-field'] ?? '']
     .flatMap(v => v.split(','))
     .filter(v => !!v)
     .sort()
     .reverse()

const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.CUSTOM,
  extractor,
});
await app.listen(process.env.PORT ?? 3000);
```

#### 使用方法

版本控制允许你对控制器、单个路由进行版本控制，还提供了一种机制让某些资源选择不参与版本控制。无论你的应用程序使用哪种版本控制方式，使用方式都是一样的。

> warning **注意** 如果应用程序启用了版本控制，但控制器或路由没有指定版本，则对该控制器/路由的任何请求都会返回 `404` 响应状态码。同样，如果收到的请求包含了一个没有对应控制器或路由的版本，也会返回 `404` 响应状态码。

#### 控制器版本

可以为控制器指定一个版本，这将为该控制器内的所有路由设置版本。

为控制器添加版本的方法如下：

```typescript
@@filename(cats.controller)
@Controller({
  version: '1',
})
export class CatsControllerV1 {
  @Get('cats')
  findAll(): string {
    return 'This action returns all cats for version 1';
  }
}
@@switch
@Controller({
  version: '1',
})
export class CatsControllerV1 {
  @Get('cats')
  findAll() {
    return 'This action returns all cats for version 1';
  }
}
```

#### 路由版本

可以为单个路由指定版本。这个版本将覆盖影响该路由的其他版本，例如控制器版本。

为单个路由添加版本的方法如下：

```typescript
@@filename(cats.controller)
import { Controller, Get, Version } from '@nestjs/common';

@Controller()
export class CatsController {
  @Version('1')
  @Get('cats')
  findAllV1(): string {
    return 'This action returns all cats for version 1';
  }

  @Version('2')
  @Get('cats')
  findAllV2(): string {
    return 'This action returns all cats for version 2';
  }
}
@@switch
import { Controller, Get, Version } from '@nestjs/common';

@Controller()
export class CatsController {
  @Version('1')
  @Get('cats')
  findAllV1() {
    return 'This action returns all cats for version 1';
  }

  @Version('2')
  @Get('cats')
  findAllV2() {
    return 'This action returns all cats for version 2';
  }
}
```

#### 多个版本

可以为一个控制器或路由指定多个版本。要使用多个版本，只需将版本设置为数组形式。

为控制器或路由添加多个版本的方法如下：

```typescript
@@filename(cats.controller)
@Controller({
  version: ['1', '2'],
})
export class CatsController {
  @Get('cats')
  findAll(): string {
    return 'This action returns all cats for version 1 or 2';
  }
}
@@switch
@Controller({
  version: ['1', '2'],
})
export class CatsController {
  @Get('cats')
  findAll() {
    return 'This action returns all cats for version 1 or 2';
  }
}
```

#### 版本 "中立"

某些控制器或路由可能不关心版本，它们的功能在所有版本中保持一致。为了支持这种场景，可以将版本设置为 `VERSION_NEUTRAL` 符号。

无论请求中是否包含版本号，或包含哪个版本号，传入的请求都会映射到标记为 `VERSION_NEUTRAL` 的控制器或路由。

> warning **注意** 对于 URI 版本控制，`VERSION_NEUTRAL` 资源在 URI 中不会包含版本信息。

为控制器或路由添加版本中立的方法如下：

```typescript
@@filename(cats.controller)
import { Controller, Get, VERSION_NEUTRAL } from '@nestjs/common';

@Controller({
  version: VERSION_NEUTRAL,
})
export class CatsController {
  @Get('cats')
  findAll(): string {
    return 'This action returns all cats regardless of version';
  }
}
@@switch
import { Controller, Get, VERSION_NEUTRAL } from '@nestjs/common';

@Controller({
  version: VERSION_NEUTRAL,
})
export class CatsController {
  @Get('cats')
  findAll() {
    return 'This action returns all cats regardless of version';
  }
}
```

#### 全局默认版本

如果你不想为每个控制器或路由单独指定版本，或者希望为所有未指定版本的控制器/路由设置一个默认版本，可以设置 `defaultVersion`，如下所示：

```typescript
@@filename(main)
app.enableVersioning({
  // ...
  defaultVersion: '1'
  // 或
  defaultVersion: ['1', '2']
  // 或
  defaultVersion: VERSION_NEUTRAL
});
```

#### 中间件版本控制

[Middlewares](https://docs.nestjs.com/middleware) 也可以使用版本控制元数据来为特定版本的路由配置中间件。为此，将版本号作为 `MiddlewareConsumer.forRoutes()` 方法的一个参数传入：

```typescript
@@filename(app.module)
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';
import { CatsController } from './cats/cats.controller';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: 'cats', method: RequestMethod.GET, version: '2' });
  }
}
```

上述代码中，`LoggerMiddleware` 仅会应用于 `/cats` 接口的版本 `2`。

> info **注意** 中间件可以与本节中描述的任意版本控制方式一起使用：`URI`、`Header`、`Media Type` 或 `Custom`。