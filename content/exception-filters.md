### 异常过滤器

Nest 自带了一个内置的**异常处理层**，负责处理应用程序中所有未处理的异常。当你的应用代码未处理某个异常时，该异常将被此层捕获，然后自动发送一个合适的用户友好响应。

<figure>
  <img class="illustrative-image" src="/assets/Filter_1.png" />
</figure>

开箱即用，此操作由一个内置的**全局异常过滤器**执行，它处理类型为 `HttpException`（及其子类）的异常。当一个异常是**未识别的**（既不是 `HttpException`，也不是其子类），内置的异常过滤器将生成以下默认的 JSON 响应：

```json
{
  "statusCode": 500,
  "message": "Internal server error"
}
```

> info **提示** 全局异常过滤器部分支持 `http-errors` 库。基本上，任何包含 `statusCode` 和 `message` 属性的抛出异常都会被正确填充并作为响应发送（而不是对未识别异常使用默认的 `InternalServerErrorException`）。

#### 抛出标准异常

Nest 提供了一个内置的 `HttpException` 类，该类从 `@nestjs/common` 包中导出。对于基于典型 HTTP REST/GraphQL API 的应用程序，当某些错误条件发生时，最佳实践是发送标准的 HTTP 响应对象。

例如，在 `CatsController` 中，我们有一个 `findAll()` 方法（一个 `GET` 路由处理器）。假设此路由处理器由于某些原因抛出异常。为了演示这一点，我们将其硬编码如下：

```typescript
@@filename(cats.controller)
@Get()
async findAll() {
  throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
}
```

> info **提示** 我们在这里使用了 `HttpStatus`。这是一个从 `@nestjs/common` 包导入的辅助枚举。

当客户端调用此端点时，响应如下所示：

```json
{
  "statusCode": 403,
  "message": "Forbidden"
}
```

`HttpException` 构造函数接受两个必填参数来决定响应：

- `response` 参数定义 JSON 响应体。它可以是一个 `字符串` 或一个 `对象`，如下所述。
- `status` 参数定义 [HTTP 状态码](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status)。

默认情况下，JSON 响应体包含两个属性：

- `statusCode`：默认为 `status` 参数中提供的 HTTP 状态码
- `message`：基于 `status` 的 HTTP 错误的简短描述

要仅覆盖 JSON 响应体中的消息部分，请在 `response` 参数中提供一个字符串。要覆盖整个 JSON 响应体，请在 `response` 参数中传递一个对象。Nest 会序列化该对象并将其作为 JSON 响应体返回。

构造函数的第二个参数 `status` 应该是一个有效的 HTTP 状态码。最佳实践是使用从 `@nestjs/common` 导入的 `HttpStatus` 枚举。

还有第三个构造函数参数（可选）—— `options` ——可以用来提供一个错误 [cause](https://nodejs.org/en/blog/release/v16.9.0/#error-cause)。这个 `cause` 对象不会被序列化为响应对象，但对于日志记录非常有用，它提供了有关导致 `HttpException` 被抛出的内部错误的有价值信息。

下面是一个覆盖整个响应体并提供错误原因的示例：

```typescript
@@filename(cats.controller)
@Get()
async findAll() {
  try {
    await this.service.findAll()
  } catch (error) {
    throw new HttpException({
      status: HttpStatus.FORBIDDEN,
      error: 'This is a custom message',
    }, HttpStatus.FORBIDDEN, {
      cause: error
    });
  }
}
```

使用上述代码，响应将如下所示：

```json
{
  "status": 403,
  "error": "This is a custom message"
}
```

#### 异常日志记录

默认情况下，异常过滤器不会记录内置异常，如 `HttpException`（及其任何子类）。当这些异常被抛出时，它们不会出现在控制台中，因为它们被视为正常应用程序流程的一部分。相同的行为也适用于其他内置异常，如 `WsException` 和 `RpcException`。

这些异常都继承自基础的 `IntrinsicException` 类，该类从 `@nestjs/common` 包导出。此类有助于区分正常应用程序操作中的异常和其他异常。

如果你想记录这些异常，你可以创建一个自定义异常过滤器。我们将在下一节中解释如何操作。

#### 自定义异常

在许多情况下，你不需要编写自定义异常，而可以使用内置的 Nest HTTP 异常，如下一节所述。如果你确实需要创建自定义异常，最佳实践是创建自己的**异常层次结构**，其中你的自定义异常继承自基础 `HttpException` 类。通过这种方法，Nest 将识别你的异常，并自动处理错误响应。让我们实现这样一个自定义异常：

```typescript
@@filename(forbidden.exception)
export class ForbiddenException extends HttpException {
  constructor() {
    super('Forbidden', HttpStatus.FORBIDDEN);
  }
}
```

由于 `ForbiddenException` 扩展了基础 `HttpException`，它将与内置异常处理程序无缝协作，因此我们可以在 `findAll()` 方法中使用它。

```typescript
@@filename(cats.controller)
@Get()
async findAll() {
  throw new ForbiddenException();
}
```

#### 内置 HTTP 异常

Nest 提供了一组继承自基础 `HttpException` 的标准异常。这些异常从 `@nestjs/common` 包中导出，代表了许多最常见的 HTTP 异常：

- `BadRequestException`
- `UnauthorizedException`
- `NotFoundException`
- `ForbiddenException`
- `NotAcceptableException`
- `RequestTimeoutException`
- `ConflictException`
- `GoneException`
- `HttpVersionNotSupportedException`
- `PayloadTooLargeException`
- `UnsupportedMediaTypeException`
- `UnprocessableEntityException`
- `InternalServerErrorException`
- `NotImplementedException`
- `ImATeapotException`
- `MethodNotAllowedException`
- `BadGatewayException`
- `ServiceUnavailableException`
- `GatewayTimeoutException`
- `PreconditionFailedException`

所有内置异常还可以使用 `options` 参数提供错误 `cause` 和错误描述：

```typescript
throw new BadRequestException('Something bad happened', {
  cause: new Error(),
  description: 'Some error description',
});
```

使用上述代码，响应将如下所示：

```json
{
  "message": "Something bad happened",
  "error": "Some error description",
  "statusCode": 400
}
```

#### 异常过滤器

虽然基础（内置）异常过滤器可以自动处理许多情况，但你可能希望对异常层拥有**完全控制**。例如，你可能想要添加日志或根据某些动态因素使用不同的 JSON 模式。**异常过滤器**正是为此目的而设计的。它们让你控制发送回客户端的响应的确切流程和内容。

让我们创建一个异常过滤器，负责捕获 `HttpException` 类的异常，并为其实现自定义响应逻辑。为此，我们需要访问底层平台的 `Request` 和 `Response` 对象。我们将访问 `Request` 对象以便提取原始的 `url` 并将其包含在日志信息中。我们将使用 `Response` 对象通过 `response.json()` 方法直接控制发送的响应。

```typescript
@@filename(http-exception.filter)
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response
      .status(status)
      .json({
        statusCode: status,
        timestamp: new Date().toISOString(),
        path: request.url,
      });
  }
}
@@switch
import { Catch, HttpException } from '@nestjs/common';

@Catch(HttpException)
export class HttpExceptionFilter {
  catch(exception, host) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();
    const status = exception.getStatus();

    response
      .status(status)
      .json({
        statusCode: status,
        timestamp: new Date().toISOString(),
        path: request.url,
      });
  }
}
```

> info **提示** 所有异常过滤器都应实现通用的 `ExceptionFilter<T>` 接口。这要求你提供具有指定签名的 `catch(exception: T, host: ArgumentsHost)` 方法。`T` 表示异常的类型。

> warning **警告** 如果你使用的是 `@nestjs/platform-fastify`，你可以使用 `response.send()` 而不是 `response.json()`。不要忘记从 `fastify` 导入正确的类型。

`@Catch(HttpException)` 装饰器将所需的元数据绑定到异常过滤器，告诉 Nest 此特定过滤器正在寻找类型为 `HttpException` 的异常，而不会处理其他异常。`@Catch()` 装饰器可以接受一个参数，也可以接受逗号分隔的列表。这使你可以为几种类型的异常同时设置过滤器。

#### 参数主机

让我们看一下 `catch()` 方法的参数。`exception` 参数是当前正在处理的异常对象。`host` 参数是一个 `ArgumentsHost` 对象。`ArgumentsHost` 是一个强大的实用对象，我们将在 [执行上下文章节](/fundamentals/execution-context)\* 中进一步探讨。在此代码示例中，我们使用它来获取传递给原始请求处理程序（在异常源的控制器中）的 `Request` 和 `Response` 对象的引用。在此代码示例中，我们使用了 `ArgumentsHost` 上的一些辅助方法来获取所需的 `Request` 和 `Response` 对象。了解更多关于 `ArgumentsHost` 的信息 [此处](/fundamentals/execution-context)。

\* 这种抽象级别的原因在于，`ArgumentsHost` 在所有上下文中都起作用（例如，我们当前正在使用的 HTTP 服务器上下文，以及 Microservices 和 WebSockets）。在执行上下文章节中，我们将看到如何使用 `ArgumentsHost` 及其辅助函数的力量，在**任何**执行上下文中访问适当的 <a href="https://docs.nestjs.com/fundamentals/execution-context#host-methods">底层参数</a>。这将使我们能够编写跨所有上下文的通用异常过滤器。

<app-banner-courses></app-banner-courses>

#### 绑定过滤器

让我们将新的 `HttpExceptionFilter` 绑定到 `CatsController` 的 `create()` 方法。

```typescript
@@filename(cats.controller)
@Post()
@UseFilters(new HttpExceptionFilter())
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
@@switch
@Post()
@UseFilters(new HttpExceptionFilter())
@Bind(Body())
async create(createCatDto) {
  throw new ForbiddenException();
}
```

> info **提示** `@UseFilters()` 装饰器从 `@nestjs/common` 包导入。

我们在上面使用了 `@UseFilters()` 装饰器。类似于 `@Catch()` 装饰器，它可以接受单个过滤器实例或逗号分隔的过滤器实例列表。在这里，我们内联创建了 `HttpExceptionFilter` 的实例。或者，你可以传递类（而不是实例），将实例化的责任交给框架，并启用**依赖注入**。

```typescript
@@filename(cats.controller)
@Post()
@UseFilters(HttpExceptionFilter)
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
@@switch
@Post()
@UseFilters(HttpExceptionFilter)
@Bind(Body())
async create(createCatDto) {
  throw new ForbiddenException();
}
```

> info **提示** 在可能的情况下，优先通过类而不是实例应用过滤器。这减少了**内存使用**，因为 Nest 可以轻松在整个模块中重用同一类的实例。

在上面的示例中，`HttpExceptionFilter` 仅应用于单个 `create()` 路由处理器，使其成为方法作用域。异常过滤器可以在不同级别上作用域：控制器/解析器/网关的方法作用域、控制器作用域，或全局作用域。
例如，要将过滤器设置为控制器作用域，你可以如下操作：

```typescript
@@filename(cats.controller)
@Controller()
@UseFilters(new HttpExceptionFilter())
export class CatsController {}
```

此构造为 `CatsController` 中定义的每个路由处理器设置 `HttpExceptionFilter`。

要创建一个全局作用域的过滤器，你可以如下操作：

```typescript
@@filename(main)
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter());
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

> warning **警告** `useGlobalFilters()` 方法不会为网关或混合应用程序设置过滤器。

全局作用域的过滤器在整个应用程序中使用，适用于每个控制器和每个路由处理器。在依赖注入方面，从任何模块外部注册的全局过滤器（如上面示例中的 `useGlobalFilters()`）无法注入依赖项，因为这是在任何模块的上下文之外完成的。为了解决这个问题，你可以通过以下方式从任何模块直接注册一个全局作用域的过滤器：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: HttpExceptionFilter,
    },
  ],
})
export class AppModule {}
```

> info **提示** 使用此方法对过滤器进行依赖注入时，请注意，无论在此构造中使用的是哪个模块，过滤器实际上是全局的。应该在哪里进行此操作？选择定义过滤器（如上面示例中的 `HttpExceptionFilter`）的模块。此外，`useClass` 不是处理自定义提供者注册的唯一方式。了解更多 [此处](/fundamentals/custom-providers)。

你可以使用此技术添加任意数量的过滤器；只需将每个过滤器添加到 providers 数组中即可。

#### 捕获所有异常

为了捕获**所有**未处理的异常（无论异常类型），请将 `@Catch()` 装饰器的参数列表留空，例如 `@Catch()`。

在下面的示例中，我们有一段与平台无关的代码，因为它使用 [HTTP 适配器](./faq/http-adapter) 来发送响应，并且不直接使用任何平台特定的对象（`Request` 和 `Response`）：

```typescript
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { HttpAdapterHost } from '@nestjs/core';

@Catch()
export class CatchEverythingFilter implements ExceptionFilter {
  constructor(private readonly httpAdapterHost: HttpAdapterHost) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    // 在某些情况下，`httpAdapter` 可能在构造函数方法中不可用，
    // 因此我们应该在这里解析它。
    const { httpAdapter } = this.httpAdapterHost;

    const ctx = host.switchToHttp();

    const httpStatus =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    const responseBody = {
      statusCode: httpStatus,
      timestamp: new Date().toISOString(),
      path: httpAdapter.getRequestUrl(ctx.getRequest()),
    };

    httpAdapter.reply(ctx.getResponse(), responseBody, httpStatus);
  }
}
```

> warning **警告** 当结合一个捕获所有异常的过滤器和一个绑定到特定类型的过滤器时，应首先声明“捕获一切”过滤器，以允许特定过滤器正确处理绑定类型。

#### 继承

通常，你会创建完全自定义的异常过滤器以满足你的应用程序需求。然而，有时你可能希望简单地扩展内置的默认**全局异常过滤器**，并根据某些因素覆盖其行为。

为了将异常处理委托给基础过滤器，你需要扩展 `BaseExceptionFilter` 并调用继承的 `catch()` 方法。

```typescript
@@filename(all-exceptions.filter)
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    super.catch(exception, host);
  }
}
@@switch
import { Catch } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception, host) {
    super.catch(exception, host);
  }
}
```

> warning **警告** 继承 `BaseExceptionFilter` 的方法作用域和控制器作用域的过滤器不应使用 `new` 实例化。相反，应让框架自动实例化它们。

全局过滤器**可以**扩展基础过滤器。这可以通过以下两种方式之一完成。

第一种方法是在实例化自定义全局过滤器时注入 `HttpAdapter` 引用：

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const { httpAdapter } = app.get(HttpAdapterHost);
  app.useGlobalFilters(new AllExceptionsFilter(httpAdapter));

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

第二种方法是使用 `APP_FILTER` token <a href="exception-filters#binding-filters">如这里所示</a>。