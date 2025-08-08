### 中间件

中间件是一个在路由处理器之前调用的函数。中间件函数可以访问 [请求](https://expressjs.com/en/4x/api.html#req) 和 [响应](https://expressjs.com/en/4x/api.html#res) 对象，以及应用程序请求-响应周期中的 `next()` 中间件函数。**下一个**中间件函数通常用名为 `next` 的变量表示。

<figure><img class="illustrative-image" src="/assets/Middlewares_1.png" /></figure>

默认情况下，Nest 的中间件等价于 [Express](https://expressjs.com/en/guide/using-middleware.html) 的中间件。以下来自官方 Express 文档的描述说明了中间件的功能：

<blockquote class="external">
  中间件函数可以执行以下任务：
  <ul>
    <li>执行任意代码。</li>
    <li>修改请求和响应对象。</li>
    <li>结束请求-响应周期。</li>
    <li>调用堆栈中的下一个中间件函数。</li>
    <li>如果当前中间件函数没有结束请求-响应周期，它必须调用 <code>next()</code> 来将控制权传递给下一个中间件函数。否则，请求将被挂起。</li>
  </ul>
</blockquote>

你可以通过函数或带有 `@Injectable()` 装饰器的类来实现自定义的 Nest 中间件。类需要实现 `NestMiddleware` 接口，而函数则没有特殊要求。我们先使用类方法来实现一个简单的中间件功能。

> warning **警告** `Express` 和 `fastify` 处理中间件的方式不同，方法签名也不同，请在此处[了解更多](/techniques/performance#middleware)。

```typescript
@@filename(logger.middleware)
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log('请求...');
    next();
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class LoggerMiddleware {
  use(req, res, next) {
    console.log('请求...');
    next();
  }
}
```

#### 依赖注入

Nest 的中间件完全支持依赖注入。就像提供者和服务一样，它们可以注入在相同模块中可用的依赖项。通常，这是通过构造函数完成的。

#### 应用中间件

中间件在 `@Module()` 装饰器中没有位置。相反，我们使用模块类的 `configure()` 方法来设置它们。包含中间件的模块必须实现 `NestModule` 接口。我们将在 `AppModule` 级别上设置 `LoggerMiddleware`。

```typescript
@@filename(app.module)
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('cats');
  }
}
@@switch
import { Module } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {
  configure(consumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('cats');
  }
}
```

在上面的例子中，我们为之前在 `CatsController` 中定义的 `/cats` 路由处理器设置了 `LoggerMiddleware`。我们还可以通过向 `forRoutes()` 方法传递一个包含路由 `path` 和请求 `method` 的对象，进一步将中间件限制为特定的请求方法。在下面的例子中，请注意我们导入了 `RequestMethod` 枚举来引用所需的请求方法类型。

```typescript
@@filename(app.module)
import { Module, NestModule, RequestMethod, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: 'cats', method: RequestMethod.GET });
  }
}
@@switch
import { Module, RequestMethod } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {
  configure(consumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: 'cats', method: RequestMethod.GET });
  }
}
```

> info **提示** 使用 `async/await` 可以将 `configure()` 方法设为异步（例如，你可以在 `configure()` 方法体内 `await` 一个异步操作的完成）。

> warning **警告** 当使用 `express` 适配器时，NestJS 应用会默认注册 `body-parser` 包中的 `json` 和 `urlencoded`。这意味着如果你想通过 `MiddlewareConsumer` 自定义该中间件，则需要在使用 `NestFactory.create()` 创建应用程序时将 `bodyParser` 标志设为 `false` 来关闭全局中间件。

#### 路由通配符

NestJS 中间件也支持基于模式的路由。例如，命名通配符 (`*splat`) 可以用作通配符来匹配路由中的任意字符组合。在以下示例中，无论路由后面有多少字符，只要以 `abcd/` 开头，中间件就会被执行。

```typescript
forRoutes({
  path: 'abcd/*splat',
  method: RequestMethod.ALL,
});
```

> info **提示** `splat` 只是通配符参数的名称，没有特殊含义。你可以将其命名为任何你喜欢的名称，例如 `*wildcard`。

路径 `'abcd/*'` 将匹配 `abcd/1`、`abcd/123`、`abcd/abc` 等。在基于字符串的路径中，连字符（`-`）和点号（`.`）被按字面解释。然而，`abcd/` 没有任何其他字符时不会匹配该路由。为此，你需要将通配符包裹在花括号中，使其成为可选的：

```typescript
forRoutes({
  path: 'abcd/{*splat}',
  method: RequestMethod.ALL,
});
```

#### 中间件消费者

`MiddlewareConsumer` 是一个辅助类。它提供了几个内置方法来管理中间件。所有方法都可以简单地以 [流利接口](https://en.wikipedia.org/wiki/Fluent_interface) 的方式链式调用。`forRoutes()` 方法可以接受单个字符串、多个字符串、一个 `RouteInfo` 对象、一个控制器类，甚至多个控制器类。大多数情况下，你可能只需要传入一个以逗号分隔的 **控制器** 列表。下面是一个包含单个控制器的示例：

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
      .forRoutes(CatsController);
  }
}
@@switch
import { Module } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';
import { CatsController } from './cats/cats.controller';

@Module({
  imports: [CatsModule],
})
export class AppModule {
  configure(consumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes(CatsController);
  }
}
```

> info **提示** `apply()` 方法可以接受一个中间件，或者多个参数来指定 <a href="/middleware#multiple-middleware">多个中间件</a>。

#### 排除路由

有时，我们可能希望从中间件应用中 **排除** 某些路由。这可以通过 `exclude()` 方法轻松实现。`exclude()` 方法接受一个字符串、多个字符串，或一个 `RouteInfo` 对象来标识要排除的路由。

以下是一个使用示例：

```typescript
consumer
  .apply(LoggerMiddleware)
  .exclude(
    { path: 'cats', method: RequestMethod.GET },
    { path: 'cats', method: RequestMethod.POST },
    'cats/{*splat}',
  )
  .forRoutes(CatsController);
```

> info **提示** `exclude()` 方法支持使用 [path-to-regexp](https://github.com/pillarjs/path-to-regexp#parameters) 包的通配符参数。

在上面的示例中，`LoggerMiddleware` 将绑定到 `CatsController` 中定义的所有路由，**除了**传递给 `exclude()` 方法的三个路由。

这种方法提供了根据特定路由或路由模式灵活应用或排除中间件的功能。

#### 函数式中间件

我们一直在使用的 `LoggerMiddleware` 类非常简单。它没有成员、没有额外的方法，也没有依赖项。我们为什么不能用一个简单的函数来定义它呢？实际上，我们可以。这种类型的中间件称为 **函数式中间件**。让我们将日志中间件从基于类的形式转换为函数式中间件，以说明两者的区别：

```typescript
@@filename(logger.middleware)
import { Request, Response, NextFunction } from 'express';

export function logger(req: Request, res: Response, next: NextFunction) {
  console.log(`请求...`);
  next();
};
@@switch
export function logger(req, res, next) {
  console.log(`请求...`);
  next();
};
```

然后在 `AppModule` 中使用它：

```typescript
@@filename(app.module)
consumer
  .apply(logger)
  .forRoutes(CatsController);
```

> info **提示** 当你的中间件不需要任何依赖时，请考虑使用更简单的 **函数式中间件**。

#### 多个中间件

如上所述，为了将多个按顺序执行的中间件绑定，只需在 `apply()` 方法中提供一个逗号分隔的列表即可：

```typescript
consumer.apply(cors(), helmet(), logger).forRoutes(CatsController);
```

#### 全局中间件

如果我们想将中间件绑定到所有已注册的路由，可以使用 `INestApplication` 实例提供的 `use()` 方法：

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
app.use(logger);
await app.listen(process.env.PORT ?? 3000);
```

> info **提示** 在全局中间件中访问 DI 容器是不可能的。使用 `app.use()` 时，可以改用 [函数式中间件](middleware#functional-middleware)。或者，你可以使用类中间件，并在 `AppModule`（或任何其他模块）中使用 `.forRoutes('*')` 来消费它。