### 管道（Pipes）

[常规管道](/pipes)与 WebSocket 管道之间没有本质区别。唯一的区别在于，你应该使用 `WsException` 而不是抛出 `HttpException`。此外，所有管道只会应用在 `data` 参数上（因为验证或转换 `client` 实例是没有意义的）。

> info **提示** `WsException` 类是从 `@nestjs/websockets` 包中导出的。

#### 绑定管道

以下示例使用了手动实例化的、作用域为方法的管道。与基于 HTTP 的应用程序一样，你也可以使用作用域为网关的管道（即，在网关类前使用 `@UsePipes()` 装饰器）。

```typescript
@@filename()
@UsePipes(new ValidationPipe({ exceptionFactory: (errors) => new WsException(errors) }))
@SubscribeMessage('events')
handleEvent(client: Client, data: unknown): WsResponse<unknown> {
  const event = 'events';
  return { event, data };
}
@@switch
@UsePipes(new ValidationPipe({ exceptionFactory: (errors) => new WsException(errors) }))
@SubscribeMessage('events')
handleEvent(client, data) {
  const event = 'events';
  return { event, data };
}
```