### 拦截器

拦截器是一个使用 `@Injectable()` 装饰器标注的类，并实现了 `NestInterceptor` 接口。

<figure><img class="illustrative-image" src="/assets/Interceptors_1.png" /></figure>

拦截器拥有一系列有用的功能，这些功能受 [面向切面编程](https://zh.wikipedia.org/wiki/%E9%9D%A2%E5%90%91%E5%88%87%E9%9D%A2%E7%BC%96%E7%A8%8B)（AOP）技术的启发。它们可以：

- 在方法执行前后绑定额外的逻辑
- 转换函数返回的结果
- 转换函数抛出的异常
- 扩展基本函数的行为
- 根据特定条件完全覆盖函数（例如用于缓存）

#### 基础知识

每个拦截器都实现 `intercept()` 方法，该方法接受两个参数。第一个参数是 `ExecutionContext` 实例（与 [守卫](/guards) 中使用的对象相同）。`ExecutionContext` 继承自 `ArgumentsHost`。我们之前在异常过滤器章节中见过 `ArgumentsHost`。它是一个对传递给原始处理程序的参数的封装，并且根据应用程序的类型包含不同的参数数组。您可以参考 [异常过滤器](https://docs.nestjs.com/exception-filters#arguments-host) 了解更多相关内容。

#### 执行上下文

通过继承 `ArgumentsHost`，`ExecutionContext` 还添加了一些新的辅助方法，这些方法提供了当前执行过程的额外细节。这些细节在构建可以跨多个控制器、方法和执行上下文工作的通用拦截器时非常有用。了解更多关于 `ExecutionContext` 的信息请参考 [这里](/fundamentals/execution-context)。

#### 调用处理器

第二个参数是 `CallHandler`。`CallHandler` 接口实现了 `handle()` 方法，您可以在拦截器中使用它来在某个时刻调用路由处理程序方法。如果您在 `intercept()` 方法的实现中没有调用 `handle()` 方法，那么路由处理程序方法将根本不会执行。

这种设计意味着 `intercept()` 方法有效地**包装了**请求/响应流。因此，您可以在最终路由处理程序执行**前后**实现自定义逻辑。显然，您可以在 `intercept()` 方法中编写在调用 `handle()` **之前**执行的代码，但如何影响之后的操作呢？因为 `handle()` 方法返回一个 `Observable`，我们可以使用强大的 [RxJS](https://github.com/ReactiveX/rxjs) 操作符进一步操作响应流。使用面向切面编程的术语，调用路由处理程序（即调用 `handle()`）被称为 [切入点](https://en.wikipedia.org/wiki/Pointcut)，表示这是我们插入额外逻辑的点。

例如，考虑一个传入的 `POST /cats` 请求。这个请求的目标是 `CatsController` 中定义的 `create()` 处理程序。如果在调用链中某个位置调用了不调用 `handle()` 方法的拦截器，`create()` 方法将不会执行。一旦调用 `handle()`（并返回其 `Observable`），`create()` 处理程序将被触发。而一旦通过 `Observable` 接收到响应流，就可以对流进行进一步操作，并将最终结果返回给调用者。

<app-banner-devtools></app-banner-devtools>

#### 切面拦截

我们将要讨论的第一个使用场景是使用拦截器记录用户交互（例如存储用户调用、异步派发事件或计算时间戳）。下面展示了一个简单的 `LoggingInterceptor`：

```typescript
@@filename(logging.interceptor)
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...');

    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => console.log(`After... ${Date.now() - now}ms`)),
      );
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor {
  intercept(context, next) {
    console.log('Before...');

    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => console.log(`After... ${Date.now() - now}ms`)),
      );
  }
}
```

> info **提示** `NestInterceptor<T, R>` 是一个泛型接口，其中 `T` 表示 `Observable<T>` 的类型（支持响应流），`R` 是被 `Observable<R>` 包装的值的类型。

> warning **注意** 拦截器像控制器、提供者、守卫等一样，可以通过其构造函数注入依赖项。

由于 `handle()` 返回一个 RxJS 的 `Observable`，我们有大量操作符可以用来操作流。在上面的例子中，我们使用了 `tap()` 操作符，它在 observable 流正常或异常终止时调用我们的匿名日志记录函数，但不会干扰响应周期。

#### 绑定拦截器

为了设置拦截器，我们使用从 `@nestjs/common` 包导入的 `@UseInterceptors()` 装饰器。像 [管道](/pipes) 和 [守卫](/guards) 一样，拦截器可以是控制器作用域、方法作用域或全局作用域的。

```typescript
@@filename(cats.controller)
@UseInterceptors(LoggingInterceptor)
export class CatsController {}
```

> info **提示** `@UseInterceptors()` 装饰器是从 `@nestjs/common` 包中导入的。

使用上面的构造方式，`CatsController` 中定义的每个路由处理程序都将使用 `LoggingInterceptor`。当有人调用 `GET /cats` 端点时，您将在标准输出中看到以下内容：

```typescript
Before...
After... 1ms
```

请注意，我们传递的是 `LoggingInterceptor` 类（而不是实例），将实例化责任交给了框架，并启用了依赖注入。与管道、守卫和异常过滤器一样，我们也可以传递一个就地实例：

```typescript
@@filename(cats.controller)
@UseInterceptors(new LoggingInterceptor())
export class CatsController {}
```

如上所述，上面的构造方式将拦截器附加到该控制器声明的每个处理程序上。如果我们希望将拦截器的作用范围限制为单个方法，只需将装饰器应用在**方法级别**即可。

要设置全局拦截器，我们使用 Nest 应用实例的 `useGlobalInterceptors()` 方法：

```typescript
const app = await NestFactory.create(AppModule);
app.useGlobalInterceptors(new LoggingInterceptor());
```

全局拦截器在整个应用程序中对每个控制器和每个路由处理程序都起作用。就依赖注入而言，从模块外部（如上面示例中使用 `useGlobalInterceptors()`）注册的全局拦截器无法注入依赖，因为这发生在任何模块上下文之外。为了解决这个问题，您可以直接从任何模块中使用以下构造方式设置拦截器：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
  ],
})
export class AppModule {}
```

> info **提示** 当使用此方法对拦截器进行依赖注入时，请注意，无论在哪个模块中使用此构造方式，拦截器实际上是全局的。应该在哪里进行？选择定义拦截器的模块（例如上面示例中的 `LoggingInterceptor`）。另外，`useClass` 不是处理自定义提供者的唯一方式。了解更多请参考 [这里](/fundamentals/custom-providers)。

#### 响应映射

我们已经知道 `handle()` 返回一个 `Observable`。该流包含从路由处理程序**返回**的值，因此我们可以轻松地使用 RxJS 的 `map()` 操作符对其进行修改。

> warning **警告** 响应映射功能不适用于库特定的响应策略（直接使用 `@Res()` 对象是被禁止的）。

让我们创建一个 `TransformInterceptor`，它将以一种简单的方式修改每个响应以演示该过程。它将使用 RxJS 的 `map()` 操作符将响应对象赋值给新创建对象的 `data` 属性，并将新对象返回给客户端。

```typescript
@@filename(transform.interceptor)
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
  data: T;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(map(data => ({ data })));
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { map } from 'rxjs/operators';

@Injectable()
export class TransformInterceptor {
  intercept(context, next) {
    return next.handle().pipe(map(data => ({ data })));
  }
}
```

> info **提示** Nest 拦截器同时支持同步和异步的 `intercept()` 方法。如有必要，您可以简单地将方法改为 `async`。

使用上面的构造方式，当有人调用 `GET /cats` 端点时，响应将如下所示（假设路由处理程序返回一个空数组 `[]`）：

```json
{
  "data": []
}
```

拦截器在创建整个应用程序中通用的需求解决方案方面具有巨大价值。例如，假设我们需要将每个 `null` 值转换为空字符串 `''`。我们可以使用一行代码实现，并将拦截器全局绑定，以便它将自动被每个注册的处理程序使用。

```typescript
@@filename()
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class ExcludeNullInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(map(value => value === null ? '' : value ));
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { map } from 'rxjs/operators';

@Injectable()
export class ExcludeNullInterceptor {
  intercept(context, next) {
    return next
      .handle()
      .pipe(map(value => value === null ? '' : value ));
  }
}
```

#### 异常映射

另一个有趣的使用场景是利用 RxJS 的 `catchError()` 操作符来覆盖抛出的异常：

```typescript
@@filename(errors.interceptor)
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  BadGatewayException,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(
        catchError(err => throwError(() => new BadGatewayException())),
      );
  }
}
@@switch
import { Injectable, BadGatewayException } from '@nestjs/common';
import { throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor {
  intercept(context, next) {
    return next
      .handle()
      .pipe(
        catchError(err => throwError(() => new BadGatewayException())),
      );
  }
}
```

#### 流覆盖

有时我们可能希望完全阻止调用处理程序并返回一个不同的值。一个明显的例子是实现缓存以提高响应速度。让我们来看一个简单的**缓存拦截器**，它从缓存中返回响应。在一个实际的例子中，我们还需要考虑其他因素，如 TTL、缓存失效、缓存大小等，但这超出了本讨论的范围。在这里，我们将提供一个演示主要概念的基本示例。

```typescript
@@filename(cache.interceptor)
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable, of } from 'rxjs';

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const isCached = true;
    if (isCached) {
      return of([]);
    }
    return next.handle();
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { of } from 'rxjs';

@Injectable()
export class CacheInterceptor {
  intercept(context, next) {
    const isCached = true;
    if (isCached) {
      return of([]);
    }
    return next.handle();
  }
}
```

我们的 `CacheInterceptor` 具有一个硬编码的 `isCached` 变量以及一个硬编码的响应 `[]`。需要注意的关键点是，我们在这里返回了一个由 RxJS 的 `of()` 操作符创建的新流，因此路由处理程序**根本不会被调用**。当有人调用使用了 `CacheInterceptor` 的端点时，响应（一个硬编码的空数组）将立即返回。为了创建一个通用的解决方案，您可以利用 `Reflector` 并创建一个自定义装饰器。`Reflector` 在 [守卫](/guards) 章节中有详细描述。

#### 更多操作符

使用 RxJS 操作符操作流的可能性为我们提供了许多能力。让我们再考虑一个常见的使用场景。假设您希望对路由请求进行**超时处理**。当您的端点在一段时间内没有返回任何内容时，您希望以错误响应终止。以下构造可以实现这一目标：

```typescript
@@filename(timeout.interceptor)
import { Injectable, NestInterceptor, ExecutionContext, CallHandler, RequestTimeoutException } from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  };
};
@@switch
import { Injectable, RequestTimeoutException } from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor {
  intercept(context, next) {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  };
};
```

5 秒钟后，请求处理将被取消。您还可以在抛出 `RequestTimeoutException` 之前添加自定义逻辑（例如释放资源）。