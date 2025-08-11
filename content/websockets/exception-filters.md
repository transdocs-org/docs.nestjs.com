### 异常过滤器

HTTP [异常过滤器](/exception-filters) 层与对应的 WebSocket 层之间的唯一区别是，你应该使用 `WsException` 而不是抛出 `HttpException`。

```typescript
throw new WsException('无效的凭证。');
```

> info **提示** `WsException` 类是从 `@nestjs/websockets` 包中导入的。

使用上面的示例，Nest 将处理抛出的异常并发送一个 `exception` 消息，其结构如下：

```typescript
{
  status: 'error',
  message: '无效的凭证。'
}
```

#### 过滤器

WebSocket 异常过滤器的行为与 HTTP 异常过滤器相同。以下示例使用了手动实例化的、作用于方法范围的过滤器。与基于 HTTP 的应用程序一样，你也可以使用网关作用域的过滤器（即，在网关类前使用 `@UseFilters()` 装饰器）。

```typescript
@UseFilters(new WsExceptionFilter())
@SubscribeMessage('events')
onEvent(client, data: any): WsResponse<any> {
  const event = 'events';
  return { event, data };
}
```

#### 继承

通常，你会创建完全自定义的异常过滤器来满足应用程序的需求。不过，也可能存在这样的用例：你希望简单地扩展 **核心异常过滤器**，并根据某些条件覆盖其行为。

要将异常处理委托给基础过滤器，你需要继承 `BaseWsExceptionFilter` 并调用继承来的 `catch()` 方法。

```typescript
@@filename()
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseWsExceptionFilter } from '@nestjs/websockets';

@Catch()
export class AllExceptionsFilter extends BaseWsExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    super.catch(exception, host);
  }
}
@@switch
import { Catch } from '@nestjs/common';
import { BaseWsExceptionFilter } from '@nestjs/websockets';

@Catch()
export class AllExceptionsFilter extends BaseWsExceptionFilter {
  catch(exception, host) {
    super.catch(exception, host);
  }
}
```

上面的实现只是一个演示该方法的框架。你实现的扩展异常过滤器应包含你自定义的 **业务逻辑**（例如，处理各种条件）。