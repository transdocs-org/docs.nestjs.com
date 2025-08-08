### 异常过滤器

HTTP [异常过滤器](/exception-filters) 层与对应微服务层之间唯一的区别是，你应该使用 `RpcException` 而不是抛出 `HttpException`。

```typescript
throw new RpcException('Invalid credentials.');
```

> info **提示** `RpcException` 类是从 `@nestjs/microservices` 包中导入的。

使用上面的示例，Nest 会处理抛出的异常并返回如下结构的 `error` 对象：

```json
{
  "status": "error",
  "message": "Invalid credentials."
}
```

#### 过滤器

微服务异常过滤器的行为与 HTTP 异常过滤器类似，只是有一个小差异。`catch()` 方法必须返回一个 `Observable`。

```typescript
@@filename(rpc-exception.filter)
import { Catch, RpcExceptionFilter, ArgumentsHost } from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { RpcException } from '@nestjs/microservices';

@Catch(RpcException)
export class ExceptionFilter implements RpcExceptionFilter<RpcException> {
  catch(exception: RpcException, host: ArgumentsHost): Observable<any> {
    return throwError(() => exception.getError());
  }
}
@@switch
import { Catch } from '@nestjs/common';
import { throwError } from 'rxjs';

@Catch(RpcException)
export class ExceptionFilter {
  catch(exception, host) {
    return throwError(() => exception.getError());
  }
}
```

> warning **警告** 当使用[混合应用程序](/faq/hybrid-application)时，默认情况下不会启用全局微服务异常过滤器。

下面的示例使用手动实例化的方法作用域过滤器。与基于 HTTP 的应用程序一样，你也可以使用控制器作用域的过滤器（即在控制器类上添加 `@UseFilters()` 装饰器）。

```typescript
@@filename()
@UseFilters(new ExceptionFilter())
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): number {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@UseFilters(new ExceptionFilter())
@MessagePattern({ cmd: 'sum' })
accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```

#### 继承

通常，你会创建完全自定义的异常过滤器来满足你的应用程序需求。然而，有些场景下你可能希望简单地扩展**核心异常过滤器**，并根据某些因素覆盖其行为。

为了将异常处理委托给基础过滤器，你需要继承 `BaseExceptionFilter` 并调用继承的 `catch()` 方法。

```typescript
@@filename()
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseRpcExceptionFilter } from '@nestjs/microservices';

@Catch()
export class AllExceptionsFilter extends BaseRpcExceptionFilter {
  catch(exception: any, host: ArgumentsHost) {
    return super.catch(exception, host);
  }
}
@@switch
import { Catch } from '@nestjs/common';
import { BaseRpcExceptionFilter } from '@nestjs/microservices';

@Catch()
export class AllExceptionsFilter extends BaseRpcExceptionFilter {
  catch(exception, host) {
    return super.catch(exception, host);
  }
}
```

以上实现只是一个展示方法的外壳。你的扩展异常过滤器实现将包含你自己定制的**业务逻辑**（例如，处理各种条件）。