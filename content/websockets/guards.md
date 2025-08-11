### 守卫

WebSocket 守卫与 [常规 HTTP 应用程序守卫](/guards) 之间没有本质区别。唯一的不同是，在 WebSocket 中你应该使用 `WsException` 而不是抛出 `HttpException`。

> info **Hint** `WsException` 类是从 `@nestjs/websockets` 包中导出的。

#### 绑定守卫

以下示例使用了方法作用域的守卫。与基于 HTTP 的应用程序一样，你也可以使用网关作用域的守卫（即，在网关类前使用 `@UseGuards()` 装饰器进行修饰）。

```typescript
@@filename()
@UseGuards(AuthGuard)
@SubscribeMessage('events')
handleEvent(client: Client, data: unknown): WsResponse<unknown> {
  const event = 'events';
  return { event, data };
}
@@switch
@UseGuards(AuthGuard)
@SubscribeMessage('events')
handleEvent(client, data) {
  const event = 'events';
  return { event, data };
}
```
